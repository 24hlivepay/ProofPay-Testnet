# Arc mainnet launch checklist

Arc mainnet went live **September 16, 2026**. Step 1 details are now
published — see below. Everything after step 1 still needs doing.

## Buyer self-refund path removed from the app, PR #19, 2026-09-26

Intended flow (user's design, restated 2026-09-26): the buyer releases after
delivery; if the two sides cannot agree either can open a dispute, funds
freeze, and ONLY the admin decides (release, refund or split via
`resolveDispute`). A buyer refunding themselves is not part of the design and
the UI never had a button for it.

Found: the contract's `refund()` (buyer-only, status Funded, i.e. before the
seller confirms delivery) exists, plus dead app code that could call it:
`refundOnChain()` (never called by any page), the `refund` ABI entry, the
`refund(string)` entry in the backend Circle allowlist (so an email-wallet
buyer could call it through the API), and a placeholder `/refund` page that said
"Refund request has been submitted" without sending anything. PR #19 (branch
`fix/remove-buyer-self-refund`, opened, NOT merged) removes all of those plus
the landing line saying refund is available. Build OK, backend 155/155 tests
pass, not click-tested. Needs the user's test on the Vercel preview (testnet),
then merge.

**Still open (cannot fix in the app):** the deployed mainnet and testnet
contracts still contain `refund()` and they are immutable, so anyone calling
the contract directly (explorer or their own wallet) can still self-refund
while status is Funded. Closing it needs a NEW escrow contract without
`refund()` (same migration as the immutable-`owner` finding in section 3).
Do not edit `foundry/src/ProofPayEscrow.sol` in place: it must match what is
deployed and verified. Related: `foundry/test/ProofPayEscrow.t.sol` has
`testBuyerCanRefundBeforeDelivery`, which would go with a new contract.

## ProofPayEscrowV2 written (draft PR #20, NOT deployed), 2026-09-26

Where `refund()` came from (checked all repos/backups): it is in the original
testnet prototype (commit eb7add6, 2026-07-26, "save current prototype"),
documented in docs/TESTING.md as "pre-delivery refund" and covered by a test,
never offered by the UI, and not in the README flow. No file records the user
asking for it; the user's rule from the start was that neither buyer nor seller
can refund themselves after depositing. The 2026-09-16 self-review missed it.

V2 (`foundry/src/ProofPayEscrowV2.sol`, branch `feat/escrow-v2-no-self-refund`,
draft PR #20) = v1 with `refund()` + `FundsRefunded` removed and a two-step
`transferOwnership`/`acceptOwnership` (fixes the immutable-owner HIGH finding).
Status numbers unchanged. v1 file NOT edited (must match deployed). 57 forge
tests pass (37 new); a control run showed the same refund call succeeds on v1.
README documents the rule ("nobody refunds themselves").

Rule enforced: after deposit, funds leave only via buyer `releaseFunds` after
seller `confirmDelivery`, or admin `resolveDispute` after `openDispute`.

**Not done / next, all user decisions:**
1. Deploy V2 (needs the owner wallet to sign; Claude never signs): testnet
   first, full flow test, then mainnet; verify on explorer; update contract
   addresses in env/config (backend + frontend) so NEW escrows use V2. Escrows
   already funded on v1 finish on v1.
2. Independent audit of V2 before real funds (diff to v1 is small).
3. Decide where the new owner points (multisig?) and do the handover.
4. Blacklist MEDIUM (resolveDispute reverts for both if either address is
   blacklisted) is NOT fixed in V2, on purpose.
5. Hardhat copy `contracts/ProofPayEscrow.sol` is now out of date, untouched.
6. Until V2 is live, a MetaMask/Rabby buyer can still call v1 `refund()`
   directly while status is Funded (see the PR #19 section above).

## PR 2 (auth infrastructure) opened, 2026-09-22

Arc Studio's plan was reviewed (6 corrections: legacy connect shape must keep
working, JWT has no buyer/seller role, SIWE domain allowlist, nonce rate
limit/cleanup, HS256 pin, Circle expiry error) and its patch reviewed: applies
cleanly, 87 tests pass with no env vars. I fixed 3 small things myself:
hostOf() so a malformed FRONTEND_URL cannot crash boot, allow VERCEL_BRANCH_URL
(previews are opened from the branch alias) and VERCEL_PROJECT_PRODUCTION_URL,
`trust proxy` for the nonce rate limit, and SIWE message address must equal the
signer. PR #2 (github.com/24hlivepay/ProofPay-Mainnet/pull/2) was tested by
the user on the preview (testnet; MetaMask showed its normal 'unknown domain'
warning for the preview URL, harmless) and MERGED 2026-09-22 (squash commit
c0e0859). It changes no behaviour for users (no endpoint requires a token,
frontend untouched). Production deploy of c0e0859 to be confirmed. `SESSION_SECRET` (Secret, different value for
Production and Preview) is optional now, REQUIRED for PR 3.
PR 3 must: make a missing SESSION_SECRET fail closed (not silently skip
enforcement), send the token from the frontend, sanitize escrows (email only
to participants/admin, verificationCode only to the seller), and handle
Circle userToken expiry with a clear "sign in with email again" message.

## ⏸️ Where we left off (2026-09-22, stopped for the day)

**Done and live/merged:** PR 1 (backend hardening, #1, 767de30, live in
production, mainnet smoke test passed) and PR 2 (auth infrastructure, #2,
c0e0859, merged, no user-visible change). Production deploy of c0e0859 was
not yet confirmed when the user stopped (Vercel takes a few minutes; check
proof-pay -> Deployments, or `gh api repos/24hlivepay/ProofPay-Mainnet/deployments?environment=Production`).

**Next (tomorrow): PR 3 — the part that actually enforces auth.**
1. User sets `SESSION_SECRET` in Vercel (Secret type, a long random string,
   a DIFFERENT value for Production and Preview). Required for PR 3.
2. Send Arc Studio the PR 3 plan request. PR 3 must: require the token on
   dispute/admin/delivered/release endpoints; make a missing SESSION_SECRET
   FAIL CLOSED (not silently skip enforcement); frontend signs a SIWE message
   (nonce from /api/auth/nonce) for MetaMask/Rabby and uses the Circle
   userToken path for email wallets; sanitize escrows (emails only to
   participants/admin, verificationCode only to the seller); show a clear
   "sign in again" banner on 401 and "Your Circle session has expired. Sign in
   with email again." (Circle code 155104) without interrupting in-flight
   on-chain actions; fix /api/wallet/connect legacy path removal only after the
   frontend ships. Keep: no token for Arc Studio, patches come as zips.
3. Test PR 3 on its Vercel preview on TESTNET first (MetaMask shows an
   "unknown domain" warning on preview URLs: expected; profile is stored per
   browser origin so it must be re-entered on each new preview link).

**Workflow (settled):** Arc Studio gets no GitHub token. It gives a patch zip
(toolbar download icon); Claude applies it on a scratch copy, reviews, runs
`npm test` (clean env), fixes small issues, pushes a BRANCH, opens the PR with
`gh` (installed and logged in on this Mac as 24hlivepay). `main` is protected
(PR required, no bypass); the user tests the Vercel preview (testnet only,
preview shares the production DATABASE_URL) and says OK, then Claude merges.
Delete every downloaded zip after use (Trash; user empties it).

**Still open, lower priority:** swap tab ON HOLD (no official Uniswap/WETH/
cirBTC on Arc mainnet; that "UniversalRouter" has owner()); Neon branch for the
Preview database; Blob token Config->Secret; independent contract audit;
README/docs still say testnet; server-side profile storage (profile is
localStorage per origin/wallet today); Circle userToken/encryptionKey in
localStorage instead of httpOnly cookies; PR 4 (CORS: allow localhost,
proofpay.online, FRONTEND_URL, deployment VERCEL_URL/branch URL).

## Arc Studio security review + PR 1 branch pushed, 2026-09-21

ProofPay was given to Arc Studio for review. Findings (verified against our
code): dispute/admin endpoints trust an unsigned `wallet` value; /delivered
had no state check; /release and /deposit trusted the caller; escrow IDs used
Math.random; /api/wallet/connect never verifies its signature; verify-seller
had no rate limit; GET /api/escrow/:id exposes emails.

PR 1 (cheap hardening) went through 3 review rounds here (v3 accepted):
on-chain status checks for delivered/release/deposit/resolved, verify-seller
10-minute lockout (with an expiry-reset bug found in v2 and fixed),
crypto-random IDs and codes, logic moved to backend/lib/hardening.js with
node:test tests that need no env vars (47 pass). Pushed by me as branch
`fix/cheap-hardening` (commit 6241ef2), NOT merged: PR link
github.com/24hlivepay/ProofPay-Mainnet/pull/new/fix/cheap-hardening.
Before merging: test on the Vercel preview with testnet escrows only
(preview shares the production DATABASE_URL).

Still open: PR 2/3 (signature/nonce auth, Circle userToken ownership check,
email visibility), PR 4 (CORS). Circle skill note: userToken/encryptionKey
should live in httpOnly cookies, ProofPay keeps them in localStorage.

GitHub `main` is now protected by a ruleset (PR required, no bypass, no force
push/delete), so direct pushes to main fail; work goes via branches + PR.
Circle CLI + skills installed in the extracted Arc Studio project
(~/Documents/proofpay_project_management_hub_a6oey0, CLI under ~/.npm-global).

## Vercel env check for storage, 2026-09-20 (read-only, via browser pane)

Confirmed in the Vercel dashboard (project `proof-pay`): `DATABASE_URL`
is set for Production + Preview (Neon integration `proofpay-db`), so
escrows/disputes/messages persist in Postgres, not `/tmp`.
`BLOB_READ_WRITE_TOKEN` is set for Production/Preview/Development, so
dispute evidence goes to Vercel Blob. Vercel flags the Blob token
"Needs Attention": it is stored as type *Config* (value visible to
anyone with project access) although it is a secret. Nothing changed.
Optional hygiene: rotate Blob credentials and re-save as Secret.
`DISPUTE_ADMIN_WALLET` (Prod/Preview Secret), `CIRCLE_API_KEY_MAINNET`
(Secret) and `VITE_CIRCLE_APP_ID_MAINNET` are also present.

## Resolved records show the full conversation (collapsible), 2026-09-20

After resolving, the admin's "past decisions" card and the buyer/seller
dispute view now show the whole thread. `DisputeThread` got a
`collapseAfter` prop: >3 messages shows the first 3 plus "See more (N
more)" / "See less". Active disputes are unchanged (all messages).
Files: `DisputeThread.jsx`, `AdminDisputes.jsx` (ResolvedCase),
`DisputeResponse.jsx`. Not click-tested (no local Circle keys).

## Admin can now converse in a dispute without settling, 2026-09-20

Gap: the admin's dispute card had one note box tied to one Confirm
button that always released funds on-chain (and required a note). Asked
for: conversation only, or release only, or release with conversation.

Backend (`server.js`): `dispute.messages[]` (from admin/buyer/seller,
text, sentAt; 200-message cap, 2000 chars, active disputes only) with
`POST /api/admin/disputes/:id/message` (admin) and
`POST /api/escrow/:id/dispute/message` (participants). The resolve
endpoint no longer requires a note.

Frontend: new `components/DisputeThread.jsx`. AdminDisputes shows the
thread + "Send message (no payment released)"; the note field is now
optional and Confirm is "Resolve and release payment". DisputeResponse
(buyer/seller) shows the thread with a reply box while the dispute is
active; an empty resolution note is no longer rendered.

Needs backend redeploy. Not click-tested (no local Circle keys). Notes:
messages have no notification (parties see them when they open the
dispute page); existing disputes have no `messages` field and work fine
(treated as empty). MyDisputes list does not show the thread.

## Stale Circle confirmation label on dispute (second occurrence), 2026-09-20

After the allowlist fix, a seller's Open Dispute popup read "Confirm
Delivery ... 0 USDC" — `openDisputeOnChain` passed no `display`, so the
shared Circle SDK reused confirmDelivery's copy (same singleton
mechanism as the 2026-09-17 plain-transfer bug). Fixed: `openDisputeOnChain`
takes `{side, name}` (Dispute.jsx derives it by comparing the connected
wallet to `order.buyerWallet`) and shows "Open Dispute", "Dispute opened
by the Seller (name)", funds frozen / none released. Also gave
`refundOnChain` and `resolveDisputeOnChain` their own displays so no
Circle contract call in `proofpayContract.js` is left without one.
Rule going forward: every `executeCircleProofPay` call must pass a
`display`. Not click-tested (no local Circle keys).

## Circle-wallet users couldn't open a dispute, 2026-09-20

A seller on a Circle (email) wallet hit "This Circle contract operation
is not allowed." on "Open dispute and freeze funds". Cause: the backend
allowlist for `/api/circle/contract-execution` (`escrowFunctions` in
`server.js`) only had createEscrow / confirmDelivery / releaseFunds /
refund — `openDispute(string)` was never added, so any Circle-wallet
dispute was rejected before reaching Circle. Pre-existing (testnet too),
not a mainnet regression. Added `openDispute(string)`.

Deliberately NOT added: `resolveDispute(string,uint256)` — admin-only and
the admin resolves with an EOA. If admin ever uses a Circle wallet, add it
then. Needs a backend redeploy on Vercel to take effect; not testable
locally (no Circle keys).

## Product / Service added to the last two summary boxes, 2026-09-20

Screenshot of SellerVerification's Escrow Summary showed no product
name. Audited every buyer/seller info box: BuyerDeposit, SellerAccept
(both steps) and WaitingSeller already had it; SellerVerification and
EscrowActive did not. Added a "Product / Service" row to both, so all
summary boxes now carry it.

## Seller invite link had no email-wallet option, 2026-09-20

User (a seller with no browser wallet) opened a buyer's invite link and
hit "MetaMask is not installed" with no alternative. Not a regression —
a design gap since testnet: `SellerAccept.jsx` only called the EOA
connect path, while Circle email login lived only on /login and always
redirected to /dashboard afterward.

Fix: the connect step now shows "Connect MetaMask / Rabby" plus "Sign in
with Email (Circle wallet)". The email button stores
`proofpay-post-login-route` (+ `proofpay-invite-resume`) in
sessionStorage and goes to /login; `OtpVerification.jsx` now navigates to
that route instead of /dashboard when set, and SellerAccept reopens at
the review step. If a Circle session already exists it shows "Continue
with Circle wallet (0x…)". Built + linted; not click-tested (needs
Circle keys locally — same blocker as before).

## ⏸️ Where we left off (2026-09-19, stopped for the day)

Mainnet has been live since 2026-09-17. Today's session covered a lot
of UI/UX polish on top of the already-live app, newest first (all
sections below have full detail):

1. Wallet dropdown spacing tightened twice (py-2 → py-1.5, plus
   smaller text and `whitespace-nowrap` so nothing wraps).
2. Dropdown width matched to the wallet pill (`w-56` → `w-full`).
3. EscrowActive: removed a redundant status box.
4. Found and fixed two records missing the standard 3 credentials
   (Name/Email/Wallet): EscrowActive was missing both wallet rows,
   SellerVerification was missing everything (had none of the three,
   for either party).
5. Order-card headers restructured twice: first moved counterparty
   name up under the Escrow ID (across all 4 order lists — Active,
   Pending, Completed, Cancelled), then trimmed to exactly 2 lines
   (name, then ID, matched font size) with Product moved into the
   stats grid instead.
6. Wallet addresses truncated everywhere shown for confirmation
   (Profile, CreateEscrow, BuyerDeposit, SellerAccept, CircleWallet
   header) — full address kept only on the Receive/deposit screen,
   with a shared `shortenAddress()` util and a `copyValue` prop
   pattern so the copy button always copies the full address.
7. Copy button restyled to match real wallet-app conventions
   (MetaMask/Etherscan-style ghost icon, no border/box) after the
   first boxed-emoji version was flagged as looking odd.
8. Buyer/seller email rows made to always show (dash when empty)
   instead of disappearing when unset.
9. Seller name/email visibility bug fixed on BuyerDeposit (was gated
   behind code-verification while Seller wallet wasn't — inconsistent,
   confusing partial reveal).
10. CreateEscrow restructured: Buyer Information made read-only
    (pulled from profile), Product Information split into its own
    box, and profile completion made mandatory before the form opens
    (covers both Circle email-login and MetaMask/Rabby wallets).
11. Profile page: real Name + Email fields (was 100% fake placeholder
    data before today), save now shows a clear confirmation and
    auto-redirects home, and a "Profile" entry added to the wallet
    dropdown on every page.

Everything above is committed and pushed to `ProofPay-Mainnet` (code)
and this checklist repo. Nothing is mid-edit or broken.

**Known local-testing limitation** (came up repeatedly today): the
backend refuses to boot locally without real `CIRCLE_API_KEY` env
vars (validated at import time, no `.env` present, only
`.env.example`). Several of today's fixes on backend-data-dependent
pages (BuyerDeposit, SellerVerification, order lists) were verified by
careful diff re-read + `vite build`/lint instead of a live click-through,
since the pattern used was already proven correct elsewhere in the
session. Worth getting real (or sandbox) Circle keys into a local
`.env` at some point so this stops being a recurring blocker.

**Not done / still open:**
- Independent professional smart-contract audit of `ProofPayEscrow.sol`
  (self-review only so far).
- Dispute pages (MyDisputes/DisputeResponse/AdminDisputes) were
  deliberately left out of the last few consistency passes (copy
  buttons, buyer/seller info) — flagged to the user multiple times,
  never explicitly requested. Worth asking about next time UI work
  comes up.

**Resume here tomorrow** — no blocked/in-progress task, just pick up
whatever the user brings up next.

## Dropdown spacing tightened once more (py-2 → py-1.5), 2026-09-19

User wanted even less vertical space between the wallet-address row,
Profile, Change Wallet, and Disconnect. Dropped all four rows'
padding from `py-2` to `py-1.5` in both `useWalletBadge.jsx` and
`Home.jsx`. Verified visually via `vite preview`.

## Dropdown width fix looked bad in practice — tightened it up, 2026-09-19

Matching the dropdown to the pill's width (previous entry) made
"Disconnect Wallet" wrap to two lines while every row still had
generous `py-3` padding — an uneven, oddly-spaced result once actually
seen. Fixed by shrinking to `text-xs`, `py-2`/`px-3`, adding
`whitespace-nowrap`, and shortening "Disconnect Wallet" to just
"Disconnect" so all three rows fit on one line at that width.
Verified visually via `vite preview`.

## Wallet dropdown now matches the pill's width exactly, 2026-09-19

Real screenshot: the dropdown menu (Profile/Change Wallet/Disconnect
Wallet) was a fixed `w-56`, visibly wider than the "Wallet:
0xf0d2...1acb" pill that opens it. Changed both copies
(`useWalletBadge.jsx`, `Home.jsx`) to `w-full` — since the dropdown's
`relative` wrapper is a flex sibling of the pill in Navbar's flex row
(no flex-grow), its natural width already equals the pill's width, so
`w-full` makes the dropdown match exactly. "Disconnect Wallet" now
wraps to two lines at that width, which is the expected tradeoff.
Verified visually via `vite preview` on both Home.jsx's dashboard and
a shared-hook page (Profile) — both align correctly.

## EscrowActive: removed a redundant status box, 2026-09-19

Real screenshot: the green "Deposit confirmed on Arc Mainnet" box and
the yellow "Funds Locked / USDC is held in the live smart contract
until the seller confirms delivery" box right below it were saying
close to the same thing back to back. Asked to keep the green box,
move just the "Funds Locked" heading above it, and drop the box +
paragraph below.

Moved the (still status-colored, still switches to "Seller Confirmed
Delivery" once delivered) heading above `TransactionProof`, removed
the wrapping colored box and its descriptive paragraph.

## Found two records still missing the standard 3 credentials, 2026-09-19

User clicked "Open Escrow" from a real Active Purchases card and
found `EscrowActive.jsx` (buyer's detail page) had Name+Email for
Buyer and Seller but no wallet row for either — the third of the
Name/Email/Wallet trio we'd standardized per record. Asked me to also
check the seller side for the same gap.

Checked `SellerVerification.jsx` (seller's equivalent detail page,
reached from Active Sales → View Sale) — it had *none* of the three,
for either party, across any of its states (Seller Accepted / Funds
Locked / Delivered / Released) — just the verification code and
delivery button, no identity info at all.

Fixed both: `EscrowActive.jsx` gained Buyer wallet / Seller wallet
rows (shortened + copyable, matching the established pattern).
`SellerVerification.jsx` gained a full new "Escrow Summary" box above
its status-dependent content — Buyer/Seller Name, Email, Wallet,
Amount, Escrow ID — shown whenever `escrowData.escrowId` is loaded, so
it's present through every status the page handles.

## Header follow-up: name+ID only (matched size), product to the grid, 2026-09-19

From another real screenshot: the header had grown to 3 lines (Escrow
ID, product name, counterparty name) with mismatched sizing (ID a big
h2, name a small caption). Asked to check understanding first before
implementing — confirmed the plan, then confirmed the role direction
explicitly: buyer's dashboard header shows the *seller's* name+ID,
seller's dashboard shows the *buyer's*.

Trimmed each header to exactly 2 lines — name first, escrow ID right
under it, both `text-lg font-bold` (equal weight now). Product name
moved out of the header into the stats grid as its own tile, so each
grid is back to its original column count
(`sm:grid-cols-2 lg:grid-cols-4` for Active/Pending/Completed,
`sm:grid-cols-3` for Cancelled). Same four files as the previous
round: `ActiveOrders.jsx`, `PendingOrders.jsx`, `CompletedOrders.jsx`,
`CancelledOrders.jsx`.

Still couldn't live-test locally (same `CIRCLE_API_KEY` boot blocker);
verified via `vite build` (clean) + full diff re-read across all four
files.

## Counterparty name moved into the card header, every order list, 2026-09-19

From a real "Active Purchases" screenshot: Escrow ID + product name at
top-left, but Seller name was buried as just one tile inside the
4-column stats grid below, not immediately visible. Asked for it
right under the serial number instead, and — confirmed explicitly in
a follow-up — the same treatment on every record type, not just
Active: Pending, Completed ("Payment Received"/"Payment Released"),
and Cancelled too, both buyer-side and seller-side.

Added a `{role}: {name}` byline directly under the product-name line
in the card header of `ActiveOrders.jsx`, `PendingOrders.jsx`,
`CompletedOrders.jsx`, and `CancelledOrders.jsx` — all four share the
same header+stats-grid card shape, each toggled by `isSellerRole` (or
`seller` in CompletedOrders). Removed the now-redundant name tile from
each stats grid and dropped each grid by one column
(`lg:grid-cols-4`→`sm:grid-cols-3`, or `sm:grid-cols-3`→`sm:grid-cols-2`
for Cancelled) so the remaining tiles fill the row evenly.

Not live-tested: same blocker as the last two rounds — local backend
won't boot without real `CIRCLE_API_KEY` env vars, and these list
pages need `GET /api/escrows` data. Verified via `vite build` (clean)
+ full diff re-read instead; the pattern is a straightforward JSX
move, structurally identical to a change already visually verified in
CreateEscrow's header this session.

## Truncated wallet addresses everywhere they're shown for confirmation, 2026-09-19

User: full 42-char addresses aren't needed in most spots — just enough
to confirm identity, "like big apps show it" (MetaMask/Etherscan-style
`0xabc1...def0`), with the same copy button so the full value is still
one click away.

Added `frontend/src/utils/address.js` (`shortenAddress`). Applied to:
`CreateEscrow.jsx` Buyer Information, `Profile.jsx` Connected Wallet,
`BuyerDeposit.jsx`/`SellerAccept.jsx` Buyer/Seller wallet rows,
`CircleWallet.jsx`'s header chip. Each `SummaryRow`/`InfoRow` gained a
separate `copyValue` prop distinct from the displayed `value`, so the
copy button always copies the full address even though only the short
form is shown.

Deliberately kept `CircleWallet.jsx`'s "Your deposit address" box
(Receive tab) full-length — that one exists specifically so someone
can send funds *to* it, and truncating an address you're handing
someone to receive money would be a real footgun, not just a style
choice. Real apps (Coinbase, MetaMask) don't truncate that particular
screen either.

## Follow-up: email rows should always show, dash when empty, 2026-09-19

User immediately caught the previous round's fix was still hiding the
row entirely when a value (e.g. seller email, since the seller hadn't
added one) was empty — confusing, looked like a missing field. Every
`SummaryRow` in these files already falls back to `value || "—"`, so
removed the `{value && (...)}` wrapper conditionals in `BuyerDeposit`,
`EscrowActive`, `SellerAccept` and the inline-div version in
`WaitingSeller` and let that existing fallback render consistently —
every row (Buyer, Buyer email, Seller, Seller email, etc.) now always
shows, with "—" when unset.

## Fixed inconsistent buyer/seller name+email visibility across records, 2026-09-19

User caught this live: on BuyerDeposit (right after the seller
accepted and shared their OTP), Seller wallet showed but Seller name
and both people's emails were missing — a confusing half-revealed
state. Root cause: Seller name/email were gated behind seller-code
`verified`, while Seller wallet was not, and Buyer email had never
been added to this summary at all.

Fixed: Seller name/email in `BuyerDeposit.jsx` now show as soon as
they exist on the record (same visibility as Seller wallet, i.e. right
after accept — no longer waiting for code verification). Added Buyer
email there too. Extended the same Buyer/Seller email fields to every
other core "escrow record" summary screen: `EscrowActive.jsx` (Buyer
email), `SellerAccept.jsx`'s post-connect review (Buyer email),
`WaitingSeller.jsx`'s pre-accept summary (Buyer email — seller isn't
known yet at that stage). `GenerateLink.jsx` intentionally skipped
(confirmed unreachable/dead route). Dispute pages (MyDisputes/
DisputeResponse/AdminDisputes) intentionally not touched — same
lower-priority call as the copy-button work, flagged to the user again
in case they want it done too.

Not live-tested end-to-end this round: the local backend refuses to
boot without real `CIRCLE_API_KEY` env vars (validated at import
time), and no `.env` exists locally — only `.env.example`. Verified by
re-reading the exact diff instead; the pattern used
(`{escrowData.xEmail && <SummaryRow .../>}`) is identical to rows
already proven correct in-browser earlier this session.

## Profile save now shows clear confirmation + auto-redirects, 2026-09-19

User felt the "Saved ✓" button-text swap alone wasn't enough feedback
after saving a profile. Added a green "✓ Profile saved — taking you
back home..." message under the Save button, and after a ~900ms pause
(so the message is actually readable) `handleSave` now navigates to
`/dashboard` automatically.

## CreateEscrow restructured: profile-driven, profile now mandatory, 2026-09-19

User's ask, three parts:
1. Buyer Information on CreateEscrow should stop being an editable
   text field — it should be a fixed box showing Name / Email / Wallet
   pulled straight from the buyer's profile.
2. The manually-typed fields (product, amount, asset, description)
   should move into their own separate Product Information box.
3. (Confirmed, no change needed) Seller Information already gets its
   own box after the seller accepts+verifies, from the 2026-09-18 work.
4. Most important, called out as "the first thing to actually do":
   profile completion should be mandatory before the buyer form is
   usable at all — for a wallet created via Circle email login *or* a
   first-time MetaMask/Rabby connection alike.

Implemented in `CreateEscrow.jsx`: Buyer Information is now three
`InfoRow`s (Name, Email, Wallet + copy button) with an "Edit in
Profile" link, no longer an `InputField`. Added a "Product
Information" box below it holding what used to be mixed into Buyer
Information. Added an early-return gate: if `getProfileName(walletAddress)`
is empty, the page shows a blocking "Complete your profile first"
screen (message text differs slightly depending on whether a wallet
is connected yet at all) instead of the form — this naturally covers
both wallet types since profile storage is keyed by address, not by
wallet type. Used lazy `useState` initializers for `buyerName`/
`buyerEmail` (not a post-mount `useEffect` alone) specifically so the
gate doesn't flash on-screen for users who already have a profile.

Verified via `vite preview` with three seeded states: no wallet
connected, wallet connected with no profile, and wallet connected with
a saved profile — each showed the correct screen.

## Copy button follow-up: restyled to match real wallet-app practice, 2026-09-19

User flagged the first version — a bordered 📋-in-a-box button, with
the dropdown showing the full raw address CSS-ellipsis-cut mid
character and wrapping to two lines inside its own bordered card —
as looking odd, and asked to check what real apps actually do.

Redesigned `CopyButton.jsx` as a minimal ghost icon button (outline
copy/checkmark SVGs, no border or background, just a subtle hover
state) — the MetaMask/Etherscan pattern. Also fixed both wallet
dropdowns (`useWalletBadge.jsx`, `Home.jsx`) to show the same
truncated `0xabc1...def0` format already used on the pill itself,
instead of the full 42-char address getting cut off mid-string by
CSS `truncate`.

## New: copy-to-clipboard buttons on wallet addresses, 2026-09-19

User asked for a copy button wherever a wallet address is shown (their
example: the wallet dropdown). Added a shared
`frontend/src/components/CopyButton.jsx` (writes to clipboard, shows a
✅ for 2s, falls back to an alert on failure) and wired it into every
wallet-address display in the main flow: the Navbar wallet dropdown
(both `useWalletBadge.jsx`'s shared menu and Home.jsx's own copy —
previously only showed the truncated address on the pill itself, now
the dropdown shows the full address + copy), Profile's Connected
Wallet box, My Wallet's header address chip, and the buyer/seller
wallet rows on BuyerDeposit.jsx and SellerAccept.jsx.

Deliberately did NOT touch the dispute pages (MyDisputes,
DisputeResponse, AdminDisputes) — lower-traffic admin/edge-case
screens with dense single-line JSX; flag if the user wants those too.

Verified via `vite preview`: buttons render and click correctly in
every spot, and the clipboard write itself was confirmed against a
pre-existing button (CircleWallet.jsx's original "Copy address") that
fails with the identical `NotAllowedError` in this sandboxed preview
browser — i.e. a testing-environment limitation (Permissions-Policy
blocks clipboard-write for automation), not a bug in the new code.
Should be re-confirmed with one real click on proofpay.online.

Also used a `variant` prop on CopyButton (not raw className string
concatenation) for the light-background instance on CircleWallet's
gradient header, specifically to avoid a repeat of the Tailwind
class-merge-order bug from 2026-09-17 (the "invisible buttons"
incident) — two conflicting `border-*`/`bg-*` utility classes on one
element is exactly the shape that bug had.

## ⏸️ Where we left off (2026-09-18, stopped for the day)

Mainnet itself has been live and tested since 2026-09-17 (real
MetaMask escrow test succeeded, dual-network toggle, Circle Console,
Vercel cutover — all done). Today's session was all about the new
Profile feature, built and pushed in three follow-up rounds (see
sections below, newest first):

1. Real user profiles (name, keyed by wallet) with buyer/seller
   auto-fill, plus a "Profile" entry in the wallet dropdown.
2. Added an Email Address field to Profile; renamed the name field's
   label to "User Name" (still free text, personal or company name).
3. Removed the "Business / Seller Name" field from CreateEscrow
   entirely — seller name/email now come from the seller's own saved
   profile at accept time, and only appear to the buyer once the
   seller is verified. Buyer's own email now shows under Buyer Name
   on the create form.

Everything above is committed and pushed to `ProofPay-Mainnet` (code)
and this checklist repo. Nothing is mid-edit or broken.

**Not done / open items:**
- Independent professional smart-contract audit of `ProofPayEscrow.sol`
  (self-review only so far — see the security-review notes further
  down this file).
- Profile's stats box (Orders / Success rate) was removed as fake
  placeholder data rather than fixed — real stats would need backend
  aggregation per wallet and haven't been built. Revisit only if asked.
- No other open bugs or half-finished work as of this stop point.

**Resume here tomorrow** by picking either of the two items above, or
whatever the user brings up next — there is no blocked/in-progress
task to continue first.

## Profile follow-up: seller name moved off the buyer form, emails wired in, 2026-09-18

User wanted the "Business / Seller Name" field gone from CreateEscrow
entirely — the buyer shouldn't type a placeholder for someone who
hasn't even connected yet. The seller now supplies their own name (and
email, silently pulled from their saved profile) when they accept via
SellerAccept, same as before for the name; added `sellerEmail` the
same way, stored on `escrow.sellerEmail` by the accept endpoint.

CreateEscrow.jsx: removed the whole "Seller Information" section and
the `sellerName` required-field check; added a `buyerEmail` line
(pulled from the buyer's own profile) shown right under Buyer Name,
and it's now included in the escrow-creation payload.

BuyerDeposit.jsx: the Seller name/email in the Escrow Summary now only
render once `sellerVerified` is true (gated behind the same `verified`
flag the code-verification step already tracks) — the buyer sees who
they're paying only after the seller has proven control of their
wallet, not the moment they accept. EscrowActive.jsx (reached only
after verification + deposit) shows Seller email unconditionally since
verification is already guaranteed there.

Also removed the "Seller Name" summary row from GenerateLink.jsx and
WaitingSeller.jsx (the two "link generated / waiting for seller"
screens) since it would always be blank at that stage now — checked
both are shown only before acceptance, confirmed via each page's own
navigate-away-on-accept logic.

## Profile follow-up: added email address field, 2026-09-18

User asked for an Email Address field too, and for the name field's
label to read "User Name" (still free text — personal or company
name). Added `getProfileEmail`/`setProfileEmail` to
`frontend/src/utils/profile.js` (same per-wallet-address localStorage
pattern as the name). Profile.jsx pre-fills email from the Circle
login email (`proofpay-email`) when the profile has none saved yet.
Both fields are plain always-editable inputs under one "Save Profile"
button, so changing either later just means re-typing and saving
again — no separate edit mode needed.

## New feature: real user profiles + buyer/seller name auto-fill, 2026-09-18

Profile.jsx was 100% fake placeholder data (hardcoded wallet address,
fake "12 Orders / 100% Success" stats, no name field). Buyer/seller
names on CreateEscrow.jsx were plain manually-typed text every time.

Implemented (localStorage-based, keyed by wallet address, no backend
schema change needed beyond one field on accept):
- `frontend/src/utils/profile.js` — `getProfileName`/`setProfileName`,
  stores a name under `proofpay-profile-name:<address>`.
- `Profile.jsx` rebuilt: shows the real connected wallet address (was
  hardcoded), editable Name field that saves to the above storage.
  Removed the fake Orders/Success stats box — showing fabricated
  numbers as real user stats was worse than showing nothing; real
  stats would need backend aggregation and weren't asked for.
- `CreateEscrow.jsx` auto-fills Buyer Name from the connected wallet's
  saved profile (still editable), and saves whatever name is typed
  back to the profile on submit — so it also self-populates over time
  for anyone who never visits the Profile page directly.
- `SellerAccept.jsx`: after the seller connects their wallet, an
  editable "Your Name" field appears (pre-filled from their saved
  profile, falling back to the buyer's placeholder text if the seller
  has none yet). On Accept, this is sent as `sellerName` and saved to
  the seller's own profile. Backend `/api/escrow/:id/accept` now
  accepts an optional `sellerName` and overwrites the placeholder with
  it.
- Added a "Profile" entry to the wallet dropdown menu (both
  `useWalletBadge.jsx`'s shared menu and `Home.jsx`'s own copy) so it's
  reachable from every page, not just by typing the URL.

Verified via `vite build` + `vite preview` with a seeded fake wallet
session before pushing: wallet address displays correctly, saved name
pre-fills on both Profile and CreateEscrow, and the wallet-dropdown
Profile link opens the page.

## More hardcoded "ARC Testnet" strings found via live testing, 2026-09-17

User spotted BuyerDeposit.jsx showing "...lock the USDC in the live ARC
Testnet contract" on mainnet. Grepping for it turned up two more in the
same shape: SellerVerification.jsx's escrow-locked message and
EscrowActive.jsx's page subtitle — both files already used
`getNetworkConfig().chainName` correctly elsewhere, just not on these
specific lines, which is presumably why the original network-wiring
pass missed them. Fixed all three. Re-swept the whole frontend for
"ARC "/"arcscan.app"/"5042002"/hardcoded testnet RPC URLs afterward —
clean except four landing-page mentions of generic "ARC Blockchain"
(Footer/Features/FAQ marketing copy), which don't claim a specific
network and don't need fixing.

## Bug found via live testing (not network-specific, affects testnet too)

**Fixed 2026-09-17:** My Wallet's "Send Token" (a plain Circle wallet
transfer) showed a Circle confirmation popup reading "Lock 1 USDC" —
escrow language on an unrelated action. Cause: Circle's Web SDK
(`circleSdk`) is a shared singleton; `executeCircleChallenge` only sets
its confirmation copy when a `display` object is passed, and
`sendCircleToken` never passed one, so it kept showing whatever label a
*previous* call had set (e.g. an earlier escrow deposit's
`"Lock ${amount} ${symbol}"` in the same session). Fixed in
`circleTransactions.js`/`CircleWallet.jsx`: `sendCircleToken` now
builds and passes its own `display` (title "Send Token", confirmLabel
"Send X SYMBOL"), so a plain transfer can no longer inherit stale
escrow copy.

## ⏸️ Where we left off (2026-09-17, live-testing in progress)

Contracts deployed/verified, frontend + backend wired for both networks,
Circle Console configured, Vercel repointed to `ProofPay-Mainnet` with
mainnet env vars set. **Now live-testing proofpay.online directly.**

- [x] Circle email sign-up on mainnet — **works**. Confirmed live: logging
      in with an email already used on testnet correctly got a *different*
      wallet address (Circle issues a separate wallet per network), which
      proves `CIRCLE_API_KEY_MAINNET` / `VITE_CIRCLE_APP_ID_MAINNET` /
      `blockchain: "ARC"` are all working end to end.
- [x] **Bug found and fixed via live testing:** switching the navbar
      toggle to testnet kept showing the *mainnet* wallet's address, with
      no testnet history. Cause: `proofpay-wallet-session` in localStorage
      is a single cached value with no network dimension, and the app
      routes straight to the dashboard whenever it's present — regardless
      of which network is currently selected. Fixed in
      `components/Navbar.jsx`: switching networks now clears that cached
      session first (only for Circle wallets — MetaMask/Rabby use the same
      address on every network, so they're unaffected), forcing a fresh
      sign-in scoped to the newly selected network. Pushed; rebuilding now.
- [x] **Second bug found via the same re-test, also fixed:** after the
      session-clearing fix above, switching networks correctly showed
      "Connect Wallet" — but clicking it threw "We could not connect your
      wallet. Please unlock MetaMask or Rabby", even for a Circle-wallet
      user. Cause: `Home.jsx`'s `handleWalletButton` called
      `handleConnectWallet()` unconditionally whenever no address was
      present, which always drives the MetaMask/Rabby flow regardless of
      the stored wallet type — a pre-existing bug the first fix newly
      exposed (Circle sessions essentially never emptied out mid-use
      before today). Fixed: `handleWalletButton` now checks
      `isCircleWallet` and routes to `/login` (email/OTP) first, matching
      a check this same file's `onChangeWallet` callback already had a
      few lines down. Also added the missing `getWalletErrorMessage()`
      case for Circle's actual "session has expired" error, which had
      been falling through to the misleading MetaMask-flavored message.
      Pushed; rebuilding now.
- [x] **Re-tested after rebuild — confirmed working end to end.** Switched
      mainnet → testnet, dashboard showed "Connect Wallet", clicking it
      correctly went to email/OTP sign-in (not the MetaMask error), and
      after verifying the OTP the user's *original* testnet wallet came
      back (Circle recognized the existing testnet identity rather than
      creating a new one). All three fixes — session clearing, correct
      reconnect routing, and Circle's own wallet lookup — work together
      correctly.
- [x] Also replaced the native `window.confirm()` network-switch dialog
      (a full browser-centered popup, felt disconnected from the small
      nav badge that triggered it) with a small inline popover anchored
      right under the badge, styled to match `Home.jsx`'s existing wallet
      dropdown. Pushed.
- [x] **Audited for the same bug class elsewhere** (every `localStorage`
      key the frontend uses) after finding it twice today. Found one more:
      `proofpay-wallet` is a second cache of the Circle wallet address
      (`OtpVerification.jsx` sets it alongside `proofpay-wallet-session`;
      `proofpayContract.js`'s Circle-wallet functions read it directly)
      that the earlier fix didn't clear. Narrow edge case — only reachable
      via a deep link/bookmark that skips `SessionLanding`'s redirect —
      but cheap to close, so `Navbar.jsx` now clears it too. The other
      keys (`proofpay-wallet-type`, `-email`, `-circle-auth`,
      `-last-safe-route`, `-escrows`) are either wallet-type-agnostic or
      transient per-escrow form state, not identity — no fix needed there.
- [x] **Navbar UX pass, 2026-09-17** (all based on direct user feedback
      while testing the live site):
      - Replaced `window.confirm()`'s network-switch dialog with a direct
        network list (click badge → pick network → switched immediately,
        matching MetaMask's own pattern) instead of an extra confirm step
      - Removed the redundant "Network: ..." tile from Home.jsx's hero
        card (Navbar's badge already covers it) and hid the cirBTC stat
        card on mainnet (no cirBTC support there)
      - Moved the Wallet button out of Home.jsx's hero card into
        `Navbar.jsx` itself (`walletSlot` prop), positioned after the
        network badge (network first, wallet second — standard dApp
        order) — but a prop only Home.jsx passed, so it was invisible on
        the other 21 pages that render Navbar. Built
        `hooks/useWalletBadge.jsx`, a lighter self-contained version of
        Home's wallet button/dropdown, and wired it into all of them
        (Login.jsx/OtpVerification.jsx deliberately excluded — no wallet
        exists yet on the pages whose job is connecting one)
      - Added `hooks/useClickOutside.js`: both the network and wallet
        dropdowns only closed via their own toggle button; clicking
        anywhere else left them stuck open. Now closes on any outside
        click without changing the selection
      - Navbar was constrained to `max-w-6xl mx-auto` like page content,
        leaving a large empty gap between the badges and the real
        browser edge on wide screens. Made it full-width (just the side
        padding remains)
- [x] **App-wide network theme, 2026-09-17.** Blue retired as the brand
      accent everywhere, not just Home.jsx/Navbar: green on mainnet,
      amber on testnet. Given ~20+ pages use blue-* Tailwind utilities,
      hand-editing every className wasn't reliable — instead
      `main.jsx` sets `data-network` on `<html>` from
      `config/network.js`, and `index.css` adds `!important` overrides
      for every distinct blue-* utility actually used in the app
      (grepped the full set: bg/text/border/border-t/shadow/ring/
      gradient from+via, plus hover:/focus:/disabled:/file: variants),
      scoped to `[data-network="mainnet"]` → green or
      `[data-network="testnet"]` → amber CSS variables. Verified against
      the compiled CSS bundle that Tailwind v4's actual gradient
      variable names (`--tw-gradient-from`, `--tw-gradient-via`) match
      what the override sets, so the two remaining blue gradient hero
      cards (`WaitingSeller.jsx`, `CircleWallet.jsx`) are covered without
      editing those files directly. The Navbar logo was inlined as SVG
      (was a static blue `.svg` file via `<img>`, uncolorable by CSS) so
      its background square also switches with the theme.

  **🔴 That version shipped a production bug — every "blue" button/card
  went invisible.** The `--net-*` colors were defined as two CSS blocks,
  `[data-network="mainnet"] {...}` and `[data-network="testnet"] {...}`,
  with identical property names and different values. Tailwind v4's
  build (Lightning CSS) silently treated them as duplicate rules and
  dropped one — confirmed directly on the deployed bundle: zero
  occurrences of `data-network=mainnet` survived minification, only
  `testnet` did. Every override then resolved `var(--net-*)` to nothing,
  so `background-color` fell back to transparent — white text on a
  transparent button over a white page, effectively invisible. Caught by
  the user screenshotting the live site, not by anything on this end
  beforehand.

  **Fixed and re-verified properly this time:** the `--net-*` values are
  now set as plain inline styles on `<html>` from a `NETWORK_COLORS`
  lookup object in `main.jsx` (`style.setProperty` in a loop) instead of
  as CSS Lightning CSS could merge away; `index.css` keeps only the
  class-name override rules. Before pushing, built the bundle, served it
  with `vite preview` locally, and confirmed via `getComputedStyle` in
  the browser that `--net-600` resolves and buttons render solid
  green/amber — not just "the build succeeded," which is what was
  wrongly treated as sufficient the first time.

  **Lesson for any future app-wide CSS change here:** `npm run build`
  passing is necessary but **not sufficient** — it does not catch a
  minifier merging/dropping rules it considers duplicates. Load the
  actual built output (`vite preview` or equivalent) and check computed
  styles before pushing, every time, not just for this one theme change.
- [x] **Logo recolored too, 2026-09-17.** `landing/Hero.jsx` and
      `SellerAccept.jsx` still loaded the static blue
      `/proofpay-logo.svg` via `<img>` (Navbar's logo was inlined
      earlier, these two weren't). Extracted a shared
      `components/ProofPayLogo.jsx` so the three call sites can't drift
      out of sync again. Verified with a local `vite preview` screenshot
      before pushing.
- [ ] Try a real MetaMask/Rabby escrow on mainnet if you want the
      strongest possible confirmation (real money, optional)

**❌ Raised and settled: "same wallet address on both networks" is not
possible for Circle email-login wallets — this is a Circle platform limit,
not a bug.** Circle's own docs: *"API Keys are scoped to either Testnet or
Mainnet... you cannot use a single API key across both environments."*
Unified EVM addressing (one address, many chains) only works *within* one
environment; Mainnet and Testnet are separate environments with separate
user directories, so a Circle-hosted wallet is unavoidably different per
network. MetaMask/Rabby wallets do **not** have this limitation — same
address on every network, since that's plain Ethereum key behavior, not
Circle-mediated. Anyone who wants one consistent address across both
networks should use MetaMask/Rabby, not Circle email login.

**✅ CONFIRMED by Circle Customer Care, 2026-09-17** (support ticket reply):
*"Circle's Testnet (Sandbox) and Mainnet (Production) environments are
entirely separate, isolated systems. Each environment has its own
independent user registry, API keys, and cryptographic key-generation
infrastructure... there is no way to link them or force an identical
wallet address across Testnet and Mainnet."* Matches what the docs said
and what today's live testing showed — no longer just an inference from
docs, Circle's own support has now said it directly. Nothing to change;
the existing design (separate wallet per network, session cleared on
switch) is correct as-is.

- [x] **Real MetaMask/Rabby escrow on mainnet — done and successful**
      (2026-09-16, confirmed by the user 2026-09-17). Strongest possible
      end-to-end confirmation of the USDC escrow contract with real
      funds: deposit, contract interaction, and settlement all worked as
      designed on live Arc Mainnet.

Genuinely still open (no urgent deadline, but real):
- Independent smart-contract security audit (see step 3 — self-review
  found 2 HIGH findings, still needs a real audit)

## 1. Get official mainnet details from Circle docs

- [x] Arc mainnet RPC URL: `https://rpc.mainnet.arc.io` (Circle primary;
      Alchemy/Blockdaemon/dRPC/QuickNode alternates also listed in docs)
- [x] Arc mainnet chain ID: `5042`
- [x] Canonical USDC contract address on Arc mainnet:
      `0x3600000000000000000000000000000000000000` — USDC is the native gas
      asset with an optional ERC-20 interface (not a plain ERC-20 deploy)
- [x] Other assets confirmed on mainnet: EURC at
      `0xbEf5f6d51CB62b58e6A8f77868681825C6fe21c1`, plus USYC/CCTP/Gateway/
      StableFX contracts. No cirBTC-equivalent found — confirms it was
      testnet-only.
- [x] Arc mainnet block explorer: `explorer.arc.io` (docs note permissioned
      access, unlike the open testnet.arcscan.app)

  **⚠️ Verify before using in any deploy:** these values were pulled via an
  automated fetch of docs.arc.io/arc/references/{rpc-endpoints,contract-addresses}
  on 2026-09-16. Open that page in a browser and check every character
  against `ARC_MAINNET_USDC` and any other address before it goes into a
  deploy script — a wrong digit here moves real funds to a dead address.

## 2. Contracts — DONE, deployed 2026-09-16

**Note:** this actually happened in a separate repo/working copy,
`~/proof/ProofPay-Mainnet` (pushed to
`github.com/24hlivepay/ProofPay-Mainnet`), not in this repo's
`foundry-mainnet/`. That repo's `foundry/` is now the source of truth for
the mainnet contracts. This repo's `foundry-mainnet/` was the original
scaffold/placeholder and was never actually used for the real deploy.

- [x] Deployer keystore: `proofpay-deployer` (same key reused from testnet —
      not a separate mainnet key as originally planned; noted, not blocking)
- [x] Deployer wallet: `0xD979e5D9EEB1126C75a7B215Ee0f79895Fe091AC`
- [x] Deploy script addresses filled in and verified against docs.arc.io
      before broadcasting
- [x] Dry-run simulated successfully before the real broadcast
- [x] **USDC escrow deployed:** `0x626B2731A11B39A782992B57ED102012b607BC79`
      — tx `0x171a47b74aa8cda0369ac240afb6374bb014d6ee6d37d13d34f6efeb9589aa82`,
      block 21188708. Confirmed on-chain: `usdc()` returns the correct USDC
      address, `owner()` returns the deployer wallet.
- [x] **EURC escrow deployed:** `0xF6f0178e40dbF82D79e7E90a9b07AB0f32b862C0`
      — tx `0x6e3fe0bad04c9ec3fffa78000a5051b077864580d5cfc5b9e901cb5067cb4dce`,
      block 21188797. Confirmed on-chain: linked token matches EURC address.
- [x] cirBTC escrow correctly skipped — no mainnet contract published yet
- [x] **Verified on the mainnet block explorer, 2026-09-17.**
      `explorer.arc.io` turned out to be publicly accessible — no
      permission wall, contrary to what the docs implied. Checked both:
      - USDC escrow `0x626B...BC79`: creator `0xD9...91AC` (deployer) ✓,
        creation tx `0x17...aa82` matches ✓, 7 transactions / 4 token
        transfers already recorded (from today's live testing)
      - EURC escrow `0xF6f0...62C0`: creator `0xD9...91AC` ✓, creation tx
        `0x6e...4dce` matches ✓, 1 transaction (the deploy itself)
- Total gas paid: ~0.1120 USDC across both deploys. Deployer balance after:
  **~9.884 USDC**.

## 3. Security (do before real funds touch it)

- [ ] Independent smart-contract security audit — the README already flags
      this as outstanding for the testnet MVP; it's a hard requirement
      before mainnet, since mainnet holds real money. **Not done — the
      self-review below is not a substitute.**
- [x] Self-reviewed `foundry/src/ProofPayEscrow.sol` 2026-09-16 (the exact
      contract now live at both mainnet escrow addresses). Reentrancy guard
      and checks-effects-interactions are correctly used everywhere funds
      move. Findings, most severe first:

  1. **HIGH — `owner` is `immutable`; there is no `transferOwnership()`.**
     It is set once at deploy time to the deployer EOA and can never be
     rotated on this contract. The still-open "does `resolveDispute` need a
     multisig?" question **cannot be answered by upgrading this deployment**
     — the only way to put a multisig in control is to deploy a *new*
     escrow contract with the multisig as constructor owner and migrate
     escrows to it. Deciding this later gets more expensive, not cheaper.
  2. **HIGH — a malicious/compromised owner key can only pay out to the two
     addresses on a given escrow, but that is enough to steal.** A buyer
     who also holds (or colludes with) the owner key can open an escrow,
     receive delivery, open a dispute, then self-resolve 100% back to
     themselves as buyer — a free refund after real delivery. This is
     purely a function of how well `0xD979e5D9EEB1126C75a7B215Ee0f79895Fe091AC`
     (the deployer/owner) is secured.
  3. **MEDIUM — `resolveDispute` makes two transfers in one atomic call.**
     If either the buyer's or seller's address is ever blacklisted at the
     token level (Circle can blacklist USDC/EURC addresses), the whole call
     reverts and **both** parties' funds are stuck — there is no
     pull-payment fallback or per-party withdrawal path. Rare, but
     unrecoverable if it happens since there is no other way to resolve
     that escrow.
  4. Minor: no fee-on-transfer/rebasing token protection (accounting trusts
     `amount` rather than a measured balance delta). Not exploitable with
     standard USDC/EURC, just don't reuse this contract for a token type
     that behaves differently.

## 4. Frontend (`frontend/`) — DONE for the MetaMask/Rabby wallet path, 2026-09-16

**Note:** this happened in `~/proof/ProofPay-Mainnet` (see step 2's note),
not in this repo.

Built a dual-network toggle rather than a one-way mainnet switch (this repo's
`frontend/src/config/network.js` is new — `NETWORKS.mainnet` /
`NETWORKS.testnet`, `getCurrentNetworkId`/`setCurrentNetworkId` persisted in
`localStorage["proofpay-network"]`, default **mainnet** unless
`VITE_DEFAULT_NETWORK=testnet` is set). `escrowAssets.js` now keys
`ESCROW_ASSETS` per network (mainnet has no `cirBTC` entry — asking for it
throws, matching step 1's finding that it has no mainnet contract).

- [x] `frontend/src/services/wallet.js` — chain add/switch now reads the
      active network's chainId/RPC instead of a hardcoded testnet object
- [x] `frontend/src/services/proofpayContract.js` — RPC URL, default escrow/
      token addresses, and the per-asset event-log start block are now
      derived from the active network; user-facing "Arc Testnet" strings
      are dynamic
- [x] All 10 originally-listed pages, plus 3 more found by grep that also
      hardcoded "Arc Testnet"/arcscan (`Home.jsx`, `landing/Hero.jsx`,
      `OtpVerification.jsx`) and `CreateEscrow.jsx` (asset dropdown)
- [x] Mainnet-specific copy fixes: the testnet "these are test tokens, no
      financial value" line now only shows on testnet (real USDC on mainnet
      otherwise), the Circle faucet button/link is hidden on mainnet, and
      `getNetworkName()`'s wallet chain-id map now recognizes `0x13b2` (Arc
      Mainnet) instead of showing "Unsupported Network"
- [x] `npm run build` and `npm run lint` both pass clean after the change
- [x] Added a visible toggle: a small network badge in `Navbar.jsx` that
      switches `proofpay-network` (with a confirm dialog) and reloads

**✅ Confirmed the actual money-moving path is safe on mainnet.** The
MetaMask/Rabby flow never reads contract addresses from the backend — it
always resolves them locally via `getEscrowAsset(assetSymbol)`, which now
correctly points at mainnet. Verified with `grep` that no frontend file
reads `escrow.escrowContractAddress` / `escrow.tokenAddress` from an API
response.

**✅ RESOLVED 2026-09-16 — backend is now network-aware end to end.**
Originally found: escrow records had no `network` field anywhere, so
testnet test data and real mainnet data were mixed in the same storage and
counted together in `/api/escrow-stats`, `/api/admin/disputes`, and
`/api/escrows`. Fixed:

- `frontend/src/services/api.js` sends `X-ProofPay-Network: mainnet|testnet`
  (from `config/network.js`) as a header on every backend request, via an
  axios interceptor — no per-call-site changes needed anywhere else in the
  frontend.
- `backend/server.js` adds `getRequestNetwork(req)` (reads that header,
  defaults to **mainnet** if missing/invalid) and `escrowNetwork(escrow)`
  (reads a stored escrow's `network` field, defaults to **testnet** for any
  record from before this change — old data is never assumed to be
  real-money data).
- `POST /api/escrow` now stamps `network` on every new record and pulls
  addresses from a per-network `ESCROW_ASSETS_BY_NETWORK` map (mainnet: the
  two verified deploy addresses from step 2, no cirBTC entry; testnet:
  unchanged). `CIRCLE_CONTRACT_ALLOWLIST` is now built per network from the
  same source instead of being hand-duplicated.
- `/api/escrows`, `/api/escrow-stats`, `/api/admin/disputes` all filter by
  `escrowNetwork(escrow) === getRequestNetwork(req)` before returning
  anything — mainnet and testnet data can no longer mix.
- `/api/health`'s `network` field is no longer hardcoded to "Arc Testnet".
- ID-based routes (`/api/escrow/:id`, accept/deposit/release/dispute/etc.)
  were deliberately **not** given a network guard — escrow IDs are unique
  regardless of network, so there is no cross-network collision risk there,
  and adding a redundant check to all ~13 of them wasn't worth the risk of
  a copy-paste mistake for no real safety gain.
- **Verified by running the server locally** (dummy `CIRCLE_API_KEY`, a
  scratch `DATA_DIR` — did not touch real data) and curling it: mainnet vs.
  testnet escrow creation, listing, and stats are correctly isolated;
  mainnet `cirBTC` is correctly rejected; `node --check` and the real
  server boot both passed clean.

**✅ RESOLVED 2026-09-16 — Circle-hosted (email-login) wallets now work on
mainnet too.** Confirmed the mainnet blockchain identifier directly from
Circle's official docs (developers.circle.com/wallets → Build onchain →
Supported blockchains table): Arc's mainnet/testnet chain codes are
**`ARC`** / `ARC-TESTNET` — not the `ARC-MAINNET` this file originally
guessed at and correctly refused to hardcode. Both `server.js`'s
`CIRCLE_BLOCKCHAIN_BY_NETWORK` and the frontend's `network.js`
`circleBlockchain` field now default to `"ARC"` for mainnet
(`CIRCLE_MAINNET_BLOCKCHAIN` env var can still override it). Verified live:
`/api/circle/wallets` with a mainnet header now reaches Circle's real API
with `blockchain=ARC` instead of 501ing (only fails on the dummy test API
key used for the check, as expected).

Also still open — Vercel production env vars, only needed if the hardcoded
fallback addresses in `escrowAssets.js` are ever rotated:

- [ ] `VITE_MAINNET_USDC_ESCROW_ADDRESS`, `VITE_MAINNET_EURC_ESCROW_ADDRESS`

## 5. Circle Developer Console

- [x] Confirmed the Circle Wallets API blockchain identifier for Arc
      Mainnet is `"ARC"` (developers.circle.com/wallets docs) — now the
      default in code, no env var needed (see step 4's backend section)
- [x] **Billing blocker (found and resolved 2026-09-16).** Circle gated
      Mainnet Wallets/Contracts behind "Upgrade to unlock this feature" — a
      card had to be added before Mainnet access was granted (pricing:
      first 1,000 active wallets/month free, then $0.05/wallet up to 5,000;
      first 25,000 API calls/month free, then $0.0005/call — ProofPay's
      early-stage volume should stay in the free tier). Card was added by
      the account owner; Mainnet is now fully unlocked.
- [x] Configured Wallets → User Controlled → Configurator → Authentication
      Methods → Email on **Mainnet** to match the existing testnet setup:
      SMTP via Resend (`noreply@proofpay.online`, `smtp.resend.com:465`,
      same pattern as testnet, using a separately-noted API key rather than
      reusing testnet's, since Resend only shows a key's value once), and
      the same OTP subject line. Confirmed with a real test email.
- [x] Checked Wallet Security Settings on Mainnet — "Confirmation UIs" is
      on by default there too, matching testnet; no change needed.
- [x] **Caught before it shipped:** the backend had a single global
      `CIRCLE_API_KEY`, used for every Circle call regardless of network.
      Setting it to the Mainnet key would have silently broken every
      testnet Circle-wallet call (and vice versa) — same mistake class as
      the blockchain identifier, just one step later. Fixed the same way:
      `CIRCLE_API_KEY_BY_NETWORK` in `server.js` maps testnet →
      `process.env.CIRCLE_API_KEY` (unchanged, already set in Vercel) and
      mainnet → `process.env.CIRCLE_API_KEY_MAINNET` (new). All 9 routes
      that called Circle now build their auth header via
      `getCircleHeaders(getRequestNetwork(req))` instead of one shared
      object. Verified locally: testnet and mainnet requests each reach
      Circle with their own key, no cross-contamination.
- [x] Mainnet Circle keys created in Console (Standard API Key named
      "ProofPay Backend", matching testnet's naming; Client Keys page is
      unused by ProofPay's code — skipped it). App ID was auto-created the
      first time Mainnet was opened in Configurator.
- [x] **Same class of bug, found again on the frontend:** `VITE_CIRCLE_APP_ID`
      is a Vite build-time env var — a single value baked into the deployed
      bundle — but which network is active is a *runtime* choice (the
      Navbar toggle). One App ID could never serve both networks correctly.
      Fixed in `config/network.js`: each network now has its own
      `circleAppId`, read from `VITE_CIRCLE_APP_ID_MAINNET` /
      `VITE_CIRCLE_APP_ID_TESTNET` (both baked into the same build; the
      toggle's page reload picks the right one at runtime).
      `VITE_CIRCLE_APP_ID` (no suffix) still works as a fallback for
      testnet only, so the existing Vercel config isn't broken by this.
- [x] **Done 2026-09-17.** Vercel's `proof-pay` project (proofpay.online)
      was still Git-connected to the old `ProofPay-Testnet` repo — switched
      it to `ProofPay-Mainnet` under Settings → Git (had to grant the
      Vercel GitHub App access to the new repo first, under
      github.com/settings/installations, since it was scoped to selected
      repos). Connecting a repo doesn't retroactively deploy its history,
      so an empty commit was pushed to trigger the first real build.
      `CIRCLE_API_KEY_MAINNET` (Secret) and `VITE_CIRCLE_APP_ID_MAINNET`
      (Config — Vercel rejects `VITE_`-prefixed vars as Secret since
      they're public in the client bundle anyway) are both set in
      Production. One more empty commit pushed to rebuild with them
      included, since adding an env var doesn't touch existing builds.
      **Note:** the Mainnet API key was briefly visible in a screenshot
      during setup — it was revoked and regenerated before being saved to
      Vercel, so the exposed value was never actually put into use.

## 6. Decide: one app for both networks, or separate deployments? — RESOLVED

Went with **one app, network toggle** (`frontend/src/config/network.js`),
not a separate deployment. Default is mainnet; testnet stays reachable via
`VITE_DEFAULT_NETWORK=testnet` or setting `localStorage["proofpay-network"]`
until a UI toggle exists (see step 4).

## Reference

- Arc mainnet public launch: September 16, 2026
  (https://www.circle.com/pressroom/circle-announces-founding-validator-cohort-and-major-integrations-for-arc-ahead-of-september-16-mainnet-launch)
- Arc is EVM-compatible (Reth execution layer), gas denominated in USDC
