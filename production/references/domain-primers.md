# Domain Primers

Read the primer(s) matching the Phase-1 domain classification. Primers state
the protocol's core invariants — a finding must break one of these to be
real, not merely ugly.

## P1 DEX / AMM

Invariants: swap k-product (or stableswap invariant) conservation; LP share
accounting; fee accrual to LPs not attackers.
Must-check: spot-price read patterns (B1), first-LP/zero-liquidity init,
`skim`/`sync` donation interactions, callback trust (A5), fee-on-transfer in
routers (F2), curve precision at extremes (B6/D3).
Classic bugs: donation breaking reserves accounting; imbalance arbitrage at
rounding corners; swap callback reentrancy.

## P2 Lending / CDP

Invariants: solvency (borrow ≤ collateral·LTV·price); per-market accounting
integrity (sum of deposits = sum of supplies); liquidation never creates profit
loops.
Must-check: oracle on BOTH borrow and liquidation paths (B*, G5), interest
accrual before reads (G4), close factor & bonus math (G1), bad-debt
socialization (G2), share inflation on collateral vaults (E1), HF division
edges (G3).
Classic bugs: mark-price manipulation → mass liquidation (Mango class);
interest index not accrued in one path; self-liquidation bonus farming.

## P3 Vault / Yield aggregator (ERC-4626 & strategies)

Invariants: share price monotonicity (non-decreasing except explicit fees);
1:1 backing (assets under management = liabilities to shareholders); strategy
adapters can't move funds anywhere but whitelisted venues.
Must-check: first-depositor inflation (E1), donation accounting (E2),
`totalAssets()` read-only reentrancy (A4), strategy `harvest` reward-token
arbitrage front-run (N1), withdrawal queue DoS (J4/J5), fee computed on
manipulable base (E4).
Classic bugs: inflation attack on low-decimal vaults; harvest() front-run of
reward swaps; emergency-withdraw skipping share burn.

## P4 Staking / Rewards distribution

Invariants: rewards distributed ≤ rewards funded; per-user reward accounting
exact (no double-claim, no loss on early exit); emission rate changes apply
fairly to all stakers.
Must-check: reward-per-token update ordering (accrue before state change),
`stake`/`withdraw` mid-epoch handling, reward token ≠ staking token paths,
fixed-emission over manipulable denominator (N1), claim DoS via token
blacklist (F5), first-caller epoch capture (H5).
Classic bugs: withdraw without accruing → lost/stolen rewards; `exit` opens
reentrancy on reward transfer; reward rate updated without checkpoint
stealing past accruals.

## P5 Governance

Invariants: one token-one vote at a fixed snapshot; proposal execution
exactly as voted (no post-vote parameter swap); timelock precedes execution.
Must-check: vote snapshot vs balance manipulation (I5), proposal creation
spam thresholds, `execute` re-entrancy on external calls, self-delegation
tricks, quorum zero-vote pass (quorum=0), timelock cancellation authority
(I4).
Classic bugs: flash-vote; delegate balance double-count in checkpoint
lib; proposalData not stored → execute uses current params.

## P6 Oracle provider / price feed consumer

Invariants: price freshness bounded; deviation bounded; manipulation cost ≫
extractable value.
Must-check: staleness (B3), L2 sequencer uptime (B3), setter auth (B4),
decimal normalization (B5), multi-feed aggregation (median vs average vs
first), fallback feed poisoning (secondary feed stale-but-deviant chosen).
Classic bugs: fallback to stale feed during volatility; `minAnswer/maxAnswer`
circuit breakers freezing price at stale bound ( Gearbox/Starship class).

## P7 Perps / derivatives

Invariants: long-side PnL = short-side PnL (conservation); insurance fund
never negative-converted silently; open interest caps enforced consistently.
Must-check: funding at boundary blocks (M1), mark manipulability (M2),
settlement price consistency open vs close (M3), liquidation bounty economics
(M6), position dust stacking (M5), ADL ordering fairness (M4).
Classic bugs: funding-rate flip harvesting; unrealized→realized PnL double
count; liquidation below maintenance but bonus > maint → solvency drain.
