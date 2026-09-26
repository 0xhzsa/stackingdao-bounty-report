# StackingDAO Bug Bounty Report

## Contents
- `FINDING-01-escrow-cores-omission.md` — the full finding (severity High):
  `stx-reserve-v2` escrow-cores omits the live v2 core → stSTX price overstated
  +35.18bps on every exit. Root cause, live-verified numbers, attack path,
  replayable PoC, fix.
- `POC-REPLAY.md` — exact call-read replay steps (this file's companion).
- `TRIAGE-LOG.md` — everything else reviewed and killed, with reasons
  (manual triage on 22 live Clarity contracts).

## Status
- [x] Found + verified live on-chain (stacks_tip ~9059439)
- [x] Re-verified live 2026-09-26 @ stacks_tip 9069597 — all values unchanged
- [x] Duplicate check vs PoX-5 audit (zero mentions — not a duplicate)
