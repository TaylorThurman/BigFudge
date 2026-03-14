---
name: implementation-loop
description: "Autonomous implementation loop that picks up tickets from TRACKER.md and builds them in dependency order. For each ticket: runs the ticket-implementer, then the code-reviewer, and if approved, moves to the next ticket. Stops when no tickets are available, a review requests changes twice, or a blocker is hit. Triggers: 'start building', 'implement the tickets', 'build the backlog', 'start the implementation loop', 'pick up available tickets', 'build what's ready', 'run the implementation loop', 'start coding the tickets'. Use after the SDLC planning chain has produced tickets."
---

# Implementation Loop

This skill runs the implementation cycle autonomously. It reads TRACKER.md, finds all tickets that are ready to build (dependencies met, status Todo), and works them in parallel — up to 3 agents implementing tickets simultaneously. For each ticket: implement it, review it, and move on. It keeps going until there's nothing left to pick up.

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
┌─────────────────────────────────────────┐
│           Read TRACKER.md               │
│     Find ALL ready tickets              │
└──────────────┬──────────────────────────┘
               │
               ▼
        ┌──────────────┐
        │ Any tickets   │──── No ───► Done. Report summary.
        │ ready?        │
        └──────┬───────┘
               │ Yes
               ▼
┌─────────────────────────────────────────┐
│   Select batch (up to 3 tickets)        │
│   from the ready pool                   │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│   Spawn parallel agents:                │
│   Agent 1 → ticket A (implement+review) │
│   Agent 2 → ticket B (implement+review) │
│   Agent 3 → ticket C (implement+review) │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│   Wait for all agents to complete       │
│   Collect results from each             │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│   Update TRACKER.md centrally           │
│   (mark completed tickets as Done)      │
└──────────────┬──────────────────────────┘
               │
               ▼
        Loop back to top
        (newly unblocked tickets
         may now be ready)
```

---

## Step 1: Find All Ready Tickets

Read TRACKER.md and scan the ticket summary table. A ticket is **ready** when:
- Its status is `Todo`
- ALL tickets listed in its "Depends On" column have status `Done`

Collect all ready tickets into a pool. This is the work available for this cycle.

If no tickets are ready but some are still `Todo`, check why — their dependencies might be `In Progress` or blocked. Report this and stop.

If all tickets are `Done`, the loop is complete. Report the summary and stop.

---

## Step 2: Select the Batch

From the ready pool, select up to 3 tickets for parallel execution. This is the **concurrency limit** — it balances throughput against merge conflict risk.

**Selection rules:**
- Pick tickets that touch different parts of the codebase when possible. If T-004 (Auth Service) and T-015 (GitHub Actions) are both ready, they're ideal for parallel work since they modify completely different files.
- Avoid pairing tickets that modify the same module. If T-006 (Expense Service) and T-007 (Expense Endpoints) are both ready and both touch `backend/app/expenses/`, run them sequentially to avoid conflicts.
- When in doubt, prefer tickets from different slices — vertical slices are designed to be independent.

If only 1 or 2 tickets are ready, run what's available. Don't wait for more.

---

## Step 3: Spawn Parallel Agents

For each ticket in the batch, spawn an agent that runs the full implement→review cycle independently:

### Each agent does:

**3a. Implement the ticket**

Invoke the ticket-implementer workflow. Pass it:
- The ticket file path
- The implementation spec path
- The TRACKER.md path (read-only — agents don't update it directly)
- The repo root

The implementer will:
1. Load context (ticket, spec, existing code)
2. Verify prerequisites (dependency check — should pass since we pre-checked)
3. Write the code and tests
4. Run tests
5. Create a feature branch and commit
6. Write a PR description

**Important:** The implementer does NOT update TRACKER.md. The orchestrator handles TRACKER updates centrally after all agents complete. This prevents concurrent write conflicts.

**3b. Review the implementation**

After implementation completes, the same agent invokes the code-reviewer on the feature branch. The reviewer:
1. Loads context (ticket, PR description, spec)
2. Runs tests and linters
3. Deep reviews across all 6 dimensions
4. Produces a structured review report with a verdict

**3c. Handle the verdict (within the agent)**

- **APPROVE**: Agent reports success. Branch and PR description are ready.
- **REQUEST CHANGES**: Agent attempts one self-correction cycle — reads the must-fix findings, re-implements addressing each finding, re-reviews. If second review approves, reports success. If it fails again, agent reports failure with both review reports.
- **BLOCKED**: Agent reports blocker immediately.

Each agent returns a result: `{ticket_id, verdict, branch_name, review_report_path, notes}`.

---

## Step 4: Collect Results and Update TRACKER

After all agents in the batch complete, the orchestrator processes results:

1. **For each APPROVE result**: Mark the ticket as `Done` in TRACKER.md.
2. **For each REQUEST CHANGES (failed twice) result**: Mark the ticket as `Blocked` in TRACKER.md. Log the failure for the report.
3. **For each BLOCKED result**: Mark the ticket as `Blocked`. Log the blocker.

Update TRACKER.md once, in a single operation, after all agents finish. This avoids concurrent write conflicts.

If git remote is available, push completed branches and create PRs for approved tickets.

---

## Step 5: Loop or Stop

After updating TRACKER, decide what to do next:

- **If there are newly unblocked tickets** (tickets whose dependencies just became `Done` in this cycle): loop back to Step 1. The next batch may include these newly ready tickets.
- **If no new tickets are unblocked and the ready pool is empty**: stop. Either everything is `Done` or remaining tickets are blocked.
- **If the ticket limit has been reached**: stop, even if more tickets are available.

---

## Step 6: Report

After the loop ends, produce a summary:

```markdown
# Implementation Loop Report

**Date:** [date]
**Tickets processed:** [count]
**Tickets remaining:** [count]
**Concurrency:** up to 3 parallel agents

## Completed Tickets

| Ticket | Title | Branch | Review Verdict | Batch | Notes |
|--------|-------|--------|----------------|-------|-------|
| T-004  | Auth Service | ticket/T-004-auth-service | APPROVE | 1 | Clean first pass |
| T-015  | CI/CD Pipeline | ticket/T-015-github-actions | APPROVE | 1 | Ran parallel with T-004 |
| T-005  | Auth Endpoints | ticket/T-005-auth-endpoints | APPROVE (2nd attempt) | 2 | Fixed missing input validation |

## Failed Tickets

| Ticket | Title | Reason | Review Reports |
|--------|-------|--------|----------------|
| T-006  | Expense Service | Failed review twice | reviews/T-006-review-1.md, reviews/T-006-review-2.md |

## Remaining Tickets

| Ticket | Title | Status | Blocked By |
|--------|-------|--------|------------|
| T-007  | Expense Endpoints | Todo | T-006 (blocked) |

## Next Steps

[What needs to happen to continue:
- "Fix the review findings on T-006 and re-run the loop"
- "All tickets complete — project is ready for integration testing"
- "Resolve the environment issue before continuing"]
```

Save this report to the repo root as `IMPLEMENTATION_REPORT.md`.

---

## Configuration

### Concurrency limit

The default concurrency is 3 parallel agents. To change it:

"Start the implementation loop with 2 agents"

Set to 1 for fully sequential execution (safer for tightly coupled codebases).

### Ticket limit per run

By default, the loop processes all available tickets until none are ready. To limit the batch size:

"Start the implementation loop, limit to 5 tickets"

The loop will stop after processing the specified number of tickets, even if more are ready.

### Skipping tickets

If a specific ticket should be skipped:

"Start the implementation loop, skip T-015 and T-016"

Skipped tickets are treated as if they don't exist — the loop won't try to implement them, and downstream tickets that depend on them will be reported as blocked.

---

## Merge Conflict Handling

When multiple agents work in parallel, merge conflicts can occur if two tickets modify the same file.

**Prevention:**
- The batch selection in Step 2 avoids pairing tickets that touch the same module
- Vertical slices are designed to be independent, so well-structured tickets rarely conflict

**Detection:**
- After all agents complete, attempt to merge each approved branch into the base branch
- If a merge conflict is detected, mark the conflicting ticket as needing manual resolution
- The first-completed branch merges cleanly; the second one gets flagged

**Resolution:**
- Don't auto-resolve merge conflicts. Report them and let a human handle it.
- The conflicting branch still has valid code — it just needs manual conflict resolution before merging.

---

## Edge Cases

### When the implementer and reviewer disagree on scope

The implementer follows the ticket's acceptance criteria. The reviewer checks the same criteria. If the reviewer flags something as "not met" that the implementer believes it covered, the self-correction cycle should resolve it — the implementer gets the specific feedback and can address it. If it persists after one retry, it's a genuine ambiguity that needs human input.

### When a ticket's tests fail due to upstream code

If tests fail because of a bug in a previously-completed ticket's code, the reviewer should catch this and issue BLOCKED. The loop reports the issue. Don't try to fix other tickets' code within the current ticket's scope.

### When the repo gets into a bad state

If git operations fail (merge conflicts, detached HEAD, etc.), stop the loop and report. Don't attempt complex git recovery — that needs human judgment.

### When the loop is interrupted

If the loop is interrupted mid-batch, the state is recoverable:
- Agents that completed have their feature branches intact
- Agents that didn't complete may have partial branches — delete and re-run
- TRACKER.md is the source of truth — if it still shows `Todo`, the ticket hasn't been delivered
- Re-running the loop will pick up where it left off based on TRACKER state

### When all ready tickets are in the same module

If the only ready tickets all touch the same code (e.g., T-006, T-007, T-008 all in `backend/app/expenses/`), run them one at a time to avoid conflicts. The batch selection should only pick one, then loop back after it completes to pick the next.
