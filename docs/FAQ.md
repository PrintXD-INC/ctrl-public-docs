# FAQ

Short answers for common Control integration questions. For the full spec, see the [Integration Guide](./INTEGRATION.md).

## Which networks is the program deployed to?

Both **Mainnet and Devnet**, at the same program ID:

```
CTRL5CCEQw5zhhBeEV8n5GKZpf3E5tYQoXhhxzUAps27
```

The program binary is identical on both networks. Pick the network via your RPC endpoint — no code change needed.

## How do I detect that a token has graduated?

Read the `curve` PDA (`[b"control-curve", mint]`) before each trade and check `curve.is_completed === 1`. If the flag is set, the curve has migrated and you should route the trade through **Meteora DAMM v2** instead of Control. See [Graduation](./INTEGRATION.md#graduation) and the [`readCurveState`](./INTEGRATION.md#3-read-on-chain-state-curve--config) example.

The curve account itself stays on chain forever — it's frozen and drained at migration time, never closed — so don't treat a `null` `getAccountInfo` response as a graduation signal. A null account means the mint was never launched on Control (or you're querying the wrong cluster), not that it graduated.

`curve.is_frozen === 1` indicates the admin has paused trading on this curve (typically while a migration is in flight). Treat it the same as graduated for routing purposes — `Buy`/`Sell` will revert.

## What is the graduation threshold?

It is **per-token**, stored on-chain as `curve.required_liquidity` (in lamports). Read the curve account at quote time — don't hard-code a value.

Program constants (same on both networks):

- **Default:** `95 SOL` (`HARDCAP_REQUIRED_LIQUIDITY = 95_000_000_000` lamports). This is what new curves get unless overridden.
- **Bounds:** `0.1 SOL ≤ required_liquidity ≤ 10,000 SOL` (`MIN_REQUIRED_LIQUIDITY` / `MAX_REQUIRED_LIQUIDITY`).
- **Per-token override:** set by the program admin at curve creation. Devnet test mints are commonly created with `10,000 SOL` to prevent mid-test graduation.

## How do I implement slippage protection?

Pass a non-zero `min_tokens_out` (Buy) or `min_sol_out` (Sell) when building the instruction. The on-chain program enforces these thresholds and reverts the transaction if the curve cannot deliver. The 3% trading fee is taken from the input — see [Fee structure](./INTEGRATION.md#fee-structure).

Compute the floor from a fresh quote ([`readCurveState`](./INTEGRATION.md#3-read-on-chain-state-curve--config)) and apply your slippage tolerance (e.g. 0.5%) on top.

## Are the IDL `args` blocks populated?

**Yes — populated for every instruction.** Pinocchio parses instruction data manually with `bytemuck` (no derive macro), so a raw shank IDL cannot introspect arg layouts. The published IDL solves this with a post-processor (`scripts/augment_idl.ts` in the on-chain program repo) that hand-rolls the args for each instruction so explorers and Anchor SDKs can decode them. `buy` declares `sol_amount: u64, min_tokens_out: u64`; `sell` declares `amount: u64, min_sol_out: u64` (the IDL uses `amount` as the field name; the [Integration Guide](./INTEGRATION.md#sell) writes it as `token_amount` in prose because it's the more descriptive label — same wire bytes either way).

The full wire format for `Buy` and `Sell` (both 1-byte and 8-byte discriminator forms, plus the args layout) is documented in the [Integration Guide](./INTEGRATION.md#buy). You don't need any other instructions to integrate.

## Where do I find the canonical IDL?

Two equivalent sources:

- This repo: [`idl/control.json`](../idl/control.json).
- Live, on-chain: <https://explorer.solana.com/address/CTRL5CCEQw5zhhBeEV8n5GKZpf3E5tYQoXhhxzUAps27/idl>.

Account orders, instruction names, 8-byte Anchor-style discriminators, and the `TradeEvent` layout are authoritative in both copies.

## Anchor or Pinocchio?

**Pinocchio runtime, dual-mode dispatcher.** The on-chain program is built on Pinocchio (no Anchor framework, no Borsh framing — args are little-endian primitives parsed via `bytemuck`). However, the dispatcher accepts **both** discriminator forms: the legacy 1-byte enum disc (`0x02` for Buy, `0x03` for Sell) **and** the 8-byte Anchor-style disc (`sha256("global:<name>")[..8]`). Old 1-byte clients keep working forever; new clients can use the 8-byte form to get IDL-driven decode in Anchor SDKs and proper rendering on Solscan / SolanaFM. See [Program](./INTEGRATION.md#program).

## Does the program emit structured trade events?

**Yes — as of May 2026, live on both Mainnet and Devnet.** Every `Buy` and `Sell` emits an Anchor self-CPI `TradeEvent` as an inner instruction. The 153-byte payload carries `mint`, `user`, `solAmount`, `tokenAmount`, fees, post-trade reserves, and (for Sell) `solToUser`. Walk `tx.meta.innerInstructions` and decode any 153-byte buffer whose first 8 bytes match the Anchor self-CPI prefix — see [`parseTradeEvents`](./INTEGRATION.md#4-decode-the-on-chain-tradeevent).

## Does Control provide a hosted REST API or SDK?

Not in v1. You must derive PDAs and build instructions client-side using the reference TypeScript in the [Integration Guide](./INTEGRATION.md#typescript-examples). REST metadata endpoints, a WebSocket event stream, and a `/buildTx` helper are planned for later releases.

## Whom do I contact for integration support?

Carlos Beltrán — <cb@print.world>. Reach out if you need a devnet test mint with known reserves, are integrating a listing / aggregator / trading bot, or hit something the docs don't cover.
