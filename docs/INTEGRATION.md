# Control — Integration Guide

How to execute **Buy** and **Sell** on the Control program from your own client.

## Table of contents

1. [Overview](#overview)
2. [Scope of v1](#scope-of-v1)
3. [Program](#program)
4. [PDA derivation](#pda-derivation)
5. [Buy](#buy)
6. [Sell](#sell)
7. [Bonding curve math](#bonding-curve-math)
8. [Fee structure](#fee-structure)
9. [Graduation](#graduation)
10. [TypeScript examples](#typescript-examples)

---

## Overview

**Control** is a bonding-curve token launchpad on Solana. Each token trades against a deterministic AMM curve embedded in the on-chain program. Every trade adjusts the curve's reserves; price is a pure function of those reserves.

When a curve accumulates its **required liquidity threshold** in real reserves it *graduates*: the remaining supply and pooled SOL are migrated into a **Meteora DAMM v2** pool, and the token continues life as a standard AMM asset. The threshold is configured per-token in the curve state (`curve.required_liquidity`) — currently `0.5 SOL` on devnet, `~95 SOL` on mainnet. Before graduation, all trading goes through Control. After graduation, route through Meteora DAMM v2.

> **In scope.** Integrators only call `Buy` and `Sell`. Token creation and admin instructions are out of scope for this guide.
>
> **Who signs trades.** The end user's wallet — Control is fully non-custodial.

---

## Scope of v1

This document covers only what you need to trade. Advanced features ship later.

**In scope (v1)**

| Topic | Description |
|---|---|
| Buy | Swap SOL → token on the bonding curve |
| Sell | Swap token → SOL on the bonding curve |
| PDA derivation | Client-side PDAs needed to build both instructions |
| Curve math | AMM formula + fee breakdown for quoting |

**Out of scope (future)**

| Topic | Description |
|---|---|
| REST API | Token metadata & market-data endpoints |
| WebSocket | New-token and graduation events |
| `/buildTx` endpoint | Server-side transaction construction |
| SDK | TypeScript / Rust client library |

> **Discovery is your responsibility in v1.** Until the public API ships, you are expected to discover Control mints via your own Solana RPC indexing (for example, subscribing to the program's account updates) or via the dev team.

---

## Program

| Field | Value |
|---|---|
| Networks | Mainnet **and** Devnet — same program ID on both |
| Program ID | `CTRL5CCEQw5zhhBeEV8n5GKZpf3E5tYQoXhhxzUAps27` |
| Framework | Pinocchio (raw BPF — *not* Anchor) |
| Instruction disc | **Dual-mode**: legacy `u8` (single byte) **or** 8-byte Anchor-style `sha256("global:<name>")[..8]` |
| Token program | `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb` (Token-2022) |
| System program | `11111111111111111111111111111111` |

> **Pinocchio runtime, dual-mode dispatch.** Instruction data starts with the discriminant followed by little-endian primitive args (parsed manually via `bytemuck`, not Borsh — no Anchor framework on-chain). The dispatcher tries the **8-byte Anchor-style discriminator** first (`sha256("global:<name>")[..8]`), then falls back to the **legacy 1-byte enum disc**. Both forms route to the same handler, so existing 1-byte clients keep working forever and new clients can use the 8-byte form to get IDL-driven decode in Anchor SDKs / Solscan / SolanaFM. Each instruction below lists both forms.

---

## PDA derivation

All PDAs are derived deterministically from the token mint. Compute them client-side — no off-chain calls required.

The program's trading state is keyed by the token `mint`. Once you know the mint, you can derive every account needed for `Buy` and `Sell` in pure code.

### Accounts you must derive

| Account | Seeds | Program |
|---|---|---|
| `config` | `[b"control-config"]` | Control |
| `curve` | `[b"control-curve", mint]` | Control |
| `vaultAta` | `getAssociatedTokenAddressSync(mint, curve, true, TOKEN_2022_PROGRAM_ID)` | ATA (Token-2022) |
| `userAta` | `getAssociatedTokenAddressSync(mint, user, false, TOKEN_2022_PROGRAM_ID)` | ATA (Token-2022) |
| `creatorFeeVault` | `[b"creator-fees", mint, creator]` | Control |
| `lpEscrow` | `[b"lp-escrow", mint]` | Control |
| `communityPool` | `[b"community-pool", mint]` | Control |
| `eventAuthority` | `[b"__event_authority"]` | Control |

> `eventAuthority` is global to the program (not per-mint) and is required on every `Buy` and `Sell` for the self-CPI `TradeEvent` emission. The bump is precomputed (`255`) and hardcoded in the program — derive it once and cache it.

> **Token-2022 ATAs.** Pass the Token-2022 program ID, not the legacy SPL Token program. The `vaultAta` uses `allowOwnerOffCurve = true` because `curve` is a PDA, not a keypair.

### Reference implementation

```ts
import { PublicKey } from '@solana/web3.js';
import {
  getAssociatedTokenAddressSync,
  TOKEN_2022_PROGRAM_ID,
} from '@solana/spl-token';

export const CONTROL_PROGRAM_ID = new PublicKey(
  'CTRL5CCEQw5zhhBeEV8n5GKZpf3E5tYQoXhhxzUAps27',
);

export function deriveConfig(): PublicKey {
  const [pda] = PublicKey.findProgramAddressSync(
    [Buffer.from('control-config')],
    CONTROL_PROGRAM_ID,
  );
  return pda;
}

export function deriveCurve(mint: PublicKey): PublicKey {
  const [pda] = PublicKey.findProgramAddressSync(
    [Buffer.from('control-curve'), mint.toBuffer()],
    CONTROL_PROGRAM_ID,
  );
  return pda;
}

export function deriveCreatorFeeVault(
  mint: PublicKey,
  creator: PublicKey,
): PublicKey {
  const [pda] = PublicKey.findProgramAddressSync(
    [Buffer.from('creator-fees'), mint.toBuffer(), creator.toBuffer()],
    CONTROL_PROGRAM_ID,
  );
  return pda;
}

export function deriveLpEscrow(mint: PublicKey): PublicKey {
  const [pda] = PublicKey.findProgramAddressSync(
    [Buffer.from('lp-escrow'), mint.toBuffer()],
    CONTROL_PROGRAM_ID,
  );
  return pda;
}

export function deriveCommunityPool(mint: PublicKey): PublicKey {
  const [pda] = PublicKey.findProgramAddressSync(
    [Buffer.from('community-pool'), mint.toBuffer()],
    CONTROL_PROGRAM_ID,
  );
  return pda;
}

// Anchor self-CPI signer for TradeEvent emission. Global, not per-mint.
// Required on every Buy and Sell.
export function deriveEventAuthority(): PublicKey {
  const [pda] = PublicKey.findProgramAddressSync(
    [Buffer.from('__event_authority')],
    CONTROL_PROGRAM_ID,
  );
  return pda;
}

export function deriveVaultAta(mint: PublicKey): PublicKey {
  const curve = deriveCurve(mint);
  return getAssociatedTokenAddressSync(
    mint,
    curve,
    true, // allowOwnerOffCurve — curve is a PDA
    TOKEN_2022_PROGRAM_ID,
  );
}
```

---

## Buy

Swap SOL for tokens on the bonding curve.

**Discriminator:** `0x02` (legacy) **or** `[102, 6, 61, 18, 1, 218, 235, 234]` (Anchor) · **Phase:** pre-graduation only · **Accounts:** 14

> **Breaking change — May 2026.** Every `Buy` now mandatorily emits an Anchor self-CPI `TradeEvent` for explorer + indexer compatibility. Clients **must** include two extra accounts at the end of the account list (slots 12 and 13 — `eventAuthority` + `program`). Old callers that send 12 accounts will fail with `NotEnoughAccountKeys`. The upgrade is live on **both Mainnet and Devnet** as of May 2026.

> **Dual-mode discriminator.** The on-chain dispatcher accepts both forms: the legacy 1-byte disc (`0x02`) or the 8-byte Anchor-style disc (`sha256("global:buy")[..8]` = `[102, 6, 61, 18, 1, 218, 235, 234]`). Old clients keep working; Anchor SDK clients get IDL-driven decode for free. Solscan / SolanaFM render the swap as `Buy` because they recognize the 8-byte form. The TS examples below use the 1-byte form for brevity — to switch to Anchor-style, replace the leading byte with the 8-byte buffer; everything else (accounts, fields, behavior) is identical.

### Args (little-endian primitives, in order after the discriminant byte)

| Name | Type | Description |
|---|---|---|
| `sol_amount` | `u64` | SOL to spend, in lamports. Includes the fee portion. |
| `min_tokens_out` | `u64` | Minimum tokens the user will accept. Use as slippage protection — the program will revert if the curve cannot deliver at least this amount. |

### Accounts (in order)

| # | Name | Flags | Description |
|---|---|---|---|
| 0 | `user` | signer, writable | End user buying tokens. Pays SOL and fees. |
| 1 | `config` | PDA, readonly | Program config — seeds `[b"control-config"]`. |
| 2 | `curve` | PDA, writable | Control curve state — seeds `[b"control-curve", mint]`. Updated on every trade. |
| 3 | `mint` | readonly | Token-2022 mint. |
| 4 | `vaultAta` | ATA, writable | Curve's associated token account for `mint` (Token-2022). Owner is the curve PDA. |
| 5 | `userAta` | ATA, writable | End user's associated token account for `mint` (Token-2022). Must exist — create it atomically in the same transaction if needed. |
| 6 | `creatorFeeVault` | PDA, writable | Receives the creator's slice — seeds `[b"creator-fees", mint, creator]`. |
| 7 | `protocolFeeWallet` | writable | Protocol fee wallet (address available from config). |
| 8 | `lpEscrow` | PDA, writable | Accumulates the LP-migration slice — seeds `[b"lp-escrow", mint]`. |
| 9 | `communityPool` | PDA, writable | Per-token community pool — seeds `[b"community-pool", mint]`. Receives both the base community fee (`config.community_fee_bps`) and the per-token addon (`curve.extra_community_fee_bps`). |
| 10 | `token2022Program` | readonly | `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb` |
| 11 | `systemProgram` | readonly | `11111111111111111111111111111111` |
| 12 | `eventAuthority` | PDA, readonly | Anchor self-CPI signer — seeds `[b"__event_authority"]`, bump `255` (precomputed). Required so the program can self-CPI back into itself with a `TradeEvent` log that Solscan / SolanaFM decode into a "Swap N for M SOL" UI. |
| 13 | `program` | readonly | `CTRL5CCEQw5zhhBeEV8n5GKZpf3E5tYQoXhhxzUAps27` — the Control program account itself, required so the runtime can route the inner-CPI back into Control. |

### Instruction data layout

```text
[ 0x02 ] [ sol_amount : u64 LE ] [ min_tokens_out : u64 LE ]
   1   +          8           +            8           = 17 bytes
```

---

## Sell

Swap tokens for SOL on the bonding curve.

**Discriminator:** `0x03` (legacy) **or** `[51, 230, 133, 164, 1, 127, 131, 173]` (Anchor) · **Phase:** pre-graduation only · **Accounts:** 13

> **Breaking change — May 2026.** Every `Sell` now mandatorily emits an Anchor self-CPI `TradeEvent` for explorer + indexer compatibility. Clients **must** include two extra accounts at the end of the account list (slots 11 and 12 — `eventAuthority` + `program`). Old callers that send 11 accounts will fail with `NotEnoughAccountKeys`. The upgrade is live on **both Mainnet and Devnet** as of May 2026.

> **Dual-mode discriminator.** The dispatcher accepts both forms: the legacy 1-byte disc (`0x03`) or the 8-byte Anchor-style disc (`sha256("global:sell")[..8]` = `[51, 230, 133, 164, 1, 127, 131, 173]`). Old clients keep working; Anchor SDK clients get IDL-driven decode for free. The TS examples below use the 1-byte form for brevity — to switch to Anchor-style, replace the leading byte with the 8-byte buffer; everything else (accounts, fields, behavior) is identical.

### Args (little-endian primitives, in order after the discriminant byte)

| Name | Type | Description |
|---|---|---|
| `token_amount` | `u64` | Tokens to sell, in base units (respect the mint's decimals). |
| `min_sol_out` | `u64` | Minimum SOL (lamports) the user will accept after fees. |

### Accounts (in order)

Sell takes **13 accounts** — the first 10 match [Buy](#buy)'s first 10 exactly. `system_program` is *not* required (the program credits SOL by mutating PDA lamports directly), so the self-CPI accounts sit at slots **11 and 12** here vs **12 and 13** in Buy, but otherwise carry the same values. The `user` pays tokens and receives SOL; fee accounts still receive their slices (paid out of the proceeds).

| # | Name | Flags |
|---|---|---|
| 0 | `user` | signer, writable |
| 1 | `config` | PDA, readonly |
| 2 | `curve` | PDA, writable |
| 3 | `mint` | readonly |
| 4 | `vaultAta` | ATA, writable |
| 5 | `userAta` | ATA, writable |
| 6 | `creatorFeeVault` | PDA, writable |
| 7 | `protocolFeeWallet` | writable |
| 8 | `lpEscrow` | PDA, writable |
| 9 | `communityPool` | PDA, writable |
| 10 | `token2022Program` | readonly |
| 11 | `eventAuthority` | PDA, readonly |
| 12 | `program` | readonly |

### Instruction data layout

```text
[ 0x03 ] [ token_amount : u64 LE ] [ min_sol_out : u64 LE ]
   1   +          8            +           8           = 17 bytes
```

> **Do not call `Sell` after graduation.** Once the curve has migrated to Meteora DAMM v2, the curve account is closed and the program will revert. Check `curve.is_completed === 1` before routing, or detect the missing curve account and fall back to Meteora DAMM v2.

---

## Bonding curve math

Constant-product AMM with virtual reserves. Use this to pre-compute quotes before submitting transactions.

### Formula

Control uses a standard `x * y = k` pool, except the reserves are the **sum** of virtual and real reserves. Virtual reserves set the initial price and smooth early trades; real reserves track actual SOL and tokens held by the curve.

```text
// BUY — user sends sol_in lamports, receives tokens_out
total_sol   = virtual_sol_reserve   + real_sol_reserve
total_token = virtual_token_reserve + real_token_reserve

tokens_out  = (total_token * sol_in) / (total_sol + sol_in)

// SELL — user sends token_in units, receives sol_out (before fees)
sol_out     = (total_sol * token_in) / (total_token + token_in)
```

### Initial reserves

| Parameter | Value |
|---|---|
| `virtual_sol_reserve` | `162 SOL` |
| `virtual_token_reserve` | `≈ 1,364,600,000 tokens` |
| Scale factor | `e / ln(2)` |
| `real_sol_reserve` (launch) | `0 SOL` |
| `real_token_reserve` (launch) | total supply deposited in the curve's vault |

> **Quoting tip.** Read the `curve` account before every quote to get the *current* real reserves; virtual reserves are constant per-mint. Apply the fee structure below to the input before feeding it into the formula.

---

## Fee structure

A 3% total is split across four destinations on every trade.

| Slice | Rate | Destination | Source | Purpose |
|---|---|---|---|---|
| Creator | `0.15%` | `creatorFeeVault` | `config.creator_fee_bps` | Accrues to the token creator. Claimable via a separate instruction. |
| Protocol | `0.60%` | `protocolFeeWallet` | `config.protocol_fee_bps` | Protocol fee accrued to the protocol treasury. |
| LP escrow | `0.25%` | `lpEscrow` | `config.lp_fee_bps` | Reserved for seeding the Meteora pool at graduation. |
| Community (base) | `1.00%` | `communityPool` | `config.community_fee_bps` | Per-token community pool — global rate. |
| Community (extra) | `1.00%` | `communityPool` | `curve.extra_community_fee_bps` | Per-token addon stored in the curve state. Routes to the same `communityPool` PDA. |
| **Total** | **3.00%** | | | |

Fees are taken from the *input* side. For a Buy, fees come out of `sol_amount` before tokens are priced; for a Sell, fees come out of the proceeds after the AMM computes `sol_out`. The contract sums `config` base fees plus `curve.extra_community_fee_bps` on every trade — read both accounts when computing exact quotes.

---

## Graduation

When real SOL reserves reach the curve's required liquidity, Control migrates the token to Meteora DAMM v2.

The graduation threshold is **per-token**, stored on-chain as `curve.required_liquidity`. Read the curve account at quote time — don't hard-code a value:

- **Devnet:** currently `0.5 SOL` (program constant `HARDCAP_REQUIRED_LIQUIDITY = 500_000_000` lamports — used as the default when curves are created).
- **Mainnet:** `~95 SOL`; the final per-token value is set by the program admin.

Graduation is automatic and atomic: the Control program seeds a Meteora pool with the accumulated `lpEscrow` SOL and the residual tokens in the curve's vault, then marks the curve completed. After graduation:

- Buy / Sell on Control will **revert** (the curve sets `is_completed = 1`).
- The token continues life as a regular Meteora DAMM v2 position.
- Route subsequent trades through Meteora or any aggregator that supports it.

> **Detect graduation before each trade.** A robust integration reads the `curve` account and checks `is_completed` (or detects the account is closed), then falls back to Meteora DAMM v2. Also check `is_frozen` — the admin can pause trading on a curve.

---

## TypeScript examples

End-to-end snippets for the four things every integrator needs: build a Buy transaction, build a Sell transaction, read the curve account for live quotes, and confirm what a trade actually delivered on-chain.

> **Discriminator form used below.** The `buildBuyTx` and `buildSellTx` examples encode the **legacy 1-byte** discriminant (`0x02` / `0x03`) for brevity. To use the **8-byte Anchor-style** form instead, swap the leading byte for the 8-byte buffer — e.g. replace `buf.writeUInt8(BUY_DISCRIMINANT, 0)` with `Buffer.from([102, 6, 61, 18, 1, 218, 235, 234]).copy(buf, 0)` and bump the buffer size by 7. Account list, args, and on-chain behavior are identical. Either form works; the dispatcher tries Anchor first and falls back to legacy.

### 1. Build a Buy transaction

Derive every PDA / ATA from the mint, encode the instruction data, attach an idempotent ATA-creation instruction, and return an unsigned `Transaction` ready for the user's wallet to sign.

```ts
import {
  Connection,
  PublicKey,
  Transaction,
  TransactionInstruction,
  SystemProgram,
  Keypair,
  sendAndConfirmTransaction,
} from '@solana/web3.js';
import {
  TOKEN_2022_PROGRAM_ID,
  getAssociatedTokenAddressSync,
  createAssociatedTokenAccountIdempotentInstruction,
} from '@solana/spl-token';
import {
  CONTROL_PROGRAM_ID,
  deriveConfig,
  deriveCurve,
  deriveCreatorFeeVault,
  deriveLpEscrow,
  deriveCommunityPool,
  deriveVaultAta,
  deriveEventAuthority,
} from './pdas';

const BUY_DISCRIMINANT = 2;

function encodeBuyData(solAmount: bigint, minTokensOut: bigint): Buffer {
  const buf = Buffer.alloc(1 + 8 + 8);
  buf.writeUInt8(BUY_DISCRIMINANT, 0);
  buf.writeBigUInt64LE(solAmount, 1);
  buf.writeBigUInt64LE(minTokensOut, 9);
  return buf;
}

export async function buildBuyTx(params: {
  connection: Connection;
  user: PublicKey;
  mint: PublicKey;
  creator: PublicKey;
  protocolFeeWallet: PublicKey;
  solAmountLamports: bigint;
  minTokensOut: bigint;
}): Promise<Transaction> {
  const {
    connection,
    user,
    mint,
    creator,
    protocolFeeWallet,
    solAmountLamports,
    minTokensOut,
  } = params;

  // 1. Derive every PDA / ATA needed.
  const config = deriveConfig();
  const curve = deriveCurve(mint);
  const vaultAta = deriveVaultAta(mint);
  const creatorFeeVault = deriveCreatorFeeVault(mint, creator);
  const lpEscrow = deriveLpEscrow(mint);
  const communityPool = deriveCommunityPool(mint);
  const eventAuthority = deriveEventAuthority();
  const userAta = getAssociatedTokenAddressSync(
    mint,
    user,
    false,
    TOKEN_2022_PROGRAM_ID,
  );

  // 2. Ensure the user's Token-2022 ATA exists in the same tx.
  const ataIx = createAssociatedTokenAccountIdempotentInstruction(
    user,
    userAta,
    user,
    mint,
    TOKEN_2022_PROGRAM_ID,
  );

  // 3. Build the Buy instruction (14 accounts — last two are for self-CPI TradeEvent).
  const buyIx = new TransactionInstruction({
    programId: CONTROL_PROGRAM_ID,
    keys: [
      { pubkey: user,                    isSigner: true,  isWritable: true  },
      { pubkey: config,                  isSigner: false, isWritable: false },
      { pubkey: curve,                   isSigner: false, isWritable: true  },
      { pubkey: mint,                    isSigner: false, isWritable: false },
      { pubkey: vaultAta,                isSigner: false, isWritable: true  },
      { pubkey: userAta,                 isSigner: false, isWritable: true  },
      { pubkey: creatorFeeVault,         isSigner: false, isWritable: true  },
      { pubkey: protocolFeeWallet,       isSigner: false, isWritable: true  },
      { pubkey: lpEscrow,                isSigner: false, isWritable: true  },
      { pubkey: communityPool,           isSigner: false, isWritable: true  },
      { pubkey: TOKEN_2022_PROGRAM_ID,   isSigner: false, isWritable: false },
      { pubkey: SystemProgram.programId, isSigner: false, isWritable: false },
      { pubkey: eventAuthority,          isSigner: false, isWritable: false },
      { pubkey: CONTROL_PROGRAM_ID,      isSigner: false, isWritable: false },
    ],
    data: encodeBuyData(solAmountLamports, minTokensOut),
  });

  const tx = new Transaction().add(ataIx, buyIx);
  tx.feePayer = user;
  tx.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
  return tx;
}

// Usage — signer is the end user's wallet (Phantom, Backpack, etc.).
// For Node, use a Keypair:
async function example() {
  const connection = new Connection('https://api.devnet.solana.com', 'confirmed');
  const user = Keypair.generate(); // replace with wallet keypair

  const tx = await buildBuyTx({
    connection,
    user: user.publicKey,
    mint: new PublicKey('<CONTROL_TOKEN_MINT>'),
    creator: new PublicKey('<CREATOR_PUBKEY>'),
    protocolFeeWallet: new PublicKey('<PROTOCOL_FEE_WALLET>'), // from config.protocol_fee_wallet
    solAmountLamports: 100_000_000n, // 0.1 SOL
    minTokensOut: 1n, // set based on a pre-trade quote + slippage
  });

  const sig = await sendAndConfirmTransaction(connection, tx, [user]);
  console.log('Buy landed:', sig);
}
```

### 2. Build a Sell transaction

Sell is symmetric to Buy. Discriminant flips to `0x03`, `sol_amount`/`min_tokens_out` become `token_amount`/`min_sol_out`, and the account list is **13 accounts**: Buy's first 10 + `token2022Program` + `eventAuthority` + `program`. `system_program` is *not* required, so the self-CPI accounts sit at slots **11 and 12** instead of **12 and 13** as in Buy. The user's ATA must already exist and hold the tokens being sold.

```ts
import {
  Connection,
  PublicKey,
  Transaction,
  TransactionInstruction,
} from '@solana/web3.js';
import {
  TOKEN_2022_PROGRAM_ID,
  getAssociatedTokenAddressSync,
} from '@solana/spl-token';
import {
  CONTROL_PROGRAM_ID,
  deriveConfig,
  deriveCurve,
  deriveCreatorFeeVault,
  deriveLpEscrow,
  deriveCommunityPool,
  deriveVaultAta,
  deriveEventAuthority,
} from './pdas';

const SELL_DISCRIMINANT = 3;

function encodeSellData(tokenAmount: bigint, minSolOut: bigint): Buffer {
  const buf = Buffer.alloc(1 + 8 + 8);
  buf.writeUInt8(SELL_DISCRIMINANT, 0);
  buf.writeBigUInt64LE(tokenAmount, 1);
  buf.writeBigUInt64LE(minSolOut, 9);
  return buf;
}

export async function buildSellTx(params: {
  connection: Connection;
  user: PublicKey;
  mint: PublicKey;
  creator: PublicKey;
  protocolFeeWallet: PublicKey;
  tokenAmount: bigint;     // raw base units (respects mint decimals)
  minSolOut: bigint;       // lamports, slippage-protected floor
}): Promise<Transaction> {
  const {
    connection,
    user,
    mint,
    creator,
    protocolFeeWallet,
    tokenAmount,
    minSolOut,
  } = params;

  // 1. Derive PDAs / ATAs.
  const config = deriveConfig();
  const curve = deriveCurve(mint);
  const vaultAta = deriveVaultAta(mint);
  const creatorFeeVault = deriveCreatorFeeVault(mint, creator);
  const lpEscrow = deriveLpEscrow(mint);
  const communityPool = deriveCommunityPool(mint);
  const eventAuthority = deriveEventAuthority();
  const userAta = getAssociatedTokenAddressSync(
    mint,
    user,
    false,
    TOKEN_2022_PROGRAM_ID,
  );

  // 2. Build the Sell instruction (13 accounts — last two are for self-CPI TradeEvent).
  const sellIx = new TransactionInstruction({
    programId: CONTROL_PROGRAM_ID,
    keys: [
      { pubkey: user,                  isSigner: true,  isWritable: true  },
      { pubkey: config,                isSigner: false, isWritable: false },
      { pubkey: curve,                 isSigner: false, isWritable: true  },
      { pubkey: mint,                  isSigner: false, isWritable: false },
      { pubkey: vaultAta,              isSigner: false, isWritable: true  },
      { pubkey: userAta,               isSigner: false, isWritable: true  },
      { pubkey: creatorFeeVault,       isSigner: false, isWritable: true  },
      { pubkey: protocolFeeWallet,     isSigner: false, isWritable: true  },
      { pubkey: lpEscrow,              isSigner: false, isWritable: true  },
      { pubkey: communityPool,         isSigner: false, isWritable: true  },
      { pubkey: TOKEN_2022_PROGRAM_ID, isSigner: false, isWritable: false },
      { pubkey: eventAuthority,        isSigner: false, isWritable: false },
      { pubkey: CONTROL_PROGRAM_ID,    isSigner: false, isWritable: false },
    ],
    data: encodeSellData(tokenAmount, minSolOut),
  });

  const tx = new Transaction().add(sellIx);
  tx.feePayer = user;
  tx.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
  return tx;
}
```

> **Do not call `Sell` after graduation.** Once the curve migrates to Meteora DAMM v2 the curve account is closed and the program reverts. Detect `is_completed === 1` on the curve account (or a missing curve account) and route through Meteora DAMM v2 instead.

### 3. Read the curve account for live quotes

The `curve` account is a 344-byte `repr(C, packed)` struct. Decode the four reserve fields plus `is_completed` / `required_liquidity` and feed them straight into the constant-product formula from [Bonding curve math](#bonding-curve-math). Read this account *before every quote* — virtual reserves are constant per-mint, but real reserves shift on every trade.

```ts
import { Connection, PublicKey } from '@solana/web3.js';
import { deriveCurve } from './pdas';

// Field offsets inside the 344-byte ControlCurve account (repr(C, packed)).
const OFFSET_VIRTUAL_SOL    = 64;
const OFFSET_VIRTUAL_TOKEN  = 72;
const OFFSET_REAL_SOL       = 80;
const OFFSET_REAL_TOKEN     = 88;
const OFFSET_REQUIRED_LIQ   = 112;
const OFFSET_IS_COMPLETED   = 120;
const OFFSET_IS_FROZEN      = 122;

export interface CurveState {
  virtualSolReserve: bigint;
  virtualTokenReserve: bigint;
  realSolReserve: bigint;
  realTokenReserve: bigint;
  requiredLiquidity: bigint;   // graduation threshold (lamports)
  isCompleted: boolean;        // true once migrated to Meteora DAMM v2
  isFrozen: boolean;           // true while migration is in flight
}

export async function readCurveState(
  connection: Connection,
  mint: PublicKey,
): Promise<CurveState | null> {
  const info = await connection.getAccountInfo(deriveCurve(mint));
  if (!info) return null; // closed → already graduated, route to Meteora DAMM v2

  const data = info.data;
  return {
    virtualSolReserve:   data.readBigUInt64LE(OFFSET_VIRTUAL_SOL),
    virtualTokenReserve: data.readBigUInt64LE(OFFSET_VIRTUAL_TOKEN),
    realSolReserve:      data.readBigUInt64LE(OFFSET_REAL_SOL),
    realTokenReserve:    data.readBigUInt64LE(OFFSET_REAL_TOKEN),
    requiredLiquidity:   data.readBigUInt64LE(OFFSET_REQUIRED_LIQ),
    isCompleted: data[OFFSET_IS_COMPLETED] === 1,
    isFrozen:    data[OFFSET_IS_FROZEN]    === 1,
  };
}

// Constant-product quote with the 3% trading fee already taken on the input side.
const FEE_BPS = 300n;            // 0.15 + 0.60 + 0.25 + 1.00 + 1.00 (community + extra) = 3%
const BPS_DENOM = 10_000n;

export function quoteBuy(state: CurveState, solInLamports: bigint): bigint {
  const totalSol   = state.virtualSolReserve   + state.realSolReserve;
  const totalToken = state.virtualTokenReserve + state.realTokenReserve;
  const netIn = solInLamports - (solInLamports * FEE_BPS) / BPS_DENOM;
  return (totalToken * netIn) / (totalSol + netIn);
}

export function quoteSell(state: CurveState, tokenIn: bigint): bigint {
  const totalSol   = state.virtualSolReserve   + state.realSolReserve;
  const totalToken = state.virtualTokenReserve + state.realTokenReserve;
  const grossOut = (totalSol * tokenIn) / (totalToken + tokenIn);
  return grossOut - (grossOut * FEE_BPS) / BPS_DENOM;
}
```

> **Per-token fees.** The `extra_community_fee_bps` field (offset `240`, `u16 LE`) on the curve adds to the base community fee. Configurable per mint at creation; clamp the total trading fee at quote time if you want exact lamport accounting. The 3% constant above covers the default config — read `extra_community_fee_bps` when the integrator needs precise per-mint slippage.

### 4. Decode the on-chain TradeEvent

Every `Buy` and `Sell` emits an Anchor self-CPI `TradeEvent` as an inner instruction on the program. The event payload is a 153-byte buffer: 8-byte Anchor self-CPI prefix + 8-byte `TradeEvent` discriminator + 12 fields. Reading it gives you the exact swap result (`solAmount`, `tokenAmount`, fees, post-trade reserves) without touching balance deltas.

```text
0..8     ANCHOR_LOG_DISC      sha256("anchor:event")[..8]   = [228, 69,165, 46, 81,203,154, 29]
8..16    TRADE_EVENT_DISC     sha256("event:TradeEvent")[..8] = [189,219,127,211, 78,230, 97,238]
16..48   mint                 pubkey (32)
48..56   solAmount            u64 LE (lamports moved into / out of the curve)
56..64   tokenAmount          u64 LE (raw base units moved)
64..65   isBuy                u8     (1 = Buy, 0 = Sell)
65..97   user                 pubkey (32)
97..105  virtualSolReserves   u64 LE (post-trade)
105..113 virtualTokenReserves u64 LE (post-trade)
113..121 realSolReserves      u64 LE (post-trade)
121..129 realTokenReserves    u64 LE (post-trade)
129..137 fee                  u64 LE (total fee taken from the trade)
137..145 creatorFee           u64 LE (creator's slice of `fee`)
145..153 solToUser            u64 LE (Sell only — net SOL credited to the seller)
```

```ts
import { Connection, PublicKey } from '@solana/web3.js';
import bs58 from 'bs58';
import { CONTROL_PROGRAM_ID } from './pdas';

// sha256("anchor:event")[..8] — every Anchor self-CPI log starts with this.
const ANCHOR_LOG_DISC = Buffer.from([228, 69, 165, 46, 81, 203, 154, 29]);
// sha256("event:TradeEvent")[..8] — TradeEvent's per-event discriminator.
const TRADE_EVENT_DISC = Buffer.from([189, 219, 127, 211, 78, 230, 97, 238]);
const TRADE_EVENT_LEN = 153;

export interface TradeEvent {
  mint: PublicKey;
  user: PublicKey;
  isBuy: boolean;
  solAmount: bigint;
  tokenAmount: bigint;
  fee: bigint;
  creatorFee: bigint;
  solToUser: bigint;
  virtualSolReserves: bigint;
  virtualTokenReserves: bigint;
  realSolReserves: bigint;
  realTokenReserves: bigint;
}

function decodeTradeEvent(buf: Buffer): TradeEvent {
  if (buf.length !== TRADE_EVENT_LEN) throw new Error(`bad event length: ${buf.length}`);
  if (!buf.slice(0, 8).equals(ANCHOR_LOG_DISC))   throw new Error('not an anchor self-CPI log');
  if (!buf.slice(8, 16).equals(TRADE_EVENT_DISC)) throw new Error('not a TradeEvent');

  return {
    mint:                 new PublicKey(buf.slice(16, 48)),
    solAmount:            buf.readBigUInt64LE(48),
    tokenAmount:          buf.readBigUInt64LE(56),
    isBuy:                buf[64] === 1,
    user:                 new PublicKey(buf.slice(65, 97)),
    virtualSolReserves:   buf.readBigUInt64LE(97),
    virtualTokenReserves: buf.readBigUInt64LE(105),
    realSolReserves:      buf.readBigUInt64LE(113),
    realTokenReserves:    buf.readBigUInt64LE(121),
    fee:                  buf.readBigUInt64LE(129),
    creatorFee:           buf.readBigUInt64LE(137),
    solToUser:            buf.readBigUInt64LE(145),
  };
}

// Walk the inner instructions emitted by the Control program and return
// every TradeEvent. Most txs have exactly one; batched ones may have more.
export async function parseTradeEvents(
  connection: Connection,
  signature: string,
): Promise<TradeEvent[]> {
  const tx = await connection.getTransaction(signature, {
    commitment: 'confirmed',
    maxSupportedTransactionVersion: 0,
  });
  if (!tx || !tx.meta) throw new Error('Transaction not found or missing meta');
  if (tx.meta.err)    throw new Error(`Transaction failed: ${JSON.stringify(tx.meta.err)}`);

  const staticKeys = tx.transaction.message.getAccountKeys().staticAccountKeys;
  const programIdx = staticKeys.findIndex((k) => k.equals(CONTROL_PROGRAM_ID));
  if (programIdx < 0) return [];

  const events: TradeEvent[] = [];
  for (const innerSet of tx.meta.innerInstructions ?? []) {
    for (const ix of innerSet.instructions) {
      if (ix.programIdIndex !== programIdx) continue;
      const data = Buffer.from(bs58.decode(ix.data));
      if (data.length !== TRADE_EVENT_LEN) continue;
      if (!data.slice(0, 8).equals(ANCHOR_LOG_DISC)) continue;
      events.push(decodeTradeEvent(data));
    }
  }
  return events;
}
```

> **Why this matters.** Solscan and SolanaFM render trade activity from this same event (same Anchor self-CPI convention as Anchor programs), which is why your txs show "Action: Swap N for M SOL" out of the box. Indexers can key on the program ID + the 16-byte prefix (`ANCHOR_LOG_DISC` + `TRADE_EVENT_DISC`) to detect every Buy/Sell across all Control mints with no per-mint subscription.

---

## See also

- [`../idl/control.json`](../idl/control.json) — canonical IDL (40 instructions, 1 event, 8-byte Anchor-style discriminators).
- [Solana Explorer IDL](https://explorer.solana.com/address/CTRL5CCEQw5zhhBeEV8n5GKZpf3E5tYQoXhhxzUAps27/idl) — fetch the on-chain copy directly.
- [FAQ](./FAQ.md) — short answers on graduation, slippage, networks.

Integration questions — Carlos Beltrán, <cb@print.world>.
