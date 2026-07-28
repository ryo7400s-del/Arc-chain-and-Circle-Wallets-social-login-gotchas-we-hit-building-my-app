# Lessons Learned: Building SnapRoll on Circle Wallets + Arc

While building **SnapRoll** (automated cross-border payroll on Circle User-Controlled Wallets + Arc), I ran into several bugs that I couldn't find documented anywhere else. Sharing them here in plain language, in case they save another Arc/Circle builder some debugging time.

Each entry follows: **what happened → why it happened → how I fixed it.**

---

## 1. A dynamic-URL page can lose its state after social login — here's how we worked around it

**What happened:** Sign-in with Google worked perfectly on my app's main page. But the moment I used the same login flow from a different page with a dynamic URL — e.g. `/approve?scheduler=0x620fcC...` — something unexpected happened: after signing in, the browser landed back on the **app's configured base URL**, not the original `/approve?scheduler=...` URL the user started from. The login itself succeeded; the page's original path and query parameters just weren't preserved automatically.

**What we observed (not a general claim about Google OAuth):** Google OAuth in general does support registering full redirect URIs with a path (e.g. `https://yourapp.com/auth/callback`) — that's standard and well documented. What we ran into was narrower: in *our* Circle Wallets social-login setup, the configured redirect returned users to a single fixed application URL, and a dynamic, per-user query string like `?scheduler=<address>` wasn't something we could bake into a static registered redirect URI anyway (its value changes per request, so it isn't suited to being the literal registered destination). We did not exhaustively test every possible Circle SDK / Google Console configuration, so we're describing what happened in our implementation rather than a definitive statement about what is or isn't possible.

**The fix:** Store the full original URL (path + query string) in the browser before redirecting to Google, and restore it manually after login completes:

1. **Before redirecting to Google**, save the exact URL the user was on:
   ```ts
   localStorage.setItem("postLoginRedirect", window.location.pathname + window.location.search);
   ```
2. **On the app's base URL**, once login succeeds, check for that saved value. If present, don't treat the login as "finished here" — stash the result and forward the browser to the original URL:
   ```ts
   const redirectTo = localStorage.getItem("postLoginRedirect");
   if (redirectTo && redirectTo !== "/") {
     localStorage.setItem("pendingLoginResult", JSON.stringify(result));
     localStorage.removeItem("postLoginRedirect");
     window.location.href = redirectTo;
   }
   ```
3. **On the destination page**, when it first loads, check for that pending result before doing anything else, and use it to finish logging in there:
   ```ts
   const pending = localStorage.getItem("pendingLoginResult");
   if (pending) {
     localStorage.removeItem("pendingLoginResult");
     // use this result to finish logging in on this page
   }
   ```

**A caveat worth flagging:** this approach stores the raw login result (which can include a `userToken` and `encryptionKey`) in `localStorage` temporarily. Production apps should think carefully about token storage and XSS exposure before relying on this pattern as-is. At a minimum, minimize how long sensitive values remain in browser storage, remove them immediately after use, and avoid persisting more authentication data than the flow actually requires.

**The lesson:** If you're building a multi-page app with Circle Wallets + social login, don't assume login "just works" identically on every page the way it does on your main entry point, especially for pages backed by dynamic, per-user URLs. Verify what actually happens in your specific setup, and build an explicit relay for the login result if the default behavior doesn't preserve where the user started.

---

## 2. A working login can suddenly start failing for no visible reason

**What happened:** Related to the issue above — after the login flow had been working for days, it suddenly started failing with an `Invalid credentials` error (Circle error code `155140`), with no code changes on my end.

**What we observed:** Circle's login flow uses a `deviceId` obtained from the SDK, which our app cached in the browser's local storage after the first successful use. When the `155140` error started appearing, clearing the site's stored cookies and local storage caused the SDK to initialize a new local device state and restart the login flow. After that, logins succeeded again.

**We were not able to confirm the exact root cause.** It's plausible that the cached `deviceId` or related login state had fallen out of sync with what Circle's backend expected, but we can't point to a specific documented mechanism for why that happens. We're describing what worked for us, not a confirmed explanation of Circle's internals.

**The fix:** Clear the site's stored data (cookies + local storage) and log in again. As a practical safeguard, we added a check in the app: if a login attempt returns error code `155140`, the app automatically clears the relevant cached keys and prompts the user to try logging in again, instead of leaving them stuck on a cryptic error with no path forward.

**The lesson:** Not every authentication failure is necessarily a logic bug in your own code. If a previously-working login suddenly breaks with no code changes, clearing locally cached identifiers/state is worth trying before you go hunting through your own logic — even if you can't fully explain why the cache went stale in the first place.

---

## 3. The same RPC request behaved differently between my terminal and my deployed app

**What happened:** Using Foundry's `cast call` from my own computer, a contract read worked fine against `https://rpc.testnet.arc.network`. But the *same* call, made from my app's server code (a Next.js API route running as a Vercel serverless function) against that same endpoint, failed with vague errors like `missing revert data` and `CALL_EXCEPTION`.

**What we observed:** Switching to a different RPC endpoint (`https://arc-testnet.drpc.org`, which worked for us at the time of writing) resolved it immediately, with no other code changes.

**Root cause not confirmed.** We can't say for certain why the first endpoint behaved differently between our local machine and Vercel's serverless environment — possibilities include differences between local and serverless execution environments, provider-side rate limiting or routing, or temporary testnet RPC behavior. We're reporting the working fix, not a diagnosed cause.

- Endpoint that failed for us (from Vercel): `https://rpc.testnet.arc.network`
- Endpoint that worked for us (from Vercel): `https://arc-testnet.drpc.org`

Note that RPC endpoints for a given testnet can change or be superseded over time — check Arc's current RPC documentation for the up-to-date list of recommended providers rather than assuming the URLs above are still the best choice by the time you read this.

**The lesson:** "It works in my terminal" and "it works from my deployed app" are two separate claims — test both independently, especially against a testnet, where RPC provider behavior can differ by execution environment in ways that are hard to fully diagnose. If one endpoint misbehaves only from your server, trying a different provider is a cheap thing to test before assuming the bug is in your own code.

---

## 4. Decimal numbers and blockchain integers don't mix

**What happened:** Approving a payment would sometimes appear to just... freeze. No error, no crash — the button just stopped doing anything.

**Why it happened:** Blockchains don't store token amounts as decimals like `0.1`. USDC, for example, stores amounts as whole integers using 6 decimal places internally, so "0.1 USDC" is actually stored as the integer `100000`. In our case, a raw decimal string like `"0.1"` was being handed directly to a function expecting a big whole-number integer type (`BigInt`), and `BigInt("0.1")` throws. Because that error wasn't surfaced through our UI error handling, the user saw no useful error message and the button appeared to just stop responding.

To be clear, a silently "frozen" UI can have several causes — an uncaught exception like this one, a Promise that never resolves, a loading state that never gets cleared, or a state update that doesn't happen the way you expect. In our specific case it was the `BigInt` conversion, but it's worth checking all of these when a UI goes quiet with no error message.

**The fix:**
- Convert user-entered decimal amounts into the correct integer format the moment they're captured, and only convert back to decimals when displaying them to a user. A helper like viem's `parseUnits`/`formatUnits` (or the ethers.js equivalent) handles this conversion safely instead of doing it by hand:
  ```ts
  import { parseUnits, formatUnits } from "viem";

  const amountOnChain = parseUnits("0.1", 6); // 100000n
  const amountForDisplay = formatUnits(amountOnChain, 6); // "0.1"
  ```
- Wrap the conversion in a try/catch, and make sure the catch block actually surfaces something to the user ("Something went wrong") instead of failing invisibly.

**The lesson:** A UI that silently "freezes" doesn't tell you *why* on its own — check for uncaught exceptions, unresolved Promises, and stuck loading state. Any place your code does `BigInt(someValue)` on a value that might contain a decimal point is worth double-checking, since it's a common, easy-to-miss cause of exactly this kind of silent failure.

---

## 5. Duplicate rows can silently pile up from "harmless" repeated actions

**What happened:** After a payment got approved, the assigned approver received the same Telegram notification **four times** in a row.

**Why it happened:** The function that registers a Telegram account as an approver simply inserted a new database row every time it ran, with no check for "does this exact registration already exist?" During development, re-testing the connect flow, or re-deploying a contract, quietly created duplicate rows each time. None of it caused a visible error — the app worked fine — so it went unnoticed until the *volume of notifications* made the duplication obvious.

**The fix:**
```ts
// Check for an existing match before creating a new row
const existing = await db
  .select()
  .where({
    wallet_address: wallet,
    scheduler_address: scheduler,
  })
  .maybeSingle();

if (existing) return existing; // don't insert a duplicate
// otherwise, proceed to insert as normal
```

**One more thing worth knowing:** the check-then-insert pattern above helps in normal use, but it isn't airtight on its own — if two requests for the same registration happen to run at nearly the same time, both could pass the "does this exist?" check before either one finishes inserting, resulting in a duplicate anyway. The more robust fix is to also add a database-level uniqueness constraint, so duplicates are rejected outright regardless of timing:
```sql
alter table approvers
add constraint approvers_wallet_scheduler_unique
unique (wallet_address, scheduler_address);
```
We added the application-level check first since it was enough to stop the duplicates we were seeing, but pairing it with a DB constraint is the more complete fix for production use.

**The lesson:** "No error was thrown" is not the same as "this is working correctly." Silent duplication is easy to miss because nothing looks broken until a side effect — in my case, spammy repeated notifications — makes it visible. Anywhere a user action could plausibly run more than once (double clicks, repeated testing, re-deploys, or simply two requests overlapping in time), ask: what happens to the database if this runs twice — and does anything actually *prevent* that, or does it just make it less likely?

---

## Closing thought

The most valuable bugs to document aren't always the hardest ones technically — they're the ones where you search for the exact error message and find little discussion of it. That was true for both the social-login redirect behavior and the `deviceId`/`155140` issue above; we couldn't find much written about either one at the time. These write-ups describe what we observed and what worked for us in our specific setup — not confirmed statements about Circle's or Google's internals — but if you're building on Circle Wallets + Arc and hit something similarly undocumented, consider writing it up too. The next builder searching for that error message will thank you.
