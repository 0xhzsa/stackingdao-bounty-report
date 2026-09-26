# PoC Replay — escrow-cores omission (read-only, no keys, no transactions)

Base: `https://api.mainnet.hiro.so`. All calls are POST
`/v2/contracts/call-read/SP4SZE494VC2YC5JYG7AYFQ44F5Q4PYV7DVMDPBG/<contract>/<fn>`
with body `{"sender":"<any-principal>","arguments":[]}`.

| # | Call | Expected (vulnerable state) |
|---|------|------------------------------|
| 1 | `stx-reserve-v2/get-escrow-cores` | 4 principals: btc-v1, btc-v2, btc-v3, ststxbtc-v1 — **no ststxbtc-v2** |
| 2 | `stacking-dao-core-ststxbtc-v2/get-shutdown-deposits` | `0x04` (false = deposits OPEN) |
| 3 | `stacking-dao-core-ststxbtc-v2/get-shutdown-init-withdraw` | `0x04` (false = withdrawals OPEN) |
| 4 | `GET /extended/v1/address/SP4SZE.../stacking-dao-core-ststxbtc-v2/balances` | `ststxbtc-token-v2` balance = 168,818,353,752 (uncounted escrow) |
| 5 | `stx-reserve-v2/get-escrowed-ststxbtc` | 32,655,750,287 (missing step-4 amount) |
| 6 | `ststxbtc-token-v2/get-total-supply` | `(ok u26983585069526)` |
| 7 | `stx-reserve-v2/get-stx-for-ststxbtc` | 1,684,442,023,239 microSTX — live active-backing leg, overstated (step-4 never enters the formula) |
| 8 | `data-stx-v2/get-stx-per-ststx` | 1,187,180 live vs 1,183,019 fair (recomputed with the missing 168,818,353,752 escrow included) |

Windows PowerShell replay (save body once):
```powershell
Set-Content callbody.json '{"sender":"SP4SZE494VC2YC5JYG7AYFQ44F5Q4PYV7DVMDPBG","arguments":[]}' -NoNewline
$base = "https://api.mainnet.hiro.so/v2/contracts/call-read/SP4SZE494VC2YC5JYG7AYFQ44F5Q4PYV7DVMDPBG"
curl.exe -s --max-time 30 -X POST -H "Content-Type: application/json" --data "@callbody.json" "$base/stx-reserve-v2/get-escrow-cores"
# ... repeat per table
```

Profit demonstration (no execution needed — deterministic view math):
overpayment per stSTX exit = (1,187,180 − 1,183,019)/1e6 ≈ 0.0035 STX per stSTX.
1,000,000 STX exit → ≈ +3,518 STX vs fair.
