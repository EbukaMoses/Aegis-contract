# VolatilityShield — Contract Upgrade Guide

This document describes how to upgrade the VolatilityShield vault contract on Soroban without migrating user funds.

---

## How upgrades work on Soroban

Soroban supports in-place WASM replacement via `env.deployer().update_current_contract_wasm(new_wasm_hash)`. The swap is atomic: the new bytecode takes effect at the next transaction while all storage (balances, admin, configuration, etc.) is preserved untouched. No funds move and no user action is required.

The upgrade pattern in this contract separates two concerns:

| Step | Function | Who calls it | What it does |
|---|---|---|---|
| 1 | `upgrade(new_wasm_hash)` | Admin | Replaces the contract WASM. Emits `Upgrade/wasm` event. |
| 2 | `migrate(new_version)` | Admin | Advances the on-chain schema version and initialises any new storage fields introduced by the new code. Emits `Migrate/version` event. |

Both steps must be executed in separate transactions after the new WASM has been uploaded to the Soroban network.

---

## Versioned storage schema

A `DataKey::Version` entry (u32) is written to instance storage by `init` and incremented by `migrate`. The current schema version is also hard-coded as `CURRENT_VERSION` in `lib.rs` so the running bytecode always knows what schema it expects.

```
v1  (initial release)   – baseline storage layout
v2  (current upgrade)   – introduces DataKey::MaxStrategies (u32, 0 = uncapped)
```

New fields introduced in a version are always given a safe default so that existing deployments that predate the feature continue to function after migration.

---

## Step-by-step upgrade procedure

### Prerequisites

- You hold the `admin` account for the deployed contract.
- The new `.wasm` artefact has been built and uploaded; you have its hash (`BytesN<32>`).

### 1. Upload the new WASM

```bash
stellar contract upload \
  --wasm target/wasm32-unknown-unknown/release/volatility_shield.wasm \
  --source admin \
  --network testnet
```

Note the returned WASM hash (32-byte hex string).

### 2. Swap the WASM

Invoke `upgrade` with the new hash. Only the admin can call this.

```bash
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source admin \
  --network testnet \
  -- upgrade \
  --new_wasm_hash <WASM_HASH_HEX>
```

At this point the new code is live but the schema version is still at the previous value. State-mutating entry points (`deposit`, `withdraw`, `rebalance`) will panic with `MigrationRequired` if the stored version is below `CURRENT_VERSION`.

### 3. Run the migration

Invoke `migrate` with the **next sequential version number** (e.g. `2` when upgrading from v1):

```bash
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source admin \
  --network testnet \
  -- migrate \
  --new_version 2
```

`migrate` is strictly sequential — passing any version other than `current + 1` returns `InvalidMigrationVersion`. To upgrade across multiple versions repeat this step for each one: `migrate(2)`, then `migrate(3)`, etc.

### 4. Verify

```bash
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source admin \
  --network testnet \
  -- get_version
```

The output should equal the new version (e.g. `2`). The vault is now fully operational on the new schema.

---

## What each migration does

### v1 → v2

- **New field:** `DataKey::MaxStrategies` (u32). Controls the maximum number of yield strategies the vault may register. `0` means uncapped.
- **Default for existing deployments:** `0` (uncapped), so existing vaults are unaffected.
- **New entry points available after migration:** `set_max_strategies(max)`, `get_max_strategies()`.

---

## Rollback

Soroban does not support WASM rollback in a single operation. If a newly deployed version has a critical bug:

1. Re-upload the previous WASM and call `upgrade` again with its hash.
2. If `migrate` has already been called and the schema advanced, the old code must be able to handle the newer schema. Plan migrations to be backward-compatible where possible (additive fields with safe defaults).

---

## Security considerations

- `upgrade` and `migrate` require admin authentication. Protect the admin key accordingly.
- When multisig is enabled (`Threshold > 0`) critical operations already route through guardian proposals. Consider wrapping the upgrade flow in a multisig proposal for additional safety on mainnet.
- The version check in `deposit`, `withdraw`, and `rebalance` ensures users cannot interact with a half-migrated contract. The window between step 2 and step 3 should be minimised.
- Emit the `Migrate/version` event and verify it on-chain before announcing the upgrade to users.
