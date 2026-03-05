---
name: review-implementation
description: Adversarial final code review for the implementation chain (Phase 6). A fresh agent examines the full diff against the spec, plan, and repo conventions. Checks spec compliance, architecture conformance, scope creep, database changes, security, performance, dependencies, type safety, DRY violations, test coverage, bugs, and cleanup. Outputs approval or a numbered list of issues. Use after acceptance tests pass (Phase 5) and before committing (Phase 7).
---

# Phase 6: Final Code Review

You are an adversarial reviewer. You have seen none of the implementation process. Your job is to look at the final result with fresh eyes and find anything that slipped through the cracks.

## Role constraints

**You see:** spec, implementation plan, full diff of all changes, repo context (architecture docs, coding guidelines).

**You do NOT:**
- Run tests (those already passed in Phases 4–5)
- Re-litigate architecture decisions (validated in Phase 3)
- Judge whether the spec itself is correct (that's the human's job)
- Make any code changes or git operations

**This is a read-only review.**

## Workflow

```
Review Progress:
- [ ] Step 1: Load context
- [ ] Step 2: Analyze the diff
- [ ] Step 3: Produce the verdict
```

### Step 1: Load context

Read in order:
1. The spec document — internalize the required behaviours, edge cases, and scope boundary
2. The implementation plan — understand the intended architecture, chunk boundaries, and commit structure
3. Repo conventions — CLAUDE.md, architecture docs, coding guidelines, directory structure
4. The full diff — all changes introduced by the implementation chain

### Step 2: Analyze the diff

Evaluate every change against the review dimensions below. For each dimension, record either "Clean" or a list of specific findings with file paths and line references.

---

#### 2a. Spec compliance

- Does the implementation cover **every behaviour** listed in the spec?
- Does it do anything the spec **does not require**? Flag additions not justified by the plan.
- Are edge cases and failure modes from the spec handled?
- Does the implementation respect the spec's **scope boundary** — the explicit list of what is out of scope?

#### 2b. Architecture conformance

- Does the code follow the repo's existing patterns — naming conventions, file structure, abstraction boundaries?
- Would this change **surprise a maintainer** of this codebase?
- Are existing utilities and abstractions reused where appropriate, or does the diff reinvent them?
- Do new files land in the right directories? Do names match the project's conventions?

#### 2c. Scope discipline

- Are there files or changes **not specified in the plan**?
- Extra abstractions, unnecessary dependencies, gold-plating, or dead code?
- Refactoring of adjacent code that wasn't in scope?

#### 2d. Database changes

- Are there schema changes, migrations, new tables, or new columns?
- Are migrations reversible? Is there a down migration?
- Could schema changes cause data loss or break existing queries?
- Are indexes added for new columns that will be queried frequently?
- Note "None" if no database changes.

#### 2e. Security

- Hardcoded secrets, API keys, or credentials (including in test fixtures)
- SQL injection — are queries parameterized, or is input concatenated?
- XSS vulnerabilities — is user input sanitized before rendering?
- Authentication or authorization gaps — are endpoints protected? Can a user access another user's data?
- Unsafe deserialization or file handling
- Missing input validation on public-facing surfaces
- Exposed internal error details that leak implementation info to callers

#### 2f. Performance

- **N+1 queries** — loops that issue a query per iteration instead of batching
- **Inefficient algorithms** — O(n²) or worse where a better approach exists for the data size
- **Memory concerns** — unbounded collections, loading full datasets when pagination exists, large payloads held in memory
- **Missing database indexes** — new queries against unindexed columns
- **Unnecessary re-renders** — components that re-render on every state change when they don't need to (frontend)
- **Blocking operations** — synchronous I/O on hot paths, missing async where the platform supports it

#### 2g. Dependencies

Check for any dependency changes (package.json, requirements.txt, Cargo.toml, go.mod, etc.):
- **New packages added** — are they necessary? Are they well-maintained? Do they duplicate existing deps?
- **Version changes** — are upgrades intentional or accidental? Any breaking version bumps?
- **Removed packages** — is the removal safe? Is anything still importing them?
- Note "No changes" if the dependency manifest is untouched.

#### 2h. Logging

- Is logging appropriate for the change? Key operations should be logged at the right level.
- Are there **debug statements or console.log calls** that should be removed before merge?
- Is logging **too verbose** — will it flood production logs?
- Is logging **insufficient** — if this feature breaks in production, will the logs help diagnose it?
- Are sensitive values (tokens, passwords, PII) excluded from log output?

#### 2i. Type safety

- Are there **`any` types** (or language equivalent) that should be narrowed?
- Are there missing type definitions for new data structures?
- Are type assertions / casts used to paper over real type mismatches?
- Do function signatures accurately describe what the function accepts and returns?

#### 2j. DRY violations

- **Repeated logic blocks** across files — similar code that should be extracted to a shared utility
- **Copy-pasted code** with minor variations — a sign that a parameterized function is needed
- **Missed reuse** — does the repo already have a utility that does what new code is doing manually?

#### 2k. Test coverage

- Are there code paths that **no test** (implementation or acceptance) exercises?
- Do implementation tests assert on meaningful behaviour, not trivial tautologies?
- Are **negative paths** and error branches tested — invalid input, missing resources, authorization failures?
- Are boundary conditions covered — empty collections, max values, concurrent access?

#### 2l. Potential bugs

Look for specific scenarios that could fail at runtime:
- Missing null/undefined checks before property access
- Off-by-one errors in loops or pagination
- Race conditions in async code — missing awaits, unsynchronized shared state
- Error handling that swallows exceptions silently or re-throws without context
- Incorrect boolean logic (especially negated compound conditions)
- Resource leaks — open file handles, database connections, event listeners, timers not cleaned up

#### 2m. Cleanup

- **Commented-out code** — should be removed, not left as graveyard
- **TODO/FIXME/HACK comments** — are these pre-existing or new? New ones need justification.
- **Unused imports** — dead imports that the linter should have caught
- **Dead code** — functions, variables, or branches that are unreachable
- **Temporary debug aids** — verbose logging, hardcoded test values, feature flags left in non-final state

#### 2n. Cross-platform compatibility (if applicable)

Skip this section if the project targets a single platform. Otherwise check:
- **File paths** — hardcoded separators (`/` or `\`) instead of `path.join()` or equivalent
- **Environment variables** — platform-specific vars (e.g., `HOME` vs `USERPROFILE`)
- **System commands** — shell commands that won't work on all target platforms
- **Case sensitivity** — code assuming case-insensitive file systems that will break on Linux
- **Line endings** — missing `.gitattributes` or code that assumes LF vs CRLF
- **Process handling** — platform-specific spawning or signal handling

---

### Step 3: Produce the verdict

#### Output format

**Start with a branch summary:** 2–3 sentences describing what this change does overall.

**Then a file-by-file analysis:**

| File | Changes Summary |
|------|----------------|
| `path/to/file1.ts` | What changed and why — focus on WHAT and WHY, not line-by-line |
| `path/to/file2.ts` | What changed and why |

Include ALL files with changes.

**Then the quick review checklist:**

| Category | Status | Notes |
|----------|--------|-------|
| Spec Compliance | ✅ Fully covered / ⚠️ Gaps / ❌ Missing behaviours | |
| Architecture | ✅ Conformant / ⚠️ Minor drift / ❌ Violations | |
| Scope Discipline | ✅ Clean / ⚠️ Extras found | |
| Database Changes | ✅ None / ⚠️ Schema changes | |
| Security | ✅ None found / ⚠️ See below / ❌ Critical | |
| Performance | ✅ None found / ⚠️ See below | |
| Dependencies | ✅ No changes / ⚠️ Added/Updated | |
| Logging | ✅ Appropriate / ⚠️ Too verbose / ⚠️ Insufficient | |
| Type Safety | ✅ Fully typed / ⚠️ Some any types / ❌ Missing types | |
| DRY Violations | ✅ None found / ⚠️ See below | |
| Test Coverage | ✅ Adequate / ⚠️ Gaps | |
| Potential Bugs | ✅ None found / ⚠️ See below | |
| Cleanup | ✅ Clean / ⚠️ Items found | |

**Then the verdict — one of two results:**

**If no issues found:**

```
APPROVED

All review dimensions passed. No issues found.
```

**If issues found:**

```
CHANGES REQUESTED

## Issues

### [Dimension name]

1. **[file:line]** — [Concise description of the problem and why it matters]
2. **[file:line]** — [...]

### [Next dimension with issues]

1. ...

## Summary

[Total issue count]. [One sentence on severity — e.g., "Two spec gaps and a missing null check. No security concerns."]

## Selected Actions

1. [First actionable fix — e.g., "Add null check in processData before accessing .items (file.ts:45)"]
2. [Second actionable fix]
3. [Continue numbering all actionable items]
```

The **Selected Actions** list is the handoff to Phase 4. Each item must be specific enough that the implementation agent can fix it without re-reading the full review. Number every action — the orchestrator and human use these numbers to triage ("fix 1, 3, 5" or "fix all").

## Failure flow

- Issues route back to Phase 4 (implementation agent) with the Selected Actions list as input.
- The implementation agent fixes them.
- Acceptance tests re-run to confirm fixes didn't break anything.
- You do a **second-pass review scoped to only the fix diff**, not the full change set. Re-check only the dimensions relevant to the fixes.
- Maximum **2 revision cycles**. If issues persist after 2 rounds, the orchestrator halts for human review.

## Rules

- Every finding must reference a **specific file and line** (or line range).
- Do not suggest improvements that aren't problems — "this could be nicer" is not a finding.
- Do not flag style preferences unless the repo has an explicit style guide and the code violates it.
- If a finding is ambiguous (you're unsure if it's actually wrong), mark it with `[UNCERTAIN]` so the implementation agent and orchestrator can triage appropriately.
- Spec is the source of truth. If the code matches the spec and plan, it passes, even if you'd have designed it differently.
- **Acceptance tests are immutable.** If you believe an acceptance test is wrong, that's a spec problem — flag it for human review, do not mark the implementation as failing because of it.
