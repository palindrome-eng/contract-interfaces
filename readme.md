# Palindrome / Reflect — Contract Interfaces

Cross-Program Invocation (CPI) interface documentation for Palindrome's on-chain programs.

Each program is documented in its own file under [`programs/`](./programs). This root file holds
the index and the conventions shared across all of them.

## Programs

| Program | Interface doc | Status |
| ------- | ------------- | ------ |
| `reflect-core` | [programs/reflect-core.md](./programs/reflect-core.md) | ✅ user-facing flows documented |
| `reflect-proxy-program` | [programs/reflect-proxy-program.md](./programs/reflect-proxy-program.md) | 🚧 stub |
| `insurance` | [programs/insurance.md](./programs/insurance.md) | 🚧 stub |
| `kamino-pool-wrapper` | [programs/kamino-pool-wrapper.md](./programs/kamino-pool-wrapper.md) | ⏸ deferred — not yet public |

## Conventions

These apply to every program doc unless that file states otherwise.

### Instruction discriminators

All programs are Anchor-based. Each instruction is dispatched by an 8-byte discriminator,
`sha256("global:<snake_case_name>")[..8]`. The discriminator values shown in each doc are taken
from that program's generated client SDK / Anchor IDL — treat the SDK/IDL as authoritative.

### Source of truth & regeneration

Account orderings and discriminators are derived from each program's generated SDK / IDL.
When a program changes, **regenerate its SDK/IDL and re-derive its doc** — do not hand-edit
account lists. Each doc names the specific SDK/IDL it was generated from.

### Amounts

All token amounts are in **base units** (token-native precision / decimals). No implicit scaling.

### Account ordering

For raw `Instruction` construction, pass each instruction's fixed accounts in the order its doc
lists. Some programs additionally resolve dynamic ("remaining") accounts **by pubkey**, making
those order-insensitive among themselves — each doc calls this out where it applies.

### Optional accounts

Where an account is optional (e.g. an admin/permissions account), the doc says so. The common
Anchor convention is to pass the program's own ID in that slot when the account is absent.

## Common addresses

Well-known Solana programs referenced across the docs:

| Program | Address |
| ------- | ------- |
| SPL Token | `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA` |
| SPL Token-2022 | `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb` |
| Associated Token | `ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL` |
| Instructions sysvar | `Sysvar1nstructions1111111111111111111111111` |
| System | `11111111111111111111111111111111` |

## Adding a program

Copy the structure of an existing doc under `programs/` (e.g. [reflect-core.md](./programs/reflect-core.md)).
Standard skeleton:

1. **Overview** — what the program does, who calls it, key concepts.
2. **Program Address** — per-environment addresses.
3. **Source of truth** — which SDK/IDL the doc is generated from.
4. **Instructions** — signature + discriminator + args for each entry point.
5. **Fixed Accounts** — ordered table (writable / signer / description) per instruction context.
6. **Remaining Accounts** — if the program resolves dynamic accounts.
7. **Types** — any custom argument structs.
8. **Addresses & Derivations** — program-specific mints, PDAs, integrated-program IDs.

Then add a row to the [Programs](#programs) table above.
