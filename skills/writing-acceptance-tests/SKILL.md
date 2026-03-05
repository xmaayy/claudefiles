---
name: writing-acceptance-tests
description: Generates executable acceptance tests from a spec document before implementation begins. Tests use bash or Python scripts against the public API — never internal method calls or test frameworks. Use when a spec is available and acceptance tests need to be written, reviewed, or validated. Also use when gating implementation on external-surface verification, or when an agent proposes modifying existing acceptance tests (the answer is almost always no).
---

# Writing Acceptance Tests

Acceptance tests are an executable verification contract written **before implementation**. They encode the spec's behaviours as observable, external checks. If these pass, the feature works. If they don't, it doesn't.

## What the Test Author Sees

The test author reads the **spec document** and the **API surface** — route definitions, request/response schemas, auth scheme. It does not read service implementations, database models, business logic, or other agents' output. The goal is to test what the spec says the API should do, not to reverse-engineer what the code already does.

## Core Rules

1. **External surface only.** Tests hit the public API, CLI, or message queue — never instantiate classes, call internal methods, or query the DB directly.
2. **Self-contained state.** Each test creates its own preconditions through the public API (e.g., POST a user before testing that user's behaviour). No seed scripts, no DB backdoors.
3. **No application imports.** Tests must not import from the system under test. No models, no services, no config objects. If you can't write the test against a black-box URL, it's not an acceptance test.
4. **No test frameworks.** No pytest, jest, or test runners. Plain scripts with explicit assertions on status codes and response bodies.
5. **Traceable.** Every test is tagged with the spec behaviour it verifies. When tests run later, failures must map back to specific spec behaviours so the orchestrator knows *what requirement* broke, not just *which script* failed.
6. **Immutable.** Nothing downstream may modify acceptance tests. If implementation can't satisfy them, the spec was wrong — halt for human review.

### Language Choice

Bash or Python are both fine. The boundary that matters is **what you import and what you assert on**, not the language.

Python is often cleaner for JSON-heavy APIs. If using Python, the rule is: the only HTTP library is `requests` (or `httpx`), and the only imports outside stdlib come from the test suite's own helpers — never from the application.

### Test-Owned Helpers

Shared helpers owned by the test suite are fine when they handle **environment concerns** orthogonal to what you're verifying. Legitimate helpers:

- **Auth plumbing** — token acquisition, header construction (so every test doesn't repeat the Auth0 dance)
- **Polling** — waiting for async operations to reach terminal status
- **Report formatting** — building human-readable output from results
- **Thin HTTP wrappers** — adding a default timeout, nothing more

A helper crosses the line if it **normalizes response bodies, translates status codes, swallows errors, or hides what actually came back from the API**. The test must always see the raw response and make its own assertions.

## Test Structure

Tests can be bash scripts or Python modules. Either way, the shape is the same: setup state through the public API, exercise the behaviour, assert on the observable outcome.

**Python pattern:**

```python
"""Check: <quoted behaviour from spec>"""
# SPEC_REF: <section or ID in spec>

import requests
from checks import CheckResult
from checks.helpers import api_request  # test-suite-owned, not application code

LAYER = "feature_name"

def run(base_url: str, token: str) -> list[CheckResult]:
    headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
    results: list[CheckResult] = []

    # --- Setup (via public API) ---
    resp = api_request("POST", f"{base_url}/v1/users",
                       headers=headers, json={"name": "testuser"})
    if resp.status_code != 201:
        results.append(CheckResult(layer=LAYER, name="Create user",
                                   passed=False, detail=f"Expected 201, got {resp.status_code}"))
        return results  # early exit — downstream steps depend on this

    user_id = resp.json()["id"]

    # --- Exercise + Assert ---
    resp = api_request("GET", f"{base_url}/v1/users/{user_id}/profile", headers=headers)
    results.append(CheckResult(layer=LAYER, name="Get profile",
                               passed=(resp.status_code == 200),
                               detail=None if resp.status_code == 200
                                      else f"Expected 200, got {resp.status_code}"))

    # --- Teardown (via public API) ---
    api_request("DELETE", f"{base_url}/v1/users/{user_id}", headers=headers)

    return results
```

**Bash pattern:**

```bash
#!/usr/bin/env bash
# BEHAVIOUR: <quoted behaviour from spec>
# SPEC_REF: <section or ID in spec>
set -euo pipefail

BASE_URL="${BASE_URL:-http://localhost:8000}"
PASS=0; FAIL=0
fail() { echo "FAIL: $1"; FAIL=$((FAIL + 1)); }
pass() { echo "PASS: $1"; PASS=$((PASS + 1)); }

# --- Setup (via public API) ---
USER_RESP=$(curl -sf -X POST "$BASE_URL/users" \
  -H "Content-Type: application/json" -d '{"name": "testuser"}')
USER_ID=$(echo "$USER_RESP" | jq -r '.id')

# --- Exercise + Assert ---
RESULT=$(curl -sf -o /dev/null -w "%{http_code}" "$BASE_URL/users/$USER_ID/profile")
[ "$RESULT" = "200" ] && pass "profile returns 200" || fail "expected 200, got $RESULT"

# --- Teardown ---
curl -sf -X DELETE "$BASE_URL/users/$USER_ID" > /dev/null 2>&1 || true

echo "Results: $PASS passed, $FAIL failed"
[ "$FAIL" -eq 0 ] || exit 1
```

## Writing Process

### Step 1: Extract behaviours from the spec

Read the spec's behaviours / requirements section. List each discrete behaviour as a one-line summary. Each becomes at least one test.

### Step 2: Write tests

For each behaviour:
- Determine the public entry point (API route, CLI command, message published)
- Identify what state the test needs and how to create it through the public API
- Determine the observable outcome (status code, response body field, side effect visible through another public endpoint)
- Write the script

**What to assert on:**
- HTTP status codes
- Response body fields (`jq` in bash, `.json()` in Python)
- Presence/absence of resources via GET after a mutation
- CLI exit codes and stdout content
- Side effects observable through other public endpoints (e.g., after POST /orders, GET /orders includes the new order)

**What NOT to assert on:**
- Database row contents
- Internal log output
- Implementation-specific headers or structure not in the spec
- Timing or performance (unless the spec defines SLAs)

### Step 3: Document assumptions

If the spec is ambiguous, you will make interpretation choices. Track every assumption:

```bash
# ASSUMPTION: Spec says "user can update profile" but doesn't specify
#   whether partial updates (PATCH) are supported. This test assumes
#   only full replacement (PUT) is available.
```

Collect all assumptions into a companion `ASSUMPTIONS.md`. This file is a **structured output for the orchestrator** — if it's non-empty, the orchestrator may halt the chain and surface the assumptions to a human before proceeding with implementation. The format should make each assumption reviewable independently:

```markdown
## Assumptions

1. **PATCH not supported** (affects: 02-profile-update.py)
   Spec says "user can update profile" but doesn't specify partial vs full replacement.
   Tests assume PUT only.

2. **Deletion returns 204** (affects: 03-user-deletion.py)
   Spec says "user can be deleted" but doesn't specify the response code.
   Tests assume 204 No Content.
```

### Step 4: Organize as layers

Group checks into **layer modules** ordered by dependency — each layer assumes the previous one passed. A health check proves the server is up before you test conversation flows, and conversation flows work before you test higher-level features built on top of them.

```
checks/
├── __init__.py          # CheckResult dataclass
├── helpers.py           # auth wrappers, polling, report formatting
├── health.py            # Layer 1: server is up
├── conversation.py      # Layer 2: core request/response loop
├── skills.py            # Layer 3: feature built on conversations
├── knowledgebase.py     # Layer 4: feature built on conversations
├── ASSUMPTIONS.md
auth.py                  # token acquisition (Auth0, etc.)
run.py                   # runner
```

Each layer module exports a `run(base_url, token, ...) -> list[CheckResult]` function. Within a layer, checks early-return on failure when downstream steps depend on earlier state (e.g., can't test message history if conversation creation failed).

The runner executes layers sequentially and **stops at the first layer that has any failure** — no point testing skills if conversations are broken:

```python
LAYERS = [health, conversation, skills, knowledgebase]  # order matters

all_results: list[CheckResult] = []
for layer_module in LAYERS:
    layer_results = layer_module.run(base_url, token, jurisdiction_id)
    all_results.extend(layer_results)
    if any(not r.passed for r in layer_results):
        break  # don't run higher layers if a dependency is broken
```

This gives two levels of short-circuiting: intra-layer (step 2 depends on step 1's output) and inter-layer (don't test features if their foundation is broken). The tradeoff is that a single run only reports the *first* problem.

## When There Is No External Interface

If the feature has no public API yet (e.g., a database migration, an internal library), write acceptance tests against the closest external surface that will exist. If nothing external exists yet:

- Write infrastructure-verification tests focused on **business-critical constraints** (indexes, foreign keys, unique constraints) rather than exhaustive schema inspection.
- Stub the test with the expected future interface and mark it `# DEFERRED: no external entry point yet`.
- Defer full acceptance testing to the task that adds the external interface.

Do not re-skin integration tests (Python method calls, class instantiation) as acceptance tests. They are different things.

## The Immutability Rule

Acceptance tests are a contract. Once written:

- Implementation agents must make code pass the tests, not edit the tests to match code.
- If an agent says "I need to change the acceptance test because..." — that means the spec was wrong or ambiguous. **Stop and escalate to a human.**
- The only valid reason to modify a test is a spec change approved by a human.

This is the single most important rule. Without it, acceptance tests are decoration.

## Gating Implementation

Run acceptance tests at these checkpoints:
- **Before implementation**: all tests should fail (they verify features that don't exist yet). If any pass, either the test is wrong or the feature already exists.
- **At halfway points**: after a major subsystem is complete, run the relevant subset.
- **Before marking done**: the full suite must pass. No exceptions, no "we'll fix it later."
