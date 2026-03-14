---
name: product-spec
description: >
  Phase 1 of the SDLC skill chain. Produces product specification documents with MoSCoW-prioritized requirements,
  user workflows, success criteria, and scope boundaries. Triggers: 'product spec', 'requirements', 'PRD',
  'user stories', 'acceptance criteria', 'scope', 'feature list', 'what should it do', 'define the scope',
  'what are the requirements', 'nail down what we're building'. Use when the user is discussing features,
  constraints, or success criteria and wants to formalize them before design. Answers "what and why" so
  architecture can answer "how". Chain: product-spec, architecture-blueprint, implementation-spec,
  tickets, test-plan.
---

# Product Spec Skill

This skill produces product specification documents that define what a system should do, who it serves, and what success looks like. The product spec is the contract between the product owner's intent and everything that follows — architecture, implementation, testing, and operations.

A good product spec prevents the downstream skills from guessing. If the architecture blueprint is "how the system is designed," the product spec is "what the system needs to accomplish and under what constraints." Getting this right saves enormous rework later.

## Where This Fits in the SDLC

This is **Phase 1** of the skill chain:

1. **Product Spec** (this skill) — What are we building and why?
2. **Architecture Blueprint** — How is the system designed?
3. **Implementation Spec** — What are the rules and interfaces for the codebase?
4. **Tickets** — What are the individual units of work?
5. **Test Plan** — How do we validate it?

The product spec feeds downstream SDLC phases — every design decision in the architecture blueprint should trace back to a requirement or constraint defined here. If a requirement doesn't exist in the product spec, it shouldn't silently appear in later phases. Do not suggest or invoke the next skill — the SDLC router (or the user) determines what runs next.

---

## Document Structure

Product documentation is split across three layers to keep things manageable as the project grows:

```
PRODUCT.md           ← Project-level context (vision, constraints, users, success criteria)
FEATURES.md          ← Index of all features with status and links
features/
  user-auth.md       ← Self-contained feature doc
  portfolio-view.md
  risk-alerts.md
```

**PRODUCT.md** holds everything that applies across the entire project — the problem statement, target users, constraints, assumptions, and overall success criteria. This is written once and updated rarely. It does NOT contain feature-specific requirements.

**FEATURES.md** is the index. A simple table listing every feature, its status, priority, and a link to its doc. This is the first place anyone looks to understand what the project does.

**features/*.md** are individual feature docs. Each one is self-contained with its own requirements, workflows, acceptance criteria, and scope. When you add a new feature, you add a new file — you don't edit existing feature docs unless those features are changing.

This structure matters because it prevents monolithic specs from becoming unmanageable. A project with 20 features should have 20 small, focused docs — not one 50-page document that nobody wants to read.

---

## When to Create What

**New project (no existing docs):**
1. Conduct the interview (see below)
2. Create PRODUCT.md with project-level context
3. Create FEATURES.md index
4. Create a feature doc for each feature identified in the interview

**New feature (project already exists):**
1. Read PRODUCT.md for project context and FEATURES.md to understand what exists
2. Check if the requested feature overlaps with an existing one — if unclear, ask the user
3. Create a new feature doc in features/
4. Add an entry to FEATURES.md

**Updating an existing feature:**
1. Read the existing feature doc
2. Update the requirements, workflows, or scope as needed
3. Add a changelog entry at the bottom of the feature doc
4. Update FEATURES.md if the status or priority changed

**Updating project-level context:**
1. Read PRODUCT.md
2. Update the relevant section (constraints, users, success criteria, etc.)
3. Add a changelog entry

---

## Core Principles

**Clarity over completeness.** A product spec with 10 crystal-clear requirements is more useful than one with 50 vague ones. Every requirement should be specific enough that two people reading it would agree on whether a given implementation satisfies it.

**Separate the what from the how.** This document defines outcomes, not mechanisms. "The system must execute trades with human approval" is a product requirement. "Use a three-gate pipeline with FastAPI dashboard" is an architecture decision — that belongs in the blueprint, not here.

**Constraints are requirements too.** Hardware limitations, budget constraints, regulatory requirements, and personal preferences ("I want to operate everything from my phone") are just as important as functional requirements. They shape the solution space and should be captured explicitly.

**One feature, one doc.** Each feature is self-contained. A feature doc should make sense on its own without reading every other feature doc. Cross-references are fine when features interact, but each doc should stand alone.

**Previous docs are sacred.** When prior versions exist, every requirement must be accounted for in the new version. Nothing gets silently dropped.

---

## Interview Phase

Before writing anything, conduct a structured interview. The goal is to extract everything the user knows about what they want, even things they haven't thought to mention. Tailor the depth of each area to the project.

**Problem & Purpose:**
- What problem does this system solve?
- Who is it for? (Single user? Team? Customers?)
- What does success look like? How will you know it's working?

**Features:**
- What should the system do? Walk through the core capabilities.
- Which features are must-haves vs nice-to-haves?
- Are any features dependent on other features?

**Constraints & Boundaries:**
- What hardware/infrastructure is available?
- What's the budget? (Zero? Limited? Flexible?)
- Are there regulatory or compliance requirements?
- What's explicitly out of scope?

**User Experience:**
- How will users interact with the system? (CLI, dashboard, mobile, API, chat?)
- What's the expected frequency of interaction?
- What information does the user need to see?

**Quality Attributes:**
- How important is uptime/reliability?
- What's the acceptable latency for key operations?
- How important is security? What's the threat model?

**Dependencies & Integrations:**
- What external systems does this need to talk to?
- What happens when a dependency is unavailable?

Not every project needs deep answers to all of these. A personal tool has different requirements than an enterprise platform. Read the room and focus on what matters for this specific project.

---

## PRODUCT.md Structure

This is the project-level document. It holds context that applies to the entire project, not to any single feature.

```markdown
# [Project Name] — Product Overview

**Version:** [vN]
**Date:** [Date]
**Status:** [Draft | Review | Approved]

## Problem Statement & Vision

[2-3 paragraphs: what problem this solves, why existing solutions don't work,
what the end state looks like when this system is working well.]

## Users & Stakeholders

[Who uses this system, their roles, how they interact, what they care about.]

## Constraints

[Hard boundaries grouped by area: infrastructure, integration, operational, budget, regulatory.]

**C-001: [Constraint Title]**
[Description and why it exists.]

## Quality Attributes

[Non-functional requirements that apply across the system: performance,
reliability, security, uptime, latency targets.]

## Success Criteria

[Measurable or observable criteria that indicate the project is working.
Specific enough that the product owner can evaluate them.]

## Assumptions & Dependencies

[What you're assuming to be true. What external factors the system depends on.]

## Changelog

### v1 — [Date]
- Initial draft
```

---

## FEATURES.md Structure

The feature index. Simple and scannable.

```markdown
# Features

| Feature | Status | Priority | Doc |
|---------|--------|----------|-----|
| User Authentication | Draft | Must | [features/user-auth.md](features/user-auth.md) |
| Portfolio View | Draft | Must | [features/portfolio-view.md](features/portfolio-view.md) |
| Risk Alerts | Draft | Should | [features/risk-alerts.md](features/risk-alerts.md) |
```

**Status values:** Draft, Review, Approved, Implemented, Deprecated
**Priority values:** Must, Should, Could, Won't (MoSCoW)

---

## Feature Doc Structure

Each feature gets its own file in features/. The filename should be a kebab-case slug of the feature name.

```markdown
# [Feature Name]

**Priority:** [Must | Should | Could | Won't]
**Status:** [Draft | Review | Approved | Implemented | Deprecated]
**Date:** [Date]
**Related Features:** [Links to other feature docs this interacts with, or "None"]

## Overview

[1-2 paragraphs: what this feature does, why it exists, who benefits from it.]

## Requirements

**FR-001: [Requirement Title]**
The system shall [specific behavior].
Acceptance: [Testable condition that proves this requirement is met.]
Priority: [Must | Should | Could | Won't]

**FR-002: [Requirement Title]**
...

## Scope

**In Scope:** [What this feature covers in the current version.]

**Out of Scope:** [Only if the user explicitly excluded something. Don't assume.]

## User Workflows

#### [Workflow Name]
**Trigger:** [What initiates this workflow]
**Steps:**
1. [Step]
2. [Step]
**Expected Outcome:** [What the user sees when it works]
**What Could Go Wrong:** [Failure scenarios and impact]

## Open Questions

[Unresolved decisions or unknowns that need answers.]

## Changelog

### v1 — [Date]
- Initial draft
```

---

## Writing Guidelines

- **Be specific.** "The system should notify the user" is too vague. "The system sends a push notification to the user's phone when a new trade proposal requires approval" is specific enough to build from.
- **Use the user's language.** If the product owner says "shadow mode," use "shadow mode" — don't rename it. Consistency with how people talk about the system prevents confusion.
- **Distinguish requirements from solutions.** "Must support human approval before changes go live" is a requirement. "Use a three-gate pipeline" is a solution. Requirements belong here; solutions belong in the architecture blueprint.
- **Every requirement should be testable.** If you can't describe how to verify a requirement is met, it's too vague.
- **Include the 'why' for constraints.** A constraint without context becomes confusing when circumstances change.
- **Keep feature docs independent.** Each feature doc should make sense on its own. Use "Related Features" links when features interact, but don't make one feature doc depend on reading another to understand it.

---

## Output

For a new project, produce:
1. `PRODUCT.md` — project-level context
2. `FEATURES.md` — feature index
3. `features/[feature-name].md` — one doc per feature identified

For a new feature, produce:
1. `features/[feature-name].md` — the new feature doc
2. Updated `FEATURES.md` — add the new entry

For an update, produce:
1. Updated doc(s) with changelog entries
2. Updated `FEATURES.md` if status or priority changed

Save all files and present them to the user.
