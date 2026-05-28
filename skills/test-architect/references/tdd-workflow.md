# TDD Workflow

How `/test-tdd` enforces red-green-refactor discipline.

## Why TDD

Writing the test first does three things that writing it after cannot:

1. **Proves the test works.** A test written after the code almost always
   passes on the first run — which means you never saw it fail, which means you
   don't know it can fail. A test that can't fail tests nothing.
2. **Forces a usable interface.** Writing the test first means designing the
   API from the caller's perspective before the implementation locks it in.
3. **Bounds the work.** You write exactly enough code to pass the test. No
   speculative generality, no untested branches.

## The Cycle

### 🔴 RED — Write one failing test

- Pick the **smallest next behavior**, not the whole feature.
- Write a single test asserting that behavior.
- Run it.
- Confirm it fails — and fails for the **right reason**.

The right reason is an **assertion failure** (`expected 5, got undefined`).
The wrong reasons:
- `ReferenceError` / `ImportError` — the function doesn't exist yet (acceptable
  on the very first test of a new module, but the next failure should be an
  assertion)
- `SyntaxError` — typo in the test
- Test framework misconfiguration

If it fails for a wrong reason, fix the test setup before writing any
implementation.

### 🟢 GREEN — Minimum code to pass

- Write the **least** code that makes the test pass.
- Hardcoding a return value is allowed if the test doesn't yet force generality.
  The next test will force it. (This is "triangulation" — it's a feature, not
  cheating.)
- Run the test. Confirm green.
- Do not write code for behaviors you haven't yet tested.

### ♻️ REFACTOR — Clean up while green

- Now improve the code: remove duplication, rename, extract.
- Refactor the **test** too — extract setup, clarify the name.
- Re-run after each change to stay green.
- This step is not optional. Skipping it accumulates the mess TDD is supposed
  to prevent.

### Repeat

Move to the next smallest behavior. One test at a time.

## Rules

- **One failing test at a time.** Don't write five tests then implement. Write
  one, make it pass, then the next.
- **Never implement ahead of a test.** If you're writing code that no failing
  test demands, stop.
- **Stay green between steps.** The suite should be green at every commit point.
- **Commit at green.** Each green state is a safe checkpoint.

## When to relax TDD

TDD is mandatory for non-trivial logic. It can be relaxed for:

- **Pure type definitions / interfaces** — nothing to test.
- **Trivial pass-through code** — a one-line delegate with no logic.
- **Exploratory spikes** — when you're learning an API and will throw the code
  away. (But rewrite test-first once you know the shape.)
- **Throwaway scripts** — explicitly one-off code the user has waived tests for.

When in doubt, write the test first. The cost is low; the safety is high.

## Anti-patterns

- **Writing the test to match buggy code.** If you write code first then write
  a test that asserts whatever the code currently does, you've cemented the bug.
  Test the *intended* behavior, then fix the code.
- **Giant first test.** Testing the entire feature in one test makes red→green
  a huge leap. Break it into the smallest behaviors.
- **Skipping the failing run.** "It'll obviously fail" — run it anyway. You will
  be surprised how often it passes (meaning the test is broken).
