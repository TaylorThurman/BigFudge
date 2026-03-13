# Trading Bot Agentic System — Architecture Blueprint

*Version 0.2 — March 7, 2026*
*Updated: Fully autonomous pipeline with two-gate approval for code changes*

---

## System Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                      YOU (Product Owner)                          │
│            Telegram on your phone — that's it                     │
│                                                                   │
│  Gate 1: Approve/reject/modify CONCEPTS (daily report)           │
│  Gate 2: Approve/reject DEPLOYMENTS (code changes only)          │
│  Escape hatch: SSH into Mac for manual intervention              │
└──────────────┬───────────────────────────────────────────────────┘
               │
         Telegram messages
               │
┌──────────────▼───────────────────────────────────────────────────┐
│                 MAC (M1 Pro — always on, lid closed)              │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │               Daily Review Pipeline (cron)                  │  │
│  │                                                             │  │
│  │  ANALYSIS (Claude API)          EXECUTION                   │  │
│  │  ┌───────────────────┐          ┌────────────────────────┐ │  │
│  │  │ Pull trade data   │          │ CONFIG changes:        │ │  │
│  │  │ Health check      │          │  Python edits YAML     │ │  │
│  │  │ Trade analysis    │  Gate 1  │  Commits to main       │ │  │
│  │  │ Strategy review   │ ──────►  │                        │ │  │
│  │  │ Pattern scan      │          │ CODE changes:          │ │  │
│  │  │ Backtest proposed │          │  claude -p (headless)  │ │  │
│  │  └───────────────────┘          │  Builds on staging/*   │ │  │
│  │                                  │  Runs tests+backtest  │ │  │
│  │                                  │  ──── Gate 2 ────►    │ │  │
│  │                                  │  Merges to main       │ │  │
│  │                                  └────────────────────────┘ │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────────────────────────┐  │
│  │  State Store     │  │  Claude Code (escape hatch)          │  │
│  │  • context.json  │  │  Available via terminal/SSH when     │  │
│  │  • change_log    │  │  you want manual control. Agent      │  │
│  │  • build_specs/  │  │  Teams, slash commands, etc.         │  │
│  └──────────────────┘  └──────────────────────────────────────┘  │
└──────────────┬───────────────────────────────────────────────────┘
               │ git push (main branch)
┌──────────────▼───────────────────────────────────────────────────┐
│                    GITHUB (Private Repo)                          │
│  main        ← config changes + merged code changes              │
│  staging/*   ← code built by headless Claude Code, awaiting Gate 2│
└──────────────┬───────────────────────────────────────────────────┘
               │ auto-pull main + restart
┌──────────────▼───────────────────────────────────────────────────┐
│             WINDOWS DESKTOP (on during market hours)              │
│                                                                   │
│  Das Trader ◄──websocket──► Trading Bot (Python)                 │
│              127.0.0.1       • Executes strategies                │
│                              • Writes trade/health logs           │
│  Git Watcher (PowerShell)    OpenSSH Server                      │
│  Watches main, restarts bot  Mac pulls logs via SCP              │
└──────────────────────────────────────────────────────────────────┘
```

---

## Project Structure

```
trading-bot/                          ← Git repo (shared between all machines)
│
├── bot/                              ← The actual trading bot (runs on Windows)
│   ├── main.py                       ← Entry point, websocket connection to Das Trader
│   ├── config.yaml                   ← Strategy parameters (daily cycle can modify)
│   ├── strategies/
│   │   ├── base.py                   ← Abstract strategy class
│   │   ├── vwap_momentum.py          ← Example strategy
│   │   └── mean_reversion.py         ← Another strategy (built by the system)
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
│   ├── config_executor.py            ← Applies config.yaml changes (direct Python)
│   ├── code_builder.py               ← Triggers Claude Code headless for code changes
│   ├── deploy_gate.py                ← Gate 2: sends test results, waits for deploy approval
│   └── prompts/                      ← System prompts for each "agent" role
│       ├── trade_analyst.md          ← Analyzes today's performance
│       ├── strategy_advisor.md       ← Suggests parameter AND strategy changes
│       ├── pattern_scanner.md        ← Looks for market themes and patterns
│       └── build_spec_writer.md      ← Converts approved suggestions into build specs
│
├── state/                            ← Persistent state (lives on Mac, NOT in git)
│   ├── context.json                  ← Rolling context: last 10 days of summaries
│   ├── change_log.json               ← Every change made, when, why, result
│   ├── strategies.json               ← Active strategies and their parameters
│   ├── performance_baseline.json     ← What "normal" looks like for each strategy
│   └── build_specs/                  ← Specs for pending code builds
│       └── 2026-03-07_mean_reversion.md  ← Example build spec
│
├── tests/                            ← Bot tests
│   ├── test_strategies.py
│   ├── test_risk_manager.py
│   ├── test_das_connector.py
│   └── backtester.py                 ← Run historical data through strategies
│
├── deploy/
│   ├── watcher.ps1                   ← PowerShell: polls git main, restarts bot
│   └── setup_windows.md             ← Instructions for Windows machine setup
│
├── .claude/
│   ├── settings.json                 ← Model preferences, permissions
│   ├── agents/                       ← Subagent definitions (for manual sessions)
│   │   ├── backend-engineer.md
│   │   ├── test-engineer.md
│   │   └── code-reviewer.md
│   └── commands/
│       ├── build-strategy.md         ← /build-strategy slash command
│       └── code-review.md            ← /code-review slash command
│
├── CLAUDE.md                         ← Project context for Claude Code
├── requirements.txt                  ← Python dependencies for bot
├── requirements-review.txt           ← Python dependencies for review pipeline
└── README.md
```

---

## The Two-Gate Pipeline (Core of the System)

### Full Flow

```
16:30 ─── CRON FIRES pipeline.py ──────────────────────────────────

  ┌──────────────────────────────────────────────────────────────┐
  │ PHASE 1: DATA GATHERING                                      │
  │                                                              │
  │  1. SSH into Windows, pull today's trade logs + health logs  │
  │  2. Health check: was bot connected all day? Data complete?  │
  │  3. Load state: context.json, change_log, strategy params   │
  └──────────────┬───────────────────────────────────────────────┘
                 │
  ┌──────────────▼───────────────────────────────────────────────┐
  │ PHASE 2: ANALYSIS (Claude API — not Claude Code)             │
  │                                                              │
  │  4. Trade Analyst — today's performance vs 10-day baseline  │
  │  5. Strategy Advisor — suggest config AND/OR code changes   │
  │  6. Pattern Scanner — spot new opportunities, regime shifts │
  │  7. Backtest — run proposed param changes against today     │
  └──────────────┬───────────────────────────────────────────────┘
                 │
  ┌──────────────▼───────────────────────────────────────────────┐
  │ ═══════════════ GATE 1: CONCEPT APPROVAL ═══════════════════ │
  │                                                              │
  │  8. Send Telegram report:                                   │
  │     📊 P&L, metrics, analysis                               │
  │     📋 Suggestions with type labels:                        │
  │        [CONFIG] Tighten stop loss 1.5% → 1.2%              │
  │        [CODE]   Add mean reversion strategy for SPY         │
  │        [CODE]   Improve exit logic with trailing stops      │
  │     Each with rationale + backtest data where available     │
  │                                                              │
  │  9. Wait for your Telegram reply                            │
  │     → "approve"                                              │
  │     → "approve 1 and 3, skip 2"                             │
  │     → "approve but make stop loss 1.3%"                     │
  │     → "reject, market was unusual today"                    │
  │                                                              │
  │  10. Parse your response (Claude API call)                  │
  └──────────────┬───────────────────────────────────────────────┘
                 │
          ┌──────┴──────┐
          │             │
    ┌─────▼─────┐ ┌────▼──────────────────────────────────────┐
    │  CONFIG   │ │  CODE CHANGES                              │
    │  CHANGES  │ │                                            │
    │           │ │  11. Generate build spec from approved     │
    │  Direct   │ │      suggestion (Claude API call using     │
    │  Python   │ │      build_spec_writer.md prompt)          │
    │  edits    │ │                                            │
    │  config   │ │  12. Create staging branch:                │
    │  .yaml    │ │      git checkout -b staging/2026-03-07    │
    │           │ │                                            │
    │  Commit   │ │  13. Run Claude Code headless:             │
    │  to main  │ │      claude -p "$(cat build_spec.md)"      │
    │           │ │        --dangerously-skip-permissions      │
    │  Push     │ │                                            │
    │           │ │      Claude Code reads CLAUDE.md, sees the │
    │  ✅ Done  │ │      project structure, writes the code,   │
    │           │ │      creates tests, runs them.             │
    │  Send     │ │                                            │
    │  confirm  │ │  14. Automated validation:                 │
    │  via      │ │      • pytest — full test suite            │
    │  Telegram │ │      • backtester — compare new vs old     │
    │           │ │      • git diff — capture all changes      │
    └───────────┘ │                                            │
                  │  IF TESTS FAIL:                             │
                  │    Retry once with error context            │
                  │    If still fails → notify you:             │
                  │    "❌ Build failed. Tests: 43/47.          │
                  │     Errors: [summary]. Needs intervention." │
                  │    Branch preserved for manual fix.         │
                  │                                            │
                  │  IF TESTS PASS:                             │
                  │                                            │
                  │ ┌──────────────────────────────────────┐   │
                  │ │ ══════ GATE 2: DEPLOY APPROVAL ══════│   │
                  │ │                                      │   │
                  │ │ 15. Send Telegram:                   │   │
                  │ │   "🔨 Code change built & tested.    │   │
                  │ │    Strategy: Mean Reversion SPY      │   │
                  │ │    Tests: 52/52 passing ✅            │   │
                  │ │    Backtest: +8.3% vs current        │   │
                  │ │    Files changed: 3 new, 1 modified  │   │
                  │ │    Diff: github.com/you/repo/compare │   │
                  │ │                                      │   │
                  │ │    Reply DEPLOY or REJECT"           │   │
                  │ │                                      │   │
                  │ │ 16. Wait for your reply              │   │
                  │ └───────────┬──────────────────────────┘   │
                  │        ┌────┴────┐                          │
                  │        ▼         ▼                          │
                  │   ┌─────────┐ ┌──────────┐                 │
                  │   │ DEPLOY  │ │ REJECT   │                 │
                  │   │         │ │          │                 │
                  │   │ Merge   │ │ Delete   │                 │
                  │   │ staging │ │ staging  │                 │
                  │   │ → main  │ │ branch   │                 │
                  │   │ Push    │ │ Log why  │                 │
                  │   │         │ │          │                 │
                  │   │ Send ✅ │ │ Send ℹ️  │                 │
                  │   └─────────┘ └──────────┘                 │
                  └────────────────────────────────────────────┘

  17. Update state: context.json, change_log.json
```

---

## The Build Spec (Bridge Between Analysis and Code)

When you approve a code change at Gate 1, the pipeline doesn't just hand Claude Code a vague instruction. It generates a **build spec** — a detailed document that Claude Code headless receives as its prompt.

The build spec is generated by a Claude API call using the `build_spec_writer.md` prompt, which takes the approved suggestion and the full project context and produces a structured spec.

### Example Build Spec (state/build_specs/2026-03-07_mean_reversion.md)

```markdown
# Build Spec: Mean Reversion Strategy for SPY

## Origin
- Generated by daily review pipeline on 2026-03-07
- Pattern scanner identified 3 consecutive range-bound days
  where momentum strategy missed mean-reversion opportunities
- Approved by user at Gate 1 on 2026-03-07 19:15 EST

## Requirements
Create a mean reversion strategy for SPY that:
- Activates when 5-min ATR drops below the 20-period average
  (indicating low volatility / range-bound conditions)
- Enters long on lower Bollinger Band touch (2 std dev, 20 period)
- Enters short on upper Bollinger Band touch
- Exits at the 20-period SMA (the mean)
- Position size: 200 shares (from config.yaml)
- Stop loss: 0.8% (from config.yaml)
- Only trades during regular hours (9:30 AM - 3:45 PM ET)

## Technical Constraints
- Must inherit from strategies.base.Strategy
- Must implement: setup(), on_bar(), on_tick(), teardown()
- Must use indicators from bot/data/indicators.py (add Bollinger
  Bands and ATR if not already present)
- Must log all trades as structured JSON matching existing format
- Must respect risk_manager limits (check before every order)
- Config parameters go in config.yaml under strategies.mean_reversion

## Files to Create
- bot/strategies/mean_reversion.py (new)
- tests/test_mean_reversion.py (new)

## Files to Modify
- bot/data/indicators.py (add Bollinger Bands, ATR if missing)
- config.yaml (add mean_reversion section with default params)
- bot/main.py (register new strategy in strategy loader)

## Files NOT to Touch
- bot/execution/risk_manager.py
- bot/execution/das_connector.py
- Any existing strategy files

## Validation Criteria
- All existing tests must still pass
- New strategy tests must cover: entry signals, exit signals,
  ATR filter activation/deactivation, position sizing, stop loss
- Backtester must run the new strategy against last 5 days of
  trade data and report: sharpe, max drawdown, win rate, P&L
```

### How the Pipeline Generates This

```python
# In code_builder.py

def generate_build_spec(approved_suggestion, context, project_info):
    """
    Takes the approved suggestion from Gate 1 and generates
    a detailed build spec for Claude Code headless.
    """
    response = call_claude_api(
        system_prompt=load_prompt("prompts/build_spec_writer.md"),
        user_message=f"""
        Approved suggestion: {json.dumps(approved_suggestion)}
        Project context: {json.dumps(context)}
        Current file structure: {project_info['file_tree']}
        Existing strategies: {project_info['strategy_list']}
        Base strategy interface: {project_info['base_strategy_code']}
        Config schema: {project_info['config_schema']}
        """
    )
    return response  # Structured markdown build spec
```

### How Claude Code Headless Executes It

```python
# In code_builder.py

def execute_build(build_spec_path, repo_path):
    """
    Runs Claude Code in headless mode to implement the build spec.
    Returns success/failure + test results.
    """
    branch_name = f"staging/{date.today().isoformat()}"

    # Create staging branch
    subprocess.run(["git", "checkout", "-b", branch_name], cwd=repo_path)

    # Run Claude Code headless with the build spec as the prompt
    result = subprocess.run(
        [
            "claude", "-p",
            f"Read the build spec below and implement it fully. "
            f"Write all code, create all tests, then run pytest "
            f"and the backtester. Report results.\n\n"
            f"{open(build_spec_path).read()}",
            "--dangerously-skip-permissions",
            "--output-format", "json",
            "--model", "sonnet"  # Sonnet for code gen (cost effective)
        ],
        cwd=repo_path,
        capture_output=True,
        timeout=600  # 10 min max
    )

    return parse_build_result(result)


def validate_build(repo_path):
    """
    Run automated validation independent of what Claude Code reported.
    Don't trust the AI's self-assessment — verify independently.
    """
    # Run full test suite ourselves
    test_result = subprocess.run(
        ["python", "-m", "pytest", "--tb=short", "-q"],
        cwd=repo_path, capture_output=True
    )

    # Run backtester ourselves
    backtest_result = subprocess.run(
        ["python", "-m", "tests.backtester", "--compare-baseline"],
        cwd=repo_path, capture_output=True
    )

    # Generate diff for Gate 2 message
    diff = subprocess.run(
        ["git", "diff", "main", "--stat"],
        cwd=repo_path, capture_output=True
    )

    return {
        "tests_passed": test_result.returncode == 0,
        "test_output": test_result.stdout.decode(),
        "backtest_output": backtest_result.stdout.decode(),
        "diff_summary": diff.stdout.decode(),
        "all_green": test_result.returncode == 0
    }
```

**Key safety detail:** The pipeline runs `pytest` and the backtester *itself* after Claude Code finishes. It doesn't trust Claude Code's self-reported results. Claude might say "all tests pass" when they don't. The pipeline verifies independently.

---

## Failure Handling

### Build Fails (Tests Don't Pass)

```python
def handle_build_failure(build_result, validation, build_spec_path, repo_path):
    """
    If the first build attempt fails, retry once with error context.
    If still fails, notify user and preserve branch for manual fix.
    """
    if not validation["all_green"] and not build_result.get("retried"):
        # Retry once with the error output as context
        retry_result = subprocess.run(
            [
                "claude", "-p",
                f"The previous build had test failures. Fix them.\n\n"
                f"Test output:\n{validation['test_output']}\n\n"
                f"Original spec:\n{open(build_spec_path).read()}",
                "--dangerously-skip-permissions",
                "--output-format", "json",
                "--session-id", build_result["session_id"]  # Continue same session
            ],
            cwd=repo_path, capture_output=True, timeout=600
        )
        retry_validation = validate_build(repo_path)

        if retry_validation["all_green"]:
            return retry_validation  # Fixed on retry

    # Still failing — notify user, preserve branch
    send_telegram(
        f"❌ Code build failed after retry.\n\n"
        f"Strategy: {build_spec['title']}\n"
        f"Tests: {validation['test_output'][:500]}\n\n"
        f"Branch `{branch_name}` preserved for manual review.\n"
        f"SSH into Mac and run Claude Code interactively to fix."
    )
    return None  # Signal failure — don't proceed to Gate 2
```

### Claude API Down

```python
def call_claude_api_with_retry(prompt, system, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = anthropic.messages.create(...)
            return response
        except Exception as e:
            if attempt == max_retries - 1:
                send_telegram(
                    f"⚠️ Daily review failed — Claude API unavailable.\n"
                    f"Error: {str(e)[:200]}\n"
                    f"Raw trade data attached for manual review."
                )
                attach_raw_data_to_telegram()
                raise
            time.sleep(30 * (attempt + 1))  # 30s, 60s, 90s backoff
```

### Windows Machine Unreachable

```python
def pull_trade_data():
    try:
        result = subprocess.run(
            ["scp", f"{WIN_USER}@{WIN_IP}:C:/trading-bot/logs/trades/{today}.json", "./"],
            timeout=30, capture_output=True
        )
        if result.returncode != 0:
            raise ConnectionError(result.stderr.decode())
    except Exception as e:
        send_telegram(
            f"⚠️ Cannot reach Windows machine.\n"
            f"Error: {str(e)[:200]}\n"
            f"Is the desktop on? Is Das Trader running?\n"
            f"Skipping today's review."
        )
        raise
```

---

## Telegram Message Examples

### Gate 1 Report

```
📊 Daily Trading Report — March 7, 2026

💰 P&L: +$342.50 (12 trades, 66.7% win rate)
📈 10-Day: +$1,847.20 (Sharpe: 1.84)
⚡ Bot uptime: 100%

📝 Analysis:
Strong morning session driven by tech rally. VWAP momentum
captured NVDA move well (+$185). Three losing trades clustered
in final hour as momentum faded — classic end-of-day mean
reversion that our momentum strategy can't capture.

Change impact (from 03-06): Position size 500→400 has reduced
P&L variance by ~18% with minimal return impact. Keeping.

📋 Suggestions:

1️⃣ [CONFIG] Tighten afternoon stop loss: 1.5% → 1.2%
   Rationale: 3 of last 5 afternoon losses exceeded 1.2%
   Backtest: would have saved $47 today, $112 over 5 days
   Risk: may exit some winners early in volatile afternoons

2️⃣ [CODE] New mean reversion strategy for SPY
   Rationale: 3 consecutive range-bound days with missed
   opportunities. Bollinger Band + ATR filter approach.
   Expected: captures $50-150/day on range-bound days
   Risk: needs to be disabled on trending days

3️⃣ [CONFIG] Reduce max daily loss limit: $1000 → $800
   Rationale: current drawdown patterns suggest tighter
   limit would have prevented 2 worst days this month
   Risk: may force early shutdown on volatile-but-profitable days

Reply with what to approve, e.g.:
  "approve all"
  "approve 1 and 3, skip 2"
  "approve 1 but make it 1.3% instead"
  "reject — market was unusual"
```

### Gate 2 Deploy Request (only for code changes)

```
🔨 Build Complete — Mean Reversion Strategy

✅ Tests: 52/52 passing
📈 Backtest (last 5 days):
   • P&L: +$487 (vs $0 without strategy)
   • Sharpe: 1.42
   • Win rate: 58%
   • Max drawdown: -$127
   • Would not have conflicted with momentum strategy

📁 Changes:
   + bot/strategies/mean_reversion.py (new, 142 lines)
   + tests/test_mean_reversion.py (new, 203 lines)
   ~ bot/data/indicators.py (+28 lines: Bollinger, ATR)
   ~ config.yaml (+8 lines: mean_reversion params)
   ~ bot/main.py (+3 lines: strategy registration)

🔗 View diff: github.com/you/trading-bot/compare/main...staging/2026-03-07

Reply DEPLOY or REJECT
```

### Confirmations

```
✅ Applied 2 changes:

1. config.yaml: stop_loss_pct 1.5% → 1.2%
2. Mean reversion strategy deployed to main

Bot will restart with new code at next market open.
Will monitor both changes for 3 days before suggesting further adjustments.
```

```
❌ Build failed after retry.

Strategy: Mean Reversion SPY
Tests failing: test_mean_reversion::test_atr_filter_deactivation
Error: ATR calculation returns NaN when lookback > available bars

Branch staging/2026-03-07 preserved.
SSH into Mac to fix manually, or wait for tomorrow's cycle to retry.
```

---

## CLAUDE.md (Project Context for Headless + Interactive)

```markdown
# Trading Bot System

## Architecture
Python trading bot connecting to Das Trader Pro via websocket on
localhost (127.0.0.1). Bot runs on Windows. Development, review,
and automated builds happen on macOS.

## How This Repo Is Used
- **Daily automated pipeline** runs on Mac via cron. It analyzes
  trades, suggests changes, and (with user approval) modifies
  config.yaml directly or triggers Claude Code headless to write
  code on a staging branch.
- **Manual interactive sessions** happen occasionally when the
  user wants to directly oversee complex work or fix failed builds.

## Key Constraints
- Das Trader connection is websocket-only, localhost only, Windows only
- Bot must handle websocket disconnections with auto-reconnect
- All trade logs must be structured JSON for automated analysis
- risk_manager.py is safety-critical — extra care on any modifications
- New strategies must inherit from strategies.base.Strategy
- All code changes must pass the full test suite + backtester before merge

## Strategy Interface (all strategies must implement)
- setup() — initialize indicators, state
- on_bar(bar) — called on each new price bar
- on_tick(tick) — called on each tick (optional)
- teardown() — cleanup at end of day

## Config Structure (config.yaml)
Tunable parameters per strategy:
- enabled, position_size, stop_loss_pct, take_profit_pct
- entry_threshold, lookback_period
- Any strategy-specific params

Global risk params:
- max_daily_loss, max_position_count

## Conventions
- Python 3.11+, type hints on all functions
- Docstrings explaining trading logic, not just code
- Trade logs: JSON with timestamp, symbol, side, qty, price,
  strategy, pnl, metadata
- Health heartbeat every 60 seconds to logs/health/
- Tests required for all strategy logic

## When Running Headless (via pipeline)
You are being called by an automated build pipeline. A build spec
will be provided as your prompt. Follow it precisely:
1. Read the spec completely before writing any code
2. Implement exactly what is specified — no more, no less
3. Create all tests specified
4. Run pytest and report results
5. Run the backtester if specified and report results
6. Do not modify files listed under "Files NOT to Touch"
```

---

## Cron Schedule (Mac M1 Pro)

```crontab
# Daily review pipeline — 30 min after market close
30 16 * * 1-5  cd ~/trading-bot && python -m review.pipeline >> logs/pipeline.log 2>&1

# Morning health check — verify bot started and is connected
35 9 * * 1-5   cd ~/trading-bot && python -m review.health_checker --morning >> logs/health.log 2>&1

# Weekend summary — broader analysis
0 10 * * 6     cd ~/trading-bot && python -m review.pipeline --weekly >> logs/pipeline.log 2>&1
```

---

## Walk-Through: A Full Day

```
09:30  Market opens. Bot on Windows connects to Das Trader,
       starts executing VWAP momentum strategy.
       
09:35  Morning health check fires on Mac. Confirms bot is
       connected and trading. No notification (all good).

12:00  Bot has executed 8 trades so far. Logs written to
       C:\trading-bot\logs\trades\2026-03-07.json

16:00  Market closes. Bot logs final summary, enters idle.

16:30  Cron fires pipeline.py on Mac.

16:31  Mac SSHs into Windows, pulls trade logs + health data.

16:32  Health check: bot was connected 09:29–16:01. ✅

16:33  Loads context.json (last 10 days of rolling history).

16:34  Claude API call #1 (trade analyst):
       "Today: +$342, 12 trades, 67% win rate. Strong AM,
       weak PM. Position size change from 03-06 working well."

16:35  Claude API call #2 (strategy advisor):
       Suggests 3 changes: 1 config, 1 code, 1 config.

16:36  Claude API call #3 (pattern scanner):
       "Range-bound conditions persisting. Mean reversion
       opportunity identified."

16:37  Backtest: proposed stop loss change would have saved $47.

16:38  Telegram report sent to your phone. Pipeline enters
       approval listener mode.

        ... you're at the gym, you'll check later ...

18:45  You read the report on your phone. Reply:
       "approve 1 and 2, skip 3"

18:45  Telegram bot picks up reply. Claude API parses it:
       {approved: [1, 2], rejected: [3], modifications: []}

18:46  CONFIG CHANGE (suggestion 1):
       Python script sets stop_loss_pct = 1.2 in config.yaml.
       Git commit, push to main.
       Telegram: "✅ Config applied: stop_loss_pct 1.5→1.2"

18:47  CODE CHANGE (suggestion 2):
       Pipeline generates build spec from the approved suggestion.
       Creates staging/2026-03-07 branch.
       Runs: claude -p "$(cat build_spec.md)" --dangerously-skip-permissions

18:52  Claude Code finishes writing mean_reversion.py, tests, etc.
       Pipeline runs pytest independently: 52/52 passing.
       Pipeline runs backtester: +$487 over 5 days simulated.

18:53  Gate 2 Telegram: "🔨 Build complete. Tests: 52/52 ✅.
       Backtest: +$487. Diff: [link]. Reply DEPLOY or REJECT."

        ... you check the diff on GitHub from your phone ...

19:10  You reply: "deploy"

19:10  Pipeline merges staging → main. Pushes.
       Telegram: "✅ Mean reversion strategy deployed.
       Will be active at next market open."

19:11  Updates context.json and change_log.json.

19:12  Pipeline finishes. Exits cleanly.

       (Windows watcher won't pull until tomorrow morning
        — bot is idle anyway, market is closed.)

09:29  Next morning. Windows watcher detects new commits.
       Pulls latest. Restarts bot. Now running both VWAP
       momentum AND mean reversion strategies.

09:35  Morning health check confirms both strategies loaded. ✅
```

---

## Build Order

### Phase 1: Get a bot trading on paper (Week 1-2)
Use Claude Code (single agent, no teams) to build:
- Das Trader websocket connector
- One simple strategy
- Structured JSON trade logging
- Health heartbeat writer
- config.yaml with tunable parameters
- Basic tests and backtester framework

Deploy to Windows. Connect to Das Trader paper trading.

### Phase 2: Daily review — reports only (Week 2-3)
With real trade data flowing:
- SSH data puller (Mac → Windows)
- Health checker
- Claude API trade analysis
- Telegram bot for report delivery
- context.json state management

No approval workflow yet. Just receive and observe reports.

### Phase 3: Gate 1 — config changes (Week 3-4)
- Strategy advisor prompt
- Backtest proposed changes
- Telegram approval listener
- Config change executor + git push
- Change log tracking

### Phase 4: Gate 2 — code changes (Week 4-5)
- Build spec writer prompt
- Claude Code headless integration (code_builder.py)
- Independent test/backtest validation
- Gate 2 Telegram flow (deploy_gate.py)
- Staging branch management
- Failure handling + retry logic

### Phase 5: Polish and go live (Week 5+)
- Pattern scanner prompt
- Weekend summary reports
- Switch from paper to live with minimal position sizes
- Tighten risk limits
- Monitor daily for 2+ weeks before scaling up
- Set up .claude/agents/ for manual escape hatch sessions

### Future: Optional enhancements
- Run a local model on Mac for cheaper analysis calls
- Add market data feeds for pre-market analysis
- Multi-strategy portfolio optimization
- Slack/Discord integration alongside Telegram
- Dashboard web UI for historical performance visualization
