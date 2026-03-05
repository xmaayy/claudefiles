---
name: implementation-runner
description: Executes an approved implementation plan chunk by chunk using subagents. Each chunk follows TDD — write failing tests, implement until they pass, lint, verify against acceptance tests, commit. Dispatches one subagent per chunk sequentially. Use when you have an approved implementation plan and want to execute it with strict scope discipline and a retry budget.
---

# Implementing Plan

Execute an approved plan chunk by chunk via subagents. Each chunk is one commit's worth of coherent change, implemented with TDD and strict scope discipline.

**Core principle:** The implementer translates the plan into code mechanically. It does not make architectural decisions (those were made in planning) and does not reinterpret requirements (the plan is its source of truth).

## When to Use

- You have an approved implementation plan with ordered chunks
- Each chunk specifies: files to change, tests to write, verification criteria, commit message
- Acceptance tests exist and are immutable

## The Process

### 1. Setup

Read the plan once. Extract all chunks with their full text. Create a todo list tracking each chunk.

Do not make subagents read the plan file — always paste the full chunk text directly into the subagent prompt.

### 2. Per Chunk

For each chunk, in order:

**Dispatch an implementer subagent** using `implementer-prompt.md`. Include the full chunk text, the working directory, and context about what previous chunks accomplished.

**If the subagent asks questions**, answer them. If you lack context to answer confidently, ask the user rather than guessing.

**The implementer executes TDD:**
1. Writes implementation tests from the chunk — runs them — they should fail
2. Writes the implementation — runs tests — they should pass
3. Runs linter, type checks, existing test suite — no regressions
4. Checks chunk verification criteria (specified acceptance test subset)
5. Self-reviews before reporting
6. Stages a git commit with the plan's commit message

**On failure — retry budget:**
- 3 attempts to fix failing tests, informed by specific failure output
- After 3 failures, escalate to the orchestrator with failure context
- The orchestrator either re-plans that chunk or halts for human review

**Mark the chunk complete** and move to the next.

### 3. Completion

After all chunks are implemented, run the full acceptance test suite and report results. The orchestrator handles what comes next (code review, commit finalization).

## Scope Discipline

The implementer MUST NOT:
- Add files or changes not specified in the chunk
- Refactor adjacent code not in scope
- Modify acceptance tests under any circumstances
- Skip writing tests
- Add dependencies not in the plan

If something seems necessary but isn't in the plan, the implementer flags it in their report. The orchestrator decides whether to re-plan or proceed.

## Handling Subagent Failures

If a subagent fails a chunk after retries, dispatch a **new** fix subagent with specific failure context rather than fixing it manually in the orchestrator session. Fixing manually leaks implementation details into orchestrator context.

If a fix loop exceeds 3 iterations, pause and reassess — the chunk may need to be broken down further or the plan may need revision. Ask the user if unsure.

## Prompt Templates

- `implementer-prompt.md` — Subagent prompt for implementing a single chunk
