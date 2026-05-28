---
name: test-architect
description: "Writes tests that match a project's existing conventions, enforces test-driven development, finds untested critical paths, and detects test smells. Use this skill when: the user asks to write tests, add test coverage, set up testing, do TDD, find what's untested, review test quality, or fix flaky tests. Also trigger on phrases like 'write a test for', 'add tests', 'improve coverage', 'test this function', 'is this tested', 'why is this test flaky', 'set up vitest/jest/pytest', 'test-driven', 'red green refactor', 'mock this'. Before writing any test, detect the project's framework, assertion style, mocking strategy, and file naming — new tests must look like the team wrote them. Language-agnostic: supports vitest, jest, pytest, go test, rspec, junit, xunit, and others."
version: "1.0.0"
license: MIT
metadata:
  author: mturac
---

# Test Architect

Writes tests that look like the team wrote them, enforces test-driven
development discipline, finds the critical paths nobody tested, and catches
the test smells that make suites flaky and brittle.

Most AI-generated tests fail in one of three ways: they use the wrong
framework, they test implementation instead of behavior, or they over-mock
until the test proves nothing. Test Architect fixes all three by **reading the
project's existing tests first** and matching them exactly.

---

## Core principles

1. **Match before you write.** Before writing a single test, detect the
   project's framework, assertion style, mocking approach, fixture pattern,
   and file naming. New tests must be indistinguishable from existing ones.

2. **Test behavior, not implementation.** A good test survives a refactor. If
   renaming a private method breaks the test, the test is wrong. Assert on
   observable outputs and side effects, not internal calls.

3. **Red before green.** For non-trivial logic, write the failing test first,
   watch it fail for the right reason, then implement. This is the only way to
   know the test actually tests something.

4. **Cover what matters.** 100% coverage of getters is worthless. Find the
   critical paths — error handling, boundary conditions, concurrency, money,
   auth — and cover those first.

5. **No flaky tests.** A test that passes 99% of the time is worse than no
   test. Detect and eliminate timing dependencies, shared state, and ordering
   assumptions.

---

## Commands

### `/test-detect` — Detect the project's testing conventions

Before writing tests, run this to build a testing profile. See
`references/framework-detection.md` for the full detection strategy.

**Quick detection:**

```bash
# JS/TS — framework from config + deps
cat package.json 2>/dev/null | grep -E '"(vitest|jest|mocha|@playwright|ava|tap)"'
ls vitest.config.* jest.config.* 2>/dev/null

# Python
cat pyproject.toml setup.cfg pytest.ini tox.ini 2>/dev/null | grep -iE 'pytest|unittest|nose'

# Go
ls *_test.go 2>/dev/null | head -1

# Ruby
ls spec/spec_helper.rb test/test_helper.rb 2>/dev/null

# Java/Kotlin
grep -rl 'org.junit\|org.testng' --include="*.xml" --include="*.gradle" . 2>/dev/null | head -1
```

Then read 2–3 existing test files to extract:

- **Framework + runner** (vitest, jest, pytest, go test, rspec, junit)
- **Structure** (`describe`/`it` vs `test()` vs class-based vs table-driven)
- **Assertion style** (`expect().toBe()` vs `assert` vs `should` vs `assertEquals`)
- **Mocking strategy** (`vi.mock`, `jest.mock`, `unittest.mock`, `gomock`, dependency injection)
- **Fixture/setup pattern** (`beforeEach`, fixtures, factories, `setUp`/`tearDown`)
- **File naming + location** (`*.test.ts` vs `*.spec.ts`, co-located vs `__tests__/` vs `tests/`)

Output a testing profile. All subsequent test writing follows it.

---

### `/test-write [target]` — Write tests for a function/module/file

1. Run `/test-detect` if no profile exists yet
2. Read the target code and understand its contract: inputs, outputs, side effects, error cases
3. Enumerate the cases that matter:
   - **Happy path** (1–2 representative cases)
   - **Boundaries** (empty, zero, max, off-by-one)
   - **Error paths** (invalid input, missing dependency, thrown exceptions)
   - **Edge cases specific to the domain** (concurrency, null on a rare path, falsy-zero)
4. Write the tests in the project's exact style
5. Run them, confirm they pass for the right reason
6. Report coverage of the cases enumerated in step 3 — not a line-coverage number

Never write a test that just calls the function and asserts it "doesn't throw."
Every test must assert a specific, meaningful outcome.

---

### `/test-tdd [feature description]` — Test-driven development loop

Enforces the red-green-refactor cycle. See `references/tdd-workflow.md`.

```
1. RED   — Write one failing test for the smallest next behavior.
           Run it. Confirm it fails, and fails for the RIGHT reason
           (assertion failure, not import error or typo).
2. GREEN  — Write the minimum code to make it pass. No more.
           Run it. Confirm green.
3. REFACTOR — Clean up code AND test while green. Re-run to stay green.
4. Repeat for the next behavior.
```

Do not write the next test until the current one is green. Do not write
implementation ahead of a failing test. One behavior at a time.

---

### `/test-gaps [path]` — Find untested critical paths

1. Load the testing profile
2. Map source files to their test files (by naming convention)
3. For files with no test, or functions with no assertion covering them, rank by risk:
   - 🔴 **Critical** — auth, money, data deletion, error handling, security boundaries
   - 🟡 **Important** — business logic, state transitions, API contracts
   - 🟢 **Low** — getters, formatters, pure display logic
4. Report the gaps ranked by risk, not by file

```
Test Gap Report: src/

🔴 CRITICAL — untested, high risk:
  src/auth/token-refresh.ts:42  refreshToken() — no test for expired-token path
  src/payments/charge.ts:88     applyRefund() — no test for partial refund

🟡 IMPORTANT — untested:
  src/orders/state-machine.ts   transitionTo() — 3 of 6 transitions untested

🟢 LOW — untested (optional):
  src/utils/format-date.ts      formatRelative() — pure function, low risk

Coverage of critical paths: 11/14 (79%)
```

Coverage of **critical paths** is the metric that matters — not total line coverage.

---

### `/test-smell [path]` — Detect test quality problems

See `references/test-smells.md` for the full catalog. Scans test files for:

- **Over-mocking** — so many mocks the test only verifies the mocks
- **Implementation coupling** — asserting on private methods or call order
- **Flaky patterns** — `setTimeout`/`sleep`, real dates/`Date.now()`, network calls, shared mutable state, order-dependent tests
- **Assertion-free tests** — tests that run code but assert nothing meaningful
- **Snapshot abuse** — giant snapshots nobody reviews
- **Conditional logic in tests** — `if`/`for` in a test body hides what's actually tested

Report each smell with the file, line, why it's a problem, and the fix.

---

## Test quality bar

Every test this skill writes must satisfy:

| Property | Check |
|----------|-------|
| **Behavioral** | Survives an internal refactor that preserves behavior |
| **Specific** | Asserts a concrete value/outcome, not just "no throw" |
| **Isolated** | No dependency on other tests, test order, or shared state |
| **Deterministic** | Same result every run — no real time, randomness, or network |
| **Fast** | Unit tests run in milliseconds; slow tests are marked integration |
| **Readable** | The test name states the behavior; a reader knows what broke from the name alone |

---

## What this skill does NOT do

- **It does not chase 100% line coverage.** Coverage is a tool, not a goal.
  Covering trivial code to hit a number wastes effort and creates brittle tests.
- **It does not write tests for code that should not exist.** If the code under
  test is dead or wrong, say so instead of testing it.
- **It does not mock everything.** Mock at architectural boundaries (network,
  filesystem, clock) — not internal collaborators.
- **It does not replace integration/e2e tests.** It focuses on the unit and
  integration layer. For full browser e2e, defer to Playwright.

---

## Passive mode

When generating new application code (not explicitly asked for tests), and the
project has an established test suite:

1. After writing a non-trivial function, offer or write a matching test in the
   project's style
2. If the user's project convention is "tests required" (detected from CI
   config or a CONTRIBUTING note), write the test automatically
3. Never mark a feature complete if its critical paths are untested
