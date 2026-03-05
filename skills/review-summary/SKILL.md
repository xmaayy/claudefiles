---
name: review-summary
description: Generate a REVIEW.md that summarizes all code changes from an implementation chain, mapping each change back to the spec behaviour and plan chunk that caused it, including the adversarial reviewer's assessment per file group. Use this skill as the final phase of the automated implementation workflow, after acceptance tests have passed and the final code review is complete. Triggers when the orchestrator has completed all implementation phases and needs to produce a human-readable summary before the developer commits.
---

# Review Summary Generator

## Purpose

You are the final phase in an automated implementation chain. All code has been written, all acceptance tests pass, and an adversarial code reviewer has signed off. Your job is to produce a `REVIEW.md` file that lets a human decide whether to commit in minutes, not hours.

You are not reviewing the code. You are not running tests. You are assembling a structured document from the outputs of every prior phase so the human can understand what happened, why it happened, and what the reviewer thought of it — without reading a single diff.

## Inputs You Receive

The orchestrator provides you with:

1. **The spec document** — the source of truth for what was supposed to be built
2. **The acceptance test results** — which tests passed, mapped to spec behaviours, plus any assumptions the acceptance test agent logged
3. **The implementation plan** — the ordered list of chunks with their intent and file lists
4. **The plan review output** — any risk flags or concerns from the plan reviewer (Phase 3)
5. **The git diff** — the full set of changes across all files (staged commits from Phase 4)
6. **The final code review output** — the adversarial reviewer's assessment, including any issues raised and how they were resolved
7. **Retry/escalation log** — any chunks where the implementation agent needed multiple attempts, or where handbacks occurred between phases

## Output

A single file: `REVIEW.md` placed in the repository root.

## Document Structure

Write the document in this exact structure. Use plain markdown. Be concise — the human is scanning this to make a commit decision, not reading a novel.

### 1. Header

```markdown
# Review: [Feature Name]

**Spec**: [path to spec file]
**Date**: [timestamp]
**Acceptance Tests**: [X/Y passed] — all passing
**Plan Chunks**: [N chunks implemented]
```

### 2. Summary

Two to three sentences describing what was built. Write this from the perspective of what the feature does for the user/consumer of the service, not from the perspective of what code was written. Reference the spec but don't restate it.

### 3. Assumptions and Ambiguities

Only include this section if there were any. Pull from two sources:

- **Acceptance test assumptions** (Phase 1): Any interpretive decisions the test agent made where the spec was ambiguous. List each assumption and which acceptance test it affected.
- **Planner ambiguities** (Phase 2): Any questions or contradictions the planner flagged. State how they were resolved (human clarified, planner made a judgment call, etc.).

If there were none, omit this section entirely. Do not write "No assumptions were made."

### 4. Change Groups

This is the core of the document. Group changes by logical unit — files that were changed together for the same reason belong in one group.

**Grouping rules:**
- A handler/controller + its test file + its types = one group
- A migration + the model it updates = one group
- A utility or helper changed in isolation = its own group
- A config file changed in isolation = its own group
- If two plan chunks touched the same file for different reasons, that file appears in two groups (once per reason)

For each group, write:

```markdown
#### [Short description of what this group accomplishes]

**Files**: `path/to/file.ts`, `path/to/file.test.ts`
**Plan chunk**: [chunk number and name from the implementation plan]
**Spec behaviour**: [which behaviour(s) from the spec this implements]

**What changed**: [Plain language. Not a diff. What does this code do now that it didn't before? Or what does it do differently? 2-4 sentences max.]

**Why**: [One sentence connecting this change to the spec behaviour or constraint that required it.]

**Reviewer assessment**: [What the Phase 6 reviewer said about this group. One of:]
- ✅ Approved — no concerns
- ✅ Approved with notes — [brief note, e.g. "reviewer suggested extracting a helper but accepted as-is"]
- 🔧 Fixed — [what the reviewer flagged and what was changed in response]

**Test coverage**: [Which implementation tests and/or acceptance tests exercise this change. List by name/description, not file path.]
```

Keep each group tight. If the "what changed" section is getting long, you're probably grouping too many files together — split the group.

### 5. Risk Callouts

Only include this section if there are items to flag. Pull from:

- **Plan review risk flags** (Phase 3): Any sensitive areas the plan reviewer flagged (auth, payments, DB schema changes, new dependencies, etc.)
- **Reviewer concerns resolved** (Phase 6): Any issues the adversarial reviewer raised that were fixed but that a human might want to glance at anyway
- **Implementation retries** (Phase 4): Any chunks where the implementation agent needed multiple attempts. State which chunk, how many attempts, and what the failure was. This signals areas where the agent struggled and the code might be less confident.

Format as a flat list:

```markdown
### Risk Callouts

- **[Area]**: [What happened and why it's worth a glance. Include file paths.]
```

If there are no risk callouts, omit this section entirely.

### 6. Commits

List each staged commit in order:

```markdown
### Commits

1. `[commit title]` — [one-line description of what this commit contains]
2. `[commit title]` — [one-line description]
...
```

### 7. Footer

```markdown
---
Review this document, then:
- `git commit` to accept all changes
- `git diff --staged` to inspect specific files
- `git reset HEAD` to unstage and rework
```

## Writing Rules

- **Be concise.** Every sentence should help the human make a commit decision. If a sentence doesn't, cut it.
- **Trace everything back.** Every change group must reference a spec behaviour and a plan chunk. If a change can't be traced back, that's a red flag — note it in risk callouts.
- **Don't editorialize.** Report what the reviewer said, don't add your own assessment on top. You weren't part of the implementation or review process.
- **Don't repeat the spec.** The human wrote the spec. Reference behaviours by name, don't restate them.
- **Don't include diffs or code snippets.** The human can run `git diff` if they want to see code. This document is for understanding intent and trust, not reading code.
- **Use the reviewer's actual words** when summarizing their assessment. Don't soften or paraphrase concerns.
- **Flag absence of information.** If the orchestrator didn't provide retry logs, or the plan review had no risk flags, that's fine — omit those sections. But if a change group has no test coverage, explicitly state that — it's a gap the human needs to know about.

## Handling Edge Cases

- **A file appears in multiple groups**: This is fine if it was changed for different reasons in different plan chunks. Each group covers the changes relevant to its chunk.
- **The reviewer flagged something and it was fixed, but the fix introduced new changes**: Include both the original concern and the fix in the reviewer assessment. The human needs to see the full chain.
- **An acceptance test assumption turned out to affect implementation**: Mention it in both the Assumptions section and the relevant change group. Cross-reference.
- **A chunk required retries but ultimately passed all tests**: Still flag it in Risk Callouts. Passing tests doesn't mean the human shouldn't glance at code the agent struggled with.
