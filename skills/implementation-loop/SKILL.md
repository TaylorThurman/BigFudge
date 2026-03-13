---
name: implementation-loop
description: "Autonomous implementation loop that picks up tickets from TRACKER.md and builds them in dependency order. For each ticket: runs the ticket-implementer, then the code-reviewer, and if approved, moves to the next ticket. Stops when no tickets are available, a review requests changes twice, or a blocker is hit. Triggers: 'start building', 'implement the tickets', 'build the backlog', 'start the implementation loop', 'pick up available tickets', 'build what's ready', 'run the implementation loop', 'start coding the tickets'. Use after the SDLC planning chain has produced tickets."
---

# Implementation Loop

This skill runs the implementation cycle autonomously. It reads TRACKER.md, finds tickets that are ready to build (dependencies met, status Todo), and for each one: implements it, reviews it, and moves on. It keeps going until there's nothing left to pick up.

The loop is the bridge between "planning is done" and "code is shipped." After the SDLC router produces tickets, this skill turns them into merged PRs without manual intervention.

## Why This Skill Exists

Without it, someone has to manually trigger the ticket-implementer and code-reviewer for each ticket — look at the tracker, figure out what's ready, invoke the implementer, wait, invoke the reviewer, check the verdict, repeat. That's orchestration work that should be automated.

## Prerequisites

Before running the loop, the project must have:
- A TRACKER.md with tickets in dependency order
- At least one ticket with status `Todo` whose dependencies are all `Done`
- An implementation spec (the implementer needs it for conventions)
- A working repo with git initialized

If any of these are missing, the loop reports what's needed and stops.

---

## The Loop

```
┌─────────────────────────────────────┐
│         Read TRACKER.md             │
│    Find next ready ticket           │
└──────────────┬──────────────────────┘
               │
               ▼
        ┌──────────────┐
        │ Any tickets   │──── No ───► Done. Report summary.
        │ ready?        │
        └──────┬───────┘
               │ Yes
               ▼
┌─────────────────────────────────────┐
│     Run ticket-implementer          │
│     on the selected ticket          │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│     Run code-reviewer               │
│     on the implementation           │
└──────────────┬──────────────────────┘
               │
               ▼
        ┌──────────────┐
        │  Verdict?     │
        └──────┬───────┘
               │
      ┌────────┼─────────┐
      ▼        ▼         ▼
   APPROVE  REQUEST    BLOCKED
      │     CHANGES      │
      │        │         ▼
      │        ▼      Stop loop.
      │     Re-implement Report blocker.
      │     with feedback
      │        │
      │        ▼
      │     Re-review
      │        │
      │     ┌──┴──┐
      │  APPROVE  Still fails
      │     │        │
      │     │        ▼
      │     │     Stop loop.
      │     │     Report: needs
      │     │     human review.
      ▼     ▼
   Mark ticket Done in TRACKER.md
   Loop back to top
```

---

## Step 1: Find the Next Ready Ticket

Read TRACKER.md and scan the ticket summary table. A ticket is **ready** when:
- Its status is `Todo`
- ALL tickets listed in its "Depends On" column have status `Done`

If multiple tickets are ready, pick the one that appears first in the table (this follows the natural dependency ordering from the tickets skill).

If no tickets are ready but some are still `Todo`, check why — their dependencies might be `In Progress` or blocked. Report this and stop.

If all tickets are `Done`, the loop is complete. Report the summary and stop.

---

## Step 2: Implement the Ticket

Invoke the ticket-implementer workflow for the selected ticket. Pass it:
- The ticket file path
- The implementation spec path
- The TRACKER.md path
- The repo root

The implementer will:
1. Load context (ticket, spec, existing code)
2. Verify prerequisites (dependency check — should pass since we pre-checked)
3. Write the code and tests
4. Run tests
5. Update TRACKER.md to Done
6. Create a feature branch and commit
7. Write a PR description

Wait for the implementer to complete. If it reports a blocker (e.g., repo won't build, dependency actually missing), stop the loop and report.

---

## Step 3: Review the Implementation

Invoke the code-reviewer workflow on the feature branch. Pass it:
- The ticket file path
- The implementation spec path
- The feature branch name
- The repo root

The reviewer will:
1. Load context (ticket, PR description, spec)
2. Run tests and linters
3. Deep review across all 6 dimensions
4. Produce a structured review report with a verdict

---

## Step 4: Handle the Verdict

### APPROVE

The implementation passed review. Proceed:
1. Verify TRACKER.md shows the ticket as `Done`
2. If git remote is available, push the branch and create the PR (`gh pr create`)
3. If no remote, the branch and PR description file are the deliverables
4. Log the result and loop back to Step 1

### REQUEST CHANGES

The reviewer found must-fix issues. Attempt one self-correction cycle:

1. Read the review report, specifically the **Must Fix** findings
2. Re-invoke the ticket-implementer with additional context: "The code review found these issues: [list of must-fix findings with file/line references]. Fix them on the existing feature branch. Do not rewrite from scratch — address each finding specifically."
3. After the re-implementation, re-invoke the code-reviewer
4. If the second review returns **APPROVE**, proceed as above
5. If the second review still returns **REQUEST CHANGES**, stop the loop and report: "Ticket [ID] failed review twice. The remaining issues need human attention." Include both review reports.

One retry is the limit. More retries risk going in circles. A human should look at persistent review failures.

### BLOCKED

The reviewer couldn't complete the review (tests won't run, missing context, ambiguous ticket). Stop the loop and report the blocker. Don't attempt to fix BLOCKED verdicts automatically — they usually indicate an environment or specification problem.

---

## Step 5: Report

After the loop ends (either all tickets done, or stopped by a blocker/failure), produce a summary:

```markdown
# Implementation Loop Report

**Date:** [date]
**Tickets processed:** [count]
**Tickets remaining:** [count]

## Completed Tickets

| Ticket | Title | Branch | Review Verdict | Notes |
|--------|-------|--------|----------------|-------|
| T-004  | Auth Service | ticket/T-004-auth-service | APPROVE | Clean first pass |
| T-005  | Auth Endpoints | ticket/T-005-auth-endpoints | APPROVE (2nd attempt) | Fixed missing input validation |

## Stopped At

[If the loop stopped early, explain why:]
- **Ticket:** [ID and title]
- **Reason:** [blocker description, or "failed review twice"]
- **Review reports:** [paths to review report files]

## Remaining Tickets

| Ticket | Title | Status | Blocked By |
|--------|-------|--------|------------|
| T-007  | Expense Endpoints | Todo | T-006 (in progress) |

## Next Steps

[What needs to happen to continue:
- "Fix the review findings on T-006 and re-run the loop"
- "All tickets complete — project is ready for integration testing"
- "Resolve the environment issue (pip install failing) before continuing"]
```

Save this report to the repo root as `IMPLEMENTATION_REPORT.md`.

---

## Configuration

### Ticket limit per run

By default, the loop processes all available tickets until none are ready. If you want to limit the batch size (e.g., only do 3 tickets per run), specify it when invoking:

"Start the implementation loop, limit to 3 tickets"

The loop will stop after processing the specified number of tickets, even if more are ready.

### Skipping tickets

If a specific ticket should be skipped (e.g., it requires manual work or an external dependency), say so:

"Start the implementation loop, skip T-015 and T-016"

Skipped tickets are treated as if they don't exist — the loop won't try to implement them, and downstream tickets that depend on them will be reported as blocked.

---

## Edge Cases

### When the implementer and reviewer disagree on scope

The implementer follows the ticket's acceptance criteria. The reviewer checks the same criteria. If the reviewer flags something as "not met" that the implementer believes it covered, the self-correction cycle should resolve it — the implementer gets the specific feedback and can address it. If it persists after one retry, it's a genuine ambiguity that needs human input.

### When a ticket's tests fail due to upstream code

If tests fail because of a bug in a previously-completed ticket's code, the reviewer should catch this and issue BLOCKED. The loop stops and reports the issue. Don't try to fix other tickets' code within the current ticket's scope.

### When the repo gets into a bad state

If git operations fail (merge conflicts, detached HEAD, etc.), stop the loop and report. Don't attempt complex git recovery — that needs human judgment.

### When dependencies form a parallel group

If T-006 and T-015 are both ready (different dependency chains), the loop picks one and does it first. It doesn't parallelize — sequential is safer and easier to debug. The one that appears first in TRACKER.md gets picked.

### When the loop is interrupted

If the loop is interrupted mid-ticket, the state is recoverable:
- If the implementer completed but reviewer hasn't run, the feature branch exists — run the reviewer manually
- If the implementer didn't complete, the feature branch may be partial — delete it and re-run
- TRACKER.md is the source of truth — if it still shows Todo, the ticket hasn't been delivered
