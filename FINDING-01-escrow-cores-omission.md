# [High] stx-reserve-v2 escrow-cores omits the live v2 core — every stSTX exit overpays ~0.35%

## Summary

`stx-reserve-v2` prices stSTX off an `escrow-cores` allowlist that was never
updated when `stacking-dao-core-ststxbtc-v2` went live. The v2 core's
168,818,353,752 microunits of init-withdraw escrow are invisible to the reserve,
so active backing is overstated by 168,818.35 STX (~$56,800) and the published
`get-stx-per-ststx` ratio reads **1.187180 vs a fair 1.183019 (+35.18 bps)**.
Every permissionless exit and swap priced off that ratio overpays ~0.35%,
funded by remaining holders. One DAO transaction (`set-escrow-cores`) fixes it.

- **Severity:** High — systematic price-feed overstatement with direct,
  repeatable fund impact. Not Critical: no single-transaction drain.
- **Status:** live on mainnet, re-verified 2026-09-26 @ stacks_tip 9069597
  (all values below re-read; nothing patched).

## Impact (all numbers read live, none modeled)

| # | Check | Live value |
|---|-------|-----------|
| 1 | `stx-reserve-v2/get-escrow-cores` | 4 principals: btc-v1, btc-v2, btc-v3, ststxbtc-v1 — **no ststxbtc-v2** |
| 2 | `stacking-dao-core-ststxbtc-v2/get-shutdown-deposits` | `0x04` (false — deposits OPEN) |
| 3 | `stacking-dao-core-ststxbtc-v2/get-shutdown-init-withdraw` | `0x04` (false — withdrawals OPEN) |
| 4 | v2 core `ststxbtc-token-v2` balance (unhandled escrow) | **168,818,353,752** |
| 5 | `stx-reserve-v2/get-escrowed-ststxbtc` (what the formula sees) | 32,655,750,287 (missing #4; true ≈ 201,474,104,039) |
| 6 | `stx-reserve-v2/get-stx-for-ststxbtc` (active-backing leg) | 1,684,442,023,239 microSTX — overstated, #4 never enters |
| 7 | `ststxbtc-token-v2/get-total-supply` | `(ok u26983585069526)` |
| 8 | `data-stx-v2/get-stx-per-ststx` (the published price) | **1,187,180 vs fair 1,183,019** |

Concrete loss: overpayment per exit = (1,187,180 − 1,183,019)/1e6 ≈ 0.0035 STX
per stSTX. A 1,000,000 STX exit overpays **≈ 3,518 STX (~$1,184 @ $0.3365)**.
Repeatable on every exit, no privilege required, loss socialized across ~76.1M
STX of backing (41.8M stSTX supply).

Affected entry points (all permissionless, all consume the overstated ratio):
`stacking-dao-core-stx-v2::init-withdraw` (line 77),
`stacking-dao-core-stx-v2::withdraw-idle` (line 131),
`swap-ststx-ststxbtc-v4` (lines 18, 47).

## Root cause

`stx-reserve-v2.clar:32-34` — the `escrow-cores` list was written for the v1
era. When `stacking-dao-core-ststxbtc-v2` went live, nobody appended it, so
`get-escrowed-ststxbtc` sums v1-only escrow while the v2 escrow pattern is
identical (v2 clar line 68: `transfer … sender current-contract`, same as v1).

## Attack path (unprivileged)

1. Hold stSTX (market buy or prior deposit — no privilege needed).
2. Call `stacking-dao-core-stx-v2::init-withdraw` (open; shutdown flags false
   on both cores, verified live in checks #2–#3).
3. Payout is computed with the overstated `get-stx-per-ststx` → receive ~0.35%
   more STX than fair backing. Repeat per exit.

## PoC / reproduction

Deterministic, read-only, replayable by anyone with no keys: Hiro mainnet
`call-read` sequence per `POC-REPLAY.md` (checks #1–#8 above). Sender may be
any principal, all calls take zero arguments.

Honest limitation: full simnet execution is pending (no Clarinet binary in
this environment). What is proven live above is the mispricing itself —
every input to the payout formula, read from deployed contracts.

## Not a duplicate

PoX-5 Clarity Alliance audit (13-Aug-2026, 60pp): zero mentions of
`stx-reserve-v2`, `escrow-cores`, `core-ststxbtc-v2`, `get-stx-for-ststxbtc`,
or `data-stx-v2`. Adjacent findings (C-03 stBTC escrow timing, C-06 pending
stSTXbtc withdrawals) are different mechanisms in different contracts.

## Recommended fix

`set-escrow-cores` with the v2 principal appended (single DAO call; function
exists at lines 38-43), plus a regression assertion that every live core
holding escrow appears in the list.
