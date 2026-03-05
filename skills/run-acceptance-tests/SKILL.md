---
name: running-acceptance-tests
description: Executes the acceptance test suite from Phase 1 against the completed implementation from Phase 4. Handles environment readiness, runs the test runner, produces structured pass/fail results mapped to spec behaviours, and formats failure context for the orchestrator's handback routing. Use when implementation chunks are committed and ready for verification, when re-running after implementation fixes, or for the final sanity check before commit. This phase sees no spec, no plan, and no source code — it operates the runner and reports what happened.
---

# Running Acceptance Tests

This phase executes the acceptance tests written in Phase 1 against the implementation produced in Phase 4. It is the moment of truth: does the code satisfy the spec as understood by an independent agent that never saw the implementation?

This agent is deliberately isolated. It does not read the spec, the plan, or the source code. It runs scripts and reports outcomes. If a test fails, this agent does not diagnose why — it produces structured output that the orchestrator routes to the right place.

## What This Agent Sees

- The acceptance test suite (scripts, runner, helpers) produced in Phase 1
- The running system (all implementation chunks committed, services started)
- Environment configuration (base URL, auth credentials, feature flags)

It does NOT see:

- The spec document
- The implementation plan
- Source code or diffs
- Output from any other agent

This isolation is the point. If this agent could read the source, it might rationalize failures. It can't, so it won't.

## Environment Readiness

Before running any tests, confirm the system is in a testable state. A test suite that fails because the server isn't running produces noise, not signal.

### Readiness checklist

```bash
#!/usr/bin/env bash
set -euo pipefail

BASE_URL="${BASE_URL:-http://localhost:8000}"
MAX_WAIT=30
INTERVAL=2

echo "Checking environment readiness..."

# 1. Is the base URL reachable?
elapsed=0
until curl -sf -o /dev/null "$BASE_URL/health" 2>/dev/null; do
    if [ "$elapsed" -ge "$MAX_WAIT" ]; then
        echo "ENVIRONMENT NOT READY: $BASE_URL/health not reachable after ${MAX_WAIT}s"
        echo "ACTION: Orchestrator should verify services are running"
        exit 2  # exit code 2 = environment problem, not test failure
    fi
    sleep "$INTERVAL"
    elapsed=$((elapsed + INTERVAL))
done
echo "  ✓ Base URL reachable"

# 2. Are required env vars set?
REQUIRED_VARS=("BASE_URL" "AUTH_TOKEN")  # adjust per project
for var in "${REQUIRED_VARS[@]}"; do
    if [ -z "${!var:-}" ]; then
        echo "ENVIRONMENT NOT READY: $var is not set"
        exit 2
    fi
done
echo "  ✓ Required env vars present"

# 3. Is the test suite where we expect it?
if [ ! -f "${ACCEPTANCE_TEST_DIR:-./checks}/run.py" ]; then
    echo "ENVIRONMENT NOT READY: Test runner not found at ${ACCEPTANCE_TEST_DIR:-./checks}/run.py"
    exit 2
fi
echo "  ✓ Test suite found"

echo "Environment ready."
```

The key convention: **exit code 2 means environment problem, exit code 1 means test failure, exit code 0 means all clear.** The orchestrator uses this distinction. An environment problem means "don't retry the tests, fix the infrastructure." A test failure means "the implementation is wrong."

If readiness fails, stop. Do not run the suite and report a wall of connection errors as "test failures." Report the environment problem and let the orchestrator handle it.

## Running the Suite

Phase 1's test suite already has a runner (`run.py`) that handles layer ordering, intra-layer short-circuiting, and `CheckResult` collection. This agent operates that runner — it doesn't reimplement it.

### Standard invocation

```bash
cd "$ACCEPTANCE_TEST_DIR"
python run.py \
    --base-url "$BASE_URL" \
    --report-file results.md \
    2>&1 | tee run_output.log

TEST_EXIT=$?
```

Always capture full output to a log file. If the runner crashes (as opposed to reporting failures gracefully), the log is the only diagnostic the orchestrator gets.

### How the runner behaves

The runner executes layers sequentially and **stops at the first layer that has any failure.** Within a layer, checks early-return when downstream steps depend on earlier state. This means a single run only reports the *first* problem area — if the health layer fails, you learn nothing about feature layers.

This is the right tradeoff. If conversations are broken, there's no value in testing skills that depend on conversations. The cost is that after fixing one layer's failures, you must re-run the full suite to discover whether deeper layers also have problems. Each re-run peels back one layer of failures.

### Runner crashes vs test failures

These are different things and the orchestrator needs to tell them apart.

| Situation | Runner exit code | Output |
|-----------|-----------------|--------|
| All tests pass | 0 | Structured results, all passed |
| Some tests fail | 1 | Structured results with failure details |
| Runner itself crashes | Non-zero (usually 2+) | Stack trace, no structured results |
| Environment not ready | 2 | Readiness error message |

If the runner crashes (no structured results in output, just a traceback), report the crash verbatim. Do not attempt to parse partial output or guess what happened.

## Interpreting Results

The runner produces `CheckResult` objects (or their bash equivalent). Collect them and produce the structured report the orchestrator needs.

### What to extract from each result

- **Test name** — which script / check function
- **Layer** — which layer module it belongs to
- **Spec behaviour tag** — the `SPEC_REF` or `BEHAVIOUR` tag from the test. This is how failures trace back to spec requirements.
- **Pass/fail**
- **Failure detail** — expected vs actual, the raw response, the status code. Everything the implementation agent needs to understand what went wrong without re-running the test.

### Failure context quality

The difference between a useful failure report and a useless one is **context**. The implementation agent that receives a failure report has never seen these tests before. It needs to understand:

1. What the test was trying to do (the behaviour tag and test name)
2. What it expected (the assertion)
3. What actually happened (the response)
4. What state the test set up (so the implementer can reproduce)

Bad failure context:
```
FAIL: test_profile_update
```

Good failure context:
```
FAIL: test_profile_update → SPEC_REF: user-profile-003
  Behaviour: "When a user PUTs to /v1/users/{id}/profile, the profile is replaced"
  Setup: Created user via POST /v1/users (id: usr_abc123)
  Action: PUT /v1/users/usr_abc123/profile with {"name": "updated"}
  Expected: 200 with updated profile in body
  Actual: 405 Method Not Allowed
  Response body: {"error": "PUT not supported on this endpoint"}
```

If the runner's `CheckResult` doesn't include this level of detail, include the raw log output for that test. Something is better than nothing, but push for the good version when writing Phase 1 tests.

## Output Format

### All tests pass

```
ACCEPTANCE TESTS: PASSED
Run: [timestamp]

Summary: [count]/[count] checks passed across [n] layers

Results by layer:
  health:         [count]/[count] ✓
  conversation:   [count]/[count] ✓
  skills:         [count]/[count] ✓
  knowledgebase:  [count]/[count] ✓

Full report: results.md
```

### Some tests fail

```
ACCEPTANCE TESTS: FAILED
Run: [timestamp]

Summary: [passed]/[total] checks passed, [failed] failed
Stopped at layer: [layer name]

Failures:

1. [layer]: [test_name] → SPEC_REF: [ref]
   Behaviour: [quoted from test comment]
   Expected: [what the test asserted]
   Actual: [what happened]
   Detail:
   [trimmed output — last 30 lines max]

2. [layer]: [test_name] → SPEC_REF: [ref]
   ...

Passing checks: [count]
  [list of passing test names, one per line, no detail needed]

Failing spec behaviours (deduplicated):
  - [SPEC_REF]: [one-line behaviour description]
  - [SPEC_REF]: [one-line behaviour description]

Full report: results.md
Full log: run_output.log
```

The "Failing spec behaviours" section at the bottom is the most important part for routing. The orchestrator uses this to scope the handback to Phase 4 — "fix these specific behaviours" — rather than sending a blob of test output and hoping the implementation agent figures out what matters.

Because the runner stops at the first failing layer, the failures listed here are all from the same layer. The orchestrator should expect that fixing these may reveal new failures in deeper layers on the next run.

### Environment failure

```
ACCEPTANCE TESTS: NOT RUN
Run: [timestamp]
Reason: Environment not ready

Detail:
[readiness check output]

Action required: [specific what — "start the API server", "set AUTH_TOKEN", etc.]
```

## Re-Run Protocol

After the implementation agent fixes code in response to failures, this phase runs again. Every re-run is a full suite run from the top — the runner always starts at layer 1.

### What to compare across runs

Track results across runs to distinguish fixes from regressions:

- Previously failing tests that now pass → fix worked
- Previously passing tests that now fail → regression introduced
- Previously failing tests that still fail → fix didn't work
- A deeper layer now reached (because the previously-failing layer passes) → progress, but new failures may appear

```
RE-RUN RESULTS (attempt [n]/3):

Fixed (were failing, now pass):
  - [test_name] → SPEC_REF: [ref]

Regressions (were passing, now fail):
  - [test_name] → SPEC_REF: [ref]
  [failure detail]

Still failing:
  - [test_name] → SPEC_REF: [ref]
  [failure detail]

Newly reached layers:
  - [layer name]: [count] passed, [count] failed
```

Regressions are more alarming than persistent failures. Flag them prominently. A regression means the fix was wrong in a way that broke something else, which often indicates a deeper architectural problem.

A newly-reached layer with failures is normal — it just means the previous layer is now clean and we're seeing the next tier of problems. This is expected behaviour given the runner's stop-at-first-failure design.

### Retry budget awareness

This agent doesn't manage retries — the orchestrator does. But the output should make the orchestrator's job easy:

- Include the attempt number in every report
- If this is attempt 3/3 and there are still failures, the output should explicitly say so. The orchestrator will escalate to Phase 2 (re-planning) or halt for human review.

## What This Phase Does NOT Do

- **Diagnose failures.** It reports what happened, not why. "Expected 200, got 500" — not "the 500 is probably because the database migration didn't run."
- **Modify tests.** The immutability rule from Phase 1 applies here too. If a test seems wrong, that's an escalation, not a fix.
- **Modify code.** This agent has no access to source code and no mandate to change anything.
- **Skip tests.** Every test runs (up to the first failing layer). The runner's layer short-circuiting is by design — this agent doesn't override it.
- **Retry on its own.** If a test fails, it fails. The orchestrator decides whether to retry. This agent runs once per invocation and reports.
- **Judge the spec.** A test that seems unreasonable is not this agent's problem. Run it, report the result.

## Edge Cases

### Flaky tests

If a test passes on one run and fails on the next with no code changes, it's flaky. This agent can't detect flakiness from a single run, but if the orchestrator runs the suite multiple times (e.g., during retry cycles) and the same test oscillates, the re-run comparison will surface it:

```
INCONSISTENCY: test_concurrent_updates passed in run 1, failed in run 2, passed in run 3
This may indicate a flaky test or a race condition. Escalate for human review.
```

### Tests that were supposed to fail

Before implementation (Phase 1 gate), all acceptance tests should fail. If this agent is invoked for that pre-implementation check, report any tests that *pass* — those are suspicious. Either the test is trivially true (asserts nothing meaningful) or the feature already exists and the spec is redundant.

```
PRE-IMPLEMENTATION CHECK:

Expected: all tests fail (features not implemented yet)

Unexpected passes:
  - [test_name] → SPEC_REF: [ref]
    This test passed before implementation. Verify it actually tests the new behaviour.

Expected failures: [count]/[total]
```

Note: because the runner stops at the first failing layer, the pre-implementation check will only report results from the first layer. If the first layer's tests all fail as expected, that's sufficient — you can't reach deeper layers anyway.

### Timeout

If the runner hangs (e.g., a test waits forever for an async operation), enforce a global timeout:

```bash
timeout 300 python run.py --base-url "$BASE_URL" 2>&1 | tee run_output.log
if [ $? -eq 124 ]; then
    echo "ACCEPTANCE TESTS: TIMED OUT after 300s"
    echo "Last output before timeout:"
    tail -20 run_output.log
fi
```

Report the timeout and the last output. The implementation agent needs to know which test was running when things stalled.
