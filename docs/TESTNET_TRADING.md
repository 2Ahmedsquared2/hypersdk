# Testnet Trading Guide

This fork of hypersdk is configured for **testnet-first trading** with code-only signing (no Phantom wallet required).

## Table of Contents

- [Quick Start](#quick-start)
- [Understanding the Hyperliquid API](#understanding-the-hyperliquid-api)
- [Code-Only Signing (Agent Wallets)](#code-only-signing-agent-wallets)
- [Testnet vs Mainnet](#testnet-vs-mainnet)
- [Step-by-Step: Trading on Testnet](#step-by-step-trading-on-testnet)
- [Perpetuals vs HIP-4 Outcomes](#perpetuals-vs-hip-4-outcomes)
- [Examples Reference](#examples-reference)
- [Safety Guidelines](#safety-guidelines)

---

## Quick Start

1. **Get a testnet wallet private key** (generate a new one for testing)
2. **Fund it via the faucet**: https://app.hyperliquid-testnet.xyz/faucet
3. **Set your private key**:
   ```bash
   cp .env.example .env
   # Edit .env and add your PRIVATE_KEY
   ```
4. **Run a trading example**:
   ```bash
   cargo run --example send_order
   ```

---

## Understanding the Hyperliquid API

This SDK wraps the official Hyperliquid API. Here's how SDK calls map to the API surfaces:

### Official API Documentation

**Main API docs**: https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api

Key pages:
- **Info endpoint** (`/info`): https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint
- **Exchange endpoint** (`/exchange`): https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/exchange-endpoint
- **WebSocket subscriptions**: https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/websocket
- **Signing**: https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/signing
- **Asset IDs**: https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/asset-ids
- **Nonces & API wallets**: https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/nonces-and-api-wallets
- **Rate limits**: https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits

### SDK to API Mapping

| SDK Method | API Endpoint | HTTP Method | Purpose | Docs Link |
|------------|--------------|-------------|---------|-----------|
| `client.perps()` | `/info` | POST | Get perpetual markets | [Info endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint/perpetuals) |
| `client.spot()` | `/info` | POST | Get spot markets | [Info endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint/spot) |
| `client.outcome_meta()` | `/info` | POST | Get HIP-4 outcome markets | [HIP-4](https://hyperliquid.gitbook.io/hyperliquid-docs/hyperliquid-improvement-proposals-hips/hip-4-outcome-markets) |
| `client.user_state(addr)` | `/info` | POST | Get user balances & positions | [Info endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint) |
| `client.place()` | `/exchange` | POST | Place order | [Exchange endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/exchange-endpoint) |
| `client.modify()` | `/exchange` | POST | Modify order | [Exchange endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/exchange-endpoint) |
| `client.cancel()` | `/exchange` | POST | Cancel order | [Exchange endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/exchange-endpoint) |
| `client.market_open()` | `/exchange` | POST | Market order (FrontendMarket) | [Exchange endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/exchange-endpoint) |
| `client.split_outcome()` | `/exchange` | POST | Split outcome shares | [HIP-4](https://hyperliquid.gitbook.io/hyperliquid-docs/hyperliquid-improvement-proposals-hips/hip-4-outcome-markets) |
| `client.merge_outcome()` | `/exchange` | POST | Merge outcome shares | [HIP-4](https://hyperliquid.gitbook.io/hyperliquid-docs/hyperliquid-improvement-proposals-hips/hip-4-outcome-markets) |
| `client.websocket()` | WebSocket | - | Real-time subscriptions | [WebSocket docs](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/websocket) |

**Endpoints**:
- **Testnet REST**: `https://api.hyperliquid-testnet.xyz`
- **Testnet WebSocket**: `wss://api.hyperliquid-testnet.xyz/ws`
- **Mainnet REST**: `https://api.hyperliquid.xyz`
- **Mainnet WebSocket**: `wss://api.hyperliquid.xyz/ws`

The SDK handles endpoint selection via `hypercore::testnet()` or `hypercore::mainnet()`.

---

## Code-Only Signing (Agent Wallets)

Hyperliquid supports **agent wallets** for programmatic trading without browser wallet extensions.

### How It Works

1. Your code signs transactions using a **private key** (secp256k1 ECDSA)
2. The signature is sent with the action to `/exchange`
3. Hyperliquid verifies the signature and executes the action

This is **not** Phantom or MetaMask — it's pure code signing via `PrivateKeySigner`.

### Setting Up an Agent Wallet

```rust
use hypersdk::hypercore::PrivateKeySigner;

// Option 1: From environment variable
let signer: PrivateKeySigner = std::env::var("PRIVATE_KEY")?.parse()?;

// Option 2: From string literal (for testing only!)
let signer: PrivateKeySigner = "your_private_key_here".parse()?;

// Option 3: From Foundry keystore (encrypted)
let signer = PrivateKeySigner::decrypt_keystore(
    "/home/user/.foundry/keystores/my_wallet",
    "password"
)?;
```

**Important**: The address derived from this private key is your **agent wallet address**. You must approve this agent in the Hyperliquid UI before it can trade.

### Approving Your Agent

**Official docs**: https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/nonces-and-api-wallets

1. Connect your **master wallet** (the one with funds) to https://app.hyperliquid-testnet.xyz
2. Go to **Settings → API**
3. Add your agent's address (derived from the private key you're using in code)
4. Approve it for trading

Now your code can sign and submit orders on behalf of your master wallet.

**Query the master address, not the agent**: When checking balances/positions, query your **master wallet address**, not the agent address. The agent only signs; funds live in the master.

---

## Testnet vs Mainnet

### Testnet (Default in This Fork)

- **Purpose**: Safe experimentation with fake funds
- **Faucet**: https://app.hyperliquid-testnet.xyz/faucet
- **API**: `https://api.hyperliquid-testnet.xyz`
- **SDK**: `hypercore::testnet()`

Trading examples in this fork (e.g., `send_order.rs`, `market_order.rs`, `split_merge_outcome.rs`) now default to **testnet**.

### Mainnet

- **Purpose**: Real trading with real assets
- **Risk**: Real money at stake
- **API**: `https://api.hyperliquid.xyz`
- **SDK**: `hypercore::mainnet()`

To use mainnet, change `hypercore::testnet()` to `hypercore::mainnet()` in examples.

**Always test on testnet first.**

---

## Step-by-Step: Trading on Testnet

### 1. Fund Your Testnet Wallet

1. Generate a new wallet (or use an existing one)
2. Visit https://app.hyperliquid-testnet.xyz/faucet
3. Enter your address and request testnet USDC

### 2. Approve Your Agent

1. Set your private key:
   ```bash
   export PRIVATE_KEY="your_private_key_here"
   ```
2. Get your agent address:
   ```bash
   cargo run --example approve_agent
   ```
3. In the Hyperliquid testnet UI (Settings → API), add this agent address

### 3. List Available Markets

**Perpetuals**:
```bash
cargo run --example list-markets
```

Maps to `client.perps()` → `/info` request with `"type": "meta"`.

**HIP-4 Outcome Markets** (prediction markets):
```bash
cargo run --example list-outcomes
```

Maps to `client.outcome_meta()` → `/info` request with `"type": "outcomeMeta"`.

### 4. Place a Perpetual Order

```bash
cargo run --example send_order
```

This example:
- Places a limit order for BTC
- Modifies it
- Cancels it

Maps to:
- `client.place()` → `/exchange` with `"type": "order"`
- `client.modify()` → `/exchange` with `"type": "batchModify"`
- `client.cancel()` → `/exchange` with `"type": "cancel"`

### 5. Place a Market Order

```bash
cargo run --example market_order -- --coin BTC --buy --size 0.01 --price 100000
```

Uses `client.market_open()` → `/exchange` with `"type": "order"` and `"orderType": {"limit": {"tif": "Ioc"}}` (Immediate-or-Cancel).

### 6. Trade HIP-4 Outcomes

**Split** (lock collateral to mint YES + NO shares):
```bash
cargo run --example split_merge_outcome -- --outcome 123 --wei 1000000
```

Maps to `client.split_outcome()` → `/exchange` with `"type": "outcomeDeploy"` and `"action": "split"`.

**Merge** (burn YES + NO to reclaim collateral):
The same example merges after a 10-second wait.

Maps to `client.merge_outcome()` → `/exchange` with `"type": "outcomeDeploy"` and `"action": "merge"`.

---

## Perpetuals vs HIP-4 Outcomes

### Perpetuals

- **What**: Traditional perp futures (BTC, ETH, etc.)
- **Asset IDs**: Numeric index (e.g., `0` = BTC, `1` = ETH)
- **Markets**: Listed via `client.perps()`
- **Docs**: https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint/perpetuals

### HIP-4 Outcome Markets (Prediction Markets)

- **What**: Binary outcome markets (YES/NO)
- **Asset IDs**: Outcome ID (e.g., `outcome: 123`)
- **Markets**: Listed via `client.outcome_meta()`
- **Mechanism**: Split to mint shares, merge to reclaim collateral, or trade YES/NO
- **Docs**: https://hyperliquid.gitbook.io/hyperliquid-docs/hyperliquid-improvement-proposals-hips/hip-4-outcome-markets

**Asset ID scheme**: https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/asset-ids

---

## Examples Reference

All examples live in `examples/hypercore/`. Key ones for testnet trading:

| Example | Purpose | API Mapping |
|---------|---------|-------------|
| `list-markets.rs` | List perp markets | `/info` → `meta` |
| `list-outcomes.rs` | List HIP-4 outcomes | `/info` → `outcomeMeta` |
| `send_order.rs` | Place/modify/cancel limit order | `/exchange` → `order`, `batchModify`, `cancel` |
| `market_order.rs` | Market order (FrontendMarket) | `/exchange` → `order` with IoC |
| `split_merge_outcome.rs` | Split & merge outcome shares | `/exchange` → `outcomeDeploy` |
| `approve_agent.rs` | Approve agent for trading | `/exchange` → `approveAgent` |
| `user_balances.rs` | Query user balances | `/info` → `clearinghouseState` |
| `websocket.rs` | Real-time market data | WebSocket subscriptions |

**Run any example**:
```bash
cargo run --example <name>
# Or with flags:
cargo run --example market_order -- --coin ETH --buy --size 0.1 --price 4000
```

---

## Safety Guidelines

1. **Testnet First**: Always test on testnet before mainnet
2. **Never Commit Keys**: Add `.env` to `.gitignore` (already done). Use `.env.example` for templates
3. **Query Master Address**: Check balances on your **master wallet**, not the agent address
4. **Agent Approval**: Your agent must be approved in the UI before it can trade
5. **Nonce Handling**: Use `NonceHandler::default()` or current Unix timestamp in milliseconds
6. **Rate Limits**: See https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits

---

## Troubleshooting

### "User or API Wallet 0x... does not exist"

- **Cause**: Agent not approved, or signature mismatch
- **Fix**: Approve your agent in the UI, or verify your `PRIVATE_KEY` is correct

### "Insufficient balance"

- **Cause**: No funds in your master wallet
- **Fix**: Use the testnet faucet: https://app.hyperliquid-testnet.xyz/faucet

### "Failed to deserialize" (HTTP 422)

- **Cause**: Malformed request payload
- **Fix**: Check the API docs or compare against working examples

### Agent approval but still failing

- **Check**: Are you querying the right address? Use the **master** wallet address for balances, not the agent.

---

## Resources

- **Official API Docs**: https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api
- **HIP-4 Outcomes**: https://hyperliquid.gitbook.io/hyperliquid-docs/hyperliquid-improvement-proposals-hips/hip-4-outcome-markets
- **Testnet Faucet**: https://app.hyperliquid-testnet.xyz/faucet
- **Testnet UI**: https://app.hyperliquid-testnet.xyz
- **SDK Examples**: `examples/hypercore/`
- **Upstream SDK**: https://github.com/infinitefield/hypersdk

---

**This fork is maintained by Ahmed ([@2Ahmedsquared2](https://github.com/2Ahmedsquared2)) for testnet experimentation with perps and HIP-4 outcomes.**
