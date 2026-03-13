# Architecture Blueprint v5 — Autonomous Trading Bot System

**Updated: March 8, 2026**
**Changes from v4:** Telegram → Discord, added Dashboard for gate approvals, added Audit Trail system

---

## System Overview

A hybrid autonomous trading pipeline using Claude Code headless mode + Python orchestration. You operate as product owner from your phone (or any browser) — no terminal unless intervention needed. The system analyzes trades daily, suggests config and code changes, and routes everything through a two-gate human approval system via a **Discord-integrated web dashboard**.

---

## Hardware & Network

| Machine | Role | Details |
|---------|------|---------|
| MacBook Pro M1 16GB | Always-on server (lid closed) | Runs daily review pipeline, cron jobs, dashboard server |
| MacBook Air M2 16GB | Daily driver | Development, manual intervention only |
| Windows Desktop | Trading execution | Das Trader Pro + trading bot, PowerShell git watcher |

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
│               │  Gate 1 & Gate 2 approvals live here      │         │
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
                    └─────────────────────────────┘
```

---

## Why Discord Over Telegram

- **Richer embeds** — structured fields, colors, inline buttons that link directly to dashboard actions
- **Threads** — each pipeline run can get its own thread for clean conversation history
- **Webhooks are simpler** — no bot polling needed for send-only notifications; webhook URL is one line of config
- **Bot interactions** — Discord bot can also expose slash commands for quick status checks (`/status`, `/last-run`, `/audit <date>`)
- **Channel organization** — separate channels for `#gate-1-config`, `#gate-2-code`, `#pipeline-runs`, `#alerts`
- **Mobile notifications** — Discord mobile app works well for push alerts, just like Telegram did

---

## Discord Channel Structure

```
📁 trading-bot (server)
├── #pipeline-runs      — Daily run summaries, health checks
├── #gate-1-config      — Config change proposals (links to dashboard)
├── #gate-2-code        — Code change proposals (links to dashboard)
├── #alerts             — Failures, anomalies, circuit breakers
├── #audit-log          — Daily digest of all approved/rejected changes
└── #general            — Manual commands, ad hoc discussion
```

---

## Dashboard

The dashboard is the **single place** for all gate interactions. Discord is notification-only — it sends you a ping with context and a link. You tap the link, land on the dashboard, and take action there.

### Tech Stack
- **FastAPI** backend (Python, runs on Mac M1)
- **Simple HTML/JS frontend** (no heavy framework needed — this is a personal tool)
- **Exposed via Cloudflare Tunnel** so you can access it from your phone anywhere
- **Auth** — simple token-based auth or Cloudflare Access (zero trust) since it's single-user

### Dashboard Pages

#### Home / Status
- Current bot status (running, stopped, error)
- Last pipeline run summary
- Today's P&L snapshot
- Next scheduled run countdown
- Quick links to recent pending approvals

#### Gate 1 — Config Approvals
- List of pending config change proposals
- Each proposal shows:
  - **What changed**: side-by-side diff of config.yaml (old vs proposed)
  - **Why**: Claude's reasoning summary from the analysis
  - **Supporting data**: relevant trade stats, backtest results
  - **Actions**: Approve / Reject / Suggest Modification
- Suggest Modification opens a text field — your feedback gets piped back into the next analysis cycle and logged to the audit trail
- History tab showing past Gate 1 decisions

#### Gate 2 — Code Approvals
- List of pending code change proposals (staging branches)
- Each proposal shows:
  - **Build spec**: the markdown doc that drove the code generation
  - **Diff view**: full code diff from staging branch vs main
  - **Test results**: pytest output, backtest comparison (before/after)
  - **Actions**: Approve & Merge / Reject / Request Changes
- Request Changes sends feedback to the audit trail and optionally re-triggers Claude Code headless with your notes
- History tab showing past Gate 2 decisions

#### Audit Timeline
- Searchable, filterable timeline of every event (see Audit Trail section below)
- Filter by: date range, event type, gate, outcome
- Each entry is expandable to show full context
- **"Why is this broken?" mode** — select a config param or code file and see every change that touched it, in order

#### Settings
- Discord webhook URL
- Pipeline schedule
- 3-day rule toggle
- Circuit breaker thresholds
- Manual pipeline trigger button

---

## Two-Gate Approval System

### Gate 1 — Config Changes
**Trigger:** Daily pipeline analysis recommends config.yaml changes (position_size, stop_loss_pct, etc.)

**Flow:**
1. Pipeline generates proposed config diff + reasoning
2. Backtest runs against proposed config
3. Event logged to audit trail: `GATE1_PROPOSED`
4. Dashboard creates pending approval card
5. Discord notification sent to `#gate-1-config` with embed + deep link to dashboard
6. You open dashboard, review diff + backtest results
7. You choose: Approve / Reject / Suggest
8. Outcome logged to audit trail: `GATE1_APPROVED`, `GATE1_REJECTED`, or `GATE1_MODIFIED`
9. If approved → Python edits config.yaml, commits to main
10. If modified → your suggestion stored, fed into next pipeline run

### Gate 2 — Code Changes
**Trigger:** Daily pipeline analysis recommends code changes (strategy logic, new indicators, etc.)

**Flow:**
1. Claude API generates a build spec (detailed markdown)
2. Claude Code headless (`claude -p`) executes build spec on a staging branch
3. Independent validation: pytest + backtester run on staging branch (NOT by Claude Code)
4. Event logged to audit trail: `GATE2_PROPOSED`
5. Dashboard creates pending approval card with diff, test results, build spec
6. Discord notification sent to `#gate-2-code` with embed + deep link
7. You open dashboard, review everything
8. You choose: Approve & Merge / Reject / Request Changes
9. Outcome logged to audit trail: `GATE2_APPROVED`, `GATE2_REJECTED`, or `GATE2_REVISION_REQUESTED`
10. If approved → staging branch merged to main
11. If revision requested → feedback stored, optionally re-triggers code gen

### 3-Day Rule
No config or code changes within 3 days of the last approved change. The pipeline enforces this — if the rule is active, it skips change proposals and reports observation-only analysis. The dashboard shows a countdown timer when the rule is in effect.

---

## Audit Trail System

This is the answer to "why is this broken right now?" The audit trail is an append-only log that records every meaningful event in the system.

### Storage
- **Primary:** `audit/YYYY-MM-DD.jsonl` — one JSON object per line, append-only, daily files
- **Backup:** Git commits themselves serve as a secondary audit trail for config/code changes
- **Retention:** Keep indefinitely (they're small text files)

### Event Schema

```json
{
  "id": "evt_20260308_163042_a1b2c3",
  "timestamp": "2026-03-08T16:30:42Z",
  "event_type": "GATE1_PROPOSED",
  "category": "gate_1",
  "pipeline_run_id": "run_20260308_1630",
  "data": {
    "changes": {
      "position_size": { "old": 100, "new": 150 },
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
| `TRADE_DATA_PULLED` | data | Windows trade logs pulled via SSH |
| `ANALYSIS_COMPLETED` | analysis | Claude API analysis finished |
| `BACKTEST_RUN` | backtest | Backtest executed with results |
| `GATE1_PROPOSED` | gate_1 | Config change proposed |
| `GATE1_APPROVED` | gate_1 | Config change approved |
| `GATE1_REJECTED` | gate_1 | Config change rejected |
| `GATE1_MODIFIED` | gate_1 | Config change sent back with suggestions |
| `GATE2_PROPOSED` | gate_2 | Code change proposed |
| `GATE2_APPROVED` | gate_2 | Code change approved and merged |
| `GATE2_REJECTED` | gate_2 | Code change rejected |
| `GATE2_REVISION_REQUESTED` | gate_2 | Code change sent back for revision |
| `CONFIG_APPLIED` | deploy | Config.yaml updated on main |
| `CODE_MERGED` | deploy | Staging branch merged to main |
| `BOT_RESTARTED` | deploy | Windows bot detected change and restarted |
| `THREE_DAY_RULE_ACTIVE` | policy | Change proposals skipped due to 3-day rule |
| `CIRCUIT_BREAKER_TRIPPED` | safety | Anomaly detected, pipeline halted |

### Querying the Audit Trail

The dashboard Audit Timeline page surfaces this visually, but the raw data is also queryable from code:

```python
# Trace every change to a specific config param
def trace_param(param_name, audit_dir="audit/"):
    events = []
    for f in sorted(Path(audit_dir).glob("*.jsonl")):
        for line in f.read_text().splitlines():
            evt = json.loads(line)
            if evt.get("data", {}).get("changes", {}).get(param_name):
                events.append(evt)
            if evt.get("data", {}).get("changes_applied", {}).get(param_name):
                events.append(evt)
    return events

# "Why is stop_loss_pct set to 0.015?"
history = trace_param("stop_loss_pct")
# Returns every proposal + outcome that touched this param, in order
```

```python
# Find all code changes in the last 30 days
def code_change_history(days=30):
    cutoff = datetime.now() - timedelta(days=days)
    events = []
    for f in sorted(Path("audit/").glob("*.jsonl")):
        for line in f.read_text().splitlines():
            evt = json.loads(line)
            if (evt["category"] == "gate_2" and
                datetime.fromisoformat(evt["timestamp"].rstrip("Z")) > cutoff):
                events.append(evt)
    return events
```

### Audit Trail in the Dashboard ("Why Is This Broken?" Mode)

The Audit Timeline page in the dashboard has a special investigation mode. You pick a config parameter or a code file from a dropdown, and it shows you a filtered, chronological view of every event that touched it — proposals, approvals, rejections, and the human notes attached to each decision. This lets you quickly trace back from a current problem to the decision chain that caused it.

For example: "stop_loss_pct is too tight and we're getting stopped out too early" → open the audit timeline → filter by `stop_loss_pct` → see that on March 5 you approved changing it from 0.02 to 0.015 based on a backtest that showed improved Sharpe → now you can decide whether to revert or adjust.

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
   - SCP from Windows: trade logs (JSON) for today
   → TRADE_DATA_PULLED logged

4. Claude API Analysis (3 prompts)
   a) Trade Analyst — reviews today's trades against strategy rules
   b) Strategy Advisor — suggests config/code improvements
   c) Pattern Scanner — looks for recurring patterns in 10-day window
   → ANALYSIS_COMPLETED logged

5. Backtest proposed changes
   → BACKTEST_RUN logged

6. Check 3-Day Rule
   - If active → log THREE_DAY_RULE_ACTIVE, skip to step 9
   - If clear → proceed

7. Gate 1: Config Changes (if any proposed)
   → GATE1_PROPOSED logged
   → Dashboard card created
   → Discord notification sent to #gate-1-config
   → Await human action on dashboard

8. Gate 2: Code Changes (if any proposed)
   - Claude API generates build spec
   - Claude Code headless writes code on staging branch
   - Independent pytest + backtest validation
   → GATE2_PROPOSED logged
   → Dashboard card created
   → Discord notification sent to #gate-2-code
   → Await human action on dashboard

9. Summary Report
   → Discord #pipeline-runs gets daily summary embed
   → PIPELINE_COMPLETED logged
```

---

## Discord Message Formats

### Pipeline Run Summary (`#pipeline-runs`)
```
🤖 Daily Pipeline — March 8, 2026

📊 Today's Trades: 12 executed, 8 winners (66.7%)
💰 P&L: +$342.50
📈 10-Day Sharpe: 1.38

🔧 Config Changes: 1 proposed → [Review on Dashboard]
💻 Code Changes: 0 proposed
⏱️ 3-Day Rule: Clear (last change: March 4)

🔗 Full Report: https://your-dashboard.example.com/runs/20260308
```

### Gate 1 Notification (`#gate-1-config`)
```
⚙️ Config Change Proposed

Parameter Changes:
  stop_loss_pct: 0.02 → 0.015

Reasoning: Tighter stops would have reduced losses by $180
           over the last 5 sessions based on backtest.

Backtest: Sharpe 1.2 → 1.4 | Win Rate 58% → 61%

👉 Review & Approve: https://your-dashboard.example.com/gate1/evt_20260308_...
```

### Gate 2 Notification (`#gate-2-code`)
```
💻 Code Change Proposed

Branch: staging/improve-momentum-filter
Files Changed: 2 (strategy/momentum.py, tests/test_momentum.py)
Tests: 14/14 passing
Backtest: +8.3% improvement in simulated returns

Build Spec: Improve momentum entry filter using RSI confirmation

👉 Review & Approve: https://your-dashboard.example.com/gate2/evt_20260308_...
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
- Bot writes structured JSON trade logs per day to a known directory
- Strategies inherit from `base.Strategy` class with `setup/on_bar/on_tick/teardown` interface
- `config.yaml` holds tunable params (position_size, stop_loss_pct, etc.)
- **PowerShell git watcher** polls main branch every 60s, auto-pulls and restarts bot on changes
- **OpenSSH on Windows** allows Mac M1 to pull trade logs via SCP

---

## State Management

### context.json (10-Day Rolling Window)
- Lives in the repo, updated each pipeline run
- Contains: last 10 days of trade summaries, current config snapshot, pending proposals, 3-day rule state
- Fed into every Claude API prompt for continuity

### Audit Log (Append-Only)
- `audit/YYYY-MM-DD.jsonl` — daily files
- Never modified, only appended
- Source of truth for "what happened and why"

### config.yaml (Bot Configuration)
- Tunable params only — no secrets
- Every change is a git commit with a message referencing the audit event ID
- Example commit: `config: update stop_loss_pct 0.02→0.015 [evt_20260308_163042_a1b2c3]`

---

## Build Phases

| Phase | What | Gate System |
|-------|------|-------------|
| 1 — Bot on Paper | Bot runs but doesn't execute real trades, logs what it would do | None |
| 2 — Reports Only | Pipeline runs analysis, sends reports to Discord, no changes proposed | None |
| 3 — Gate 1 Config | Pipeline can propose config changes, requires dashboard approval | Gate 1 |
| 4 — Gate 2 Code | Pipeline can propose code changes via Claude Code headless | Gate 1 + Gate 2 |
| 5 — Go Live | Full autonomy with human-in-the-loop approvals | Gate 1 + Gate 2 |

---

## Build Order (Implementation)

1. **Foundation** — repo structure, config.yaml schema, base.Strategy class, trade log format
2. **Pipeline skeleton** — cron, SSH data pull, health checks, context.json management
3. **Analysis layer** — Claude API prompts (trade analyst, strategy advisor, pattern scanner)
4. **Audit trail** — event logging, JSONL writer, query helpers
5. **Discord integration** — webhook notifications, bot for slash commands, channel setup
6. **Dashboard** — FastAPI + HTML/JS, Gate 1 page, Gate 2 page, Audit Timeline
7. **Cloudflare Tunnel** — expose dashboard securely for mobile access
8. **Gate 1 flow** — end-to-end config proposal → dashboard review → apply → git commit
9. **Gate 2 flow** — build spec → Claude Code headless → staging branch → test → dashboard review → merge
10. **Windows git watcher** — PowerShell auto-pull + restart on main branch changes

---

## CLAUDE.md (For Claude Code Headless)

```markdown
# CLAUDE.md — Trading Bot

## What This Repo Is
Autonomous trading bot with human-in-the-loop approval pipeline.

## Rules
- NEVER modify config.yaml directly — config changes go through Gate 1
- NEVER push to main — always work on staging/* branches
- NEVER modify audit trail files
- All code changes must have corresponding tests
- Follow existing patterns in base.Strategy class
- Keep strategies stateless between sessions
- Trade logs are read-only input — never modify them

## Structure
- strategies/ — strategy implementations inheriting base.Strategy
- pipeline/ — daily analysis pipeline code
- dashboard/ — FastAPI dashboard app
- audit/ — append-only event logs (DO NOT TOUCH)
- config.yaml — bot configuration (DO NOT MODIFY DIRECTLY)
- context.json — rolling state window
- tests/ — pytest test suite

## Testing
- Run: pytest tests/ -v
- Backtest: python -m backtester --config config.yaml --strategy <n>
```

---

## Failure Handling

| Failure | Response | Audit Event |
|---------|----------|-------------|
| SSH to Windows fails | Retry 3x, then alert Discord `#alerts` | `PIPELINE_FAILED` |
| Claude API timeout | Retry with exponential backoff, then skip analysis | `PIPELINE_FAILED` |
| Backtest crashes | Skip change proposal, report observation-only | `BACKTEST_RUN` (with error) |
| Claude Code headless fails | Log error, skip Gate 2, alert Discord | `GATE2_PROPOSED` (with error) |
| Dashboard unreachable | Discord notifications still work as fallback | `CIRCUIT_BREAKER_TRIPPED` |
| Git push fails | Retry 3x, then alert and halt | `PIPELINE_FAILED` |
| Approval timeout (24h) | Auto-reject, log as expired | `GATE1_REJECTED` / `GATE2_REJECTED` |

---

## Security Notes

- Dashboard behind Cloudflare Tunnel + Access (zero trust, your email only)
- Discord bot token stored in `.env`, never committed
- Claude API key stored in `.env`, never committed
- SSH keys for Windows connection stored on Mac M1 only
- No secrets in config.yaml or context.json
- Git repo is private
