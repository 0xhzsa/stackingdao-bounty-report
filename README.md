# StackingDAO Bug Bounty Report — PRIVATE

> **DO NOT MAKE THIS REPO PUBLIC.** The StackingDAO Immunefi program requires
> **Category 3: Approval Required** for any publication. Publishing this finding
> before private submission via `bugs.immunefi.com`, triage, fix, and written
> approval **voids the bounty** and endangers user funds. This repo is private
> storage + submission staging only.

## Contents
- `FINDING-01-escrow-cores-omission.md` — the full finding (severity High):
  `stx-reserve-v2` escrow-cores omits the live v2 core → stSTX price overstated
  +35.18bps on every exit. Root cause, live-verified numbers, attack path,
  replayable PoC, fix.
- `POC-REPLAY.md` — exact call-read replay steps (this file's companion).
- `TRIAGE-LOG.md` — everything else reviewed and killed, with reasons
  (Slither-equivalent manual triage on 22 live Clarity contracts, IDOR claim
  killed as out-of-scope + public-data).

## Submission checklist (do on bugs.immunefi.com)
1. Program: StackingDAO → Submit a Bug → Smart Contract.
2. Paste `FINDING-01-escrow-cores-omission.md` sections into Title / Severity /
   Target / Impact / Root Cause / Attack Path / PoC / Fix.
3. Severity: High. Do NOT mark Critical (no single-tx drain; systematic
   overpayment — let triage confirm).
4. Keep this repo PRIVATE until written Category-3 approval arrives.

## Status
- [x] Found + verified live on-chain (stacks_tip ~9059439)
- [x] Duplicate check vs PoX-5 audit (zero mentions — not a duplicate)
- [ ] Submitted to Immunefi (needs your account)
- [ ] Triage response
- [ ] Fix confirmed on-chain
- [ ] Publication approval (only then may this go public)
