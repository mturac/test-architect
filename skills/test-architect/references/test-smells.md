# Test Smells Catalog

What `/test-smell` detects, why each is a problem, and the fix.

## 1. Over-mocking

**Smell:** The test mocks so many collaborators that it only verifies the mocks
were called — not that anything real works.

```js
// SMELL — this proves nothing about real behavior
const repo = { findById: vi.fn().mockReturnValue({ id: 1 }) };
const mapper = { toDto: vi.fn().mockReturnValue({ id: 1 }) };
const logger = { info: vi.fn() };
const svc = new UserService(repo, mapper, logger);
await svc.getUser(1);
expect(repo.findById).toHaveBeenCalledWith(1);  // tests the mock, not the code
```

**Fix:** Mock only at architectural boundaries (network, DB, clock, filesystem).
Use real instances of internal collaborators. Assert on the returned value.

## 2. Implementation coupling

**Smell:** The test asserts on private methods, internal call order, or
internal state — so it breaks when you refactor without changing behavior.

```python
# SMELL — asserts a private method was called
service.process(order)
service._validate.assert_called_once()  # breaks if _validate is inlined
```

**Fix:** Assert on the observable outcome (return value, emitted event, saved
record, thrown exception) — never on how the result was produced.

## 3. Flaky patterns

**Smell:** The test depends on real time, randomness, network, or shared state.

```js
// SMELL — real timer, will flake under load
await sleep(100);
expect(job.done).toBe(true);

// SMELL — real date, fails at midnight / in other timezones
expect(formatDate(new Date())).toBe("2026-05-28");
```

**Fix:**
- Inject/fake the clock (`vi.useFakeTimers()`, `freezegun`, `clockwork`).
- Seed or inject randomness.
- Mock network at the boundary.
- Reset shared state in `beforeEach`; never rely on test order.

## 4. Assertion-free tests

**Smell:** The test runs code but asserts nothing meaningful — it only checks
"didn't throw."

```python
# SMELL — passes even if the result is completely wrong
def test_calculate():
    calculate_total(cart)  # no assertion
```

**Fix:** Every test asserts a specific expected value or a specific raised
exception. "Doesn't throw" is acceptable only when not-throwing IS the
behavior under test — and even then, assert the successful return.

## 5. Snapshot abuse

**Smell:** Giant snapshots that no human reviews. A 400-line snapshot gets
"updated" on every change without anyone reading the diff.

**Fix:** Snapshot small, stable, intentional output. For large structures,
assert on the specific fields that matter. Never snapshot data that changes
every run (timestamps, IDs) without normalizing it first.

## 6. Conditional logic in tests

**Smell:** `if`/`for`/`try` in the test body hides what's actually being tested
and can make a test pass by skipping its own assertions.

```js
// SMELL — if the branch is never taken, the test passes vacuously
if (result.items.length > 0) {
  expect(result.items[0].id).toBe(1);
}
```

**Fix:** One test asserts one scenario. If you need multiple scenarios, write
multiple tests (or a parameterized/table-driven test where the framework
supports it). No branching logic in a test body.

## 7. Mystery guest

**Smell:** The test depends on external fixture files, a seeded database, or
data set up far away — so a reader can't tell what makes it pass.

**Fix:** Set up the data the test needs inside the test (or an obvious local
factory). The test should be readable top-to-bottom without hunting.

## 8. Test interdependence

**Smell:** Test B only passes if Test A ran first (shared module state, a
record A created, ordering).

**Fix:** Each test is fully isolated — sets up its own state, tears it down.
Tests must pass in any order and in isolation (`it.only` should always work).

## 9. Eager / multi-purpose test

**Smell:** One test checks ten unrelated things. When it fails, you don't know
which behavior broke.

**Fix:** One behavior per test. The test name names the behavior. A failure
message alone should tell you what broke.

## Reporting Format

For each smell found, report:

```
🟡 Over-mocking — src/services/user.test.ts:24
   The test mocks repo, mapper, and logger, then only asserts repo.findById
   was called. It would pass even if getUser returned the wrong data.
   Fix: use a real mapper, assert on the returned DTO value.
```
