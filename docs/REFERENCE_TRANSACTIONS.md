# Reference Transactions

Real on-chain examples of each Control instruction, plus the account layout for parsing `CreateToken` (which integrators do **not** invoke — see the callout below).

For the Buy/Sell wire format and TypeScript examples, see the [Integration Guide](./INTEGRATION.md). For shorter answers about graduation, slippage, and the IDL, see the [FAQ](./FAQ.md).

## Table of contents

1. [Reference signatures](#reference-signatures)
2. [Parsing the Create transaction](#parsing-the-create-transaction)

---

## Reference signatures

These are real, successful transactions for each instruction Control exposes today. They are produced by the program author for reference — use them to verify your decoder's account ordering and discriminant byte against ground-truth on-chain data.

### Devnet

| Instruction | Discriminant | Signature |
|---|---|---|
| Create | `0x09` | [`gr4wLuLv7R…5gUVdoU`](https://solscan.io/tx/gr4wLuLv7R5mmy4NZnW6fQXvu43RSfNUYfrH8tjV5CEAK2VP9u6aw5ZF3G4E3FkvNzSsyeGrxotVZdtX5gUVdoU?cluster=devnet) |
| Buy | `0x02` | [`5DBWkW9L8m…FbvHNCW2`](https://solscan.io/tx/5DBWkW9L8mfS2AajeKcmhL2aTy5jBnYLYrNzxowrVPA1dTQFyWhjTDRd3ezeG8WuSwtd3e6E2wM9F1ZVFbvHNCW2?cluster=devnet) |
| Sell | `0x03` | [`2fnFi1iPSS…4KUL7MdE`](https://solscan.io/tx/2fnFi1iPSSQLu7zGCqyvvB3yKUyXcmFh2cePfCjeoUrJrLGofSDSRHgTawXNvryBzcU56BE1MvdoJFSq4KUL7MdE?cluster=devnet) |

### Mainnet

> **Pending — refresh after May 2026 upgrade.** The same self-CPI `TradeEvent` upgrade that already landed on Devnet (and changed Buy/Sell to **14 / 13 accounts**) is rolling out to Mainnet shortly. Old Mainnet sample signatures would be misleading for any new integrator — they use the **pre-upgrade 12 / 11-account layout** that Mainnet itself will move off of. Fresh Mainnet rows will be added to this table the moment the deploy lands; until then, build against the Devnet samples above.

> **Current as of May 2026, live on Devnet, pending Mainnet.** Every Buy/Sell now emits an Anchor self-CPI `TradeEvent` on the inner instructions, and the new Devnet signatures above all carry the event log + use the new **14-account Buy / 13-account Sell** layout. To decode the event payload (exact `solAmount`, `tokenAmount`, fees, post-trade reserves) walk `tx.meta.innerInstructions` — see [`parseTradeEvents`](./INTEGRATION.md#4-decode-the-on-chain-tradeevent) in the Integration Guide. Treat the entries above as ground truth for the **post-upgrade account layout, discriminant byte, and event log shape**.

---

## Parsing the Create transaction

> **Do not invoke `CreateToken`.** It is admin-restricted by design — the `admin` co-signer (account #2 below) gates the instruction, and that will not change. Tokens can only be minted via the Control platform itself. This section exists so **indexers, listing pipelines, and discovery bots** can recognize a Create tx on the wire and extract the new mint without inventing their own decoder.

### Why parse Create?

To detect new launches in real time. Each Create tx introduces a brand-new bonding curve to the platform, and downstream systems (aggregators, trading bots, portfolio trackers) want to start quoting and trading the moment it lands.

### Discriminant

`CreateToken` uses the single byte `0x09` (decimal `9`) as the leading byte of instruction data, consistent with the rest of the program. See [Pinocchio, not Anchor](./INTEGRATION.md#program) for why.

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

Refer to a Solscan-decoded sample tx (Devnet Create signature above) for the current byte layout of the metadata args. A Mainnet sample will be added once the May 2026 self-CPI upgrade lands.

### Detecting new tokens in real time

Subscribe to logs or transactions on the Control program ID and filter by the leading instruction-data byte:

```
Program ID:                CTRL5CCEQw5zhhBeEV8n5GKZpf3E5tYQoXhhxzUAps27
Match condition:           instruction_data[0] == 0x09
```

Each match is a new mint launching on the platform. From there:

1. Read account `#0` to get the new mint pubkey.
2. Derive the curve PDA from the mint (or take account `#4` directly) — see [PDA derivation](./INTEGRATION.md#pda-derivation).
3. Hand the mint to the parsers in the [Integration Guide](./INTEGRATION.md#typescript-examples) and you are ready to quote, buy, or sell.

> **Reminder — parse only.** Do not attempt to construct or send a `CreateToken` transaction from your own client. It will fail without the admin signer, by design.
