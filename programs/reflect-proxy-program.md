# reflect-proxy-program — CPI Documentation

> Part of [contract-interfaces](../readme.md). Shared conventions and well-known Solana program addresses live in the repo readme.

## Overview

The Reflect Proxy Program lets integrators launch **branded, fee-bearing stablecoins** backed by a yield-bearing stablecoin (**USDC+**). An integrator creates a *proxy* with their own branded mint and a fee (bps); users **wrap** USDC+ into the branded token and **unwrap** back, while accrued yield is split between the integrator's fee and depositor principal. The program also exposes a protocol-level **USDC ↔ USDC+ stable-swap**.

Two facilities, two discriminator ranges:

| Facility | Instructions | Discriminators | Who calls |
| -------- | ------------ | :------------: | --------- |
| **Proxy** | `create_proxy`, `wrap`, `unwrap`, `claim_fees`, `update_fee`, `update_authority` | `0`–`5` | integrators + their users |
| **Stable-swap** | `swap`, `withdraw` | `100`–`101` | anyone (`swap`) / protocol admin (`withdraw`) |

Built with [Pinocchio](https://github.com/anza-xyz/pinocchio). Amounts are **base units** (USDC / USDC+ / branded tokens are all 6-decimal).

## Program Address

| Environment | Address |
| ----------- | ------- |
| Production | `ProxycrBkRh1S1241nBRkqrJTgTX1ginHDR8xTLjYSE` |

## Source of truth

Discriminators, account orderings, and arguments here are taken from the generated **SDK** (`sdk/rust`, `sdk/ts`) and the Pinocchio program source. Regenerate the SDK and re-derive this doc when the program changes.

## Instruction encoding

Unlike the Anchor programs in this repo, instruction data is **`[discriminator: u8] ++ borsh(args)`** — a **single** leading byte, not an 8-byte Anchor discriminator. Accounts are **positional** (no by-pubkey resolution): pass them in exactly the order each table lists.

## Concepts

- **ProxyState** — per-integrator PDA, seeds `["proxy", branded_mint]`. Raw `repr(C)` account (owner-checked, **no** Anchor discriminator prefix), `LEN = 115`:

  | Offset | Field | Type | Notes |
  | -----: | ----- | ---- | ----- |
  | 0 | `branded_mint` | `Pubkey` | the integrator's branded token |
  | 32 | `stablecoin_mint` | `Pubkey` | backing stablecoin (USDC+) |
  | 64 | `fee` | `u16` (LE) | integrator fee, basis points |
  | 66 | `principal` | `u64` (LE) | depositor principal, in USDC value |
  | 74 | `integrators_commission` | `u64` (LE) | accrued fees, in USDC value |
  | 82 | `authority` | `Pubkey` | integrator authority |
  | 114 | `bump` | `u8` | PDA bump |

- **Proxy vault** — the proxy's USDC+ holdings: the **ATA of `proxy_state`** for the USDC+ mint (standard Associated Token program). Created by the integrator after `create_proxy` (not by the program).
- **Branded mint** — must have **6 decimals**, **mint authority = `proxy_state` PDA**, and **no freeze authority** (enforced by `create_proxy`).
- **Crank** — every state-changing proxy instruction first re-prices the vault via the oracle and books interest: `interest = max(0, vault·price − (principal + commission))`, then `commission += interest·fee/10000` and `principal += interest − that`.
- **Oracle** — the **Doppler** USDC+/USDC price feed, pinned in-program (`PRiceL8ZFBt4C3eKeVXbPmUJ4q5Nc7JN1CMaNfL9EkK`, max staleness 200 slots). _Marked "to change in production."_
- **Protocol authority** — PDA seeds `["auth"]`; owns the stable-swap USDC and USDC+ vaults. The privileged `withdraw` signer is a fixed admin key (`KuBAWM6pZ43ip8b7eB3kjJwUqdPwYpSuoCFkn4bcLgw`). _Both "to change in production."_
- **Virtual offsets** — wrap/unwrap share math uses `VIRTUAL_SHARES = VIRTUAL_ASSETS_USDC = 1_000_000` to blunt first-depositor inflation.

---

## Proxy facility

### `create_proxy` — discriminator `0`

Initialises a ProxyState PDA. Args: `fee: u16` (bps), `bump: u8` (the `proxy_state` PDA bump). The USDC+ vault is created separately by the integrator (e.g. an ATA-create in the same tx).

| # | Account | Writable | Signer | Description |
| -: | ------- | :------: | :----: | ----------- |
| 1 | `authority` | ❌ | ✅ | Integrator authority stored in ProxyState |
| 2 | `payer` | ✅ | ✅ | Funds ProxyState rent |
| 3 | `proxy_state` | ✅ | ❌ | New ProxyState PDA — seeds `["proxy", branded_mint]` |
| 4 | `branded_mint` | ✅ | ❌ | Branded mint (6 dec, mint auth = `proxy_state`, no freeze auth) |
| 5 | `system_program` | ❌ | ❌ | Optional — defaults to `11111111111111111111111111111111` |

### `wrap` — discriminator `1`

Deposit USDC+, mint branded tokens. Cranks first. Args: `amount: u64` (USDC+ in), `min_branded_tokens: u64` (slippage).

| # | Account | Writable | Signer | Description |
| -: | ------- | :------: | :----: | ----------- |
| 1 | `user` | ❌ | ✅ | Depositor |
| 2 | `stablecoin_user_token_account` | ✅ | ❌ | User's USDC+ ATA (source) |
| 3 | `branded_user_token_account` | ✅ | ❌ | User's branded-token ATA (mint destination) |
| 4 | `proxy_state` | ✅ | ❌ | ProxyState PDA |
| 5 | `stablecoin_proxy_state_vault` | ✅ | ❌ | Proxy USDC+ vault (ATA of `proxy_state`) |
| 6 | `branded_mint` | ✅ | ❌ | Branded mint |
| 7 | `oracle` | ❌ | ❌ | Doppler oracle |
| 8 | `token_program` | ❌ | ❌ | SPL Token program |

### `unwrap` — discriminator `2`

Burn branded tokens, withdraw USDC+. Cranks first. Args: `amount: u64` (branded to burn), `min_usdc_plus: u64` (slippage). **Same account layout as `wrap`.**

### `claim_fees` — discriminator `3`

Integrator claims accrued commission as USDC+. Cranks first, then resets commission to zero. **No args.**

| # | Account | Writable | Signer | Description |
| -: | ------- | :------: | :----: | ----------- |
| 1 | `authority` | ❌ | ✅ | Integrator authority (must match `ProxyState.authority`) |
| 2 | `stablecoin_authority_token_account` | ✅ | ❌ | Authority's USDC+ ATA (fee destination) |
| 3 | `proxy_state` | ✅ | ❌ | ProxyState PDA |
| 4 | `stablecoin_proxy_state_vault` | ✅ | ❌ | Proxy USDC+ vault |
| 5 | `oracle` | ❌ | ❌ | Doppler oracle |
| 6 | `token_program` | ❌ | ❌ | SPL Token program |

### `update_fee` — discriminator `4`

Set the integrator fee. Args: `fee: u16` (bps).

| # | Account | Writable | Signer | Description |
| -: | ------- | :------: | :----: | ----------- |
| 1 | `authority` | ❌ | ✅ | ProxyState authority |
| 2 | `proxy_state` | ✅ | ❌ | ProxyState PDA |

### `update_authority` — discriminator `5`

Transfer ProxyState authority. Args: `new_authority: Pubkey`. **Same account layout as `update_fee`.**

---

## Stable-swap facility

### `swap` — discriminator `100`

Swap USDC ↔ USDC+ at the oracle price, less a **0.01%** fee (`SWAP_FEE_BPS = 1`). Args: `amount: u64`, `is_usdc: bool` (`true` = USDC→USDC+, `false` = USDC+→USDC), `min_output_amount: u64` (slippage). The protocol-authority PDA signs the output transfer from its vaults.

| # | Account | Writable | Signer | Description |
| -: | ------- | :------: | :----: | ----------- |
| 1 | `user` | ❌ | ✅ | Swapper |
| 2 | `usdc_user_token_account` | ✅ | ❌ | User's USDC ATA |
| 3 | `usdc_plus_user_token_account` | ✅ | ❌ | User's USDC+ ATA |
| 4 | `authority` | ❌ | ❌ | Protocol authority PDA — seeds `["auth"]` |
| 5 | `usdc_authority_token_account` | ✅ | ❌ | Protocol USDC vault (ATA of `authority`) |
| 6 | `usdc_plus_authority_token_account` | ✅ | ❌ | Protocol USDC+ vault (ATA of `authority`) |
| 7 | `oracle` | ❌ | ❌ | Doppler oracle |
| 8 | `token_program` | ❌ | ❌ | SPL Token program |

### `withdraw` — discriminator `101`

**Permissioned** — the `authority` signer must equal the admin key `KuBAWM6pZ43ip8b7eB3kjJwUqdPwYpSuoCFkn4bcLgw`. Moves `amount` from a protocol vault to the admin (used to rebalance the swap pools). Args: `amount: u64`.

| # | Account | Writable | Signer | Description |
| -: | ------- | :------: | :----: | ----------- |
| 1 | `authority` | ❌ | ✅ | Protocol admin (must equal the pinned admin key) |
| 2 | `authority_token_account` | ✅ | ❌ | Admin's USDC/USDC+ ATA (destination) |
| 3 | `protocol_authority` | ❌ | ❌ | Protocol authority PDA — seeds `["auth"]` |
| 4 | `protocol_authority_token_account` | ✅ | ❌ | Protocol USDC/USDC+ vault (source) |
| 5 | `token_program` | ❌ | ❌ | SPL Token program |

---

## Addresses & Derivations

> Well-known Solana programs (Token, Associated Token, System) are in the [repo readme](../readme.md#common-addresses).

### Pinned addresses

| Name | Address | Notes |
| ---- | ------- | ----- |
| Program | `ProxycrBkRh1S1241nBRkqrJTgTX1ginHDR8xTLjYSE` | |
| USDC+ mint | `usd63SVWcKqLeyNHpmVhZGYAqfE5RHE8jwqjRA2ida2` | backing stablecoin |
| USDC mint | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` | stable-swap counter-asset |
| Doppler oracle | `PRiceL8ZFBt4C3eKeVXbPmUJ4q5Nc7JN1CMaNfL9EkK` | USDC+/USDC price; _to change in production_ |
| Protocol admin | `KuBAWM6pZ43ip8b7eB3kjJwUqdPwYpSuoCFkn4bcLgw` | `withdraw` signer; _to change in production_ |

### PDAs & token accounts

| Account | Derivation |
| ------- | ---------- |
| ProxyState | PDA `["proxy", branded_mint]` |
| Protocol authority | PDA `["auth"]` |
| Proxy USDC+ vault | ATA(`proxy_state`, USDC+ mint) |
| Protocol USDC / USDC+ vaults | ATA(`protocol_authority`, USDC mint) / ATA(`protocol_authority`, USDC+ mint) |
