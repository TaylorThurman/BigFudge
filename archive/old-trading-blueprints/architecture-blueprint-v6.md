# Architecture Blueprint v6 — Autonomous Trading Bot System

**Updated: March 8, 2026**
**Changes from v5:** Added Gate 3 (shadow/paper validation), strategy lifecycle management, per-strategy independence, enhanced Gate 2 validation suite, shadow trade engine, Strategy Management dashboard page

---

## System Overview

A hybrid autonomous trading pipeline using Claude Code headless mode + Python orchestration. You operate as product owner from your phone (or any browser) — no terminal unless intervention needed. The system analyzes trades daily, suggests config and code changes, and routes everything through a **three-gate** human approval system via a **Discord-integrated web dashboard**.

The three gates:
- **Gate 1** — Config tuning (position size, stop loss, etc.) — approve and apply immediately
- **Gate 2** — Code changes (strategy logic, new strategies) — approve and deploy to shadow mode
- **Gate 3** — Shadow validation (paper trading with real market data) — promote to live or reject

---

## Hardware & Network

| Machine | Role | Details |
|---------|------|---------|
| MacBook Pro M1 16GB | Always-on server (lid closed) | Runs daily review pipeline, cron jobs, dashboard server |
| MacBook Air M2 16GB | Daily driver | Development, manual intervention only |
| Windows Desktop | Trading execution | Das Trader Pro + trading bot + shadow trade engine, PowerShell git watcher |

All machines on the same local network. GitHub private repo is the single source of truth.

---

## Core Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        MAC M1 PRO (Server)                         │
│                                                                     │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────────────┐     │
│  │ Cron     │───>│ Python       │───>│ Claude API            │     │
│  │ 16:30 ET │    │ Orchestrator │    │ (Analysis Prompts)    │     │
│  └──────────┘    └──────┬───────┘    └───────────┬───────────┘     │
│                         │                         │                 │
│                         v                         v                 │
│               ┌─────────────────┐    ┌───────────────────────┐     │
│               │ Backtest Engine │    │ Claude Code Headless  │     │
│               │ (pytest + sim)  │    │ (Code Generation)     │     │
│               └────────┬────────┘    └───────────┬───────────┘     │
│                        │                          │                 │
│                        v                          v                 │
│               ┌──────────────────────────────────────────┐         │
│               │         AUDIT TRAIL (audit_log.jsonl)     │         │
│               │  Every event timestamped & persisted      │         │
│               └──────────────────┬───────────────────────┘         │
│                                  │                                  │
│                                  v                                  │
│               ┌──────────────────────────────────────────┐         │
│               │     DASHBOARD (localhost + Cloudflare)    │         │
│               │  Gate 1, 2, 3 approvals live here         │         │
│               │  Strategy management + audit timeline     │         │
│               │  Discord notifications link back here     │         │
│               └──────────────────┬───────────────────────┘         │
│                                  │                                  │
└──────────────────────────────────┼──────────────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │       DISCORD BOT           │
                    │  Notifications + Deep Links  │
                    │  to Dashboard approval pages  │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │       GITHUB (Private)       │
                    │  main branch = production     │
                    │  staging/* = pending changes   │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │     WINDOWS DESKTOP          │
                    │  PowerShell git watcher       │
                    │  auto-pulls main, restarts    │
                    │  Das Trader ↔ Bot (websocket) │
                    │  Shadow trade engine (paper)  │
                    └─────────────────────────────┘
```

---

## Strategy Lifecycle

Every strategy is an **independent unit** with its own mode, config, and gate history. Strategies never affect each other's flow — the 3-day rule, shadow evaluation, and approvals all operate per strategy.

### Strategy Modes

```
disabled ──→ shadow ──→ live
    ↑           ↑         │
    │           │         │ (significant code change approved)
    │           └─────────┘
    │                     │
    └─────────────────────┘ (manual disable or rejection)
```

- **`disabled`** — strategy code exists but doesn't run at all (`setup()` never called)
- **`shadow`** — strategy runs against live market data, generates signals, but executes phantom trades instead of real orders; results logged for evaluation
- **`live`** — strategy executes real orders through Das Trader

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

A strategy cannot change mode while it has an open position (live) or an unresolved phantom trade (shadow). The dashboard shows the strategy's current position state, and if it's holding, the toggle buttons are grayed out with a message: "Currently holding 200 shares of AAPL — close position before changing mode." Every blocked attempt is logged as `STRATEGY_MODE_BLOCKED` in the audit trail.

---

## Config Structure (Per-Strategy)

config.yaml evolves from a flat parameter list to a per-strategy structure:

```yaml
global:
  max_total_exposure: 10000     # across all live strategies
  max_strategies_live: 3         # max strategies in live mode at once

strategies:
  momentum_v2:
    mode: live                   # live | shadow | disabled
    position_size: 100
    stop_loss_pct: 0.02
    take_profit_pct: 0.04
    max_daily_trades: 5

  rsi_mean_reversion:
    mode: shadow                 # currently being evaluated
    shadow_start: "2026-03-08"
    shadow_days: 5               # evaluate for 5 trading days
    position_size: 75
    stop_loss_pct: 0.015
    take_profit_pct: 0.03
    max_daily_trades: 3

  breakout_v1:
    mode: disabled               # turned off, not running
    position_size: 50
    stop_loss_pct: 0.025
    take_profit_pct: 0.05
    max_daily_trades: 4
```

Gate 1 config changes are **per-strategy** — a proposal to change `momentum_v2.stop_loss_pct` doesn't affect `rsi_mean_reversion` at all. The 3-day rule is also per-strategy: if momentum_v2 had a config change on March 5, rsi_mean_reversion can still receive changes on March 6.

---

## Why Discord Over Telegram

- **Richer embeds** — structured fields, colors, inline buttons that link directly to dashboard actions
- **Threads** — each pipeline run can get its own thread for clean conversation history
- **Webhooks are simpler** — no bot polling needed for send-only notifications; webhook URL is one line of config
- **Bot interactions** — Discord bot can also expose slash commands for quick status checks (`/status`, `/last-run`, `/audit <date>`)
- **Channel organization** — separate channels per gate plus alerts and audit
- **Mobile notifications** — Discord mobile app works well for push alerts

---

## Discord Channel Structure

```
📁 trading-bot (server)
├── #pipeline-runs      — Daily run summaries, health checks
├── #gate-1-config      — Config change proposals (links to dashboard)
├── #gate-2-code        — Code change proposals (links to dashboard)
├── #gate-3-shadow      — Shadow evaluation progress + completion (links to dashboard)
├── #alerts             — Failures, anomalies, circuit breakers
├── #audit-log          — Daily digest of all approved/rejected changes
└── #general            — Manual commands, ad hoc discussion
```

---

## Dashboard

The dashboard is the **single place** for all gate interactions and strategy management. Discord is notification-only — it sends you a ping with context and a link. You tap the link, land on the dashboard, and take action there.

### Tech Stack
- **FastAPI** backend (Python, runs on Mac M1)
- **Simple HTML/JS frontend** (no heavy framework needed — this is a personal tool)
- **Exposed via Cloudflare Tunnel** so you can access it from your phone anywhere
- **Auth** — simple token-based auth or Cloudflare Access (zero trust) since it's single-user

### Dashboard Pages

#### Home / Status
- Current bot status (running, stopped, error)
- Strategy overview table (name, mode, today's P&L per strategy)
- Active shadow evaluations with progress bars
- Last pipeline run summary
- Next scheduled run countdown
- Quick links to recent pending approvals across all gates

#### Gate 1 — Config Approvals
- List of pending config change proposals (per-strategy)
- Each proposal shows:
  - **Strategy name** and current mode
  - **What changed**: side-by-side diff of that strategy's config params (old vs proposed)
  - **Why**: Claude's reasoning summary from the analysis
  - **Supporting data**: relevant trade stats, backtest results for that strategy
  - **Actions**: Approve / Reject / Suggest Modification
- Suggest Modification opens a text field — your feedback gets piped back into the next analysis cycle and logged to the audit trail
- History tab showing past Gate 1 decisions

#### Gate 2 — Code Approvals
- List of pending code change proposals (staging branches)
- Each proposal shows:
  - **Strategy affected** and whether it's new or existing
  - **Change classification**: significant (triggers shadow) vs minor (refactor/logging only)
  - **Build spec**: the markdown doc that drove the code generation
  - **Diff view**: full code diff from staging branch vs main
  - **Validation results** (see Gate 2 Validation Suite below):
    - pytest output (unit + integration + regression)
    - In-sample backtest (same data window the analysis used)
    - Out-of-sample backtest (held-out window the AI didn't see)
    - Metric comparison table: baseline vs in-sample vs out-of-sample
    - Threshold check results with green/yellow/red indicators
  - **Actions**: Approve & Deploy to Shadow / Reject / Request Changes
  - For minor changes: Approve & Keep Current Mode (no shadow needed)
- If the change affects a currently live strategy, a warning displays: "This strategy is currently live. Approving will move it to shadow mode for re-evaluation."
- Request Changes sends feedback to the audit trail and optionally re-triggers Claude Code headless with your notes
- History tab showing past Gate 2 decisions

#### Gate 3 — Shadow Evaluations
- **Active shadow evaluations** at the top — each one with:
  - Strategy name
  - Progress bar (Day 3 of 5)
  - Interim stats: trade count, win rate, P&L so far
  - Link to full phantom trade log
- **Completed evaluations awaiting your decision** — each one with:
  - Strategy name and description
  - Side-by-side comparison: backtest predictions (from Gate 2) vs actual shadow performance
  - Full metrics table with green/yellow/red indicators against thresholds
  - Trade-by-trade phantom log (expandable) so you can see exactly what it did and why
  - **Actions**: Promote to Live / Reject / Extend Shadow (add more days)
- History tab — past Gate 3 decisions

#### Strategy Management
- All strategies in a control table:

```
Strategy              Mode        Status          Actions
─────────────────────────────────────────────────────────────
momentum_v2          🟢 live     No position     [Shadow] [Disable]
rsi_mean_reversion   🟡 shadow   Day 3/5         [Disable] [Extend]
breakout_v1          ⚫ disabled  —              [Shadow]
```

- Clicking an action checks mid-trade guard first
- If blocked: "momentum_v2 is currently holding 200 shares of AAPL — close position before changing mode."
- Every manual toggle gets logged as `STRATEGY_MODE_CHANGED` with your notes
- Per-strategy 3-day rule countdown timers
- Per-strategy performance history (mini chart of daily P&L)

#### Audit Timeline
- Searchable, filterable timeline of every event
- Filter by: date range, event type, gate, outcome, **strategy name**
- Each entry is expandable to show full context
- **"Why is this broken?" mode** — select a strategy + config param or code file and see every change that touched it, in order

#### Settings
- Discord webhook URL
- Pipeline schedule
- Default shadow evaluation period (days)
- Minimum performance thresholds (Sharpe, max drawdown, min trade count)
- Circuit breaker thresholds
- Manual pipeline trigger button

---

## Three-Gate Approval System

### Gate 1 — Config Changes (Per-Strategy)
**Trigger:** Daily pipeline analysis recommends config.yaml changes for a specific strategy (position_size, stop_loss_pct, etc.)

**Flow:**
1. Pipeline generates proposed config diff + reasoning for the specific strategy
2. Backtest runs: current config vs proposed config on same data window for that strategy
3. Event logged to audit trail: `GATE1_PROPOSED`
4. Dashboard creates pending approval card
5. Discord notification sent to `#gate-1-config` with embed + deep link to dashboard
6. You open dashboard, review diff + backtest results
7. You choose: Approve / Reject / Suggest
8. Outcome logged to audit trail: `GATE1_APPROVED`, `GATE1_REJECTED`, or `GATE1_MODIFIED`
9. If approved → Python edits that strategy's config in config.yaml, commits to main
10. If modified → your suggestion stored, fed into next pipeline run
11. Strategy stays in its current mode (config changes don't trigger shadow)

**3-Day Rule (Per-Strategy):** No config changes to the same strategy within 3 days of its last approved change. Other strategies are unaffected.

### Gate 2 — Code Changes (with Validation Suite)
**Trigger:** Daily pipeline analysis recommends code changes (strategy logic, new indicators, new strategy)

**Flow:**
1. Claude API generates a build spec (detailed markdown) that classifies the change:
   - **Significant**: touches entry/exit logic, signal generation, adds/removes indicators, or is a brand new strategy
   - **Minor**: refactor, logging, comments, test-only changes with identical runtime behavior
2. Pipeline captures **baseline metrics** for any affected strategy on a 60-day window (the "before" snapshot)
3. Claude Code headless (`claude -p`) executes build spec on a staging branch
4. **Validation suite runs on staging** (independent of Claude Code):
   a. **pytest** — unit tests + integration tests + regression tests across ALL strategies (not just the changed one)
   b. **In-sample backtest** — run the changed strategy on the same data window the analysis used (e.g., last 10 days)
   c. **Out-of-sample backtest** — run the changed strategy on a held-out window the AI never saw (e.g., 30 days before the analysis window)
   d. **Metric comparison** — baseline vs in-sample vs out-of-sample:
      - Sharpe ratio
      - Win rate
      - Max drawdown
      - Profit factor
      - Average win/loss ratio
      - Total P&L
      - Trade count
   e. **Threshold checks** — auto-flag if any metric fails minimum requirements:
      - Sharpe < 1.0 on out-of-sample → 🔴 red flag
      - Max drawdown > configured threshold → 🔴 red flag
      - Trade count < minimum (too few trades to be statistically meaningful) → 🟡 warning
      - Out-of-sample performance significantly worse than in-sample → 🟡 warning (possible overfit)
5. Event logged to audit trail: `GATE2_PROPOSED` (includes full validation results)
6. Dashboard creates pending approval card with all validation data
7. Discord notification sent to `#gate-2-code` with embed + deep link
8. You open dashboard, review code diff + validation results + threshold indicators
9. You choose:
   - **Approve & Deploy to Shadow** (for significant changes) → strategy enters shadow mode
   - **Approve & Keep Current Mode** (for minor changes) → code merges, strategy mode unchanged
   - **Reject** → staging branch deleted
   - **Request Changes** → feedback stored, optionally re-triggers code gen
10. Outcome logged to audit trail: `GATE2_APPROVED`, `GATE2_REJECTED`, or `GATE2_REVISION_REQUESTED`
11. If approved (significant) → staging merged to main + config.yaml updated to `mode: shadow` for that strategy
12. If approved (minor) → staging merged to main, no mode change
13. If the affected strategy was `live`, the dashboard shows a warning before approval: "This strategy is currently live. Approving this significant change will move it to shadow mode for re-evaluation."

### Gate 3 — Shadow Evaluation (Paper Trading Validation)
**Trigger:** A strategy enters `shadow` mode (via Gate 2 approval of significant change, or manual toggle from dashboard)

**Flow:**
1. Strategy begins running in shadow mode on the Windows bot — processes live market data, generates signals, logs phantom trades instead of real orders
2. Event logged to audit trail: `GATE3_SHADOW_DEPLOYED` (includes shadow_days target, evaluation criteria)
3. Discord notification sent to `#gate-3-shadow`: "Shadow evaluation started for rsi_mean_reversion (5 days)"
4. **Each day during the evaluation period**, the pipeline:
   a. Pulls shadow trade logs from Windows via SSH
   b. Computes interim metrics
   c. Logs `GATE3_PROGRESS` to audit trail
   d. Sends Discord update to `#gate-3-shadow`: "Day 3 of 5: rsi_mean_reversion — 8 phantom trades, +$215, 62% win rate"
   e. **No changes are made to the shadow strategy during evaluation** — it runs untouched to produce clean results
5. When the evaluation period completes (N trading days elapsed):
   a. Pipeline compiles full performance report from all phantom trades
   b. Compares actual shadow performance against:
      - The backtest predictions from the Gate 2 proposal
      - The minimum performance thresholds
   c. Calculates all key metrics: Sharpe, win rate, max drawdown, profit factor, average win/loss, total phantom P&L
   d. Event logged to audit trail: `GATE3_EVALUATION_COMPLETE` (includes full results + comparison)
6. Dashboard creates Gate 3 approval card
7. Discord notification sent to `#gate-3-shadow` with results summary + deep link
8. You open dashboard, review:
   - Backtest predicted Sharpe 1.3 → Shadow actual Sharpe 1.1
   - Win rate, drawdown, profit factor, trade count
   - Trade-by-trade phantom log
   - Threshold check indicators (green/yellow/red)
9. You choose:
   - **Promote to Live** → strategy mode changes from `shadow` to `live`, starts executing real orders
   - **Reject** → strategy mode changes to `disabled`
   - **Extend Shadow** → adds more trading days to the evaluation (e.g., another 3-5 days), evaluation continues
10. Outcome logged to audit trail: `GATE3_PROMOTED_TO_LIVE`, `GATE3_REJECTED`, or `GATE3_EXTENDED`

### How the Three Gates Chain Together (New Strategy Example)

```
Day 0 (Pipeline Run):
  → Claude API suggests new RSI mean reversion strategy
  → Claude Code headless writes code on staging branch
  → Validation suite: pytest ✅, in-sample ✅, out-of-sample ✅, thresholds ✅
  → GATE2_PROPOSED logged
  → Discord #gate-2-code: "New strategy proposed" → Dashboard link

Day 0 (You review Gate 2):
  → Dashboard shows code diff, validation results, metric comparison
  → All thresholds green
  → You approve → "Approve & Deploy to Shadow"
  → GATE2_APPROVED logged
  → Code merged to main
  → config.yaml: rsi_mean_reversion added with mode: shadow, shadow_days: 5
  → Windows bot picks up changes, starts running strategy in shadow mode
  → GATE3_SHADOW_DEPLOYED logged
  → Discord #gate-3-shadow: "Shadow evaluation started (5 days)"

Days 1-4 (Daily pipeline runs):
  → Pipeline pulls shadow trade logs from Windows
  → GATE3_PROGRESS logged each day
  → Discord #gate-3-shadow: "Day 3 of 5: 8 trades, +$215, 62% win rate"

Day 5 (Pipeline run):
  → Shadow period complete — all 5 trading days elapsed
  → Full evaluation compiled: 12 phantom trades, +$380, Sharpe 1.1, 58% win rate
  → GATE3_EVALUATION_COMPLETE logged
  → Dashboard Gate 3 card created
  → Discord #gate-3-shadow: "Evaluation complete" → Dashboard link

Day 5 (You review Gate 3):
  → Dashboard shows: backtest predicted Sharpe 1.3, shadow actual 1.1
  → Win rate 58%, max drawdown 1.8%, profit factor 1.6
  → All thresholds green (Sharpe > 1.0, drawdown within limits, enough trades)
  → You promote to live
  → GATE3_PROMOTED_TO_LIVE logged
  → config.yaml: rsi_mean_reversion mode changes to live
  → Windows bot picks up change, starts executing real orders
```

### How the Three Gates Chain Together (Existing Strategy Change Example)

```
Day 0 (Pipeline Run):
  → Claude API suggests significant change to momentum_v2 (currently live)
  → Claude Code headless writes code on staging branch
  → Validation suite: pytest ✅, in-sample ✅, out-of-sample shows improvement
  → GATE2_PROPOSED logged (flagged: "affects live strategy")
  → Discord #gate-2-code: "Code change proposed for momentum_v2 (LIVE)" → Dashboard link

Day 0 (You review Gate 2):
  → Dashboard shows warning: "momentum_v2 is currently live. Approving this
     significant change will move it to shadow mode for re-evaluation."
  → You check: momentum_v2 has no open position ✅
  → You approve → "Approve & Deploy to Shadow"
  → GATE2_APPROVED logged
  → Code merged to main
  → config.yaml: momentum_v2 mode changes from live to shadow, shadow_days: 5
  → Windows bot picks up changes: stops real execution, starts shadow mode with new code
  → GATE3_SHADOW_DEPLOYED logged

Days 1-5: Shadow evaluation runs (same as new strategy flow)

Day 5 (You review Gate 3):
  → Shadow results look good, improvement confirmed
  → You promote back to live
  → GATE3_PROMOTED_TO_LIVE logged
  → momentum_v2 back in live mode with updated code
```

---

## Shadow Trade Engine (Windows Bot)

The Windows bot needs to handle three strategy modes simultaneously. The key principle: **shadow strategies run the full strategy logic but route order execution to a phantom trade logger instead of Das Trader.**

### How It Works

```
                    ┌─────────────────────────┐
                    │     Market Data Feed     │
                    │  (Das Trader websocket)  │
                    └────────────┬────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │     Strategy Router      │
                    │  reads config.yaml mode  │
                    └─┬──────────┬──────────┬─┘
                      │          │          │
              mode:live   mode:shadow   mode:disabled
                      │          │          │
                      v          v          │
              ┌───────────┐ ┌───────────┐   └→ skip
              │ Real Order│ │ Phantom   │
              │ Executor  │ │ Trade     │
              │ (DAS API) │ │ Logger    │
              └─────┬─────┘ └─────┬─────┘
                    │             │
                    v             v
              trades/         shadow/{strategy}/
              2026-03-08.json 2026-03-08.json
```

The strategy code itself doesn't know whether it's live or shadow — it calls the same `self.submit_order()` method. The bot's execution layer checks the strategy's mode and routes accordingly. This keeps strategy code clean and testable.

### Phantom Trade Tracking

When a shadow strategy generates a signal, the phantom trade logger:
1. Records the signal: ticker, direction, size, entry price at signal time, stop loss, target
2. Creates a phantom position that tracks against live market data
3. On each subsequent tick, checks if the phantom position would have hit its stop or target
4. When the phantom trade resolves, records the exit: price, timestamp, P&L, exit reason
5. Writes everything to the shadow log file for that strategy

### Shadow Trade Log Format

```
shadow/rsi_mean_reversion/2026-03-08.json
```

```json
[
  {
    "id": "shadow_001",
    "strategy": "rsi_mean_reversion",
    "timestamp_entry": "2026-03-08T10:32:15Z",
    "action": "BUY",
    "ticker": "MSFT",
    "shares": 75,
    "entry_price": 412.50,
    "stop_loss": 406.31,
    "take_profit": 420.75,
    "timestamp_exit": "2026-03-08T11:15:42Z",
    "exit_price": 418.20,
    "exit_reason": "take_profit",
    "pnl": 427.50,
    "status": "closed"
  },
  {
    "id": "shadow_002",
    "strategy": "rsi_mean_reversion",
    "timestamp_entry": "2026-03-08T14:05:30Z",
    "action": "SHORT",
    "ticker": "AAPL",
    "shares": 75,
    "entry_price": 198.30,
    "stop_loss": 201.25,
    "take_profit": 194.50,
    "timestamp_exit": null,
    "exit_price": null,
    "exit_reason": null,
    "pnl": null,
    "status": "open"
  }
]
```

Open phantom trades carry over to the next day and continue tracking until they resolve or the market session ends (at which point they're force-closed at the last price, just like a real trade would be based on your strategy's teardown logic).

---

## Audit Trail System

Append-only log recording every meaningful event in the system. This is the answer to "why is this broken right now?"

### Storage
- **Primary:** `audit/YYYY-MM-DD.jsonl` — one JSON object per line, append-only, daily files
- **Backup:** Git commits serve as a secondary audit trail for config/code changes
- **Retention:** Keep indefinitely (small text files)

### Event Schema

```json
{
  "id": "evt_20260308_163042_a1b2c3",
  "timestamp": "2026-03-08T16:30:42Z",
  "event_type": "GATE1_PROPOSED",
  "category": "gate_1",
  "strategy": "momentum_v2",
  "pipeline_run_id": "run_20260308_1630",
  "data": {
    "changes": {
      "stop_loss_pct": { "old": 0.02, "new": 0.015 }
    },
    "reasoning": "Last 5 days show tighter stops would have saved $X...",
    "backtest_result": {
      "sharpe_before": 1.2,
      "sharpe_after": 1.4,
      "win_rate_before": 0.58,
      "win_rate_after": 0.61
    }
  },
  "outcome": null,
  "outcome_timestamp": null,
  "outcome_by": null,
  "notes": null
}
```

When you approve/reject/modify, the outcome gets appended as a new event that references the original:

```json
{
  "id": "evt_20260308_171500_d4e5f6",
  "timestamp": "2026-03-08T17:15:00Z",
  "event_type": "GATE1_APPROVED",
  "category": "gate_1",
  "strategy": "momentum_v2",
  "pipeline_run_id": "run_20260308_1630",
  "references": "evt_20260308_163042_a1b2c3",
  "data": {
    "changes_applied": {
      "stop_loss_pct": { "old": 0.02, "new": 0.015 }
    },
    "git_commit": "abc123f"
  },
  "outcome": "approved",
  "outcome_by": "human",
  "notes": "Looks good, tighter stops make sense given recent volatility"
}
```

### Event Types

| Event Type | Category | Description |
|------------|----------|-------------|
| `PIPELINE_STARTED` | pipeline | Daily cron kicked off |
| `PIPELINE_COMPLETED` | pipeline | Pipeline finished successfully |
| `PIPELINE_FAILED` | pipeline | Pipeline errored out |
| `HEALTH_CHECK_PASS` | health | Pre-run health check passed |
| `HEALTH_CHECK_FAIL` | health | Pre-run health check failed |
| `TRADE_DATA_PULLED` | data | Windows trade logs (live + shadow) pulled via SSH |
| `ANALYSIS_COMPLETED` | analysis | Claude API analysis finished |
| `BACKTEST_RUN` | backtest | Backtest executed with results |
| `GATE1_PROPOSED` | gate_1 | Config change proposed for a strategy |
| `GATE1_APPROVED` | gate_1 | Config change approved |
| `GATE1_REJECTED` | gate_1 | Config change rejected |
| `GATE1_MODIFIED` | gate_1 | Config change sent back with suggestions |
| `GATE2_PROPOSED` | gate_2 | Code change proposed (includes validation results) |
| `GATE2_APPROVED` | gate_2 | Code change approved (significant → shadow, minor → keep mode) |
| `GATE2_REJECTED` | gate_2 | Code change rejected |
| `GATE2_REVISION_REQUESTED` | gate_2 | Code change sent back for revision |
| `GATE3_SHADOW_DEPLOYED` | gate_3 | Strategy entered shadow mode for evaluation |
| `GATE3_PROGRESS` | gate_3 | Daily interim update during shadow evaluation |
| `GATE3_EVALUATION_COMPLETE` | gate_3 | Shadow period finished, results compiled |
| `GATE3_PROMOTED_TO_LIVE` | gate_3 | Strategy promoted from shadow to live |
| `GATE3_REJECTED` | gate_3 | Strategy rejected after shadow evaluation |
| `GATE3_EXTENDED` | gate_3 | Shadow evaluation period extended |
| `CONFIG_APPLIED` | deploy | config.yaml updated on main |
| `CODE_MERGED` | deploy | Staging branch merged to main |
| `BOT_RESTARTED` | deploy | Windows bot detected change and restarted |
| `STRATEGY_MODE_CHANGED` | strategy | Manual mode toggle from dashboard |
| `STRATEGY_MODE_BLOCKED` | strategy | Mode change blocked — strategy mid-trade |
| `THREE_DAY_RULE_ACTIVE` | policy | Config change skipped for a strategy due to 3-day rule |
| `CIRCUIT_BREAKER_TRIPPED` | safety | Anomaly detected, pipeline halted |

### Querying the Audit Trail

The dashboard Audit Timeline page surfaces this visually, but the raw data is also queryable:

```python
# Trace every change to a specific strategy's config param
def trace_param(strategy_name, param_name, audit_dir="audit/"):
    events = []
    for f in sorted(Path(audit_dir).glob("*.jsonl")):
        for line in f.read_text().splitlines():
            evt = json.loads(line)
            if evt.get("strategy") != strategy_name:
                continue
            if evt.get("data", {}).get("changes", {}).get(param_name):
                events.append(evt)
            if evt.get("data", {}).get("changes_applied", {}).get(param_name):
                events.append(evt)
    return events

# "Why is momentum_v2's stop_loss_pct set to 0.015?"
history = trace_param("momentum_v2", "stop_loss_pct")
```

```python
# Full lifecycle history of a strategy
def strategy_history(strategy_name, audit_dir="audit/"):
    events = []
    for f in sorted(Path(audit_dir).glob("*.jsonl")):
        for line in f.read_text().splitlines():
            evt = json.loads(line)
            if evt.get("strategy") == strategy_name:
                events.append(evt)
    return events

# See everything that ever happened to rsi_mean_reversion
history = strategy_history("rsi_mean_reversion")
# Gate 2 proposed → Gate 2 approved → shadow deployed → progress x5 →
# evaluation complete → promoted to live → config change → ...
```

---

## Daily Pipeline Flow

**Cron: 16:30 ET (after market close)**

```
1. PIPELINE_STARTED logged

2. Health Check
   - Mac M1 disk/memory OK?
   - Windows reachable via SSH?
   - GitHub accessible?
   - Das Trader logs exist for today?
   → HEALTH_CHECK_PASS or HEALTH_CHECK_FAIL logged

3. Pull Trade Data
   - SCP from Windows: live trade logs + shadow trade logs for today
   → TRADE_DATA_PULLED logged

4. Shadow Evaluation Check (Gate 3)
   - For each strategy in shadow mode:
     a. Compute interim or final metrics from phantom trades
     b. If evaluation period NOT complete:
        → log GATE3_PROGRESS
        → Discord #gate-3-shadow: interim stats
     c. If evaluation period IS complete:
        → compile full performance report
        → compare against Gate 2 backtest predictions + thresholds
        → log GATE3_EVALUATION_COMPLETE
        → create Gate 3 dashboard card
        → Discord #gate-3-shadow: "Evaluation complete" + link

5. Claude API Analysis (3 prompts, per live strategy)
   a) Trade Analyst — reviews today's trades against strategy rules
   b) Strategy Advisor — suggests config/code improvements
   c) Pattern Scanner — looks for recurring patterns in 10-day window
   → ANALYSIS_COMPLETED logged

6. Backtest proposed changes (per strategy)
   → BACKTEST_RUN logged

7. Check 3-Day Rule (per strategy)
   - Each strategy checked independently
   - If active for a strategy → skip that strategy's proposals
   → THREE_DAY_RULE_ACTIVE logged per affected strategy

8. Gate 1: Config Changes (per strategy, if any proposed and 3-day rule clear)
   → GATE1_PROPOSED logged
   → Dashboard card created
   → Discord #gate-1-config notification
   → Await human action on dashboard

9. Gate 2: Code Changes (if any proposed)
   - Claude API generates build spec with change classification
   - Pipeline captures baseline metrics for affected strategy
   - Claude Code headless writes code on staging branch
   - Validation suite: pytest + in-sample + out-of-sample + thresholds
   → GATE2_PROPOSED logged
   → Dashboard card created with full validation results
   → Discord #gate-2-code notification
   → Await human action on dashboard

10. Summary Report
    → Discord #pipeline-runs gets daily summary embed
    → PIPELINE_COMPLETED logged
```

---

## Discord Message Formats

### Pipeline Run Summary (`#pipeline-runs`)
```
🤖 Daily Pipeline — March 8, 2026

📊 Live Strategies:
  momentum_v2: 8 trades, 6 winners, +$285
  breakout_v1: 4 trades, 2 winners, +$57.50

👻 Shadow Strategies:
  rsi_mean_reversion: Day 3/5 — 3 phantom trades, +$92

💰 Total Live P&L: +$342.50
📈 10-Day Sharpe (momentum_v2): 1.38

🔧 Config Changes: 1 proposed for momentum_v2 → [Review]
💻 Code Changes: 0 proposed
⏱️ 3-Day Rule: momentum_v2 clear | breakout_v1 2 days remaining

🔗 Full Report: https://your-dashboard.example.com/runs/20260308
```

### Gate 1 Notification (`#gate-1-config`)
```
⚙️ Config Change Proposed — momentum_v2

Parameter Changes:
  stop_loss_pct: 0.02 → 0.015

Reasoning: Tighter stops would have reduced losses by $180
           over the last 5 sessions based on backtest.

Backtest: Sharpe 1.2 → 1.4 | Win Rate 58% → 61%

👉 Review & Approve: https://your-dashboard.example.com/gate1/evt_20260308_...
```

### Gate 2 Notification (`#gate-2-code`)
```
💻 Code Change Proposed — rsi_mean_reversion (NEW STRATEGY)

Classification: 🔶 Significant (will deploy to shadow if approved)
Branch: staging/add-rsi-mean-reversion
Files Changed: 3

Validation Results:
  pytest: ✅ 18/18 (0 regressions)
  In-sample:  Sharpe 1.4 | Win 63% | Drawdown 1.2%
  Out-of-sample: Sharpe 1.2 | Win 58% | Drawdown 1.9%
  Thresholds: 🟢 All passing

👉 Review & Approve: https://your-dashboard.example.com/gate2/evt_20260308_...
```

### Gate 3 Progress (`#gate-3-shadow`)
```
👻 Shadow Update — rsi_mean_reversion (Day 3 of 5)

Today: 3 phantom trades, 2 winners, +$92
Cumulative: 8 phantom trades, 5 winners (62%), +$215
Shadow Sharpe so far: 1.05

📊 Dashboard: https://your-dashboard.example.com/gate3/rsi_mean_reversion
```

### Gate 3 Evaluation Complete (`#gate-3-shadow`)
```
✅ Shadow Evaluation Complete — rsi_mean_reversion

Results (5 trading days):
  Phantom Trades: 12
  Win Rate: 58%
  Total Phantom P&L: +$380
  Sharpe: 1.1
  Max Drawdown: 1.8%

vs Backtest Prediction:
  Sharpe: 1.3 predicted → 1.1 actual
  Win Rate: 63% predicted → 58% actual

Thresholds: 🟢 All passing (Sharpe > 1.0, drawdown < 3%)

👉 Promote or Reject: https://your-dashboard.example.com/gate3/evt_20260308_...
```

### Alert (`#alerts`)
```
🚨 Pipeline Failure

Stage: Trade Data Pull
Error: SSH connection to Windows timed out after 30s
Time: 16:31:02 ET

Action Required: Check Windows machine is awake and SSH is running.
🔗 Details: https://your-dashboard.example.com/runs/20260308
```

---

## Trading Bot (Windows)

- Bot connects to Das Trader Pro via websocket on `127.0.0.1` (localhost only, Windows only)
- Bot reads per-strategy config from config.yaml including `mode` field
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

## State Management

### context.json (10-Day Rolling Window)
- Lives in the repo, updated each pipeline run
- Contains: last 10 days of trade summaries (per strategy), current config snapshot, pending proposals, 3-day rule state (per strategy), active shadow evaluations
- Fed into every Claude API prompt for continuity

### Audit Log (Append-Only)
- `audit/YYYY-MM-DD.jsonl` — daily files
- Never modified, only appended
- Source of truth for "what happened and why"
- Every event tagged with `strategy` field for per-strategy filtering

### config.yaml (Bot Configuration)
- Per-strategy structure with `mode` field and tunable params
- No secrets
- Every change is a git commit with a message referencing the audit event ID
- Example commit: `config: momentum_v2 stop_loss_pct 0.02→0.015 [evt_20260308_163042_a1b2c3]`
- Example commit: `config: rsi_mean_reversion mode shadow→live [evt_20260313_171500_d4e5f6]`

---

## Build Phases

| Phase | What | Gate System |
|-------|------|-------------|
| 1 — Bot on Paper | Bot runs all strategies in shadow mode only, logs phantom trades | None |
| 2 — Reports Only | Pipeline runs analysis, sends reports to Discord, no changes proposed | None |
| 3 — Gate 1 Config | Pipeline can propose config changes per strategy, requires dashboard approval | Gate 1 |
| 4 — Gate 2 Code | Pipeline can propose code changes with validation suite | Gate 1 + Gate 2 |
| 5 — Gate 3 Shadow | Code changes deploy to shadow mode, require paper trading validation before going live | Gate 1 + Gate 2 + Gate 3 |
| 6 — Go Live | Full system with all three gates, strategies promoted to live after shadow validation | Gate 1 + Gate 2 + Gate 3 |

---

## Build Order (Implementation)

1. **Foundation** — repo structure, per-strategy config.yaml schema, base.Strategy class, trade log format, shadow trade log format
2. **Pipeline skeleton** — cron, SSH data pull (live + shadow logs), health checks, context.json management
3. **Analysis layer** — Claude API prompts (trade analyst, strategy advisor, pattern scanner) — per-strategy analysis
4. **Audit trail** — event logging with strategy field, JSONL writer, query helpers, strategy history helper
5. **Discord integration** — webhook notifications for all channels (#gate-1, #gate-2, #gate-3, #pipeline-runs, #alerts), bot for slash commands
6. **Dashboard** — FastAPI + HTML/JS: Home/Status, Gate 1, Gate 2, Gate 3, Strategy Management, Audit Timeline, Settings
7. **Cloudflare Tunnel** — expose dashboard securely for mobile access
8. **Gate 1 flow** — per-strategy config proposal → dashboard review → apply → git commit
9. **Gate 2 flow** — build spec with change classification → Claude Code headless → staging branch → validation suite (pytest + in-sample + out-of-sample + thresholds) → dashboard review → merge + auto-shadow for significant changes
10. **Shadow trade engine** — phantom trade logger in Windows bot, strategy router by mode, shadow log writer, phantom position tracker against live market data
11. **Gate 3 flow** — shadow log collection via SSH → daily progress tracking → evaluation completion → dashboard review → promote/reject/extend
12. **Windows git watcher** — PowerShell auto-pull + restart on main branch changes (handles mode transitions)

---

## CLAUDE.md (For Claude Code Headless)

```markdown
# CLAUDE.md — Trading Bot

## What This Repo Is
Autonomous trading bot with human-in-the-loop three-gate approval pipeline.
Strategies are independent units with lifecycle: disabled → shadow → live.

## Rules
- NEVER modify config.yaml directly — config changes go through Gate 1
- NEVER push to main — always work on staging/* branches
- NEVER modify audit trail files
- NEVER modify shadow trade logs
- All code changes must have corresponding tests
- All strategy changes must include regression tests for ALL existing strategies
- Follow existing patterns in base.Strategy class
- Keep strategies stateless between sessions
- Strategy code must not be mode-aware — use self.submit_order() and let the execution layer route
- Trade logs and shadow logs are read-only input — never modify them

## Structure
- strategies/ — strategy implementations inheriting base.Strategy
- pipeline/ — daily analysis pipeline code
- dashboard/ — FastAPI dashboard app
- audit/ — append-only event logs (DO NOT TOUCH)
- trades/ — live trade logs from Windows (READ ONLY)
- shadow/ — phantom trade logs per strategy from Windows (READ ONLY)
- config.yaml — per-strategy bot configuration (DO NOT MODIFY DIRECTLY)
- context.json — rolling state window
- tests/ — pytest test suite (unit + integration + regression)

## Build Spec Classification
When writing a build spec, always classify the change:
- SIGNIFICANT: touches entry/exit logic, signal generation, adds/removes indicators, new strategy
- MINOR: refactor, logging, comments, test-only changes with identical runtime behavior
This classification determines whether Gate 2 approval triggers shadow mode.

## Testing
- Unit tests: pytest tests/unit/ -v
- Integration tests: pytest tests/integration/ -v
- Regression (all strategies): pytest tests/regression/ -v
- Backtest: python -m backtester --config config.yaml --strategy <name>
- Backtest (out-of-sample): python -m backtester --config config.yaml --strategy <name> --window out-of-sample
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

---

## Security Notes

- Dashboard behind Cloudflare Tunnel + Access (zero trust, your email only)
- Discord bot token stored in `.env`, never committed
- Claude API key stored in `.env`, never committed
- SSH keys for Windows connection stored on Mac M1 only
- No secrets in config.yaml or context.json
- Git repo is private
