# ProofPay contracts registry

Single place for every ProofPay escrow contract, old and new: addresses, deploy
transactions, what changed, what is live, and what is still open. Keep this file
updated whenever a contract is deployed, retired or its config changes.
Last updated: 2026-09-26.

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
| Mainnet | USDC | `0x626B2731A11B39A782992B57ED102012b607BC79` | `0x171a47b74aa8cda0369ac240afb6374bb014d6ee6d37d13d34f6efeb9589aa82` | 21188708 | LIVE, in use by the app |
| Mainnet | EURC | `0xF6f0178e40dbF82D79e7E90a9b07AB0f32b862C0` | `0x6e3fe0bad04c9ec3fffa78000a5051b077864580d5cfc5b9e901cb5067cb4dce` | 21188797 | LIVE, in use by the app |
| Testnet | USDC | `0xCd0f43E573899809ff96C560439570A760698C9a` | `0x79e8933c8df6707c0f5a91fc3f0e162f270100eb4514994d6d8536901dfe3f73` | 53590676 | in use until V2 is wired in |
| Testnet | EURC | `0xa4322D8ba3E040A3028FD6ABaC3c6a5625ed4ca7` | not recorded here | n/a | in use until V2 is wired in |
| Testnet | cirBTC | `0x8bfeD6F70Eb595946543b192b6E63d75A0bBEf4B` | not recorded here | n/a | in use until V2 is wired in |

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
| Testnet | USDC | `0xbf28D1d4cb480DDAc52c23670aFECA94D4d719a1` | `0xe89ec93681f74ed0e0ffb7fe6368981c106251c7fa0110aa289420f9d299e7e5` | 64079701 | DEPLOYED 2026-09-26, not yet wired into the app |
| Testnet | EURC | `0x7117B300A01C969082DE898F1B1f699F6e8188B3` | `0x5d6aec739ba0f8e0d3fcae397eed1ed71737a4b64e452ec3dd68c9fbaedb83fb` | 64079701 | DEPLOYED 2026-09-26, not yet wired into the app |
| Mainnet | USDC | not deployed | | | waits for testnet flow test + audit |
| Mainnet | EURC | not deployed | | | waits for testnet flow test + audit |

Deployed with `foundry/script/DeployV2Escrows.s.sol` from the deployer wallet
(gas paid 0.1433 USDC for both). On-chain check 2026-09-26, both testnet contracts:
code present, `owner()` = the deployer, `pendingOwner()` = zero address,
`usdc()` = the right token (USDC `0x3600…0000` / EURC `0x89B5…D72a`),
`paused()` = false, and a call to `refund(string)` reverts (the function does not exist).
Explorer source verification: NOT verified yet (checked, `is_verified` false).

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

- App on mainnet: v1 USDC and v1 EURC.
- App on testnet: v1 USDC, EURC and cirBTC.
- V2 exists on testnet only, unwired.

## Next steps, in order (all need the owner)

1. Wire the V2 testnet addresses into the app config on a branch (testnet only) and test the full flow on the Vercel preview.
2. Independent audit of V2.
3. Deploy V2 on mainnet with the same script (dry run first), verify on the explorer, update this file.
4. Point new mainnet escrows at V2. Escrows already funded on v1 finish on v1.
5. Decide the owner: keep the admin EOA, or a multisig after the backend change.
6. Optional: fix the blacklist stuck-funds issue in a later version.
