---
name: architecture-blueprint
description: >
  Phase 2 of the SDLC skill chain. Produces comprehensive architecture blueprint documents with component design,
  data flow, state machines, security model, and Mermaid diagrams. Triggers: 'blueprint', 'architecture doc',
  'system design', 'technical spec', 'document the architecture', 'write up the design', 'how is it designed'.
  Use when the user is discussing components, pipelines, infrastructure, or design decisions and wants to
  formalize them. Scoped to system design — use implementation-spec for code structure, test-plan for testing.
  Chain: product-spec, architecture-blueprint, implementation-spec, tickets, test-plan.
---

# Architecture Blueprint Skill

This skill produces comprehensive architecture blueprint documents in Markdown. Blueprints capture the design of a system — its components, data flow, state machines, infrastructure, security model, and the decisions that shaped them.

## Where This Fits in the SDLC

This is **Phase 2** of the skill chain:

1. **Product Spec** — What are we building and why?
2. **Architecture Blueprint** (this skill) — How is the system designed?
3. **Implementation Spec** — What are the rules and interfaces for the codebase?
4. **Tickets** — What are the individual units of work?
5. **Test Plan** — How do we validate it?

The blueprint consumes the product spec and answers: given these requirements and constraints, how should the system be structured? Every design decision should trace back to a requirement in the product spec. The blueprint then feeds the implementation spec (which details how to build what's designed here), tickets (which break the work into deliverable slices), and the test plan (which validates the design works).

### What Belongs Here vs. Other Skills

**In this skill (design decisions):**
- Component relationships and responsibilities
- Data flow and pipeline architecture
- State machines and lifecycle rules
- Infrastructure topology and network layout
- Security model and credential boundaries
- Configuration schema and propagation
- Safety guardrails and approval workflows
- Design rationale and decision log

**In implementation-spec (build details):**
- Build phases and implementation order
- Code structure and repo layout
- Module interfaces and code samples
- CLAUDE.md and developer onboarding
- Dependency lists and requirements files

**In test-plan (validation):**
- Test layers (unit, integration, regression)
- Backtest and validation suites
- Acceptance criteria and thresholds
- Test tooling and commands

When writing the blueprint, you will naturally touch on topics that belong to other skills — that's fine. The key is to describe the *design intent* here and leave the *implementation detail* to the appropriate downstream skill. For example: "The crypto bot runs as a persistent service" belongs in the blueprint. Deployment procedures, CI/CD pipelines, and infrastructure-as-code belong in tickets as implementation work.

---

## Core Principles

**Completeness over brevity.** A blueprint's job is to be the single source of truth for how a system is *designed*. Every section should contain enough detail that someone unfamiliar with the project could understand the system's architecture and begin reasoning about it. When in doubt, include more detail rather than less.

**Previous blueprints are sacred.** When a prior version exists, every piece of information it contains must be accounted for in the new version. Nothing gets silently dropped. If something was removed or changed, the changelog must explain why. This prevents the slow erosion of institutional knowledge that happens when documents get rewritten from scratch.

**Diagrams are mandatory and always use Mermaid.** Every blueprint includes Mermaid diagrams for visual clarity. All diagrams must use Mermaid syntax — no ASCII art, no plaintext boxes, no alternative formats. At minimum: a system/component diagram and a data flow diagram. Additional diagrams (state machines, sequence diagrams, network topology) should be included wherever they add understanding. Even for simple systems or sparse inputs, the Mermaid requirement is non-negotiable.

**Trace to requirements.** When a product spec exists, design decisions should reference the requirements they satisfy. This traceability ensures nothing is designed without a reason and no requirement is left unaddressed.

---

## Handling Previous Blueprints

Before writing anything, determine whether a previous blueprint exists. Check for:
1. An uploaded `.md` file in the conversation
2. Blueprint content pasted directly into the conversation
3. References to a previous version ("update the blueprint", "here's what we had", "revise v3")

### If a previous blueprint exists:

1. **Read it completely** before writing a single line of the new version.
2. **Create a section-by-section inventory** of everything the previous blueprint contains. Do this in your thinking — the user doesn't need to see the inventory, but you need it to ensure nothing is lost.
3. **Identify content that belongs in other SDLC skills.** If the previous blueprint contains deployment scripts, test strategies, rollback procedures, or other operational content, note these for migration to the appropriate skill document. Do not silently drop them — flag them in the changelog as "migrated to [skill name]."
4. **Merge forward**: Every *design* detail from the previous version must appear in the new version unless the user has explicitly said to remove or change it.
5. **Generate a Changelog section** at the end of the document.

### If no previous blueprint exists:

Gather as much information as possible from the conversation before writing. If a product spec exists, use it as the primary input. Ask clarifying questions if major sections would be left empty.

---

## Blueprint Structure

Every blueprint follows this structure. All sections are required. If a section is not applicable to the project, include it with a brief explanation of why it doesn't apply — do not silently omit it, because a future reader needs to know the omission was intentional.

### Document Header

```markdown
# [Project Name] — Architecture Blueprint

**Version:** [vN — e.g., v1, v2, v6]
**Date:** [Date of this version]
**Previous Version:** [Date and version number, or "N/A — initial draft"]
**Status:** [Draft | Review | Approved | Superseded]
**Author(s):** [Who wrote/revised this version]
**Product Spec:** [Reference to product spec version, or "N/A"]
```

### Section Order

1. Executive Summary
2. Design Principles & Constraints
3. System Overview
4. Hardware & Infrastructure Topology
5. Component Architecture
6. Data Flow & Pipeline
7. State Machine & Lifecycle Definitions
8. Configuration Management
9. Safety & Guardrails
10. Security Model
11. Dependencies & External Services
12. Notification & Alerting
13. Logging & Audit Trail
14. Scaling & Evolution Notes
15. Decision Log
16. Diagrams
17. Glossary
18. Open Questions & TODOs
19. Changelog

---

## Section Details

What follows is guidance on what each section should contain. Adapt the depth and focus to the project, but every section must be substantive.

### 1. Executive Summary
A concise overview (3-5 paragraphs) of what the system does, why it exists, and how it works at a high level. A reader should understand the system's purpose and general approach from this section alone. Include the core value proposition and the key architectural decisions that define the system.

### 2. Design Principles & Constraints
The foundational rules and constraints that shaped the architecture. These are the "why" behind the design — the things that must remain true for the system to stay coherent.

Separate hard constraints (non-negotiable) from soft principles (preferred but flexible). Explain the reasoning behind each one. If a product spec exists, reference the constraints defined there.

### 3. System Overview
A high-level description of the system's major components and how they relate to each other. This is the 10,000-foot view. Identify each major subsystem, its responsibility, and its interfaces with other subsystems. Include a Mermaid component diagram here.

### 4. Hardware & Infrastructure Topology
Every physical or virtual machine, server, device, or environment the system runs on. For each, document: its role, OS, specs (if relevant), network location, and what services it runs. Include how machines communicate (SSH, HTTP, local network, etc.). Include a Mermaid network/topology diagram.

### 5. Component Architecture
Deep dive into each component or module. For each component: its responsibility, inputs, outputs, key abstractions, and how it interacts with other components. Describe inheritance hierarchies, patterns used (strategy pattern, observer, etc.), and any abstractions that are important to understand.

Focus on the *design* of each component — what it does and why it's structured that way. Detailed code samples and module file listings belong in the implementation spec.

### 6. Data Flow & Pipeline
How data moves through the system from input to output. Describe each stage of processing, what triggers it, what data is transformed, and where results go. For event-driven or pipeline-based systems, describe the sequence of operations in order. Include timing context (when things run and why). Include a Mermaid sequence or flow diagram.

### 7. State Machine & Lifecycle Definitions
Any entities in the system that have distinct states or modes and rules governing transitions between them. Document each state, valid transitions, guards/conditions on transitions, and what triggers them. Include a Mermaid state diagram for each state machine.

### 8. Configuration Management
How the system is configured at a design level. Describe the configuration strategy: what is configurable, what format configs use, what each key area controls, how configuration changes propagate through the system, and any validation rules or constraints on config values. Describe the schema in prose (e.g., "a top-level `strategies` key contains an array of strategy objects, each with a `name`, `enabled` flag, and `parameters` map") rather than including actual YAML, JSON, TOML, or other code config blocks. Actual config files and examples belong in the implementation spec.

### 9. Safety & Guardrails
Protective mechanisms that prevent the system from doing harm. This includes rate limiting, position limits, circuit breakers, validation gates, approval workflows, cool-down periods, and any other mechanism designed to prevent mistakes or contain damage. Explain what each guardrail protects against and how it works at a design level.

### 10. Security Model
How the system handles authentication, authorization, credential storage, network exposure, and data protection. Document every secret the system uses, where it's stored (without revealing actual values), and who/what has access. Describe network boundaries and what's exposed to what.

### 11. Dependencies & External Services
Every external dependency: libraries, APIs, services, brokers, tools. For each, document: what it's used for, how it's accessed, and what happens if it's unavailable. Group by category (runtime dependencies, external services). Failure impact is especially important — it drives design decisions around redundancy and graceful degradation.

### 12. Notification & Alerting
How the system communicates status, events, and alerts to humans. Describe each notification channel, what triggers notifications, what information they contain, and any escalation paths. Focus on the design of the notification system, not the implementation details of specific integrations.

### 13. Logging & Audit Trail
What gets logged, where, and in what format. Describe log schemas, retention approach, what events are captured, and how log integrity is maintained. For audit trails, explain what information is recorded for compliance or debugging purposes.

### 14. Scaling & Evolution Notes
Where the system could grow, what's intentionally deferred, and what would need to change if requirements shift. Document known limitations and the paths to overcome them. Include any future ideas that have been discussed but not yet committed to.

### 15. Decision Log
Key architectural decisions and the reasoning behind them. For each decision, document the context, options considered, what was chosen, and why. This prevents revisiting settled questions and preserves institutional knowledge.

Format:
```markdown
#### [Decision Title]
- **Date:** [When decided]
- **Context:** [What prompted this decision]
- **Options Considered:** [What alternatives were evaluated]
- **Decision:** [What was chosen]
- **Rationale:** [Why this option won]
```

### 16. Diagrams
Collected reference for all diagrams in the document. At minimum:
- **System Component Diagram** — high-level view of all major components
- **Data Flow Diagram** — how data moves through the system end-to-end
- **Network/Infrastructure Topology** — physical or virtual layout of machines and connections

Additional diagrams when applicable: state machine diagrams, sequence diagrams, entity relationship diagrams. All diagrams use Mermaid syntax.

### 17. Glossary
Define every domain-specific term, acronym, or concept used in the document. A reader should never have to guess what a term means.

### 18. Open Questions & TODOs
Unresolved decisions, known gaps, and items that need follow-up. Be specific about what the question is, who needs to answer it, and any deadline or dependency.

### 19. Changelog
Always included, even for v1 initial drafts. Documents the history of the document.

For v1 (initial draft):
```markdown
## Changelog

### v1 — Initial Draft
- Initial architecture blueprint created from [source: product spec vN / conversation / etc.]
- [Note any major assumptions or gaps flagged in Open Questions]
```

For subsequent versions:
```markdown
## Changelog (vN-1 -> vN)

### Added
- [New sections, components, or details]

### Changed
- [Modifications to existing content, with explanation]

### Removed
- [Content intentionally dropped, with explanation]

### Migrated
- [Content moved to another SDLC skill document, with destination]

### Clarified
- [Sections rewritten for clarity without changing meaning]
```

The **Migrated** category is used when content from a previous blueprint has been moved to the implementation spec or test plan rather than dropped.

---

## Writing Guidelines

- **Be specific.** Don't say "the system sends a notification." Say "the orchestrator sends a Discord embed to the #trading-alerts channel containing the strategy name, proposed change, and a link to the dashboard approval card."
- **Use concrete examples in prose.** When describing config schemas, describe them in natural language ("a strategy object has a name string, an enabled boolean, and a parameters map"). When describing data flow, describe a representative scenario. Do not include code blocks for config files, shell commands, or sample payloads — those belong in the implementation spec.
- **Name things consistently.** Pick one name for each component and use it everywhere. Define it in the glossary.
- **Write for the future reader.** Assume the person reading this document has access to the codebase but no context on the design. They need to understand not just *what* the system does, but *why* it does it that way.
- **Prefer prose over bullet lists for complex topics.** Bullet lists are great for enumerating items, but design rationale and system behavior deserve full sentences and paragraphs.
- **Design intent, not implementation detail.** Describe what components do and why they're structured that way. Leave file paths, code samples, deploy scripts, and test commands for the downstream skills.

---

## Output

Produce the blueprint as a single Markdown file. The filename should follow the pattern:
`[project-name]-architecture-blueprint-v[N].md`

Save the file and present it to the user.

---

## Downstream Handoff

The blueprint output is consumed by downstream SDLC phases. Do not suggest or invoke the next skill — the SDLC router (or the user) determines what runs next. For context, the phases that read this document are:
- **Implementation Spec** — Uses component architecture to produce code structure, module interfaces, conventions, and developer context.
- **Tickets** — Uses architecture and implementation spec to produce individual, scoped work tickets organized in vertical slices.
- **Test Plan** — Uses component architecture and safety guardrails to produce test layers, validation suites, and acceptance criteria.
