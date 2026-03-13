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

## Core Principles

**Clarity over completeness.** A product spec with 10 crystal-clear requirements is more useful than one with 50 vague ones. Every requirement should be specific enough that two people reading it would agree on whether a given implementation satisfies it.

**Separate the what from the how.** This document defines outcomes, not mechanisms. "The system must execute trades with human approval" is a product requirement. "Use a three-gate pipeline with FastAPI dashboard" is an architecture decision — that belongs in the blueprint, not here.

**Constraints are requirements too.** Hardware limitations, budget constraints, regulatory requirements, and personal preferences ("I want to operate everything from my phone") are just as important as functional requirements. They shape the solution space and should be captured explicitly.

**Previous specs are sacred.** Same principle as the architecture blueprint — when a prior version exists, every requirement must be accounted for in the new version. Nothing gets silently dropped.

---

## Handling Previous Specs

Before writing, determine whether a previous product spec exists. Check for uploaded files, pasted content, or references to prior versions.

### If a previous spec exists:

1. Read it completely before writing anything new.
2. Create an inventory of all existing requirements and constraints.
3. Merge forward — every requirement from the previous version appears in the new version unless the user explicitly removes it.
4. Generate a Changelog section at the end.

### If no previous spec exists:

Gather as much information as possible from the conversation. The interview phase (below) is especially important for first drafts. Ask clarifying questions rather than making assumptions — the whole point of this document is to eliminate ambiguity.

---

## Interview Phase

Before writing the spec, conduct a structured interview. The goal is to extract everything the user knows about what they want, even things they haven't thought to mention. Tailor the depth of each area to the project.

**Problem & Purpose:**
- What problem does this system solve?
- Who is it for? (Single user? Team? Customers?)
- What does success look like? How will you know it's working?

**Functional Requirements:**
- What should the system do? Walk through the core workflows.
- What are the inputs and outputs?
- What decisions does the system make vs. what decisions does a human make?

**Constraints & Boundaries:**
- What hardware/infrastructure is available?
- What's the budget? (Zero? Limited? Flexible?)
- Are there regulatory or compliance requirements?
- What's explicitly out of scope? What should the system NOT do?

**User Experience:**
- How will users interact with the system? (CLI, dashboard, mobile, API, chat?)
- What's the expected frequency of interaction? (Hourly? Daily? Weekly?)
- What information does the user need to see? What actions do they need to take?

**Quality Attributes:**
- How important is uptime/reliability?
- What's the acceptable latency for key operations?
- How important is security? What's the threat model?
- Does it need to scale? To what degree?

**Dependencies & Integrations:**
- What external systems does this need to talk to?
- Are there APIs, brokers, services, or data sources involved?
- What happens when a dependency is unavailable?

Not every project needs deep answers to all of these. A personal tool has different requirements than an enterprise platform. Read the room and focus on what matters for this specific project.

---

## Product Spec Structure

### Document Header

```markdown
# [Project Name] — Product Specification

**Version:** [vN]
**Date:** [Date]
**Previous Version:** [version + date, or "N/A — initial draft"]
**Status:** [Draft | Review | Approved]
**Author(s):** [Who wrote/revised this]
```

### Section Order

1. Problem Statement & Vision
2. Users & Stakeholders
3. Functional Requirements
4. Non-Functional Requirements
5. Constraints
6. Scope Boundaries
7. User Workflows
8. Success Criteria
9. Assumptions & Dependencies
10. Open Questions
11. Changelog (always included)

---

## Section Details

### 1. Problem Statement & Vision

What problem does this system solve, and what's the vision for how it solves it? This should be 2-3 paragraphs that anyone could read and understand. No jargon. No implementation details. Just the problem and the intended solution at a high level.

Include: why existing solutions don't work (or why you're building instead of buying), what the end state looks like when this system is working well, and any key insight or approach that makes this system's approach distinctive.

### 2. Users & Stakeholders

Who uses this system and what do they care about? For each user type, describe their role, how they interact with the system, what they need from it, and what would make them frustrated.

For single-user systems, this section is shorter but still valuable — it forces you to articulate what "you as user" actually need day-to-day vs. what sounds cool in theory.

### 3. Functional Requirements

The heart of the spec. Each requirement should be specific, testable, and traceable. Use a consistent format for every requirement — no exceptions:

```markdown
**FR-001: [Requirement Title]**
The system shall [specific behavior]. [Additional detail or context if needed.]
Acceptance: [How do you know this requirement is met? Testable condition.]
Priority: [Must | Should | Could | Won't]
```

Priorities use the **MoSCoW method**:
- **Must** — Non-negotiable for the current version. The system doesn't ship without these.
- **Should** — Important, expected in the current version, but the system is usable without them.
- **Could** — Desirable if time and resources allow. Nice improvements, not core.
- **Won't** — Explicitly not in this version, but acknowledged for future consideration.

Every requirement gets exactly two fields after the description: `Acceptance` and `Priority`. No other fields (no "Rationale" — the description itself should explain the reasoning). This consistency matters because downstream skills and reviewers depend on a predictable structure.

Group related requirements under headings (e.g., "Trading Execution," "Analysis Pipeline," "User Interface"). The groupings should reflect the user's mental model of the system, not the technical architecture.

### 4. Non-Functional Requirements

Quality attributes that apply across the system: performance, reliability, security, usability, maintainability. Use the same format as functional requirements — same two fields (`Acceptance` and `Priority` using MoSCoW), same structure. The only difference is these describe how the system behaves rather than what it does.

Examples: response time targets, uptime expectations, data retention requirements, accessibility needs, auditability requirements.

### 5. Constraints

Hard boundaries that the solution must work within. These are non-negotiable and come from the environment, not from design choices.

Organize constraints by area of concern when the project spans multiple domains. For example, a system with both hardware and integration constraints should group them clearly:

```markdown
#### Infrastructure Constraints
**C-001: [Constraint Title]**
[Description and why it exists.]

#### Integration Constraints
**C-002: [Constraint Title]**
[Description and why it exists.]

#### Operational Constraints
**C-003: [Constraint Title]**
[Description and why it exists.]
```

If a constraint spans multiple areas of concern, note which areas it affects. The groupings help downstream skills (especially architecture and operations) understand which domain each constraint impacts.

Each constraint should explain what it is and why it exists — the "why" helps future readers understand whether the constraint still applies.

### 6. Scope Boundaries

Explicitly define what's in scope for the current version. This section prevents scope creep and aligns expectations.

**In Scope:** What this system will do in its current version. This should be a clear summary derived from the functional requirements above.

**Out of Scope:** Only include this subsection if the user has explicitly stated things that are out of scope, or if items were discussed and deliberately excluded during the interview phase. Do not infer or assume things are out of scope — if there's ambiguity about whether something is in or out, capture it in the Open Questions section instead and ask for clarification. The product owner decides scope, not the spec writer.

### 7. User Workflows

Walk through the key user workflows step by step. These are the "day in the life" scenarios that show how the system fits into the user's actual routine.

Every workflow uses exactly this structure — no variations:

```markdown
#### [Workflow Name]
**Trigger:** [What initiates this workflow — time of day, event, user action]
**Steps:**
1. [Step]
2. [Step]
...
**Expected Outcome:** [What the user sees/has when the workflow completes successfully]
**What Could Go Wrong:** [Failure scenarios and their consequences]
```

These four fields (`Trigger`, `Steps`, `Expected Outcome`, `What Could Go Wrong`) are the only fields in a workflow. Don't add other fields like "Failure Modes" or "Recovery" — keep it consistent. The "What Could Go Wrong" field captures both what can fail and the user-visible impact.

These workflows are invaluable for architecture and testing — they define the paths that must work correctly.

### 8. Success Criteria

How will you know the system is working? Define measurable or observable criteria that indicate success. These should be things the product owner can evaluate, not internal metrics.

Good success criteria are specific: "I can approve or reject a trade proposal from my phone in under 30 seconds" is better than "the system should be fast."

### 9. Assumptions & Dependencies

What are you assuming to be true? What external factors does the system depend on? Document these so that if an assumption turns out to be wrong, you know which requirements are affected.

Examples: "Das Trader Pro API will remain available and unchanged," "Exchange API rate limits will stay at current levels," "The Mac M1 will remain powered on 24/7."

### 10. Open Questions

Unresolved decisions or unknowns that need answers before (or during) architecture work. For each: what's the question, who needs to answer it, what's the impact of not answering it, and any deadline.

### 11. Changelog

Always included. For initial drafts, the changelog notes the creation of the document and any key decisions made during the interview phase. For revisions, it documents every change between versions.

```markdown
## Changelog

### v1 — [Date]
- Initial draft based on [conversation / interview / prior documentation]
- [Key decisions or assumptions made during drafting]
```

For revisions:

```markdown
## Changelog (vN-1 -> vN)

### Added
- [New requirements, constraints, or workflows]

### Changed
- [Modifications to existing requirements, with explanation]

### Removed
- [Requirements dropped, with explanation]

### Clarified
- [Requirements rewritten for clarity without changing intent]
```

---

## Writing Guidelines

- **Be specific.** "The system should notify the user" is too vague. "The system sends a push notification to the user's phone when a new trade proposal requires approval" is specific enough to build from.
- **Use the user's language.** If the product owner says "shadow mode," use "shadow mode" — don't rename it to "simulation mode" for the spec. Consistency with how people actually talk about the system prevents confusion.
- **Distinguish requirements from solutions.** "Must support human approval before changes go live" is a requirement. "Use a three-gate pipeline" is a solution. Requirements belong here; solutions belong in the architecture blueprint.
- **Every requirement should be testable.** If you can't describe how to verify a requirement is met, it's too vague. Refine it until you can.
- **Include the 'why' for constraints.** A constraint without context becomes confusing when circumstances change. "No new hardware purchases (budget is $0 for infrastructure)" explains itself; "No new hardware purchases" leaves the reader guessing.

---

## Output

Produce the product spec as a single Markdown file. The filename should follow the pattern:
`[project-name]-product-spec-v[N].md`

Save the file and present it to the user.

