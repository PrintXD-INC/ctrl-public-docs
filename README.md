# ctrl-public-docs

Public integration documentation for **Control**, an on-chain bonding-curve token launchpad on Solana.

This repository documents how to integrate **Buy** and **Sell** with the Control program from your own client (DEX aggregator, trading bot, portfolio tool, custom UI).

> Prefer a browsable HTML version? The same docs are mirrored at <https://ctrl.print.world/docs/>.

## Program

| Field | Value |
|---|---|
| Program ID | `CTRL5CCEQw5zhhBeEV8n5GKZpf3E5tYQoXhhxzUAps27` |
| Networks | Mainnet **and** Devnet — same program ID on both |
| Framework | Pinocchio (raw BPF — not Anchor) |
| Token standard | Token-2022 (`TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb`) |
| Instruction discriminator | 8-byte Anchor sighash (`sha256("global:<name>")[..8]`) — what the IDL declares and what Solscan/SolanaFM decode against. Legacy 1-byte enum disc (`0x02` Buy, `0x03` Sell, etc.) is also accepted by the on-chain dispatcher for backward compatibility. See the per-instruction sections below for the full byte arrays. |

## Scope of v1

This first release documents only the two instructions an external integrator needs to call:

- **Buy** — discriminator `0x02` (legacy) or `[102, 6, 61, 18, 1, 218, 235, 234]` (Anchor), swap SOL → token on the bonding curve
- **Sell** — discriminator `0x03` (legacy) or `[51, 230, 133, 164, 1, 127, 131, 173]` (Anchor), swap token → SOL on the bonding curve

The Control program ships **40 instructions** in total. Everything else (token creation, admin, migration, post-migration distribution, buyback engine, daily jackpot, IDL upload) is internal and **integrators do not need to call it**. The full set is enumerated in the on-chain IDL at [`idl/control.json`](./idl/control.json).

> **Graduation safety.** Once a curve fills its `required_liquidity` threshold, Control automatically migrates the token to a Meteora DAMM v2 pool, the curve account is closed, and further `Buy`/`Sell` calls revert. Detect graduation client-side (`curve.is_completed === 1` or curve account missing) and re-route to Meteora DAMM v2 from that point on. See [Graduation](./docs/INTEGRATION.md#graduation).

## Documentation

- [**Integration Guide**](./docs/INTEGRATION.md) — full Buy/Sell reference: PDAs, instruction layouts, bonding-curve math, fee structure, graduation handling, TypeScript examples.
- [**Reference Transactions**](./docs/REFERENCE_TRANSACTIONS.md) — real on-chain Create/Buy/Sell signatures (Devnet + Mainnet), and the parse-only account layout for `CreateToken`.
- [**FAQ**](./docs/FAQ.md) — short answers on graduation, slippage, networks, and where to find the IDL.

## IDL

- [`idl/control.json`](./idl/control.json) — canonical IDL, copied from the on-chain program. 40 instructions, 1 event (`TradeEvent`), populated `args` for every instruction, 8-byte Anchor-style discriminators alongside the legacy 1-byte form. Account orders, instruction names, and event/type layouts are authoritative.
- Live copy on Solana Explorer: <https://explorer.solana.com/address/CTRL5CCEQw5zhhBeEV8n5GKZpf3E5tYQoXhhxzUAps27/idl>

> **Args blocks are now populated.** Pinocchio parses instruction data manually with `bytemuck` (no derive macro, not Borsh), so a raw shank IDL cannot introspect arg layouts. The published IDL solves this with a post-processor (`scripts/augment_idl.ts` in the on-chain program repo) that hand-rolls the args for each instruction so explorers and Anchor SDKs can decode them. `buy` declares `sol_amount: u64, min_tokens_out: u64`; `sell` declares `amount: u64, min_sol_out: u64`. The full wire format for `Buy` and `Sell` is in [Integration Guide → Buy](./docs/INTEGRATION.md#buy) and [→ Sell](./docs/INTEGRATION.md#sell).

## Contact

Integration questions — Carlos Beltrán, <cb@print.world>.
