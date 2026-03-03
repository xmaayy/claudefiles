---
name: trace-issue
description: "You MUST use this when debugging, fixing bugs, or investigating unexpected behavior. Performs thorough static tracing and root cause analysis before proposing any changes."
---

# Investigating and Fixing Issues

## Overview

Systematically investigate issues through static tracing and analysis before touching any code. Understand the problem fully, trace it to its root cause, then propose minimal, safe fixes that preserve existing logic and intent.

## The Process

**1. Understand the symptom:**
- Get a clear description of what's going wrong (expected vs actual behavior)
- Ask clarifying questions one at a time if the report is ambiguous
- Identify reproduction steps if available
- Check for recent changes that correlate with when the issue appeared (git log, recent commits)

**2. Static tracing:**
- Start from the symptom and trace backwards through the code
- Map out the relevant call chain: entry point → intermediate functions → where the bug likely lives
- Read each file involved carefully — don't skim
- Identify:
  - What data flows through the path
  - Where transformations or decisions happen
  - What assumptions the code makes
  - Where those assumptions could break
- Check adjacent code for related context: types, constants, config, tests
- Look for recent changes to any file in the call chain (`git log -p --follow <file>`)

**3. Narrow the root cause:**
- Form 2-3 hypotheses about what's going wrong
- For each hypothesis, identify what evidence would confirm or rule it out
- Trace through the code path with the failing input mentally or by reading tests
- Eliminate hypotheses until one remains, or identify what runtime information you'd need
- If you can't determine root cause statically, suggest specific targeted logging or reproduction steps — don't guess

**4. Present findings:**
- Explain the call chain you traced (keep it concise, use inline code references)
- State the root cause with supporting evidence from the code
- Explain *why* the current code behaves the way it does — what was the original intent?
- If the original intent is unclear, say so and ask before proceeding

## Proposing a Fix

**5. Propose the fix before implementing:**
- Describe the change in plain language
- Explain what the fix does and *doesn't* change
- Call out any existing logic, behavior, or intent that the fix preserves
- Flag anything you're unsure about — ask rather than assume
- If the fix touches shared code, note downstream callers that could be affected
- Wait for approval before writing any code

**6. Classify the change:**
- **Safe**: Fixes a clear bug without altering intended behavior (e.g., null check, off-by-one, typo)
- **Behavioral**: Changes how something works, even subtly — always requires explicit approval
- **Structural**: Moves or reorganizes code — must preserve exact existing behavior, requires approval

Never make a behavioral or structural change without stating it clearly and getting a go-ahead.

**7. Implement the fix:**
- Make the smallest change that addresses the root cause
- Do not refactor, rename, or "improve" surrounding code unless asked
- Do not change function signatures, return types, or public interfaces without approval
- Add or update tests that cover the specific failure case
- Verify the fix doesn't break existing tests

## Key Principles

- **Trace first, fix second** — No code changes until you understand the full call chain
- **Preserve intent** — The original author wrote it that way for a reason. Understand that reason before changing anything
- **Minimal diff** — Touch only what's necessary to fix the issue
- **No drive-by changes** — Don't sneak in refactors, style fixes, or "improvements"
- **Hypothesize and eliminate** — Don't jump to the first theory. Consider alternatives
- **Ask when uncertain** — If you're not sure about the original intent or the right fix, stop and ask
- **Show your work** — Present the trace and reasoning so the human can verify your logic
