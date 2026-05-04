# Reference Transactions

Real on-chain examples of each Control instruction, plus the account layout for parsing `CreateToken` (which integrators do **not** invoke — see the callout below).

For the Buy/Sell wire format and TypeScript examples, see the [Integration Guide](./INTEGRATION.md). For shorter answers about graduation, slippage, and the IDL, see the [FAQ](./FAQ.md).

## Table of contents

1. [Reference signatures](#reference-signatures)
2. [Test mints](#test-mints)
3. [Parsing the Create transaction](#parsing-the-create-transaction)

---

## Reference signatures

These are real, successful transactions for each instruction Control exposes today. They are produced by the program author for reference — use them to verify your decoder's account ordering, discriminator, and inner-instruction TradeEvent shape against ground-truth on-chain data.

> The **Discriminator** column shows the legacy 1-byte form alongside the 8-byte Anchor form for human reference. Every sample transaction linked here was actually signed with the **8-byte Anchor form** (`sha256("global:<name>")[..8]`); raw decoded `instruction.data` will start with the 8-byte prefix, not the legacy single byte. Both forms work on-chain; the dispatcher routes them to the same handler.

### Devnet

| Instruction | Discriminator | Signature |
|---|---|---|
| Create | `0x09` (legacy) / `[84, 52, 204, 228, 24, 140, 234, 75]` (Anchor) | [`qNUbbvq2vf…hZh6pDfJ`](https://solscan.io/tx/qNUbbvq2vfJBPieUKztMWcJ31kgJPZiD9iHSBmpGLBPcPnxyEdPYMKh6TU3127i1x1CLSZgZPDPB3FQhZh6pDfJ?cluster=devnet) |
| Buy | `0x02` (legacy) / `[102, 6, 61, 18, 1, 218, 235, 234]` (Anchor) | [`2XMdkUgaaY…ZGjkJNiaGStv`](https://solscan.io/tx/2XMdkUgaaY9mbTT4jKefZLFTP3bUfCqSEUU9kPHKoDzwa6zx9k9GRregdALKCdge6aw2Qexh2q72ZGjkJNiaGStv?cluster=devnet) |
| Sell | `0x03` (legacy) / `[51, 230, 133, 164, 1, 127, 131, 173]` (Anchor) | [`5Bn3Wijq8R…CJTbvr7cE`](https://solscan.io/tx/5Bn3Wijq8RRqfNYjfSFSEVb1BqP9eWQouNZJRS547WVzatiFQw5hhgphvvkrbeAqP64itLLFeBCjWFcCJTbvr7cE?cluster=devnet) |

### Mainnet

| Instruction | Discriminator | Signature |
|---|---|---|
| Create | `0x09` (legacy) / `[84, 52, 204, 228, 24, 140, 234, 75]` (Anchor) | [`5aMjqtnmzr…FQ3dUYAe`](https://solscan.io/tx/5aMjqtnmzrKvXAZRq2daDe1Mj755273dRqz2iviy48hgtjWys9rTvLxzUKxMv5F7ew8SgHFAKV1kJEbiFQ3dUYAe) |
| Buy | `0x02` (legacy) / `[102, 6, 61, 18, 1, 218, 235, 234]` (Anchor) | [`2yBpVSUqDt…Z4SN6iX4`](https://solscan.io/tx/2yBpVSUqDt7dz7kx2ZCst9toqECBWqb4v81pDCsJy8uTh2t5P2yvgDtCg9siHJuiNjv8iZPD8DLJBjLwZ4SN6iX4) |
| Sell | `0x03` (legacy) / `[51, 230, 133, 164, 1, 127, 131, 173]` (Anchor) | [`2bEG5cx9Ch…NL1humAKQ`](https://solscan.io/tx/2bEG5cx9ChPoGh9pf7tSj78NQzF1UYHweJMSHgSqCbaq4RcQo1gpPKUBSgeoNfVJuaYEbHpjaiS2jSkNL1humAKQ) |

> **Same wire format on both networks.** Mainnet and Devnet share the Program ID `CTRL5CCEQw5zhhBeEV8n5GKZpf3E5tYQoXhhxzUAps27` and the identical post-May-2026 layout: **14-account Buy / 13-account Sell**, dual-mode discriminator dispatch (legacy 1-byte or 8-byte Anchor-style), and the self-CPI `TradeEvent` emitted as an inner instruction on every trade. The same client code works against either network — pick the cluster via your RPC endpoint, no code changes required.

> **Current as of May 2026 — live on both Mainnet and Devnet.** All six sample signatures above were produced with the 8-byte Anchor-style discriminator and carry the event log under the new account layout. To decode the event payload (exact `solAmount`, `tokenAmount`, fees, post-trade reserves) walk `tx.meta.innerInstructions` — see [`parseTradeEvents`](./INTEGRATION.md#4-decode-the-on-chain-tradeevent) in the Integration Guide. Treat the entries above as ground truth for the **account layout, both discriminator forms, and event log shape**.

---

## Test mints

Canonical mints used by the sample transactions above and by integration tests. Hand any of these to [`readCurveState`](./INTEGRATION.md#3-read-on-chain-state-curve--config) and the rest of the [Integration Guide](./INTEGRATION.md#typescript-examples) to walk through a full Buy/Sell round-trip without producing your own mint.

| Network | Mint | Notes |
|---|---|---|
| Devnet (canonical) | `6XyiLwNWGwkuWunt2XX7uiMfJJDgw8wdeowVnPxmizRe` | Created with `required_liquidity = 10,000 SOL` so it stays pre-graduation across long-running test runs. Use this for any new integration work. |
| Devnet (legacy) | `8poC3bFzLNZPuzvSRNiuEDACsW5Ymtw7HNKWX55JgavW` | Mint backing the Devnet Create / Buy / Sell sample signatures linked above. |
| Mainnet | `2awKV3D3r8T6jLKsyHTzjsG2hMtNbsW2vL4gfGsTVXsw` | Mint backing the Mainnet Create / Buy / Sell sample signatures linked above. |

> Need a fresh devnet mint with custom reserves or a non-default `required_liquidity`? Email <cb@print.world>.

---

## Parsing the Create transaction

> **Do not invoke `CreateToken`.** It is admin-restricted by design — the `admin` co-signer (account #2 below) gates the instruction, and that will not change. Tokens can only be minted via the Control platform itself. This section exists so **indexers, listing pipelines, and discovery bots** can recognize a Create tx on the wire and extract the new mint without inventing their own decoder.

### Why parse Create?

To detect new launches in real time. Each Create tx introduces a brand-new bonding curve to the platform, and downstream systems (aggregators, trading bots, portfolio trackers) want to start quoting and trading the moment it lands.

### Discriminator

`CreateToken` is dispatched by either form:

- **Legacy 1-byte:** `0x09` (decimal `9`) as the leading byte of instruction data.
- **Anchor-style 8-byte:** `[84, 52, 204, 228, 24, 140, 234, 75]` (= `sha256("global:create_token")[..8]`).

Both route to the same on-chain handler. See [Pinocchio runtime, dual-mode dispatch](./INTEGRATION.md#program) for the dispatcher contract.

### Account layout

Pulled directly from [`idl/control.json`](../idl/control.json) (instruction index 9 — `CreateToken`). The order is positional and stable.

| #  | Account             | Role                  | Notes |
|----|---------------------|-----------------------|-------|
| 0  | `mint`              | writable + signer     | The new Token-2022 mint. One-shot keypair signer. |
| 1  | `payer`             | writable + signer     | Token creator; pays rent + create-fee. |
| 2  | `admin`             | readonly + signer     | Control admin co-signer. Required — gates the instruction. |
| 3  | `config`            | PDA, readonly         | Control config PDA. |
| 4  | `curve`             | PDA, writable         | Control curve PDA, initialized in this tx. |
| 5  | `protocolFeeWallet` | writable              | Receives the create-fee. |
| 6  | `vaultAta`          | writable              | Curve vault token account (Token-2022). |
| 7  | `lpEscrow`          | PDA, writable         | LP escrow PDA seeded for this mint. |
| 8  | `creatorFeeVault`   | PDA, writable         | Per-mint creator fee vault PDA. |
| 9  | `systemProgram`     | readonly              | System program. |
| 10 | `token2022Program`  | readonly              | Token-2022 program. |
| 11 | `ataProgram`        | readonly              | Associated Token Program. |

### What an integrator typically extracts

| Source | Field |
|---|---|
| Account 0 | New mint pubkey |
| Account 1 | Creator (token launcher) wallet |
| Account 4 | Curve PDA — start watching for trades from here |
| Instruction data | Token-2022 metadata args (name, symbol, URI) |

Refer to a Solscan-decoded sample tx (either Create signature in the tables above) for the current byte layout of the metadata args. Mainnet and Devnet share the same wire format, so either reference tx is authoritative.

### Detecting new tokens in real time

Subscribe to logs or transactions on the Control program ID and filter on either discriminator form:

```
Program ID:                CTRL5CCEQw5zhhBeEV8n5GKZpf3E5tYQoXhhxzUAps27
Match condition (legacy):  instruction_data[0] == 0x09
Match condition (Anchor):  instruction_data[0..8] == [84, 52, 204, 228, 24, 140, 234, 75]
```

Either filter is sufficient on its own — the dispatcher routes both forms to the same handler, so you do not need to match both. Pick the form your indexing pipeline already understands.

Each match is a new mint launching on the platform. From there:

1. Read account `#0` to get the new mint pubkey.
2. Derive the curve PDA from the mint (or take account `#4` directly) — see [PDA derivation](./INTEGRATION.md#pda-derivation).
3. Hand the mint to the parsers in the [Integration Guide](./INTEGRATION.md#typescript-examples) and you are ready to quote, buy, or sell.

> **Reminder — parse only.** Do not attempt to construct or send a `CreateToken` transaction from your own client. It will fail without the admin signer, by design.
