# [Medium] stx-reserve-v2 escrow-cores omits the live v2 core — stSTX ratio UNDERSTATED ~35 bps, every stSTX exit underpaid

## Important correction note

An earlier internal draft claimed the ratio was **overstated** (+35 bps, exits
overpaid). That direction was wrong. Re-derivation from live mainnet state
(and following `compute-ratio` step-by-step) shows the opposite: because
`get-stx-for-ststxbtc = ststxbtc-v2 supply − escrowed-ststxbtc`, and the
omitted v2 core's escrow is invisible, the "claimed" term is **overstated**,
`active-backing` is **understated**, and therefore `get-stx-per-ststx` is
**UNDERSTATED by 4,151 units = −34.96 bps**. stSTX holders are systematically
**underpaid** on exits, and `swap-ststxbtc-for-ststx` over-mints stSTX.

## Summary

`stx-reserve-v2` prices stSTX off an `escrow-cores` allowlist that was never
updated when `stacking-dao-core-ststxbtc-v2` went live. The v2 core's
168,818,353,752 microunits of init-withdraw escrow are invisible to the reserve,
so the published `get-stx-per-ststx` reads **1,187,413 vs a fair 1,191,564
(−34.96 bps)**. Every permissionless stSTX exit is **underpaid** ~0.35% (about
4,151 STX per 1,000,000 stSTX exit ≈ $1.4K @ 0.3365), and the inverse swap
path mints excess stSTX from ststxbtc. One DAO transaction
(`set-escrow-cores`) fixes it.

- **Severity:** Medium — systematic price-feed understatement with direct,
  repeatable fund impact on stSTX holders; no single-transaction drain, no
  privileged actor needed. Triage boundary Low/Medium.
- **Status:** live on mainnet, re-verified 2026-09-26 (all values below re-read
  at current tip; nothing patched).

## Impact (all numbers read live, recomputation shown)

Live reads @ current tip:
- `stx-reserve-v2/get-escrow-cores` → 4 principals: btc-v1, btc-v2, btc-v3,
  ststxbtc-v1 — **no ststxbtc-v2**.
- v2 core `ststxbtc-token-v2` balance (unhandled escrow) → **168,818,353,752**.
- `stx-reserve-v2/get-escrowed-ststxbtc` → 32,655,750,287 (missing the above).
- `stx-reserve-v2/get-total-stx` → 76,075,087,218,203.
- `stx-reserve-v2/get-stx-for-withdrawals` → 843,962,221,599.
- `stx-reserve-v2/get-stx-for-ststxbtc` → 26,948,929,319,239.
- `ststx-token/get-total-supply` → 41,976,884,034,431.
- `data-stx-v2/get-live-escrow` → 1,315,217,964,487.
- `data-stx-v2/get-stx-per-ststx` → **1,187,413**.

Recomputation (exact integer arithmetic):

| term | live | fair (escrow includes v2 core) |
|------|------|-------------------------------|
| claimed | 26,948,929,319,239 + 843,962,221,599 = 27,792,891,540,838 | (26,948,929,319,239 − 168,818,353,752) + 843,962,221,599 = 27,624,073,187,086 |
| active-backing | 48,282,195,677,365 | 48,451,014,031,117 |
| active-supply | 41,976,884,034,431 − 1,315,217,964,487 = 40,661,666,069,944 | same |
| ratio (×1e6) | 1,187,413 | 1,191,564 |

Understatement = 4,151 units = **34.958 bps**; misattributed backing =
168,818.353752 STX ≈ **$56,807 @ $0.3365**.

Concrete loss: a 1,000,000 stSTX exit pays 1,187,413 STX when fair backing is
1,191,564 STX → exiting holder loses ≈ 4,151 STX. Repeatable on every exit,
no privilege required.

Affected entry points (all permissionless, all consume the understated ratio):
`stacking-dao-core-stx-v2::init-withdraw` (line 77, `get-sbtc-per`... ratio),
`stacking-dao-core-stx-v2::withdraw-idle` (line 131),
`swap-ststx-ststxbtc-v4::swap-ststx-for-ststxbtc` (line 18),
`swap-ststx-ststxbtc-v4::swap-ststxbtc-for-ststx` (line 47, over-mint).

## Root cause

`stx-reserve-v2.clar:32-34` — the `escrow-cores` list was written for the v1
era. When `stacking-dao-core-ststxbtc-v2` went live, nobody appended it, so
`get-escrowed-ststxbtc` sums v1-era escrow only while the v2 core holds real
escrow under the identical lock pattern (v2 clar line 68:
`transfer … sender current-contract`, same as v1).

## Attack path (unprivileged)

1. Hold stSTX (market buy or prior deposit — no privilege needed).
2. Call `stacking-dao-core-stx-v2::init-withdraw` (open; shutdown flags false
   on both cores, verified live).
3. Payout computed with the understated `get-stx-per-ststx` → exiting holder
   receives ~0.35% less STX than fair backing. Repeat per exit; the difference
   stays in the reserve/system rather than going to the user.

Note the value does not leave the protocol — it is retained from the exiting
holder. This is a systematic mispricing bug (users underpaid; ststxbtc→stSTX
swap path mints excess), consistent with a Medium price-feed/accounting defect
rather than a drain.

## PoC / reproduction

Deterministic, read-only, replayable by anyone with no keys: Hiro mainnet
`call-read` sequence (sender = any principal, zero args), then the integer
recomputation above:
1. `stx-reserve-v2/get-escrow-cores`
2. `stacking-dao-core-ststxbtc-v2/get-shutdown-deposits` → false
3. `stacking-dao-core-ststxbtc-v2/get-shutdown-init-withdraw` → false
4. v2 core `ststxbtc-token-v2/get-balance` → 168,818,353,752
5. `stx-reserve-v2/get-escrowed-ststxbtc` → 32,655,750,287
6. `stx-reserve-v2/get-total-stx`, `get-stx-for-withdrawals`,
   `get-stx-for-ststxbtc`
7. `ststx-token/get-total-supply`, `data-stx-v2/get-live-escrow`
8. `data-stx-v2/get-stx-per-ststx` → 1,187,413 vs recomputed-fair 1,191,564

Honest limitation: full simnet execution is pending (no Clarinet binary in
this environment). What is proven live is the mispricing itself — every input
to the payout formula, read from deployed contracts.

## Not a duplicate

PoX-5 Clarity Alliance audit (13-Aug-2026, 60pp): zero mentions of
`stx-reserve-v2`, `escrow-cores`, `core-ststxbtc-v2`, `get-stx-for-ststxbtc`,
or `data-stx-v2`. Adjacent findings (C-03 stBTC escrow timing, C-06 pending
stSTXbtc withdrawals) are different mechanisms in different contracts.

## Recommended fix

`set-escrow-cores` with the v2 principal appended (single DAO call; function
exists at lines 38-43), plus a regression assertion that every live core
holding escrow appears in the list.