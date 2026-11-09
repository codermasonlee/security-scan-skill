# Vulnerability Checklist — Full Taxonomy

Sweep every applicable category. Each item: the code smell → what to check.
This file is deliberately exhaustive; precision comes from the kill gates, not
from thinning coverage here.

## A. Reentrancy & callbacks

- **A1 Single-function reentrancy** — external call (`.call`, transfer, token
  hook) before state finalization. Check effects-interactions ordering on
  every value-exit function.
- **A2 Cross-function reentrancy** — call in `f()` reenters `g()` sharing the
  same state (stakedBalance/claim paths). Map state shared across functions.
- **A3 Cross-contract reentrancy** — attacker reenters a *different* protocol
  sharing the same token/lp position.
- **A4 Read-only reentrancy** — view functions (price/totalAssets/accounting)
  return stale mid-transaction state during a callback; downstream protocols
  consuming the view are the victim. Check ERC-4626 `totalAssets`,
  exchange-rate getters, pool price getters called from hooks.
- **A5 Token hooks** — ERC-777 `tokensReceived`, ERC-721/1155
  `onERC*Received`, ERC-20 hooks (EIP-777-style), Uniswap swap callbacks:
  attacker-controlled code runs inside your function.
- **A6 Compiler-level lock bypass** — hand-written reentrancy guards with
  broken logic; Vyper 0.2.15/0.3.0 buggy `@nonreentrant` (Curve 2023-07-30).
- **A7 Reentrancy lock scope** — lock held on one contract but state mutated
  in another; lock not covering view path that matters.

## B. Oracles & prices

- **B1 Spot AMM price manipulation** — price derived from `balanceOf(pair)`
  or reserves read at current block; attacker flashes liquidity. Require
  TWAP or external feed.
- **B2 TWAP misuse** — window too short vs attack cost; TWAP taken over a
  manipulable pool; arithmetic-average-time misused as if time-weighted.
- **B3 Staleness / freshness** — missing `updatedAt` checks on Chainlink-style
  feeds; L2 `sequencerUptimeFeed` missing; grace periods wrong.
- **B4 Operator/setter control** — admin can set any price; feed address
  settable post-deploy without timelock.
- **B5 Decimal mismatch** — 1e6/1e8/1e18 mixing; normalization applied to
  wrong side; `10 ** (18 - feedDecimals)` underflow for 18-decimal feed.
- **B6 Single-block manipulation of curves** — stableswap invariant corner
  cases, imbalanced-liquidity price extremes.
- **B7 Exchange-rate staleness** — ERC-4626/vault share price cached; Bearer/
  PT/YT price feeds (Pendle-style) manipulated via their underlying AMM.
- **B8 Fee/pricing of illiquid collateral** — thin-liquidity collateral priced
  at AMM; mark vs index divergence in perps.

## C. Access control & authorization

- **C1 Missing modifiers** — state-changing externals with no auth; especially
  `sweep`/`rescue`/`set*`/`pause`/`upgrade`/`withdraw*`.
- **C2 tx.origin auth** — phishing-compatible.
- **C3 Role escalation** — role granting role; initializer callable twice;
  `onlyProxy` missing on implementation.
- **C4 Initializer front-running** — unproxied or unsealed initializers on
  deployment; `initialize` re-runnable after upgrade.
- **C5 Parameter abuse by permitted roles** — keeper/strategies that can set
  arbitrary swap targets → arbitrary `transferFrom` (approve-drain via
  router/callback params).
- **C6 Signature auth** — missing nonce (replay), missing chain-id
  (cross-chain replay), missing deadline, missing `address(this)` in domain
  separator, ecrecover(0) on malformed sig, signature malleability (s high/low).
- **C7 Merkle claims** — leaf hashes ambiguous (double-hash vs single),
  index reuse, proof arrays not length-checked, claim double-counting.
- **C8 EIP-7702 delegation** — delegated EOAs carrying code that hijacks
  approval flows; sweeper contracts resident in user EOAs.

## D. Arithmetic & precision

- **D1 Rounding direction** — share mint/redeem, debt interest, liquidation
  bonus: which side truncates; attacker wins dust systematically.
- **D2 Division-before-multiply** — `a / b * c` patterns.
- **D3 Precision loss** — 1e18 rate math on 1e6 tokens losing meaningful
  precision; Ray vs Wad confusion.
- **D4 Casting truncation** — `uint256→uint128/uint96` silent truncation in
  accounting; negative int→uint wraps.
- **D5 Overflow** — pre-0.8 SafeMath gaps; `unchecked` blocks hiding
  decrement of externally-influenced values.
- **D6 Signed/unsigned mixing** — PnL accounting int256 → uint256 casts of
  negative values.

## E. ERC-4626 / vault share mechanics

- **E1 First-depositor inflation** — `shares = assets * totalShares/totalAssets`
  with 0 supply; donation inflating share price; rounding dust minting 0.
  Look for virtual share offset / token decimals guard.
- **E2 Donation attacks** — direct token transfer inflating/deflating
  accounting that keys on `balanceOf(this)` instead of internal ledger.
- **E3 Deposit/withdraw asymmetry** — different rounding or fee application
  between mint/redeem paths enabling arbitrage against passive holders.
- **E4 Fee accounting** — performance/management fees computed on inflated
  or flash-manipulated totals; fee-on-transfer double count on deposit.
- **E5 max* functions** — `maxRedeem/maxWithdraw` inconsistency with real
  liquidity → revert DoS or infinite loop in routers.
- **E6 Preview vs actual divergence** — `previewRedeem` returns more than
  `redeem` delivers (integrators can be griefed).

## F. Token integration assumptions

- **F1 Return value checking** — ignored bool return (non-standard tokens
  silently failing); low-level `.call` to tokens missing return data decode.
- **F2 Fee-on-transfer** — `amount` credited ≠ amount received.
- **F3 Rebasing tokens** — balance-based accounting drifts.
- **F4 Approve race** — `approve` without `safeApprove`/`forceApprove`
  pattern on USDT-class tokens (non-zero → non-zero revert).
- **F5 Blacklist/blocklist tokens** — receiver blocked → revert DoS on
  shared paths (claim/liquidation); attacker grieves via blacklisted address.
- **F6 Odd decimals** — USDC-style 6-decimals in 1e18 math paths.
- **F7 Transfer to zero-address** — no validation, tokens lost.

## G. Lending & liquidation

- **G1 Liquidation math** — close factor bypass, bonus stacking, self-
  liquidation profit, partial-liquidation rounding harvesting.
- **G2 Bad debt path** — socialization vs reserve absorb; underwater accounts
  minting unlimited bad debt to reserve.
- **G3 Health factor edges** — division by zero when collateral 0; HF check
  off-by-one at exactly 1e18; borrow enable/disable mismatch per market.
- **G4 Interest accrual** — indexes not accrued before critical reads;
  accrual skipped in liquidation path; borrow index manipulation via
  interaction timing.
- **G5 Oracle in liquidation** — liquidatable price vs borrowable price
  divergence; different feed freshness per side.
- **G6 Dutch-auction style liquidations** — start price computed manipulable,
  keeper-only profit, price never reaches profitable → stuck debt.

## H. MEV, ordering & front-running

- **H1 Missing deadline / slippage params** — swaps/mints without user
  deadline; `amountOutMin=0` defaults in integrators.
- **H2 First-caller privileges** — initialization of rewards/epochs claimable
  by first caller; auction start manipulation.
- **H3 Block.timestamp** — < seconds-resolution assumptions; miner-controllable
  ±1s windows in auctions/vesting/rewards.
- **H4 Sandwichable accounting** — deposits/withdrawals that re-balance pools
  quotable and front-runnable against passive LPs.
- **H5 Reward epoch timing** — first action in epoch captures entire period
  rewards; last-second reward dumping.

## I. Governance, proxy & upgradeability

- **I1 Uninitialized proxies** — implementation direct-call initialization;
  logic contract with live initializers.
- **I2 Storage collisions** — upgraded storage layout shifts; namespaced
  storage (EIP-7201) missing.
- **I3 delegatecall targets** — user-influenced target; mutable target without
  timelock.
- **I4 Timelock bypass** — operations missing timelock (esp. fee setters,
  oracle setters, adapter registry changes); decreaseTimelock-to-zero paths.
- **I5 Quorum manipulation** — flash-loaned votes; snapshot timing;
  delegated balance double-counting.
- **I6 Beacon/UUPS pitfalls** — UUPS upgrade auth missing (`_authorizeUpgrade`
  public); beacon admin == protocol admin (upgrade = drain in one tx).

## J. DoS & griefing

- **J1 Unbounded loops** — user-growable arrays iterated in critical paths;
  market/vault registries iterated in accrual.
- **J2 Revert-on-fail shared paths** — one bad user (token-blocked, odd
  fallback) reverts batch operations (liquidation bots, keeper functions,
  distribution sweeps).
- **J3 Dust-griefing** — 1-wei deposits creating entries that break
  invariants (share=0, index division) or block finalization.
- **J4 Queue starvation** — FIFO queues where free entries block paid ones;
  cancel/refund paths that reorder.
- **J5 Withdrawal window games** — gate opening/closing manipulated; epoch
  boundary reverts.

## K. Cryptography & signatures

- **K1 Replay** — as C6; plus cross-protocol replay of identical message
  formats.
- **K2 Merkle proofs** — second-preimage via unhashed leaves; `abi.encodePacked`
  on dynamic types (collision).
- **K3 keccak type confusion** — hashing structs with different encodings
  accepted as same commitment.
- **K4 On-chain key custody** — private keys in source (deploy scripts,
  comments), profanity-style vanity addresses.
- **K5 Threshold/multisig assumptions** — validator sets with N-M too close
  (Ronin 4/5); key rotation absent.

## L. Bridges & cross-domain

- **L1 Message replay** — no nonce/ledger tracking (Nomad zero-proof class);
  merkle root rotation races.
- **L2 Chain-id binding** — messages valid across deployments.
- **L3 Finality assumptions** — withdrawal before source finality; sequencer
  reorg exposure on L2s.
- **L4 Burn/mint verification** — mint without verified burn; forged
  precompile proofs (BNB bridge class).
- **L5 Config poisoning** — remotes/paths settable; trusted path substitution.

## M. Perps & derivatives

- **M1 Funding rate manipulation** — funding computed from spot-perp spread
  attackable with small capital at boundary blocks.
- **M2 Mark/index divergence** — mark from manipulable AMM while index is
  external; liquidations triggered by manipulable mark.
- **M3 PnL accounting** — negative PnL casting; settlement between open/close
  using different prices; realized/unrealized mixing.
- **M4 ADL & insurance fund** — socialized loss ordering; insurance fund
  insolvency winding down positions at unfair prices.
- **M5 Position accounting overflow** — size caps absent; leverage via
  dust-position stacking.
- **M6 Liquidation keeper griefing** — bounty < gas at scale; order bounty
  double-claim windows.

## N. Economic & game-theoretic

- **N1 Incentive mispricing** — reward rate vs deposit base manipulable
  (deposit-boost-withdraw-keep patterns; rate = fixed emissions / staked).
- **N2 Curve convexity exploitation** — boost formulas (Curve vecRV) where
  boost = f(balanced²/total²) allow multi-account or timewarp farming.
- **N3 Discount/bond front-run** — bond pricing lagging market; discount
  claimable by arbitrageur not protocol.
- **N4 TVL/cap gaming** — caps measured on manipulable totals; mint caps
  bypassed via split transactions.
- **N5 Fee switch timing** — fee activation harvesting accrued value owned
  by LPs pre-activation.
- **N6 Rectification-for-profit** — protocol self-repair mechanisms (rebase
  if sub-peg, forced conversion) exploitable directionally.
