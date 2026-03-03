---
name: plan-implementation
description: Use this skill to turn a pre-existing plan/spec into a series of sequential implementation steps
---

# Overview

Write a comprehensive implementation plan for an AI agent assuming the agent has zero context for our codebase. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. Do NOT create any new features in anticipation of future use; if its not in the spec dont do it. Follow YAGNI. Ensure that there will be no duplication of code in the implementation, explore the existing repo deeply reuse what you can without violating the architectural principles of the repository.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. When describing tests, be explicit about what to assert and why; don't assume the agent will infer the right test boundaries.

# Setup
Explore the repository to understand what skills are available to you, what the testing procedures are, how to start/stop/interact with the infrastructure (if applicable). Make a note of these and include the specific instructions with each task where necessary. I.e. Make sure when you say to run the integration tests, you inform the agent of which skill or command to use.

If there are any setup instructions (package installation, infrastructure setup via docker, etc) make sure this is included at the start of that implementation section or as a preface to the whole implementation plan. Give specific instructions and conditions for continuing.

# Plan

## Bite-Sized Task Granularity

**Each step should be a single, verifiable action:**
- "Write the failing test(s)" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Commit" - step

## Testing
The whole implementation should include all three levels of testing:

### Unit Tests
Write unit tests that can be run without any extra infrastructure. These should test atomic units of logic and should only be written when the task involves a single function or class. These are the minimum required tests.

### Integration Tests
Write integration tests that cover larger sections of the system and verify that multiple elements work together correctly, potentially involving infrastructure or containers. Sprinkle these throughout the implementation as you build.
### Acceptance Tests

Acceptance tests validate the system from the outside, the way a user or operator would interact with it. They use bash commands (curl, CLI invocations, database queries) against a running system, **not Python method calls against instantiated classes**. NEVER re-skin integration tests as acceptance tests. If the feature doesn't have an external interface yet, the acceptance test should go through whatever entry point is closest to the real one (CLI, API route, message queue).

If the feature genuinely has no external entry point (no API route, CLI, or message queue), acceptance tests should verify the infrastructure layer independently after starting the service. Keep infrastructure verification focused on business-critical constraints (indexes, partial indexes, foreign keys, unique constraints) rather than exhaustive schema inspection. If the migration ran and the integration tests pass, the column definitions are correct. Defer full acceptance testing to the task that adds the external interface.

When full acceptance testing is possible, complete at least one or two acceptance tests after finishing the implementation to test new features and to ensure that no regressions were introduced in previous functionality. Use the files targeted by the specification to determine how to structure these tests. Write each acceptance test in plain language explaining what should be done and why, providing exact bash commands along with expected output. For example, if you made changes to a certain API route, provide instructions to test that route, including the curl command, input data, and expected output data.

Acceptance tests should be their own step, aside from implementation. There should be final acceptance tests before an implementation is considered complete. If there is a halfway point where a new feature is complete or a major change is made, you should gate the continuation of the plan on some acceptance tests. You don't need to commit after acceptance tests unless changes were made to fix something.

## Plan Document Structure

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** Use executing-plans skill to implement this plan task-by-task.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

**Setup:** [Key setup (environment, docker, .env, etc)]

**Key Tasks:**
  - List of
  - Tasks and
  - 1 sentence
  - describing them

---
```

## Task Structure
Tasks should be structured to follow TDD; write tests for the feature, verify they fail then implement and re-test.

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
Add the files you've created and updated `git add -u` and `git add <specific new file>`
Call `git commit -m "<a short but descriptive implementation message>"`

## Disclaimer
- Use short but informative commit messages
- Add only the files you just edited
- Never proceed if there are git errors
- Never commit implementation plans
```
