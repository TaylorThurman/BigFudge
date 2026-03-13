# Architecture Blueprint v7 — Autonomous Trading Bot System

**Updated: March 8, 2026**
**Changes from v6:** Re-integrated multi-domain architecture (equities + crypto) that was dropped in v5. Crypto bot runs 24/7 on Mac via launchd, connects to exchanges via ccxt. Domain concept applies across all three gates, strategy lifecycle, shadow engine, dashboard, and audit trail.

---

## System Overview

A hybrid autonomous trading pipeline using Claude Code headless mode + Python orchestration. You operate as product owner from your phone (or any browser) — no terminal unless intervention needed. The system analyzes trades daily, suggests config and code changes, and routes everything through a **three-gate** human approval system via a **Discord-integrated web dashboard**.

The system operates across **two domains** — equities and crypto — that share pipeline infrastructure but have different execution targets, schedules, and risk profiles.

The three gates (per-domain, per-strategy):
- **Gate 1** — Config tuning (position size, stop loss, etc.) — approve and apply immediately
- **Gate 2** — Code changes (strategy logic, new strategies) — approve and deploy to shadow mode
- **Gate 3** — Shadow validation (paper trading with real market data) — promote to live or reject

---

## Hardware & Network

| Machine | Role | Details |
|---------|------|---------|
| MacBook Pro M1 16GB | Always-on server (lid closed) | Runs daily review pipeline, cron jobs, dashboard server, **crypto trading bots (24/7)** |
| MacBook Air M2 16GB | Daily driver | Development, manual intervention only |
| Windows Desktop | Equities trading execution | Das Trader Pro + equities bot + equities shadow trade engine, PowerShell git watcher |

All machines on the same local network. GitHub private monorepo is the single source of truth.

---

## Domain Concept

The system operates across **domains** — independent trading environments that share pipeline infrastructure but have different execution targets, schedules, and risk profiles.

### Domain: Equities
- **Execution target:** Das Trader Pro via websocket (Windows, localhost)
- **Market hours:** 9:30 AM – 4:00 PM ET, Mon–Fri
- **Bot location:** Windows desktop
- **Log retrieval:** SSH/SCP from Mac → Windows
- **Review schedule:** 16:30 ET weekdays (30 min after close)
- **Assets:** US equities (SPY, NVDA, AAPL, etc.)

### Domain: Crypto
- **Execution target:** Exchange APIs via ccxt (Binance, Coinbase, Kraken, etc.)
- **Market hours:** 24/7/365
- **Bot location:** Mac M1 Pro (same machine as pipeline)
- **Log retrieval:** Local filesystem read (no SSH needed)
- **Review schedule:** 00:00 UTC daily (configurable — could be multiple per day)
- **Assets:** Crypto pairs (BTC/USDT, ETH/USDT, SOL/USDT, etc.)

### What's Shared Across Domains
- Same GitHub monorepo
- Same three-gate approval pipeline (Gate 1 config, Gate 2 code, Gate 3 shadow)
- Same Discord bot (messages tagged with domain: 📈 equities, ₿ crypto)
- Same Claude Code headless for both analysis AND code generation
- Same dashboard (single UI, domain filter on every page)
- Same audit trail format (every event tagged with `domain` + `strategy`)
- Same per-strategy lifecycle: disabled → shadow → live
- Same per-strategy 3-day rule (per domain, independent)

### What's Different Per Domain
- Bot code and strategy implementations (separate base classes)
- Execution layer (Das Trader websocket vs ccxt REST/WS)
- Risk management (market hours vs 24/7, exchange health, funding rates)
- Config files (equities/bot/config.yaml vs crypto/bot/config.yaml)
- Log locations and retrieval method
- Pipeline schedule (16:30 ET vs 00:00 UTC)
- Backtester data source (trade logs vs exchange OHLCV API)
- Deployment mechanism (PowerShell watcher on Windows vs launchd + bash watcher on Mac)

---

## Core Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        MAC M1 PRO (Server)                         │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │           Crypto Trading Bots (run 24/7 as launchd service) │   │
│  │                                                              │   │
│  │  Exchange connectors (ccxt) ◄──► Binance / Coinbase / etc  │   │
│  │  Executes crypto strategies    Writes trade/health logs     │   │
│  │  Shadow trade engine (crypto)  Reads crypto/bot/config.yaml │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │         Daily Review Pipelines (cron — one per domain)      │   │
│  │                                                              │   │
│  │  EQUITIES PIPELINE (16:30 M-F)    CRYPTO PIPELINE (00:00)  │   │
│  │  ┌──────────────────────┐         ┌──────────────────────┐  │   │
│  │  │ Pull trade data SSH  │         │ Read local logs      │  │   │
│  │  │ Pull shadow logs SSH │         │ Read shadow logs     │  │   │
│  │  │ Health check         │  Gate   │ Health check         │  │   │
│  │  │ Shadow eval check    │ 1/2/3  │ Shadow eval check    │  │   │
│  │  │ Trade analysis       │ ──────►│ Trade analysis       │  │   │
│  │  │ Strategy review      │         │ Strategy review      │  │   │
│  │  │ Pattern scan         │         │ Pattern scan         │  │   │
│  │  │ Backtest proposed    │         │ Backtest proposed    │  │   │
│  │  └──────────────────────┘         └──────────────────────┘  │   │
│  │                                                              │   │
│  │  Shared: Claude Code headless, Discord bot, Gate 1/2/3 flow, │  │
│  │          staging branch workflow, audit trail                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              DASHBOARD (FastAPI + HTML/JS)                    │  │
│  │                                                               │  │
│  │  Gate 1, 2, 3 approvals — domain filter on every page        │  │
│  │  Strategy management (equities + crypto in one view)          │  │
│  │  Audit timeline + "Why is this broken?" mode                  │  │
│  │  Exposed via Cloudflare Tunnel                                │  │
│  │  Discord notifications link back here                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              AUDIT TRAIL (audit_log.jsonl)                    │  │
│  │  Every event timestamped, domain-tagged & persisted           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
            ┌──────────────┴──────────────┐
            │       DISCORD BOT           │
            │  Notifications + Deep Links  │
            │  to Dashboard approval pages  │
            │  Tagged: 📈 equities, ₿ crypto│
            └──────────────┬──────────────┘
                           │
            ┌──────────────┴──────────────┐
            │       GITHUB (Private)       │
            │  Monorepo — both domains     │
            │  main branch = production     │
            │  staging/equities/* = pending │
            │  staging/crypto/*   = pending │
            └──────────┬──────────┬────────┘
                       │          │
          ┌────────────┴──┐   ┌──┴────────────────────────────┐
          │ WINDOWS       │   │  MAC — Crypto Bots (24/7)     │
          │ (market hours)│   │                                │
          │ Das Trader ↔  │   │  Git watcher (bash/launchd)   │
          │ Equities Bot  │   │  Watches main, restarts crypto │
          │ + shadow eng  │   │  bots on new commits           │
          │               │   │                                │
          │ PS git watcher│   │  Crypto bots read updated      │
          │ OpenSSH for   │   │  config.yaml, load new strats  │
          │ log pulls     │   │                                │
          └───────────────┘   └────────────────────────────────┘
```

---

## Strategy Lifecycle

Every strategy is an **independent unit** with its own domain, mode, config, and gate history. Strategies never affect each other's flow — the 3-day rule, shadow evaluation, and approvals all operate per strategy, per domain.

### Strategy Modes

```
disabled ──→ shadow ──→ live
    ↑           ↑         │
    │           │         │ (significant code change approved)
    │           └─────────┘
    │                     │
    └─────────────────────┘ (manual disable or rejection)
```

- **`disabled`** — strategy code exists but doesn't run at all
- **`shadow`** — strategy runs against live market data, generates signals, but executes phantom trades instead of real orders; results logged for evaluation
- **`live`** — strategy executes real orders (through Das Trader for equities, through exchange API for crypto)

### Transition Rules

| Transition | Trigger | Guard |
|------------|---------|-------|
| disabled → shadow | Manual toggle on dashboard | — |
| disabled → live | Not allowed | Must pass through shadow first |
| shadow → live | Gate 3 approval (promote) | Shadow evaluation must be complete |
| shadow → disabled | Manual toggle or Gate 3 rejection | Not mid-phantom-trade |
| live → shadow | Gate 2 approval of significant code change, or manual toggle | Not mid-trade (holding position) |
| live → disabled | Manual toggle on dashboard | Not mid-trade (holding position) |

### Mid-Trade Guard

A strategy cannot change mode while it has an open position (live) or an unresolved phantom trade (shadow). The dashboard shows the strategy's current position state, and if it's holding, the toggle buttons are grayed out with an explanation. Every blocked attempt is logged as `STRATEGY_MODE_BLOCKED` in the audit trail.

For crypto: since markets are 24/7, mid-trade guards are especially important — there's no "market close" to force-resolve positions.

---

## Config Structure (Per-Strategy, Per-Domain)

### Equities Config

```yaml
# equities/bot/config.yaml

global:
  max_total_exposure: 10000     # across all live equities strategies
  max_strategies_live: 3         # max strategies in live mode at once
  max_daily_loss: 500

strategies:
  momentum_v2:
    mode: live
    position_size: 100
    stop_loss_pct: 0.02
    take_profit_pct: 0.04
    max_daily_trades: 5

  rsi_mean_reversion:
    mode: shadow
    shadow_start: "2026-03-08"
    shadow_days: 5
    position_size: 75
    stop_loss_pct: 0.015
    take_profit_pct: 0.03
    max_daily_trades: 3

  breakout_v1:
    mode: disabled
    position_size: 50
    stop_loss_pct: 0.025
    take_profit_pct: 0.05
    max_daily_trades: 4
```

### Crypto Config

```yaml
# crypto/bot/config.yaml

exchange:
  id: binance                    # ccxt exchange ID
  sandbox: true                  # Paper trading mode
  rate_limit_ms: 100             # Min ms between API calls
  # API keys loaded from environment variables, NOT this file
  # CRYPTO_API_KEY, CRYPTO_API_SECRET

pairs:
  - BTC/USDT
  - ETH/USDT
  - SOL/USDT

global:
  max_portfolio_drawdown_pct: 5.0    # Kill switch: stop all if portfolio -5%
  max_position_pct: 20.0             # No single position > 20% of portfolio
  max_open_positions: 5
  daily_loss_limit_usd: 500
  portfolio_base_currency: USDT
  max_strategies_live: 3

strategies:
  grid_trading:
    mode: live
    pairs: [BTC/USDT, ETH/USDT]
    grid_levels: 10
    grid_spacing_pct: 0.5
    position_size_usd: 100
    take_profit_pct: 1.5
    stop_loss_pct: 3.0

  trend_follow:
    mode: shadow
    shadow_start: "2026-03-08"
    shadow_days: 7              # 7 calendar days (24/7 market)
    pairs: [SOL/USDT]
    timeframe: 1h
    ema_fast: 12
    ema_slow: 26
    position_size_usd: 200
    trailing_stop_pct: 2.0

candle_interval: 5m
health_heartbeat_seconds: 60
```

Gate 1 config changes are **per-strategy** — a proposal to change `grid_trading.grid_spacing_pct` doesn't affect `trend_follow`. The 3-day rule is per-strategy per-domain.

---

## Project Structure

```
trading-system/                       ← Git monorepo (shared between all machines)
│
├── equities/                          ← Equities trading bot (runs on Windows)
│   ├── bot/
│   │   ├── main.py                    ← Entry point, websocket to Das Trader
│   │   ├── config.yaml                ← Per-strategy equities config
│   │   ├── strategies/
│   │   │   ├── base.py                ← Abstract strategy class (equities)
│   │   │   ├── momentum_v2.py
│   │   │   └── rsi_mean_reversion.py
│   │   ├── execution/
│   │   │   ├── das_connector.py       ← Das Trader websocket client
│   │   │   ├── order_manager.py       ← Order routing, position tracking
│   │   │   ├── risk_manager.py        ← Position sizing, drawdown limits
│   │   │   ├── strategy_router.py     ← Dispatches by mode: live/shadow/disabled
│   │   │   └── phantom_trader.py      ← Shadow trade logger (equities)
│   │   ├── data/
│   │   │   ├── market_data.py         ← Real-time data feed handling
│   │   │   └── indicators.py          ← Technical indicator calculations
│   │   └── utils/
│   │       ├── logger.py              ← Structured logging (JSON format)
│   │       └── health.py              ← Heartbeat writer, connection status
│   │
│   ├── logs/
│   │   ├── trades/                    ← Live trade logs: YYYY-MM-DD.json
│   │   ├── shadow/                    ← Phantom trades per strategy
│   │   │   └── {strategy_name}/YYYY-MM-DD.json
│   │   ├── health/
│   │   └── errors/
│   │
│   ├── tests/
│   │   ├── unit/
│   │   ├── integration/
│   │   ├── regression/                ← Tests ALL equities strategies
│   │   └── backtester.py              ← Historical data backtester
│   │
│   └── deploy/
│       ├── watcher.ps1                ← PowerShell: polls git main, restarts bot
│       └── setup_windows.md
│
├── crypto/                            ← Crypto trading bot (runs on Mac)
│   ├── bot/
│   │   ├── main.py                    ← Entry point, exchange connection loop
│   │   ├── config.yaml                ← Per-strategy crypto config
│   │   ├── strategies/
│   │   │   ├── base.py                ← Abstract strategy class (CryptoStrategy)
│   │   │   ├── grid_trading.py
│   │   │   └── trend_follow.py
│   │   ├── execution/
│   │   │   ├── exchange_connector.py  ← ccxt wrapper — unified exchange interface
│   │   │   ├── order_manager.py       ← Order routing, position tracking (crypto)
│   │   │   ├── risk_manager.py        ← Crypto-specific: funding rates, liquidation
│   │   │   ├── strategy_router.py     ← Dispatches by mode: live/shadow/disabled
│   │   │   └── phantom_trader.py      ← Shadow trade logger (crypto)
│   │   ├── data/
│   │   │   ├── market_data.py         ← Exchange websocket + REST OHLCV
│   │   │   └── indicators.py          ← Technical indicators for crypto
│   │   └── utils/
│   │       ├── logger.py
│   │       └── health.py              ← Exchange health, rate limit monitoring
│   │
│   ├── logs/
│   │   ├── trades/                    ← Live trade logs: YYYY-MM-DD.json
│   │   ├── shadow/                    ← Phantom trades per strategy
│   │   │   └── {strategy_name}/YYYY-MM-DD.json
│   │   ├── health/
│   │   └── errors/
│   │
│   ├── tests/
│   │   ├── unit/
│   │   ├── integration/
│   │   ├── regression/                ← Tests ALL crypto strategies
│   │   └── backtester.py              ← Pulls historical OHLCV from exchanges
│   │
│   └── deploy/
│       ├── watcher.sh                 ← Bash: polls git main, restarts crypto bots
│       ├── crypto-bot.plist           ← launchd service definition (auto-start, auto-restart)
│       └── setup_mac.md
│
├── shared/                            ← Code shared across domains
│   ├── audit.py                       ← Audit trail writer/reader
│   ├── discord_bot.py                 ← Discord notifications + slash commands
│   └── gate_utils.py                  ← Common gate logic (3-day rule, etc.)
│
├── review/                            ← Daily review pipeline (runs on Mac)
│   ├── pipeline.py                    ← Main orchestration — accepts --domain flag
│   ├── data_puller.py                 ← Domain-aware: SSH for equities, local for crypto
│   ├── health_checker.py              ← Validates data completeness (both domains)
│   ├── shadow_evaluator.py            ← Tracks shadow progress, triggers Gate 3
│   ├── analyst.py                     ← Claude API calls — per-strategy analysis
│   ├── code_builder.py                ← Claude Code headless for code changes
│   ├── validator.py                   ← Validation suite: pytest + in/out-of-sample + thresholds
│   ├── reporter.py                    ← Format and send Discord notifications
│   └── prompts/
│       ├── equities/                  ← Equities-specific analysis prompts
│       │   ├── trade_analyst.md
│       │   ├── strategy_advisor.md
│       │   └── pattern_scanner.md
│       ├── crypto/                    ← Crypto-specific analysis prompts
│       │   ├── trade_analyst.md
│       │   ├── strategy_advisor.md    ← Crypto market structure awareness
│       │   └── pattern_scanner.md
│       └── shared/
│           └── build_spec_writer.md   ← Shared across domains
│
├── dashboard/
│   ├── backend/
│   │   ├── app.py                     ← FastAPI entry point
│   │   ├── routes/                    ← REST + WebSocket routes
│   │   ├── auth.py                    ← Token-based auth (single user)
│   │   └── ingest.py                  ← JSON logs → SQLite sync
│   ├── frontend/                      ← HTML/JS served by FastAPI
│   └── deploy/
│       └── dashboard.plist            ← launchd service definition
│
├── state/
│   ├── equities/
│   │   └── context.json               ← 10-day rolling context for equities
│   ├── crypto/
│   │   └── context.json               ← 10-day rolling context for crypto
│   └── dashboard.db                   ← SQLite for dashboard queries
│
├── audit/                             ← Append-only event logs
│   └── YYYY-MM-DD.jsonl               ← Both domains in same files (tagged)
│
├── .claude/
│   ├── settings.json
│   ├── agents/
│   │   ├── backend-engineer.md
│   │   ├── test-engineer.md
│   │   └── code-reviewer.md
│   └── commands/
│       ├── build-strategy.md
│       └── code-review.md
│
├── CLAUDE.md                          ← Project context for Claude Code
├── requirements-equities.txt          ← Python deps for equities bot
├── requirements-crypto.txt            ← Python deps for crypto bot (ccxt, etc.)
├── requirements-review.txt            ← Python deps for review pipeline
└── README.md
```

---

## Crypto Bot Architecture Details

### Exchange Connectivity

The crypto bot uses [ccxt](https://github.com/ccxt/ccxt) as a unified exchange abstraction. The bot code doesn't care if it's trading on Binance, Coinbase, or Kraken — the connector handles the differences.

```python
# crypto/bot/execution/exchange_connector.py (simplified)

import ccxt

class ExchangeConnector:
    """
    Unified exchange interface using ccxt.
    Supports both REST and websocket for market data.
    """
    
    def __init__(self, exchange_id: str, config: dict):
        self.exchange = getattr(ccxt, exchange_id)({
            'apiKey': config['api_key'],
            'secret': config['api_secret'],
            'sandbox': config.get('sandbox', True),  # Paper trading by default
            'options': {'defaultType': 'spot'}
        })
    
    def place_order(self, symbol, side, amount, order_type='limit', price=None):
        return self.exchange.create_order(symbol, order_type, side, amount, price)
    
    def get_balance(self):
        return self.exchange.fetch_balance()
    
    def get_ohlcv(self, symbol, timeframe='5m', limit=100):
        return self.exchange.fetch_ohlcv(symbol, timeframe, limit=limit)
    
    def get_ticker(self, symbol):
        return self.exchange.fetch_ticker(symbol)
    
    def get_open_orders(self, symbol=None):
        return self.exchange.fetch_open_orders(symbol)
    
    def cancel_order(self, order_id, symbol):
        return self.exchange.cancel_order(order_id, symbol)
```

### Crypto Strategy Interface

Similar to equities but adapted for 24/7 markets and crypto-specific concepts:

```python
# crypto/bot/strategies/base.py

from abc import ABC, abstractmethod

class CryptoStrategy(ABC):
    """
    Base class for all crypto strategies.
    
    Key differences from equities Strategy:
    - No market open/close (24/7)
    - Uses candle-based loop instead of tick-by-tick
    - Must handle exchange-specific quirks (rate limits, etc.)
    - Funding rate awareness for futures/perps
    
    Like equities strategies, crypto strategies are mode-unaware.
    They call self.submit_order() and the execution layer routes
    based on mode (live → exchange API, shadow → phantom logger).
    """
    
    @abstractmethod
    def setup(self, connector, config: dict):
        """Initialize indicators, state, subscribe to pairs."""
        pass
    
    @abstractmethod
    def on_candle(self, symbol: str, candle: dict):
        """Called on each new candle close."""
        pass
    
    @abstractmethod
    def on_tick(self, symbol: str, ticker: dict):
        """Called on each price update (optional, higher frequency)."""
        pass
    
    @abstractmethod
    def on_fill(self, order: dict):
        """Called when an order fills. Update position tracking."""
        pass
    
    def should_pause(self) -> bool:
        """Override for pause conditions (extreme volatility, low liquidity, etc.)."""
        return False
    
    @abstractmethod
    def teardown(self):
        """Cleanup: cancel open orders, log final state."""
        pass
```

### Crypto vs. Equities: Key Architectural Differences

| Aspect | Equities | Crypto |
|--------|----------|--------|
| Broker/Exchange | Das Trader (websocket, localhost) | ccxt (REST + WS, remote APIs) |
| Market hours | 9:30–16:00 ET, M–F | 24/7/365 |
| Bot runs on | Windows desktop | Mac M1 Pro |
| Bot lifecycle | Starts morning, idles at close | Runs as persistent service (launchd) |
| Log retrieval | SSH/SCP from Mac → Windows | Local filesystem read |
| Position types | Long/short shares | Spot, limit, stop-limit (futures later) |
| Risk factors | Market hours, halts, gaps | Exchange downtime, API rate limits, funding rates, liquidation |
| Price data | Das Trader data feed | Exchange websocket + REST OHLCV |
| Strategy base class | `base.Strategy` (on_bar/on_tick) | `CryptoStrategy` (on_candle/on_tick/on_fill) |
| Backtester data | Historical trade logs | Historical OHLCV from exchange API |
| Shadow evaluation days | Trading days (M-F only) | Calendar days (24/7) |
| Pipeline schedule | 16:30 ET weekdays | 00:00 UTC daily (configurable) |
| Config path | equities/bot/config.yaml | crypto/bot/config.yaml |
| Deployment mechanism | PowerShell git watcher | Bash git watcher + launchd service |

### Crypto Bot as a Service (Mac)

The crypto bot runs 24/7 as a launchd service:

```xml
<!-- crypto/deploy/crypto-bot.plist -->
<!-- Install: cp crypto-bot.plist ~/Library/LaunchAgents/ && launchctl load ~/Library/LaunchAgents/crypto-bot.plist -->

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.trading.crypto-bot</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/python3</string>
        <string>-m</string>
        <string>crypto.bot.main</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/you/trading-system</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/you/trading-system/crypto/logs/stdout.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/you/trading-system/crypto/logs/stderr.log</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>CRYPTO_API_KEY</key>
        <string>SET_VIA_KEYCHAIN_OR_ENV</string>
        <key>CRYPTO_API_SECRET</key>
        <string>SET_VIA_KEYCHAIN_OR_ENV</string>
    </dict>
</dict>
</plist>
```

The git watcher for crypto on Mac:

```bash
#!/bin/bash
# crypto/deploy/watcher.sh
# Polls git main, restarts crypto bot on new commits

cd ~/trading-system

LOCAL=$(git rev-parse HEAD)
REMOTE=$(git ls-remote origin main | cut -f1)

if [ "$LOCAL" != "$REMOTE" ]; then
    echo "$(date): New commits detected, pulling and restarting crypto bot..."
    git pull origin main

    # Restart crypto bot via launchd
    launchctl stop com.trading.crypto-bot
    sleep 2
    launchctl start com.trading.crypto-bot

    echo "$(date): Crypto bot restarted with latest code."
fi
```

### Crypto Risk Management

```python
# crypto/bot/execution/risk_manager.py — additional concerns beyond equities

class CryptoRiskManager:
    """
    Extends common risk logic with crypto-specific checks.
    """
    
    def check_exchange_health(self) -> bool:
        """Verify exchange API is responsive before trading."""
        pass
    
    def check_rate_limits(self) -> bool:
        """Ensure we have API call budget remaining."""
        pass
    
    def check_funding_rate(self, symbol: str) -> float:
        """For futures: check funding rate before opening positions."""
        pass
    
    def check_spread(self, symbol: str, max_spread_pct: float) -> bool:
        """Verify bid-ask spread is reasonable. Low-liquidity pairs can have dangerous spreads."""
        pass
    
    def check_portfolio_exposure(self) -> dict:
        """Crypto-specific: check correlation exposure. BTC + ETH + SOL might all dump together."""
        pass
    
    def emergency_close_all(self):
        """Kill switch: cancel all open orders and market-sell all positions."""
        pass
```

### Crypto Backtester

Unlike equities (where we backtest against our own trade logs), crypto backtesting pulls historical data directly from exchanges:

```python
# crypto/tests/backtester.py

def fetch_historical_data(symbol, timeframe, days):
    """
    Pull historical OHLCV from exchange for backtesting.
    ccxt makes this easy — no need for separate data provider.
    """
    exchange = ccxt.binance({'enableRateLimit': True})
    since = exchange.parse8601((datetime.utcnow() - timedelta(days=days)).isoformat())
    ohlcv = exchange.fetch_ohlcv(symbol, timeframe, since=since, limit=1000)
    return pd.DataFrame(ohlcv, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])
```

### API Key Security

Crypto API keys are sensitive — never stored in config.yaml or git:

```
# Keys stored as environment variables on Mac
# Set in ~/.zshrc or via macOS Keychain

export CRYPTO_API_KEY="..."
export CRYPTO_API_SECRET="..."

# Or per-exchange if using multiple:
export BINANCE_API_KEY="..."
export BINANCE_API_SECRET="..."
export COINBASE_API_KEY="..."
export COINBASE_API_SECRET="..."
```

The launchd plist references these. The bot code reads them via `os.environ`.

---

## Equities Trading Bot (Windows)

- Bot connects to Das Trader Pro via websocket on `127.0.0.1` (localhost only, Windows only)
- Bot reads per-strategy config from equities/bot/config.yaml including `mode` field
- **Strategy Router** dispatches each strategy based on its mode:
  - `live` → real order executor (Das Trader API)
  - `shadow` → phantom trade logger
  - `disabled` → strategy's `setup()` is never called
- Bot writes structured JSON trade logs per day:
  - `trades/YYYY-MM-DD.json` for live trades
  - `shadow/{strategy_name}/YYYY-MM-DD.json` for phantom trades
- Strategies inherit from `base.Strategy` class with `setup/on_bar/on_tick/teardown` interface
- Strategy code doesn't know its mode — it calls `self.submit_order()` and the execution layer routes it
- **PowerShell git watcher** polls main branch every 60s, auto-pulls and restarts bot on changes
- **OpenSSH on Windows** allows Mac M1 to pull both live and shadow trade logs via SCP

---

## Shadow Trade Engine

Both domains use the same pattern: **shadow strategies run full strategy logic but route order execution to a phantom trade logger instead of the real broker/exchange.**

### How It Works (Same Pattern, Both Domains)

```
                    ┌─────────────────────────┐
                    │     Market Data Feed     │
                    │  Equities: Das Trader WS │
                    │  Crypto: Exchange WS/REST│
                    └────────────┬────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │     Strategy Router      │
                    │  reads config.yaml mode  │
                    └─┬──────────┬──────────┬─┘
                      │          │          │
              mode:live   mode:shadow   mode:disabled
                      │          │          │
                      v          v          └→ skip
              ┌───────────┐ ┌───────────┐
              │ Real Order│ │ Phantom   │
              │ Executor  │ │ Trade     │
              │ Eq: DAS   │ │ Logger    │
              │ Cr: ccxt  │ │           │
              └─────┬─────┘ └─────┬─────┘
                    │             │
                    v             v
              trades/         shadow/{strategy}/
              YYYY-MM-DD.json YYYY-MM-DD.json
```

The strategy code itself doesn't know whether it's live or shadow — it calls `self.submit_order()`. The bot's execution layer checks the mode and routes accordingly.

### Shadow Evaluation: Equities vs Crypto

| Aspect | Equities | Crypto |
|--------|----------|--------|
| Shadow days | Trading days (M-F) | Calendar days (24/7) |
| Typical evaluation | 5 trading days (~1 week) | 5-7 calendar days |
| Force-close rule | End of trading day | Optional — crypto can carry overnight |
| Data source for phantom tracking | Das Trader price feed | Exchange ticker/candles |

---

## Domain-Aware Data Puller

```python
# review/data_puller.py

def pull_trade_data(domain: str, date: str):
    """Domain-aware: SSH for equities, local for crypto."""
    if domain == "equities":
        # SSH into Windows, pull trade logs + shadow logs
        scp(f"{WIN_USER}@{WIN_IP}:C:/trading-system/equities/logs/trades/{date}.json", "./")
        scp(f"{WIN_USER}@{WIN_IP}:C:/trading-system/equities/logs/shadow/", "./", recursive=True)
    elif domain == "crypto":
        # Local file — crypto bot runs on this same Mac
        log_path = Path.home() / "trading-system" / "crypto" / "logs" / "trades" / f"{date}.json"
        if not log_path.exists():
            raise FileNotFoundError(f"No crypto trade log for {date}")
        return json.loads(log_path.read_text())
```

---

## 24/7 Markets: Crypto Review Scheduling

Unlike equities with a clear "market close" trigger, crypto runs continuously. Default is once daily at midnight UTC:

```
# Default: once daily at midnight UTC
0 0 * * *  python -m review.pipeline --domain crypto

# Alternative: every 8 hours for more active management
0 0,8,16 * * *  python -m review.pipeline --domain crypto

# Weekend: equities off, crypto gets extra analysis
0 10 * * 6  python -m review.pipeline --domain crypto --weekly
```

The "daily" concept for crypto uses UTC day boundaries. Trade logs are split by UTC date. The rolling 10-day context window works the same way.

---

## Why Discord Over Telegram

- **Richer embeds** — structured fields, colors, inline buttons that link directly to dashboard actions
- **Threads** — each pipeline run can get its own thread for clean conversation history
- **Webhooks are simpler** — no bot polling needed for send-only notifications
- **Bot interactions** — slash commands for quick status checks (`/status`, `/last-run`, `/audit <date>`)
- **Channel organization** — separate channels per gate plus alerts and audit
- **Mobile notifications** — Discord mobile app works well for push alerts

---

## Discord Channel Structure

```
📁 trading-bot (server)
├── #pipeline-runs      — Daily run summaries, health checks (both domains)
├── #gate-1-config      — Config change proposals (tagged: 📈 or ₿)
├── #gate-2-code        — Code change proposals (tagged: 📈 or ₿)
├── #gate-3-shadow      — Shadow evaluation progress + completion (tagged: 📈 or ₿)
├── #alerts             — Failures, anomalies, circuit breakers (both domains)
├── #audit-log          — Daily digest of all approved/rejected changes
└── #general            — Manual commands, ad hoc discussion
```

---

## Dashboard

The dashboard is the **single place** for all gate interactions and strategy management across both domains. Discord is notification-only. Every page has a domain filter (All / Equities / Crypto).

### Tech Stack
- **FastAPI** backend (Python, runs on Mac M1)
- **Simple HTML/JS frontend** (no heavy framework — personal tool)
- **Exposed via Cloudflare Tunnel** for mobile access anywhere
- **Auth** — simple token-based auth or Cloudflare Access (zero trust) since it's single-user

### Dashboard Pages

#### Home / Status
- Domain toggle: All / Equities / Crypto
- Current bot status per domain (running, stopped, error)
- Strategy overview table (name, domain, mode, today's P&L per strategy)
- Active shadow evaluations with progress bars
- Today's combined P&L (equities + crypto, separate colors)
- Last pipeline run summary per domain
- Quick links to recent pending approvals across all gates

#### Gate 1 — Config Approvals
- Domain filter at top
- List of pending config change proposals (per-strategy, per-domain)
- Each proposal shows strategy name, domain, current mode, config diff, reasoning, backtest
- Actions: Approve / Reject / Suggest Modification

#### Gate 2 — Code Approvals
- Domain filter at top
- Each proposal shows strategy affected, domain, change classification (significant/minor)
- Full validation suite results (pytest, in-sample, out-of-sample, thresholds)
- Actions: Approve & Deploy to Shadow / Approve & Keep Current Mode / Reject / Request Changes

#### Gate 3 — Shadow Evaluations
- Domain filter at top
- Active evaluations with progress bars (note: crypto counts calendar days, equities trading days)
- Completed evaluations with backtest-vs-shadow comparison
- Actions: Promote to Live / Reject / Extend Shadow

#### Strategy Management
- All strategies from both domains in one control table:

```
Domain     Strategy              Mode        Status          Actions
───────────────────────────────────────────────────────────────────────
📈 Equity  momentum_v2          🟢 live     No position     [Shadow] [Disable]
📈 Equity  rsi_mean_reversion   🟡 shadow   Day 3/5         [Disable] [Extend]
📈 Equity  breakout_v1          ⚫ disabled  —              [Shadow]
₿ Crypto   grid_trading         🟢 live     2 BTC grids     [Shadow] [Disable]
₿ Crypto   trend_follow         🟡 shadow   Day 2/7         [Disable] [Extend]
```

- Domain filter at top
- Per-strategy performance history (mini chart of daily P&L)
- Per-strategy 3-day rule countdown timers

#### Audit Timeline
- Domain filter at top
- Searchable, filterable timeline of every event
- Filter by: date range, event type, gate, outcome, **domain**, **strategy name**
- "Why is this broken?" mode — select a strategy + config param and see every change

#### Settings
- Discord webhook URL
- Pipeline schedules (per domain)
- Default shadow evaluation period (per domain — trading days for equities, calendar days for crypto)
- Minimum performance thresholds (per domain)
- Circuit breaker thresholds (per domain)
- Manual pipeline trigger button (per domain)

---

## Three-Gate Approval System

All three gates work identically across both domains. The only differences are in the data sources, schedules, and domain-specific context in the analysis prompts.

### Gate 1 — Config Changes (Per-Strategy, Per-Domain)
Same as v6 — but domain-tagged. Config proposals are per-strategy within a domain.

### Gate 2 — Code Changes (with Validation Suite)
Same as v6 — but staging branches are namespaced: `staging/equities/2026-03-07` or `staging/crypto/2026-03-07`.

### Gate 3 — Shadow Evaluation
Same as v6 — but shadow days are measured differently:
- **Equities:** trading days (M-F only)
- **Crypto:** calendar days (24/7 market)

### 3-Day Rule (Per-Strategy, Per-Domain)
No config or code changes to the same strategy within 3 days of its last approved change. The rule is per-strategy and per-domain — changes to `momentum_v2` (equities) don't affect `grid_trading` (crypto).

---

## Discord Message Examples

### Equities Pipeline Run (`#pipeline-runs`)
```
📈 Equities Pipeline — March 8, 2026

📊 Today's Trades: 12 executed, 8 winners (66.7%)
💰 P&L: +$342.50
📈 10-Day Sharpe: 1.38
👻 Shadow: rsi_mean_reversion — Day 3/5, +$215 phantom

🔧 Config Changes: 1 proposed (momentum_v2) → [Review]
💻 Code Changes: 0 proposed
⏱️ 3-Day Rule: momentum_v2 clear, rsi_mean_reversion locked (Day 1)

🔗 Full Report: https://your-dashboard.example.com/runs/20260308
```

### Crypto Pipeline Run (`#pipeline-runs`)
```
₿ Crypto Pipeline — March 8, 2026 (UTC)

📊 24h Trades: 18 fills, 14 winners (77.8%)
💰 P&L: +$127.30
💱 Exchange: Binance (sandbox mode)

📋 Per Strategy:
  Grid (BTC/USDT): +$85.80 — 12 fills
  Grid (ETH/USDT): +$41.50 — 6 fills, tighter range today
👻 Shadow: trend_follow — Day 2/7, +$62 phantom

BTC held $67k-68k range, ideal for grid. ETH correlation
drifted slightly — monitoring. SOL trend_follow showing
early promise in shadow.

Funding rates: BTC +0.01% (neutral), ETH +0.03% (slightly long-heavy)

🔧 Suggestions:
1️⃣ [CONFIG] Widen ETH grid spacing: 0.5% → 0.7%
   Rationale: ETH swings wider than BTC, current grid
   getting filled too quickly with small profit per fill.

🔗 Full Report: https://your-dashboard.example.com/runs/20260308-crypto
```

### Emergency: Crypto Kill Switch via Discord
```
You: /kill crypto

Bot: ⚠️ Emergency shutdown initiated for crypto domain.
     ✅ 4 orders cancelled (2 BTC, 2 ETH)
     ✅ Positions closed: BTC 0.015 @ $67,420, ETH 0.5 @ $3,847
     ✅ Emergency P&L: -$12.30 (slippage on market close)
     Crypto bot stopped. Reply "/start crypto" to restart.
```

---

## Audit Trail System

Same as v6, but every event is tagged with `domain`:

```json
{
  "id": "evt_20260308_000042_x7y8z9",
  "timestamp": "2026-03-08T00:00:42Z",
  "event_type": "GATE1_PROPOSED",
  "category": "gate_1",
  "domain": "crypto",
  "strategy": "grid_trading",
  "pipeline_run_id": "run_20260308_0000_crypto",
  "data": {
    "changes": {
      "grid_spacing_pct": { "old": 0.5, "new": 0.7 }
    },
    "reasoning": "ETH swings wider than BTC..."
  }
}
```

The audit timeline on the dashboard can filter by domain, making it easy to trace "why did this crypto strategy change?"

---

## State Management

### context.json (10-Day Rolling Window) — Per Domain
- `state/equities/context.json` and `state/crypto/context.json`
- Each contains: last 10 days of trade summaries (per strategy), current config snapshot, pending proposals, 3-day rule state, active shadow evaluations
- Fed into the corresponding domain's Claude API prompts for continuity

### Audit Log (Append-Only)
- `audit/YYYY-MM-DD.jsonl` — both domains in same files, tagged with `domain` field
- Never modified, only appended
- Source of truth for "what happened and why"

### config.yaml (Per Domain)
- `equities/bot/config.yaml` and `crypto/bot/config.yaml`
- Per-strategy structure with `mode` field
- No secrets
- Every change is a git commit referencing the audit event ID

---

## Build Phases

| Phase | What | Gate System |
|-------|------|-------------|
| 1 — Equities Bot on Paper | Bot runs strategies in shadow mode only, logs phantom trades | None |
| 2 — Reports Only | Pipeline runs equities analysis, sends reports to Discord | None |
| 3 — Gate 1 Config (Equities) | Config changes for equities strategies via dashboard | Gate 1 |
| 4 — Gate 2 Code (Equities) | Code changes with validation suite for equities | Gate 1 + Gate 2 |
| 5 — Gate 3 Shadow (Equities) | Shadow validation before going live for equities | Gate 1 + Gate 2 + Gate 3 |
| 6 — Crypto Bot Foundation | ccxt connector, first crypto strategy, launchd service | None (sandbox) |
| 7 — Crypto Pipeline Integration | Crypto analysis, domain-aware pipeline, crypto Gate 1/2/3 | Gate 1 + Gate 2 + Gate 3 |
| 8 — Go Live (Both Domains) | Full system, all three gates, both domains | Gate 1 + Gate 2 + Gate 3 |

---

## Build Order (Implementation)

1. **Equities Foundation** — repo structure, per-strategy config.yaml schema, base.Strategy class, trade log format, shadow trade log format
2. **Pipeline skeleton** — cron, SSH data pull (live + shadow logs), health checks, context.json management, `--domain` flag infrastructure
3. **Analysis layer** — Claude API prompts (trade analyst, strategy advisor, pattern scanner) — per-strategy equities analysis
4. **Audit trail** — event logging with domain + strategy fields, JSONL writer, query helpers
5. **Discord integration** — webhook notifications for all channels, bot for slash commands, domain tags (📈/₿)
6. **Dashboard** — FastAPI + HTML/JS: Home/Status, Gate 1/2/3, Strategy Management, Audit Timeline, Settings — with domain filter
7. **Cloudflare Tunnel** — expose dashboard securely for mobile access
8. **Gate 1 flow** — per-strategy config proposal → dashboard review → apply → git commit
9. **Gate 2 flow** — build spec with change classification → Claude Code headless → staging branch → validation suite → dashboard review → merge + auto-shadow
10. **Shadow trade engine (equities)** — phantom trade logger in Windows bot, strategy router by mode, shadow log writer
11. **Gate 3 flow** — shadow log collection via SSH → daily progress → evaluation completion → dashboard review → promote/reject/extend
12. **Windows git watcher** — PowerShell auto-pull + restart on main branch changes
13. **Crypto bot foundation** — exchange connector (ccxt wrapper), one simple crypto strategy (grid trading), crypto-specific risk manager, trade logging (same JSON schema with crypto fields), health heartbeat + exchange health monitor, launchd service + bash git watcher on Mac, crypto backtester (historical OHLCV), tests for all crypto components — deploy on Mac, connect to exchange sandbox
14. **Crypto pipeline integration** — crypto prompts for trade analyst/strategy advisor/pattern scanner, domain-aware data puller (local read for crypto), crypto review schedule (cron at midnight UTC), crypto health checks (every 4 hours), separate state files (state/crypto/*), Gate 1 + Gate 2 + Gate 3 working for crypto domain, shadow trade engine for crypto bot
15. **Go live** — switch equities from paper to live (minimal positions), switch crypto from sandbox to live (minimal positions), tighten risk limits both domains, monitor daily for 2+ weeks per domain before scaling

---

## Cron Schedule (Mac M1 Pro)

```crontab
# EQUITIES
# Daily review pipeline — 30 min after market close
30 16 * * 1-5  cd ~/trading-system && python -m review.pipeline --domain equities >> logs/equities-pipeline.log 2>&1

# Morning health check — verify equities bot started and is connected
35 9 * * 1-5   cd ~/trading-system && python -m review.health_checker --domain equities >> logs/equities-health.log 2>&1

# Weekend summary — broader analysis (equities only, market closed)
0 10 * * 6     cd ~/trading-system && python -m review.pipeline --domain equities --weekly >> logs/equities-pipeline.log 2>&1

# CRYPTO
# Daily review (midnight UTC — crypto runs 24/7)
0 0 * * *      cd ~/trading-system && python -m review.pipeline --domain crypto >> logs/crypto-pipeline.log 2>&1

# Health check — every 4 hours (crypto runs 24/7)
0 */4 * * *    cd ~/trading-system && python -m review.health_checker --domain crypto >> logs/crypto-health.log 2>&1

# Git watcher — check for new commits every 60 seconds (restarts crypto bot)
* * * * *      cd ~/trading-system && bash crypto/deploy/watcher.sh >> logs/crypto-watcher.log 2>&1
```

---

## CLAUDE.md (For Claude Code Headless)

```markdown
# CLAUDE.md — Trading System

## What This Repo Is
Autonomous multi-domain trading system (equities + crypto) with human-in-the-loop three-gate approval pipeline. Strategies are independent units with lifecycle: disabled → shadow → live.

## Domains
1. **Equities** — Python bot connecting to Das Trader Pro via websocket (Windows).
   Strategies in equities/bot/strategies/ inheriting base.Strategy.
2. **Crypto** — Python bot connecting to exchanges via ccxt library.
   Runs on Mac as a launchd service. Trades crypto pairs 24/7.
   Strategies in crypto/bot/strategies/ inheriting CryptoStrategy.

## Repo Layout
- `equities/` — Equities bot code, logs, tests, deploy scripts
- `crypto/` — Crypto bot code, logs, tests, deploy scripts
- `shared/` — Code used by both domains (audit, discord, gate utils)
- `review/` — Daily analysis pipeline (runs on Mac, separate
  schedules for equities (16:30 ET) and crypto (00:00 UTC))
- `dashboard/` — FastAPI dashboard with domain filter
- `state/` — Per-domain context files + SQLite
- `audit/` — Append-only event logs (both domains, tagged)

## Rules
- NEVER modify config.yaml directly — config changes go through Gate 1
- NEVER push to main — always work on staging/* branches
- NEVER modify audit trail files
- NEVER modify shadow trade logs
- All code changes must have corresponding tests
- All strategy changes must include regression tests for ALL strategies in that domain
- Strategy code must not be mode-aware — use self.submit_order()
- Trade logs and shadow logs are read-only input
- Shared code in shared/ must not import from equities/ or crypto/

## Key Constraints — Equities
- Das Trader connection is websocket-only, localhost only, Windows only
- Bot must handle websocket disconnections with auto-reconnect
- New strategies must inherit from equities.bot.strategies.base.Strategy

## Key Constraints — Crypto
- Uses ccxt for exchange connectivity — NEVER hardcode exchange-specific APIs
- API keys loaded from environment variables, never from config.yaml
- Must respect exchange rate limits (ccxt handles most, but be aware)
- New strategies must inherit from crypto.bot.strategies.base.CryptoStrategy

## Build Spec Classification
When writing a build spec, always classify the change and note which domain:
- SIGNIFICANT: touches entry/exit logic, signal generation, adds/removes indicators, new strategy
- MINOR: refactor, logging, comments, test-only changes with identical runtime behavior

## Crypto Strategy Interface
- setup(connector, config) — initialize
- on_candle(symbol, candle) — new candle close
- on_tick(symbol, ticker) — price update (optional)
- on_fill(order) — order filled
- teardown() — cleanup

## Equities Strategy Interface
- setup() — initialize indicators, state
- on_bar(bar) — each new price bar
- on_tick(tick) — each tick (optional)
- teardown() — cleanup at end of day

## Testing
- Unit tests: pytest {domain}/tests/unit/ -v
- Integration tests: pytest {domain}/tests/integration/ -v
- Regression: pytest {domain}/tests/regression/ -v
- Backtest: python -m {domain}.tests.backtester --config {domain}/bot/config.yaml --strategy <n>
```

---

## Failure Handling

| Failure | Response | Audit Event |
|---------|----------|-------------|
| SSH to Windows fails | Retry 3x, then alert Discord `#alerts` | `PIPELINE_FAILED` |
| Claude API timeout | Retry with exponential backoff, then skip analysis | `PIPELINE_FAILED` |
| Backtest crashes | Skip change proposal, report observation-only | `BACKTEST_RUN` (with error) |
| Claude Code headless fails | Log error, skip Gate 2, alert Discord | `GATE2_PROPOSED` (with error) |
| Validation suite fails (pytest) | Include failures in Gate 2 card, flag as 🔴 | `GATE2_PROPOSED` (with failures) |
| Out-of-sample significantly worse than in-sample | Flag as 🟡 possible overfit warning | `GATE2_PROPOSED` (with warning) |
| Shadow strategy has no trades during evaluation | Extend automatically, flag on dashboard | `GATE3_PROGRESS` (no trades) |
| Dashboard unreachable | Discord notifications still work as fallback | `CIRCUIT_BREAKER_TRIPPED` |
| Git push fails | Retry 3x, then alert and halt | `PIPELINE_FAILED` |
| Approval timeout (24h) | Auto-reject, log as expired | `GATE1_REJECTED` / `GATE2_REJECTED` |
| Mode change blocked (mid-trade) | Show warning on dashboard, log attempt | `STRATEGY_MODE_BLOCKED` |
| Exchange API down (crypto) | Pause crypto bot, alert Discord, retry every 5 min | `EXCHANGE_HEALTH_FAILED` |
| Exchange rate limit hit (crypto) | Back off per ccxt, log warning | `RATE_LIMIT_WARNING` |
| Crypto bot crash | launchd auto-restarts, alert Discord if repeated | `BOT_CRASH` |

---

## Security Notes

- Dashboard behind Cloudflare Tunnel + Access (zero trust, your email only)
- Discord bot token stored in `.env`, never committed
- Claude API key stored in `.env`, never committed
- SSH keys for Windows connection stored on Mac M1 only
- **Crypto API keys stored as environment variables on Mac, never in config.yaml or git**
- No secrets in config.yaml or context.json
- Git repo is private

---

## Future Enhancements

- Run a local model on Mac for cheaper analysis calls
- Add market data feeds for pre-market equities analysis
- Multi-strategy portfolio optimization (cross-domain)
- Multiple exchange support for crypto (arbitrage opportunities)
- Futures/perpetuals support for crypto (funding rate strategies)
- Cross-domain correlation analysis (crypto-equities hedging)
- Dashboard: mobile-optimized PWA
- Dashboard: push notifications for threshold alerts
- Dashboard: interactive backtest launcher
- Dashboard: strategy editor (preview config changes before pushing)
