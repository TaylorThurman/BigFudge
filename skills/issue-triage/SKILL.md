---
name: issue-triage
description: "Triages production issues and unexpected behavior by investigating code, specs, and architecture to determine what's wrong and where the fix belongs. Routes to the correct resolution path: direct bug fix via ticket-implementer, requirement change via sdlc-router, or config/environment fix. Triggers: 'something is broken', 'this isn't working right', 'I found a bug', 'the bot isn't doing X', 'unexpected behavior', 'it should be doing X but it's doing Y', 'production issue', 'why is it doing this', 'this doesn't look right', 'debug this', 'investigate this issue'. Use when the user reports a problem with live or running code and doesn't know which layer the problem is at."
---

# Issue Triage

This skill is the front door for "something's wrong." When you notice the live system isn't behaving the way you expect, this skill investigates, figures out which layer the problem is at, and routes it to the right fix.

You don't need to know whether it's a bug, a missing requirement, or an architecture gap. Just describe what you see vs what you expected. The triage process figures out the rest.

## Why This Skill Exists

A problem in production could originate at any layer: the code could have a bug, the spec could be incomplete, the architecture might not support what you need, or it could be a simple config issue. Without triage, you'd have to diagnose the layer yourself before knowing what to do. This skill does that diagnosis.

---

## The Triage Process

```
User reports: "It's doing X, but I expected Y"
        │
        ▼
  Phase 1: Understand the Problem
        │
        ▼
  Phase 2: Investigate
        │
        ▼
  Phase 3: Classify
        │
        ├── Code Bug ──────────► Create fix ticket → ticket-implementer
        ├── Spec Gap ──────────► sdlc-router (change request)
        ├── Architecture Gap ──► sdlc-router (fundamental change)
        ├── Config/Env Issue ──► Direct fix + ops documentation
        └── Can't Determine ───► Escalate to user with findings
```

---

## Phase 1: Understand the Problem

Get a clear picture of the issue before looking at any code.

### 1.1 Capture the Report

From the user's description, extract:
- **Observed behavior**: What is actually happening? Be specific — "it's broken" isn't enough. What output, error, or behavior did they see?
- **Expected behavior**: What should be happening instead? This is the gap we're investigating.
- **Context**: When does it happen? Always, or only under certain conditions? Did it used to work? Did anything change recently?
- **Severity**: Is this blocking all usage, degrading performance, or a cosmetic/edge case issue?

If the user's description is vague, ask for specifics. "The bot isn't working" could mean anything — "the bot isn't closing positions when RSI drops below 30" is actionable.

### 1.2 Locate Relevant Components

Based on the problem description, identify which parts of the system are likely involved:
- Which feature area? (authentication, expenses, trading, reporting, etc.)
- Which layer? (frontend UI, API endpoint, service logic, database, external integration)
- Which tickets originally built this? (check TRACKER.md for the feature area)

---

## Phase 2: Investigate

This is a systematic top-down investigation. Start with what the system is supposed to do, then compare against what it actually does.

### 2.1 Check the Spec

Read the product documentation and find the requirement(s) that cover the reported behavior. Start with FEATURES.md to identify which feature is involved, then read the relevant feature doc in features/ for its requirements and acceptance criteria. Also check PRODUCT.md for project-level constraints that might apply. Ask:
- **Is the expected behavior specified?** If the user expects something that no feature doc defines, this is a spec gap, not a bug.
- **Is the spec ambiguous?** If the requirement could be interpreted multiple ways, the code might be following a valid interpretation that differs from the user's expectation.
- **Is the spec contradictory?** Sometimes two requirements (possibly in different feature docs) conflict, and the code implements one at the expense of the other.

Record what the spec says about this behavior. If no feature doc addresses it at all, that's already a classification signal.

### 2.2 Check the Architecture

Read the architecture blueprint for the relevant component. Ask:
- **Does the architecture support the expected behavior?** If the user expects real-time updates but the architecture is polling-based, the architecture doesn't support the expectation.
- **Is there a data flow gap?** Does the data needed for the expected behavior actually flow through the right components?
- **Are there constraints that prevent the expected behavior?** (rate limits, batch processing, eventual consistency, etc.)

### 2.3 Check the Implementation Spec

Read the implementation spec for the relevant module interfaces. Ask:
- **Do the interfaces support the expected behavior?** If the service layer doesn't expose a method needed for the expected behavior, the implementation spec has a gap.
- **Are there conventions that might affect behavior?** (error handling patterns, validation rules, data types)

### 2.4 Check the Code

Now read the actual code. This is where you determine if the implementation matches the spec or deviates from it.

Find the relevant files by:
1. Checking the ticket that built this feature (ticket file has file paths and interfaces)
2. Searching the codebase for the relevant function/class/endpoint
3. Tracing the execution path from the entry point (API route, UI action, scheduled task) through the service layer to the data layer

Compare the code against the spec:
- **Does the code implement what the spec says?** If the spec says "close position when RSI < 30" and the code checks `RSI < 20`, that's a code bug.
- **Does the code handle the edge case the user hit?** If the spec says "validate input" and the code validates most inputs but misses the user's case, that's a code bug.
- **Is the code correct but producing unexpected results for a valid reason?** (data type precision, timing issues, race conditions, etc.)

### 2.5 Check Tests

Read the tests for the relevant feature. Ask:
- **Do tests cover the reported scenario?** If yes and they pass, the code might be correct and the issue is elsewhere.
- **Are tests missing for this scenario?** Missing test coverage is a signal that the edge case wasn't considered.
- **Do existing tests assert the wrong thing?** A test might pass while testing the wrong behavior.

### 2.6 Check Configuration and Environment (if applicable)

If the code and spec look correct, check:
- Environment variables and config files
- Database state (missing data, corrupted records, schema drift)
- External service connectivity (API keys, endpoints, rate limits)
- Deployment state (wrong version deployed, missing migrations)

---

## Phase 3: Classify and Route

Based on the investigation, classify the issue into one of four categories.

### Category 1: Code Bug

**The spec is correct, but the code doesn't match it.**

Signals:
- The spec clearly defines the expected behavior
- The code deviates from the spec (wrong logic, missing edge case, incorrect calculation)
- Tests are either missing or assert the wrong thing

Action:
1. Write a **bug ticket** in the project's ticket format:
   - Description: what's wrong, where in the code, what the spec says
   - Acceptance criteria: the fix criteria (specific behavior that must be corrected)
   - Test requirements: what tests need to be added or fixed
   - Dependencies: usually none (bug fixes are standalone)
2. Add the ticket to TRACKER.md with status `Todo`
3. Save the ticket file in the appropriate slice directory
4. Report to the user: "This is a code bug. [Explain the root cause.] I've created ticket [ID] for the fix. The ticket-implementer can pick it up, or you can start the implementation-loop."

The fix goes directly to the ticket-implementer — no planning chain needed.

### Category 2: Spec Gap

**The code matches the spec, but the spec doesn't cover what the user expects.**

Signals:
- The expected behavior is not specified in the product spec
- The code is correct according to what the spec says
- The user's expectation is reasonable but was never captured as a requirement

Action:
1. Write a **change request** describing:
   - The missing requirement (what the user expects)
   - Why it's reasonable (user workflow, edge case, business logic)
   - The current spec gap (which section should cover this but doesn't)
2. Report to the user: "This isn't a bug — the code is doing what the spec says. But the spec doesn't cover [the expected behavior]. This needs a requirement update. Feed this change request into the sdlc-router to update the product spec and generate tickets."
3. Provide the change request in a format the sdlc-router can consume (structured markdown with the new requirement, constraints, and acceptance criteria).

This goes to the sdlc-router as an updated requirement.

### Category 3: Architecture Gap

**The architecture fundamentally can't support the expected behavior.**

Signals:
- The expected behavior requires a capability the system isn't designed for
- Adding it would need new components, different data flow, or infrastructure changes
- It's not a matter of adding code — the design doesn't accommodate it

Action:
1. Write a **change request** describing:
   - The capability gap
   - What architectural changes would be needed
   - Impact on existing functionality
2. Report to the user: "The current architecture doesn't support [the expected behavior] because [explanation]. This needs a design change. The sdlc-router should evaluate whether the architecture blueprint, implementation spec, and tickets need updating."
3. Flag this as a fundamental change — the sdlc-router's escalation rules may apply.

This goes to the sdlc-router as a fundamental change request.

### Category 4: Config / Environment Issue

**The code and spec are correct, but the environment is wrong.**

Signals:
- The code logic is correct
- The spec is adequate
- The issue is in configuration, data, or external dependencies

Action:
1. Identify the specific config/environment problem
2. Fix it directly if possible (update config, fix data, restore connectivity)
3. Report to the user: "The code is correct. The issue was [config problem]. I've [fixed it / here's what needs to change]."
4. If this type of issue could recur, suggest adding a health check or validation (this could become a ticket).

This is a direct fix — no planning chain or implementer needed.

### Can't Determine

If the investigation is inconclusive:
1. Present what you found at each layer
2. Explain which layers check out and which are ambiguous
3. Ask the user for more context: specific inputs that trigger the issue, logs, timestamps, or reproduction steps
4. Don't guess — an incorrect classification sends the fix down the wrong path

---

## Phase 4: Produce the Triage Report

Always produce a written report, regardless of classification.

```markdown
# Issue Triage Report

**Date:** [date]
**Reported by:** [user]
**Severity:** [blocking / degraded / cosmetic]

## Problem Statement

**Observed:** [what's happening]
**Expected:** [what should happen]
**Context:** [when, how often, any recent changes]

## Investigation

### Spec Check
[What the spec says about this behavior. Quote the relevant requirement if it exists.]

### Architecture Check
[Whether the architecture supports the expected behavior.]

### Code Check
[What the code does, and whether it matches the spec. Include file paths and line references.]

### Test Check
[Whether tests cover this scenario, and what they assert.]

### Config/Environment Check
[If applicable, what was found.]

## Classification

**Category:** [Code Bug / Spec Gap / Architecture Gap / Config Issue / Inconclusive]
**Root Cause:** [One-sentence explanation]

## Resolution Path

[What happens next:
- For bugs: ticket ID and path to ticket file
- For spec gaps: change request for the sdlc-router
- For architecture gaps: change request with impact analysis
- For config: what was fixed or needs fixing
- For inconclusive: what additional information is needed]
```

Save the report to the project directory as `triage/[date]-[brief-description].md`. Create the `triage/` directory if it doesn't exist.

---

## Edge Cases

### When the user is wrong about expected behavior

Sometimes the system is working correctly and the user's expectation is based on a misunderstanding. If the spec clearly defines the behavior and the code matches, explain what the spec says and why the system behaves this way. Don't create a ticket for "working as designed." If the user still disagrees, that's a spec gap — their expectation is reasonable enough to warrant a requirement discussion.

### When multiple layers are broken

If you find both a code bug AND a spec gap (e.g., the spec is ambiguous AND the code picked the wrong interpretation), address both. Create a change request for the spec clarity AND a bug ticket for the code fix. The spec update should go through the router first, then the bug fix.

### When the issue is intermittent

Intermittent issues are often config/environment, timing, or concurrency problems. Focus the investigation on what varies between success and failure: input data, timing, system load, external service availability. If you can't reproduce it from the code alone, say so and ask for logs or reproduction steps.

### When there's no spec to check against

If the project doesn't have SDLC documents (no product spec, no architecture blueprint), you can still investigate the code and tests. Classify as best you can, but note that without specs, the classification is based on code intent rather than documented requirements. Suggest creating specs to prevent future ambiguity.

### When the issue is in a dependency

If the root cause is in a third-party library or external service, that's not fixable through the SDLC chain. Report the dependency issue, suggest a workaround if one exists, and if a code change is needed to handle the dependency's behavior (retry logic, fallback, version pin), create a ticket for that.
