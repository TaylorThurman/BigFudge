---
name: tickets
description: >
  Phase 4 of the SDLC skill chain. Generates vertically-sliced development tickets with TRACKER.md and
  individual ticket files. Triggers: 'tickets', 'tasks', 'issues', 'work items', 'stories', 'break this
  into tickets', 'create the backlog', 'what should I build first', 'decompose this into work items',
  'plan the sprints'. Each ticket is sized for a single coding agent session (~500-1,500 tokens) with
  acceptance criteria, dependencies, and relevant interfaces. Includes CI/CD and infrastructure as tickets.
  Chain: product-spec, architecture-blueprint, implementation-spec, tickets, test-plan.
---

# Ticket Generator Skill

This skill produces a set of scoped, well-defined development tickets from an architecture blueprint and/or implementation spec. Each ticket is a self-contained unit of work — small enough that a coding agent can consume it in a single context window and produce correct, complete code.

## Where This Fits in the SDLC

This is **Phase 4** of the skill chain:

1. **Product Spec** — What are we building and why?
2. **Architecture Blueprint** — How is the system designed?
3. **Implementation Spec** — What are the rules and interfaces for the codebase?
4. **Tickets** (this skill) — What are the individual units of work?
5. **Test Plan** — How do we validate it?

The ticket skill consumes the architecture blueprint (for what to build) and the implementation spec (for how to build it) and produces ordered, dependency-aware tickets that a coding agent or developer picks up one at a time.

### What Belongs Here vs. Other Skills

**In this skill (work decomposition):**
- Individual, scoped development tasks
- Acceptance criteria for each task
- Dependency ordering between tickets
- Relevant interface contracts per ticket (pulled from implementation spec)
- Relevant architecture context per ticket (pulled from blueprint)

**NOT in this skill:**
- Module interface definitions (implementation spec)
- Coding conventions and patterns (implementation spec)
- System design rationale (architecture blueprint)
- Test strategy, validation suites, and coverage targets (test plan)

---

## Core Principles

**One ticket, one reason to change.** A ticket should have exactly one reason to change in the future. If two parts of a ticket can break independently, be tested independently, or be modified without affecting each other — they belong in separate tickets. This is the fundamental rule that all other decomposition guidance flows from.

The test: ask "if I need to change or fix part of this ticket later, would I also need to touch the other parts?" If no, they're separate tickets. A bot status indicator and a P&L summary can break independently, be tested independently, and be modified independently — so they're separate tickets, even if they live on the same page.

**How to apply this to common patterns:**

- **UI pages/screens:** A page is NOT one ticket. Each distinct component or widget is its own ticket — it has its own reason to change (different data source, different layout, different update frequency). A dashboard with a status indicator, strategy cards, and progress bars is at minimum 3 component tickets plus a layout/shell ticket. The shell ticket's reason to change is the page structure; each component's reason to change is its specific data and presentation.

- **Service classes with multiple methods:** A service class where each method serves a different consumer (different endpoint, different component, different feature) is NOT one ticket. Each method that serves an independent consumer is its own ticket, because that method changes when its consumer's needs change — not when unrelated methods change. A `DashboardService` with `get_bot_status()`, `get_portfolio_pnl()`, and `get_pending_approvals()` is at least 3 tickets — each method is independently testable and changes for independent reasons.

- **Cross-cutting configuration:** Configuring the same thing (polling intervals, permissions, validation rules) across multiple independent components is NOT one ticket. Each component's configuration changes independently — the bot status polling interval has nothing to do with the approval badges polling interval. Group configuration tickets by the component they serve, not by the type of configuration.

- **API endpoints:** An endpoint with complex business logic splits along consumer boundaries. If one route serves one consumer with one data transformation, it's one ticket. If a route aggregates data for multiple independent purposes, split by purpose.

- **Multi-step workflows:** Each step that can fail independently or be modified independently is its own ticket. A workflow that validates input, processes data, and sends notifications is 3 tickets — input validation rules change for different reasons than notification formatting.

**Simple CRUD is the exception.** A straightforward create/read/update/delete for a single resource (one model, one set of routes, one set of tests) can stay as one ticket — all the operations exist for the same reason (managing that resource) and change together.

**Self-contained context.** Each ticket must include everything a coding agent needs to complete the work without reading the full architecture blueprint or implementation spec. Pull in the specific interfaces, data structures, and conventions that are relevant — don't just reference them.

**Token-efficient.** Each ticket should target 300–800 tokens. That's enough for a clear description, acceptance criteria, relevant interfaces, and dependency notes — without bloating the agent's context window. The implementation spec (3K–5K tokens) plus a single ticket (300–800 tokens) should be the complete input a coding agent needs. If a ticket is approaching 1,000+ tokens, it's doing too much — split it.

**Prioritized within slices.** Within each vertical slice, tickets are ordered by impact: the most critical functionality first, nice-to-haves last. Across slices, order by dependency and business value — the slice that delivers the most value or unblocks the most other work comes first. The TRACKER's ticket order IS the priority order — the first ticket listed is the highest priority.

**Vertical slices over horizontal layers.** Organize tickets so each slice delivers a working, interactable feature end-to-end — data model through backend logic through any user-facing surface. After completing a slice, the user should be able to run the system and see that feature working. The only exception is a thin foundation slice (Slice 0) for shared infrastructure that every feature depends on. Never organize tickets by technical layer (all database, then all backend, then all frontend) — that delays feedback and hides design problems.

**Traceable to requirements.** Every ticket (except Slice 0 foundation) traces to a specific requirement in a feature doc via its `Requirements` field. This creates a direct link: business requirement → ticket → code change → PR. If you can't point to the requirement a ticket delivers, the ticket either shouldn't exist or the requirement is missing from the product docs.

**Testable outcomes.** Every ticket has acceptance criteria that can be verified — either by running a command, checking a behavior, or inspecting an output. "It works" is not acceptance criteria. "Running `python -m pytest tests/test_parser.py` passes with all 4 test cases" is.

**Standardized naming.** Ticket IDs propagate through the entire delivery chain:
- Branch: `feature/T-XXX-short-description` (e.g., `feature/T-003-zigbee-reader`)
- Commits: `T-XXX: Description` (e.g., `T-003: Add Zigbee sensor reader with persistence`)
- PR title: `T-XXX: Description`
This makes every code change traceable back to its ticket, requirement, and feature.

---

## Inputs

Before generating tickets, gather as much context as possible:

1. **Architecture Blueprint** (strongly preferred) — Provides the component architecture, data flow, state machines, and design decisions.
2. **Implementation Spec** (strongly preferred) — Provides module interfaces, code structure, conventions, and dependencies.
3. **Product Documentation** (for traceability) — Read FEATURES.md to identify which features are in scope, then read the relevant feature docs in features/ for their requirements (FR-XXX format). Each ticket should trace back to the specific requirements it delivers.
4. **Conversation context** — If upstream documents don't exist, gather enough from the conversation to decompose the work. Suggest the user create upstream documents first for best results.

If upstream documents exist, read them completely before writing any tickets.

---

## Ticket Structure

Every ticket follows this format:

```markdown
## T-[NNN]: [Ticket Title]

**Slice:** [Which vertical slice this belongs to — e.g., "Slice 0: Foundation" or "Slice 3: Preference Learning"]
**Depends On:** [T-XXX, T-YYY, or "None — can start immediately"]
**Requirements:** [FR-XXX from features/feature-name.md — the specific requirements this ticket delivers, or "Foundation — no direct feature requirement"]
**Estimated Scope:** [Small | Medium — one focused concern per ticket]

### Description
[1-3 paragraphs explaining what this ticket accomplishes. Be specific about what gets built, what it connects to, and why it exists in this order. A coding agent reading only this description should understand the task completely.]

### Relevant Interfaces
[Pull the specific interface contracts from the implementation spec that this ticket needs to implement or consume. Include the actual code signatures — don't just reference them. Only include interfaces directly relevant to this ticket.]

```python
# Example: include the actual abstract base class or function signature
class SensorReader(ABC):
    @abstractmethod
    def read(self) -> SensorReading:
        """Return the latest reading from this sensor."""
        ...
```

### Acceptance Criteria
[Bulleted list of testable conditions. Each criterion should be verifiable by running a command, checking a behavior, or inspecting output.]

- [ ] [Criterion 1 — specific and testable]
- [ ] [Criterion 2 — specific and testable]
- [ ] [Criterion 3 — specific and testable]

### Notes
[Optional. Any gotchas, edge cases, or context that doesn't fit above. Keep brief.]
```

---

## Ticket Generation Process

### Step 1: Identify Slice 0 (Foundation)

Read the architecture blueprint and implementation spec. Identify the minimal shared infrastructure that every feature depends on:

- Project scaffolding (directory structure, config loading, base classes)
- Shared data models that cross module boundaries
- Common utilities (logging setup, error types, database connection)

Slice 0 should be as thin as possible — only what's genuinely shared. If something is only used by one feature, it belongs in that feature's vertical slice, not in the foundation.

### Step 2: Identify Vertical Feature Slices

Each slice delivers one feature end-to-end. A slice includes everything needed for that feature to work — data model, business logic, API/interface layer, and any integration points. After completing a slice, the system should be runnable and that feature should be interactable.

To identify slices, look at the architecture blueprint's component architecture and data flow:
- Each major component or pipeline often maps to one vertical slice
- Group tightly-coupled components into a single slice
- Keep loosely-coupled components in separate slices

Order slices so that:
1. The most foundational feature comes first (the one other features are least likely to depend on)
2. Features that depend on earlier features come later
3. User-facing integration features (dashboards, notifications) come after the features they surface

Example for a smart home system:
- **Slice 0:** Project structure, config, base classes, shared models
- **Slice 1:** Sensor ingestion end-to-end (Zigbee reader → data model → basic status API endpoint)
- **Slice 2:** Automation engine (rules engine → HVAC/lighting control → manual override via API)
- **Slice 3:** Preference learning (data collection → model → suggestion API)
- **Slice 4:** Energy tracking (meter reading → aggregation → weekly report generation)
- **Slice 5:** Alerting (event detection → Telegram notification → alert history)
- **Slice 6:** Dashboard (web UI — page shell, sensor status widget, automation controls widget, energy chart widget, alert feed widget, each as separate tickets)

### Step 3: Decompose Each Slice into Tickets

Within each slice, break the work into tickets that are each completable in a single focused session. There is no target range for tickets per slice — a simple slice might produce 3 tickets, a complex dashboard slice might produce 10+. The right number of tickets is however many it takes to ensure each ticket is one focused concern.

Guidelines for scoping — every ticket is **one focused concern**:
- **Small ticket** (~300-500 tokens): Single file or class, one concern. Example: "SensorReading data model and database schema."
- **Medium ticket** (~500-800 tokens): Multiple files serving one concern. Example: "ZigbeeReader that polls sensors and persists readings."

There is no "large" tier. If a ticket feels large, it's multiple concerns — split it.

**The independence test:** For each piece of work in the ticket, ask: "Can this break without the other pieces breaking? Can this be tested without testing the other pieces? Can this be modified without touching the other pieces?" If the answer to any of these is yes, split them into separate tickets.

**Example — wrong way (multiple reasons to change in one ticket):**
> "Home status page: bot status indicator, per-strategy cards, shadow evaluation progress bars, combined P&L, pipeline summaries, approval badges, HTMX partial refresh, mobile responsive layout"

Each of these has its own data source, its own presentation logic, and its own reason to change. Fixing the P&L calculation doesn't require touching the approval badges. Changing the strategy card layout doesn't affect the pipeline summary. Yet they're all in one ticket — which means the implementer has to hold all of them in context, the reviewer has to verify all of them, and a bug in any one of them blocks the entire ticket.

**Example — right way (one reason to change per ticket):**

| Ticket | Reason to change | Independent of |
|--------|-----------------|----------------|
| Dashboard page shell and layout grid | Page structure changes | All component content |
| Bot status indicator component | Bot status logic or display changes | Every other component |
| Strategy overview cards | Strategy data or card layout changes | Every other component |
| Shadow evaluation progress bars | Evaluation tracking changes | Every other component |
| Portfolio P&L summary | P&L calculation or display changes | Every other component |
| Pipeline and research run summaries | Pipeline/research data changes | Every other component |
| Approval quick-link badges | Approval queue logic changes | Every other component |
| Bot status + portfolio P&L service methods | Status and P&L data aggregation changes | Approval and pipeline service methods |
| Pipeline + approval service methods | Pipeline and approval data queries change | Status and P&L service methods |

Each ticket is independently buildable, testable, and reviewable. A bug in one doesn't block the others. A change request for one doesn't create noise in the others' PRs.

Within a slice, tickets should still follow a logical order — data model before business logic before interface layer. But across slices, the key is that each slice is independently completable and produces a working feature.

### Step 4: Assign Dependencies

For each ticket, identify which other tickets must be completed first. Use the ticket IDs (T-001, T-002, etc.) to express dependencies.

Rules:
- Slice 0 tickets have no dependencies (or depend only on other Slice 0 tickets)
- First ticket of each feature slice depends on the relevant Slice 0 tickets
- Tickets within a slice depend on earlier tickets in the same slice
- Cross-slice dependencies should be rare — if two slices are tightly coupled, consider merging them

Minimize dependency chains. Slices that don't depend on each other can be worked in parallel.

### Step 5: Embed Relevant Context

For each ticket, pull in the specific interfaces and data structures from the implementation spec that the ticket needs. Don't make the coding agent look things up — include the actual code signatures in the ticket. But only include what's directly relevant — don't dump the entire interfaces section into every ticket.

---

## Output Format

Produce the tickets as a **multi-file output** — one tracker file plus one file per ticket, organized by slice. This keeps each ticket small enough for a coding agent to consume individually, and scales to projects of any size.

### Directory Structure

```
[project-name]-tickets/
├── TRACKER.md                          # Summary, status tracking, dependency map
├── slice-0-foundation/
│   ├── T-001-project-scaffolding.md    # Individual ticket file
│   ├── T-002-shared-data-models.md
├── slice-1-sensor-ingestion/
│   ├── T-003-zigbee-reader.md
│   ├── T-004-sensor-status-api.md
├── slice-2-automation/
│   ├── T-005-rules-engine.md
│   └── ...
└── ...
```

Directory names follow the pattern `slice-N-short-name/`. Ticket filenames follow the pattern `T-NNN-kebab-title.md`.

### TRACKER.md

The tracker is the single file a human reads for the big picture. It contains the document header, the summary table with status tracking, and the changelog. It does NOT contain full ticket content — just references.

```markdown
# [Project Name] — Ticket Tracker

**Version:** [vN]
**Date:** [Date]
**Architecture Blueprint:** [Reference to version]
**Implementation Spec:** [Reference to version]
**Total Tickets:** [count]
**Vertical Slices:** [count]

## Slice Overview

| Slice | Feature | Tickets | Status |
|-------|---------|---------|--------|
| Slice 0 | Foundation | T-001, T-002 | Todo |
| Slice 1 | Sensor Ingestion | T-003, T-004 | Todo |
| Slice 2 | Automation | T-005, T-006, T-007 | Todo |
| ... | ... | ... | ... |

## Ticket Summary

| ID | Title | Slice | Depends On | Scope | Status |
|----|-------|-------|------------|-------|--------|
| T-001 | Project scaffolding and config loader | Slice 0 | None | Small | Todo |
| T-002 | Shared data models and database setup | Slice 0 | T-001 | Small | Todo |
| T-003 | Zigbee sensor reader and persistence | Slice 1 | T-002 | Medium | Todo |
| T-004 | Sensor status API endpoint | Slice 1 | T-003 | Small | Todo |
| T-005 | Automation rules engine | Slice 2 | T-004 | Medium | Todo |
| ... | ... | ... | ... | ... | ... |

## Changelog

### v1 — Initial Tickets
- Generated [N] tickets across [M] vertical slices from [architecture blueprint vX / implementation spec vY]
- [Note any assumptions or gaps]
```

Status values: **Todo** | **In Progress** | **In Review** | **Done** | **Blocked**

The Slice Overview table lets the user see at a glance which features are complete. The Ticket Summary table provides the detailed view. Both are updated as work progresses.

### Individual Ticket Files

Each ticket is its own markdown file using the ticket structure defined above. One file = one unit of work for a coding agent.

The ticket file is self-contained — a coding agent reads the ticket file + the implementation spec and has everything it needs. It never needs to read the tracker or other ticket files.

---

## Writing Guidelines

- **Be specific in acceptance criteria.** "Tests pass" is not enough. "Running `pytest tests/test_config.py` passes all 3 cases (valid config, missing required field, invalid value)" is.
- **Include actual code signatures, not references.** Don't say "Implement the interface defined in Section 3 of the implementation spec." Instead, include the relevant abstract base class or function signature directly in the ticket.
- **Keep tickets self-contained.** A coding agent should be able to complete a ticket by reading only: (1) the ticket file, (2) the implementation spec's conventions section, and (3) the existing codebase. It should never need to read the tracker, other tickets, or the full architecture blueprint.
- **Name tickets by what they produce, not what they do.** "Config parser module" over "Write the config parser." This makes the tracker scannable.
- **Name files descriptively.** `T-003-zigbee-reader.md` over `T-003.md`. A developer browsing the directory should know what each ticket covers without opening the file.
- **Flag ambiguity.** If the architecture blueprint or implementation spec leaves something underspecified that a ticket needs, note it in the ticket's Notes section rather than guessing silently.

---

## Output

Create the ticket directory structure, write each ticket as an individual file, and write the TRACKER.md. The directory should follow the pattern:
`[project-name]-tickets/`

Save the directory and present the TRACKER.md to the user, along with a summary of how many tickets and slices were generated.

---

## Downstream Handoff

Tickets feed directly into development. Do not suggest or invoke the next skill — the SDLC router (or the user) determines what runs next. After completing each vertical slice, the system should have a new working, interactable feature.

Note: Deployment procedures, CI/CD pipelines, and infrastructure-as-code are implementation work and should be included as tickets (e.g., "Set up GitHub Actions workflow," "Configure AWS ECS task definitions").
