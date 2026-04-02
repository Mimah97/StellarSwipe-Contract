# feat: Add per-network config and network-aware deploy script

## Summary

Parameters like `MIN_STAKE`, `MAX_FEE_RATE`, and oracle addresses differ between testnet and mainnet. Hardcoding these values caused confusing test failures and potential financial risk.

This PR introduces a `NetworkConfig` structure stored in per-network JSON files, loaded at deploy time based on the `STELLAR_NETWORK` environment variable. Mainnet config is explicitly blocked from ever running in automated tests or CI.

## Changes

### New files

| File | Purpose |
|------|---------|
| `config/testnet.json` | Testnet config — safe for CI and automated tests |
| `config/mainnet.json` | Mainnet config — guarded by `REPLACE_WITH_` sentinels |
| `scripts/deploy.ts` | Network-aware deploy script that reads config by network |

### Modified files

| File | Change |
|------|--------|
| `.github/workflows/ci.yml` | Added `STELLAR_NETWORK: testnet` env default + mainnet guard step |

## Config fields (NetworkConfig)

```ts
interface NetworkConfig {
  min_stake: number;      // Minimum stake in stroops (i128)
  max_fee_rate: number;   // Max fee in basis points (u32)
  oracle_address: string; // Soroban oracle contract address
  admin: string;          // Admin account StrKey (G...)
}
```

## Testnet vs Mainnet values

| Field | Testnet | Mainnet |
|-------|---------|---------|
| `min_stake` | `10_000_000` (10 XLM) | `100_000_000` (100 XLM) |
| `max_fee_rate` | `500` (5%) | `100` (1%) |
| `oracle_address` | Testnet placeholder | `REPLACE_WITH_MAINNET_ORACLE_ADDRESS` |
| `admin` | Testnet placeholder | `REPLACE_WITH_MAINNET_ADMIN_ADDRESS` |

## Mainnet safety

- `config/mainnet.json` uses `REPLACE_WITH_` sentinels — `deploy.ts` hard-fails if any are unset
- `deploy.ts` exits with an error if `CI=true` and `STELLAR_NETWORK=mainnet`
- CI workflow pins `STELLAR_NETWORK: testnet` at the env level
- A dedicated guard step runs before checkout and fails the job if mainnet is ever set in CI

## Checklist

- [x] `config/testnet.json` and `config/mainnet.json` present with documented field explanations
- [x] Deploy script uses correct config based on `STELLAR_NETWORK` flag
- [x] Testnet and mainnet configs have clearly different values (especially `admin` and `min_stake`)
- [x] CI only ever uses testnet config (env default + guard step)
- [x] No secrets or real mainnet keys committed
- [ ] Tests / `cargo check` pass for crates touched
- [ ] Linked issue / ticket (if any):
