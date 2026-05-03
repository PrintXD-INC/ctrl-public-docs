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
| Instruction discriminant | Single `u8` byte (no Anchor 8-byte hashed disc) |

## Scope of v1

This first release documents only the two instructions an external integrator needs to call:

- **Buy** — discriminant `0x02`, swap SOL → token on the bonding curve
- **Sell** — discriminant `0x03`, swap token → SOL on the bonding curve

The Control program ships 37 instructions in total. Everything else (token creation, admin, migration, post-migration distribution, buyback engine, daily jackpot) is internal and **integrators do not need to call it**. The full set is enumerated in the on-chain Shank IDL at [`idl/control.json`](./idl/control.json).

> **Graduation safety.** Once a curve fills its `required_liquidity` threshold, Control automatically migrates the token to a Meteora DAMM v2 pool, the curve account is closed, and further `Buy`/`Sell` calls revert. Detect graduation client-side (`curve.is_completed === 1` or curve account missing) and re-route to Meteora DAMM v2 from that point on. See [Graduation](./docs/INTEGRATION.md#graduation).

## Documentation

- [**Integration Guide**](./docs/INTEGRATION.md) — full Buy/Sell reference: PDAs, instruction layouts, bonding-curve math, fee structure, graduation handling, TypeScript examples.
- [**Reference Transactions**](./docs/REFERENCE_TRANSACTIONS.md) — real on-chain Create/Buy/Sell signatures (Devnet + Mainnet), and the parse-only account layout for `CreateToken`.
- [**FAQ**](./docs/FAQ.md) — short answers on graduation, slippage, networks, and where to find the IDL.

## IDL

- [`idl/control.json`](./idl/control.json) — canonical Shank IDL, copied from the on-chain program. 37 instructions, account orders, and discriminants.
- Live copy on Solana Explorer: <https://explorer.solana.com/address/CTRL5CCEQw5zhhBeEV8n5GKZpf3E5tYQoXhhxzUAps27/idl>

> Pinocchio parses instruction data manually with `bytemuck` (not Borsh), so Shank cannot introspect arg layouts and the `args` blocks in the IDL are intentionally empty. The wire format for `Buy` and `Sell` is in [Integration Guide → Buy](./docs/INTEGRATION.md#buy) and [→ Sell](./docs/INTEGRATION.md#sell).

## Contact

Integration questions — Carlos Beltrán, <cb@print.world>.
