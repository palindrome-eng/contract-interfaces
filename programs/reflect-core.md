# Reflect Yield Aggregator — CPI Documentation

> Part of [contract-interfaces](../readme.md). Shared conventions — Anchor discriminators, base units, account ordering, and well-known Solana program addresses — live in the repo readme and are not repeated here.

## Overview

This document describes the Cross-Program Invocation (CPI) interface for the **user-facing** mint / redeem / swap flows of the Reflect yield aggregator (`reflect-core`).

Reflect organises capital into **strategies**. Each strategy is one of two **types**, and the type determines which instructions and account layout you use:

| Strategy type | Receipt tokens (examples) | User instructions | Deposit shape |
| ------------- | ------------------------- | ----------------- | ------------- |
| **Lending**   | USDC+, USDT+              | `mint_lending`, `redeem_lending` | Single SPL in, single SPL out |
| **Bond**      | USTR+ (T-bill basket)     | `mint_bond`, `redeem_bond`, `swap_bond` | Multi-asset basket |

* **Mint**: deposit earn token(s), receive receipt token.
* **Redeem**: burn receipt token, receive earn token(s) back.
* **Swap** (bond only): exchange one basket asset for another at oracle prices.

Yield-venue / basket-asset interactions are passed via **remaining accounts** (see [Remaining Accounts](#remaining-accounts)). Amounts are in **base units** (token-native precision).

> **Scope:** this document covers only the five user-facing instructions. The protocol's admin / operational instructions (strategy deployment, pool-registry management, introspection policy, access control, oracle config, GLV lifecycle) are out of scope here — generate them from the IDL / `reflect-sdk` if needed.

## Program Address

| Environment | Address |
| ----------- | ------- |
| **Mock / dev** (current) | `moCKesf95X6JJQ4ZwyDTnWAQD5tbx5ZpnBnwEY7bFmy` |
| Production | _TBD — not finalised_ |

> ⚠️ The program ID is a **mock** placeholder for now. Replace with the production address before publishing for integrators.

## Source of truth

Discriminators and fixed-account orderings in this document are taken from the generated **`reflect-sdk`** crate (`sdk/rust`, codama-generated from the program IDL) and the on-chain instruction contexts. When the program changes, **regenerate the SDK and re-derive this document** rather than editing account lists by hand.

The legacy `reflect-exchange-rates` crate referenced by earlier versions of this doc is **deprecated and removed** — it is no longer a dependency anywhere in the codebase. Use `reflect-sdk` instruction builders instead.

---

## Core Instructions

### Lending — `mint_lending`

```rust
// Discriminator: [90, 249, 128, 208, 109, 184, 58, 135]
pub fn mint_lending(
    ctx: Context<LendingContext>,
    deposit_amount: u64,        // amount of the strategy's earn SPL to deposit
    min_receipt_receive: u64,   // minimum receipt tokens accepted (slippage protection)
) -> Result<()>
```

### Lending — `redeem_lending`

```rust
// Discriminator: [35, 217, 140, 168, 34, 29, 129, 77]
pub fn redeem_lending(
    ctx: Context<LendingContext>,
    receipt_burn_amount: u64,   // amount of receipt token to burn
    min_redeem: u64,            // minimum earn SPL accepted back (slippage protection)
) -> Result<()>
```

### Bond — `mint_bond`

```rust
// Discriminator: [234, 94, 85, 225, 167, 102, 169, 32]
pub fn mint_bond(
    ctx: Context<BondContext>,
    input_value: u64,           // total USD-denominated value to deposit
    min_output_value: u64,      // minimum receipt tokens accepted (slippage protection)
    transfers: MultiInput,      // per-asset deposit amounts (see Types)
) -> Result<()>
```

### Bond — `redeem_bond`

```rust
// Discriminator: [237, 148, 187, 58, 24, 181, 75, 170]
pub fn redeem_bond(
    ctx: Context<BondContext>,
    input_value: u64,           // receipt token value to burn
    min_output_value: u64,      // minimum output value accepted (slippage protection)
    outputs: MultiInput,        // per-asset withdrawal preferences (see Types)
) -> Result<()>
```

### Bond — `swap_bond`

```rust
// Discriminator: [54, 176, 185, 194, 23, 104, 115, 109]
pub fn swap_bond(
    ctx: Context<BondContext>,
    in_asset_id: u8,            // basket asset id being sold
    in_amount: u64,            // amount of in_asset to sell
    out_asset_id: u8,          // basket asset id being bought
    min_out_amount: u64,       // minimum out_asset accepted (slippage protection)
) -> Result<()>
```

---

## Fixed Accounts

Accounts are resolved **by pubkey**, but for raw `Instruction` construction pass the fixed accounts in the exact order below, followed by the [remaining accounts](#remaining-accounts).

`admin_permissions` is optional: when the caller is not a permissioned admin, set this entry to the Reflect program ID.

### `LendingContext` — 14 accounts (`mint_lending`, `redeem_lending`)

| # | Account | Writable | Signer | Description |
| -: | ------- | :------: | :----: | ----------- |
| 1 | `user` | ✅ | ✅ | User interacting with the strategy |
| 2 | `admin_permissions` | ❌ | ❌ | Optional `UserPermissions` PDA (or program ID if none) |
| 3 | `main` | ✅ | ❌ | Main PDA — seeds `["main"]` |
| 4 | `pool_registry` | ❌ | ❌ | Pool registry PDA — seeds `["pool_registry", 0u8]` |
| 5 | `strategy_controller` | ✅ | ❌ | Strategy controller PDA — seeds `["dex_controller", <strategy_index: u8>]` |
| 6 | `receipt_mint` | ✅ | ❌ | Receipt token mint (e.g. USDC+) — must equal the controller's canonical mint |
| 7 | `user_receipt_ata` | ✅ | ❌ | User's ATA for the receipt mint |
| 8 | `user_earn_ata` | ✅ | ❌ | User's ATA for the earn SPL (e.g. USDC) |
| 9 | `strategy_controller_earn_token_account` | ✅ | ❌ | Controller's ATA for the earn SPL |
| 10 | `token_program` | ❌ | ❌ | SPL Token program |
| 11 | `token_2022_program` | ❌ | ❌ | SPL Token-2022 program |
| 12 | `associated_token_program` | ❌ | ❌ | Associated Token program |
| 13 | `instruction_sysvar_account` | ❌ | ❌ | Instructions sysvar (introspection policy gate) |
| 14 | `system_program` | ❌ | ❌ | System program |

### `BondContext` — 13 accounts (`mint_bond`, `redeem_bond`, `swap_bond`)

| # | Account | Writable | Signer | Description |
| -: | ------- | :------: | :----: | ----------- |
| 1 | `user` | ✅ | ✅ | User interacting with the strategy |
| 2 | `admin_permissions` | ❌ | ❌ | Optional `UserPermissions` PDA (or program ID if none) |
| 3 | `main` | ✅ | ❌ | Main PDA — seeds `["main"]` |
| 4 | `strategy_controller` | ✅ | ❌ | Strategy controller PDA — seeds `["dex_controller", <strategy_index: u8>]` |
| 5 | `receipt_mint` | ✅ | ❌ | Receipt token mint (e.g. USTR+) — must equal the controller's canonical mint |
| 6 | `user_receipt_ata` | ✅ | ❌ | User's ATA for the receipt mint |
| 7 | `asset_registry` | ❌ | ❌ | Asset registry PDA — seeds `["asset_registry"]` |
| 8 | `multiocular` | ❌ | ❌ | MultiOcular oracle-router PDA — seeds `["multiocular"]` |
| 9 | `token_program` | ❌ | ❌ | SPL Token program |
| 10 | `token_program_2022` | ❌ | ❌ | SPL Token-2022 program |
| 11 | `associated_token_program` | ❌ | ❌ | Associated Token program |
| 12 | `instruction_sysvar_account` | ❌ | ❌ | Instructions sysvar (introspection policy gate) |
| 13 | `system_program` | ❌ | ❌ | System program |

> Note: `BondContext` has **no** `pool_registry`, `user_earn_ata`, or `strategy_controller_earn_token_account`. The per-asset token accounts are supplied as remaining accounts instead, and pricing/TVL come from `asset_registry` + `multiocular` + per-asset pool PDAs.

---

## Remaining Accounts

After the fixed accounts, append the venue / asset accounts. **All remaining accounts are matched by pubkey, so their order among themselves does not matter** — but the full required *set* must be present or the instruction fails with `MissingAccount`.

> ⚠️ **Maintenance caveat (option-a manual listing).** The lists below mirror the program's account resolvers (`routing_handler/pool_dispatch.rs`, `oracles/resolver.rs`, `transfer_resolver.rs`) as of this writing. They are **not** auto-generated and will drift if venues/derivations change. There is currently **no off-chain resolver helper** that assembles these for you (the old `reflect-exchange-rates` helper is gone); a caller must derive the set from on-chain `pool_registry` + `CapitalConductor` (lending) or `BondBasket` + `MultiOcular` (bond) state. A resolver helper is the recommended follow-up.

In the tables: **R[i]** = pubkey stored at index `i` of the pool's `CapitalDestination.accounts` registry entry; **derived** = PDA/ATA the caller computes; **const** = well-known program ID. All PDAs are derived against the Reflect program ID unless noted. Writability follows each venue's own CPI requirements.

### Lending remaining accounts

Lending appends one **account group per active pool** in the strategy's capital conductor. For each pool, supply its **TVL** accounts (always required) and, when routing into/out of that pool, its **routing** accounts. Kamino and Save additionally require **refresh** accounts.

<details>
<summary><b>Reserve</b> (exit-hatch vault)</summary>

| Account | Source |
| ------- | ------ |
| `reserve_vault` | derived — seeds `["reserve_vault", <strategy_index: u8>]` |
</details>

<details>
<summary><b>Kamino</b> — registry slots: [0] reserve, [1] lending_market, [2] scope_oracle, [3] liquidity_supply_vault, [4] collateral_mint, [5] collateral_supply_vault, [6] farm_collateral</summary>

**TVL:** `reserve` (R0), `obligation` (derived).
**Refresh:** `reserve` (R0), `lending_market` (R1), `scope_oracle` (R2), `obligation` (derived), `klend_program` (const).
**Routing (15):** `reserve` (R0), `lending_market` (R1), `lending_market_authority` (derived `["lma", lending_market]`), `reserve_liquidity_supply` (R3), `reserve_collateral_mint` (R4), `user_source_liquidity` (derived ATA(controller, earn_mint, Token)), `collateral_supply_vault` (R5), `user_metadata` (derived `["user_meta", controller]`), `obligation` (derived `[[0],[0], controller, lending_market, default, default]`), `earn_mint`, `scope_oracle` (R2), `obligation_farm_state` (derived `["user", farm_collateral, obligation]` @ farms program, or `klend_program` when farms disabled), `reserve_farm_state` (R6), `klend_program` (const), `farms_program` (const).

Programs: `klend_program = KLend2g3cP87fffoy8q1mQqGKjrxjC8boSyAYavgmjD`, `farms_program = FarmsPZpWu9i7Kky8tPN37rs2TpmMrAZrC7S7vJa91Hr`.
</details>

<details>
<summary><b>Jupiter Lend</b> — registry slots: [0] lending_admin, [1] liquidity_owner, [2] lending, [3] supply_token_reserve_liquidity, [4] lending_supply_position, [5] rate_model, [6] vault, [7] rewards_rate_model, [8] claim_account, [9] f_token_mint</summary>

**TVL:** `lending` (R2), `controller_f_token_account` (derived ATA(controller, f_token_mint, Token)).
**Routing (15):** `lending_admin` (R0), `lending` (R2), `supply_token_reserves_liquidity` (R3), `lending_supply_position_on_liquidity` (R4), `rate_model` (R5), `vault` (R6), `claim_account` (R8), `liquidity_owner` (R1), `rewards_rate_model` (R7), `earn_mint`, `f_token_mint` (R9), `controller_earn_account` (derived ATA), `controller_f_token_account` (derived ATA), `lending_program` (const), `liquidity_program` (const).

Programs: `lending_program = jup3YeL8QhtSx1e253b2FDvsMNC87fDrgQZivbrndc9`, `liquidity_program = jupeiUmn818Jg1ekPURTpr4mFo29p46vygyykFJ3wZC`.
</details>

<details>
<summary><b>MarginFi</b> — registry slots: [0] marginfi_group, [1] bank, [2] bank_liquidity_vault, [3] marginfi_account, [4] bank_oracle</summary>

**TVL:** `marginfi_account` (R3), `bank` (R1).
**Routing (7):** `marginfi_group` (R0), `marginfi_account` (R3), `bank` (R1), `bank_liquidity_vault` (R2), `bank_liquidity_vault_authority` (derived `["liquidity_vault_auth", bank]`), `bank_oracle` (R4), `marginfi_program` (const).

Program: `marginfi_program = MFv2hWf31Z9kbCa1snEPYctwafyhdvnV7FZnsebVacA`.
</details>

<details>
<summary><b>Save</b> (Solend fork) — registry slots: [0] reserve, [1] lending_market, [2] reserve_liquidity_supply, [3] reserve_collateral_mint, [4] reserve_collateral_supply, [5] pyth_oracle, [6] switchboard_oracle, [7] obligation</summary>

**TVL:** `reserve` (R0), `obligation` (R7).
**Refresh:** `reserve` (R0), `pyth_oracle` (R5), `switchboard_oracle` (R6), `save_program` (const).
**Routing (13):** `reserve` (R0), `lending_market` (R1), `lending_market_authority` (derived `[lending_market]`), `reserve_liquidity_supply` (R2), `reserve_collateral_mint` (R3), `reserve_collateral_supply` (R4), `pyth_oracle` (R5), `switchboard_oracle` (R6), `obligation` (R7), `earn_mint`, `controller_liquidity_ata` (derived ATA), `controller_collateral_ata` (derived ATA), `save_program` (const).

Program: `save_program = So1endDq2YkqhipRh3WViPa8hdiSpxWy6z3Z6tMCpAo`.
</details>

<details>
<summary><b>Loopscale</b> — registry slots: [0] vault, [1] strategy, [2] market_information, [3] lp_mint</summary>

**TVL:** `user_lp_ta` (derived ATA(controller, lp_mint, Token-2022)), `vault_stake_lp_ta` (derived ATA, optional pre-stake), `strategy` (R1), `lp_mint` (R3).
**Routing (14 + cosigner):** `loopscale_program` (const), `event_authority` (derived `["__event_authority"]`), `vault` (R0), `strategy` (R1), `market_information` (R2), `lp_mint` (R3), `user_lp_ta` (derived ATA), `strategy_principal_ta` (derived ATA(strategy, principal_mint)), `principal_token_program` (const, resolved from principal mint owner), `loopscale_nonce` (derived `["loopscale_nonce", controller, vault]`), `vault_stake` (derived `["vault_stake", loopscale_nonce, vault]` @ loopscale), `vault_stake_lp_ta` (derived ATA), `vault_rewards_info` (derived `["vault_rewards_info", vault]`), `user_rewards_info` (derived `["user_rewards_info", vault_stake]`), plus the Loopscale **cosigner `bs_auth`** which **must be a signer on the transaction**.

Program: `loopscale_program = 1oopBoJG58DgkUVKkEzKgyG9dvRmpgeEm1AVjoHkF78`. Only the 4-account (vaults) shape is wired today; MFI-backed Loopscale pools are rejected.
</details>

<details>
<summary><b>GLV</b> (GMX-Solana, async settlement) — registry slots: [0] store, [1] token_map, [2] glv, [3] glv_token_mint, [4] primary_market, [5] market_token_mint, [6] short_token (USDC), [7] long_token, [8] strategy_glv_ata, [9] glv_pool_state</summary>

GLV is **asynchronous**: a user `mint_lending` / `redeem_lending` only *initiates* a deposit/withdrawal; settlement happens out-of-band via admin `glv_settle_pending` / `glv_cancel_stale`. The account set is **dynamic** — it includes, in addition to the registry slots:

* `glv_pool_state` (R9, Reflect-owned) and `strategy_glv_ata` (R8, Token-2022).
* Every `market` / `market_token` / price `feed` listed in the on-chain `GlvPoolState` registry auxiliary.
* For mint: `glv_deposit` PDA + its three escrow ATAs (glv_token Token-2022, market_token, initial_short_token/USDC), the per-pool `proxy_owner` PDA (`["glv_proxy", glv_pool_state]`), `gmsol_store_program` (const), `glv_token_program` (Token-2022).
* For redeem: `glv_withdrawal` PDA + four escrow ATAs (glv_token, market_token, final_long_token, final_short_token/USDC), same proxy/program accounts.
* Every occupied pending-deposit / pending-withdrawal action PDA (for TVL valuation).

Program: `gmsol_store_program = Gmso1uvJnLbawvw7yezdfCDcPydwW2s2iqG3w6MDucLo`. Because the set is read from live `GlvPoolState`, assemble it from on-chain state rather than statically.
</details>

### Bond remaining accounts

Bond instructions price and value the basket from on-chain state. For **each asset in the basket**, supply:

| Account | Source | Purpose |
| ------- | ------ | ------- |
| Oracle feed(s) | from `MultiOcular` `PairRoutes.oracles[].oracle_address` for the asset→USD (fallback asset→USDC) pair | price resolution |
| `market_pool` PDA | derived — seeds `["market_pool", controller, bond_mint]` | basket TVL / volume |
| `issuer_pool` PDA | derived — seeds `["issuer_pool", controller, bond_mint]` | (issuer-venue flows) |
| `user_ata` | derived — ATA(user, bond_mint) — Token or Token-2022, whichever exists | transfer in/out |
| `controller_ta` | derived — controller's seeded vault PDA or ATA(controller, bond_mint) | transfer in/out |

`mint_bond` / `redeem_bond` move the assets named in the `transfers` / `outputs` `MultiInput`; `swap_bond` moves only the `in_asset_id` / `out_asset_id` pair. Only the touched assets' transfer ATAs are required, but **all** basket assets' oracle + pool accounts are needed for valuation.

---

## Types

```rust
// Per-asset amount for bond mint/redeem.
pub struct TokenInput {
    pub asset_id: u8,   // basket asset id (see Asset Registry)
    pub amount: u64,    // max amount of this asset to spend (mint) / prefer (redeem)
}

pub struct MultiInput {
    pub inputs: Vec<TokenInput>,
}
```

---

## Addresses & Derivations

> Well-known Solana programs (Token, Token-2022, ATA, Instructions sysvar, System) are in the [repo readme](../readme.md#common-addresses).

### Token mints

| Token | Mint | Notes |
| ----- | ---- | ----- |
| USDC | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` | earn token (Strategy 1) |
| USDT | `Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB` | earn token (Strategy 2) |
| USDC+ | `usd63SVWcKqLeyNHpmVhZGYAqfE5RHE8jwqjRA2ida2` | receipt — _verify against current deploy_ |
| USDT+ | `uSDtYeMVYuQwhziLKMpdMz74WPFNytoWLGGiU9SDnZx` | receipt — _verify against current deploy_ |
| USTR+ | _TBD_ | bond receipt — not finalised |

### Reflect PDAs

| PDA | Seeds |
| --- | ----- |
| Main | `["main"]` |
| Pool Registry | `["pool_registry", 0u8]` |
| Strategy Controller | `["dex_controller", <strategy_index: u8>]` |
| Asset Registry | `["asset_registry"]` |
| MultiOcular | `["multiocular"]` |
| Reserve Vault | `["reserve_vault", <strategy_index: u8>]` |
| Market Pool (bond) | `["market_pool", <controller>, <bond_mint>]` |
| Issuer Pool (bond) | `["issuer_pool", <controller>, <bond_mint>]` |
