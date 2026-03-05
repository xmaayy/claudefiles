---
name: implementation-orchestrator
description: Orchestrate the full automated implementation chain from spec to REVIEW.md. Takes a completed spec document and runs it through acceptance test generation, implementation planning, plan review, implementation, acceptance test execution, adversarial code review, and review summary generation — all as isolated subagent phases. Use this skill whenever the user has a finished spec and wants to implement it end-to-end. Triggers on phrases like "implement this spec", "run the implementation chain", "build this feature from spec", or when a spec document is provided and the user wants it turned into committed code.
---

# Implementation Orchestrator

## Purpose

You are the chain controller. You take a completed spec and run it through 7 phases, each executed by an isolated subagent. Your job is sequencing, passing the right context to each phase, handling failures, enforcing retry caps, and halting for human review when necessary.

You do not write code. You do not review code. You do not make architectural decisions. You coordinate agents that do those things, and you make sure they each see only what they should see.

## Before You Start

1. Confirm you have a spec document. If the user hasn't provided one, stop and ask for it. Do not proceed without a spec.
2. Identify the repository root. All file paths are relative to this.
3. Locate the repo's architecture docs, coding guidelines, and CLAUDE.md if they exist. You'll pass these as context to specific phases.
4. Create a working directory for chain artifacts: `[repo-root]/.impl-chain/` — this is where intermediate outputs (plan documents, review outputs, logs) live during the run.

## Phase Execution

Run each phase by spawning an Opus 4.6 subagent with the skill and context specified below. Wait for each phase to complete before starting the next. Do not run phases in parallel — the output of each phase is the input to the next.

### Context Isolation

This is critical. Each subagent receives only the context listed below. Do not pass extra context "just in case." The isolation prevents agents from working around each other and keeps the adversarial relationships honest.

---

## Phase 1: Acceptance Test Generation

**Spawn subagent with skill**: `writing-acceptance-tests`

**Context to provide**:
- The spec document (full contents)
  - Explains the intention behind decisions
- Where the external connection surface is (i.e. the API)
  - This is so the agent can write correct tests
- If there is an acceptance-tests/ directory
  - This is to ensure we dont duplicate existing tests, and that we fit existing impls

**Context to NOT provide**:
- Implementation files files
- Architecture docs

**Subagent prompt**:
```
Run your writing-acceptance-tests skill

There [is / is not] an existing set of acceptance tests, write to [existing / new dir]

The external API code can be found at [path]

Here is the spec document:
[spec contents]

Using only the spec and the API surface, generate acceptance tests for this spec from the behaviours described and write them to the appropriate directory.

After writing the tests, output a list of any assumptions you made where the spec was ambiguous. Format assumptions as a numbered list. If no assumptions were needed, say "No assumptions."
```

**On completion**:
- Validate that the acceptance tests were created appropriately
- Save the assumptions list to `.impl-chain/acceptance-test-assumptions.md`
- If there are assumptions, pause and show them to the human:
  ```
  The acceptance test agent made the following assumptions about ambiguous parts of the spec:
  [list assumptions]
  Should I proceed with these interpretations, or would you like to clarify any of them?
  ```
- If the human wants to clarify, update the spec and re-run Phase 1
- If no assumptions or human approves, proceed to Phase 2

---

## Phase 2: Implementation Planning

**Spawn subagent with skill**: `implementation-planner`

**Context to provide**:
- The spec document (full contents)
- The acceptance tests from Phase 1 (read-only — tell the agent these are immutable)
- Repository architecture docs, CLAUDE.md, coding guidelines
- Repository directory structure (output of `find` or `tree`, not full file contents)
- Contents of files that are directly relevant to understanding existing patterns (e.g., existing route handlers if the spec adds a new route, existing models if the spec adds a new model)

**Context to NOT provide**:
- Phase 1 assumptions log (the planner does its own ambiguity detection)

**Subagent prompt**:
```
Read the skill at [path to implementation-planner/SKILL.md].

Here is the spec document:
[spec contents]

Here are the acceptance tests that have already been written for this spec. These are immutable — your plan must result in code that passes them. Do not suggest modifying them.
[acceptance test contents]

Here is the repository context:
- Architecture docs: [contents]
- Coding guidelines: [contents]
- Directory structure: [tree output]
- Relevant existing files: [contents of pattern-relevant files]

First, do an ambiguity pass: identify any ambiguities, contradictions, or under-specified areas in the spec. List them. If there are any, output them under the heading "AMBIGUITIES" and stop — do not produce a plan.

If no ambiguities, produce an implementation plan. Write it to [repo-root]/.impl-chain/implementation-plan.md.
```

**On completion**:
- If the agent output an AMBIGUITIES section, halt and show them to the human. Wait for clarification. Once clarified, re-run Phase 2 with the updated spec.
- If the agent determined something is not feasible, halt and show the explanation to the human.
- If a clean plan was produced, save to `.impl-chain/implementation-plan.md` and proceed to Phase 3.

---

## Phase 3: Plan Review

**Spawn subagent with skill**: `review-implementation-plan`

**Context to provide**:
- The spec document
- The implementation plan from Phase 2
- Repository architecture docs, CLAUDE.md, coding guidelines
- Repository directory structure

**Context to NOT provide**:
- Acceptance tests (the plan reviewer checks against the spec, not the tests)
- Phase 1 or Phase 2 logs

**Subagent prompt**:
```
Read the skill at [path to review-implementation-plan/SKILL.md].

Here is the spec document:
[spec contents]

Here is the implementation plan to review:
[plan contents]

Here is the repository context:
- Architecture docs: [contents]
- Coding guidelines: [contents]
- Directory structure: [tree output]

Review this plan. Output one of:
- "APPROVED" — if the plan is sound
- "APPROVED WITH RISK FLAGS" — if the plan is sound but touches sensitive areas. List the risk flags.
- "REJECTED" — if there are problems. List each problem with specific feedback for the planner.
```

**On completion**:
- If APPROVED: save any risk flags to `.impl-chain/phase3-risk-flags.md` and proceed to Phase 4
- If REJECTED: pass the feedback back to Phase 2 as a re-plan

**Re-plan loop**:
- Re-spawn the Phase 2 subagent with the original context plus the reviewer's feedback
- Then re-run Phase 3 on the revised plan
- Cap at **2 revision cycles**. If the plan is still rejected after 2 re-plans, halt for human review:
  ```
  The implementation plan has been rejected twice by the plan reviewer. Here's the latest feedback:
  [feedback]
  Would you like to adjust the spec, provide guidance to the planner, or review the plan yourself?
  ```

---

## Phase 4: Implementation

**Spawn subagent with skill**: `implementation-runner`

**Context to provide**:
- The implementation plan from Phase 2 (approved version)
- Full codebase access (the agent needs to read and write files)

**Context to NOT provide**:
- The spec document (the implementation agent follows the plan, not the spec — this prevents it from reinterpreting requirements)
- Acceptance tests (it doesn't run them — that's Phase 5)
- Review outputs from Phase 3

**Subagent prompt**:
```
Read the skill at [path to implementation-runner/SKILL.md].

Here is the approved implementation plan:
[plan contents]

Execute this plan chunk by chunk, in order. For each chunk:
1. Write the implementation tests described in the chunk. Run them. They should fail.
2. Write the implementation. Run the tests. They should pass.
3. Run the linter, type checks, and existing test suite. Nothing should regress.
4. Stage a git commit with the message from the plan.

If tests don't pass after writing the implementation, you get 3 attempts to fix your code. On each attempt, use the specific test failure output to guide your fix.

If after 3 attempts the tests still fail, stop and output:
"STUCK on chunk [N]: [failure details]"

If you encounter something that seems necessary but isn't in the plan, do not add it. Stop and output:
"UNPLANNED DEPENDENCY in chunk [N]: [what you think is needed and why]"

Do not modify any files in .impl-chain/acceptance-tests/.

After completing all chunks, output "IMPLEMENTATION COMPLETE" and list all staged commits.
```

**On completion**:
- If IMPLEMENTATION COMPLETE: save the commit list to `.impl-chain/phase4-commits.md` and proceed to Phase 5
- If STUCK: decide based on the failure details:
  - If it looks like a code problem, re-spawn Phase 4 for just that chunk with additional guidance
  - If it looks like a plan problem, hand back to Phase 2 with the failure context for re-planning that chunk, then re-run Phase 3 and Phase 4 for the affected chunk
  - Cap at **2 escalation cycles** to Phase 2. If still stuck, halt for human review.
- If UNPLANNED DEPENDENCY: halt and show the human. They can either update the spec/plan or tell you how to proceed.

---

## Phase 5: Run Acceptance Tests

**Spawn subagent with skill**: `run-acceptance-tests`

**Context to provide**:
- The acceptance test scripts from Phase 1 (paths only — the agent just runs them)

**Context to NOT provide**:
- The spec
- The plan
- The codebase (beyond what's needed to run the tests)
- Any prior phase outputs

**Subagent prompt**:
```
Read the skill at [path to run-acceptance-tests/SKILL.md].

Run every acceptance test script in [repo-root]/.impl-chain/acceptance-tests/.

For each test, report:
- Test name
- The spec behaviour it verifies
- PASS or FAIL
- If FAIL: expected result vs actual result

Output a summary: [X/Y passed].
```

**On completion**:
- Save results to `.impl-chain/phase5-acceptance-results.md`
- If all pass: proceed to Phase 6
- If any fail: route the failure details back to Phase 4

**Acceptance test failure loop**:
- Re-spawn Phase 4 scoped to fixing the failing behaviours. Provide the failure details (which test, expected vs actual) as context.
- The implementation agent gets 3 attempts to fix.
- After fixing, re-run Phase 5.
- If still failing after the implementation agent's retries, escalate to Phase 2 to re-plan the affected chunks.
- Cap at **2 escalation cycles** to Phase 2. If still failing, halt for human review:
  ```
  Acceptance tests are still failing after re-planning. This likely indicates a spec ambiguity or a fundamentally wrong approach.
  
  Failing tests:
  [list failures with expected vs actual]
  
  Would you like to review the spec, adjust the acceptance tests, or inspect the implementation?
  ```

---

## Phase 6: Final Code Review

**Spawn subagent with skill**: `review-implementation`

**Context to provide**:
- The spec document
- The implementation plan (approved version)
- The full git diff of all staged changes (`git diff --staged`)
- Repository architecture docs, CLAUDE.md, coding guidelines

**Context to NOT provide**:
- Phase 1-5 logs and intermediate outputs
- Acceptance test results (the reviewer judges the code on its own merits, not on whether tests passed)

**Subagent prompt**:
```
Read the skill at [path to review-implementation/SKILL.md].

Here is the spec document:
[spec contents]

Here is the implementation plan that was followed:
[plan contents]

Here is the full diff of all changes:
[git diff --staged output]

Here is the repository context:
- Architecture docs: [contents]
- Coding guidelines: [contents]

Review this implementation. For each concern, specify:
- Which file(s)
- What the issue is
- Severity: "must fix" or "suggestion"

Output one of:
- "APPROVED" — no must-fix issues
- "APPROVED WITH SUGGESTIONS" — no must-fix issues, but suggestions listed
- "CHANGES REQUESTED" — must-fix issues listed with specific feedback
```

**On completion**:
- Save the full review to `.impl-chain/phase6-review.md`
- If APPROVED or APPROVED WITH SUGGESTIONS: proceed to Phase 7
- If CHANGES REQUESTED: route the must-fix issues back to Phase 4

**Code review fix loop**:
- Re-spawn Phase 4 with the reviewer's must-fix feedback as additional context, scoped to the specific files flagged
- After fixes, re-run Phase 5 (acceptance tests must still pass after fixes)
- Then re-run Phase 6 — the reviewer does a second pass on just the fixes
- Cap at **2 revision cycles**. If the reviewer still requests changes, halt for human review.

---

## Phase 7: Review Summary

**Spawn subagent with skill**: `review-summary`

**Context to provide**:
- The spec document
- Phase 1 assumptions log (`.impl-chain/phase1-assumptions.md`)
- The implementation plan
- Phase 3 risk flags (`.impl-chain/phase3-risk-flags.md`) if they exist
- The full git diff of all staged changes
- Phase 5 acceptance test results (`.impl-chain/phase5-acceptance-results.md`)
- Phase 6 review output (`.impl-chain/phase6-review.md`)
- Phase 4 commit list (`.impl-chain/phase4-commits.md`)
- Any retry/escalation logs from the orchestrator's handling of failures

**Context to NOT provide**:
- Nothing held back. This is the one phase that gets the full picture, because its job is to assemble a comprehensive summary for the human.

**Subagent prompt**:
```
Read the skill at [path to review-summary/SKILL.md].

Generate a REVIEW.md for the human to review before committing. Here is the context from all phases:

Spec document:
[spec contents]

Acceptance test assumptions (Phase 1):
[contents of phase1-assumptions.md, or "None"]

Implementation plan:
[plan contents]

Plan review risk flags (Phase 3):
[contents of phase3-risk-flags.md, or "None"]

Full diff of all changes:
[git diff --staged output]

Acceptance test results (Phase 5):
[contents of phase5-acceptance-results.md]

Code review output (Phase 6):
[contents of phase6-review.md]

Staged commits:
[contents of phase4-commits.md]

Retry/escalation log:
[any retry events you tracked, or "No retries needed"]

Write REVIEW.md to the repository root: [repo-root]/REVIEW.md
```

**On completion**:
- Present the REVIEW.md to the human
- Print a final summary:
  ```
  Implementation chain complete.
  
  Acceptance tests: [X/Y passed]
  Code review: [APPROVED / APPROVED WITH SUGGESTIONS]
  Risk callouts: [count, or "None"]
  Retries needed: [count, or "None"]
  
  Review REVIEW.md, then commit when ready.
  ```

---

## Retry and Escalation Tracking

Throughout the chain, maintain a log at `.impl-chain/escalation-log.md`. For every retry or escalation event, append:

```markdown
## [Timestamp]
**Phase**: [which phase]
**Event**: [RETRY | ESCALATE_TO_PHASE_N | HALT_FOR_HUMAN]
**Chunk**: [which plan chunk, if applicable]
**Reason**: [what failed and why]
**Attempt**: [N of max]
**Resolution**: [what happened — fixed, re-planned, or halted]
```

This log is passed to Phase 7 for inclusion in the REVIEW.md risk callouts.

## Retry Cap Summary

| Failure | First retry target | Escalation target | Max cycles before human halt |
|---------|-------------------|-------------------|------------------------------|
| Phase 4: tests don't pass | Phase 4 (3 self-fix attempts) | Phase 2 (re-plan chunk) | 2 escalations to Phase 2 |
| Phase 5: acceptance tests fail | Phase 4 (fix behaviour) | Phase 2 (re-plan chunk) | 2 escalations to Phase 2 |
| Phase 6: changes requested | Phase 4 (fix flagged files) | — | 2 review cycles |
| Phase 3: plan rejected | Phase 2 (re-plan with feedback) | — | 2 re-plans |

## Cleanup

After the human commits (or discards), the `.impl-chain/` directory can be deleted. It's a scratch space for the chain run, not a permanent artifact. The only permanent output is `REVIEW.md` in the repo root (and the human can delete that after committing too).
