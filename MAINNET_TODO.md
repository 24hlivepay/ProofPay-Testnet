# Arc mainnet launch checklist

Arc mainnet went live **September 16, 2026**. Step 1 details are now
published — see below. Everything after step 1 still needs doing.

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
- [ ] Verify the deployed contracts on the mainnet block explorer
      (`explorer.arc.io` — docs note permissioned access; not yet confirmed
      this deployer/account can view it)
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
- [ ] Still open: set these two in Vercel's frontend env —
      `VITE_CIRCLE_APP_ID_TESTNET` (same value as the current
      `VITE_CIRCLE_APP_ID`, or leave that var as-is and skip this one) and
      `VITE_CIRCLE_APP_ID_MAINNET` (the Mainnet App ID) — and this one in
      the backend env: `CIRCLE_API_KEY_MAINNET` (the new Mainnet API key,
      **not** the existing `CIRCLE_API_KEY`, which stays pointed at
      testnet). Until these are set, mainnet Circle-wallet calls fail
      cleanly at Circle's end (empty/wrong key) even though all the
      app-side wiring is done.

## 6. Decide: one app for both networks, or separate deployments? — RESOLVED

Went with **one app, network toggle** (`frontend/src/config/network.js`),
not a separate deployment. Default is mainnet; testnet stays reachable via
`VITE_DEFAULT_NETWORK=testnet` or setting `localStorage["proofpay-network"]`
until a UI toggle exists (see step 4).

## Reference

- Arc mainnet public launch: September 16, 2026
  (https://www.circle.com/pressroom/circle-announces-founding-validator-cohort-and-major-integrations-for-arc-ahead-of-september-16-mainnet-launch)
- Arc is EVM-compatible (Reth execution layer), gas denominated in USDC
