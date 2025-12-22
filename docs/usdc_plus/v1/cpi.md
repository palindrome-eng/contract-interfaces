# Reflect USDC+ Strategy CPI Documentation

## Overview

This document describes the Cross-Program Invocation interface for the Reflect USDC+ Drift strategy mint/redeem flows using native Solana instructions (i.e., you manually build an `Instruction` with Anchor’s 8-byte discriminator + serialised arguments).

* **Mint**: deposit **USDC**, receive **USDC+** (receipt token representing the share).
* **Redeem**: burn **USDC+**, receive **USDC** back.

Amounts are in **base units** (USDC and USDC+ both use **6 decimals** precision).

## Program address

Protocol program sits at `rFLctqnUuxLmYsW5r9zNujfJx9hGpnP1csXr9PYwVgX`.

## Core Instructions (CPI Targets)

These are the on-chain instruction entrypoints to CPI into:

```rust

pub const MINT_DRIFT_S1_DISCRIMINATOR: [u8; 8] = [119, 79, 157, 143, 109, 89, 78, 100];
pub fn mint_drift_s1<'info>(
    ctx: Context<'_, '_, 'info, 'info, SupplyManagementS1<'info>>,
    usdc_amount: u64,     // amount of USDC to deposit
    min_usdc_amount: u64, // minimum output accepted (slippage protection)
) -> Result<()>

pub const REDEEM_DRIFT_S1_DISCRIMINATOR: [u8; 8] = [204, 97, 69, 120, 27, 255, 6, 43];
pub fn redeem_drift_s1<'info>(
    ctx: Context<'_, '_, 'info, 'info, SupplyManagementS1<'info>>,
    rusd_burn_amount: u64,  // amount of USDC+ to burn
    min_lst_redeem: u64,    // minimum output accepted (slippage protection)
    activity_flag: bool,    // set to false
) -> Result<()>
```

### Arguments

* `min_usdc_amount`: **minimum receipt output** the user is willing to accept for `usdc_amount` input (fails if output < min).
* `min_lst_redeem`: **minimum redeem output** the user is willing to accept when burning `rusd_burn_amount` (fails if output < min).
* `activity_flag`: **fee rejection** must be false or the instruction may fail (fee-protected path).


## Accounts

Both `mint_drift_s1` and `redeem_drift_s1` use the same accounts layout:


### Account list (order + mutability)

When building a raw `Instruction`, pass accounts in this exact order.
If passing `admin_permissions`, set this entry to the reflect program ID (rFLctqnUuxLmYsW5r9zNujfJx9hGpnP1csXr9PYwVgX).

|  # | Account                        | Writable | Signer | Name                                         |  Address                                        |
| -: | ------------------------------ | :------: | :----: | -------------------------------------------- | -------------------------------------------- |
|  1 | `user`                         |     ✅    |    ✅   | User interacting with the USDC+ supply       | signer address                               |
|  2 | `main`                         |     ✅    |    ❌   | Main PDA of Reflect protocol                 | 4BXzppSAgWDHmcN7AwMAmDphJj3BFdbCFo3Sos2Vms6v |   
|  3 | `usdc_controller`              |     ✅    |    ❌   | USDC strategy controller                     | 579cFgopyAezPgYzTyjYa8Gwphfw4YZ1cJADrMLHEPG5 |   
|  4 | `admin_permissions`            |     ❌    |    ❌   | Optional credentials account                 | derived from the signer (or reflect program) |
|  5 | `user_receipt_ata`             |     ✅    |    ❌   | User’s USDC+ token account                   | derived from the signer                      |
|  6 | `user_usdc_ata`                |     ✅    |    ❌   | User’s USDC token account                    | derived from the signer                      |
|  7 | `controller_usdc_ata`          |     ✅    |    ❌   | Controller USDC vault ATA                    | DRZE8YVE3UfsYzfTaPFKaNPtwG5BpYRBRXi6rEtzm3s5 |
|  8 | `usdc_plus_mint`               |     ✅    |    ❌   | USDC+ mint                                   | usd63SVWcKqLeyNHpmVhZGYAqfE5RHE8jwqjRA2ida2  |
|  9 | `drift`                        |     ❌    |    ❌   | Must match Drift program id                  | dRiftyHA39MWEi3m9aunc5MzRF1JYuBsbn6VPcn33UH  |
| 10 | `state`                        |     ✅    |    ❌   | Drift state                                  | 5zpq7DvB6UdFFvpmBPspGPNfUGoBRRCE2HHg5u3gxcsN |
| 11 | `user_stats`                   |     ✅    |    ❌   | Drift user stats                             | GkQmTinf982CB3uoExekVtpNLeU5Vg4K3LAVxKb4ZLY6 |
| 12 | `referrer_user_stats`          |     ✅    |    ❌   | Drift referrer stats                         | 6zKsm3xy9CRwTaBsgRkoYkgSaVarZuaDApfqYq1EanTu |
| 13 | `referrer_user`                |     ✅    |    ❌   | Drift referrer user                          | 9XuTCfYKnecKCw4sSYZXdnR85miLuVWETZSMBRMTNVrj |
| 14 | `user_account`                 |     ✅    |    ❌   | Drift “reflect user”                         | F82oESzqX9fSGf9Sf3SP8PUfCGM9qUvHBAtvhZTqZrvt |
| 15 | `spot_market_vault`            |     ✅    |    ❌   | SPL token account (owned by SPL Token)       | GXWqPpjQpdz7KZw9p7f5PX2eGxHAhvpNXiviFkAB8zXg |
| 16 | `drift_vault`                  |     ✅    |    ❌   | Used by Drift (burn/settlement path)         | JCNCMFXo5M5qwUPg2Utu1u6YWp3MbygxqBsBeXXJfrw  |
| 17 | `token_program`                |     ❌    |    ❌   | SPL Token program                            | TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA  |
| 18 | `system_program`               |     ❌    |    ❌   | System program                               | 11111111111111111111111111111111             |
| 19 | `clock`                        |     ❌    |    ❌   | Sysvar Clock                                 | SysvarC1ock11111111111111111111111111111111  |
| 20 | `usdc_oracle`                  |     ✅    |    ❌   | Lazer oracle for USDC                        | 9VCioxmni2gDLv11qufWzT3RDERhQE4iY5Gf7NTfYyAV |
| 21 | `drift_usdc_spot_market`       |     ✅    |    ❌   | USDC spot market                             | 6gMq3mRCKf8aP3ttTyYhuijVZ2LGi14oDsBbkgubfLB3 |

