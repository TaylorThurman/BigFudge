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

**Start here:** Say "Plan this project — [your requirements]" for a new project, or "New requirements — [what changed]" to update an existing one. The router reads whatever docs already exist, figures out which planning steps are needed (not every change needs all five), and runs them in order. The output is a set of tickets in TRACKER.md, ready to build.

**Skills:**

| Skill | What it produces | When it runs |
|-------|-----------------|--------------|
| `sdlc-router` | Execution plan + orchestration | Always — it decides what else runs |
| `product-spec` | PRODUCT.md, FEATURES.md, features/*.md | New project or new feature |
| `architecture-blueprint` | Components, data flow, infrastructure design | New system design or design changes |
| `implementation-spec` | Code conventions, module interfaces, CLAUDE.md, project README | New codebase patterns or interface changes |
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

**Start here:** Say "Start the implementation loop" after planning is done. It reads TRACKER.md, finds all tickets whose dependencies are complete (`In Review` or `Done`), and spawns up to 3 agents working in parallel. As each agent finishes, main is merged into the feature branch, tests run, and a PR is created — but never auto-merged. The project owner reviews and approves every PR before code enters main. When a PR is created the ticket moves to `In Review`, which unblocks dependent tickets so agents keep working without waiting for human approval. If two branches conflict during the merge, a conflict resolution agent reads both tickets' context and resolves it intelligently. The loop keeps going until all tickets have PRs up or it hits a blocker.

**Skills:**

| Skill | What it does | When it runs |
|-------|-------------|--------------|
| `implementation-loop` | Orchestrates parallel agents, creates PRs, resolves conflicts, updates TRACKER | After planning produces tickets |
| `ticket-implementer` | Reads a ticket, writes code + tests, creates a feature branch and PR | Once per ticket (up to 3 in parallel) |
| `code-reviewer` | Reviews the PR across 6 dimensions, produces verdict (approve/request changes/blocked) | After each implementation |

Each agent retries once on "request changes." If it fails twice, that ticket is marked blocked and the loop continues with other available tickets. Merge conflicts between parallel branches are resolved by a dedicated agent that understands both tickets' intent — only unresolvable conflicts escalate to the project owner.

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

**Start here:** Say "Something is broken — [describe the issue]", for example "Something is broken — the bot isn't closing positions when RSI drops below 30." It investigates the code, specs, and architecture to figure out which layer the problem is at, then routes it to the right fix path. Bugs go straight to the implementer. Missing requirements or design gaps go back through the planning chain.

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

| Phase | Skills | Names |
|-------|--------|-------|
| Plan  | 6      | sdlc-router, product-spec, architecture-blueprint, implementation-spec, tickets, test-plan |
| Build | 3      | implementation-loop, ticket-implementer, code-reviewer |
| Fix   | 1      | issue-triage |
| **Total** | **10** | |

**TRACKER Status Lifecycle:** `Todo → In Progress → In Review → Done` (and `Blocked`). Tickets move to `In Review` when code is complete and a PR is created. They move to `Done` only after the project owner approves and merges the PR. Dependencies are considered met at `In Review` — agents don't wait for human approval to start dependent work.

---

## Ticket Principles

Every ticket has **one reason to change.** If two parts of a ticket can break independently, be tested independently, or be modified without affecting each other — they belong in separate tickets. A page is not one ticket (each component has its own reason to change). A service class with methods serving different consumers is not one ticket. Cross-cutting configuration belongs with the component it serves, not in a separate pass. Tickets are sized Small or Medium — there is no Large tier.

**Prioritization:** Tickets are ordered by impact within each vertical slice. The TRACKER's ticket order IS the priority order. High-impact, foundational work comes first.

**Traceability:** Every ticket (except Slice 0 foundation) links to a specific requirement via its `Requirements` field (e.g., `FR-003 from features/risk-alerts.md`). This creates a direct chain: business requirement → ticket → branch → PR → merged code.

**Naming conventions** flow from ticket to branch to commit to PR:

| Artifact | Format | Example |
|----------|--------|---------|
| Branch | `feature/T-XXX-short-desc` | `feature/T-003-zigbee-reader` |
| Commit | `T-XXX: Description` | `T-003: Add Zigbee sensor reader with persistence` |
| PR title | `T-XXX: Description` | `T-003: Add Zigbee sensor reader with persistence` |

---

## Product Documentation Structure

The product-spec skill produces documentation in a feature-per-doc architecture to keep things manageable as the project grows:

```
PRODUCT.md           ← Project-level context (vision, constraints, users, success criteria)
FEATURES.md          ← Index of all features with status, priority, and links
features/
  user-auth.md       ← Self-contained feature doc with requirements, workflows, scope
  portfolio-view.md
  risk-alerts.md
```

**PRODUCT.md** holds everything that applies across the entire project — problem statement, target users, constraints, quality attributes, and overall success criteria. Written once, updated rarely.

**FEATURES.md** is the index. A table listing every feature, its status (Draft / Review / Approved / Implemented / Deprecated), priority (MoSCoW), and a link to its doc.

**features/*.md** are individual feature docs, each self-contained with its own requirements (FR-XXX format), user workflows, acceptance criteria, and scope. Adding a new feature means adding a new file — not editing existing ones.

---

## Getting Started

Each phase is triggered by a specific phrase. Use these exact phrases to avoid ambiguity:

| Phase     | What to say                                  | What happens                                               |
| --------- | -------------------------------------------- | ---------------------------------------------------------- |
| **Plan (new)** | "Plan this project — [your requirements]" | Runs the full planning chain, produces specs and tickets |
| **Plan (update)** | "New requirements — [what changed]" | Reads existing docs, re-runs only the planning steps that need updating |
| **Build** | "Start the implementation loop"              | Picks up tickets, implements them in parallel, creates PRs |
| **Fix**   | "Something is broken — [describe the issue]" | Triages the problem, routes to the right fix path          |

Other useful phrases:

| Action | What to say |
|--------|------------|
| Implement a single ticket | "Implement this ticket — T-001" |
| Review a PR | "Review this PR" or "Review T-004" |
| Resume a previous build | "Start the implementation loop" (it reads TRACKER and picks up where it left off) |

Each phase runs independently. Planning does not auto-start building. Building does not auto-start after planning. You control when each phase begins.

---

## Roadmap

### PM Board Integration (MCP)
Connect ticket tracking to an external board (ClickUp, Linear, Jira) so tickets live in a proper PM tool instead of markdown files. The tickets skill would create items on the board, the implementation-loop would read status from it, and the issue-triage would create bug tickets there. Managed via MCP connector.

### Document Store (MCP)
When building a project, the planning skills generate documents (product spec, architecture blueprint, implementation spec, test plan). These should be stored and retrieved through a dedicated document store via MCP rather than living as files in the project repo. Skills write documents to the store as they produce them, and read from it when they need context — the ticket-implementer pulls the implementation spec, the code-reviewer pulls the architecture blueprint, the issue-triage pulls specs to compare against code, etc. Keeps the repo focused on code.

### Output Structure Review
Audit the file and folder structure that skills produce to ensure generated artifacts land in clean, consistent directories across projects.
