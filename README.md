# Delta-15 Parity Engine

Delta-15 Parity Engine is a Rust application that executes a dual-sided pre-order strategy on quarter-hour binary prediction markets for Bitcoin, Ethereum, Solana, and XRP. It submits limit bids on both outcome tokens prior to period start and redeems settled positions when markets resolve.

[![Rust](https://img.shields.io/badge/rust-1.70%2B-orange.svg)](https://www.rust-lang.org/)
[![Polymarket](https://img.shields.io/badge/Polymarket-15m%20markets-blue)](https://polymarket.com)

---

## Overview

The engine targets Polymarket's 15-minute Up/Down markets for major crypto assets. By placing buy orders on both Up and Down tokens at favorable prices before each period begins, it aims to lock in a payoff at settlement when one side settles at $1 per share. Execution is governed by configurable signals and risk controls.

---

## Architecture

### Period Structure

Markets align to 15-minute intervals in Eastern Time (:00, :15, :30, :45). The engine identifies markets via slug patterns:

- `btc-updown-15m-{timestamp}`
- `eth-updown-15m-{timestamp}`
- `sol-updown-15m-{timestamp}`
- `xrp-updown-15m-{timestamp}`

`timestamp` is the period start in Unix seconds. The Gamma API is used to locate current and upcoming period markets.

### Execution Sequence

1. **Pre-order submission** — Within `place_order_before_mins` of the next period, the engine optionally evaluates a signal on the current market. If the signal is favorable, it submits limit buy orders on both Up and Down for the next period. If unfavorable, it skips that period.

2. **Fill tracking** — Order status is monitored via the CLOB API (or simulated via price vs limit in simulation mode). Per-asset state tracks fills, expiry, and any partial exits.

3. **Both sides filled** — If the winning side’s price exceeds `sell_opposite_above` and time remaining is at or below `sell_opposite_time_remaining`, the engine sells the losing side and holds the winner to resolution.

4. **Single-side exposure** — If only one side fills, risk controls may trigger: sell the filled side when price reaches `danger_price` or when `danger_time_passed` elapses, then cancel the unfilled order.

5. **Mid-market orders** — When enabled, the engine can also submit limit orders on the current period market using dynamic prices, subject to signal and time conditions.

6. **Redemption** — A background process checks resolved markets at `market_closure_check_interval_seconds` and redeems winning positions, updating PnL.

### Flow Sketch

```
Current period (e.g. 2:00–2:15 ET)
    |
    +-- Optional: mid-market orders when signal permits
    |
    +-- Within place_order_before_mins of next period (e.g. 2:13)
            |
            +-- Evaluate signal on current market
            +-- Good signal: place Up + Down limit buys on next period
            +-- Bad signal: skip next period
                    |
                    v
Next period (e.g. 2:15)
    |
    +-- Fills: both / one / none
    +-- Both filled: optional sell-loser strategy, hold winner
    +-- One filled: danger logic, sell + cancel
    +-- Period end: redeem winner, record PnL
```

---

## Supported Markets

| Asset | Slug pattern | Example |
|-------|--------------|---------|
| BTC   | `btc-updown-15m-{ts}` | `btc-updown-15m-1771007400` |
| ETH   | `eth-updown-15m-{ts}` | `eth-updown-15m-1771007400` |
| SOL   | `sol-updown-15m-{ts}` | `sol-updown-15m-1771007400` |
| XRP   | `xrp-updown-15m-{ts}` | `xrp-updown-15m-1771007400` |

All four assets are handled in parallel with independent state.

---

## Capabilities

- Multi-asset execution for BTC, ETH, SOL, XRP
- Dual-sided pre-order strategy before each period
- Signal-based gating for pre-orders and mid-market orders
- Sell-loser logic when both sides are filled and winner price is high
- Single-side risk management (price or time-based exit)
- Optional mid-market limit orders on the current period
- Simulation mode using price vs limit for fill logic
- Automatic redemption of settled winning positions
- CLI for manual redemption by condition ID or wallet scan

---

## Prerequisites

- Rust 1.70 or newer (install via `rustup`)
- Polymarket CLOB credentials (API key, secret, passphrase)
- Optional: private key and proxy wallet for signing and redemption
- Network access to Polymarket Gamma and CLOB APIs

---

## Build and Run

```bash
git clone https://github.com/crellos/polymarket-arbitrage-bot-pre-order-15m-markets.git
cd polymarket-arbitrage-bot-pre-order-15m-markets

cargo build --release
```

Output binary: `target/release/polymarket-arbitrage-bot`.

---

## Configuration

Settings are loaded from `config.json` by default; use `-c` or `--config` to override.

```bash
cp config.json.example config.json
# Edit config.json with your Polymarket credentials
```

### Polymarket API

| Field | Purpose |
|-------|---------|
| `gamma_api_url` | Gamma API base URL |
| `clob_api_url` | CLOB API base URL |
| `api_key`, `api_secret`, `api_passphrase` | CLOB auth |
| `private_key` | Wallet key (hex) for signing; optional if monitoring only |
| `proxy_wallet_address` | Proxy wallet for trades and redemption |
| `signature_type` | CLOB signature type (e.g. 2) |

### Strategy

| Field | Purpose |
|-------|---------|
| `price_limit` | Limit price for pre-orders (e.g. 0.45) |
| `shares` | Size per Up and Down order |
| `place_order_before_mins` | Minutes before next period to place pre-orders |
| `check_interval_ms` | Main loop interval |
| `simulation_mode` | If true, no real orders; fills derived from price vs limit |
| `sell_opposite_above` | Min winner price to trigger sell-loser (e.g. 0.84) |
| `sell_opposite_time_remaining` | Max minutes left to trigger sell-loser |
| `market_closure_check_interval_seconds` | Interval for market resolution checks |

### Signal

| Field | Purpose |
|-------|---------|
| `enabled` | Use signal for pre-orders and mid-market orders |
| `stable_min`, `stable_max` | Good signal: Up/Down in this range |
| `clear_threshold` | Bad signal if either side >= this |
| `clear_remaining_mins` | Time condition for clear signal |
| `danger_price` | Sell matched side when price <= this (one-side risk) |
| `danger_time_passed` | Sell after this many minutes with one side only |
| `one_side_buy_risk_management` | `"price"`, `"time"`, or `"none"` |
| `mid_market_enabled` | Allow mid-market orders on current period |

---

## Commands

**Run engine (live or simulation):**

```bash
./target/release/polymarket-arbitrage-bot
./target/release/polymarket-arbitrage-bot --config /path/to/config.json
```

Enable simulation by setting `strategy.simulation_mode` to `true`.

**Redeem positions:**

```bash
# Redeem by condition ID
./target/release/polymarket-arbitrage-bot --redeem --condition-id 0x...

# Redeem all redeemable positions for proxy wallet
./target/release/polymarket-arbitrage-bot --redeem
```

`--redeem` requires `proxy_wallet_address` in config.

**Logging:**

```bash
RUST_LOG=info ./target/release/polymarket-arbitrage-bot
RUST_LOG=debug ./target/release/polymarket-arbitrage-bot
```

---

## Signal Logic

- **Favorable:** Up and Down in `stable_min`–`stable_max`, no clear condition. Allows pre-orders and (when enabled) mid-market orders.
- **Unfavorable:** Clear condition (e.g. one side >= `clear_threshold` near period end). Pre-orders for the next period are skipped.
- **Both filled:** Sell loser when winner price >= `sell_opposite_above` and time left <= `sell_opposite_time_remaining`, then hold winner to resolution.
- **One side filled:** Apply `one_side_buy_risk_management`: exit by `danger_price` or `danger_time_passed`, then cancel the other order.

---

## Risk Notice

This software interacts with real funds and live Polymarket APIs.

- No guarantee of profit; historical or simulated performance is not predictive.
- API or connectivity issues may prevent orders or redemption.
- Market risk includes slippage, illiquidity, and resolution rules.
- Secure API keys and private keys; do not commit them to version control.

Use simulation and small sizes before live use.
