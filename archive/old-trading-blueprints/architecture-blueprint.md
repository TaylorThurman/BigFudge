# Trading Bot Agentic System — Architecture Blueprint

*Version 0.1 — March 7, 2026*

---

## System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        YOU (Product Owner)                          │
│         Telegram on your phone — approve, reject, modify            │
└───────────────┬──────────────────────────────┬──────────────────────┘
                │                              │
        Daily reports &                Interactive build
        approval requests              sessions when needed
                │                              │
┌───────────────▼──────────────────┐ ┌─────────▼──────────────────────┐
│     MAC (M1 Pro, always on)      │ │   MAC (M2 Air, your laptop)    │
│                                  │ │                                │
│  ┌────────────────────────────┐  │ │  ┌────────────────────────┐   │
│  │   Daily Review Pipeline    │  │ │  │  Claude Code Agent     │   │
│  │   (Python cron job)        │  │ │  │  Teams (interactive)   │   │
│  │                            │  │ │  │                        │   │
│  │  • Pull trade data (SSH)   │  │ │  │  • Team Lead (Opus)    │   │
│  │  • Health check            │  │ │  │  • Backend Engineer    │   │
│  │  • Claude API analysis     │  │ │  │  • Test Engineer       │   │
│  │  • Send Telegram report    │  │ │  │  • Code Reviewer       │   │
│  │  • Wait for approval       │  │ │  │                        │   │
│  │  • Execute changes         │  │ │  │  Builds & refactors    │   │
│  │  • Git push                │  │ │  │  the trading bot       │   │
│  └────────────────────────────┘  │ │  └───────────┬────────────┘   │
│                                  │ │              │                 │
│  ┌────────────────────────────┐  │ │              │ git push       │
│  │   State Store              │  │ │              │                 │
│  │   ~/trading-system/state/  │  │ └──────────────┼────────────────┘
│  │                            │  │                │
│  │  • context.json (rolling)  │  │                │
│  │  • trade_history/          │  │                │
│  │  • change_log.json         │  │                │
│  │  • strategies.json         │  │                │
│  └────────────────────────────┘  │                │
└───────────────┬──────────────────┘                │
                │                                   │
                │ git push (param changes)          │
                │                                   │
┌───────────────▼───────────────────────────────────▼─────────────────┐
│                    GITHUB (Private Repo)                             │
│                    trading-bot/                                      │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                │ auto-pull + restart
                                │
┌───────────────────────────────▼─────────────────────────────────────┐
│                WINDOWS DESKTOP (on during market hours)              │
│                                                                     │
│  ┌──────────────┐    websocket     ┌─────────────────────────────┐  │
│  │  Das Trader   │◄──────────────►│  Trading Bot (Python)        │  │
│  │  (GUI app)    │  127.0.0.1      │                             │  │
│  │               │                 │  • Executes strategies      │  │
│  └──────────────┘                 │  • Writes trade logs         │  │
│                                    │  • Writes connection status  │  │
│  ┌─────────────────────────────┐  │  • Writes health heartbeat   │  │
│  │  Git Watcher (PowerShell)   │  └─────────────────────────────┘  │
│  │  Polls repo every 60s       │                                    │
│  │  Restarts bot on new commits│  ┌─────────────────────────────┐  │
│  └─────────────────────────────┘  │  OpenSSH Server (enabled)   │  │
│                                    │  Mac pulls logs via SCP     │  │
│                                    └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Project Structure

```
trading-bot/                          ← Git repo (shared between all machines)
│
├── bot/                              ← The actual trading bot (runs on Windows)
│   ├── main.py                       ← Entry point, websocket connection to Das Trader
│   ├── config.yaml                   ← Strategy parameters (what the daily cycle modifies)
│   ├── strategies/
│   │   ├── base.py                   ← Abstract strategy class
│   │   ├── vwap_momentum.py          ← Example strategy
│   │   └── mean_reversion.py         ← Another strategy
│   ├── execution/
│   │   ├── das_connector.py          ← Das Trader websocket client
│   │   ├── order_manager.py          ← Order routing, position tracking
│   │   └── risk_manager.py           ← Position sizing, drawdown limits, kill switches
│   ├── data/
│   │   ├── market_data.py            ← Real-time data feed handling
│   │   └── indicators.py             ← Technical indicator calculations
│   └── utils/
│       ├── logger.py                 ← Structured logging (JSON format)
│       └── health.py                 ← Heartbeat writer, connection status
│
├── logs/                             ← Written by bot on Windows, pulled by Mac
│   ├── trades/                       ← One JSON file per day: 2026-03-07.json
│   ├── health/                       ← Connection status, heartbeats
│   └── errors/                       ← Exceptions, disconnections
│
├── review/                           ← Daily review pipeline (runs on Mac)
│   ├── pipeline.py                   ← Main orchestration script (cron entry point)
│   ├── data_puller.py                ← SSH into Windows, grab trade logs
│   ├── health_checker.py             ← Validate data completeness
│   ├── analyst.py                    ← Claude API calls for trade analysis
│   ├── reporter.py                   ← Format and send Telegram messages
│   ├── approval_listener.py          ← Telegram bot, waits for your response
│   ├── change_executor.py            ← Applies approved changes (config or code)
│   └── prompts/                      ← System prompts for each "agent" role
│       ├── trade_analyst.md          ← Analyzes today's performance
│       ├── strategy_advisor.md       ← Suggests parameter/strategy changes
│       ├── pattern_scanner.md        ← Looks for market themes and patterns
│       └── code_modifier.md          ← Generates specific code/config changes
│
├── state/                            ← Persistent state (lives on Mac, not in git)
│   ├── context.json                  ← Rolling context: last 10 days of summaries
│   ├── change_log.json               ← Every change made, when, why, result
│   ├── strategies.json               ← Active strategies and their parameters
│   ├── performance_baseline.json     ← What "normal" looks like for each strategy
│   └── pending_structural.json       ← Queued changes too big for auto-apply
│
├── tests/                            ← Bot tests (used during build sessions)
│   ├── test_strategies.py
│   ├── test_risk_manager.py
│   ├── test_das_connector.py
│   └── backtester.py                 ← Run historical data through strategies
│
├── deploy/
│   ├── watcher.ps1                   ← PowerShell script: polls git, restarts bot
│   └── setup_windows.md             ← Instructions for Windows machine setup
│
├── .claude/
│   ├── settings.json                 ← Agent Teams enabled, model preferences
│   ├── agents/                       ← Custom subagent definitions
│   │   ├── backend-engineer.md
│   │   ├── test-engineer.md
│   │   └── code-reviewer.md
│   └── commands/
│       ├── build-strategy.md         ← /build-strategy slash command
│       ├── refactor.md               ← /refactor slash command
│       └── code-review.md            ← /code-review slash command
│
├── CLAUDE.md                         ← Project context for Claude Code
├── requirements.txt                  ← Python dependencies for bot
├── requirements-review.txt           ← Python dependencies for review pipeline
└── README.md
```

---

## Component 1: Claude Code Agent Teams (Build Side)

This is for interactive sessions when you want to build a new strategy, refactor existing code, or make structural changes. You sit at your M2 Air and direct the team.

### CLAUDE.md (Project Context)

```markdown
# Trading Bot System

## Architecture
This is a Python trading bot that connects to Das Trader Pro via websocket
on localhost (127.0.0.1). The bot runs on a Windows machine. Development
and review happen on macOS.

## Key Constraints
- Das Trader connection is websocket-only, localhost only, Windows only
- Bot must handle websocket disconnections gracefully with auto-reconnect
- All trade logs must be structured JSON for automated analysis
- config.yaml is the ONLY file the daily review pipeline modifies automatically
- Strategy logic changes require a full interactive build session
- Risk manager is a safety-critical module — changes require extra review

## File Ownership Rules (for Agent Teams)
- bot/strategies/ → Backend Engineer owns
- bot/execution/ → Backend Engineer owns
- bot/data/ → Backend Engineer owns
- tests/ → Test Engineer owns
- bot/utils/ → Shared — coordinate before editing
- config.yaml → NEVER edited during build sessions (review pipeline only)

## Conventions
- Python 3.11+, type hints on all functions
- Strategies must inherit from strategies.base.Strategy
- All strategies must implement: setup(), on_bar(), on_tick(), teardown()
- Trade logs: JSON with fields: timestamp, symbol, side, qty, price,
  strategy, pnl, metadata
- Health heartbeat: write to logs/health/ every 60 seconds
- Tests required for all strategy logic — no merge without passing tests

## Config Structure (config.yaml)
Parameters the daily review can modify:
- strategy.*.enabled (bool)
- strategy.*.position_size (int)
- strategy.*.stop_loss_pct (float)
- strategy.*.take_profit_pct (float)
- strategy.*.entry_threshold (float)
- strategy.*.lookback_period (int)
- risk.max_daily_loss (float)
- risk.max_position_count (int)

Parameters that require interactive session:
- Adding/removing strategies
- Changing strategy logic
- Modifying execution/order routing
- Any changes to risk_manager.py
```

### .claude/settings.json

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },
  "model": "opus",
  "permissions": {
    "allow": [
      "Read",
      "Write",
      "Edit",
      "Bash(python*)",
      "Bash(pytest*)",
      "Bash(git*)",
      "Bash(cat*)",
      "Bash(ls*)",
      "Bash(grep*)"
    ]
  }
}
```

### Agent Definitions

**.claude/agents/backend-engineer.md**
```markdown
---
name: backend-engineer
description: Implements trading bot strategy logic, execution, and data handling
tools:
  allow: [Read, Write, Edit, Bash, Glob, Grep]
model: sonnet
---

You are a senior Python backend engineer specializing in algorithmic trading
systems. You write clean, typed, well-documented code.

Key responsibilities:
- Implement trading strategies in bot/strategies/
- Build and maintain the Das Trader connector in bot/execution/
- Handle market data processing in bot/data/

Rules:
- NEVER modify config.yaml — that's managed by the review pipeline
- NEVER modify files in tests/ — the Test Engineer owns those
- Always handle websocket disconnections with reconnect logic
- Every strategy must inherit from strategies.base.Strategy
- Include docstrings explaining the trading logic, not just the code
- Log all trades as structured JSON
```

**.claude/agents/test-engineer.md**
```markdown
---
name: test-engineer
description: Writes tests, runs backtests, validates edge cases
tools:
  allow: [Read, Write, Edit, Bash, Glob, Grep]
model: sonnet
---

You are a QA engineer focused on financial software testing.
You understand that bugs in trading systems cost real money.

Key responsibilities:
- Write unit tests for all strategy logic in tests/
- Build and maintain the backtester in tests/backtester.py
- Validate edge cases: market open/close, halts, gaps, partial fills
- Run backtests against proposed strategies before they go live

Rules:
- Own all files in tests/
- Test failure scenarios: what happens when Das Trader disconnects
  mid-order? When the market gaps through a stop loss?
- Every strategy needs tests for: entry signals, exit signals,
  position sizing, stop loss behavior, and daily P&L limits
- Backtest results must include: sharpe ratio, max drawdown,
  win rate, profit factor, average win/loss ratio
```

**.claude/agents/code-reviewer.md**
```markdown
---
name: code-reviewer
description: Reviews all code for correctness, safety, and trading-specific risks
tools:
  allow: [Read, Glob, Grep]
  deny: [Write, Edit, Bash]
model: opus
---

You are a senior code reviewer with expertise in trading systems.
You catch bugs that lose money.

Focus areas:
- Off-by-one errors in indicator calculations
- Race conditions in order execution
- Float precision issues in price comparisons
- Missing error handling for API/websocket failures
- Risk manager bypass (can any code path skip risk checks?)
- Hardcoded values that should be in config.yaml
- State that doesn't reset cleanly between trading days

You CANNOT modify files. You report issues to the team lead
with file, line number, severity (critical/high/medium/low),
and a clear explanation of the risk.
```

### Slash Commands

**.claude/commands/build-strategy.md**
```markdown
---
description: Build a new trading strategy from requirements
---

Create an agent team to build a new trading strategy. Use delegate mode.

Spawn three teammates:
1. Backend Engineer — implements the strategy in bot/strategies/,
   creates the Das Trader integration if needed
2. Test Engineer — writes comprehensive tests and runs backtests
   against historical data
3. Code Reviewer (read-only) — reviews all code for trading-specific
   bugs and risk issues

Workflow:
1. Team lead: break down the requirements into a task list with
   dependencies. The strategy implementation must complete before
   tests can run. Code review happens after both.
2. Backend Engineer submits a plan before coding. Lead reviews.
3. Test Engineer writes tests based on the strategy spec, not the
   implementation (test-first).
4. After implementation, Test Engineer runs tests and backtests.
5. Code Reviewer does final review.
6. Lead synthesizes: test results, backtest metrics, reviewer
   findings, and a go/no-go recommendation.

User requirements: $ARGUMENTS
```

---

## Component 2: Daily Review Pipeline (Autonomous Side)

This runs on your M1 Pro via cron. It's a Python script, not an agent framework.

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    DAILY PIPELINE FLOW                        │
│                                                              │
│  16:30  ┌──────────────┐                                    │
│  ──────►│ 1. Pull Data  │ SSH into Windows, grab today's    │
│         │              │ trade logs + health logs            │
│         └──────┬───────┘                                    │
│                │                                             │
│                ▼                                             │
│         ┌──────────────┐                                    │
│         │ 2. Health    │ Validate: was bot connected all    │
│         │    Check     │ day? Any gaps? Complete data?       │
│         └──────┬───────┘                                    │
│                │                                             │
│                ▼                                             │
│         ┌──────────────┐                                    │
│         │ 3. Load      │ Read context.json (last 10 days),  │
│         │    Context   │ change_log, strategy params         │
│         └──────┬───────┘                                    │
│                │                                             │
│                ▼                                             │
│         ┌──────────────┐  Claude API call #1                │
│         │ 4. Analyze   │  System: trade_analyst.md          │
│         │    Trades    │  Input: today's trades + context   │
│         │              │  Output: performance summary        │
│         └──────┬───────┘                                    │
│                │                                             │
│                ▼                                             │
│         ┌──────────────┐  Claude API call #2                │
│         │ 5. Strategy  │  System: strategy_advisor.md       │
│         │    Review    │  Input: analysis + multi-day trend  │
│         │              │  Output: suggested changes          │
│         └──────┬───────┘                                    │
│                │                                             │
│                ▼                                             │
│         ┌──────────────┐  (Optional) Claude API call #3     │
│         │ 6. Pattern   │  System: pattern_scanner.md        │
│         │    Scan      │  Input: market data + sector moves  │
│         │              │  Output: new strategy ideas          │
│         └──────┬───────┘                                    │
│                │                                             │
│                ▼                                             │
│         ┌──────────────┐                                    │
│         │ 7. Backtest  │  If changes suggested, run quick   │
│         │    Proposed  │  backtest against today's data      │
│         │    Changes   │  with proposed new parameters       │
│         └──────┬───────┘                                    │
│                │                                             │
│                ▼                                             │
│         ┌──────────────┐                                    │
│         │ 8. Send      │  Telegram message with:            │
│         │    Report    │  • P&L summary                     │
│         │              │  • Key metrics                      │
│         │              │  • Suggested changes + backtest     │
│         │              │  • "Reply APPROVE, REJECT, or       │
│         │              │     your modifications"             │
│         └──────┬───────┘                                    │
│                │                                             │
│                ▼                                             │
│         ┌──────────────┐                                    │
│         │ 9. Wait for  │  Telegram bot listener             │
│         │    Response  │  Timeout: 4 hours, then remind     │
│         │              │  Parses with Claude API call #4     │
│         └──────┬───────┘                                    │
│                │                                             │
│           ┌────┴────┐                                        │
│           ▼         ▼                                        │
│     ┌──────────┐ ┌──────────┐                               │
│     │ APPROVED │ │ REJECTED │──► Log rejection, done        │
│     └────┬─────┘ └──────────┘                               │
│          │                                                   │
│          ▼                                                   │
│   ┌────────────────┐                                        │
│   │ 10. Categorize │                                        │
│   │     Change     │                                        │
│   └───┬────────┬───┘                                        │
│       │        │                                             │
│       ▼        ▼                                             │
│  ┌─────────┐ ┌───────────┐                                  │
│  │ CONFIG  │ │ STRUCTURAL│                                   │
│  │ CHANGE  │ │ CHANGE    │                                   │
│  │         │ │           │                                   │
│  │ Modify  │ │ Add to    │                                   │
│  │ config  │ │ pending_  │                                   │
│  │ .yaml   │ │ structural│                                   │
│  │ git push│ │ .json for │                                   │
│  │ done    │ │ next      │                                   │
│  └─────────┘ │ interactive                                   │
│              │ session   │                                   │
│              └───────────┘                                   │
│                                                              │
│  11. Update state: context.json, change_log.json             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### State Management (context.json)

This is the "memory" that persists across days. It gets passed into every Claude API call so the analyst has historical context.

```json
{
  "last_updated": "2026-03-07T16:45:00",
  "rolling_window_days": 10,
  "daily_summaries": [
    {
      "date": "2026-03-07",
      "total_pnl": 342.50,
      "trade_count": 12,
      "win_rate": 0.667,
      "largest_win": 185.00,
      "largest_loss": -92.30,
      "strategies_active": ["vwap_momentum"],
      "bot_uptime_pct": 100.0,
      "notable_events": ["Strong tech rally, NVDA +4.2%"],
      "changes_applied": "Reduced position size from 500 to 400 shares (applied 03-06)",
      "change_impact": "Position size reduction appears to have reduced volatility without significantly impacting P&L"
    }
  ],
  "cumulative_metrics": {
    "total_pnl_10d": 1847.20,
    "avg_daily_pnl": 184.72,
    "sharpe_10d": 1.84,
    "max_drawdown_10d": -412.00,
    "win_rate_10d": 0.62
  },
  "active_strategies": {
    "vwap_momentum": {
      "enabled": true,
      "current_params": {
        "position_size": 400,
        "stop_loss_pct": 1.5,
        "take_profit_pct": 3.0,
        "lookback_period": 20
      },
      "performance_since_last_change": {
        "days": 1,
        "pnl": 342.50,
        "trades": 12
      }
    }
  },
  "pending_ideas": [
    "Strategy advisor suggested a mean reversion strategy for range-bound days (03-05). Queued for interactive session."
  ]
}
```

### Example Prompt: Trade Analyst (review/prompts/trade_analyst.md)

```markdown
You are a senior quantitative trading analyst reviewing the daily
performance of an automated trading system.

You will receive:
1. Today's trade log (every trade with timestamps, P&L, and metadata)
2. A rolling context of the last 10 trading days
3. Current strategy parameters
4. A log of recent changes and their impact

Your job:
- Summarize today's performance with key metrics
- Compare to the 10-day baseline — is today normal or anomalous?
- Identify which trades were strong and which were problematic
- Note any patterns: time-of-day effects, sector correlations,
  volatility regime changes
- If a parameter change was recently applied, assess its impact
  vs the baseline (minimum 3 days of data before concluding)
- Flag any risk concerns: approaching daily loss limits, unusual
  drawdown patterns, concentrated exposure

Be specific. Reference actual trades by timestamp and symbol.
Don't give vague "the strategy performed well" summaries.

Output format:
{
  "summary": "2-3 sentence overview",
  "metrics": { ... },
  "trade_analysis": [ ... ],
  "patterns_observed": [ ... ],
  "risk_flags": [ ... ],
  "change_impact_assessment": "..." or null if no recent changes
}
```

### Example Prompt: Strategy Advisor (review/prompts/strategy_advisor.md)

```markdown
You are a senior trading strategist reviewing performance data and
suggesting optimizations to an automated trading system.

You will receive:
1. Today's trade analysis (from the trade analyst)
2. Rolling 10-day context with cumulative metrics
3. Current strategy parameters and configuration
4. History of past changes and their measured impact

Your job:
- Based on the data, suggest specific parameter changes if warranted
- Be conservative — only suggest changes when there's clear evidence
- Never suggest a change within 3 days of the last change (need data)
- Quantify expected impact: "Tightening stop loss from 1.5% to 1.2%
  would have saved approximately $X on today's losing trades"
- If no changes are needed, say so. Doing nothing is a valid recommendation.
- If you see an opportunity for a new strategy, describe it briefly
  and flag it as a "structural suggestion" for an interactive session

Categorize each suggestion:
- CONFIG: can be applied by modifying config.yaml (parameter tuning)
- STRUCTURAL: requires code changes (new strategy, logic changes)

For CONFIG suggestions, output exact YAML changes:
{
  "type": "CONFIG",
  "changes": {
    "strategy.vwap_momentum.stop_loss_pct": 1.2,
    "strategy.vwap_momentum.position_size": 350
  },
  "rationale": "...",
  "expected_impact": "...",
  "risk": "..."
}

For STRUCTURAL suggestions, output a brief spec:
{
  "type": "STRUCTURAL",
  "title": "Mean reversion strategy for range-bound days",
  "description": "...",
  "priority": "medium",
  "queue_for_interactive": true
}
```

---

## Component 3: Windows Deployment

### Git Watcher (deploy/watcher.ps1)

```powershell
# Runs on Windows, polls git repo for changes to bot/ or config.yaml
# Restarts the trading bot when changes are detected

$REPO_PATH = "C:\trading-bot"
$BOT_PROCESS_NAME = "trading-bot"
$POLL_INTERVAL = 60  # seconds
$BRANCH = "main"
$LOG_FILE = "$REPO_PATH\logs\watcher.log"

function Write-Log($msg) {
    $ts = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    "$ts | $msg" | Tee-Object -Append $LOG_FILE
}

function Restart-Bot {
    Write-Log "Stopping bot..."
    Get-Process -Name $BOT_PROCESS_NAME -ErrorAction SilentlyContinue |
        Stop-Process -Force
    Start-Sleep -Seconds 2

    Write-Log "Starting bot..."
    Start-Process -FilePath "python" `
        -ArgumentList "$REPO_PATH\bot\main.py" `
        -WorkingDirectory "$REPO_PATH" `
        -WindowStyle Hidden

    Write-Log "Bot restarted."
}

Write-Log "Watcher started. Polling every ${POLL_INTERVAL}s."

while ($true) {
    Set-Location $REPO_PATH

    # Fetch latest
    git fetch origin $BRANCH 2>$null

    # Check if behind
    $LOCAL = git rev-parse HEAD
    $REMOTE = git rev-parse "origin/$BRANCH"

    if ($LOCAL -ne $REMOTE) {
        Write-Log "New commits detected. Pulling..."
        git pull origin $BRANCH

        # Only restart if bot/ or config.yaml changed
        $CHANGED = git diff --name-only "$LOCAL" "$REMOTE"
        if ($CHANGED -match "^(bot/|config\.yaml)") {
            Write-Log "Bot-relevant files changed: $CHANGED"
            Restart-Bot
        } else {
            Write-Log "Changes don't affect bot. Skipping restart."
        }
    }

    Start-Sleep -Seconds $POLL_INTERVAL
}
```

### Enable OpenSSH on Windows

```
Settings → System → Optional Features → Add → OpenSSH Server → Install
```

Then in PowerShell (admin):
```powershell
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

Mac can now pull logs:
```bash
scp user@WINDOWS_IP:C:/trading-bot/logs/trades/2026-03-07.json ./
```

---

## Component 4: Telegram Bot (Approval Workflow)

A lightweight Python bot using `python-telegram-bot` that handles the reporting and approval flow.

### Message Format (what you receive)

```
📊 Daily Trading Report — March 7, 2026

💰 P&L: +$342.50 (12 trades, 66.7% win rate)
📈 10-Day: +$1,847.20 (Sharpe: 1.84)
⚡ Bot uptime: 100%

Key observations:
• Strong performance during morning tech rally
• 3 losing trades clustered in final hour as momentum faded
• VWAP strategy captured the NVDA move well (+$185)

Change impact: Position size reduction (500→400) applied 03-06
has reduced daily P&L variance by ~18% with minimal impact on
total return. Recommend keeping current size.

📋 Suggestions:
1. [CONFIG] Tighten afternoon stop loss to 1.2% (currently 1.5%)
   → Would have saved ~$47 on today's late losses
   → Backtest result: +$31 improvement on today's data

2. [STRUCTURAL] Consider mean reversion strategy for range-bound
   days. Third consecutive day with opportunities the momentum
   strategy missed. Queued for next interactive session.

Reply: APPROVE / REJECT / or describe modifications
```

### Your Response Options

```
You:   "approve"
       → Applies suggestion #1 (config change), queues #2

You:   "approve 1, reject 2"
       → Applies #1 only

You:   "approve but make stop loss 1.3% instead"
       → Script sends to Claude to parse, applies modified version

You:   "reject, strategy is fine, market was unusual today"
       → Logs your reasoning, no changes applied
```

---

## Component 5: Cron Schedule (Mac)

```crontab
# Daily review pipeline — runs 30 min after market close
30 16 * * 1-5  cd ~/trading-system && python -m review.pipeline >> logs/pipeline.log 2>&1

# Morning health check — verify bot started and is connected
35 9 * * 1-5   cd ~/trading-system && python -m review.health_checker --morning >> logs/health.log 2>&1

# Weekend summary — broader analysis on Saturday morning
0 10 * * 6     cd ~/trading-system && python -m review.pipeline --weekly >> logs/pipeline.log 2>&1
```

---

## How the Two Sides Connect

### Interactive Build Session (you at your M2 Air)

```bash
# You open terminal, navigate to repo
cd ~/trading-bot

# Start Claude Code with agent teams
claude

# You type:
> /build-strategy A mean reversion strategy for SPY that activates
> when 5-min ATR drops below the 20-period average. Entry on
> Bollinger Band touches, exit at the mean. Position size 200 shares,
> 0.8% stop loss.

# Agent team spins up, builds the strategy, tests it, reviews it.
# When done, you review the output and commit.

> git push origin main

# Windows machine auto-pulls and restarts with the new strategy.
```

### Daily Autonomous Cycle (no human needed until report)

```
16:30 — Cron fires pipeline.py on Mac
16:31 — Mac SSHs into Windows, pulls today's trade logs
16:32 — Health check passes (bot was connected all day)
16:33 — Loads context.json (last 10 days of history)
16:34 — Calls Claude API: trade analysis
16:35 — Calls Claude API: strategy suggestions
16:36 — Runs quick backtest on proposed parameter changes
16:37 — Sends Telegram report to your phone
         ... you're at dinner, you'll look at it later ...
19:15 — You read the report, reply "approve"
19:15 — Telegram bot picks up your reply
19:15 — Calls Claude API: parse approval (confirmed: apply change)
19:16 — Modifies config.yaml, commits, pushes to git
19:17 — Windows machine pulls, restarts bot with new config
         (market is closed, so no disruption)
19:17 — Updates context.json and change_log.json
19:17 — Sends confirmation: "✅ Applied: stop_loss_pct 1.5→1.2.
         Will monitor for 3 days before suggesting further changes."
```

---

## Build Order (What to Implement First)

### Phase 1: Get a bot trading on paper (Week 1-2)
Use Claude Code (single agent, no teams needed yet) to build:
- Das Trader websocket connector
- One simple strategy (VWAP momentum or whatever your preferred approach)
- Structured JSON trade logging
- Health heartbeat writer
- config.yaml with tunable parameters
- Basic tests

Deploy to Windows. Connect to Das Trader paper trading. Let it run.

### Phase 2: Daily review script (Week 2-3)
With real trade data flowing:
- SSH data puller (Mac → Windows)
- Health checker
- Single Claude API call for analysis (start simple)
- Telegram bot for report delivery
- context.json state file

No approval workflow yet — just receive reports and observe.

### Phase 3: Approval workflow + auto-apply (Week 3-4)
- Strategy advisor prompt
- Backtest proposed changes
- Telegram approval listener
- Config change applier + git push
- Change log tracking

### Phase 4: Agent Teams for structural work (Week 4+)
- Set up .claude/agents/ definitions
- Create /build-strategy command
- Start using Agent Teams for new strategies and refactors
- Feed structural suggestions from daily review into build sessions

### Phase 5: Go live (When ready)
- Switch from paper to live trading with minimal position sizes
- Tighten risk manager limits
- Monitor daily for 2+ weeks before increasing size
- Add the pattern scanner prompt for new strategy ideas
