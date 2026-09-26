# ProofPay contracts registry

Single place for every ProofPay escrow contract, old and new: addresses, deploy
transactions, what changed, what is live, and what is still open. Keep this file
updated whenever a contract is deployed, retired or its config changes.
Last updated: 2026-09-26 (evening).

Related files: `MAINNET_TODO.md` (running log of decisions and bug fixes),
code repo `~/proof/ProofPay-Mainnet` (`foundry/src`, `foundry/test`,
`foundry/script`).

## The rule every contract must follow

Once the buyer has deposited, **neither the buyer nor the seller can take the
money out alone.** It leaves the contract in only two ways:

1. The seller calls `confirmDelivery`, then the buyer calls `releaseFunds` (money goes to the seller).
2. Either side calls `openDispute`. The funds freeze, buyer and seller can do
   nothing more, and only the admin (owner) calls `resolveDispute` to decide:
   full refund to the buyer, full payment to the seller, or any split.

There must be no buyer-side `refund()` or any other way to pull funds back
without a dispute. The v1 contracts below break this rule (see "v1 known issue").

## One contract per token

A contract holds ONE token, set in its constructor (the variable is called
`usdc` in the code but takes any ERC-20). So each version is deployed once per
token: one USDC escrow and one EURC escrow (cirBTC too on testnet).

## Networks

| | Arc Mainnet | Arc Testnet |
|---|---|---|
| Chain ID | 5042 | 5042002 |
| RPC | https://rpc.mainnet.arc.io | https://rpc.testnet.arc.network |
| Explorer | https://explorer.arc.io | https://testnet.arcscan.app |
| Gas | native USDC (18 decimals) | native USDC (18 decimals) |

Deployer and current owner of every contract below (an EOA, also the backend's
`DISPUTE_ADMIN_WALLET`): `0xD979e5D9EEB1126C75a7B215Ee0f79895Fe091AC`.
Foundry keystore name on the laptop: `proofpay-deployer` (a `.backup` copy exists).
Balances read 2026-09-26: mainnet 5.83 USDC, testnet 151.87 USDC.

## Tokens (ERC-20 interface, 6 decimals)

| Token | Mainnet | Testnet |
|---|---|---|
| USDC | `0x3600000000000000000000000000000000000000` | `0x3600000000000000000000000000000000000000` |
| EURC | `0xbEf5f6d51CB62b58e6A8f77868681825C6fe21c1` | `0x89B50855Aa3bE2F677cD6303Cec089B5F319D72a` |
| cirBTC (8 decimals) | none published for mainnet as of 2026-09-26 (an Interop on Arc announcement of 2026-09-23 says cirBTC is on Arc: re-check docs.arc.io before assuming) | `0xf0C4a4CE82A5746AbAAd9425360Ab04fbBA432BF` |

## Contract registry

### v1: `ProofPayEscrow` (source `foundry/src/ProofPayEscrow.sol`, do NOT edit)

| Network | Token | Address | Deploy tx | Block | Status |
|---|---|---|---|---|---|
| Mainnet | USDC | `0x626B2731A11B39A782992B57ED102012b607BC79` | `0x171a47b74aa8cda0369ac240afb6374bb014d6ee6d37d13d34f6efeb9589aa82` | 21188708 | RETIRED and PAUSED (2026-09-26, block 22880961, tx `0x94986e62db41c6c98bcdae0bdebb62fe3955f1c26ffbbfce16ecb9e36a7c0ac0`). Held 0 tokens |
| Mainnet | EURC | `0xF6f0178e40dbF82D79e7E90a9b07AB0f32b862C0` | `0x6e3fe0bad04c9ec3fffa78000a5051b077864580d5cfc5b9e901cb5067cb4dce` | 21188797 | RETIRED and PAUSED (2026-09-26, block 22880961, tx `0x0d94ddec105d71f00faf4b3cfd03dfa3bbdf8c624d4e8bdae2d5d315957f8428`). Held 0 tokens |
| Testnet | USDC | `0xCd0f43E573899809ff96C560439570A760698C9a` | `0x79e8933c8df6707c0f5a91fc3f0e162f270100eb4514994d6d8536901dfe3f73` | 53590676 | retired from the app by PR #21 (after merge). No `pause()` exists on it |
| Testnet | EURC | `0xa4322D8ba3E040A3028FD6ABaC3c6a5625ed4ca7` | not recorded here | n/a | retired from the app by PR #21 (after merge). No `pause()` exists on it |
| Testnet | cirBTC | `0x8bfeD6F70Eb595946543b192b6E63d75A0bBEf4B` | not recorded here | n/a | retired from the app by PR #21 (after merge). No `pause()` exists on it |

On-chain check 2026-09-26: mainnet USDC and EURC v1 both return the expected
token from `usdc()`, `owner()` is the deployer above, `paused()` is false.
Testnet v1 contracts also return the expected tokens and the same owner.

**v1 known issue (cannot be fixed on these contracts, they are immutable):**
- `refund()` lets the buyer alone take the deposit back, until the seller calls
  `confirmDelivery`. It came in with the original testnet prototype (commit
  `eb7add6`, 2026-07-26), was documented as a "pre-delivery refund" feature and
  was missed by the 2026-09-16 self-review. The user's rule was always that
  nobody can refund themselves. The app never offered it and the app code that
  could call it is removed in PR #19, but a MetaMask/Rabby buyer can still call
  the contract directly.
- `owner` is `immutable`: the dispute admin can never be rotated or moved to a multisig.
- `resolveDispute` sends to both parties in one call, so if either address is
  blacklisted on USDC/EURC the call reverts and both parties' funds are stuck.
- Independent audit: not done. Only a self-review.

### V2: `ProofPayEscrowV2` (source `foundry/src/ProofPayEscrowV2.sol`, draft PR #20)

Exactly v1 with two changes:
1. `refund()` and the unused `FundsRefunded` event are removed.
2. `owner` is transferable with a two-step `transferOwnership` then `acceptOwnership` (the old owner stays in control until the new one accepts, so a wrong address cannot lock the role).

`Status` numbers are unchanged (None 0, Funded 1, Delivered 2, Released 3, Refunded 4, Disputed 5), so the backend and frontend mappings still hold. `Refunded` is now only reachable through `resolveDispute` giving the buyer the full amount.

| Network | Token | Address | Deploy tx | Block | Status |
|---|---|---|---|---|---|
| Testnet | USDC | `0xbf28D1d4cb480DDAc52c23670aFECA94D4d719a1` | `0xe89ec93681f74ed0e0ffb7fe6368981c106251c7fa0110aa289420f9d299e7e5` | 64079701 | DEPLOYED 2026-09-26. Wired into the app by PR #21 (open, not merged) |
| Testnet | EURC | `0x7117B300A01C969082DE898F1B1f699F6e8188B3` | `0x5d6aec739ba0f8e0d3fcae397eed1ed71737a4b64e452ec3dd68c9fbaedb83fb` | 64079701 | DEPLOYED 2026-09-26. Wired into the app by PR #21 (open, not merged) |
| Mainnet | USDC | `0xbA8cf9bE18DE912dC98a6422906b1D8F0e56F76B` | `0x145e96c4ba7857404a56a57d7a3a1c31ede58101055224d97598d8c8b3f994e4` | 22880209 | DEPLOYED 2026-09-26 by the owner. LIVE in the app since PR #31 (merged 2026-09-26) |
| Mainnet | EURC | `0x7894E539a16b0D1aE272BE4ebF998353C6E15C86` | `0x5e7ff5b11194d10616e9a4da869197ddea39c38393b76c9b28295c7545daef2a` | 22880209 | DEPLOYED 2026-09-26 by the owner. LIVE in the app since PR #31 (merged 2026-09-26) |

Deployed with `foundry/script/DeployV2Escrows.s.sol` from the deployer wallet
(gas paid 0.1433 USDC for both). On-chain check 2026-09-26, both testnet contracts:
code present, `owner()` = the deployer, `pendingOwner()` = zero address,
`usdc()` = the right token (USDC `0x3600…0000` / EURC `0x89B5…D72a`),
`paused()` = false, and a call to `refund(string)` reverts (the function does not exist).
Explorer source verification: DONE 2026-09-26, see below.

Mainnet V2 (2026-09-26): deployed with the same script from the deployer wallet, total paid ~0.0997
native USDC. Read-only checks on chain: code identical to the tested source apart from the token-address
slot, `owner()` = the deployer `0xD979...91AC`, `pendingOwner()` = zero, right token (USDC `0x3600...0000` /
EURC `0xbEf5...21c1`), not paused, `refund(string)` does not exist. Explorer source verification: DONE 2026-09-26, see below. The v1 mainnet escrows held 0 tokens on 2026-09-26.

**Explorer source verification (2026-09-26): all four V2 contracts show "exact match"** on the Blockscout explorers
(explorer.arc.io and testnet.arcscan.app), file path `src/ProofPayEscrowV2.sol`, solc `v0.8.28+commit.7893614a`,
EVM `prague`, optimizer off. Mainnet USDC was verified by hand with the Standard JSON input (single-file form only gave a
"partial match" because it renames the file, which changes the metadata hash); the other three were then matched
automatically by the Blockscout Bytecode Database. To repeat: `forge verify-contract <addr> src/ProofPayEscrowV2.sol:ProofPayEscrowV2 --show-standard-json-input`
in `foundry/`, upload that JSON. On the explorer every contract is named `ProofPayEscrowV2`; the token is only visible
as constructor argument `usdcAddress` (or via `usdc()` in Read contract). The source header still says "NOT YET DEPLOYED"; do not edit it, that would break the exact match.

Tests: `forge test` in `foundry/` = 59 pass (37 in `ProofPayEscrowV2.t.sol`
including that `refund` does not exist, that buyer and seller cannot move funds
while Funded, Delivered or Disputed, that only the owner resolves, a fuzz test
that the locked amount is fully accounted for, and the ownership handover; 2 for
the deploy script; 20 older). A control run showed the same refund call succeeds on v1.

## v1 vs V2 at a glance

| | v1 | V2 |
|---|---|---|
| Buyer can refund alone before delivery | yes (`refund()`) | no |
| Money leaves only via release or dispute | no | yes |
| Owner can be changed | no (immutable) | yes, two-step |
| Dispute split (any amount) | yes | yes |
| Pause blocks only new escrows | yes | yes |
| Status numbers | 0..5 | same 0..5 |
| Blacklist stuck-funds issue | yes | yes (not fixed on purpose) |
| Audited | no | no |

## Where the app reads contract addresses (update when switching)

- Backend: `backend/server.js`, `ESCROW_ASSETS_BY_NETWORK` (per network and token).
  Each escrow record also stores the contract address it was created on, which
  the resolution check uses (`tx.to` must equal it).
- Frontend: `frontend/src/services/proofpayContract.js` (`getEscrowAsset`) and
  `frontend/src/config/network.js`; the README contract table lists testnet addresses.
- Backend rule for the owner: `resolveDispute` transactions must be sent from
  `DISPUTE_ADMIN_WALLET`, and admin login is a wallet signature. So the V2 owner
  must be that same EOA. Moving to a multisig needs a backend change first.
- Still to check before mainnet: whether the frontend routes existing escrows
  to the contract they were created on, or always to the address in the config.
  Escrows already funded on v1 have to finish on v1.

## What is live today (2026-09-26)

- App on mainnet: **V2 USDC and V2 EURC** (PR #31). The v1 mainnet escrows are retired and paused and held 0 tokens.
- App on testnet: **V2 USDC and V2 EURC** (PR #21 merged). cirBTC is no longer offered on testnet
  (its v1 escrow is retired, there is no V2 cirBTC escrow).
- V2 exists on both networks (2 contracts each), all four verified on the explorers (exact match).

## Retiring the v1 contracts

- **Testnet v1** (USDC, EURC, cirBTC): they have NO `pause()` (deployed before
  it was added), so they cannot be paused. Retiring them = removing them from the
  app config, which PR #21 does. Escrows created on them can no longer be
  operated from the app (the frontend always uses the configured address).
- **Mainnet v1** (USDC, EURC): they do have `pause()`. `foundry/script/PauseV1Escrows.s.sol`
  (mainnet only, dry run passed, PR #20) pauses both so nothing new can be created
  on them. Pausing does not touch existing escrows. Needs the owner's signature.
  Do it when V2 goes live on mainnet, in the same window.

## Admin pause / resume in the dashboard (PR #21)

New "Contracts" tab, URL `<site>/#/admin/contracts` (the app uses a hash router).
Per escrow contract of the current network: address, on-chain owner, paused or
not (live from the chain), and a Pause / Resume button. It sends `pause()` /
`unpause()` from the admin's MetaMask/Rabby wallet, only if that wallet is the
contract's on-chain owner; Circle email wallets are refused with a message.
Pause blocks NEW escrows only. Works on v1 mainnet too (same functions).
Not written to the admin audit log (the on-chain transaction is the record).
The pause transaction itself has not been tested yet: it needs the admin wallet.

## Next steps, in order (all need the owner)

0. DONE 2026-09-26: mainnet V2 deployed, PR #31 merged (app uses it), v1 mainnet escrows paused. Any
   mainnet deal created on v1 but not funded before the switch must be created again.

1. PR #21 (wires V2 into testnet + admin Pause/Resume tab): test the full flow and the pause/resume on its Vercel preview, then merge.
2. Independent audit of V2.
3. Deploy V2 on mainnet with the same script (dry run first), verify on the explorer, update this file.
4. Point new mainnet escrows at V2. Escrows already funded on v1 finish on v1.
5. Decide the owner: keep the admin EOA, or a multisig after the backend change.
6. Optional: fix the blacklist stuck-funds issue in a later version.
