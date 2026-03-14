---
name: sdlc-router
description: "Orchestrator for the SDLC planning chain. Receives new requirements or change requests, assesses the current project state by reading existing documents, determines which planning phases need to run, executes them in order with validation between each, and escalates to the user only when genuinely ambiguous. Triggers: 'plan this project', 'new requirements', 'new feature request', 'add a strategy', 'update the requirements', 'start the SDLC', 'run the planning chain', 'here are new requirements', 'define the project'. This skill handles PLANNING only — it produces specs and tickets. It does NOT build code or run the implementation loop. If the user wants to start building from existing tickets, use the implementation-loop skill instead."
---

# SDLC Router

This skill orchestrates the SDLC skill chain. Instead of requiring the user to manually invoke each phase (product-spec, architecture-blueprint, implementation-spec, tickets, test-plan), the router figures out what the project needs and drives the chain automatically.

The router exists because not every change needs every phase. A new trading strategy with well-defined parameters might only need a product spec update, new tickets, and a test plan revision — the architecture and implementation spec might already cover the pattern. Blindly running all five phases wastes time and tokens. The router's job is to be smart about what actually needs to happen.

## The SDLC Chain

The router orchestrates these five phases:

1. **Product Spec** — What are we building and why? (requirements, constraints, success criteria)
2. **Architecture Blueprint** — How is the system designed? (components, data flow, security)
3. **Implementation Spec** — What are the rules for the codebase? (interfaces, conventions, CLAUDE.md)
4. **Tickets** — What are the individual units of work? (vertical slices, acceptance criteria)
5. **Test Plan** — How do we validate it? (test layers, tooling, thresholds)

Each phase has its own skill. The router invokes them as needed — it does not produce documents itself.

---

## How the Router Works

### Step 1: Assess Current State

Read the project directory to understand what exists. Look for files matching these patterns:

```
project-directory/
├── *-product-spec-v*.md
├── *-architecture-blueprint-v*.md
├── *-implementation-spec-v*.md
├── *-tickets/
│   ├── TRACKER.md
│   └── slice-*/T-*.md
└── *-test-plan-v*.md
```

For each document found, note its version number and read it. The router needs to understand the current state of the project to make good decisions about what needs to change.

If no project directory exists or it's empty, this is a greenfield project — all phases will likely be needed.

### Step 2: Understand the Input

The input to the router is one of:

**New requirements** — A set of requirements for something that doesn't exist yet. This could come from a user describing what they want, or from an upstream chain (like a research pipeline) delivering structured output. For greenfield projects, this typically triggers the full chain.

**Updated requirements** — Changes to an existing project. A new feature, a modified constraint, a new strategy to add. This is where the router earns its keep — it needs to figure out which documents are affected.

**Change request** — A specific modification ("change the database from SQLite to PostgreSQL", "add a new API endpoint", "update the deployment to use ECS"). This might only affect one or two phases.

**Triage output** — A change request produced by the issue-triage skill after investigating a production problem. Triage output is pre-classified: it tells you whether the issue is a spec gap (missing/wrong requirement) or an architecture gap (design can't support the needed behavior). The triage report includes the root cause analysis and a structured change request. Treat triage output as a high-confidence change request — the investigation has already been done, so focus on which phases need to run rather than re-diagnosing the problem. Note: if the triage classified the issue as a code bug, it routes directly to the ticket-implementer and skips the router entirely.

Read the input carefully. Identify:
- What is being added, changed, or removed?
- Which existing documents does this affect?
- Is this a new capability or a modification to an existing one?

### Step 3: Determine the Execution Plan

For each phase, decide: **run**, **skip**, or **check-and-decide**.

#### Product Spec

**Run** when: new requirements arrive (greenfield or new feature), requirements are being modified, or the input is structured requirements from an upstream chain.

**Skip** when: the change is purely technical (refactoring, infrastructure change) and doesn't affect what the system does from the user's perspective.

#### Architecture Blueprint

**Run** when: new components are needed, data flow changes, infrastructure topology changes, or a new integration is required.

**Skip** when: the existing architecture already supports the new requirements. This is common when adding a new instance of an existing pattern — for example, adding a new trading strategy when the architecture already has a strategy framework. The key question is: does this change require new *design decisions*, or does it fit within existing design decisions?

**Check-and-decide**: Read the updated product spec and the existing architecture blueprint. Ask: are there new requirements that the current architecture doesn't address? If every new requirement maps to an existing component or pattern, skip. If any requirement needs something the architecture doesn't have, run.

#### Implementation Spec

**Run** when: new module interfaces are needed, coding conventions change, or the architecture was updated in a way that affects code structure.

**Skip** when: the change fits within existing interfaces and conventions. Adding a new strategy that implements an existing abstract base class doesn't need an implementation spec update — the interface is already defined.

**Check-and-decide**: Read the updated product spec (and updated architecture if it was re-run) against the existing implementation spec. Ask: does the developer need any new information to implement this correctly? If the existing interfaces, conventions, and CLAUDE.md cover it, skip.

#### Tickets

**Run** when: there's work to be done. Almost always run — even small changes need tickets to track the work.

**Skip** when: the only change was documentation (product spec clarification, architecture diagram update) with no code impact.

#### Test Plan

**Run** when: new test layers are needed, tooling changes, coverage targets change, or new validation gates are required.

**Skip** when: the existing test plan's tooling, layers, and conventions already cover the new work. Adding a new feature that uses the same test patterns doesn't need a test plan update — the tickets' acceptance criteria drive the specific tests, and the test plan already defines how to test.

**Check-and-decide**: Ask: does the developer need any new information about *how to test* that isn't already in the test plan? If the testing approach, tools, and coverage targets already apply, skip.

### Step 4: Execute the Plan

Run each phase in order, invoking the corresponding skill. Between each phase, validate the output before proceeding.

**Validation between phases:**

After each phase completes, read the output and check:

1. **Completeness** — Does the output address all the new/changed requirements? If the product spec added 3 new requirements, do all 3 appear in the architecture (if architecture was run)?

2. **Consistency** — Does the output conflict with existing documents that aren't being updated? If the architecture was skipped, does the new implementation spec still align with the existing architecture?

3. **Open questions** — Did the phase produce open questions that need answers before the next phase can proceed? If the product spec has unresolved questions that the architecture needs, those must be resolved first.

If validation fails, decide whether to:
- **Re-run the phase** with additional guidance (e.g., "the output missed requirement FR-015")
- **Run a previously-skipped phase** (e.g., "the implementation spec assumes an interface that doesn't exist in the architecture — the architecture needs updating first")
- **Escalate to the user** (e.g., "the new requirements conflict with an existing constraint — which takes priority?")

### Step 5: Report Results

When the chain completes, provide a summary:
- Which phases were run and which were skipped (with reasoning)
- What documents were created or updated (with version numbers)
- Any open questions or decisions that were deferred
- What the next steps are — specifically: tickets are ready for the **implementation-loop** skill, which autonomously picks up tickets in dependency order, runs the **ticket-implementer** and **code-reviewer** for each, and keeps going until everything is built or a blocker is hit. Alternatively, tickets can be implemented one at a time by invoking the ticket-implementer and code-reviewer manually.

---

## Escalation Rules

The router should handle routine decisions autonomously. Escalate to the user only when:

**Conflicting requirements.** The new input contradicts an existing requirement or constraint, and the router can't determine which takes priority. Example: "The new strategy requires sub-millisecond execution, but the architecture specifies a polling-based pipeline with 1-second intervals."

**Fundamental design changes.** The new requirements need an architectural pattern the system doesn't have, and adopting it would affect existing functionality. Example: "Adding real-time streaming requires replacing the REST API with WebSockets, which would break existing clients."

**Ambiguous scope.** The input is too vague to determine what's needed. Example: "Make the system faster" — faster at what? Which component? What's the target?

**Cost or risk implications.** The change involves infrastructure changes with cost implications, or affects safety-critical components (like trading risk limits).

Do NOT escalate for:
- Routine decisions within existing patterns (which strategy pattern to use, how to name a new module)
- Technical choices that follow established conventions (database schema for a new entity that follows existing patterns)
- Phase ordering or skipping decisions — that's the router's core job

---

## Working with Upstream Chains

When input comes from an automated upstream chain (like a trading strategy research pipeline), the router should:

1. **Validate the input format.** The input should be structured markdown similar to a product spec — requirements with acceptance criteria, constraints, and success metrics. If the input is too sparse, escalate rather than guessing.

2. **Treat the input as approved requirements.** The user has already approved the upstream output. The router doesn't second-guess whether the strategy is a good idea — it focuses on how to implement it.

3. **Map to existing project structure.** The new requirements get folded into the existing product spec as new or updated functional requirements, not created as a separate document.

4. **Preserve version history.** When updating existing documents, increment the version number and document what changed in the changelog. The previous version should remain accessible.

---

## Output

The router itself doesn't produce files — it invokes other skills that produce files. The router's output is the execution summary described in Step 5, plus any escalation questions if they arise.

When invoking skills, pass the relevant context:
- For product-spec: the new requirements and the existing product spec (if updating)
- For architecture-blueprint: the updated product spec and the existing blueprint (if updating)
- For implementation-spec: the updated architecture (or existing if skipped) and existing implementation spec
- For tickets: the architecture and implementation spec (current versions)
- For test-plan: all upstream documents (current versions)

---

## Examples

**Example 1: New trading strategy (existing project)**

Input: "Research chain approved a mean reversion strategy for BTC/USDT. Parameters: 20-period SMA, entry at 2 standard deviations, position size 0.5 BTC max, stop loss at 3%."

Router assessment:
- Product spec: **Run** — new functional requirements for the strategy
- Architecture: **Check** — reads existing blueprint, finds strategy framework already exists with abstract base class and pipeline integration. **Skip.**
- Implementation spec: **Check** — reads existing spec, finds StrategyBase interface already defined. New strategy implements existing interface. **Skip.**
- Tickets: **Run** — need tickets for: implement MeanReversionStrategy class, add configuration, write tests, backtest validation
- Test plan: **Check** — existing test plan covers strategy testing patterns. **Skip.**

Result: Only product spec and tickets are generated. Fast, focused.

**Example 2: Greenfield project**

Input: "I want to build a personal expense tracker. React frontend, FastAPI backend, PostgreSQL, deployed on AWS with GitHub Actions."

Router assessment: No existing documents. All phases needed.
- Product spec: **Run**
- Architecture: **Run**
- Implementation spec: **Run**
- Tickets: **Run**
- Test plan: **Run**

Result: Full chain executed.

**Example 3: Infrastructure change**

Input: "Migrate the database from SQLite to PostgreSQL on AWS RDS."

Router assessment:
- Product spec: **Skip** — no functional change from user perspective
- Architecture: **Run** — infrastructure topology changes, new external service
- Implementation spec: **Run** — database connection patterns change, new dependencies
- Tickets: **Run** — migration work items needed
- Test plan: **Check** — test tooling might need database fixture changes. **Run** if integration test approach changes.

Result: Architecture, implementation spec, tickets, and possibly test plan.
