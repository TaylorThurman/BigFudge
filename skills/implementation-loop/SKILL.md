---
name: implementation-loop
description: "Autonomous implementation loop that picks up tickets from TRACKER.md and builds them in dependency order. For each ticket: runs the ticket-implementer, then the code-reviewer, and if approved, moves to the next ticket. Stops when no tickets are available, a review requests changes twice, or a blocker is hit. Triggers: 'start building', 'implement the tickets', 'build the backlog', 'start the implementation loop', 'pick up available tickets', 'build what's ready', 'run the implementation loop', 'start coding the tickets'. Use after the SDLC planning chain has produced tickets."
---

# Implementation Loop

This skill runs the implementation cycle autonomously. It reads TRACKER.md, finds all tickets that are ready to build (dependencies met, status Todo), and works them in parallel — up to 3 agents implementing tickets simultaneously. As each agent finishes, the orchestrator immediately updates the tracker and checks for newly unblocked work. It keeps going until there's nothing left to pick up.

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
┌──────────────────────────────────────────────┐
│            Read TRACKER.md                   │
│      Find ALL ready tickets                  │
│      (Todo + all dependencies Done)          │
└──────────────┬───────────────────────────────┘
               │
               ▼
        ┌──────────────┐
        │ Any tickets   │──── No ───► Done. Report summary.
        │ ready?        │
        └──────┬───────┘
               │ Yes
               ▼
┌──────────────────────────────────────────────┐
│  Spawn up to 3 agents from the ready pool    │
│  Each agent: implement → review → report     │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│  As each agent completes:                    │
│   1. Orchestrator updates TRACKER            │
│   2. Merge branch into main                  │
│   3. Check for newly unblocked tickets       │
│   4. If a slot opened + tickets ready,       │
│      spawn a new agent immediately           │
└──────────────┬───────────────────────────────┘
               │
               ▼
        ┌──────────────┐
        │ All agents    │──── No ───► Wait for running agents.
        │ finished?     │
        └──────┬───────┘
               │ Yes
               ▼
        Loop back to top
```

The key difference from a batch model: the orchestrator doesn't wait for all 3 agents to finish before doing anything. As soon as one agent completes, its ticket gets merged and the tracker gets updated. If that unblocks new tickets, a new agent spawns immediately to fill the open slot.

---

## The Orchestrator

The orchestrator is the single point of control. Agents do not update TRACKER.md or merge branches — they report back to the orchestrator, which handles all shared state.

**The orchestrator owns:**
- Reading and writing TRACKER.md (agents never touch it)
- Merging approved branches into main
- Rebasing other in-flight branches after a merge
- Spawning and tracking agents
- Deciding when to stop

**Each agent owns:**
- Its own feature branch
- The implement→review cycle for one ticket
- Reporting its result back to the orchestrator

This separation prevents race conditions. There's one writer for shared state (the orchestrator) and multiple readers/workers (the agents).

---

## Step 1: Find All Ready Tickets

Read TRACKER.md and scan the ticket summary table. A ticket is **ready** when:
- Its status is `Todo`
- ALL tickets listed in its "Depends On" column have status `Done`

Collect all ready tickets into a pool. This is the work available right now.

If no tickets are ready but some are still `Todo`, check why — their dependencies might be `In Progress` or blocked. Report this and stop.

If all tickets are `Done`, the loop is complete. Report the summary and stop.

---

## Step 2: Spawn Agents

From the ready pool, spawn up to 3 agents. Each agent works on one ticket independently.

There are no restrictions on which tickets can run in parallel. Two tickets in the same module can run simultaneously — merge conflicts are handled by the orchestrator's rebase strategy (see Merge and Rebase below).

If only 1 or 2 tickets are ready, run what's available. Don't wait for more.

### Each agent does:

**2a. Implement the ticket**

Invoke the ticket-implementer workflow. Pass it:
- The ticket file path
- The implementation spec path
- The repo root

The implementer will:
1. Load context (ticket, spec, existing code)
2. Verify prerequisites (dependency check — should pass since the orchestrator pre-checked)
3. Write the code and tests
4. Run tests
5. Create a feature branch and commit
6. Write a PR description

**Important:** The implementer does NOT update TRACKER.md or merge into main. It creates a feature branch and reports back.

**2b. Review the implementation**

After implementation completes, the same agent invokes the code-reviewer on the feature branch. The reviewer:
1. Loads context (ticket, PR description, spec)
2. Runs tests and linters
3. Deep reviews across all 6 dimensions
4. Produces a structured review report with a verdict

**2c. Handle the verdict (within the agent)**

- **APPROVE**: Agent reports success. Branch and PR description are ready.
- **REQUEST CHANGES**: Agent attempts one self-correction cycle — reads the must-fix findings, re-implements addressing each finding, re-reviews. If second review approves, reports success. If it fails again, agent reports failure with both review reports.
- **BLOCKED**: Agent reports blocker immediately.

Each agent returns a result: `{ticket_id, verdict, branch_name, review_report_path, notes}`.

---

## Step 3: Process Completions

As each agent completes (not waiting for all agents), the orchestrator immediately:

### On APPROVE:

1. **Merge the branch into main.** If the branch merges cleanly, great. If there's a conflict, rebase it (see Merge and Rebase below).
2. **Update TRACKER.md.** Mark the ticket as `Done`.
3. **Check for newly unblocked tickets.** Read TRACKER.md — did this completion unblock any `Todo` tickets whose dependencies are now all `Done`?
4. **Spawn a new agent** if there's an open slot (fewer than 3 running) and a newly ready ticket. This keeps the pipeline full.

### On REQUEST CHANGES (failed twice):

1. Mark the ticket as `Blocked` in TRACKER.md.
2. Log the failure with both review reports.
3. Do NOT merge. The branch stays as-is for human review.
4. Check for open slots and spawn if possible.

### On BLOCKED:

1. Mark the ticket as `Blocked` in TRACKER.md.
2. Log the blocker.
3. Check for open slots and spawn if possible.

**A single agent failing does not stop the loop.** Other agents keep running, and new agents can spawn for unrelated tickets.

---

## Merge and Rebase

This is how the orchestrator handles the fact that multiple agents branch from the same base and may modify overlapping files.

### The merge order

Agents complete at different times. The orchestrator merges branches in completion order — first agent done gets merged first.

### When a branch has conflicts

After Agent 1's branch merges into main, Agent 2's branch (which was based on the pre-merge main) may have conflicts. The orchestrator handles this:

1. **Attempt rebase.** Run `git rebase main` on Agent 2's branch. This replays Agent 2's changes on top of the updated main (which now includes Agent 1's work).

2. **If rebase succeeds cleanly:** Run the test suite on the rebased branch. If tests pass, merge it. If tests fail, the rebase introduced a logical conflict — mark the ticket for human review.

3. **If rebase has conflicts:** Attempt automatic resolution for simple conflicts (e.g., both agents added imports to the same file — combine them). Use `git rebase --continue` after resolving each conflict.

4. **If auto-resolution fails:** Mark the ticket as needing human resolution. The branch is left in its pre-rebase state so the human can resolve conflicts manually. The ticket goes to `Blocked` in TRACKER with a note: "Merge conflict with [other ticket] — needs manual resolution."

### Why rebase instead of merge commits

Rebase produces a clean linear history. Each ticket's changes appear as a coherent set of commits on main, not interleaved with merge commits. This makes it easier to understand what each ticket changed and to revert if needed.

### Worst case

If all 3 agents modify the same file and conflicts cascade, the first merge goes clean, the second might need a simple rebase, and the third might need human help. This is fine — the orchestrator got 2 out of 3 merged automatically, which is better than running everything sequentially.

---

## Step 4: Loop or Stop

The loop runs continuously as agents complete and new work becomes available. It stops when:

- **All tickets are `Done`.** Success — report and finish.
- **No agents are running and no tickets are ready.** Remaining tickets are all blocked by dependencies that aren't met. Report what's blocked and why.
- **The ticket limit has been reached.** Stop even if more tickets are available.

---

## Step 5: Report

After the loop ends, produce a summary:

```markdown
# Implementation Loop Report

**Date:** [date]
**Tickets processed:** [count]
**Tickets remaining:** [count]
**Concurrency:** up to 3 parallel agents

## Completed Tickets

| Ticket | Title | Branch | Review Verdict | Notes |
|--------|-------|--------|----------------|-------|
| T-004  | Auth Service | ticket/T-004-auth-service | APPROVE | Clean first pass |
| T-015  | CI/CD Pipeline | ticket/T-015-github-actions | APPROVE | Ran parallel with T-004 |
| T-005  | Auth Endpoints | ticket/T-005-auth-endpoints | APPROVE (2nd attempt) | Fixed missing input validation |

## Failed Tickets

| Ticket | Title | Reason | Review Reports |
|--------|-------|--------|----------------|
| T-006  | Expense Service | Failed review twice | reviews/T-006-review-1.md, reviews/T-006-review-2.md |

## Merge Conflicts

| Ticket | Conflicted With | Resolution |
|--------|----------------|------------|
| T-007  | T-006 | Auto-rebased, tests passed |
| T-008  | T-007 | Manual resolution needed |

## Remaining Tickets

| Ticket | Title | Status | Blocked By |
|--------|-------|--------|------------|
| T-009  | Dashboard | Todo | T-007 (blocked - merge conflict) |

## Next Steps

[What needs to happen to continue:
- "Resolve the merge conflict on T-008's branch and re-run the loop"
- "All tickets complete — project is ready for integration testing"
- "Fix the review findings on T-006 and re-run"]
```

Save this report to the repo root as `IMPLEMENTATION_REPORT.md`.

---

## Configuration

### Concurrency limit

The default concurrency is 3 parallel agents. To change it:

"Start the implementation loop with 2 agents"

Set to 1 for fully sequential execution (useful for debugging or tightly coupled codebases).

### Ticket limit per run

By default, the loop processes all available tickets until none are ready. To limit:

"Start the implementation loop, limit to 5 tickets"

The loop will stop after processing the specified number of tickets, even if more are ready.

### Skipping tickets

If a specific ticket should be skipped:

"Start the implementation loop, skip T-015 and T-016"

Skipped tickets are treated as if they don't exist — the loop won't try to implement them, and downstream tickets that depend on them will be reported as blocked.

---

## Edge Cases

### When the implementer and reviewer disagree on scope

The implementer follows the ticket's acceptance criteria. The reviewer checks the same criteria. If the reviewer flags something as "not met" that the implementer believes it covered, the self-correction cycle should resolve it. If it persists after one retry, it's a genuine ambiguity that needs human input.

### When a ticket's tests fail due to upstream code

If tests fail because of a bug in a previously-completed ticket's code, the reviewer should catch this and issue BLOCKED. The loop reports the issue. Don't try to fix other tickets' code within the current ticket's scope.

### When the repo gets into a bad state

If git operations fail (detached HEAD, corrupted index, etc.), stop the loop and report. Don't attempt complex git recovery — that needs human judgment.

### When the loop is interrupted

If the loop is interrupted mid-run, the state is recoverable:
- Agents that completed have their feature branches intact
- Agents that didn't complete may have partial branches — delete and re-run
- TRACKER.md is the source of truth — if it still shows `Todo`, the ticket hasn't been delivered
- Re-running the loop will pick up where it left off based on TRACKER state

### When a rebase changes behavior without conflict

A rebase can succeed (no git conflicts) but still break things if Agent 1's changes affect Agent 2's logic. This is why the orchestrator runs tests after every rebase. If tests fail after a clean rebase, the ticket is marked for human review — the code needs someone to understand the interaction between the two changes.
