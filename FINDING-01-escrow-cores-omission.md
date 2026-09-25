# StackingDAO — stx-reserve-v2 escrow-cores omits active v2 core
## Severity: High (price-feed overstatement, systematic exit overpayment) — triage to decide High/Medium boundary; NOT Critical (no single-tx drain)

## Target
- `SP4SZE494VC2YC5JYG7AYFQ44F5Q4PYV7DVMDPBG.stx-reserve-v2` lines 32-34 (escrow-cores list)
- Consumed by `.data-stx-v2` `compute-ratio` (data-stx-v2.clar:31-50) → `get-stx-per-ststx`
- Both contracts are listed in-scope assets on the StackingDAO Immunefi program
  (Smart Contract scope; Primacy of Impact for Critical/High; PoC required)

## Impact (quantified, live mainnet state @ stacks_tip ~9059439)
- `escrow-cores` = [btc-v1, btc-v2, btc-v3, ststxbtc-v1]. Omits
  `stacking-dao-core-ststxbtc-v2`, which is LIVE (shutdown-deposits=false,
  shutdown-init-withdraw=false, verified on-chain) and holds 168,818,353,752
  microunits of escrowed ststxbtc-token-v2 (its init-withdraw escrow, same pattern
  as v1 — v2 clar line 68: `transfer … sender current-contract`).
- `get-escrowed-ststxbtc` counts 32,655,750,287; true escrow ≈ 201,474,104,039.
- `get-stx-for-ststxbtc` (active backing) therefore OVERSTATED by 168,818.35 STX.
- `get-stx-per-ststx` reported 1.187180 vs true 1.183019 → **+35.18 bps on every
  stSTX valuation** (41.8M stSTX supply; ~76.1M STX backing).
- Every permissionless stSTX exit overpays ~0.35%, funded by remaining holders:
  `stacking-dao-core-stx-v2::init-withdraw` (line 77) and `::withdraw-idle`
  (line 131) both price payouts with the overstated ratio; `swap-ststx-ststxbtc-v4`
  consumes it for swaps (lines 18, 47). Example: a 1,000,000 STX exit overpays
  ~3,518 STX. Repeatable on every exit, no privilege required.

## Root cause
`stx-reserve-v2.clar:32-34` — when `stacking-dao-core-ststxbtc-v2` went live
(scope added 13-Aug-2026), nobody added it to `escrow-cores`. One-tx DAO fix:
`set-escrow-cores` appending the v2 principal (function exists, lines 38-43).

## Attack path (unprivileged, all permissionless entry points)
1. Hold stSTX (market buy or prior deposit — no privilege needed).
2. Call `stacking-dao-core-stx-v2::init-withdraw` (open; shutdown flags false
   on both stx-v2 and ststxbtc-v2 cores, verified live).
3. Payout computed with overstated `get-stx-per-ststx` → receive ~0.35% more STX
   than fair backing. Repeat per exit. Loss sits with remaining holders.

## PoC / reproduction (deterministic, replayable by anyone, no keys needed)
Hiro mainnet `call-read` sequence (sender = any principal, zero args):
1. `stx-reserve-v2/get-escrow-cores` → 4 principals, no v2 core.
2. `stacking-dao-core-ststxbtc-v2/get-shutdown-deposits` → 0x04 (false = open).
3. `stacking-dao-core-ststxbtc-v2/get-shutdown-init-withdraw` → 0x04 (open).
4. v2 core `ststxbtc-token-v2` balance → 168,818,353,752 (unhandled escrow).
5. `stx-reserve-v2/get-escrowed-ststxbtc` → 32,655,750,287 (missing the above).
6. `data-stx-v2/get-stx-per-ststx` → 1,187,180 vs recomputed-fair 1,183,019.
Honest limitation: full simnet execution pending (clarinet Windows binary not
obtainable in this environment); the mispricing itself is proven live above.

## Not-a-duplicate evidence
- PoX-5 Clarity Alliance audit (13-Aug-2026, 60pp): zero mentions of
  `stx-reserve-v2`, `escrow-cores`, `core-ststxbtc-v2`, `get-stx-for-ststxbtc`,
  `data-stx-v2`. Adjacent findings (C-03 stBTC escrow timing, C-06 pending stSTXbtc
  withdrawals) are different mechanisms in different contracts.
- Fix available to DAO in one tx; mispricing live at time of writing.

## Recommended fix
`set-escrow-cores` with v2 appended (DAO call), plus a regression assert that
every live core holding escrow appears in the list.
