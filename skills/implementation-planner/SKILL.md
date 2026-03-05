---
name: implementation-planner
description: Generates a detailed, task-by-task implementation plan from a spec. Use when a specification exists and needs to be broken into ordered, independently-verifiable implementation chunks. Produces plans consumable by a separate implementation agent with zero codebase context. Enforces TDD, scope discipline, and architecture alignment.
---

# Implementation Planning (Phase 2)

This skill produces an **implementation plan** — the bridge between a spec and code. The plan is consumed by a separate implementation agent that follows it mechanically. That agent sees the plan and the codebase, but NOT the spec. Therefore: if it's not in the plan, it won't be built. If it's ambiguous in the plan, it'll be built wrong.

Separately, an independent agent has already generated acceptance tests from the same spec (Phase 1). Neither the planner nor the implementer should read acceptance tests. Both the plan and the acceptance tests derive from the spec independently — convergence at Phase 5 validates that both interpretations are correct.

## Inputs

1. **Spec document** — source of truth for requirements
2. **Codebase context** — architecture docs, coding guidelines, existing patterns, directory structure

The planner does NOT see acceptance tests. This is deliberate — independent interpretation of the spec by both the planner and the test author gives stronger validation than cross-referencing.

## Setup

Before planning, deeply explore the repository:

- Identify testing tools, commands, and conventions (unit, integration, acceptance)
- Identify infrastructure setup (docker, env vars, package install)
- Read architecture docs, CLAUDE.md, coding guidelines
- Map existing patterns for similar features (file structure, abstractions, naming)
- Note start/stop/interaction commands for any services

Include specific setup instructions (package install, infra bootstrap, env config) as a preface to the plan or inline with the first task that needs them.

## Ambiguity Detection (Do This First)

Before writing the plan, do a pass for ambiguities and contradictions in the spec. If found, output them as a list of questions and **halt** — do not plan around ambiguities. Similarly, if something in the spec is technically infeasible, halt with an explanation rather than planning a workaround silently.

## Architecture Alignment

Read the repo's conventions and design the plan to **fit, not fight** existing structure:

- How are similar features structured? Reuse those patterns.
- What abstractions exist? Use them; don't invent new ones.
- Flag any conflict between the spec and current architecture.
- Explore the repo deeply — reuse code without violating architectural principles.
- Follow YAGNI: nothing speculative, nothing not in the spec.

## Plan Document Structure

Every plan MUST start with this header:

```markdown
# [Feature Name] Implementation Plan

> **For the implementation agent:** Follow this plan task-by-task. Do not deviate.
> Do not add scope. Flag blockers instead of working around them.

**Goal:** [One sentence]

**Architecture:** [2-3 sentences on approach and how it fits existing patterns]

**Tech Stack:** [Key technologies/libraries]

**Setup:** [Environment, docker, .env, package install — with exact commands and success criteria]

**Retry Policy:** 3 fix attempts per failing task. If still failing, escalate — do not proceed. If something seems needed but isn't in the plan, flag it — do not add it silently.

**Key Tasks:**
- Task 1: [one sentence]
- Task 2: [one sentence]
- ...

---
```

## Task Granularity

Each task = one commit's worth of coherent change. Each task is independently verifiable. Within a task, each step is a single action:

- Write the failing test(s) → one step
- Run to confirm failure → one step
- Implement → one step
- Run to confirm pass → one step
- Run linter/type checks/existing suite → one step
- Commit → one step

## Task Structure

```markdown
### Task N: [Descriptor]

**What this accomplishes:** [Plain language — what increment this represents]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py` (describe what changes and why)
- Test: `tests/exact/path/to/test.py`

**Step 1: Write failing test(s)**
[What to test, what to assert, and why. Be explicit about test boundaries —
don't assume the implementation agent will infer them. Favour unit tests for
speed; include integration tests when the task involves multiple components.]

**Step 2: Verify failure**
Run: `<exact command>`
Expected: FAIL — [specific expected failure, e.g. "ImportError: no module named X"]

**Step 3: Implement**
[Describe what the code must do, edge cases to handle, which existing patterns
to follow. Don't output full implementations — describe clearly. Only include
literal code for: interfaces/signatures that must stay consistent across tasks,
specific library calls, or prompts/config the user specified.]

**Step 4: Verify pass**
Run: `<same command as Step 2>`
Expected: PASS

**Step 5: Run full checks**
Run: `<linter>`, `<type checker>`, `<existing test suite>`
Expected: No regressions

**Step 6: Check verification criteria**
[Describe what observable behaviour should now work, from the consumer's
perspective. e.g. "POST /users with valid payload returns 201 and the user
appears in GET /users".]

**Step 7: Commit**
`git add -u && git add <specific new files>`
`git commit -m "<short descriptive message>"`
```

## Testing Strategy

Two levels are written as part of the plan:

**Unit tests** — within each task. Test atomic logic. No infrastructure needed. Minimum required.

**Integration tests** — in tasks involving multiple components. May require infrastructure.

Acceptance tests exist separately from Phase 1, authored by an independent agent that read the same spec. The planner and test author never see each other's work. Convergence is validated in Phase 5 when acceptance tests run against the finished implementation as a blind check.

Gate continuation on verification criteria at natural boundaries: after a major feature completes, after a breaking change, before the next major section.

## Scope Discipline

The implementation agent follows the plan literally. Enforce:

- No files or changes not specified in a task
- No refactoring adjacent code outside scope
- No speculative abstractions or "while we're here" improvements
- No new dependencies not explicitly listed
- If something seems necessary but isn't in the plan, the agent flags it — not adds it

## Coverage Mapping

Before finalizing the plan, verify that **every spec behaviour** maps to at least one task's verification criteria. If any behaviour is unaccounted for, the plan is incomplete.

## Rules

- Never commit implementation plans to the repo
- Short, informative commit messages
- Add only files just edited
- Never proceed past git errors
- Every task must have explicit test commands with expected outcomes
