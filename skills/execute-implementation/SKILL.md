---
name: execute-implementation
description: Use when executing implementation plans with independent tasks in the current session
---

# Subagent-Driven Development

Execute a plan by dispatching a fresh subagent per task, with two-stage review after each: spec compliance first, then code quality.

**Core principle:** Fresh subagent per task + two-stage review (spec then quality) = high quality, fast iteration.

## When to Use

Use this skill when all three conditions are true:
- You have an existing implementation plan with discrete tasks
- The tasks are mostly independent of each other (not tightly coupled)
- You want to execute within the current session

If tasks are tightly coupled, execute manually or brainstorm further before dispatching. If you need parallel sessions, use `executing-plans` instead.

## Example Workflow

```
You: Executing implementation plan using subagents.

[Read plan file once: docs/plans/feature-plan.md]
[Extract all 5 tasks with full text and context]
[Create TodoWrite with all tasks]

Task 1: Feature Implementation

Implementer: "Got it. Implementing now..."
[Later] Implementer:
  - Implemented install-hook command
  - Added tests, 5/5 passing
  - Self-review: Found I missed --force flag, added it
  - Committed

[Dispatch spec compliance reviewer]
Spec reviewer: ✅ Spec compliant - all requirements met, nothing extra

[Dispatch code quality reviewer]
Code reviewer: Strengths: Good test coverage, clean. Issues: None. Approved.

[Mark Task 1 complete]

Task 2: Recovery modes

[Dispatch implementer with full task text + context]

Implementer: [No questions, proceeds]
Implementer:
  - Added verify/repair modes
  - 8/8 tests passing
  - Self-review: All good
  - Committed

[Dispatch spec compliance reviewer]
Spec reviewer: ❌ Issues:
  - Missing: Progress reporting (spec says "report every 100 items")
  - Extra: Added --json flag (not requested)

[Implementer fixes issues]
Implementer: Removed --json flag, added progress reporting

[Spec reviewer reviews again]
Spec reviewer: ✅ Spec compliant now

[Dispatch code quality reviewer]
Code reviewer: Issues (Important): Magic number (100)

[Implementer fixes]
Implementer: Extracted PROGRESS_INTERVAL constant

[Code reviewer reviews again]
Code reviewer: ✅ Approved

[Mark Task 2 complete]

...

[After all tasks]
[Dispatch final code-reviewer for entire implementation]
Final reviewer: All requirements met, ready to merge

Done!
```

## The Process

## Prompt Templates

- `~/.claude/skills/execute-implementation/implementer-prompt.md` — Dispatch implementer subagent
- `~/.claude/skills/execute-implementation/spec-reviewer-prompt.md` — Dispatch spec compliance reviewer subagent

### 1. Setup
Read the plan file once and extract all tasks with their full text and context. Create a TodoWrite with all tasks. Do not make subagents read the plan file themselves — always provide the full task text and surrounding context directly.

### 2. Per Task

For each task, follow this sequence:

**Dispatch an implementer subagent** using `implementer-prompt.md`. Include the full task text, relevant context about where the task fits in the overall plan, and any outputs or decisions from previous tasks.

**If the subagent asks questions**, answer them clearly and completely before letting them proceed. If you don't have enough context to answer confidently, ask the user rather than guessing.

**The implementer implements, tests, commits, and self-reviews.** Self-review is valuable but does not replace the formal review stages below.

**Dispatch a spec compliance reviewer** using `spec-reviewer-prompt.md`. The spec reviewer confirms the implementation matches the spec — nothing missing, nothing extra. Spec compliance must pass before moving to code quality review. If the spec reviewer finds issues, the implementer fixes them and the spec reviewer reviews again. Repeat until approved.

**Dispatch a code quality reviewer** using the `requesting-code-review` skill. Provide the implementer's summary of what was implemented, the task description, and the git SHAs from before and after the implementer's commits. The code quality reviewer checks for clean code, good patterns, and maintainability. Critical and Important issues must be fixed and re-reviewed. Minor issues are at your discretion.

**Mark the task complete** in TodoWrite and move to the next task.

### 3. Completion
After all tasks are complete, dispatch a final code quality reviewer for the entire implementation and validate the acceptance tests.

## Hard Constraints

These will break the process if violated:

- Never skip either review stage (spec compliance AND code quality are both required for every task).
- Never start code quality review before spec compliance is approved. Always spec first, then quality.
- Never proceed to the next task while either review has open issues.
- Never dispatch multiple implementation subagents in parallel. They will create conflicts.
- Never accept "close enough" on spec compliance. If the reviewer found issues, the task is not done.

## Best Practices

These keep quality high but won't break the process if occasionally bent:

- Don't make subagents read the plan file. Provide the full task text directly to avoid misinterpretation or wasted context.
- Always include scene-setting context so the subagent understands where their task fits in the larger implementation.
- If a subagent fails a task, dispatch a new fix subagent with specific instructions rather than fixing it manually in the orchestrator session. Fixing manually bleeds implementation details into your orchestrator context, making subsequent task dispatch less clean.
- If a review loop exceeds 3 iterations, pause and reassess. The task description may need clarification or the task may need to be broken down further. Ask the user if you're unsure.

## Integration

**Required workflow skills:**
- **plan-implementation** — Creates the plan this skill executes
- **requesting-code-review** — Code review template for reviewer subagents
- **finishing-a-development-branch** — Completes development after all tasks

**Subagents should use:**
- **test-driven-development** — Subagents follow TDD for each task
