---
name: code-reviewer
description: "Reviews code produced by the ticket-implementer skill. Runs tests, checks adherence to implementation spec conventions, evaluates test coverage, identifies security and performance concerns, and produces a structured review report. Optionally adds PR comments when GitHub access is available. Triggers: 'review this PR', 'review the code', 'check this implementation', 'code review', 'is this ready to merge', 'review T-004', 'look at this branch', 'review before merge'. Use after a ticket has been implemented and before merging."
---

# Code Reviewer

This skill reviews code produced by the ticket-implementer (or any developer) before it gets merged. It's the quality gate between "code is written" and "code is in main."

The reviewer works from a feature branch in an existing repository. It expects the branch to contain implementation work for a specific ticket, with an associated PR description or ticket file describing what was built and why.

## Why This Skill Exists

Without structured review guidance, a code review tends to focus on surface-level style issues while missing deeper problems — security holes, missing edge cases, spec violations, inadequate tests. This skill enforces a comprehensive review checklist so that every PR gets the same level of scrutiny regardless of who (or what) reviews it.

## The Review Cycle

Every review goes through five phases:

1. **Load Context** — Understand what was built and what the expectations are
2. **Run Verification** — Execute tests and linters to establish a factual baseline
3. **Deep Review** — Inspect the code against multiple quality dimensions
4. **Produce Report** — Write a structured review with findings and verdict
5. **Deliver Feedback** — Add PR comments (if GitHub available) and save the report

---

## Phase 1: Load Context

Before reading any code, understand what the code is supposed to do.

### 1.1 Read the Ticket

Find and read the ticket file that this PR implements. The ticket contains:
- Acceptance criteria (the primary checklist for this review)
- Dependencies (which tickets this one builds on)
- Implementation notes (constraints and interfaces the code must satisfy)

If the PR description references a ticket ID (e.g., `[T-006]`), find the ticket file in the project's ticket directory.

### 1.2 Read the PR Description

The PR description (either on GitHub or in a `PR_DESCRIPTION.md` file) tells you:
- What was changed and why
- Which acceptance criteria the author claims are met
- What tests were added
- Any decisions or follow-up items

Compare the PR description against the ticket. Flag any acceptance criteria not mentioned.

### 1.3 Read the Implementation Spec

Find the project's implementation spec (`*-implementation-spec-v*.md`). This is the convention rulebook. During review, you'll check the code against these conventions:
- File placement and naming
- Error handling patterns
- Import boundaries
- Type annotation requirements
- Logging and security rules

### 1.4 Read the Architecture Blueprint (if needed)

Read the architecture doc when the PR introduces:
- A new component or service
- Changes to data flow between modules
- Database schema modifications
- API contract changes

This lets you verify the implementation matches the designed architecture.

---

## Phase 2: Run Verification

Establish facts before forming opinions.

### 2.1 Check Out the Branch

Switch to the feature branch being reviewed:

```bash
git checkout [branch-name]
```

### 2.2 Install Dependencies

Ensure dependencies are current:

```bash
# Backend
cd backend && pip install -r requirements.txt

# Frontend
cd frontend && npm install
```

If dependency installation fails due to environment constraints, note it and proceed with code inspection only.

### 2.3 Run Tests

Execute the full test suite for the affected area:

```bash
# Backend
cd backend && python -m pytest -v --tb=short 2>&1

# Frontend
cd frontend && npm test 2>&1
```

Record:
- Total tests run
- Tests passed / failed / skipped
- Any error output from failures
- Whether new tests were included in the run

If tests cannot be executed (missing dependencies, environment issues), note this explicitly in the review report. This is a limitation, not a pass — the review proceeds with code inspection.

### 2.4 Run Linters

Run the project's configured linters:

```bash
# Python
ruff check . 2>&1 || flake8 . 2>&1 || echo "No Python linter configured"

# JavaScript/TypeScript
npx eslint . 2>&1 || echo "No JS linter configured"
```

Record any lint violations.

---

## Phase 3: Deep Review

This is the core of the review. Check the code across six dimensions.

### 3.1 Acceptance Criteria Verification

Go through each acceptance criterion from the ticket:

For each criterion:
1. Is there code that implements it?
2. Is there at least one test that verifies it?
3. Does the test actually assert the right thing (not just run without errors)?

Mark each criterion as: **Met**, **Partially Met** (code exists but test is missing or weak), or **Not Met**.

### 3.2 Spec Adherence

Compare the code against the implementation spec conventions:

- **File placement**: Are new files in the correct directories per the repo layout?
- **Naming**: Do functions, classes, variables follow the project's conventions?
- **Error handling**: Does the code use the project's error patterns (custom exceptions, error response format)?
- **Imports**: Are import boundaries respected (no circular imports, no reaching into private modules)?
- **Types**: Are type hints / types present where the project requires them?
- **Logging**: Does the code log appropriately (and never log secrets)?

Flag any deviations. Minor style issues get a "nit" label. Convention violations that could cause downstream problems are flagged as "must fix."

### 3.3 Test Quality

Evaluate the tests themselves:

- **Coverage**: Does every acceptance criterion have at least one test?
- **Edge cases**: Are error paths tested (invalid input, missing data, unauthorized access)?
- **Assertions**: Do tests make meaningful assertions (not just "no exception thrown")?
- **Independence**: Can tests run in any order without side effects?
- **Naming**: Do test names describe what they verify?
- **Fixtures**: Are test fixtures used correctly (not duplicating setup code)?

A test suite that only tests happy paths is insufficient. Flag missing edge case coverage.

### 3.4 Security Review

Check for common security issues relevant to the change:

- **Input validation**: Is user input validated and sanitized before use?
- **Authentication/Authorization**: Do endpoints check permissions? Are there missing auth guards?
- **Secret handling**: Are passwords, tokens, or keys ever logged, hardcoded, or exposed in responses?
- **SQL injection**: Are queries parameterized (not string-concatenated)?
- **Data isolation**: In multi-user contexts, can one user access another's data?
- **Dependency safety**: Are new dependencies from trusted sources with no known vulnerabilities?

Flag any security concern as "must fix" regardless of severity — security issues don't get "nit" labels.

### 3.5 Performance Review

Check for performance issues that could cause problems at scale:

- **N+1 queries**: Are there loops that issue individual DB queries instead of batch queries?
- **Missing indexes**: Do new query patterns need database indexes?
- **Unbounded results**: Are list endpoints paginated? Could a query return millions of rows?
- **Memory**: Are large datasets loaded entirely into memory when streaming would work?
- **Blocking operations**: Are there synchronous I/O calls in async code paths?

Only flag genuine concerns — not theoretical optimizations for code that handles 10 records.

### 3.6 Scope Check

Verify the PR stays within the ticket's boundaries:

- Does the PR include changes not related to the ticket?
- Are there "drive-by" refactors that weren't requested?
- Is there code that implements features from future tickets?
- Are there unnecessary files (documentation, configs, helpers not used by this ticket)?

Scope creep makes reviews harder and increases merge conflict risk. Flag out-of-scope changes.

---

## Phase 4: Produce Report

Write a structured review report. The report is the primary deliverable.

### Report Format

```markdown
# Code Review: [Ticket ID] — [Ticket Title]

**Branch:** [branch name]
**Reviewer:** Code Reviewer (automated)
**Date:** [date]
**Verdict:** [APPROVE | REQUEST CHANGES | BLOCKED]

---

## Summary

[2-3 sentence overview of the implementation quality and key findings]

---

## Test Results

- **Tests run:** [count]
- **Passed:** [count]
- **Failed:** [count]
- **Skipped:** [count]
- **Lint violations:** [count or "clean"]

---

## Acceptance Criteria

| # | Criterion | Status | Test | Notes |
|---|-----------|--------|------|-------|
| 1 | [criterion text] | Met / Partially Met / Not Met | [test name] | [notes] |
| 2 | ... | ... | ... | ... |

---

## Findings

### Must Fix

[Numbered list of issues that must be resolved before merge. Each finding includes:
- What: description of the issue
- Where: file and line reference
- Why: why this matters (security risk, spec violation, missing coverage, etc.)
- Suggestion: how to fix it]

### Should Fix

[Issues that should be addressed but aren't merge-blockers. Same format as Must Fix.]

### Nits

[Minor style or preference issues. Brief — just file:line and the suggestion.]

---

## Security

[Summary of security review. Either "No security concerns identified" or a list of findings.]

---

## Performance

[Summary of performance review. Either "No performance concerns identified" or a list of findings.]

---

## Scope

[Either "PR stays within ticket scope" or a list of out-of-scope changes found.]

---

## Verdict Rationale

[Explain the verdict:
- APPROVE: All acceptance criteria met, tests pass, no must-fix issues
- REQUEST CHANGES: Has must-fix issues that need to be addressed
- BLOCKED: Cannot complete review (tests won't run, dependencies missing, ticket unclear)]
```

### Verdict Rules

- **APPROVE** when: All acceptance criteria are Met, all tests pass, no must-fix findings, no security concerns
- **REQUEST CHANGES** when: Any acceptance criterion is Not Met, or there are must-fix findings, or tests fail
- **BLOCKED** when: Cannot determine correctness (tests won't install/run, missing context, ambiguous ticket)

Be honest with the verdict. A borderline PR gets REQUEST CHANGES, not APPROVE. It's easier to fix issues before merge than after.

---

## Phase 5: Deliver Feedback

### 5.1 Save the Review Report

Save the report as a markdown file:

```
[repo-root]/reviews/[ticket-id]-review.md
```

Create the `reviews/` directory if it doesn't exist.

### 5.2 Add PR Comments (if GitHub available)

If the PR exists on GitHub and `gh` CLI is available:

1. For each **must-fix** finding, add an inline comment at the relevant file and line:
   ```bash
   gh pr review [PR-number] --comment --body "[finding description and suggestion]"
   ```

2. Submit the review with the appropriate verdict:
   ```bash
   # For APPROVE
   gh pr review [PR-number] --approve --body "All acceptance criteria met. See full review report at reviews/[ticket-id]-review.md"

   # For REQUEST CHANGES
   gh pr review [PR-number] --request-changes --body "[summary of must-fix items]. See full review report at reviews/[ticket-id]-review.md"
   ```

If `gh` is not available or the PR doesn't exist on GitHub, skip this step. The review report file is the primary output.

### 5.3 Save the Review Report

**When running standalone** (not inside the implementation-loop): commit the review report to the feature branch so it's part of the PR history:

```bash
git add reviews/
git commit -m "[T-XXX] Add code review report

Verdict: [APPROVE/REQUEST CHANGES/BLOCKED]
Findings: [count] must-fix, [count] should-fix, [count] nits"
```

**When running inside the implementation-loop**: do NOT commit to the feature branch. Write the review report file but leave it uncommitted. The orchestrator manages all branch operations to prevent conflicts between parallel agents. Report the review verdict and file path back to the orchestrator.

---

## Edge Cases and Guidance

### When the ticket file can't be found

If the PR references a ticket ID but the ticket file is missing from the repo, check:
- The PR description (it may contain the acceptance criteria inline)
- The TRACKER.md (it has ticket summaries)

If you still can't determine what the code is supposed to do, issue a BLOCKED verdict.

### When tests pass but code is wrong

Tests passing doesn't mean the code is correct — tests could be testing the wrong thing. Always read the test assertions critically. A test that creates a user and asserts `response.status_code == 200` without checking the response body isn't verifying much.

### When you disagree with the architecture

The reviewer's job is to check that the implementation matches the spec, not to redesign the system. If you spot a fundamental architecture concern, note it under "Should Fix" with a suggestion to discuss it — but don't block the PR for an architecture preference.

### When the PR is too large

If the PR touches more than ~15 files or changes more than ~500 lines, flag this under Scope. Large PRs are hard to review thoroughly. Suggest splitting if the changes are logically separable.

### When reviewing code that depends on unfinished work

If the ticket depends on code from another ticket that's stubbed or mocked, that's expected. Verify the stubs match the interfaces defined in the implementation spec. Don't flag stubs as "incomplete code."

### When the same issue appears multiple times

Don't repeat the same finding for every occurrence. State the issue once, list all locations where it appears, and count it as a single finding.
