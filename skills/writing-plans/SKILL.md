---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Save plans to:** `docs/plans/YYYY-MM-DD-<feature-name>.md`

## Bite-Sized Task Granularity

**Each step is one action (2-5 minutes):**
- "Write the failing test(s)" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Commit" - step

## Testing
Some tests will not be runnable without a container (e.x. database, integration, etc). If the task is a unit-testable (updating a type, small function, etc) you can run a unit test. If its more in depth (migration, inter-service call, etc) check to see if there is a custom command/skill for running tests or if there are instructions for running tests in the claude.md/readme. Look at existing tests to determine what is and is not unit testable.

If there are no instructions for coding agents to run tests AND tests would fail or otherwise block the implementer on checking the tests fail before implementation, skip the `Run test to verify it fails` step. There are no other exceptions for skipping this step.

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** Use executing-plans skill to implement this plan task-by-task.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

**Key Tasks:**
  - List of
  - Tasks and
  - 1 sentence
  - describing them

---
```

## Task Structure

```markdown
### Task N: [Task Descriptor]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Step 1: Write the new test(s) / Updates to Existing Tests**

Description of exactly what the test(s) should be testing and why thats relevant. Keep in mind that each new test is maintenance and review burden so they should not be overdone.

In the case we are changing existing functionality, we should also update the existing tests relevant to our changes to adhere to the new feature.

Favour unit tests for faster development where possible, but make sure that integration tests are created or exist for this feature.

**Step 2: Run test to verify it fails**
One of:
  Run: `pytest tests/path/test.py::test_name -v`
or:
  Run: <testing skill name>

Expected: FAIL with "function not defined"


**Step 3: Implementation**
 - What it needs to do, why it needs to do that and any foreseeable edge cases where relevant.
 - Describe the implementation in detail, but dont output a full implementation.
 - Prefer clear and concise explanations, only using code if explicit function calls are required (e.g. specific libraries, optimized paths, etc)
   - Exceptions are:
     - Prompts or details the user explicitly requested / mentioned
     - Interfaces (i.e. function declarations that should be kept consistent across tasks)
     - Important names or orther details that the user mentions
 - Each extra LOC and character are maintenance burden.

**Step 4: Run test to verify it passes**

Run: (same test as before)
Expected: PASS

**Step 5: Commit**
Add the files you've created and updated
Call `git commit -m "<a short but descriptive implementation message>"`
- Do not make the message too long
- ** DO NOT PROCEED IF YOU ENCOUNTER GIT ERRORS **
- Never run anything other than git add and commit.
- Never re-initialize the branch. There should be nothing out of the ordinary here.
- You should never proceed if we see a problem.
```


## Remember
- Exact file paths always
- Complete code in plan (not "add validation")
- Exact commands with expected output
- Reference relevant skills with @ syntax
- DRY, YAGNI, TDD, frequent commits


## Execution Handoff

After saving the plan, tell them where the plan was saved and what their options are:

**"Plan complete and saved to `docs/plans/<filename>.md`. Use either Subagents or `execute-plans` to execute it after review.**
