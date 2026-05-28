# Framework Detection

How `/test-detect` builds a testing profile without guessing.

## The Problem

AI assistants default to whatever framework they saw most in training —
usually Jest for JS, unittest for Python. But the project may use Vitest,
pytest, or have strong conventions about structure and mocking. Writing tests
in the wrong style means they get rejected in review or, worse, merged and
become inconsistent noise.

## Detection Order

### Step 1: Config files (authoritative)

| Language | Files to check | What they tell you |
|----------|---------------|--------------------|
| JS/TS | `vitest.config.*`, `jest.config.*`, `package.json` scripts/deps, `.mocharc*`, `playwright.config.*` | Runner, environment, setup files |
| Python | `pyproject.toml [tool.pytest]`, `pytest.ini`, `setup.cfg`, `tox.ini` | pytest vs unittest, markers, fixtures path |
| Go | presence of `*_test.go`, `go.mod` | Always `testing`; check for `testify`, `gomock` |
| Ruby | `.rspec`, `spec/spec_helper.rb`, `test/test_helper.rb` | RSpec vs Minitest |
| Java/Kotlin | `build.gradle`, `pom.xml` | JUnit 4 vs 5, TestNG, MockK, Mockito |
| Rust | `Cargo.toml`, `#[cfg(test)]` blocks | Built-in test, `proptest` |
| C#/.NET | `*.csproj` | xUnit, NUnit, MSTest |

### Step 2: Read 2–3 existing test files

Config tells you the framework. Existing tests tell you the **conventions**.
Pick the most recently modified tests (they reflect current team style, not
legacy patterns). Extract:

- **Test block structure**
  - `describe()` + `it()` (BDD nesting)
  - flat `test()` calls
  - class-based (`class TestFoo(TestCase)`)
  - table-driven (Go: `tests := []struct{...}`)
- **Assertion library + style**
  - `expect(x).toBe(y)` / `expect(x).toEqual(y)`
  - `assert x == y` / `self.assertEqual(x, y)`
  - `x.should.equal(y)` (chai/should)
  - `assert.Equal(t, want, got)` (testify)
- **Mocking / stubbing**
  - `vi.mock()` / `vi.fn()` (Vitest)
  - `jest.mock()` / `jest.fn()`
  - `unittest.mock.patch` / `MagicMock`
  - `monkeypatch` fixture (pytest)
  - dependency injection (no mocking library — fakes passed in)
  - `gomock` / hand-written fakes (Go)
- **Setup / teardown / fixtures**
  - `beforeEach` / `afterEach`
  - pytest `@fixture`
  - factory functions (`makeUser()`, `buildOrder()`)
  - `setUp` / `tearDown` methods
  - database rollback in teardown
- **File naming + location**
  - `foo.test.ts` vs `foo.spec.ts`
  - co-located (`src/foo.ts` + `src/foo.test.ts`)
  - sibling dir (`__tests__/foo.test.ts`)
  - mirror tree (`tests/unit/foo_test.py`)

### Step 3: Write the profile

Produce a compact profile that all test writing follows:

```
Testing Profile — detected 2026-05-28
  Framework:   Vitest 1.6
  Structure:   describe/it, nested
  Assertions:  expect().toBe() / .toEqual() / .toThrow()
  Mocking:     vi.mock() for modules, vi.fn() for callbacks
  Fixtures:    factory functions in test/factories/
  Naming:      *.test.ts, co-located with source
  Setup:       beforeEach for fresh state, no global setup
```

## Ambiguity Resolution

- **Two frameworks present** (e.g. Jest config + Vitest config): check which
  one the test `script` in package.json actually runs. That's the live one.
- **No existing tests at all**: pick the framework that matches the ecosystem
  default for the detected stack and toolchain, and state the choice explicitly
  to the user before writing. For Vite projects → Vitest. For Next.js → Jest or
  Vitest (check which is installed). For Python with pyproject → pytest.
- **Mixed styles in existing tests**: follow the newest files, not the oldest.
  Note the inconsistency to the user.

## What NOT to do

- Do not introduce a new framework into a project that already has one.
- Do not switch assertion libraries mid-suite.
- Do not add a mocking library if the project uses dependency injection.
- Do not co-locate tests if the project uses a separate `tests/` tree (or vice versa).
