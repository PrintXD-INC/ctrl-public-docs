# FAQ

Short answers for common Control integration questions. For the full spec, see the [Integration Guide](./INTEGRATION.md).

## Which networks is the program deployed to?

Both **Mainnet and Devnet**, at the same program ID:

```
CTRL5CCEQw5zhhBeEV8n5GKZpf3E5tYQoXhhxzUAps27
```

The program binary is identical on both networks. Pick the network via your RPC endpoint — no code change needed.

## How do I detect that a token has graduated?

Read the `curve` PDA (`[b"control-curve", mint]`) before each trade and check either of:

- `curve.is_completed === 1` — the curve has migrated.
- `getAccountInfo(curve)` returns `null` — the curve account has been closed.

If either is true, route the trade through **Meteora DAMM v2** instead of Control. See [Graduation](./INTEGRATION.md#graduation) and the [`readCurveState`](./INTEGRATION.md#3-read-the-curve-account-for-live-quotes) example.

`curve.is_frozen === 1` indicates the admin has paused trading on this curve (typically while a migration is in flight). Treat it the same as graduated for routing purposes — `Buy`/`Sell` will revert.

## What is the graduation threshold?

It is **per-token**, stored on-chain as `curve.required_liquidity` (in lamports). Read the curve account at quote time — don't hard-code a value.

Defaults observed today:

- **Devnet:** `0.5 SOL` (`HARDCAP_REQUIRED_LIQUIDITY = 500_000_000` lamports).
- **Mainnet:** `~95 SOL`; the per-token value is set by the program admin at curve creation.

## How do I implement slippage protection?

Pass a non-zero `min_tokens_out` (Buy) or `min_sol_out` (Sell) when building the instruction. The on-chain program enforces these thresholds and reverts the transaction if the curve cannot deliver. The 3% trading fee is taken from the input — see [Fee structure](./INTEGRATION.md#fee-structure).

Compute the floor from a fresh quote ([`readCurveState`](./INTEGRATION.md#3-read-the-curve-account-for-live-quotes)) and apply your slippage tolerance (e.g. 0.5%) on top.

## Why is the IDL `args` block empty for every instruction?

Control is built on **Pinocchio**, which parses instruction data manually with `bytemuck`. There is no derive macro, so Shank cannot introspect arg layouts. This is a permanent design decision, not a pending update.

The wire format for `Buy` (`0x02`) and `Sell` (`0x03`) is documented in the [Integration Guide](./INTEGRATION.md#buy). You don't need any other instructions to integrate.

## Where do I find the canonical IDL?

Two equivalent sources:

- This repo: [`idl/control.json`](../idl/control.json).
- Live, on-chain: <https://explorer.solana.com/address/CTRL5CCEQw5zhhBeEV8n5GKZpf3E5tYQoXhhxzUAps27/idl>.

Account orders, instruction names, and discriminants are authoritative in both copies. The repo file mirrors the on-chain Shank IDL.

## Anchor or Pinocchio?

**Pinocchio.** Instruction data is `[u8 discriminant] [little-endian primitive args...]` — no 8-byte hashed Anchor discriminant, no Borsh framing. See [Program](./INTEGRATION.md#program).

## Does the program emit structured trade events?

**Yes — as of May 2026.** Every `Buy` and `Sell` emits an Anchor self-CPI `TradeEvent` as an inner instruction. The 153-byte payload carries `mint`, `user`, `solAmount`, `tokenAmount`, fees, post-trade reserves, and (for Sell) `solToUser`. Walk `tx.meta.innerInstructions` and decode any 153-byte buffer whose first 8 bytes match the Anchor self-CPI prefix — see [`parseTradeEvents`](./INTEGRATION.md#4-decode-the-on-chain-tradeevent). Live on Devnet, rolling out to Mainnet shortly.

## Does Control provide a hosted REST API or SDK?

Not in v1. You must derive PDAs and build instructions client-side using the reference TypeScript in the [Integration Guide](./INTEGRATION.md#typescript-examples). REST metadata endpoints, a WebSocket event stream, and a `/buildTx` helper are planned for later releases.

## Whom do I contact for integration support?

Carlos Beltrán — <cb@print.world>. Reach out if you need a devnet test mint with known reserves, are integrating a listing / aggregator / trading bot, or hit something the docs don't cover.
