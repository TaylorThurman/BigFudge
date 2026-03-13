# Trading Bot Agentic System — Architecture Blueprint

*Version 0.4 — March 8, 2026*
*Updated: Multi-domain architecture + dashboard for performance monitoring & trade review*

---

## System Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                      YOU (Product Owner)                          │
│                                                                   │
│  Telegram: Gate 1/2 approvals, kill switches, alerts             │
│  Dashboard: Performance, live trades, history, backtests         │
│  Escape hatch: SSH into Mac for manual intervention              │
│                                                                   │
│  Reports come per-domain: equities and crypto separately         │
└──────────────┬──────────────────────────┬────────────────────────┘
               │                          │
         Telegram messages          Browser (any device)
               │                          │
┌──────────────▼──────────────────────────▼────────────────────────┐
│                 MAC (M1 Pro — always on, lid closed)              │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │           Crypto Trading Bots (run 24/7 on Mac)             │ │
│  │                                                              │ │
│  │  Exchange connectors (ccxt) ◄──► Binance / Coinbase / etc  │ │
│  │  Executes crypto strategies    Writes trade/health logs     │ │
│  │  Managed by systemd/launchd    Restarts on failure          │ │
│  │  Pulls config from crypto-bot/config.yaml                   │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │         Daily Review Pipelines (cron — one per domain)      │ │
│  │                                                              │ │
│  │  EQUITIES PIPELINE (16:30 M-F)    CRYPTO PIPELINE (00:00)  │ │
│  │  ┌──────────────────────┐         ┌──────────────────────┐  │ │
│  │  │ Pull trade data SSH  │         │ Read local logs      │  │ │
│  │  │ Health check         │         │ Health check         │  │ │
│  │  │ Trade analysis       │  Gate 1 │ Trade analysis       │  │ │
│  │  │ Strategy review      │ ──────► │ Strategy review      │  │ │
│  │  │ Pattern scan         │         │ Pattern scan         │  │ │
│  │  │ Backtest proposed    │         │ Backtest proposed    │  │ │
│  │  └──────────────────────┘         └──────────────────────┘  │ │
│  │                                                              │ │
│  │  Shared: Claude Code headless, Telegram bot, Gate 1/2 flow,  │ │
│  │          staging branch workflow                              │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Dashboard (FastAPI + React)                     │ │
│  │                                                              │ │
│  │  FastAPI backend ◄──► SQLite (trade history, backtests)    │ │
│  │       │                                                      │ │
│  │       ├── REST API: performance, trades, strategies, logs   │ │
│  │       ├── WebSocket: live positions + P&L (real-time)       │ │
│  │       └── Serves React SPA (static build)                   │ │
│  │                                                              │ │
│  │  Accessible from any device on local network / Tailscale    │ │
│  │  Auth: token-based (single user)                            │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────────────────────────┐  │
│  │  State Store     │  │  Claude Code (escape hatch)          │  │
│  │  • SQLite DB     │  │  Available via terminal/SSH when     │  │
│  │  • context.json  │  │  you want manual control. Agent      │  │
│  │  • change_log    │  │  Teams, slash commands, etc.         │  │
│  │  • build_specs/  │  │                                      │  │
│  └──────────────────┘  └──────────────────────────────────────┘  │
└──────────────┬───────────────────────────────────────────────────┘
               │ git push (main branch)
┌──────────────▼───────────────────────────────────────────────────┐
│                    GITHUB (Private Monorepo)                      │
│  main        ← config changes + merged code changes              │
│  staging/*   ← code built by headless Claude Code, awaiting Gate 2│
│                                                                   │
│  Bot code for BOTH domains lives in this repo                    │
└──────────────┬───────────────────────────────────────────────────┘
               │ 
          ┌────┴────────────────────────────┐
          │ auto-pull main + restart         │ git pull on Mac
          │                                  │ (local — no SSH needed)
┌─────────▼────────────────────────────┐  ┌─▼────────────────────────────────┐
│  WINDOWS DESKTOP (market hours)      │  │  MAC — Crypto Bots (24/7)        │
│                                      │  │                                   │
│  Das Trader ◄──ws──► Equities Bot   │  │  Git watcher (launchd)           │
│             127.0.0.1                │  │  Watches main, restarts crypto   │
│                                      │  │  bots on new commits             │
│  Git Watcher (PowerShell)            │  │                                   │
│  Watches main, restarts bot          │  │  Crypto bots read updated        │
│  OpenSSH Server for log pulls        │  │  config.yaml, load new strategies│
└──────────────────────────────────────┘  └──────────────────────────────────┘
```

---

## Domain Concept

The system now operates across **domains** — independent trading environments that share pipeline infrastructure but have different execution targets, schedules, and risk profiles.

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
- Same two-gate approval pipeline (Gate 1 concepts, Gate 2 deploys)
- Same Telegram bot (messages tagged with domain: 📈 equities, ₿ crypto)
- Same Claude Code headless for both analysis AND code generation
- Same state management pattern (separate state files per domain)
- Same 3-day rule between changes (per domain, independent)
- Same dashboard (single UI, domain filter on every page)
- Same SQLite database for all historical data

### What's Different Per Domain
- Bot code and strategy implementations
- Exchange/broker connectors
- Risk parameters and position sizing logic
- Review schedules and health check timing
- Trade log formats (similar structure, different fields)
- Backtester data sources

---

## Project Structure

```
trading-system/                        ← Git monorepo (shared between all machines)
│
├── equities/                          ← Equities trading bot (runs on Windows)
│   ├── bot/
│   │   ├── main.py                    ← Entry point, websocket connection to Das Trader
│   │   ├── config.yaml                ← Strategy parameters (daily cycle can modify)
│   │   ├── strategies/
│   │   │   ├── base.py                ← Abstract strategy class (equities)
│   │   │   ├── vwap_momentum.py
│   │   │   └── mean_reversion.py
│   │   ├── execution/
│   │   │   ├── das_connector.py       ← Das Trader websocket client
│   │   │   ├── order_manager.py       ← Order routing, position tracking
│   │   │   └── risk_manager.py        ← Position sizing, drawdown limits, kill switches
│   │   ├── data/
│   │   │   ├── market_data.py         ← Real-time data feed handling
│   │   │   └── indicators.py          ← Technical indicator calculations
│   │   └── utils/
│   │       ├── logger.py              ← Structured logging (JSON format)
│   │       └── health.py              ← Heartbeat writer, connection status
│   │
│   ├── logs/                          ← Written by bot on Windows, pulled by Mac
│   │   ├── trades/                    ← One JSON file per day: 2026-03-07.json
│   │   ├── health/                    ← Connection status, heartbeats
│   │   └── errors/                    ← Exceptions, disconnections
│   │
│   ├── tests/
│   │   ├── test_strategies.py
│   │   ├── test_risk_manager.py
│   │   ├── test_das_connector.py
│   │   └── backtester.py              ← Run historical data through strategies
│   │
│   └── deploy/
│       ├── watcher.ps1                ← PowerShell: polls git main, restarts bot
│       └── setup_windows.md           ← Instructions for Windows machine setup
│
├── crypto/                            ← Crypto trading bot (runs on Mac)
│   ├── bot/
│   │   ├── main.py                    ← Entry point, exchange connections, strategy loop
│   │   ├── config.yaml                ← Strategy params, exchange config, pair selection
│   │   ├── strategies/
│   │   │   ├── base.py                ← Abstract strategy class (crypto)
│   │   │   ├── grid_trading.py        ← Example: grid strategy for range-bound pairs
│   │   │   └── trend_follow.py        ← Example: momentum/trend following
│   │   ├── execution/
│   │   │   ├── exchange_connector.py  ← ccxt wrapper — unified exchange interface
│   │   │   ├── order_manager.py       ← Order routing, position tracking (crypto)
│   │   │   └── risk_manager.py        ← Crypto-specific: funding rates, liquidation
│   │   ├── data/
│   │   │   ├── market_data.py         ← Websocket feeds from exchanges
│   │   │   └── indicators.py          ← Technical indicators (shared logic where possible)
│   │   └── utils/
│   │       ├── logger.py              ← Structured logging (JSON, same format as equities)
│   │       └── health.py              ← Heartbeat, exchange connection status
│   │
│   ├── logs/                          ← Written locally on Mac (no SSH needed)
│   │   ├── trades/                    ← One JSON file per day: 2026-03-07.json
│   │   ├── health/                    ← Connection status, heartbeats
│   │   └── errors/                    ← Exceptions, disconnections, API rate limits
│   │
│   ├── tests/
│   │   ├── test_strategies.py
│   │   ├── test_risk_manager.py
│   │   ├── test_exchange_connector.py
│   │   └── backtester.py              ← Backtest against historical OHLCV data
│   │
│   └── deploy/
│       ├── watcher.sh                 ← Bash: polls git main, restarts crypto bots
│       ├── crypto-bot.plist           ← launchd service definition (auto-start, auto-restart)
│       └── setup_mac.md               ← Instructions for Mac crypto bot setup
│
├── shared/                            ← Code shared across domains
│   ├── models.py                      ← Common data models: Trade, Position, Signal, etc.
│   ├── indicators.py                  ← Indicator functions usable by both domains
│   ├── trade_log.py                   ← Unified trade log schema + validation
│   └── risk_common.py                 ← Shared risk primitives (max drawdown calc, etc.)
│
├── dashboard/                         ← Web dashboard (runs on Mac as a service)
│   ├── backend/
│   │   ├── main.py                    ← FastAPI app entry point
│   │   ├── config.py                  ← Dashboard settings (port, auth, DB path)
│   │   ├── auth.py                    ← Token-based auth (single user)
│   │   ├── db.py                      ← SQLite connection + migrations
│   │   ├── ingest.py                  ← Syncs JSON trade logs → SQLite on interval
│   │   ├── routes/
│   │   │   ├── performance.py         ← P&L, metrics, equity curves, drawdown
│   │   │   ├── trades.py              ← Historical trade browser, filters, export
│   │   │   ├── strategies.py          ← Active strategies, config, status
│   │   │   ├── live.py                ← Live positions, open orders, unrealized P&L
│   │   │   ├── backtests.py           ← Backtest result browser + comparison
│   │   │   ├── system.py              ← Bot health, pipeline status, change log
│   │   │   └── reports.py             ← Stored daily reports (Claude analysis text)
│   │   ├── ws.py                      ← WebSocket manager for real-time live data
│   │   └── queries.py                 ← SQL query library for all dashboard data
│   │
│   ├── frontend/
│   │   ├── package.json
│   │   ├── src/
│   │   │   ├── App.jsx                ← Router, layout, auth wrapper
│   │   │   ├── pages/
│   │   │   │   ├── Dashboard.jsx      ← Home: headline metrics, mini charts, alerts
│   │   │   │   ├── Performance.jsx    ← Deep P&L analysis, equity curves, heatmaps
│   │   │   │   ├── Trades.jsx         ← Filterable trade table, trade detail modal
│   │   │   │   ├── Strategies.jsx     ← Strategy cards, live config, toggle on/off
│   │   │   │   ├── LiveView.jsx       ← Real-time positions, open orders (WebSocket)
│   │   │   │   ├── Backtests.jsx      ← Backtest results, strategy comparisons
│   │   │   │   ├── System.jsx         ← Health, uptime, pipeline runs, change log
│   │   │   │   └── Reports.jsx        ← Browse daily Claude analysis reports
│   │   │   ├── components/            ← Reusable charts, tables, cards
│   │   │   └── hooks/                 ← useWebSocket, useAuth, useFetch
│   │   └── build/                     ← Production build (served by FastAPI)
│   │
│   ├── deploy/
│   │   ├── dashboard.plist            ← launchd service definition
│   │   └── setup.md                   ← Setup instructions
│   │
│   └── tests/
│       ├── test_ingest.py
│       ├── test_queries.py
│       └── test_routes.py
│
├── review/                            ← Daily review pipelines (runs on Mac)
│   ├── pipeline.py                    ← Main orchestration — accepts --domain flag
│   ├── data_puller.py                 ← Domain-aware: SSH for equities, local for crypto
│   ├── health_checker.py              ← Validate data completeness (per domain)
│   ├── analyst.py                     ← Runs Claude Code headless for analysis session
│   ├── reporter.py                    ← Format and send Telegram messages
│   ├── approval_listener.py           ← Telegram bot, waits for your response
│   ├── config_executor.py             ← Applies config.yaml changes (knows which config)
│   ├── code_builder.py                ← Runs Claude Code headless for code changes
│   ├── deploy_gate.py                 ← Gate 2: sends test results, waits for deploy approval
│   └── prompts/                       ← Prompt files loaded into Claude Code sessions
│       ├── equities/                  ← Equities-specific analysis prompts
│       │   ├── trade_analyst.md
│       │   ├── strategy_advisor.md
│       │   ├── pattern_scanner.md
│       │   └── build_spec_writer.md
│       └── crypto/                    ← Crypto-specific analysis prompts
│           ├── trade_analyst.md       ← Includes funding rates, liquidation analysis
│           ├── strategy_advisor.md    ← Crypto market structure awareness
│           ├── pattern_scanner.md     ← On-chain signals, exchange flows, regime shifts
│           └── build_spec_writer.md
│
├── state/                             ← Persistent state (lives on Mac, NOT in git)
│   ├── dashboard.db                   ← SQLite: all trades, backtests, reports, metrics
│   ├── equities/
│   │   ├── context.json               ← Rolling context: last 10 days of summaries
│   │   ├── change_log.json            ← Every change made, when, why, result
│   │   ├── strategies.json            ← Active strategies and their parameters
│   │   ├── performance_baseline.json  ← What "normal" looks like for each strategy
│   │   └── build_specs/
│   │
│   └── crypto/
│       ├── context.json               ← Rolling context: last 10 days of summaries
│       ├── change_log.json
│       ├── strategies.json
│       ├── performance_baseline.json
│       └── build_specs/
│
├── .claude/
│   ├── settings.json                  ← Model preferences, permissions
│   ├── agents/
│   │   ├── backend-engineer.md
│   │   ├── test-engineer.md
│   │   └── code-reviewer.md
│   └── commands/
│       ├── build-strategy.md          ← /build-strategy slash command
│       └── code-review.md             ← /code-review slash command
│
├── CLAUDE.md                          ← Project context for Claude Code (updated below)
├── requirements.txt                   ← Python deps for equities bot
├── requirements-crypto.txt            ← Python deps for crypto bot (ccxt, etc.)
├── requirements-review.txt            ← Python deps for review pipeline
├── requirements-dashboard.txt         ← Python deps for dashboard (fastapi, uvicorn, etc.)
└── README.md
```

---

## Crypto Bot Architecture Details

### Exchange Connectivity

The crypto bot uses [ccxt](https://github.com/ccxt/ccxt) as a unified exchange abstraction. This means the bot code doesn't care if it's trading on Binance, Coinbase, or Kraken — the connector handles the differences.

```python
# crypto/bot/execution/exchange_connector.py (simplified)

import ccxt

class ExchangeConnector:
    """
    Unified exchange interface using ccxt.
    Supports both REST and websocket for market data.
    """
    
    def __init__(self, exchange_id: str, config: dict):
        # ccxt creates the right exchange object based on ID
        self.exchange = getattr(ccxt, exchange_id)({
            'apiKey': config['api_key'],
            'secret': config['api_secret'],
            'sandbox': config.get('sandbox', True),  # Paper trading by default
            'options': {'defaultType': 'spot'}  # or 'future' for perps
        })
    
    def place_order(self, symbol: str, side: str, amount: float, 
                    order_type: str = 'limit', price: float = None) -> dict:
        """Place an order. Returns order dict with id, status, filled, etc."""
        return self.exchange.create_order(symbol, order_type, side, amount, price)
    
    def get_balance(self) -> dict:
        """Returns {asset: {free, used, total}} for all assets."""
        return self.exchange.fetch_balance()
    
    def get_ohlcv(self, symbol: str, timeframe: str = '5m', limit: int = 100) -> list:
        """Fetch OHLCV candles for strategy calculations."""
        return self.exchange.fetch_ohlcv(symbol, timeframe, limit=limit)
    
    def get_ticker(self, symbol: str) -> dict:
        """Current price, bid, ask, volume."""
        return self.exchange.fetch_ticker(symbol)
    
    def get_open_orders(self, symbol: str = None) -> list:
        return self.exchange.fetch_open_orders(symbol)
    
    def cancel_order(self, order_id: str, symbol: str) -> dict:
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
    """
    
    @abstractmethod
    def setup(self, connector, config: dict):
        """Initialize indicators, state, subscribe to pairs."""
        pass
    
    @abstractmethod
    def on_candle(self, symbol: str, candle: dict):
        """
        Called on each new candle close.
        candle: {timestamp, open, high, low, close, volume}
        """
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
        """
        Override to implement pause conditions.
        E.g., pause during extreme volatility, low liquidity hours, etc.
        Default: never pause.
        """
        return False
    
    @abstractmethod
    def teardown(self):
        """Cleanup: cancel open orders, log final state."""
        pass
```

### Crypto Config Structure

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

global_risk:
  max_portfolio_drawdown_pct: 5.0    # Kill switch: stop all if portfolio -5%
  max_position_pct: 20.0             # No single position > 20% of portfolio
  max_open_positions: 5
  daily_loss_limit_usd: 500          # Stop trading for the day if hit
  portfolio_base_currency: USDT

strategies:
  grid_trading:
    enabled: true
    pairs: [BTC/USDT, ETH/USDT]
    grid_levels: 10
    grid_spacing_pct: 0.5
    position_size_usd: 100           # Per grid level
    take_profit_pct: 1.5
    stop_loss_pct: 3.0
    
  trend_follow:
    enabled: false
    pairs: [SOL/USDT]
    timeframe: 1h
    ema_fast: 12
    ema_slow: 26
    position_size_usd: 200
    trailing_stop_pct: 2.0

candle_interval: 5m                  # Main loop interval for on_candle
health_heartbeat_seconds: 60
```

### Crypto vs. Equities: Key Architectural Differences

| Aspect | Equities | Crypto |
|--------|----------|--------|
| Broker/Exchange | Das Trader (websocket, localhost) | ccxt (REST + WS, remote APIs) |
| Market hours | 9:30–16:00 ET, M–F | 24/7/365 |
| Bot runs on | Windows desktop | Mac M1 Pro |
| Bot lifecycle | Starts morning, idles at close | Runs as a persistent service (launchd) |
| Log retrieval | SSH/SCP from Mac → Windows | Local filesystem read |
| Position types | Long/short shares | Spot, limit, stop-limit (futures later) |
| Risk factors | Market hours, halts, gaps | Exchange downtime, API rate limits, funding rates, liquidation |
| Price data | Das Trader data feed | Exchange websocket + REST OHLCV |
| Strategy loop | on_bar / on_tick per symbol | on_candle / on_tick per pair |
| Backtester data | Historical trade logs | Historical OHLCV from exchange API |
| Pipeline schedule | 16:30 ET weekdays | 00:00 UTC daily (configurable) |
| Config path | equities/bot/config.yaml | crypto/bot/config.yaml |

### Crypto Bot as a Service (Mac)

The crypto bot runs 24/7 as a launchd service on the Mac:

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
# Run via launchd or cron every 60 seconds

cd ~/trading-system
CURRENT=$(git rev-parse HEAD)
git fetch origin main --quiet
LATEST=$(git rev-parse origin/main)

if [ "$CURRENT" != "$LATEST" ]; then
    echo "$(date): New commits detected, pulling and restarting crypto bot..."
    git pull origin main --quiet
    
    # Restart via launchctl
    launchctl stop com.trading.crypto-bot
    sleep 2
    launchctl start com.trading.crypto-bot
    
    echo "$(date): Crypto bot restarted with latest code."
fi
```

---

## The Two-Gate Pipeline (Updated for Multi-Domain)

### Pipeline Invocation

The pipeline now accepts a `--domain` flag to run for a specific domain:

```bash
# Equities review (after market close)
python -m review.pipeline --domain equities

# Crypto review (daily at midnight UTC)
python -m review.pipeline --domain crypto

# Both (manual or weekly summary)
python -m review.pipeline --domain all
```

### Domain-Aware Data Pulling

```python
# review/data_puller.py

def pull_trade_data(domain: str, date: str) -> dict:
    """
    Pull trade data based on domain.
    Equities: SSH into Windows, SCP the file.
    Crypto: Read local file on same machine.
    """
    if domain == "equities":
        # SSH into Windows, grab trade logs
        log_path = f"C:/trading-system/equities/logs/trades/{date}.json"
        result = subprocess.run(
            ["scp", f"{WIN_USER}@{WIN_IP}:{log_path}", f"./equities_trades_{date}.json"],
            timeout=30, capture_output=True
        )
        if result.returncode != 0:
            raise ConnectionError(f"Cannot reach Windows: {result.stderr.decode()}")
        return json.load(open(f"./equities_trades_{date}.json"))
    
    elif domain == "crypto":
        # Local file — crypto bot runs on this same Mac
        log_path = Path.home() / "trading-system" / "crypto" / "logs" / "trades" / f"{date}.json"
        if not log_path.exists():
            raise FileNotFoundError(f"No crypto trade log for {date}")
        return json.load(open(log_path))
```

### Full Flow (Same Structure, Domain-Aware)

```
CRON FIRES pipeline.py --domain <equities|crypto>
─────────────────────────────────────────────────

  ┌──────────────────────────────────────────────────────────────┐
  │ PHASE 1: DATA GATHERING (Python — no LLM)                    │
  │                                                              │
  │  equities: SSH into Windows, pull trade logs + health logs   │
  │  crypto:   Read local logs from ~/trading-system/crypto/logs │
  │                                                              │
  │  Both: Health check, load state/<domain>/context.json        │
  │        Stage data into analysis_input.json for Claude Code   │
  └──────────────┬───────────────────────────────────────────────┘
                 │
  ┌──────────────▼───────────────────────────────────────────────┐
  │ PHASE 2: ANALYSIS (Claude Code headless — full repo access)  │
  │                                                              │
  │  Single Claude Code session with domain-specific prompt.     │
  │  Can read strategy source code, run Python calculations,     │
  │  inspect indicators, check config — not just trade data.     │
  │                                                              │
  │  4. Trade Analyst — today's performance vs 10-day baseline  │
  │  5. Strategy Advisor — reads strategy code, suggests        │
  │     config AND/OR code changes with full code awareness     │
  │  6. Pattern Scanner — spot opportunities, regime shifts,    │
  │     cross-reference with what strategies already implement  │
  │  7. Backtest — can run quick Python validation of ideas     │
  │  8. Build Spec Draft — if code changes suggested, produces  │
  │     detailed spec informed by having already read the code  │
  │                                                              │
  │  Outputs structured JSON: analysis_output.json              │
  └──────────────┬───────────────────────────────────────────────┘
                 │
  ┌──────────────▼───────────────────────────────────────────────┐
  │ ═══════════════ GATE 1: CONCEPT APPROVAL ═══════════════════ │
  │                                                              │
  │  9. Send Telegram report (tagged with domain emoji):        │
  │     📈 Equities: P&L, metrics, suggestions                  │
  │     ₿ Crypto: P&L, metrics, suggestions                     │
  │     Each suggestion labeled [CONFIG] or [CODE]              │
  │                                                              │
  │  10. Wait for your Telegram reply                           │
  │     → "approve all"                                         │
  │     → "approve 1 and 3, skip 2"                            │
  │     → "approve but make grid spacing 0.3%"                 │
  │     → "reject"                                              │
  │                                                              │
  │  11. Parse your response (simple text parsing or light API) │
  └──────────────┬───────────────────────────────────────────────┘
                 │
          ┌──────┴──────┐
          │             │
    ┌─────▼─────┐ ┌────▼──────────────────────────────────────┐
    │  CONFIG   │ │  CODE CHANGES                              │
    │  CHANGES  │ │                                            │
    │           │ │  12. Refine build spec from analysis       │
    │  Direct   │ │      output (already has draft from        │
    │  Python   │ │      Phase 2 where it read the code)      │
    │  edits    │ │  13. Create staging/<domain>/YYYY-MM-DD    │
    │  to the   │ │  14. Run Claude Code headless (new session │
    │  correct  │ │      for code gen — uses build spec)       │
    │  config   │ │  15. Automated validation (domain tests)   │
    │  .yaml    │ │                                            │
    │           │ │  IF TESTS PASS:                            │
    │  Commit   │ │  ┌──────────────────────────────────────┐  │
    │  to main  │ │  │ ══════ GATE 2: DEPLOY APPROVAL ══════│  │
    │  Push     │ │  │                                      │  │
    │           │ │  │ Telegram with test + backtest results│  │
    │  ✅ Done  │ │  │ Reply DEPLOY or REJECT               │  │
    │           │ │  └──────────────────────────────────────┘  │
    └───────────┘ │                                            │
                  │  DEPLOY → merge staging → main → push     │
                  │  REJECT → delete branch, log reason       │
                  └────────────────────────────────────────────┘

  17. Update state/<domain>/context.json and change_log.json
  
  POST-DEPLOY:
    equities: Windows git watcher pulls + restarts equities bot
    crypto:   Mac git watcher pulls + restarts crypto bot service
```

### Why Claude Code Headless for Analysis (Not Just Code Gen)

Previous versions of this blueprint used the Claude API for analysis and reserved Claude Code headless only for code generation. That was wrong. Here's why.

**The analysis phase needs code awareness.** When the strategy advisor suggests "add a trailing stop to the VWAP momentum strategy," the quality of that suggestion depends entirely on whether it can see the current implementation. With API-only analysis, the advisor is working blind — it might suggest something that already exists, conflicts with existing logic, or doesn't fit the code structure. With Claude Code headless, the advisor can:

- **Read strategy source code** to understand what's already implemented
- **Inspect indicators.py** to know what calculations are available vs. need building
- **Check config.yaml** to see current parameter values and structure
- **Read risk_manager.py** to understand safety constraints before suggesting changes
- **Run quick Python calculations** on trade data (pandas aggregations, statistical tests) instead of guessing at patterns
- **Produce better build specs** because it already saw the code it's proposing to change

**This is one Claude Code session, not four separate API calls.** The analysis prompt tells Claude Code to work through each role sequentially (trade analyst → strategy advisor → pattern scanner → backtest validation). This means each role builds on the prior one's findings — the pattern scanner knows what the trade analyst already identified, the advisor's suggestions are grounded in patterns that were already confirmed. And the whole session shares one codebase read rather than stuffing source code into four separate API call contexts.

**The cost tradeoff is worth it.** Yes, a Claude Code headless session is slower and more expensive than four API calls. But the analysis it produces is fundamentally better, and analysis quality directly determines the quality of every suggestion that reaches Gate 1. Bad analysis → bad suggestions → you reject everything → the system provides no value. Good analysis → good suggestions → you approve and the system improves itself.

### Analysis Session: How It Works

```python
# review/analyst.py

def run_analysis(domain: str, trade_data: dict, context: dict) -> dict:
    """
    Runs a single Claude Code headless session for the full analysis phase.
    Claude Code has access to the entire repo — it can read strategy code,
    run calculations, inspect configs, and produce informed suggestions.
    """
    
    # Stage the input data where Claude Code can find it
    analysis_input = {
        "domain": domain,
        "date": trade_data["date"],
        "trades": trade_data["trades"],
        "health": trade_data["health"],
        "context": context,  # rolling 10-day window
    }
    input_path = f"state/{domain}/analysis_input.json"
    Path(input_path).write_text(json.dumps(analysis_input, indent=2))
    
    # Load the domain-specific analysis prompt
    prompt = build_analysis_prompt(domain, input_path)
    
    # Run Claude Code headless — it has full repo access
    result = subprocess.run(
        [
            "claude", "-p", prompt,
            "--dangerously-skip-permissions",
            "--output-format", "json",
            "--model", "sonnet"
        ],
        cwd=Path.home() / "trading-system",
        capture_output=True,
        timeout=300  # 5 min max for analysis
    )
    
    # Claude Code writes its output to a structured file
    output_path = f"state/{domain}/analysis_output.json"
    if Path(output_path).exists():
        return json.load(open(output_path))
    
    # Fallback: parse from stdout if file wasn't written
    return parse_analysis_output(result.stdout.decode())


def build_analysis_prompt(domain: str, input_path: str) -> str:
    """
    Constructs the analysis prompt by combining the domain-specific
    role prompts into a single session prompt.
    """
    roles = ["trade_analyst", "strategy_advisor", "pattern_scanner"]
    role_prompts = []
    for role in roles:
        prompt_path = f"review/prompts/{domain}/{role}.md"
        role_prompts.append(Path(prompt_path).read_text())
    
    return f"""You are the daily review analyst for the {domain} trading system.
    
Read the analysis input data at {input_path}, then work through each role below
in sequence. For each role, you have full access to the codebase — read strategy
source code, check indicators, inspect configs, run Python calculations as needed.

After completing all roles, write your full analysis output as structured JSON
to state/{domain}/analysis_output.json with this schema:

{{
  "summary": "headline performance summary",
  "metrics": {{ "pnl": ..., "trades": ..., "win_rate": ..., ... }},
  "analysis": {{
    "trade_analysis": "detailed performance breakdown",
    "strategy_review": "code-informed strategy assessment",
    "pattern_scan": "patterns and opportunities identified"
  }},
  "suggestions": [
    {{
      "id": 1,
      "type": "CONFIG or CODE",
      "title": "short description",
      "rationale": "why, informed by code inspection",
      "details": "specific change proposed",
      "backtest_estimate": "expected impact if calculable",
      "build_spec_draft": "if type=CODE, initial spec based on code read"
    }}
  ]
}}

═══ ROLE 1: TRADE ANALYST ═══
{role_prompts[0]}

═══ ROLE 2: STRATEGY ADVISOR ═══
{role_prompts[1]}

═══ ROLE 3: PATTERN SCANNER ═══
{role_prompts[2]}

Remember: you can and should read the actual strategy source code in
{domain}/bot/strategies/ before making suggestions. Inspect what indicators
exist, what the current config values are, and how the risk manager works.
Suggestions grounded in the actual code are far more valuable than generic ones.
"""
```

### Two Separate Claude Code Sessions (Analysis vs. Code Gen)

The pipeline uses Claude Code headless twice, but in separate sessions with different purposes:

```
SESSION 1: Analysis (Phase 2)
├── Purpose: Understand what happened, why, and what to change
├── Access: Read-only in practice (reads code, data, config)
├── Output: analysis_output.json with suggestions + draft build specs
├── Duration: ~2-3 minutes
└── Triggered: Every pipeline run

SESSION 2: Code Generation (after Gate 1 approval)
├── Purpose: Implement the approved code changes
├── Access: Read-write (creates files, modifies code, runs tests)
├── Output: Working code on staging branch, test results
├── Duration: ~5-10 minutes
├── Triggered: Only when CODE suggestions are approved
└── Input: Refined build spec (based on analysis session's draft)
```

The analysis session never writes code to the repo. It reads code to produce better analysis and suggestions. The code generation session gets a much better build spec because the analysis session already inspected the codebase and produced a draft spec grounded in reality.

### Staging Branch Convention (Updated)

```
staging/equities/2026-03-07         ← equities code change
staging/crypto/2026-03-07           ← crypto code change
staging/crypto/2026-03-07-grid      ← second crypto change same day (disambiguated)
staging/shared/2026-03-07           ← changes to shared/ module
```

---

## Crypto-Specific Considerations

### 24/7 Markets: Review Scheduling

Unlike equities with a clear "market close" trigger, crypto runs continuously. Options for review cadence:

```
# Default: once daily at midnight UTC
0 0 * * *  python -m review.pipeline --domain crypto

# Alternative: every 8 hours for more active management
0 0,8,16 * * *  python -m review.pipeline --domain crypto

# Weekend: equities off, crypto gets extra analysis
0 10 * * 6  python -m review.pipeline --domain crypto --weekly
```

The "daily" concept for crypto uses UTC day boundaries. Trade logs are still split by UTC date. The rolling 10-day context window works the same way.

### Crypto Risk Management Differences

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
        """
        For futures: check funding rate before opening positions.
        High funding = expensive to hold position.
        """
        pass
    
    def check_spread(self, symbol: str, max_spread_pct: float) -> bool:
        """
        Verify bid-ask spread is reasonable before trading.
        Low-liquidity pairs can have dangerous spreads.
        """
        pass
    
    def check_portfolio_exposure(self) -> dict:
        """
        Crypto-specific: check correlation exposure.
        BTC + ETH + SOL might all dump together.
        """
        pass
    
    def emergency_close_all(self):
        """
        Kill switch: cancel all open orders and market-sell all positions.
        Triggered by max drawdown or manual Telegram command.
        """
        pass
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

### Crypto Backtester

Unlike equities (where we backtest against our own trade logs), crypto backtesting can pull historical data directly from exchanges:

```python
# crypto/tests/backtester.py

def fetch_historical_data(symbol: str, timeframe: str, days: int) -> pd.DataFrame:
    """
    Pull historical OHLCV from exchange for backtesting.
    ccxt makes this easy — no need for separate data provider.
    """
    exchange = ccxt.binance({'enableRateLimit': True})
    since = exchange.parse8601((datetime.utcnow() - timedelta(days=days)).isoformat())
    ohlcv = exchange.fetch_ohlcv(symbol, timeframe, since=since, limit=1000)
    return pd.DataFrame(ohlcv, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])
```

---

## Dashboard Architecture

### Why a Dashboard

Telegram is great for approvals and alerts, but it's terrible for browsing data. You can't scroll through 200 historical trades in a Telegram message, compare two backtest runs side by side, or glance at an equity curve while on your phone. The dashboard is a read-heavy companion to Telegram's write-heavy approval flow.

**Telegram:** action-oriented — approve, reject, kill switch, alerts
**Dashboard:** analysis-oriented — how's it doing, what happened, should I worry

### Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Backend | FastAPI (Python) | Fits existing Python stack, async, fast, great WebSocket support |
| Database | SQLite | Zero config, single file, handles this read volume easily, lives in state/ |
| Frontend | React (Vite build) | Served as static files by FastAPI, no separate server needed |
| Charts | Recharts / Lightweight Charts | TradingView-style price charts, equity curves, heatmaps |
| Real-time | WebSocket (FastAPI native) | Live positions and P&L without polling |
| Auth | Bearer token (single user) | Simple, secure enough for local network / Tailscale |
| Process | launchd service on Mac | Runs alongside crypto bot, auto-restarts |

### How Data Flows Into the Dashboard

The bots and pipelines write JSON log files (same as before). The dashboard doesn't change how any existing component works — it's purely additive. A background ingestion process syncs those JSON files into SQLite so the dashboard can query them efficiently.

```
┌──────────────┐     JSON logs     ┌──────────────┐     SQL queries    ┌──────────────┐
│  Equities    │ ──── (pulled ────►│              │ ◄────────────────── │              │
│  Bot         │      via SSH)     │   SQLite     │                    │   FastAPI    │
│  (Windows)   │                   │   dashboard  │ ──── JSON ────────►│   Backend    │
│              │                   │   .db        │                    │      │       │
│  Crypto      │ ──── (local) ───►│              │                    │      │ WS    │
│  Bot (Mac)   │                   │              │ ◄─── writes ────── │      │       │
│              │                   │              │     (pipeline      │      ▼       │
│  Pipeline    │ ──── (direct) ──►│              │      results,      │   React     │
│  (Mac)       │                   │              │      reports)      │   Frontend   │
└──────────────┘                   └──────────────┘                    └──────────────┘

Ingestion: cron job every 5 min reads new JSON log entries → INSERT into SQLite
Pipeline: writes analysis reports + backtest results directly to SQLite after each run
Live data: FastAPI queries bot state files + exchange APIs → pushes via WebSocket
```

### Data Ingestion

```python
# dashboard/backend/ingest.py

"""
Syncs JSON trade logs into SQLite.
Runs on a 5-minute cron or as a background task inside FastAPI.
Idempotent — safe to re-run (uses trade IDs to deduplicate).
"""

def ingest_trades(domain: str, db: sqlite3.Connection):
    """
    Read JSON trade logs, insert any new trades into SQLite.
    Each trade has a unique ID (timestamp + symbol + strategy + sequence).
    """
    log_dir = TRADE_LOG_PATHS[domain]  # equities/logs/trades/ or crypto/logs/trades/
    
    last_ingested = db.execute(
        "SELECT MAX(timestamp) FROM trades WHERE domain = ?", (domain,)
    ).fetchone()[0]
    
    for log_file in sorted(log_dir.glob("*.json")):
        trades = json.load(open(log_file))
        for trade in trades:
            if trade["timestamp"] > (last_ingested or ""):
                db.execute("""
                    INSERT OR IGNORE INTO trades 
                    (id, domain, timestamp, symbol, side, qty, price, 
                     strategy, pnl, fees, metadata)
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                """, (
                    trade["id"], domain, trade["timestamp"], trade["symbol"],
                    trade["side"], trade["qty"], trade["price"],
                    trade["strategy"], trade["pnl"], trade.get("fees", 0),
                    json.dumps(trade.get("metadata", {}))
                ))
    db.commit()


def ingest_health(domain: str, db: sqlite3.Connection):
    """Sync health heartbeat logs for uptime tracking."""
    pass


def ingest_backtest_result(result: dict, db: sqlite3.Connection):
    """
    Called directly by the pipeline after running a backtest.
    Stores full backtest results for dashboard browsing.
    """
    db.execute("""
        INSERT INTO backtests
        (id, domain, timestamp, strategy_name, config_snapshot,
         days_tested, total_pnl, sharpe, max_drawdown, win_rate,
         trade_count, equity_curve, notes)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    """, (
        result["id"], result["domain"], result["timestamp"],
        result["strategy"], json.dumps(result["config"]),
        result["days"], result["pnl"], result["sharpe"],
        result["max_drawdown"], result["win_rate"],
        result["trade_count"], json.dumps(result["equity_curve"]),
        result.get("notes", "")
    ))
    db.commit()
```

### SQLite Schema

```sql
-- state/dashboard.db

-- ═══════════════════════════════════════════════
-- TRADES
-- ═══════════════════════════════════════════════

CREATE TABLE trades (
    id              TEXT PRIMARY KEY,
    domain          TEXT NOT NULL,          -- 'equities' or 'crypto'
    timestamp       TEXT NOT NULL,          -- ISO 8601
    symbol          TEXT NOT NULL,          -- 'NVDA', 'BTC/USDT'
    side            TEXT NOT NULL,          -- 'buy' or 'sell'
    qty             REAL NOT NULL,
    price           REAL NOT NULL,
    strategy        TEXT NOT NULL,          -- 'vwap_momentum', 'grid_trading'
    pnl             REAL,                   -- realized P&L (NULL for open leg)
    fees            REAL DEFAULT 0,
    metadata        TEXT,                   -- JSON blob for domain-specific fields
    created_at      TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_trades_domain_ts ON trades(domain, timestamp);
CREATE INDEX idx_trades_strategy ON trades(strategy, timestamp);
CREATE INDEX idx_trades_symbol ON trades(symbol, timestamp);

-- ═══════════════════════════════════════════════
-- DAILY SUMMARIES (pre-aggregated for fast dashboard loads)
-- ═══════════════════════════════════════════════

CREATE TABLE daily_summaries (
    date            TEXT NOT NULL,
    domain          TEXT NOT NULL,
    total_pnl       REAL,
    trade_count     INTEGER,
    win_count       INTEGER,
    loss_count      INTEGER,
    win_rate        REAL,
    avg_win         REAL,
    avg_loss        REAL,
    max_win         REAL,
    max_loss        REAL,
    sharpe          REAL,
    max_drawdown    REAL,
    strategies_active TEXT,                 -- JSON array of strategy names
    PRIMARY KEY (date, domain)
);

-- ═══════════════════════════════════════════════
-- STRATEGIES (current config + status)
-- ═══════════════════════════════════════════════

CREATE TABLE strategies (
    id              TEXT PRIMARY KEY,       -- 'equities:vwap_momentum'
    domain          TEXT NOT NULL,
    name            TEXT NOT NULL,
    enabled         INTEGER NOT NULL,       -- 0 or 1
    config_snapshot TEXT NOT NULL,          -- JSON of current params
    deployed_at     TEXT,                   -- when this version went live
    last_trade_at   TEXT,                   -- most recent trade
    total_pnl       REAL DEFAULT 0,
    trade_count     INTEGER DEFAULT 0,
    updated_at      TEXT DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════
-- BACKTESTS
-- ═══════════════════════════════════════════════

CREATE TABLE backtests (
    id              TEXT PRIMARY KEY,
    domain          TEXT NOT NULL,
    timestamp       TEXT NOT NULL,
    strategy_name   TEXT NOT NULL,
    config_snapshot TEXT NOT NULL,          -- JSON: params used for this backtest
    days_tested     INTEGER,
    total_pnl       REAL,
    sharpe          REAL,
    max_drawdown    REAL,
    win_rate        REAL,
    trade_count     INTEGER,
    equity_curve    TEXT,                   -- JSON array of {date, equity} points
    notes           TEXT,                   -- why this was run, what it tested
    approved        INTEGER,               -- NULL=pending, 1=deployed, 0=rejected
    created_at      TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_backtests_strategy ON backtests(strategy_name, timestamp);

-- ═══════════════════════════════════════════════
-- PIPELINE REPORTS (Claude analysis text)
-- ═══════════════════════════════════════════════

CREATE TABLE reports (
    id              TEXT PRIMARY KEY,
    domain          TEXT NOT NULL,
    date            TEXT NOT NULL,
    report_type     TEXT NOT NULL,          -- 'daily', 'weekly', 'health'
    analysis_text   TEXT NOT NULL,          -- full Claude analysis output
    suggestions     TEXT,                   -- JSON array of suggestions made
    approved        TEXT,                   -- JSON: which suggestions were approved
    created_at      TEXT DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════
-- CHANGE LOG
-- ═══════════════════════════════════════════════

CREATE TABLE change_log (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    domain          TEXT NOT NULL,
    timestamp       TEXT NOT NULL,
    change_type     TEXT NOT NULL,          -- 'config', 'code', 'manual'
    description     TEXT NOT NULL,
    before_value    TEXT,                   -- JSON: what it was
    after_value     TEXT,                   -- JSON: what it became
    impact_notes    TEXT,                   -- filled in after observation period
    created_at      TEXT DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════
-- HEALTH SNAPSHOTS
-- ═══════════════════════════════════════════════

CREATE TABLE health_snapshots (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    domain          TEXT NOT NULL,
    timestamp       TEXT NOT NULL,
    bot_status      TEXT NOT NULL,          -- 'running', 'stopped', 'error'
    connection_ok   INTEGER,               -- 1=connected to broker/exchange
    last_heartbeat  TEXT,
    uptime_pct      REAL,                  -- rolling 24h uptime
    error_message   TEXT,
    metadata        TEXT                   -- JSON: exchange-specific health info
);

CREATE INDEX idx_health_domain_ts ON health_snapshots(domain, timestamp);
```

### Dashboard Pages

#### 1. Home Dashboard (`/`)

The landing page — glanceable health of the entire system in one screen.

**Headline cards (top row):**
- Today's combined P&L (equities + crypto, separate colors)
- Open positions count (with unrealized P&L)
- Win rate (rolling 10 days)
- System status (green/yellow/red per domain)

**Equity curve (main chart):**
- Cumulative P&L over time, line per domain
- Toggle: 7d / 30d / 90d / all-time
- Drawdown overlay (shaded area below zero line)

**Recent activity (bottom):**
- Last 10 trades (both domains, color-coded)
- Last pipeline run summary
- Any pending Gate 1/2 approvals
- Recent change log entries

#### 2. Performance (`/performance`)

Deep dive into how the system is making (or losing) money.

**Metrics panel:**
- P&L: daily, weekly, monthly, all-time (filterable by domain)
- Sharpe ratio (rolling windows: 10d, 30d, 90d)
- Sortino ratio (downside risk focus)
- Max drawdown (current + historical worst)
- Win rate, profit factor, average R-multiple
- Expectancy per trade

**Visualizations:**
- Equity curve with benchmark overlay (SPY for equities, BTC for crypto)
- Daily P&L bar chart (green/red bars)
- P&L heatmap by day-of-week × hour-of-day (when do we make money?)
- Monthly returns table (like a hedge fund tearsheet)
- Drawdown chart (underwater plot)
- Rolling Sharpe chart

**Breakdowns:**
- P&L by strategy (stacked bar or pie)
- P&L by symbol (which tickers/pairs are profitable?)
- P&L by time of day (AM vs PM for equities, hour buckets for crypto)
- Win rate by strategy, by symbol, by day-of-week

#### 3. Trade History (`/trades`)

Browse, filter, search, and analyze every historical trade.

**Filterable table:**
- Columns: date, time, domain, symbol, strategy, side, qty, price, P&L, fees
- Filters: domain, strategy, symbol, date range, side (buy/sell), P&L range
- Sort by any column
- Search by symbol
- Export to CSV

**Trade detail modal (click any row):**
- Full trade details including metadata
- Entry and exit prices
- Holding duration
- What strategy/config produced this trade
- What the Claude analysis said about this trade (if available)

**Aggregate stats for current filter:**
- Total P&L of filtered trades
- Win rate, avg win, avg loss
- Lets you ask: "how did mean_reversion do on SPY last week?"

#### 4. Strategies (`/strategies`)

What's currently live and how each strategy is performing.

**Strategy cards (one per active strategy):**
- Name, domain, enabled/disabled status
- Current config params (from config.yaml)
- Performance since deployment: P&L, trade count, win rate, Sharpe
- Last trade timestamp
- Deployed date
- Config change history (what was tweaked, when, impact)

**Strategy comparison table:**
- Side-by-side metrics for all strategies
- Sortable: best Sharpe, best P&L, highest win rate
- Quick visual: which strategies are carrying the portfolio

**Config viewer:**
- Current config.yaml rendered in a readable format
- Diff from previous version (what changed last)

#### 5. Live View (`/live`)

Real-time view of what's happening right now. Uses WebSocket for sub-second updates.

**Open positions (real-time):**
- Symbol, side, qty, entry price, current price, unrealized P&L
- Color-coded: green = winning, red = losing
- Per domain: equities positions + crypto positions

**Open orders:**
- Pending limit orders, stop orders
- Strategy that placed them
- Cancel button (sends kill command — stretch goal)

**Live P&L ticker:**
- Today's realized + unrealized P&L, updating in real-time
- Sparkline of intraday P&L

**Bot status indicators:**
- Equities bot: connected / disconnected / market closed
- Crypto bot: connected / API healthy / rate limit status
- Last heartbeat timestamp for each

#### 6. Backtests (`/backtests`)

Review all backtest results — both from the pipeline and manual runs.

**Backtest list (sortable/filterable):**
- Date run, strategy, domain, days tested
- Key metrics: P&L, Sharpe, max drawdown, win rate
- Status: deployed / rejected / pending
- Who requested it: pipeline or manual

**Backtest detail view (click any row):**
- Full equity curve chart
- Trade-by-trade breakdown
- Config snapshot used
- Comparison vs. current live performance of same strategy
- Notes from Claude analysis about why this was suggested

**Compare mode:**
- Select 2+ backtests → overlay equity curves
- Side-by-side metrics table
- Useful for: "the pipeline suggested 3 different grid spacings — which backtest looked best?"

#### 7. System Health (`/system`)

Infrastructure monitoring and operational history.

**Bot health (per domain):**
- Current status: running / stopped / error
- Uptime chart: rolling 24h, 7d, 30d (like a status page)
- Last heartbeat, last trade, last error
- Connection history timeline

**Pipeline history:**
- List of all pipeline runs: domain, timestamp, duration, result
- What suggestions were made, which were approved
- Link to full analysis report

**Change log:**
- Every config and code change, chronological
- Before/after values
- Impact notes (filled in after observation period)
- 3-day rule status: "Next change allowed in 2 days"

**Resource usage (nice-to-have):**
- Mac CPU / memory usage
- API call counts (Claude, exchange)
- Rate limit headroom

#### 8. Reports (`/reports`)

Archive of every Claude analysis report, searchable and browsable.

**Report list:**
- Date, domain, type (daily/weekly/health)
- Quick preview of headline metrics
- Clickable to see full analysis text

**Report detail:**
- Full Claude analysis text (formatted markdown)
- Suggestions made and their approval status
- Trade data that was analyzed
- Useful for: "what did the system think about last Tuesday?"

### Backend API Structure

```python
# dashboard/backend/main.py

from fastapi import FastAPI, Depends, WebSocket
from fastapi.staticfiles import StaticFiles
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(title="Trading System Dashboard")

# Serve React build
app.mount("/static", StaticFiles(directory="dashboard/frontend/build/static"))

# ── REST routes ──────────────────────────────────────────────

# Performance
app.include_router(performance_router, prefix="/api/performance")
# GET /api/performance/summary?domain=all&period=30d
# GET /api/performance/equity-curve?domain=crypto&period=90d
# GET /api/performance/daily-pnl?domain=equities&start=2026-01-01&end=2026-03-07
# GET /api/performance/heatmap?domain=all
# GET /api/performance/monthly-returns?domain=all

# Trades
app.include_router(trades_router, prefix="/api/trades")
# GET /api/trades?domain=crypto&strategy=grid_trading&symbol=BTC/USDT&limit=50&offset=0
# GET /api/trades/{trade_id}
# GET /api/trades/stats?domain=equities&strategy=vwap_momentum&start=2026-03-01
# GET /api/trades/export?format=csv&domain=all&start=2026-01-01

# Strategies
app.include_router(strategies_router, prefix="/api/strategies")
# GET /api/strategies?domain=all
# GET /api/strategies/{strategy_id}
# GET /api/strategies/{strategy_id}/history  (config change timeline)
# GET /api/strategies/{strategy_id}/performance

# Live
app.include_router(live_router, prefix="/api/live")
# GET /api/live/positions?domain=all
# GET /api/live/orders?domain=crypto
# GET /api/live/status  (bot health summary)

# Backtests
app.include_router(backtests_router, prefix="/api/backtests")
# GET /api/backtests?domain=crypto&strategy=grid_trading
# GET /api/backtests/{backtest_id}
# GET /api/backtests/compare?ids=bt_001,bt_002,bt_003

# System
app.include_router(system_router, prefix="/api/system")
# GET /api/system/health
# GET /api/system/pipeline-runs?limit=20
# GET /api/system/change-log?domain=equities
# GET /api/system/uptime?domain=crypto&period=7d

# Reports
app.include_router(reports_router, prefix="/api/reports")
# GET /api/reports?domain=crypto&type=daily&limit=10
# GET /api/reports/{report_id}

# ── WebSocket ────────────────────────────────────────────────

@app.websocket("/ws/live")
async def live_feed(websocket: WebSocket):
    """
    Real-time feed for LiveView page.
    Pushes: positions, open orders, unrealized P&L, bot status.
    Updates every 1-5 seconds depending on data type.
    """
    await websocket.accept()
    # ... push loop reading from bot state + exchange APIs
```

### Live Data: How It Works

The live view is the trickiest part because it needs real-time data from two different sources.

**Crypto (running on same Mac):**
The dashboard reads directly from the crypto bot's shared state. The crypto bot maintains a state file (`crypto/bot/state/live_positions.json`) that it updates on every fill. The dashboard's WebSocket handler reads this file and pushes changes to the frontend.

```python
# In crypto bot's order_manager.py — writes state for dashboard
def update_live_state(self):
    state = {
        "positions": self.get_open_positions(),
        "orders": self.get_open_orders(),
        "unrealized_pnl": self.calc_unrealized_pnl(),
        "updated_at": datetime.utcnow().isoformat()
    }
    Path("crypto/bot/state/live_positions.json").write_text(json.dumps(state))
```

**Equities (running on Windows):**
More limited — the equities bot on Windows writes its live state to a file, and the dashboard polls it via SSH on a 10-second interval during market hours. Not truly real-time, but close enough. Alternatively, the equities bot could push state to a simple HTTP endpoint that the Mac polls.

```python
# In dashboard WebSocket handler
async def get_equities_live_state():
    """Poll Windows for live equities state. Only during market hours."""
    if not is_market_open():
        return {"positions": [], "status": "market_closed"}
    
    result = subprocess.run(
        ["ssh", f"{WIN_USER}@{WIN_IP}", "cat", 
         "C:/trading-system/equities/bot/state/live_positions.json"],
        capture_output=True, timeout=5
    )
    if result.returncode == 0:
        return json.loads(result.stdout)
    return {"positions": [], "status": "unreachable"}
```

### Access and Authentication

The dashboard runs on the Mac, accessible from any device on your local network. For remote access (checking from the gym, traveling), you have a few options.

**Local network (simplest):**
Dashboard binds to `0.0.0.0:8080`. Access from any device on the same WiFi at `http://<mac-ip>:8080`.

**Tailscale (recommended for remote access):**
Install Tailscale on the Mac. Dashboard becomes accessible at `http://<tailscale-hostname>:8080` from any device with Tailscale installed — your phone, your laptop, anywhere. No port forwarding, no public exposure.

**Auth:**
Single-user token auth. On first setup, you generate a token. Store it in the browser. All API calls require `Authorization: Bearer <token>`.

```python
# dashboard/backend/auth.py

import secrets
from fastapi import Security, HTTPException
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()
DASHBOARD_TOKEN = os.environ.get("DASHBOARD_TOKEN")  # set once during setup

async def verify_token(credentials: HTTPAuthorizationCredentials = Security(security)):
    if credentials.credentials != DASHBOARD_TOKEN:
        raise HTTPException(status_code=401, detail="Invalid token")
    return credentials.credentials
```

### Dashboard as a Service (Mac)

```xml
<!-- dashboard/deploy/dashboard.plist -->

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.trading.dashboard</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/python3</string>
        <string>-m</string>
        <string>uvicorn</string>
        <string>dashboard.backend.main:app</string>
        <string>--host</string>
        <string>0.0.0.0</string>
        <string>--port</string>
        <string>8080</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/you/trading-system</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>EnvironmentVariables</key>
    <dict>
        <key>DASHBOARD_TOKEN</key>
        <string>SET_DURING_SETUP</string>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/you/trading-system/dashboard/logs/stdout.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/you/trading-system/dashboard/logs/stderr.log</string>
</dict>
</plist>
```

### Pipeline Integration

The review pipeline writes to the dashboard DB at key points, so the dashboard always has fresh data without scraping log files:

```python
# Additions to review/pipeline.py

from dashboard.backend.db import get_db
from dashboard.backend.ingest import ingest_backtest_result

def run_pipeline(domain: str):
    db = get_db()
    
    # ... existing pipeline logic ...
    
    # After analysis phase — store the report
    db.execute("""
        INSERT INTO reports (id, domain, date, report_type, analysis_text, suggestions)
        VALUES (?, ?, ?, 'daily', ?, ?)
    """, (report_id, domain, today, analysis_text, json.dumps(suggestions)))
    
    # After backtest — store results
    if backtest_result:
        ingest_backtest_result(backtest_result, db)
    
    # After Gate 1 approval — store what was approved
    db.execute("""
        UPDATE reports SET approved = ? WHERE id = ?
    """, (json.dumps(approved_items), report_id))
    
    # After config/code change — log it
    db.execute("""
        INSERT INTO change_log (domain, timestamp, change_type, description, before_value, after_value)
        VALUES (?, ?, ?, ?, ?, ?)
    """, (domain, now, change_type, description, json.dumps(before), json.dumps(after)))
    
    db.commit()
```

---

## Telegram Message Examples (Updated)

### Equities Gate 1 Report (unchanged, with emoji tag)

```
📈 Equities Daily Report — March 7, 2026

💰 P&L: +$342.50 (12 trades, 66.7% win rate)
📊 10-Day: +$1,847.20 (Sharpe: 1.84)
⚡ Bot uptime: 100%

📝 Analysis:
Strong morning session driven by tech rally. VWAP momentum
captured NVDA move well (+$185). Three losing trades clustered
in final hour as momentum faded.

📋 Suggestions:

1️⃣ [CONFIG] Tighten afternoon stop loss: 1.5% → 1.2%
   Backtest: would have saved $47 today
   
2️⃣ [CODE] New mean reversion strategy for SPY
   Expected: captures $50-150/day on range-bound days

Reply: "approve all", "approve 1, skip 2", etc.
```

### Crypto Gate 1 Report (new)

```
₿ Crypto Daily Report — March 7, 2026 (UTC)

💰 P&L: +$127.40 (34 fills, grid + trend)
📊 10-Day: +$892.10 (Sharpe: 1.23)
⚡ Bot uptime: 99.8% (2 min exchange timeout at 03:14 UTC)
💱 Exchange: Binance (sandbox mode)

📝 Per-Strategy:
  Grid (BTC/USDT): +$89.20 — 28 fills, range held well
  Grid (ETH/USDT): +$41.50 — 6 fills, tighter range today
  Trend (SOL/USDT): -$3.30 — no clear trend, 0 entries

📝 Analysis:
BTC held $67k-68k range, ideal for grid. ETH correlation
drifting — may want independent grid params. SOL trend
strategy correctly stayed flat during chop.

Funding rates: BTC +0.01% (neutral), ETH +0.03% (slightly long-heavy)

📋 Suggestions:

1️⃣ [CONFIG] Widen ETH grid spacing: 0.5% → 0.7%
   Rationale: ETH swings wider than BTC, current grid
   getting whipsawed. Backtest: +$12/day improvement.

2️⃣ [CONFIG] Add SOL/USDT to grid strategy
   Rationale: SOL range-bound 3 of last 5 days.
   Grid would have captured $25-40/day.

3️⃣ [CODE] Add exchange health monitor with auto-pause
   Rationale: 2-min timeout today. Bot should pause
   trading during exchange issues, not just retry.

Reply: "approve all", "approve 1 and 3, skip 2", etc.
```

### Crypto Gate 2 Deploy

```
₿🔨 Build Complete — Exchange Health Monitor

✅ Tests: 38/38 passing
📈 Backtest: N/A (infrastructure change, not strategy)
🧪 Integration test: simulated 5 timeout scenarios, all handled

📁 Changes:
   + crypto/bot/execution/health_monitor.py (new, 98 lines)
   + crypto/tests/test_health_monitor.py (new, 145 lines)
   ~ crypto/bot/main.py (+12 lines: health check integration)
   ~ crypto/bot/config.yaml (+3 lines: health monitor params)

🔗 Diff: github.com/you/trading-system/compare/main...staging/crypto/2026-03-07

Reply DEPLOY or REJECT
```

### Emergency: Crypto Kill Switch via Telegram

```
You: "kill crypto"

Bot: ⚠️ Emergency shutdown initiated for crypto domain.
     Cancelling all open orders...
     ✅ 4 orders cancelled (2 BTC, 2 ETH)
     Market-selling open positions...
     ✅ Positions closed: BTC 0.015 @ $67,420, ETH 0.5 @ $3,847
     Realized P&L on emergency close: -$23.40
     
     Crypto bot stopped. Reply "start crypto" to restart.
```

---

## CLAUDE.md (Updated for Multi-Domain)

```markdown
# Trading System — Multi-Domain

## Architecture
This is a monorepo containing trading bots for two domains:

1. **Equities** — Python bot connecting to Das Trader Pro via websocket
   on localhost (127.0.0.1). Runs on Windows. Trades US equities
   during market hours (9:30-16:00 ET).

2. **Crypto** — Python bot connecting to exchanges via ccxt library.
   Runs on Mac as a launchd service. Trades crypto pairs 24/7.

Development, review, and automated builds happen on macOS.

## Repo Layout
- `equities/` — Equities bot code, logs, tests, deploy scripts
- `crypto/` — Crypto bot code, logs, tests, deploy scripts  
- `shared/` — Common models, indicators, and risk primitives
- `review/` — Daily review pipeline (serves both domains)
- `dashboard/` — Web dashboard (FastAPI backend + React frontend)
- `state/` — Per-domain state files + SQLite DB (not in git)

## How This Repo Is Used
- **Daily automated pipelines** run on Mac via cron. Separate
  schedules for equities (16:30 ET) and crypto (00:00 UTC).
  Each runs a Claude Code headless analysis session (reads code +
  trade data, produces suggestions), then (with user approval)
  modifies config or runs a second Claude Code headless session
  for code generation on a staging branch.
- **Manual interactive sessions** happen occasionally for complex
  work or failed build fixes.

## Key Constraints — Equities
- Das Trader connection is websocket-only, localhost only, Windows only
- Bot must handle websocket disconnections with auto-reconnect
- New strategies must inherit from equities.bot.strategies.base.Strategy

## Key Constraints — Crypto
- Uses ccxt for exchange connectivity — NEVER hardcode exchange-specific APIs
- API keys are environment variables, NEVER in config files or code
- Bot runs as persistent launchd service — must handle restarts gracefully
- Must respect exchange rate limits (ccxt handles most, but be aware)
- New strategies must inherit from crypto.bot.strategies.base.CryptoStrategy
- Sandbox mode by default — never switch to live without explicit config change

## Key Constraints — Both
- All trade logs must be structured JSON for automated analysis
- risk_manager.py is safety-critical in BOTH domains — extra care
- All code changes must pass full test suite + backtester before merge
- 3-day rule: wait 3 days between changes per domain to observe impact
- Pipeline writes reports, backtests, and changes to dashboard SQLite DB
- Dashboard is read-only — it never modifies bot config or state

## Dashboard
- FastAPI backend in dashboard/backend/, React frontend in dashboard/frontend/
- SQLite database at state/dashboard.db — single source of truth for historical data
- Dashboard is read-only: it queries data but never modifies bot behavior
- Pipeline writes to SQLite directly at key points (reports, backtests, changes)
- Trade log ingestion syncs JSON log files → SQLite every 5 minutes
- Live data via WebSocket: reads crypto bot state locally, equities via SSH poll
- Auth: single bearer token in DASHBOARD_TOKEN env var

## Equities Strategy Interface
- setup() — initialize indicators, state
- on_bar(bar) — called on each new price bar
- on_tick(tick) — called on each tick (optional)
- teardown() — cleanup at end of day

## Crypto Strategy Interface
- setup(connector, config) — initialize, subscribe to pairs
- on_candle(symbol, candle) — called on each candle close
- on_tick(symbol, ticker) — called on each price update (optional)
- on_fill(order) — called when an order fills
- should_pause() — return True to pause during bad conditions
- teardown() — cancel orders, cleanup

## Conventions
- Python 3.11+, type hints on all functions
- Docstrings explaining trading logic, not just code
- Trade logs: JSON with timestamp, symbol, side, qty, price,
  strategy, pnl, metadata (domain-specific fields allowed)
- Health heartbeat every 60 seconds to logs/health/
- Tests required for all strategy logic
- Shared code in shared/ must not import from equities/ or crypto/

## When Running Headless (via pipeline)
You are being called by an automated build pipeline. A build spec
will be provided as your prompt. Follow it precisely:
1. Read the spec completely before writing any code
2. Note which domain this spec targets (equities or crypto)
3. Implement exactly what is specified — no more, no less
4. Create all tests specified
5. Run pytest for the target domain and report results
6. Run the backtester if specified and report results
7. Do not modify files listed under "Files NOT to Touch"
8. Do not modify files in the OTHER domain unless spec says to
```

---

## Cron Schedule (Mac M1 Pro — Updated)

```crontab
# ═══════════════════════════════════════════════════
# EQUITIES
# ═══════════════════════════════════════════════════

# Daily review pipeline — 30 min after market close
30 16 * * 1-5  cd ~/trading-system && python -m review.pipeline --domain equities >> logs/equities-pipeline.log 2>&1

# Morning health check — verify equities bot started and is connected
35 9 * * 1-5   cd ~/trading-system && python -m review.health_checker --domain equities --morning >> logs/equities-health.log 2>&1

# ═══════════════════════════════════════════════════
# CRYPTO
# ═══════════════════════════════════════════════════

# Daily review pipeline — midnight UTC (7pm ET / 8pm ET DST)
0 0 * * *      cd ~/trading-system && python -m review.pipeline --domain crypto >> logs/crypto-pipeline.log 2>&1

# Health check — every 4 hours (crypto runs 24/7)
0 */4 * * *    cd ~/trading-system && python -m review.health_checker --domain crypto >> logs/crypto-health.log 2>&1

# Git watcher — check for new commits every 60 seconds (restarts crypto bot)
* * * * *      cd ~/trading-system && bash crypto/deploy/watcher.sh >> logs/crypto-watcher.log 2>&1

# ═══════════════════════════════════════════════════
# COMBINED
# ═══════════════════════════════════════════════════

# Weekend summary — broader analysis across both domains
0 10 * * 6     cd ~/trading-system && python -m review.pipeline --domain all --weekly >> logs/weekly-pipeline.log 2>&1

# ═══════════════════════════════════════════════════
# DASHBOARD
# ═══════════════════════════════════════════════════

# Trade log ingestion — sync JSON logs to SQLite every 5 minutes
*/5 * * * *    cd ~/trading-system && python -m dashboard.backend.ingest >> logs/dashboard-ingest.log 2>&1

# Dashboard runs as a launchd service (not cron) — see dashboard/deploy/dashboard.plist
```

---

## Walk-Through: A Full Day (Multi-Domain)

```
00:00  CRYPTO PIPELINE FIRES (midnight UTC).
       Reads local crypto trade logs from yesterday.
       Health check: crypto bot connected 24h, 2 brief timeouts. ✅
       Claude Code analysis: grid strategy performing well on BTC,
       ETH grid too tight. Suggests widening ETH grid spacing.
       Pipeline writes report + backtest results to dashboard SQLite.
       
00:02  Telegram: "₿ Crypto Daily Report..."
       You're asleep. Pipeline enters approval listener mode.
       (Timeout: 12 hours. If no response, logs "skipped" and moves on.)

07:00  You wake up. Read crypto report on phone. Reply:
       "approve 1, skip 2 and 3"
       
07:01  Config change applied to crypto/bot/config.yaml.
       Git push to main. Mac watcher detects, restarts crypto bot.
       Telegram: "₿ ✅ Config applied: ETH grid spacing 0.5→0.7%"
       Change logged to dashboard SQLite.
       
       (Crypto bot is now running with updated config.)

07:05  Over coffee, you open the dashboard on your phone.
       Home page shows: crypto +$127 yesterday, 3 strategies active,
       equities market opens in 2h25m. Equity curve trending up.
       You tap into Strategies — see the ETH grid spacing just
       changed. Previous 3 config changes listed with impact notes.

09:30  Market opens. Equities bot on Windows connects to Das Trader.
       Starts executing VWAP momentum strategy.

09:35  Equities morning health check fires on Mac. Confirms equities
       bot is connected and trading. No notification (all good).

12:00  Both bots running simultaneously:
       - Equities bot: 8 trades so far on NVDA, SPY
       - Crypto bot: 15 grid fills on BTC, 4 on ETH (wider spacing working)
       
       During lunch, you pull up the dashboard Live View.
       See 1 open equities position (NVDA long, +$42 unrealized).
       Crypto: 3 grid orders pending on BTC, 2 on ETH.
       Today's combined P&L: +$189 and counting.

16:00  Equities market closes. Equities bot enters idle.
       Crypto bot keeps trading.

16:30  EQUITIES PIPELINE FIRES.
       Mac SSHs into Windows, pulls equities trade logs.
       Claude Code analysis: good day, +$342. Suggests new strategy.
       Writes report + analysis to dashboard SQLite.
       
16:38  Telegram: "📈 Equities Daily Report..."

17:00  You read the equities report at the gym.
       "approve 1 and 2"
       
17:01  CONFIG + CODE changes kick off for equities domain.
       (Crypto bot is unaffected, still trading.)

17:06  Gate 2: "📈🔨 Build Complete — Mean Reversion Strategy..."
       You check the diff, reply "deploy".
       
17:07  Merged to main. Windows watcher will pull tomorrow AM.
       Telegram: "📈 ✅ Mean reversion deployed. Active at next open."

       Meanwhile, crypto bot has been happily running all day.

21:00  Before bed, you check the dashboard Performance page.
       Monthly returns table: February was +$4,200 equities,
       +$1,800 crypto. March on pace for similar.
       P&L heatmap shows: equities strongest 10am-12pm,
       crypto grid captures most fills during Asian session.
       You browse the Backtests page — compare the 3 grid
       spacing options the pipeline tested last week.

00:00  Next midnight UTC. Crypto pipeline fires again.
       Reports on ETH grid spacing change: "Working well, +18% more
       fills with wider spacing." 3-day observation period continues.
       Dashboard updates automatically — you'll see it tomorrow.
```

---

## Build Order (Updated for Multi-Domain)

### Phase 1: Get equities bot trading on paper (Week 1-2)
Use Claude Code to build:
- Das Trader websocket connector
- One simple strategy (VWAP momentum)
- Structured JSON trade logging
- Health heartbeat writer
- config.yaml with tunable parameters
- Basic tests and backtester framework

Deploy to Windows. Connect to Das Trader paper trading.

### Phase 2: Daily review — reports only (Week 2-3)
With real equities trade data flowing:
- SSH data puller (Mac → Windows)
- Health checker
- Claude Code headless analysis session (equities prompts)
- Telegram bot for report delivery
- context.json state management
- `--domain` flag infrastructure in pipeline

No approval workflow yet. Just receive and observe reports.

### Phase 3: Gate 1 — config changes for equities (Week 3-4)
- Strategy advisor prompt (equities)
- Backtest proposed changes
- Telegram approval listener
- Config change executor + git push
- Change log tracking

### Phase 4: Gate 2 — code changes for equities (Week 4-5)
- Build spec writer prompt
- Claude Code headless integration (code_builder.py)
- Independent test/backtest validation
- Gate 2 Telegram flow (deploy_gate.py)
- Staging branch management
- Failure handling + retry logic

### Phase 5: Crypto bot foundation (Week 5-6)
- Exchange connector (ccxt wrapper)
- One simple crypto strategy (grid trading)
- Crypto-specific risk manager
- Trade logging (same JSON schema, crypto fields)
- Health heartbeat + exchange health monitor
- launchd service setup + git watcher on Mac
- Crypto backtester (pulling historical OHLCV)
- Tests for all crypto components

Deploy on Mac. Connect to exchange sandbox (paper trading).

### Phase 6: Crypto pipeline integration (Week 6-7)
With real crypto trade data flowing:
- Crypto prompts for trade analyst, strategy advisor, pattern scanner
- Domain-aware data puller (local read for crypto)
- Crypto review schedule (cron at midnight UTC)
- Crypto health checks (every 4 hours)
- Separate state files (state/crypto/*)
- Gate 1 + Gate 2 working for crypto domain

### Phase 7: Dashboard — core views (Week 7-8)
- SQLite schema + migrations
- Trade log ingestion (JSON → SQLite sync)
- FastAPI backend skeleton + auth
- REST routes: performance summary, trade history, strategies
- React frontend: home dashboard, performance page, trade browser
- Strategy cards with current config and P&L
- launchd service for dashboard
- Pipeline integration: writes reports + backtests to SQLite

Dashboard is usable with historical data at this point.

### Phase 8: Dashboard — live data + advanced views (Week 8-9)
- WebSocket for live positions (crypto: local read, equities: SSH poll)
- Live View page: open positions, orders, real-time P&L
- Backtest browser + comparison mode
- System health page: uptime charts, pipeline history, change log
- Reports archive: browse daily Claude analysis text
- P&L heatmap (day × hour), monthly returns table
- CSV export for trade history
- Tailscale setup for remote access

### Phase 9: Polish and go live (Week 9+)
- Pattern scanner prompts (both domains)
- Weekend combined summary reports
- Switch equities from paper to live (minimal positions)
- Switch crypto from sandbox to live (minimal positions)
- Tighten risk limits both domains
- Monitor daily for 2+ weeks per domain before scaling
- Telegram kill switch command for each domain
- Set up .claude/agents/ for manual escape hatch sessions

### Future: Optional enhancements
- Run a local model on Mac for cheaper analysis calls
- Add market data feeds for pre-market equities analysis
- Multi-strategy portfolio optimization (cross-domain)
- Slack/Discord integration alongside Telegram
- Multiple exchange support for crypto (arbitrage opportunities)
- Futures/perpetuals support for crypto (funding rate strategies)
- Cross-domain correlation analysis (crypto-equities hedging)
- Dashboard: mobile-optimized PWA for phone-native experience
- Dashboard: push notifications for threshold alerts (drawdown, uptime)
- Dashboard: interactive backtest launcher (run backtests from the UI)
- Dashboard: strategy editor (preview config changes before pushing via Telegram)
