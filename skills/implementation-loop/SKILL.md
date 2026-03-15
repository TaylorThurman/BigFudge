---
name: implementation-loop
description: "Autonomous implementation loop that picks up tickets from TRACKER.md and builds them in dependency order. For each ticket: runs the ticket-implementer, then the code-reviewer, and if approved, moves to the next ticket. Stops when no tickets are available, a review requests changes twice, or a blocker is hit. Triggers: 'start the implementation loop', 'run the implementation loop', 'implement the tickets', 'build the backlog', 'pick up available tickets', 'start the build loop'. This skill turns existing tickets into code — it does NOT create tickets or run planning. Use after the SDLC planning chain has already produced tickets in TRACKER.md. If the user has requirements but no tickets yet, use the sdlc-router skill instead."
---

# Implementation Loop

This skill runs the implementation cycle autonomously. It reads TRACKER.md, finds all tickets that are ready to build (dependencies met, status Todo), and works them in parallel — up to 3 agents implementing tickets simultaneously. As each agent finishes, the orchestrator immediately updates the tracker and checks for newly unblocked work. It keeps going until there's nothing left to pick up.

The loop is the bridge between "planning is done" and "code is ready for review." After the SDLC router produces tickets, this skill turns them into PRs ready for human approval. **No code is merged into main without the project owner approving the PR.** The loop creates PRs — the human decides when they merge.

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
│   1. Merge main into feature branch          │
│   2. Run tests                               │
│   3. Push branch and create PR               │
│   4. Update TRACKER (In Review)              │
│   5. Check for newly unblocked tickets       │
│   6. If a slot opened + tickets ready,       │
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

The key difference from a batch model: the orchestrator doesn't wait for all 3 agents to finish before doing anything. As soon as one agent completes, it merges main into the feature branch, creates a PR, and updates the tracker. If that unblocks new tickets, a new agent spawns immediately to fill the open slot.

**PRs are never auto-merged.** The project owner reviews and approves every PR before it enters main. The ticket moves from `In Review` to `Done` only after the PR is merged by the human.

---

## The Orchestrator

The orchestrator is the single point of control. Agents do not update TRACKER.md or merge branches — they report back to the orchestrator, which handles all shared state.

**The orchestrator owns:**
- Reading and writing TRACKER.md (agents never touch it)
- Merging main into feature branches to incorporate changes from other agents
- Pushing branches and creating PRs (PRs are never auto-merged — the human approves)
- Spawning and tracking agents
- Deciding when to stop

**Each agent owns:**
- Its own feature branch
- The implement→review cycle for one ticket
- Reporting its result back to the orchestrator

This separation prevents race conditions. There's one writer for shared state (the orchestrator) and multiple readers/workers (the agents).

---

## TRACKER Status Lifecycle

```
Todo → In Progress → In Review → Done
                  ↘ Blocked
```

- **Todo**: Not started, waiting for dependencies.
- **In Progress**: An agent is actively implementing and reviewing the ticket.
- **In Review**: Code review passed, PR created, awaiting human approval. The code is complete — it just needs the project owner to review and merge the PR.
- **Done**: Human approved and merged the PR. Code is on main.
- **Blocked**: Implementation failed twice, conflict resolution failed, or a blocker was hit.

A ticket's dependencies are considered **met** when they are `In Review` or `Done`. An `In Review` ticket has passed automated code review and all tests — the work is finished, it's just waiting in the approval queue.

---

## Step 1: Find All Ready Tickets

Read TRACKER.md and scan the ticket summary table. A ticket is **ready** when:
- Its status is `Todo`
- ALL tickets listed in its "Depends On" column have status `In Review` or `Done`

Collect all ready tickets into a pool. This is the work available right now.

If no tickets are ready but some are still `Todo`, check why — their dependencies might be `In Progress` or blocked. Report this and stop.

If all tickets are `In Review` or `Done`, the loop is complete. Report the summary and stop.

---

## Step 2: Spawn Agents

From the ready pool, spawn up to 3 agents. Each agent works on one ticket independently.

There are no restrictions on which tickets can run in parallel. Two tickets in the same module can run simultaneously — merge conflicts are handled by the orchestrator's merge strategy (see Merge and Conflict Resolution below).

If only 1 or 2 tickets are ready, run what's available. Don't wait for more.

### Each agent does:

**2a. Start from a clean main**

Before invoking the implementer, the orchestrator ensures the agent starts from the latest main:

```bash
git checkout main
git pull origin main
```

This is critical. Without this step, an agent that just finished one ticket will still be on that ticket's feature branch — and the next ticket's work will be committed to the wrong branch.

**2b. Implement the ticket**

Invoke the ticket-implementer workflow. Pass it:
- The ticket file path
- The implementation spec path
- The repo root

The implementer will:
1. Load context (ticket, spec, existing code)
2. Verify prerequisites (dependency check — should pass since the orchestrator pre-checked)
3. Write the code and tests
4. Run tests
5. Checkout main, then create a new feature branch (`feature/T-XXX-description`) from main and commit
6. Write a PR description

**Important:** The implementer does NOT update TRACKER.md or merge into main. It creates a feature branch and reports back.

**2c. Review the implementation**

After implementation completes, the same agent invokes the code-reviewer on the feature branch. The reviewer:
1. Loads context (ticket, PR description, spec)
2. Runs tests and linters
3. Deep reviews across all 6 dimensions
4. Produces a structured review report with a verdict

**2d. Handle the verdict (within the agent)**

- **APPROVE**: Agent reports success. Branch and PR description are ready.
- **REQUEST CHANGES**: Agent attempts one self-correction cycle — reads the must-fix findings, re-implements addressing each finding, re-reviews. If second review approves, reports success. If it fails again, agent reports failure with both review reports.
- **BLOCKED**: Agent reports blocker immediately.

Each agent returns a result: `{ticket_id, verdict, branch_name, review_report_path, notes}`.

---

## Step 3: Process Completions

As each agent completes (not waiting for all agents), the orchestrator immediately:

### On APPROVE:

1. **Merge main into the feature branch.** Pull updated main into the agent's feature branch:

```bash
git checkout feature/T-XXX-description
git merge main
```

If it merges cleanly, run the test suite. If tests pass, proceed to step 2. If there's a conflict or tests fail after merge, go to Merge and Conflict Resolution below.

2. **Push the branch and create a PR.** This is mandatory — every ticket gets its own PR:

```bash
git push -u origin feature/T-XXX-description
gh pr create --title "T-XXX: Description" --body "## Ticket\n[ticket content]\n\n## Changes\n[summary]"
```

If `gh` is not available, push the branch and note the PR needs to be created manually. The branch MUST be pushed regardless.

3. **Update TRACKER.md.** Mark the ticket as `In Review` and record the PR number/link. Example:

```
| T-007 | Bot status component | Slice 2 | T-006 | Small | In Review (#12) |
```

The ticket moves to `Done` only after the human merges the PR. **This TRACKER update is not optional** — it's what unblocks dependent tickets.

4. **Check for newly unblocked tickets.** Tickets whose dependencies are all `In Review` or `Done` are now ready. An `In Review` ticket has passed code review and tests — the code is complete, it's just awaiting human sign-off.
5. **Spawn a new agent** if there's an open slot (fewer than 3 running) and a newly ready ticket. This keeps the pipeline full.

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

## Merge and Conflict Resolution

This is how the orchestrator handles the fact that multiple agents branch from the same base and may modify overlapping files.

### The merge workflow

When an agent completes and its code review passes, the orchestrator prepares the branch for a PR:

1. **Pull updated main into the feature branch.** Run `git merge main` on the feature branch. This brings in any work that was merged into main since the agent branched (e.g., PRs the human approved while this agent was working). Both sets of changes are preserved.

2. **If merge succeeds cleanly:** Run the test suite on the merged branch. If tests pass, push the branch and create a PR to merge into main. If tests fail, a logical conflict exists (the two tickets interact in a way that breaks behavior) — proceed to the conflict resolution agent.

3. **If merge has line-level conflicts:** Don't attempt auto-resolution. Proceed to the conflict resolution agent.

Note: since PRs require human approval, main only moves forward when the human merges a PR. Multiple agents may create PRs around the same time, but the human controls the merge order. When the human merges one PR, the other open PRs may need their branches updated — but that happens on the next loop iteration or when the human requests it.

### Conflict Resolution Agent

When a merge has line conflicts or merges cleanly but tests break (logical conflict), the orchestrator spawns a dedicated **conflict resolution agent**. This agent has full context to make an informed decision about how to combine the two tickets' work.

The conflict resolution agent receives:
- Both ticket files (the two tickets whose changes conflict)
- The implementation spec (coding conventions and module interfaces)
- The review reports from both tickets (what was implemented and why)
- The conflict diff (which files and lines are in conflict)
- The test failure output (if it's a logical conflict with a clean merge)

The agent's job:

1. **Understand both tickets' intent.** Read both tickets and their review reports to understand what each change was trying to accomplish.

2. **Resolve the conflict.** For line-level conflicts: examine the conflicting sections, understand what each agent wrote and why, and produce a merged version that satisfies both tickets' acceptance criteria. For logical conflicts: identify the interaction between the two changes and modify the code so both tickets' behavior works correctly together.

3. **Run tests.** After resolving, run the full test suite. Both tickets' tests must pass.

4. **If resolution succeeds:** Commit the resolved merge on the feature branch. The orchestrator pushes and creates the PR to merge into main. The human reviews and approves as usual.

5. **If resolution fails:** The agent reports that it couldn't reconcile the two changes. The ticket is marked `Blocked` in TRACKER with a note: "Conflict with [other ticket] — needs human resolution." The branch is left in its pre-merge state so the human can resolve it manually.

One attempt only. If the conflict resolution agent can't fix it, a human needs to look. Don't loop on conflict resolution — it risks making things worse.

---

## Step 4: Loop or Stop

The loop runs continuously as agents complete and new work becomes available. It stops when:

- **All tickets are `In Review` or `Done`.** All work has been implemented and PRs are ready for human approval. Report and finish.
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

## PRs Ready for Review

| Ticket | Title | Branch | PR | Review Verdict | Notes |
|--------|-------|--------|----|----------------|-------|
| T-004  | Auth Service | feature/T-004-auth-service | #12 | APPROVE | Clean first pass |
| T-015  | CI/CD Pipeline | feature/T-015-github-actions | #13 | APPROVE | Ran parallel with T-004 |
| T-005  | Auth Endpoints | feature/T-005-auth-endpoints | #14 | APPROVE (2nd attempt) | Fixed missing input validation |

## Failed Tickets

| Ticket | Title | Reason | Review Reports |
|--------|-------|--------|----------------|
| T-006  | Expense Service | Failed review twice | reviews/T-006-review-1.md, reviews/T-006-review-2.md |

## Merge Conflicts

| Ticket | Conflicted With | Resolution |
|--------|----------------|------------|
| T-007  | T-006 | Auto-merged main, tests passed |
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

### When a merge succeeds but changes behavior

A merge can succeed (no git conflicts) but still break things if Agent 1's changes affect Agent 2's logic. This is why the orchestrator runs tests after every merge. If tests fail after a clean merge of main into the feature branch, the conflict resolution agent is spawned — it has both tickets' context and can understand the interaction between the two changes to fix the logical conflict.
