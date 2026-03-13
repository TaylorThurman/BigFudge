---
name: ticket-implementer
description: "Implements a single development ticket from the SDLC skill chain. Reads the ticket, loads project context (implementation spec, CLAUDE.md, architecture), writes the code and tests, verifies tests pass, updates TRACKER.md, and creates a PR. Triggers: 'implement this ticket', 'pick up T-001', 'work on this story', 'start coding', 'implement the next ticket', 'build this feature', 'pick up the next task'. Use this skill whenever a developer or agent needs to take a ticket from the backlog and turn it into working, tested code with a pull request."
---

# Ticket Implementer

This skill turns a single SDLC ticket into working, tested code with a pull request. It's the bridge between planning (the SDLC chain) and code that runs.

The implementer works within an existing repository. It expects the repo to have at least a basic project structure in place — it doesn't bootstrap from scratch. Foundation tickets (project scaffolding, database setup) should be completed before handing off feature tickets to this skill.

## Why This Skill Exists

Without structured guidance, a coding agent receiving a ticket might jump straight to writing code, skip tests, ignore the project's conventions, or miss acceptance criteria. This skill enforces a disciplined workflow: understand the context first, then code, then test, then deliver. Every ticket follows the same cycle, which keeps quality consistent across dozens or hundreds of tickets.

## The Implementation Cycle

Every ticket goes through five phases:

1. **Load Context** — Read the ticket and the project's reference documents
2. **Verify Prerequisites** — Check that dependencies are met before starting
3. **Implement** — Write the code following project conventions
4. **Verify** — Run tests, check acceptance criteria
5. **Deliver** — Update tracker, commit, create PR

---

## Phase 1: Load Context

Before writing any code, read these documents in order:

### 1.1 Read the Ticket

The ticket file is the primary input. It contains:
- What to build (scope and description)
- Acceptance criteria (how to verify the work is correct)
- Dependencies (which tickets must be complete first)
- Implementation notes (hints, constraints, relevant interfaces)

Read the entire ticket. Understand what "done" looks like before writing a single line.

### 1.2 Read the Implementation Spec

Find the project's implementation spec (`*-implementation-spec-v*.md` in the project directory). This is the coding rulebook — it defines:
- Repository layout and where new files go
- Module interfaces and contracts between components
- Naming conventions, error handling patterns, validation rules
- The CLAUDE.md / developer context section

If the implementation spec has a CLAUDE.md section, treat those rules as hard constraints. They exist because previous work established patterns that downstream code depends on.

### 1.3 Read the Architecture Blueprint (if needed)

You don't always need the full architecture doc. Read it when:
- The ticket introduces a new component or service
- The ticket involves data flow between multiple modules
- The ticket touches infrastructure (database schema, API contracts, deployment)
- You're unsure how the piece you're building fits into the larger system

Skip it when the ticket is straightforward and the implementation spec provides enough context (e.g., adding validation logic to an existing module).

### 1.4 Read Existing Code

Before adding new code, understand what's already there. Look at:
- The module you're modifying (read the existing files)
- Adjacent modules that your code will interact with (read their interfaces)
- Existing tests (understand the test patterns and fixtures in use)

This prevents duplicate code, inconsistent patterns, and broken integrations.

---

## Phase 2: Verify Prerequisites

### 2.1 Check Dependency Status

Read `TRACKER.md` and verify that every ticket listed as a dependency for this ticket is marked `Done`. If any dependency is not `Done`, stop and report the blocker. Do not proceed — implementing out of order creates integration problems that are harder to fix than waiting.

Format the blocker report clearly:

```
BLOCKED: Cannot implement [this ticket ID] — [this ticket title]

Missing dependencies:
- [dependency ticket ID]: [title] — currently [status]

These must be completed first. The dependency exists because [brief explanation from the ticket].
```

### 2.2 Verify the Repo Builds

Run the project's build/check command before making changes. If the repo is already broken, that's not your problem to fix — report it and stop. You need a clean baseline to work from.

For a typical project:
- Backend: `cd backend && pip install -r requirements.txt && python -m pytest --co -q` (collect tests without running)
- Frontend: `cd frontend && npm install && npm run build`

If the build commands aren't obvious, check the implementation spec's CLAUDE.md section or look for a Makefile, package.json scripts, or similar.

---

## Phase 3: Implement

### 3.1 Plan Before Coding

Before writing code, think through:
- Which files need to be created or modified?
- What's the order of operations? (e.g., models before services, services before routes)
- Are there any decisions the ticket leaves open? If so, choose the simplest option that satisfies the acceptance criteria.

### 3.2 Follow Project Conventions

The implementation spec defines the rules. Common things to watch for:
- **Naming**: Follow the established patterns (snake_case, camelCase, whatever the project uses)
- **File placement**: Put new files where the repo layout says they go
- **Error handling**: Use the project's error patterns (custom exception classes, error response format)
- **Imports**: Follow the import ordering and boundary rules
- **Types**: Use type hints / TypeScript types if the project does

When in doubt, look at how existing code in the same module handles similar concerns. Consistency with the existing codebase is more important than theoretical best practices.

### 3.3 Write Tests Alongside Code

Tests are not optional. Every ticket must include tests that verify its acceptance criteria. Write tests as you go, not as an afterthought.

**What to test:**
- Each acceptance criterion from the ticket should map to at least one test
- Edge cases mentioned in the ticket's implementation notes
- Error paths (invalid input, missing data, unauthorized access)

**Test patterns:**
- Follow the existing test structure (look at how other tests in the project are organized)
- Use the project's test fixtures and helpers
- Name tests descriptively — someone reading the test name should understand what it verifies

**What NOT to test:**
- Don't test framework behavior (e.g., don't test that FastAPI returns 404 for unknown routes)
- Don't test other tickets' functionality
- Don't write tests that duplicate existing coverage

### 3.4 Keep Scope Tight

Implement exactly what the ticket asks for. Resist the urge to:
- Refactor adjacent code that "could be better"
- Add features not in the acceptance criteria
- Optimize prematurely
- Fix unrelated bugs you notice

If you spot something worth addressing, note it in the PR description as a follow-up item. Scope creep in tickets causes merge conflicts, review delays, and broken integrations.

---

## Phase 4: Verify

### 4.1 Run All Tests

Run the full test suite for the affected area, not just your new tests:

```bash
# Backend example
cd backend && python -m pytest -v

# Frontend example
cd frontend && npm test
```

All tests must pass — both your new ones and the existing ones. If an existing test breaks, you've introduced a regression. Fix it before proceeding.

### 4.2 Walk Through Acceptance Criteria

Go through each acceptance criterion in the ticket one by one. For each:
1. Verify there's at least one test covering it
2. Verify the test passes
3. If the criterion involves something that can't be automated (UI appearance, performance feel), note it as needing manual verification in the PR description

### 4.3 Lint and Format

Run the project's linter and formatter if configured:

```bash
# Common patterns
ruff check . && ruff format .     # Python
eslint . && prettier --write .     # JavaScript/TypeScript
```

Fix any issues. Don't submit code with lint warnings.

---

## Phase 5: Deliver

### 5.1 Update TRACKER.md

Change the ticket's status from `Todo` (or `In Progress`) to `Done` in TRACKER.md. Use the exact status values the tracker defines (check the tracker's own conventions — some use `Done`, others `Complete`, others checkmarks).

### 5.2 Create a Feature Branch and Commit

Branch naming convention: `ticket/[ticket-id]-[brief-description]`

Examples:
- `ticket/T-006-expense-service`
- `ticket/T-012-frontend-auth-ui`

Commit with a clear message that references the ticket:

```
[T-006] Implement expense service layer

- Add ExpenseService with create, list, get, update, delete
- Add input validation for amount (positive, two decimal places)
- Add category validation against allowed categories
- Add date validation (not future-dated)
- Add per-user authorization (users only see own expenses)
- Tests: 14 new tests covering all acceptance criteria
```

### 5.3 Push and Create PR

Push the branch and create a pull request. The PR should include:

**Title**: `[T-XXX] Brief description of what was implemented`

**Body**:
```
## Ticket
[Link to or quote of the ticket]

## Changes
- What was added/modified (bullet points, brief)

## Acceptance Criteria Verification
- [ ] [Criterion 1] — verified by [test name or manual check]
- [ ] [Criterion 2] — verified by [test name or manual check]
...

## Tests Added
- [test file]: [what it covers]

## Notes
- Any decisions made that weren't specified in the ticket
- Any follow-up items noticed during implementation
```

---

## Edge Cases and Guidance

### When the ticket is ambiguous

If the ticket doesn't specify something you need to know (e.g., exact error message format, specific validation rule), check the implementation spec first. If it's not there either, choose the simplest reasonable option and document your choice in the PR description under "Notes."

### When you discover a bug in existing code

Don't fix it in this ticket's PR. Note it in the PR description as a follow-up. Mixing bug fixes with feature work makes reviews harder and history unclear.

### When tests are hard to write

Some things are genuinely hard to test (file system interactions, time-dependent behavior, external API calls). Use the project's existing mocking patterns. If none exist, use standard mocking (`unittest.mock` for Python, `jest.mock` for JavaScript). Don't skip the test — write the best test you can.

### When the ticket depends on an external service

If the ticket involves an external service (database, API, cloud resource), check whether the project has test fixtures or mocks for it. Use those. Don't make tests that require a running external service unless the project explicitly sets that up in its test infrastructure.

### When you're unsure about scope

The ticket's acceptance criteria are your scope boundary. If something isn't in the acceptance criteria, it's not in scope. If you think the acceptance criteria are incomplete, note it in the PR description — don't silently expand scope.
