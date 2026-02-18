---
name: integrating-reflect-money
description: Comprehensive guidance for integrating Reflect Money APIs and SDKs — yield-bearing stablecoins (USDC+, USDT+, USTR+), whitelabel branded stablecoins, restaking (RLP), oracle (Doppler), and protocol analytics on Solana. Use for endpoint selection, SDK usage, integration flows, error handling, and production hardening.
license: MIT
metadata:
  author: reflect-money
  version: "1.1.0"
tags:
  - reflect-money
  - reflect
  - usdc-plus
  - usdt-plus
  - ustr-plus
  - yield-stablecoin
  - capital-efficient-stablecoin
  - whitelabel-stablecoin
  - lending-aggregation
  - solana
  - defi
  - rlp
  - restaking
  - stable-ts
  - proxy-ts
  - oracle-ts
  - doppler
---

# Reflect Money Integration

Single skill for all Reflect Money APIs and SDKs. Yield-bearing stablecoins on Solana.

**API Base URL**: `https://prod.api.reflect.money`
**Docs**: `https://docs.reflect.money`
**GitHub**: `https://github.com/palindrome-eng`
**Audit Report**: `https://alpha.reflect.money/static/Reflect-DeltaNeutral-Sep-2025-OffsideLabs.pdf`

---

## Use / Do Not Use

**Use when:**
- The task involves minting or redeeming USDC+, USDT+, or USTR+.
- The task involves issuing a whitelabel/branded yield-bearing stablecoin (proxy).
- The task involves querying APY, exchange rates, supply caps, or protocol stats.
- The task involves restaking into the Reflect Liquidity Pool (RLP).
- The task involves reading or writing oracle price data via Doppler.
- The user needs to integrate Reflect into a dApp, backend, or DeFi protocol.
- The user is debugging a Reflect SDK or API call.

**Do not use when:**
- The task is generic Solana setup with no Reflect API or SDK usage.
- The task involves Pyth, Switchboard, or other oracles not built by Reflect/Blueshift.

**Triggers**: `usdc+`, `usdt+`, `ustr+`, `yield stablecoin`, `capital efficient stablecoin`, `reflect money`, `reflect protocol`, `mint usdc+`, `redeem usdc+`, `lending aggregation`, `cross-margin rate farming`, `whitelabel stablecoin`, `branded stablecoin`, `proxy stablecoin`, `rlp`, `restaking reflect`, `reflect liquidity pool`, `reflect apy`, `reflect sdk`, `stable.ts`, `proxy.ts`, `oracle.ts`, `reflectmoney`, `insured stablecoin`, `yield on usdc`, `yield on usdt`, `reflect oracle`, `doppler oracle`, `treasury stablecoin onchain`

---

## Intent Router

| User intent | Path | First action |
|---|---|---|
| Mint USDC+ | [Stablecoin SDK](#stablecoin-sdk) or [REST](#stablecoin-endpoints) | `new UsdcPlusStablecoin(connection)` → `.load()` → `.mint()` |
| Redeem USDC+ | [Stablecoin SDK](#stablecoin-sdk) or [REST](#stablecoin-endpoints) | `.load()` → `.redeem()` |
| Get APY | [REST](#stablecoin-endpoints) | `GET /stablecoin/apy` |
| Get exchange rate | [REST](#stablecoin-endpoints) | `GET /stablecoin/{index}/exchange-rate` |
| Get supply cap | [REST](#stablecoin-endpoints) | `GET /stablecoin/limits` |
| Issue branded stablecoin | [Whitelabel SDK](#whitelabel-sdk) or [REST](#integration-setup) | `Proxy.initialize()` or `POST /integration/initialize/flow` |
| Whitelabel mint | [REST](#integration-operations) | `POST /integration/mint` (auto-inits user account) |
| Atomic USDC → branded | [REST](#integration-operations) | `POST /integration/flow/mint` |
| Atomic branded → USDC | [REST](#integration-operations) | `POST /integration/flow/redeem` |
| Quote for whitelabel | [REST](#integration-quote-endpoints) | `POST /integration/quote/{type}` |
| Quote for full flow | [REST](#integration-quote-endpoints) | `POST /integration/quote/flow/{type}` |
| Claim integration fees | [REST](#integration-operations) | `POST /integration/claim` |
| Integration analytics | [REST](#integration-analytics) | `GET /integration/{id}/stats` |
| Restake into RLP | [Restaking SDK](#restaking-sdk) | `new RLP(connection)` → `rlp.restake()` |
| Protocol TVL / analytics | [REST](#stats--events) | `GET /stats` |
| Read oracle price | [Oracle SDK](#oracle-sdk) | `new Doppler(connection, admin)` → `doppler.fetchOracle()` |
| Update / create oracle | [Oracle SDK](#oracle-sdk) | `doppler.updateOracle()` or `doppler.createOracleAccount()` (operators only) |

---

## Developer Quickstart

```typescript
import { Connection, Keypair, TransactionMessage, VersionedTransaction } from "@solana/web3.js";
import { UsdcPlusStablecoin } from "@reflectmoney/stable.ts";
import BN from "bn.js";

const connection = new Connection("https://api.mainnet-beta.solana.com");
const stablecoin = new UsdcPlusStablecoin(connection);
await stablecoin.load(connection); // REQUIRED before every session

const AMOUNT = new BN(1000 * 1_000_000);  // 1000 USDC (6 decimals)
const MIN_OUT = new BN(999 * 1_000_000);   // 0.1% slippage

const ix = await stablecoin.mint(user.publicKey, AMOUNT, MIN_OUT);
const { blockhash } = await connection.getLatestBlockhash();
const { value: lut } = await connection.getAddressLookupTable(stablecoin.lookupTable);

// ALWAYS VersionedTransaction, ALWAYS include ALT
const tx = new VersionedTransaction(
  new TransactionMessage({ instructions: ix, payerKey: user.publicKey, recentBlockhash: blockhash })
    .compileToV0Message([lut])
);
tx.sign([keypair]);
await connection.sendRawTransaction(tx.serialize(), { skipPreflight: true, maxRetries: 0 });
```

**REST equivalent:**
```bash
# Quote
curl -X POST https://prod.api.reflect.money/stablecoin/quote/mint \
  -H "Content-Type: application/json" \
  -d '{"amount": 1000000, "index": 0}'

# Mint — returns base64 VersionedTransaction, deserialise, sign, send
curl -X POST https://prod.api.reflect.money/stablecoin/mint \
  -H "Content-Type: application/json" \
  -d '{"amount": 1000000, "index": 0, "feePayer": "<pubkey>"}'
```

---

## What Is Reflect Money

Reflect Money is **credibly-neutral stablecoin infrastructure on Solana**. It tokenizes onchain DeFi yield strategies as interest-bearing US Dollar stablecoins. Programs manage all deposits — zero human custody, zero admin keys, minted and redeemed at will with zero fees.

**Circle Alliance Member** — Reflect is NOT affiliated with or endorsed by Circle Inc. Entirely separate entities.
**Funding**: $3.75M — a16z Crypto CSX, Solana Ventures, Equilibrium, Big Brain VC, Colosseum.
**Audit**: Offside Labs, September 2025 — USD Coin Plus strategy. All findings resolved.

---

## The Strategy — Cross-Margin Rate Farming

USDC+ and USDT+ both use the **same strategy**:

1. User deposits base stablecoin → Reflect program holds it in non-custodial pool.
2. Program scans Solana lending markets (Drift, Kamino, Jupiter Lend) for best rates.
3. USDC distributed automatically across venues to capture highest available interest.
4. User receives USDC+ — which **natively appreciates in value** in their wallet, no claiming required.
5. Convert back to USDC at any time; 1 USDC+ is always worth slightly more than 1 USDC.

**Yield source**: Interest paid by overcollateralised borrowers on Solana money markets.

---

## Live and Upcoming Stablecoins

| Token | Collateral | Status | Index | SDK Class | Yield |
|-------|-----------|--------|-------|-----------|-------|
| USDC+ | USDC (Circle) | ✅ Live | 0 | `UsdcPlusStablecoin` | Native appreciation |
| USDT+ | USDT (Tether) | ⏳ Upcoming | TBD | TBD | Native appreciation |
| USTR+ | US Treasury Bills (aggregated) | ⏳ Upcoming | TBD | TBD | TBD |

> **Hard rule**: Only USDC+ is confirmed live by the protocol team. The SDK docs list `LstStablecoin` (Index 2) as "Live" — this is a known discrepancy. Confirm live stablecoins at runtime via `GET /stablecoin/types`.

> **Closed beta**: Reflect is currently in closed beta. Users must be whitelisted via `POST /integration/whitelist` before they can transact.

---

## SDK Packages

```bash
yarn add @reflectmoney/stable.ts    # mint/redeem stablecoins
yarn add @reflectmoney/proxy.ts     # whitelabel branded stablecoins
yarn add @reflectmoney/restaking    # RLP restaking (post-beta only)
yarn add @reflectmoney/oracle.ts    # Doppler oracle price feeds
```

---

## Stablecoin SDK

**Package**: `@reflectmoney/stable.ts`

**Gotchas**:
- Always call `.load(connection)` before any operation. Never skip or cache across sessions.
- Always use `VersionedTransaction` (V0). Legacy transactions will fail.
- ALTs are **required** for all stablecoin SDK calls — pass `[lut]` to `compileToV0Message()`.
- ALTs are **NOT required** for `ReflectTokenizedBond` — `compileToV0Message()` with no args.
- Use `stablecoin.index` — never hardcode the index number.
- USDC+ natively appreciates — no `ReflectTokenizedBond` interaction needed.
- `rebalance()` is **permissioned** — never call from integrations.
- On first mint, call `initializeTokenAccounts()` before minting.
- Slippage for redeem: because USDC+ has appreciated, `MIN_RECEIVED` should be **higher** than the USDC+ input amount.

### Core Modules (rarely needed for basic integrations)

**`Reflect`** — general-purpose protocol setup. Not required for standard flows.

**`ReflectKeeper`** — permissioned keeper functions. Admin only, never call from integrations.

```typescript
import { UsdcPlusStablecoin, ReflectKeeper } from "@reflectmoney/stable.ts";

// Admin only
const initIx     = await ReflectKeeper.initializeMain(admin);
const capIx      = await stablecoin.updateCap(admin, BigInt(1_000_000_000_000_000));
const freezeIx   = await stablecoin.freeze(admin, true, 0);
const unfreezeIx = await stablecoin.freeze(admin, false, 0);
```

**`PdaClient`** — PDA derivation.

```typescript
import { PdaClient } from "@reflectmoney/stable.ts";

const mainPda        = await PdaClient.deriveMain();
const controllerPda  = await PdaClient.deriveController(0);  // 0 = USDC+
const permissionsPda = await PdaClient.derivePermissions(adminAddress);
```

### Abstract Stablecoin Class

```typescript
abstract class Stablecoin {
  public index: number;              // read as property, never hardcode
  public name: string;
  public connection: Connection;
  public controller: PublicKey;
  public collaterals: Collateral[];
  public stablecoinMint: PublicKey;
  public pythClient: HermesClient;
  public driftClient?: DriftClient;  // only for Drift-powered strategies

  abstract initializeTokenAccounts(owner: PublicKey, mints: PublicKey[], signer: PublicKey): Promise<TransactionInstruction[]>;
  abstract mint(signer: PublicKey, amount: BN | number, minimumReceived: BN | number, ...args: any[]): Promise<TransactionInstruction[]>;
  abstract redeem(signer: PublicKey, amount: BN | number, minimumReceived: BN | number, ...args: any[]): Promise<TransactionInstruction[]>;
  abstract rebalance(signer: PublicKey): Promise<TransactionInstruction[]>; // PERMISSIONED — never call
}
```

### Mint

```typescript
import { UsdcPlusStablecoin } from "@reflectmoney/stable.ts";
import { Connection, Keypair, TransactionMessage, VersionedTransaction } from "@solana/web3.js";
import BN from "bn.js";

const connection = new Connection("https://api.mainnet-beta.solana.com");
const stablecoin = new UsdcPlusStablecoin(connection);
await stablecoin.load(connection);

const AMOUNT  = new BN(1000 * Math.pow(10, 6));  // $1000 USDC
const MIN_OUT = new BN(999  * Math.pow(10, 6));   // 0.1% slippage

const ix = await stablecoin.mint(user.publicKey, AMOUNT, MIN_OUT);
const { blockhash } = await connection.getLatestBlockhash();
const { value: lut } = await connection.getAddressLookupTable(stablecoin.lookupTable);

const tx = new VersionedTransaction(
  new TransactionMessage({ instructions: ix, payerKey: user.publicKey, recentBlockhash: blockhash })
    .compileToV0Message([lut])
);
tx.sign([user]);
await connection.sendRawTransaction(tx.serialize());
```

### Redeem

```typescript
// USDC+ has appreciated — burn 999 USDC+, expect >= 1000 USDC back
const AMOUNT  = new BN(999  * Math.pow(10, 6));
const MIN_OUT = new BN(1000 * Math.pow(10, 6));

const ix = await stablecoin.redeem(user.publicKey, AMOUNT, MIN_OUT);
// Build + send VersionedTransaction identically to mint
```

### First-Time User

```typescript
const initIx = await stablecoin.initializeTokenAccounts(
  user.publicKey,
  [stablecoin.stablecoinMint],
  user.publicKey
);
// Include initIx in same tx as, or before, the first mint
```

### ReflectTokenizedBond (future stablecoins only)

USDC+ and USDT+ natively appreciate — this is not needed for them. For future stablecoins distributing yield via receipt tokens (e.g. USDJ when live):

```typescript
import { ReflectTokenizedBond, UsdjStablecoin } from "@reflectmoney/stable.ts";

const stablecoin = new UsdjStablecoin(connection);
const bond = new ReflectTokenizedBond(connection);

const depositIx  = bond.deposit(user.publicKey, stablecoin.index, AMOUNT);
const withdrawIx = bond.withdraw(user.publicKey, stablecoin.index, AMOUNT);

// NO ALT — compileToV0Message() with no arguments
const tx = new VersionedTransaction(
  new TransactionMessage({ instructions: [depositIx], payerKey: user.publicKey, recentBlockhash: blockhash })
    .compileToV0Message()
);
```

---

## Whitelabel SDK

**Package**: `@reflectmoney/proxy.ts`

**Gotchas**:
- `Proxy.initialize()` is one-time — creates on-chain accounts. Never call twice for the same proxy.
- Only use the constructor after on-chain proxy accounts are already initialised.
- `Proxy.loadFromMint()` — preferred load path, derives proxy state from branded mint.
- `new Proxy({ connection, proxyStateAddress })` — alternative if you have the state address directly. Note: parameter is `proxyStateAddress`, not `proxyAddress`.
- `proxy.load()` — lazily loads state; call again to refresh after on-chain changes.
- `claimFees()` — authority only.
- Uses `@solana/kit` (`createSolanaRpc`, `address`) not `@solana/web3.js`.

**Exported constants:**
- `USDC_PLUS_MINT` — USDC+ mint address on mainnet
- `USDC_PLUS_ORACLE` — Oracle tracking real-time USDC+ exchange rate
- `PROXY_STATE_SEED` — PDA seed for proxy state derivation
- `REFLECT_PROXY_PROGRAM_PROGRAM_ADDRESS` — Proxy program ID

```typescript
import { Proxy, USDC_PLUS_MINT, PdaClient } from "@reflectmoney/proxy.ts";
import { createSolanaRpc, address } from "@solana/kit";

const connection = createSolanaRpc("https://api.mainnet-beta.solana.com");

// --- ONE-TIME SETUP ---
const createIx = await Proxy.initialize({
  signer: myWallet,
  authority: address("AuthorityAddress"),   // optional, defaults to signer
  brandedMint: address("YourBrandedMint"),
  feeBps: 50,                                // 0.5% fee on all flows
  stablecoinMint: address(USDC_PLUS_MINT)
});

// --- LOAD BY MINT (preferred) ---
const proxy = await Proxy.loadFromMint({ connection, brandedMint: address("YourBrandedMint") });

// --- LOAD BY STATE ADDRESS ---
const proxy2 = new Proxy({ connection, proxyStateAddress: address("ProxyStateAddress") });
await proxy2.load(); // call again anytime to refresh

// --- OPERATIONS ---
const depositIx  = await proxy.deposit({ signer: myWallet, amount: 1_000_000 });
const withdrawIx = await proxy.withdraw({ signer: myWallet, amount: 1_000_000 });
const claimIx    = await proxy.claimFees(); // authority only

// --- PDA DERIVATION ---
const [proxyStateAddress, bump] = await PdaClient.deriveProxyState(address("YourBrandedMint"));
```

---

## Restaking SDK

**Package**: `@reflectmoney/restaking`

**Status**: ⚠️ NOT AVAILABLE until Reflect exits public beta. Do not build production integrations until then.

**Gotchas**: Withdrawals require a cooldown period — read cooldown state before surfacing to users.

```typescript
import { RLP } from "@reflectmoney/restaking";
import BN from "bn.js";

const rlp = new RLP(connection);

// Monitoring
const lockups     = await rlp.getLockups();
const byAsset     = await rlp.getLockupsByAsset(mintPublicKey);
const myDeposits  = await rlp.getDepositsByUser(myPublicKey);
const myCooldowns = await rlp.getCooldownsByUser(myPublicKey);
const byCooldown  = await rlp.getCooldownsByDeposit(new BN(0));

// PDA derivation
const settings = rlp.deriveSettings();
const lockup   = rlp.deriveLockup(new BN(0));
const asset    = rlp.deriveAsset(mintPublicKey);

// Actions (post-beta only)
const restakeIx  = rlp.restake(signer.publicKey, new BN(1000 * 1_000_000), new BN(0));
const withdrawIx = rlp.withdraw(signer.publicKey, new BN(1000 * 1_000_000), new BN(0));
```

---

## Oracle SDK

**Package**: `@reflectmoney/oracle.ts`
**Program ID**: `PRicevBH6BaeaE8qmrxrwGBZ5hSZ9vjBNue5Ygot1ML`
**Underlying**: Fork of Doppler by Blueshift — **32 CUs per update** (base Doppler = 21 CUs; Reflect fork adds a Clock sysvar call)

**Gotchas**:
- `Doppler` constructor requires an admin keypair — oracle writes are permissioned to the oracle creator.
- Price payloads are 8 bytes (u64). Always use `createPricePayload()` to encode, `readPriceFromPayload()` to decode.
- Always use the same `U8Array8Serializer()` across `updateOracle` and `fetchOracle` calls.
- Pass `BigInt(Date.now())` or the current Solana slot as `slot` — used for staleness checks.
- Most integrators only need `fetchOracle` — write methods are for oracle operators.

```typescript
import { Doppler, U8Array8Serializer, createPricePayload, readPriceFromPayload } from "@reflectmoney/oracle.ts";
import { Connection, Keypair, PublicKey } from "@solana/web3.js";

const connection = new Connection("https://api.mainnet-beta.solana.com");
const admin = Keypair.fromSecretKey(/* admin keypair bytes */);
const doppler = new Doppler(connection, admin);
const oraclePublicKey = new PublicKey("ORACLE_ADDRESS");

// READ (most common)
const oracleData = await doppler.fetchOracle(oraclePublicKey, new U8Array8Serializer());
if (oracleData) {
  const price = readPriceFromPayload(oracleData.payload);
  // price: 1000000n = 1.000000 USDC (6-decimal fixed point)
}

// CREATE oracle account (operators only)
const oracleAccount = await doppler.createOracleAccount(
  "my-oracle-seed",
  new U8Array8Serializer(),
  { slot: 0n, payload: createPricePayload(BigInt(1_000_000)) }
);

// UPDATE oracle (operators only)
await doppler.updateOracle(
  oraclePublicKey,
  { slot: BigInt(Date.now()), payload: createPricePayload(BigInt(1_000_000)) },
  new U8Array8Serializer()
);
```

---

## REST API

Base URL: `https://prod.api.reflect.money`

All transaction endpoints return `{ "success": true, "data": { "transaction": "<base64>" } }`. Deserialise with `VersionedTransaction.deserialize(Buffer.from(b64, 'base64'))`, sign, and send.

### Stablecoin Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/health` | Health check |
| `GET` | `/stablecoin/types` | List available stablecoin types |
| `GET` | `/stablecoin/limits` | Supply caps, current supply, remaining capacity |
| `GET` | `/stablecoin/apy` | Realtime APY for all stablecoins |
| `GET` | `/stablecoin/{index}/apy` | Realtime APY for specific stablecoin |
| `GET` | `/stablecoin/{index}/apy/historical` | Historical APY (`days` param) |
| `GET` | `/stablecoin/exchange-rates` | Latest exchange rates for all stablecoins |
| `GET` | `/stablecoin/{index}/exchange-rate` | Realtime exchange rate for specific stablecoin |
| `GET` | `/stablecoin/exchange-rates/historical` | Historical rates (`index` + `days` params) |
| `POST` | `/stablecoin/quote/{type}` | Quote for mint or redeem (`type`: `mint` or `redeem`) |
| `POST` | `/stablecoin/mint` | Generate mint transaction |
| `POST` | `/stablecoin/burn` | Generate burn (redeem) transaction |

### Integration Setup

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/integration/upload` | Upload brand image + JSON metadata → returns URLs |
| `POST` | `/integration/initialize` | Initialize integration (branded mint + fee config) |
| `POST` | `/integration/initialize-stablecoin` | Create branded token mint with metadata |
| `POST` | `/integration/transfer-authority` | Transfer mint authority to Reflect |
| `POST` | `/integration/initialize/flow` | Complete setup in a single transaction (recommended) |
| `POST` | `/integration/initialize-vault` | Initialize proxy vault (admin only) |

### Integration Config & Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/integration/{id}/config` | Get integration configuration |
| `POST` | `/integration/{id}/config/update` | Update fee configuration |
| `GET` | `/integration/{authority}/integrations` | All integrations for an authority address |
| `GET` | `/integration/verified` | All verified integrations (public) |
| `GET` | `/integration/check/{stablecoin}` | Check if branded mint is verified (returns bool) |
| `POST` | `/integration/api-key/reveal` | Reveal API key (60s expiry signature required) |
| `POST` | `/integration/api-key/rotate` | Rotate API key — old key immediately invalidated |

### Integration Operations

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/integration/whitelist` | Whitelist users (closed beta requirement) |
| `POST` | `/integration/initialize-user-account` | Init user branded token account (standalone) |
| `POST` | `/integration/mint` | Whitelabel mint (auto-inits user account if needed) |
| `POST` | `/integration/flow/mint` | Atomic: USDC → USDC+ → branded (single tx) |
| `POST` | `/integration/redeem` | Whitelabel redeem |
| `POST` | `/integration/flow/redeem` | Atomic: branded → USDC+ → USDC (single tx) |
| `POST` | `/integration/claim` | Generate fee claim transaction |

> `/integration/mint` automatically detects if the user's branded token account needs initializing and includes the init instruction. Set `feePayer` to your address to sponsor account creation costs on behalf of users.

> `/integration/initialize-user-account` is only needed if you want to init accounts separately from minting. Otherwise `/integration/mint` handles it.

### Integration Quote Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/integration/quote/{type}` | Quote: USDC+ ↔ branded tokens (`type`: `mint` or `redeem`) |
| `POST` | `/integration/quote/flow/{type}` | Quote: full USDC ↔ branded flow (handles both conversions) |

### Integration Analytics

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/integration/{id}/stats` | Integration statistics |
| `GET` | `/integration/{id}/stats/historical` | Historical stats (`period` param: `7` or `30` days, default 7) |
| `GET` | `/integration/{id}/exchange-rate` | Current exchange rate for the integration |
| `GET` | `/integration/{id}/events` | Recent events for the integration |

### Stats & Events

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/stats` | Protocol statistics (TVL, volume) |
| `GET` | `/stats/historical` | Historical TVL + volume (`days` param: 1–365) |
| `GET` | `/events/all/{limit}` | Most recent protocol events up to limit |
| `GET` | `/events/{signer}` | Events for a specific signer address |

### API Key Auth

**Message to sign:**
- Reveal: `"Reveal API key for integration {integrationId} at timestamp {timestamp}"`
- Rotate: `"Rotate API key for integration {integrationId} at timestamp {timestamp}"`

```bash
curl -X POST https://prod.api.reflect.money/integration/api-key/reveal \
  -H "Content-Type: application/json" \
  -d '{
    "integrationId": "<integration-id>",
    "signer": "<authority-pubkey>",
    "signature": "<base58-signature>",
    "timestamp": 1704067200000
  }'
```

> ⚠️ Signatures expire after **60 seconds**. Generate timestamp immediately before signing. Rotation is irreversible — old key invalidated with no recovery.

---

## Error Handling Pattern

```typescript
interface ReflectResult<T> {
  ok: boolean;
  result?: T;
  error?: { code: string | number; message: string; retryable: boolean };
}

async function reflectAction<T>(action: () => Promise<T>): Promise<ReflectResult<T>> {
  try {
    return { ok: true, result: await action() };
  } catch (err: any) {
    const code = err?.code ?? err?.status ?? "UNKNOWN";
    if (code === 429 || code === "RATE_LIMITED")
      return { ok: false, error: { code: "RATE_LIMITED", message: "Rate limited", retryable: true } };
    if (typeof code === "number" && code >= 400 && code < 500)
      return { ok: false, error: { code, message: err?.message ?? `HTTP_${code}`, retryable: code === 429 || code === 408 } };
    if (err?.code === "ECONNRESET" || err?.code === "ETIMEDOUT")
      return { ok: false, error: { code: err.code, message: "Network error", retryable: true } };
    return { ok: false, error: { code, message: err?.message ?? "UNKNOWN_ERROR", retryable: false } };
  }
}
```

On retryable errors: `delay = min(base * 2^attempt + random(0, jitter), maxDelay)`.

---

## Risk Framework

| Risk | Prevention | Elimination |
|------|-----------|-------------|
| Smart contract | Code + Economic Audits | Global Insurance (RLP) |
| Exchange / DEX | Use of insured DEXs | Global Insurance |
| Interest rate | Economic Strategy Audits | Strategy fallbacks |
| Collateral | Asset Classification (S-tier only) | Limitation to S-tier |

**Insurance covers principal only.** Yield is variable and not guaranteed. Insurance nodes attest every 450ms. Integrators must disclose variable yield to users.

**USDC+ risk factors**: Smart contract risks compound across all active lending venues. Withdrawal capacity depends on utilisation rates per venue. Yield fluctuates with Solana money market conditions.

---

## Production Hardening

1. **Decimals**: All Reflect tokens use **6 decimal places** — multiply by `1_000_000`. Never use 10^9.
2. **`.load()` every session**: Call `stablecoin.load(connection)` before every set of operations. Never cache across sessions.
3. **VersionedTransaction always**: `compileToV0Message()` — legacy transactions fail.
4. **ALTs**: Required for stablecoin SDK calls via `stablecoin.lookupTable`. Not required for `ReflectTokenizedBond`.
5. **Slippage guardrails**: Always pass real `minimumReceived`. Never pass zero — exposes users to sandwich attacks.
6. **Redeem slippage direction**: For USDC+, `MIN_RECEIVED` should be **greater** than input USDC+ amount (token has appreciated).
7. **Closed beta**: All users need whitelisting via `POST /integration/whitelist` before transacting.
8. **Timeouts**: 5s for reads, 30s for transaction submission.
9. **Retries**: Only on transient/network/429. Never retry 4xx client errors.
10. **API key signing**: Generate `timestamp` immediately before signing — 60s expiry.
11. **Proxy constructor param**: `proxyStateAddress` (not `proxyAddress`).
12. **User messaging**: Communicate that insurance covers principal only; yield is variable.
13. **Not Circle**: Never describe USDC+ as a Circle product. Reflect is a Circle Alliance member only.

---

## Integration Patterns

### Pattern A — REST (Any Backend)
```
GET  /stablecoin/apy              → show yield
POST /stablecoin/quote/mint       → get quote
POST /stablecoin/mint             → generate tx → client signs + sends
POST /stablecoin/burn             → generate tx → client signs + sends
```

### Pattern B — TypeScript dApp (Stablecoin SDK)
```typescript
const { publicKey, signTransaction } = useWallet();
const stablecoin = new UsdcPlusStablecoin(connection);
await stablecoin.load(connection);
const ix = await stablecoin.mint(publicKey, amount, minReceived);
// Build VersionedTransaction with ALT, pass to wallet adapter to sign
```

### Pattern C — Whitelabel Integration
```
1. POST /integration/upload              → upload brand image + metadata
2. POST /integration/initialize/flow     → single tx: mint + authority + integration setup
3. POST /integration/whitelist           → whitelist your users (closed beta)
4. Users: POST /integration/flow/mint   → atomic USDC → USDC+ → branded
5. Users: POST /integration/flow/redeem → atomic branded → USDC+ → USDC
6. You:   POST /integration/claim       → harvest fee revenue
```

### Pattern D — Analytics Display
```
GET /stats                              → protocol TVL + volume
GET /stats/historical?days=30           → 30-day chart data
GET /integration/{id}/stats             → your integration's stats
GET /integration/{id}/stats/historical  → your integration's historical stats
```

---

## Fresh Context Policy

1. Treat live documentation at `docs.reflect.money` as source of truth over this file.
2. If behaviour differs from expectations, fetch the relevant docs page before proceeding.
3. If docs cannot be fetched, note that context may be stale and continue with best-known guidance.
4. **Known discrepancy**: SDK docs list `LstStablecoin` (Index 2) as "Live" — per the protocol team only USDC+ is currently live. Verify at runtime via `GET /stablecoin/types`.
5. USDT+ and USTR+ launches will add new SDK classes and API parameters. Re-check docs before integrating.
6. Closed beta status may change — check whitelist requirements before shipping to users.

---

## Operational References

| Resource | URL |
|----------|-----|
| Docs home | https://docs.reflect.money |
| Stablecoin SDK | https://docs.reflect.money/api-reference/stablecoin |
| Whitelabel SDK | https://docs.reflect.money/api-reference/proxy |
| Restaking SDK | https://docs.reflect.money/api-reference/restaking |
| Oracle SDK | https://docs.reflect.money/api-reference/oracle |
| USDC+ strategy | https://docs.reflect.money/strategies/USDC |
| Risk overview | https://docs.reflect.money/risk-management/notice |
| Security audit | https://alpha.reflect.money/static/Reflect-DeltaNeutral-Sep-2025-OffsideLabs.pdf |
| GitHub | https://github.com/palindrome-eng |
| Protocol home | https://reflect.money |
