# rlp — CPI Documentation

> Part of [contract-interfaces](../readme.md). Shared conventions and well-known Solana program addresses live in the repo readme.

## Overview

**RLP** (Reflect Liquidity Pool) is a **liquid insurance fund**. Liquidity providers deposit a whitelisted asset and receive **LP tokens**; the pool can be **slashed** (up to a per-tx cap, by a permissioned crank) to cover losses elsewhere, and **rewards** can be deposited to grow the pool's value per LP token. Withdrawals are **two-step** with a cooldown, during which the LP position remains slashable. It also offers an oracle-priced **swap** between whitelisted pool assets.

RLP is the **junior tranche / insurance backstop** in the [reflect-tranches](./reflect-tranches.md) product (the [reflect-proxy-program](./reflect-proxy-program.md) is the senior tranche). It is asset-agnostic; assets are priced via **Pyth** or **Switchboard** oracles. Admin and crank operations are **role-gated** (see [Access control](#access-control)).

## Program Address

| Environment | Address |
| ----------- | ------- |
| Production (deployed) | `rhLMe6vyM1wVLJaxrWUckVmPxSia58nSWZRDtYQow6D` |

> Note: the reflect-tranches orchestration layer (`@reflectmoney/insurance` SDK and its README) still references the stale address `rLPmw7RxwLbArmZnYRFDBNKCd18vCkPjCB5pouwBAct` — that SDK/README should be updated to the deployed address above.

## Source of truth

Discriminators, account orderings, and arguments are taken from the program's **Anchor IDL** (`rlp/target/idl/rlp.json`) and generated SDK (`rlp/sdk`). 8-byte Anchor discriminators. Regenerate the IDL/SDK and re-derive this doc when the program changes.

## Concepts

| Account | PDA seeds | Purpose |
| ------- | --------- | ------- |
| `Settings` | `["settings"]` | Global singleton — pool/asset counts, global freeze, `swap_fee_bps`, access control |
| `LiquidityPool` | `["liquidity_pool", index: u8]` | A pool; `lp_token` mint, `cooldown_duration` |
| `Asset` | `["asset", mint]` | A whitelisted asset — its `oracle` (Pyth/Switchboard) and `access_level` |
| `Cooldown` | `["cooldown", …]` | A pending withdrawal — `authority`, `liquidity_pool_id`, `unlock_ts` |
| `UserPermissions` | `["permissions", authority]` | Role holder for an authority (see roles below) |

Pool/cooldown token vaults (`*_pool`, `*_lp_token_account`, `user_lp_account`) are **ATAs** of the owning PDA for the relevant mint.

**Roles** (`Role`): `UNSET, PUBLIC, TESTEE, FREEZE, CRANK, MANAGER, SUPREMO`. The **CRANK** role gates `deposit_rewards` and `slash`; `MANAGER`/`SUPREMO` gate setup/admin. `Action` enumerates every gated operation; `update_action_role` maps roles→actions and `update_role_holder` grants/revokes a holder's roles. `AccessLevel` (`Public`/`Private`) controls per-asset access.

---

## User flows

### `restake` — deposit `[disc 97,161,241,167,6,32,213,53]`

Deposit a whitelisted asset into a pool, mint LP tokens. Args: `liquidity_pool_index: u8`, `amount: u64`, `min_lp_tokens: u64` (slippage).

| # | Account | W | S | Description |
| -: | ------- | :-: | :-: | ----------- |
| 1 | `signer` | ✅ | ✅ | Depositor |
| 2 | `settings` | ✅ | ❌ | `["settings"]` |
| 3 | `permissions` | ❌ | ❌ | Optional — depositor's `UserPermissions` (private assets) |
| 4 | `liquidity_pool` | ❌ | ❌ | `["liquidity_pool", index]` |
| 5 | `lp_token` | ✅ | ❌ | Pool LP mint |
| 6 | `user_lp_account` | ✅ | ❌ | Depositor's LP ATA |
| 7 | `asset` | ❌ | ❌ | `["asset", asset_mint]` |
| 8 | `asset_mint` | ✅ | ❌ | Deposited asset mint |
| 9 | `user_asset_account` | ✅ | ❌ | Depositor's asset ATA (source) |
| 10 | `pool_asset_account` | ✅ | ❌ | Pool asset vault (ATA of pool) |
| 11 | `oracle` | ❌ | ❌ | Asset oracle (Pyth/Switchboard) |
| 12 | `token_program` | ❌ | ❌ | SPL Token |
| 13 | `associated_token_program` | ❌ | ❌ | ATA program |
| 14 | `system_program` | ❌ | ❌ | System |

### `request_withdrawal` — start cooldown `[disc 251,85,121,205,56,201,12,177]`

Lock LP tokens into a `Cooldown`; the timer starts. LP stays slashable until claimed. Args: `liquidity_pool_id: u8`, `amount: u64` (LP tokens).

| # | Account | W | S | Description |
| -: | ------- | :-: | :-: | ----------- |
| 1 | `signer` | ✅ | ✅ | Withdrawer |
| 2 | `settings` | ✅ | ❌ | `["settings"]` |
| 3 | `permissions` | ❌ | ❌ | Optional `UserPermissions` |
| 4 | `liquidity_pool` | ✅ | ❌ | `["liquidity_pool", id]` |
| 5 | `lp_token_mint` | ✅ | ❌ | Pool LP mint |
| 6 | `signer_lp_token_account` | ✅ | ❌ | Withdrawer's LP ATA (source) |
| 7 | `cooldown` | ✅ | ❌ | `["cooldown", …]` — new cooldown record |
| 8 | `cooldown_lp_token_account` | ✅ | ❌ | Escrow LP ATA owned by `cooldown` |
| 9 | `token_program` | ❌ | ❌ | SPL Token |
| 10 | `associated_token_program` | ❌ | ❌ | ATA program |
| 11 | `system_program` | ❌ | ❌ | System |

### `withdraw` — claim after cooldown `[disc 183,18,70,156,148,109,161,34]`

After `unlock_ts`, burn the escrowed LP and receive the proportional asset share. Args: `liquidity_pool_id: u8`, `cooldown_id: u64`.

| # | Account | W | S | Description |
| -: | ------- | :-: | :-: | ----------- |
| 1 | `signer` | ✅ | ✅ | Withdrawer |
| 2 | `settings` | ✅ | ❌ | `["settings"]` |
| 3 | `permissions` | ❌ | ❌ | Optional `UserPermissions` |
| 4 | `liquidity_pool` | ✅ | ❌ | `["liquidity_pool", id]` |
| 5 | `lp_token_mint` | ✅ | ❌ | Pool LP mint |
| 6 | `cooldown_lp_token_account` | ✅ | ❌ | Escrow LP ATA (burned from) |
| 7 | `cooldown` | ✅ | ❌ | `["cooldown", …]` — closed on claim |
| 8 | `token_program` | ❌ | ❌ | SPL Token |
| 9 | `system_program` | ❌ | ❌ | Optional — System |

### `swap` — asset ↔ asset `[disc 248,198,158,145,225,117,135,200]`

Swap between two whitelisted pool assets at oracle prices, less `settings.swap_fee_bps`. Args: `amount_in: u64`, `min_out: Option<u64>` (slippage).

| # | Account | W | S | Description |
| -: | ------- | :-: | :-: | ----------- |
| 1 | `signer` | ✅ | ✅ | Swapper |
| 2 | `admin` | ❌ | ❌ | Optional `UserPermissions` (private swap) |
| 3 | `settings` | ❌ | ❌ | `["settings"]` |
| 4 | `liquidity_pool` | ❌ | ❌ | `["liquidity_pool", index]` |
| 5 | `token_from` | ❌ | ❌ | Input asset mint |
| 6 | `token_from_asset` | ❌ | ❌ | `["asset", token_from]` |
| 7 | `token_from_oracle` | ❌ | ❌ | Input oracle |
| 8 | `token_to` | ❌ | ❌ | Output asset mint |
| 9 | `token_to_asset` | ❌ | ❌ | `["asset", token_to]` |
| 10 | `token_to_oracle` | ❌ | ❌ | Output oracle |
| 11 | `token_from_pool` | ✅ | ❌ | Input pool vault (ATA) |
| 12 | `token_to_pool` | ✅ | ❌ | Output pool vault (ATA) |
| 13 | `token_from_signer_account` | ✅ | ❌ | Swapper's input ATA |
| 14 | `token_to_signer_account` | ✅ | ❌ | Swapper's output ATA |
| 15 | `token_program` | ❌ | ❌ | SPL Token |
| 16 | `associated_token_program` | ❌ | ❌ | ATA program |

---

## Crank flows (CRANK role)

### `deposit_rewards` `[disc 52,249,112,72,206,161,196,1]`

Add asset to a pool **without minting LP** — raises value per LP token (yield routing). Args: `amount: u64`.

| # | Account | W | S | Description |
| -: | ------- | :-: | :-: | ----------- |
| 1 | `signer` | ✅ | ✅ | Crank (CRANK role) |
| 2 | `settings` | ✅ | ❌ | `["settings"]` |
| 3 | `permissions` | ❌ | ❌ | Crank's `UserPermissions` |
| 4 | `liquidity_pool` | ❌ | ❌ | `["liquidity_pool", index]` |
| 5 | `signer_asset_token_account` | ✅ | ❌ | Crank's asset ATA (source) |
| 6 | `asset_mint` | ❌ | ❌ | Asset mint |
| 7 | `asset` | ❌ | ❌ | `["asset", asset_mint]` |
| 8 | `asset_pool` | ✅ | ❌ | Pool asset vault (ATA) |
| 9 | `token_program` | ❌ | ❌ | SPL Token |
| 10 | `associated_token_program` | ❌ | ❌ | ATA program |
| 11 | `system_program` | ❌ | ❌ | System |

### `slash` `[disc 204,141,18,161,8,177,92,142]`

Extract asset from a pool to a destination (insurance payout). Args: `liquidity_pool_id: u8`, `amount: u64` (per-tx cap enforced).

| # | Account | W | S | Description |
| -: | ------- | :-: | :-: | ----------- |
| 1 | `signer` | ✅ | ✅ | Crank (CRANK role) |
| 2 | `permissions` | ✅ | ❌ | Crank's `UserPermissions` |
| 3 | `settings` | ✅ | ❌ | `["settings"]` |
| 4 | `liquidity_pool` | ❌ | ❌ | `["liquidity_pool", id]` |
| 5 | `mint` | ✅ | ❌ | Asset mint |
| 6 | `asset` | ✅ | ❌ | `["asset", mint]` |
| 7 | `liquidity_pool_token_account` | ✅ | ❌ | Pool asset vault (source) |
| 8 | `destination` | ❌ | ❌ | Recipient token account |
| 9 | `token_program` | ❌ | ❌ | SPL Token |

---

## Setup & admin (MANAGER / SUPREMO)

Account flags: `s` = signer, `w` = writable, `opt` = optional, PDAs noted. `UserPermissions` accounts use seeds `["permissions", authority]`.

- **`initialize_rlp`** `[disc 122,110,49,121,142,92,82,100]` — create the `Settings` singleton (caller becomes SUPREMO). Args: `swap_fee_bps: u16`, `cooldown_duration: u64`. Accounts: `signer`(ws), `permissions`(w,pda), `settings`(w,pda), `system_program`.
- **`initialize_lp`** `[disc 110,252,116,251,81,191,57,96]` — create a pool + LP mint. Args: `cooldown_duration: u64`, `cooldowns: u64`. Accounts: `signer`(ws), `permissions`(w,pda), `settings`(w,pda), `liquidity_pool`(w,pda), `lp_token_mint`, `system_program`, `token_program`, `associated_token_program`.
- **`initialize_lp_token_account`** `[disc 209,159,81,216,119,73,6,149]` — create a pool's asset vault ATA. Args: `liquidity_pool_index: u8`. Accounts: `signer`(ws), `admin`(w,pda), `settings`(w,pda), `liquidity_pool`(pda), `asset`(pda), `mint`, `lp_mint_token_account`(w,pda), `system_program`, `token_program`, `associated_token_program`.
- **`add_asset`** `[disc 81,53,134,142,243,73,42,179]` — whitelist an asset with its oracle. Args: `access_level: AccessLevel`. Accounts: `signer`(ws), `admin`(w,pda), `settings`(w,pda), `asset`(w,pda), `asset_mint`(w), `oracle`(w), `system_program`.
- **`update_deposit_cap`** `[disc 175,41,137,203,27,184,245,164]` — set/clear a pool deposit cap. Args: `lockup_id: u64`, `new_cap: Option<u64>`. Accounts: `signer`(ws), `admin`(w,pda), `settings`(w,pda).
- **`freeze_functionality`** `[disc 65,152,119,202,25,239,206,157]` — freeze/unfreeze an `Action`. Args: `action: Action`, `freeze: bool`. Accounts: `admin`(ws), `settings`(w,pda), `system_program`, `admin_permissions`(w,pda).

## Access control

- **`create_permission_account`** `[disc 168,123,227,158,114,102,13,95]` — open a `UserPermissions` for an authority. Args: `new_admin: Pubkey`. Accounts: `settings`(pda), `new_creds`(w,pda), `caller`(ws), `system_program`.
- **`update_role_holder`** `[disc 96,224,166,55,4,62,152,53]` — grant/revoke a holder's role. Args: `address: Pubkey`, `role: Role`, `update: Update`. Accounts: `admin`(ws), `settings`(w,pda), `admin_permissions`(w,pda), `update_admin_permissions`(w), `strategy`(w), `system_program`.
- **`update_action_role`** `[disc 154,250,181,15,202,36,158,230]` — map a `Role` to an `Action`. Args: `action: Action`, `role: Role`, `update: Update`. Accounts: `admin`(ws), `settings`(w,pda), `system_program`, `admin_permissions`(w,pda).

---

## Types

```rust
enum Role   { UNSET, PUBLIC, TESTEE, FREEZE, CRANK, MANAGER, SUPREMO }
enum Action { Restake, Withdraw, Slash, PublicSwap, PrivateSwap,
              FreezeRestake, FreezeWithdraw, FreezeSlash, FreezePublicSwap, FreezePrivateSwap,
              InitializeLiquidityPool, AddAsset, UpdateDepositCap, DepositRewards,
              Management, SuspendDeposits, UpdateRole, UpdateAction }
enum AccessLevel { Public, Private }
enum Oracle      { Pyth, Switchboard }
enum Update      { Add, Remove }
```

## Addresses & Derivations

> Well-known Solana programs (Token, Associated Token, System) are in the [repo readme](../readme.md#common-addresses).

| PDA | Seeds |
| --- | ----- |
| Settings | `["settings"]` |
| LiquidityPool | `["liquidity_pool", index: u8]` |
| Asset | `["asset", asset_mint]` |
| Cooldown | `["cooldown", authority, pool_id, cooldown_id]` |
| UserPermissions | `["permissions", authority]` |
| Pool / cooldown vaults | ATA(owning PDA, asset-or-LP mint) |
