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

**One ticket, one focus.** Each ticket should do one thing well. A coding agent reading a ticket should never think "this is actually three tasks." If it feels too big, split it.

**Self-contained context.** Each ticket must include everything a coding agent needs to complete the work without reading the full architecture blueprint or implementation spec. Pull in the specific interfaces, data structures, and conventions that are relevant — don't just reference them.

**Token-efficient.** Each ticket should target 500–1,500 tokens. That's enough for a clear description, acceptance criteria, relevant interfaces, and dependency notes — without bloating the agent's context window. The implementation spec (3K–5K tokens) plus a single ticket (500–1,500 tokens) should be the complete input a coding agent needs.

**Vertical slices over horizontal layers.** Organize tickets so each slice delivers a working, interactable feature end-to-end — data model through backend logic through any user-facing surface. After completing a slice, the user should be able to run the system and see that feature working. The only exception is a thin foundation slice (Slice 0) for shared infrastructure that every feature depends on. Never organize tickets by technical layer (all database, then all backend, then all frontend) — that delays feedback and hides design problems.

**Testable outcomes.** Every ticket has acceptance criteria that can be verified — either by running a command, checking a behavior, or inspecting an output. "It works" is not acceptance criteria. "Running `python -m pytest tests/test_parser.py` passes with all 4 test cases" is.

---

## Inputs

Before generating tickets, gather as much context as possible:

1. **Architecture Blueprint** (strongly preferred) — Provides the component architecture, data flow, state machines, and design decisions.
2. **Implementation Spec** (strongly preferred) — Provides module interfaces, code structure, conventions, and dependencies.
3. **Conversation context** — If neither document exists, gather enough from the conversation to decompose the work. Suggest the user create upstream documents first for best results.

If both documents exist, read them completely before writing any tickets.

---

## Ticket Structure

Every ticket follows this format:

```markdown
## T-[NNN]: [Ticket Title]

**Slice:** [Which vertical slice this belongs to — e.g., "Slice 0: Foundation" or "Slice 3: Preference Learning"]
**Depends On:** [T-XXX, T-YYY, or "None — can start immediately"]
**Estimated Scope:** [Small | Medium | Large — relative to a single coding session]

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
- **Slice 6:** Dashboard (web UI consuming all existing API endpoints)

### Step 3: Decompose Each Slice into Tickets

Within each slice, break the work into tickets that are each completable in a single focused session. A vertical slice typically produces 2-5 tickets.

Guidelines for scoping:
- **Small ticket** (~500 tokens, 1-2 hours): Single file or class. Example: "SensorReading data model and database schema."
- **Medium ticket** (~1,000 tokens, 2-4 hours): A module with 2-3 files. Example: "ZigbeeReader that polls sensors and persists readings."
- **Large ticket** (~1,500 tokens, 4-8 hours): A feature slice that's simple enough to not need splitting. Example: "Telegram alerter with event detection, message formatting, and send queue."

Prefer small and medium tickets. Large tickets should be rare — if you're writing one, consider whether it can be split.

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
