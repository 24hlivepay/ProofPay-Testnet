# Arc mainnet launch checklist

Arc mainnet went live **September 16, 2026**. Step 1 details are now
published — see below. Everything after step 1 still needs doing.

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
