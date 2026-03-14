---
name: test-plan
description: >
  Phase 5 of the SDLC skill chain. Produces project-level test plan documents defining test layers, tooling,
  thresholds, validation gates, and test commands. Triggers: 'test plan', 'testing strategy', 'test suite',
  'QA plan', 'validation', 'regression tests', 'how do we test this', 'write the test plan', 'define the
  test layers', 'what should we test'. Defines HOW to test (tools, patterns, coverage targets) — individual
  test cases are derived from ticket acceptance criteria during implementation. Chain: product-spec,
  architecture-blueprint, implementation-spec, tickets, test-plan.
---

# Test Plan Skill

This skill produces test plan documents that define how a system is validated — what gets tested, how it's tested, what passes, and what fails. The test plan ensures that the system actually does what the product spec says it should, that the architecture holds together under real conditions, and that changes don't break things.

## Where This Fits in the SDLC

This is **Phase 5** of the skill chain:

1. **Product Spec** — What are we building and why?
2. **Architecture Blueprint** — How is the system designed?
3. **Implementation Spec** — What are the rules and interfaces for the codebase?
4. **Tickets** — What are the individual units of work?
5. **Test Plan** (this skill) — How do we validate it?

The test plan draws from all upstream documents:
- **Product documentation** provides the acceptance criteria — the "what success looks like" that tests must verify. This lives across PRODUCT.md (project-level success criteria), FEATURES.md (feature index), and individual feature docs in features/ (per-feature requirements and acceptance criteria).
- **Architecture blueprint** provides the component boundaries and data flows — the seams where integration tests live.
- **Implementation spec** provides the module interfaces and build phases — the contracts that unit tests verify.

### What Belongs Here vs. Other Skills

**In this skill (validation):**
- Test layers and their purposes (unit, integration, regression, E2E, etc.)
- Test tooling and frameworks
- Test commands and how to run them
- Acceptance criteria and pass/fail thresholds
- Backtest and simulation approaches
- Data requirements for testing (fixtures, mocks, historical data)
- Test coverage expectations
- Continuous validation (what runs on every change vs. periodically)

**Not in this skill:**
- System design (architecture blueprint)
- Code structure and build order (implementation spec)
- Deployment and CI/CD pipelines (tickets — as implementation work)

---

## Core Principles

**Every requirement is testable.** For each requirement in the feature docs (features/*.md), there should be at least one test that verifies it's met. If a requirement can't be tested, it's either too vague (refine it) or it's aspirational (acknowledge it as such).

**Test at the right level.** Unit tests for logic, integration tests for component interactions, end-to-end tests for user workflows. Testing everything at the E2E level is slow and brittle. Testing everything at the unit level misses integration bugs. The test plan defines where each type of validation belongs.

**Thresholds over feelings.** Where possible, define numeric thresholds for pass/fail. "Performance should be good" isn't testable. "Sharpe ratio > 1.0 over 30-day out-of-sample" is. For subjective qualities, define what "good enough" looks like in observable terms.

**Previous plans are sacred.** Same principle as all SDLC skills — nothing gets silently dropped when updating.

---

## Handling Previous Plans

Same approach as other skills: read completely, inventory, merge forward, changelog.

---

## Test Plan Structure

### Document Header

```markdown
# [Project Name] — Test Plan

**Version:** [vN]
**Date:** [Date]
**Previous Version:** [version + date, or "N/A — initial draft"]
**Status:** [Draft | Review | Approved]
**Author(s):** [Who wrote/revised this]
**Architecture Blueprint:** [Reference to blueprint version]
**Implementation Spec:** [Reference to implementation spec version]
```

### Section Order

1. Testing Overview
2. Test Layers
3. Test Tooling & Frameworks
4. Test Data & Fixtures
5. Acceptance Criteria & Thresholds
6. Validation Suite (Automated Gates)
7. Manual Validation
8. Test Commands Reference
9. Coverage Expectations
10. Regression Strategy
11. Open Questions
12. Changelog

---

## Section Details

### 1. Testing Overview

A brief summary (2-3 paragraphs) of the overall testing approach. What's the testing philosophy for this project? What are the highest-risk areas that need the most validation? What's the balance between automated and manual testing?

### 2. Test Layers

Define each layer of testing, what it validates, and where it fits. For each layer:

```markdown
### [Layer Name] (e.g., Unit Tests)

**Purpose:** [What this layer validates]
**Scope:** [What's included — which modules, which behaviors]
**Location:** [Where these tests live in the codebase]
**Runs When:** [On every commit? Nightly? On demand? As part of a gate?]
**Expected Runtime:** [How long a full run takes]

**What to Test:**
- [Specific category of tests for this layer]
- [Another category]

**What NOT to Test at This Layer:**
- [Things that belong at a different layer, and why]
```

Common layers (adapt to the project):
- **Unit Tests** — Individual functions and classes in isolation
- **Integration Tests** — Component interactions across module boundaries
- **Regression Tests** — Full sweep of all strategies/features to catch unintended side effects
- **Backtests / Simulations** — Historical data validation (especially relevant for trading, ML, etc.)
- **Shadow / Paper Testing** — Live-data validation without real consequences
- **End-to-End Tests** — Full workflow tests from input to output

### 3. Test Tooling & Frameworks

What tools are used for testing and why. Include:
- Test runner (pytest, jest, etc.)
- Mocking/stubbing libraries
- Assertion libraries
- Coverage tools
- Any custom test utilities or harnesses

For each tool, note the version and any configuration that matters (pytest flags, config files, etc.).

### 4. Test Data & Fixtures

What data do tests need, and where does it come from?

- **Fixtures** — Static test data bundled with the tests. What formats, where they live, how they're maintained.
- **Mocks** — What external dependencies are mocked (APIs, databases, exchanges) and how.
- **Historical Data** — For backtests: where historical data comes from, how much is needed, how it's refreshed.
- **Synthetic Data** — Any generated test data, how it's produced, and what scenarios it covers.

If certain tests require specific environment setup (API keys, network access, running services), document those requirements clearly so a developer knows which tests they can run locally vs. which require a specific environment.

### 5. Acceptance Criteria & Thresholds

The numeric and observable criteria that determine whether the system passes. Map these back to specific requirements in the feature docs (e.g., "FR-003 from features/risk-alerts.md").

For quantitative systems (trading, ML, analytics):
```markdown
| Metric | Threshold | Measured Over | Source |
|--------|-----------|---------------|--------|
| Sharpe Ratio | > 1.0 | 30-day OOS | Backtester |
| Win Rate | > 55% | 30-day OOS | Backtester |
| Max Drawdown | < 10% | Any period | Risk manager |
```

For general systems:
```markdown
| Criterion | Threshold | How Measured |
|-----------|-----------|-------------|
| API response time | < 200ms p95 | Load test |
| Uptime | > 99.5% | Monitoring |
| Zero data loss | All events in audit trail | Audit verification |
```

### 6. Validation Suite (Automated Gates)

If the system has automated validation gates (like a CI pipeline or the three-gate system in a trading bot), define exactly what runs at each gate:

For each gate or validation checkpoint:
- What tests run
- What thresholds must be met
- What happens on pass vs. fail
- How results are reported

This section is the contract for what "validated" means in automated contexts.

### 7. Manual Validation

Not everything can be automated. Define what requires human judgment:
- User experience reviews
- Dashboard visual checks
- Notification content review
- Shadow trading evaluation (where human judgment decides promote/reject)

For each manual validation step, define what the reviewer should look for and what "good" looks like.

### 8. Test Commands Reference

A quick-reference table of all test commands:

```markdown
| What | Command | Expected Runtime |
|------|---------|-----------------|
| All unit tests | `pytest tests/unit/ -v` | ~30s |
| Integration tests | `pytest tests/integration/ -v` | ~2m |
| Regression (all strategies) | `pytest tests/regression/ -v` | ~5m |
| Backtest (specific strategy) | `python -m backtester --strategy <name>` | ~10m |
| Full validation suite | `make validate` | ~20m |
```

### 9. Coverage Expectations

What level of test coverage is expected, and what does "coverage" mean for this project:
- Line coverage targets (if applicable)
- Branch coverage targets (if applicable)
- Which modules must have coverage vs. which are exempt (generated code, config, etc.)
- How coverage is measured and reported

### 10. Regression Strategy

How the project prevents regressions:
- What constitutes a regression test vs. a unit test
- When regression tests run (every commit? every PR? nightly?)
- How new regression tests are added when bugs are found
- Cross-component regression: how changes to shared code are validated across all consumers

### 11. Open Questions

Testing-specific unknowns: gaps in test coverage, undecided tooling choices, data availability issues, etc.

### 12. Changelog

Always included, even for v1 initial drafts.

For v1 (initial draft):
```markdown
## Changelog

### v1 — Initial Draft
- Initial test plan created from [source: architecture blueprint vN, implementation spec vN, etc.]
- [Note any major assumptions or gaps in test coverage]
```

For subsequent versions, use the standard Added / Changed / Removed / Clarified categories.

---

## Writing Guidelines

- **Be specific about commands.** A developer should be able to copy-paste test commands and run them. Include flags, paths, and any required environment setup.
- **Connect tests to requirements.** Where possible, reference which feature doc requirement (e.g., "FR-003 from features/risk-alerts.md") or architecture component each test layer validates. This traceability makes it clear why each test exists.
- **Define "pass" precisely.** For every threshold, specify the exact metric, the exact threshold, and the exact data source. Ambiguous pass/fail criteria lead to arguments and skipped tests.
- **Acknowledge what's not tested.** Every system has gaps in test coverage. It's better to document them explicitly than to pretend they don't exist.

---

## Output

Produce the test plan as a single Markdown file. The filename should follow the pattern:
`[project-name]-test-plan-v[N].md`

Save the file and present it to the user.

---

## Downstream Handoff

The test plan is the final phase of the SDLC skill chain. Do not suggest or invoke additional skills — the SDLC router (or the user) determines if the chain is complete. When this phase finishes, the project has everything needed to begin development: a product spec defining what to build, an architecture blueprint defining the design, an implementation spec defining code conventions and interfaces, tickets defining the work items, and this test plan defining how to validate it all works.
