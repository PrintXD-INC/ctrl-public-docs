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
| Create | `0x09` | [`3oHKPanxoP…1KWnQa2c`](https://solscan.io/tx/3oHKPanxoPohptipTB2DEBKwRRAiAwdn1whhSdNoc9ZywAAvmeGBo5FEB2tVDMXmAZsr88FhpjXvtGGA1KWnQa2c?cluster=devnet) |
| Buy | `0x02` | [`2ed1XR61MY…67VwMmbh`](https://solscan.io/tx/2ed1XR61MY9NMD3YUCHvFe7br1bkcgNamRmumYVtZqW917NsgijfZpzAhEKGzmiAvWp5jCLf4n3rVVGW67VwMmbh?cluster=devnet) |
| Sell | `0x03` | [`2wavMJs6R1…9DJUzHP`](https://solscan.io/tx/2wavMJs6R16XHoSuPbDoEP2xfvLvrTZEKN152bT9QJgQHVjRSgeCZSd95jscj9tpuYe1go4AiBVV8QBe9DJUzHP?cluster=devnet) |

### Mainnet

| Instruction | Discriminant | Signature |
|---|---|---|
| Create | `0x09` | [`5n3LuV6BMz…MM8DZ3oo`](https://solscan.io/tx/5n3LuV6BMzxa3ea7Gokm6cRaTwK8iH5jrmPH3stCYGvSrBZs4mvmUi7cZ8rxpzM16AgWGgLTvnGjDqseMM8DZ3oo) |
| Buy | `0x02` | [`5H5QqqYaqc…Tn9bDXW6`](https://solscan.io/tx/5H5QqqYaqcNks9oZQvAysNVX75SLUA78iPsquaUV8gLf7g5YCfowGWjw3KmXBYyH4FmZaUWgzEdBmPfgTn9bDXW6) |
| Sell | `0x03` | [`5Yo5JVA5dS…tdno43tv`](https://solscan.io/tx/5Yo5JVA5dSp8cFdvW2u2ouHo7wVcqPQHfbCuDcR3h5KXExmmUzY4ijBHCYLf1JRCveR5rYRZv4n591VQtdno43tv) |

> **Snapshot, not a frozen contract.** These signatures are a current snapshot. The program author is rolling out a change to include full CPI data logs in every transaction, so future Sell txs will surface the SOL swap value via inner-instruction logs (today you read it from the SOL balance delta — see [`parseTradeOutcome`](./INTEGRATION.md#4-confirm-trade-results-from-a-transaction)). When that lands the table will be refreshed; treat current entries as ground truth for the **account layout and discriminant byte**, and re-pull a fresh tx after the upgrade for any **log-based parsing**.

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

Refer to a Solscan-decoded sample tx (Devnet or Mainnet Create signatures above) for the current byte layout of the metadata args, and re-pull after the CPI-logs upgrade for a structured representation.

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
