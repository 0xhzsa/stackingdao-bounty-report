# Triage Log — reviewed and killed (StackingDAO + related)

## StackingDAO Clarity (22 live contracts, mainnet source)
- data-stbtc-v1 pending-vs-supply underflow shape → KILLED (atomic lock/escrow/burn paths; invariant holds; live supply 9.55 sBTC vs pending 500 sats).
- stbtc-staker-bond-1-v2 rollover STX delta → real logic gap but KILLED (strategy-manager key only = privileged, out of scope).
- signer-admin-v1 + both signer managers → KILLED (DAO-gated, per-instance keys, no confusion).
- stbtc-token mint/burn, receive-migration, signer-payout keeper, permissionless claims (own-rewards only), tracking refresh (value-neutral), swap-v4 ordering, strategy/calculator gates → all KILLED with live-code reasons.
- Veda/Intuition tracks: Veda enter/exit+Manager 5 leads killed; Veda fuzz 3/3 green (1920 calls); Intuition Slither 34/34 killed + vault fuzz green (25k calls).
