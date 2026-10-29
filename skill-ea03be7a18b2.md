<!-- codemesthiri/MarketChamp:.claude/skills/cleancode/SKILL.md | https://github.com/codemesthiri/MarketChamp/blob/HEAD/.claude/skills/cleancode/SKILL.md -->

---
name: cleancode
description: >
  Trigger /cleancode to enforce production-grade code standards via mandatory
  pre-write, post-write, and delivery gates. Enforces test-first (Red-Green-
  Refactor) discipline for every behavioral change. Blocks shortcuts, silent
  failures, untested behavior, and unverified claims of quality.
---

# cleancode

## Phase 0 — Classify context (state explicitly before coding)

Two axes — context and TDD applicability — both must be stated up front.

**Context:**
- `production` → all gates apply (default if unstated).
- `prototype` / `spike` → relaxed; mark every file/function with `# PROTOTYPE: refactor before production`. Tests not required, but state "tests deferred — spike" explicitly.
- `hotfix` → minimal change allowed; you MUST output a tech-debt note naming the file, the shortcut, and the proper fix. Hotfixes still require a regression test that reproduces the bug (see Phase 1 step 6 — bug-fix variant).

**TDD applicability:**
- `behavioral` → adds, changes, or removes observable behavior (a function output, endpoint, state transition, side effect). The Red-Green-Refactor cycle applies; see Phase 1 step 6 and the TDD discipline section below.
- `non-behavioral` → carve-out. Test-first does NOT apply to: styling/CSS/visual-only changes, pure formatting, doc-only edits, rename refactors with no behavior change, generated-file updates, dependency bumps that change no logic, configuration/env edits, migration scaffolding. Existing tests must still pass after the change.

If new behavior is touched even partially, treat the whole change as `behavioral`.

## Phase 1 — Pre-write gate (answer in the response, not silently)

Before writing code, answer each in one line. If you cannot answer, stop and investigate.

1. **Standard pattern**: which official framework/library pattern does this follow? Cite it (doc URL or class name). If none exists, justify the custom approach.
   - **Research clause (optional, not blocking):** when the standard pattern is non-obvious, or you suspect framework best-practice may have shifted since your training cutoff (e.g. async Django views, React state mgmt, DRF versioning, ORM perf patterns, container/auth library choices), `WebFetch` the current official docs or a reputable recent post and cite the URL inline. Use sparingly — most cleancode work does not need this. Skip when the pattern is well-established or when the feature doc already prescribes the approach.
2. **Root cause**: what is the underlying cause of the problem? Confirm you are not patching a symptom.
3. **Security** (only items relevant to this code): input validation, parameterized queries, output escaping, authn/authz check, no secrets in source.
4. **Scalability** (only if data-touching): N+1 avoided (`select_related`/`prefetch_related`), indexes on filter/join columns, pagination for unbounded sets.
5. **Failure modes**: list every error path and how it is handled. `except: pass` and bare `except:` are forbidden.
6. **TDD plan (behavioral changes ONLY)**: name the smallest piece of new behavior to add first. Describe the test that would fail without it and pass with it (concrete inputs, concrete expected outputs — not "should work"). State which axis it sits on: `unit` (no external code), `integration` (real DB / real internal collaborators), `e2e` (full stack including UI). For bug fixes: describe the test that reproduces the bug — it MUST fail on current `main` before the fix is applied; that same test stays in the suite forever as the regression guard.

## Phase 2 — Hard rules while writing

**Production-code hard limits** (violations require an inline justification comment):
- Functions ≤ 50 lines, files ≤ 300 lines, cyclomatic branches ≤ 10 per function.
- No magic numbers/strings — extract to named constants.
- No hardcoded user IDs, emails, env-specific paths, or credentials.
- No `TODO`/`FIXME` without a tracked ticket reference (e.g. `# TODO(#123): ...`).
- No debug artifacts (`print`, `console.log`, `debugger`, commented-out code).
- No catching `Exception`/`BaseException` without re-raise or logged + handled response.
- No business logic in views/controllers/handlers — push to service/model layer.
- Public functions: type hints (Python) or types (TS) + docstring/JSDoc stating purpose, params, returns, raises.

Naming: descriptive verbs/nouns. Banned identifiers unless scope < 5 lines: `data`, `temp`, `tmp`, `info`, `handler`, `manager`, `do_stuff`, `process`, `result` (alone), `obj`, `item` (outside loops).

DRY: if a block ≥ 5 lines repeats, extract. Comments explain WHY, never WHAT.

**Test-design hard rules** (apply to every new or changed test):
- **Behavior, not implementation.** Tests assert on observable outputs (return values, persisted state, emitted events, HTTP responses, rendered DOM text/roles), NOT on private methods, internal state, or the call shape of internal collaborators. A test should still pass after an internal refactor that preserves behavior.
- **AAA structure.** Each test splits into Arrange / Act / Assert sections. One Act per test. Multiple Asserts are fine as long as they verify the same logical concept (e.g., a paginated response: assert status + page count + items = one concept). The "one assert per test" cargo cult is wrong — Beck doesn't require it.
- **Independent.** Tests must pass in any order and in isolation. No shared mutable state, no order-dependent setup, no "this test only works after test_X." Use fixtures/factories for setup; tear down via the framework's lifecycle.
- **Fast.** Target < 100 ms per unit test, < 1 s per integration test. Slow tests break the red-green-refactor loop. If a test needs `sleep`, redesign: use a deterministic signal (event, condition variable, freezegun-style fake clock).
- **Real at internal boundaries, fake only at process boundaries.** Inside your codebase: real classes, real DB, real ORM. Outside your codebase: mock at the seams (HTTP clients, third-party SDKs, cloud APIs, system clock). Mocking internal collaborators couples tests to implementation and defeats refactor safety.
- **Descriptive test names.** Name the behavior, not the function (`test_user_login_with_expired_token_returns_401` not `test_login`). The test name + assertion + diff of failure should fully explain what broke.
- **No snapshot tests** unless the snapshot is reviewed by a human in the same PR. Auto-accepted snapshots test nothing; they freeze the current bug into the suite.

## Phase 3 — Post-write verification (RUN, do not assert)

Before declaring done, EXECUTE and report results:

1. Run the project's test suite (or the relevant subset). Paste pass/fail counts.
2. Run the project's linter / type-checker / formatter if configured (`ruff`, `mypy`, `eslint`, `tsc`, `black`, `prettier`). Paste output or "clean".
3. **TDD evidence (behavioral changes):** report that the new test was observed FAILING before the production code was written, and PASSING after. Quote the red phase failure (the actual stack trace or assert diff). If you cannot demonstrate the test failed first — e.g., you added test and code in one go — say so explicitly. Don't fake the evidence.
4. **Sanity check (recommended, optional):** comment out the new production code or invert one branch; confirm the new test then fails. Revert. Catches tests that pass without exercising the path. Skip for trivial behavior; do it for anything with multiple branches.
5. Grep the diff for: `pass\s*$` after `except`, `print(`, `console\.log`, `TODO`, `FIXME`, `XXX`, hardcoded numeric literals > 1.
6. Re-read the diff. Confirm: every Phase-1 answer still holds, every Phase-2 rule satisfied. Test count went UP in proportion to behavioral changes — if production lines changed but test count is flat, justify.

If any step fails or a tool is unavailable, say so explicitly. Do not silently skip.

## Phase 4 — Delivery format (required structure when presenting code)

Your reply must include, in this order:
1. **Context**: production | prototype | hotfix + behavioral | non-behavioral.
2. **Approach**: which standard pattern + one-line WHY.
3. **Trade-offs**: at least one explicit trade-off or "none material because ...".
4. **TDD cycle (behavioral only)**: state the loop you ran:
   - **Red**: which test you wrote first, and the observed failure (quote it).
   - **Green**: the minimal code change that turned the test green.
   - **Refactor**: what you cleaned up while keeping the suite green, OR "no refactor — final code is already minimal and clean."
5. **Verification**: tests run, linter run, results (numbers, not adjectives).
6. **The code**.
7. **Follow-ups**: anything deferred, tech debt, or risks the user should accept.

## TDD discipline — Red-Green-Refactor

The non-negotiable loop for every `behavioral` change. Sources: Kent Beck's *Test Driven Development: By Example* (2002, canonical), Martin Fowler's current TDD overview (https://martinfowler.com/bliki/TestDrivenDevelopment.html, updated Dec 2023).

1. **Red — write a failing test first.**
   - Smallest test that captures the new behavior.
   - Concrete inputs, concrete expected outputs.
   - Run the test. Observe the failure. The failure must be for the *right reason*: a `NameError`/import error/compile error counts as red; a green run is NOT a valid red and means the test is mis-targeted.
2. **Green — minimum code to make the test pass.**
   - Resist adding abstractions, error handling, or edge cases not yet tested. Add the literal minimum.
   - "Fake it till you make it" is fine: returning a hard-coded literal that satisfies the current test is valid — the next failing test forces generalization.
   - Run the FULL test suite (not just the new test). Green = nothing else broke.
3. **Refactor — clean up while staying green.**
   - Fowler: *"the most common way that I hear to screw up TDD is neglecting the third step."* Do not skip this.
   - Remove duplication, improve names, extract helpers, apply Phase 2 hard rules.
   - Re-run the suite after every non-trivial refactor edit. The point of having tests is to refactor fearlessly — exercise the safety net.

**Bug-fix variant:** the Red test is the test that reproduces the bug. Confirm it fails on current `main` BEFORE applying the fix. The fix makes it pass. Refactor as needed. The test stays in the suite as a permanent regression guard.

**When to stop the loop:** when the next failing test you can think of either has no business value or duplicates one already in the suite.

**Why test-first (and not test-after):** writing the test first forces you to think about the interface before the implementation, which is the core design benefit Fowler highlights. Test-after produces tests that document what you happened to write, not what the consumer needs.

## Anti-pattern reference (never produce these)

**Production code:**

```python
# Silent suppression — FORBIDDEN
try: process_payment()
except: pass

# User-ID hotfix — FORBIDDEN
if user.id == 123: return special_logic()

# Magic number — FORBIDDEN
if count > 100: ...
# Required:
MAX_ITEMS_PER_PAGE = 100
if count > MAX_ITEMS_PER_PAGE: ...

# N+1 — FORBIDDEN when ORM offers a join
for u in User.objects.all(): u.profile.bio
# Required:
for u in User.objects.select_related('profile'): u.profile.bio

# Bare except hiding errors — FORBIDDEN
try: user = User.objects.get(id=uid)
except: pass
# Required: catch the specific exception, log, return a typed error.
```

**Test code:**

```python
# Asserts on private state — couples test to implementation
def test_user_creation():
    user = create_user(...)
    assert user._internal_id_counter == 1  # FORBIDDEN

# Order-dependent test — fragile, will break on parallel runs
def test_b():
    user = User.objects.get(username='created_in_test_a')  # FORBIDDEN

# No assertion — passes even if production code is deleted
def test_addition():
    add(2, 2)  # FORBIDDEN — must assert observable behavior

# Mocking internal collaborators — couples test to internal call shape
def test_create_invoice():
    with mock.patch('billing.services.tax_calculator.compute') as m:  # FORBIDDEN if tax_calculator is internal
        m.return_value = 0
        create_invoice(...)
# Allowed: mock the HTTP/SDK boundary (e.g., the third-party tax API client), not the internal Python function.

# time.sleep in tests — flaky, slow
def test_async_job():
    enqueue_job(...)
    time.sleep(5)  # FORBIDDEN — use a deterministic signal, fake clock, or polling-with-timeout against a real condition
    assert job_complete()

# Snapshot test without manual review — freezes the current bug into the suite
def test_render_page():
    assert render(page) == SNAPSHOT_BLOB  # FORBIDDEN unless the snapshot was diff-reviewed in this PR
```

## Done definition

Code is "clean" only when:
- Phase 0 stated (context + TDD axis).
- Phase 1 answered in-line (including the TDD plan if behavioral).
- Phase 2 rules satisfied (production AND test-design rules).
- Red-Green-Refactor executed and reported for every behavioral change.
- Phase 3 tools actually executed with results shown.
- Phase 4 delivery format followed.

Missing any one → not done. Say so and fix it.

---

## Project-specific addendum

The rules above are framework-agnostic and ship unchanged across every project that imports this skill. The addendum below is the ONLY project-aware section — when copying this skill to a new project, replace this section wholesale; the rest stays as-is.

**Current project: MarketChamp** (Python 3.9 / Django 4.2 / DRF / React + Vite).

**Backend (Python / Django / DRF):**
- Test runner: `pytest` (via `pytest-django`). Local invocation that matches CI:
  ```
  source djangovenv/bin/activate && \
    BW_SESSION="${BW_SESSION:-}" SECRET_KEY=dummy KITE_ENCRYPTION_KEY=dummy \
    KITE_API_KEY=dummy KITE_API_SECRET=dummy pytest -q
  ```
  Without `BW_SESSION` + the dummy `*_KEY` env vars, Django settings fail at import (Bitwarden lookups). See `CLAUDE_INSTRUCTIONS.md § Pre-commit CI parity`.
- Test layout: `tests/` at repo root (cross-cutting + project-wide), `<app>/tests/` per-app (logic local to that app). Follow whichever convention is already established in the area you're editing.
- DB tests: real Postgres via `pytest-django`; mark with `@pytest.mark.django_db`. Don't mock the ORM.
- API tests: `django.test.Client` (session/middleware in scope) or DRF `APIClient` (serializer / permission focus).
- Kite mocking: do NOT mock at the function level — mock the boundary class `SafeKiteClient` so tests don't reach `api.kite.trade`. Existing import-isolation guard: `tests/test_kite_import_isolation.py` will fail CI if any production module names `KiteConnect` outside `safe_client.py`.
- Linter: `ruff check .` — CI runs this, must be clean before commit.
- Type checker: not in CI today; don't add untyped public functions either way.

**Frontend (React + Vite):**
- React tests not wired in this repo yet. When adding the first one: Vitest + React Testing Library is the recommended stack (matches Vite, behavior-first via `getByRole`/`getByText`, no JSDOM gymnastics). Establish the testing harness in a dedicated PR, not bundled with the first behavioral test.
- Behavior-first queries only: `getByRole`, `getByLabelText`, `getByText`. Forbidden: `getByTestId` (implementation hook), `container.querySelector` (implementation hook).
- No snapshot tests unless the snapshot is reviewed in the same PR.

**Markers / Skips:**
- `@pytest.mark.skip` requires an inline reason AND a tracked ticket: `@pytest.mark.skip(reason="blocked by #123 — flaky DB cleanup")`. Same rule as `TODO`.
- `@pytest.mark.xfail` same rule.

**Project-specific gotchas:**
- Don't import `KiteConnect` outside `safe_client.py` — the regression-grep test will fail CI.
- Don't add a test that hits real Kite from CI — `SafeKiteClient`'s write-call guard would block writes, but the test would still flake on auth.
- BWS-injected env vars (`BWS_ACCESS_TOKEN`, `BWS_PROJECT_ID`) are not available locally without the dummy stand-ins above; tests that depend on real secrets belong in a manual smoke-test path, not the automated suite.
