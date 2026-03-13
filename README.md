# TradingMind

An autonomous SDLC skill chain that takes a project from idea to working code — and handles production issues when things go wrong.

## How It Works

There are three phases: **Plan**, **Build**, and **Fix**. Each phase is made up of skills that chain together automatically.

---

### Phase 1: Plan

Turn requirements into actionable tickets.

```
sdlc-router
    │
    ├── product-spec          What are we building and why?
    ├── architecture-blueprint    How is the system designed?
    ├── implementation-spec   What are the coding rules?
    ├── tickets               What are the units of work?
    └── test-plan             How do we validate it?
```

**Start here:** Give the `sdlc-router` your requirements — "build an expense tracker" or "add a new trading strategy." It figures out which planning steps are needed (not every change needs all five) and runs them in order. The output is a set of tickets in TRACKER.md, ready to build.

**Skills:**

| Skill | What it produces | When it runs |
|-------|-----------------|--------------|
| `sdlc-router` | Execution plan + orchestration | Always — it decides what else runs |
| `product-spec` | Requirements, constraints, success criteria | New project or new feature |
| `architecture-blueprint` | Components, data flow, infrastructure design | New system design or design changes |
| `implementation-spec` | Code conventions, module interfaces, CLAUDE.md | New codebase patterns or interface changes |
| `tickets` | TRACKER.md + individual ticket files per slice | Almost always — work needs to be tracked |
| `test-plan` | Test layers, tooling, coverage targets | New test patterns or tooling changes |

---

### Phase 2: Build

Turn tickets into working, reviewed code.

```
implementation-loop
    │
    ├── ticket-implementer    Write the code and tests
    └── code-reviewer         Review before merge
```

**Start here:** Run the `implementation-loop` after planning is done. It reads TRACKER.md, finds tickets whose dependencies are all complete, and for each one: implements it, reviews it, and moves to the next. It keeps going until everything is built or it hits a blocker.

**Skills:**

| Skill | What it does | When it runs |
|-------|-------------|--------------|
| `implementation-loop` | Picks up tickets in dependency order, orchestrates implement→review | After planning produces tickets |
| `ticket-implementer` | Reads a ticket, writes code + tests, updates TRACKER, creates a PR | Once per ticket |
| `code-reviewer` | Reviews the PR across 6 dimensions, produces verdict (approve/request changes/blocked) | After each implementation |

The loop retries once on "request changes." If it fails twice, it stops and asks for human help.

---

### Phase 3: Fix

Handle production issues by routing them to the right layer.

```
issue-triage
    │
    ├── Code bug ──────► ticket-implementer (direct fix)
    ├── Spec gap ──────► sdlc-router (update requirements)
    ├── Design gap ────► sdlc-router (update architecture)
    └── Config issue ──► direct fix
```

**Start here:** Tell `issue-triage` what's wrong — "the bot isn't closing positions when RSI drops below 30." It investigates the code, specs, and architecture to figure out which layer the problem is at, then routes it to the right fix path. Bugs go straight to the implementer. Missing requirements or design gaps go back through the planning chain.

**Skills:**

| Skill | What it does | When it runs |
|-------|-------------|--------------|
| `issue-triage` | Investigates a problem, classifies the root cause, routes to fix | When something isn't working as expected |

---

## The Full Picture

```
         ┌──────────────────────────────────────┐
         │          "Something's wrong"          │
         └──────────────┬───────────────────────┘
                        ▼
                  issue-triage
                        │
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
       Code bug    Spec/Design gap   Config fix
          │             │              │
          │             ▼              └──► Done
          │       sdlc-router
          │          │
          │          ├── product-spec
          │          ├── architecture-blueprint
          │          ├── implementation-spec
          │          ├── tickets
          │          └── test-plan
          │             │
          ▼             ▼
       implementation-loop
          │
          ├── ticket-implementer
          └── code-reviewer
                  │
                  ▼
            Working code
```

---

## Skill Count

| Phase | Skills | Purpose |
|-------|--------|---------|
| Plan  | 6      | Requirements → Design → Tickets |
| Build | 3      | Tickets → Code → Reviewed PRs |
| Fix   | 1      | Production issues → Right fix path |
| **Total** | **10** | |

---

## Getting Started

1. **New project:** Tell the sdlc-router what you want to build. It runs the planning chain and produces tickets.
2. **Build it:** Run the implementation-loop. It works through tickets automatically.
3. **Something breaks:** Describe the problem to issue-triage. It figures out where the fix belongs.

---

## Roadmap

### PM Board Integration (MCP)
Connect ticket tracking to an external board (ClickUp, Linear, Jira) so tickets live in a proper PM tool instead of markdown files. The tickets skill would create items on the board, the implementation-loop would read status from it, and the issue-triage would create bug tickets there. Managed via MCP connector.

### Document Store (MCP)
Move SDLC documents (product spec, architecture blueprint, implementation spec, test plan) out of the project repo and into a dedicated document store. Keeps the repo focused on code while making specs accessible across projects. Managed via MCP connector.

### Scheduled Implementation Loop
Run the implementation-loop as a cron job that periodically checks TRACKER.md for ready tickets and builds them automatically. Removes the need to manually trigger the loop after planning completes.

### Trading Strategy Research Chain
An upstream chain that researches, backtests, and validates trading strategies before feeding approved strategies into the sdlc-router as structured requirements. Closes the loop from "research idea" to "deployed strategy."

### Skill Packaging and Distribution
Package the skill chain for sharing and installation. Make it possible to install the full SDLC chain into a new environment with a single command.

### Output Structure Review
Audit the file and folder structure that skills produce to ensure generated artifacts land in clean, consistent directories across projects.
