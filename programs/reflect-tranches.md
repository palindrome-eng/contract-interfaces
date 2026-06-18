# reflect-tranches — Orchestration Guide

> Part of [contract-interfaces](../readme.md). Unlike the other entries, this is **not a program** — it is a client/orchestration layer, so it has no program ID or CPI surface of its own. It composes two programs documented here: [reflect-proxy-program](./reflect-proxy-program.md) and [rlp](./rlp.md).

## What it is

Reflect Tranches turns any yield-bearing stablecoin (e.g. **USDC+**) into **senior** and **junior** yield tranches by orchestrating two on-chain programs from the client (`@reflectmoney/tranches` TS SDK + a crank service):

| Tranche | Program | Token | Position |
| ------- | ------- | ----- | -------- |
| **Senior** | [reflect-proxy-program](./reflect-proxy-program.md) (whitelabel) | branded mint (e.g. `seniorUSDC+`) | insured, stable yield |
| **Junior** | [rlp](./rlp.md) (insurance fund) | pool LP token | amplified yield, first-loss |

A senior holder gets reduced-but-protected yield; a junior holder earns their own yield **plus** the senior's redirected fee, but absorbs losses first via RLP slashing.

## How it composes

**Yield routing (periodic crank):**
1. The Proxy splits USDC+ yield by its fee (e.g. `3000` bps = 30% to the integrator, 70% to senior principal).
2. Crank calls Proxy [`claim_fees`](./reflect-proxy-program.md) → receives the 30% as USDC+.
3. Crank calls RLP [`deposit_rewards`](./rlp.md) → raises junior value-per-LP (no new LP minted).

**Loss / slash (on a loss event):**
1. Crank calls RLP [`slash`](./rlp.md) (≤ 10% of pool per tx) → extracts USDC+ from the junior pool.
2. Crank transfers that USDC+ directly into the Proxy vault ATA → restores senior backing. The Proxy's crank attributes it to principal on the next `wrap`/`unwrap`/`claim_fees`.

**User flows:**
- Senior: [`wrap`](./reflect-proxy-program.md) to deposit, [`unwrap`](./reflect-proxy-program.md) to redeem (instant).
- Junior: [`restake`](./rlp.md) to deposit; withdrawal is two-step — [`request_withdrawal`](./rlp.md) then [`withdraw`](./rlp.md) after the cooldown (LP stays slashable during cooldown).

## Yield modes

| Mode | Senior yield | Junior yield |
| ---- | ------------ | ------------ |
| **SeniorFloat** (default) | `B × (1 − fee)` | `B × (1 + R × fee)` |
| **SeniorFixed** | fixed `X` | `B + R × (B − X)` (can go negative) |

where `B` = base asset APY, `fee` = proxy fee fraction, `R` = senior/junior TVL ratio. Junior is amplified by the redirected senior fee, scaled by the TVL ratio. Full formulas, break-even, and loss-impact tables are in the program repo's README and the `tranche-rate-sdk` (Rust) / simulator.

## Components (in the `reflect-tranches` repo)

| Path | Role |
| ---- | ---- |
| `package/` (`@reflectmoney/tranches`) | TS orchestration SDK — wires Proxy + RLP + vaults |
| `tranche-rate-sdk/` | Rust senior/junior APY math |
| `cli/` | deploy + crank + status commands |
| `simulator/` | APY / loss / historical modeling |

## Operational notes

- **Crank authority** — the Proxy's `authority` must be (or delegate to) the crank wallet so it can `claim_fees`; the crank wallet also needs the RLP **CRANK** role for `deposit_rewards` / `slash`.
- **Slash cap** — RLP caps slashing per tx; large losses require multiple `slash` transactions.
- **Oracles** — the Proxy uses a Doppler feed, RLP uses Pyth/Switchboard; both must track the same USDC+ price.
- **Cooldown exposure** — junior LP remains slashable through the withdrawal cooldown (by design — insurance can't be front-run).
