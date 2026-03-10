# Reflect Yield Aggregator CPI Documentation

## Overview

This document describes the Cross-Program Invocation interface for the Reflect yield aggregator mint/redeem flows for **Strategy 1 (USDC+)** and **Strategy 2 (USDT+)**.

Both strategies share the same accounts layout and differ only in which deposit token (USDC vs USDT), strategy controller and strategy controller token accounts they reference. Yield venue interactions (Drift, Jupiter Lend, Kamino, etc.) are passed via **remaining accounts**.

* **Mint**: deposit earn token (USDC or USDT), receive receipt token (USDC+ or USDT+).
* **Redeem**: burn receipt token (USDC+ or USDT+), receive earn token back.

Amounts are in **base units** (all tokens use **6 decimals** precision).

## Program Address

| Environment | Address |
| ----------- | ------- |
| Production  | `rFLctqnUuxLmYsW5r9zNujfJx9hGpnP1csXr9PYwVgX` |


## Core Instructions

### Strategy 1 — USDC+

```rust
// Discriminator: [0x70, 0xd1, 0x1b, 0x44, 0x08, 0x37, 0xb3, 0xf8]
pub fn mint_strategy1<'info>(
    ctx: Context<'_, '_, 'info, 'info, YieldAggregator<'info>>,
    usdc_deposit_amount: u64,   // amount of USDC to deposit
    min_receipt_receive: u64,   // minimum USDC+ output accepted (slippage protection)
) -> Result<()>

// Discriminator: [0xd7, 0x95, 0xdf, 0x3a, 0xad, 0x5d, 0x5a, 0xc3]
pub fn redeem_strategy1<'info>(
    ctx: Context<'_, '_, 'info, 'info, YieldAggregator<'info>>,
    receipt_burn_amount: u64,   // amount of USDC+ to burn
    min_spl_redeem: u64,        // minimum USDC output accepted (slippage protection)
) -> Result<()>
```

### Strategy 2 — USDT+

```rust
// Discriminator: [0xe5, 0x40, 0x2d, 0x61, 0x89, 0xb8, 0xd3, 0xc8]
pub fn mint_strategy2<'info>(
    ctx: Context<'_, '_, 'info, 'info, YieldAggregator<'info>>,
    usdt_deposit_amount: u64,   // amount of USDT to deposit
    min_receipt_receive: u64,   // minimum USDT+ output accepted (slippage protection)
) -> Result<()>

// Discriminator: [0x60, 0xea, 0xf6, 0x8c, 0x1f, 0x0c, 0x19, 0xda]
pub fn redeem_strategy2<'info>(
    ctx: Context<'_, '_, 'info, 'info, YieldAggregator<'info>>,
    receipt_burn_amount: u64,   // amount of USDT+ to burn
    min_spl_redeem: u64,        // minimum USDT output accepted (slippage protection)
) -> Result<()>
```

### Arguments

| Argument | Description |
| -------- | ----------- |
| `usdc_deposit_amount` / `usdt_deposit_amount` | Amount of earn token to deposit (mint flow). |
| `min_receipt_receive` | Minimum receipt tokens the user is willing to accept. Transaction fails if output is below this. |
| `receipt_burn_amount` | Amount of receipt tokens (USDC+ / USDT+) to burn (redeem flow). |
| `min_spl_redeem` | Minimum earn tokens the user is willing to receive back. Transaction fails if output is below this. |

---

## Accounts

All four instructions (`mint_strategy1`, `redeem_strategy1`, `mint_strategy2`, `redeem_strategy2`) use the same `YieldAggregator` accounts layout. When building a raw `Instruction`, pass accounts in this exact order.

If passing `admin_permissions` as `None`, set this entry to the Reflect program ID (`rFLctqnUuxLmYsW5r9zNujfJx9hGpnP1csXr9PYwVgX`).

| # | Account | Writable | Signer | Description |
| -: | ------- | :------: | :----: | ----------- |
| 1 | `user` | ✅ | ✅ | User interacting with the strategy |
| 2 | `admin_permissions` | ❌ | ❌ | Optional credentials account (or program ID if none) |
| 3 | `main` | ✅ | ❌ | Main PDA — seeds: `["main"]`, program ID |
| 4 | `strategy_controller` | ✅ | ❌ | Strategy controller PDA — seeds: `["dex_controller", <strategy_index>]`, program ID |
| 5 | `receipt_mint` | ✅ | ❌ | Receipt token mint (USDC+ or USDT+) |
| 6 | `user_receipt_ata` | ✅ | ❌ | User's receipt token ATA (derived from user + receipt mint) |
| 7 | `user_earn_ata` | ✅ | ❌ | User's earn token ATA (derived from user + USDC/USDT mint) |
| 8 | `strategy_controller_earn_token_account` | ✅ | ❌ | Controller's earn token ATA (derived from controller + USDC/USDT mint) |
| 9 | `token_program` | ❌ | ❌ | SPL Token program — `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA` |
| 10 | `associated_token_program` | ❌ | ❌ | Associated Token program — `ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL` |
| 11 | `instruction_sysvar_account` | ❌ | ❌ | Sysvar Instructions — `Sysvar1nstructions1111111111111111111111111` |
| 12 | `system_program` | ❌ | ❌ | System program — `11111111111111111111111111111111` |

### Remaining Accounts

All accounts related to the underlying yield venues (Drift, Jupiter Lend, Kamino, etc.) are passed as **remaining accounts** after the 12 fixed accounts above. These are strategy-specific and include protocol state, market accounts, vaults, oracles, and program IDs for each venue the capital conductor routes through. 

**Accounts in the remianing can be provided in any order.**

The remaining accounts can be obtained using the [`reflect-exchange-rates`](https://crates.io/crates/reflect-exchange-rates) library:

**USDC+ (Strategy 1):**
```rust
use reflect_exchange_rates::accounts_api::{
    get_supply_change_accounts_usdc,
    get_supply_change_accounts_usdc_with_config,
};
```

**USDT+ (Strategy 2):**
```rust
use reflect_exchange_rates::accounts_api::{
    get_supply_change_accounts_usdt,
    get_supply_change_accounts_usdt_with_config,
};
```


## Addresses

### Token Mints

| Token | Mint Address |
| ----- | ------------ |
| USDC  | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` |
| USDT  | `Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB` |
| USDC+  | `usd63SVWcKqLeyNHpmVhZGYAqfE5RHE8jwqjRA2ida2` |
| USDT+  | `uSDtYeMVYuQwhziLKMpdMz74WPFNytoWLGGiU9SDnZx` |


### Strategy Indexes

| Strategy | Index | Earn Token | Receipt Token |
| -------- | :---: | ---------- | ------------- |
| Strategy 1 | `0` | USDC | USDC+ |
| Strategy 2 | `1` | USDT | USDT+ |

### PDA Derivation

| PDA | Seeds |
| --- | ----- |
| Main | `["main"]` + program ID |
| Strategy Controller | `["dex_controller", <strategy_index as u8>]` + program ID |
