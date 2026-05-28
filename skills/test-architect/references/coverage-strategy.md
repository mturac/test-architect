# Coverage Strategy

How `/test-gaps` decides what to test — and what to ignore.

## Line coverage is a trap

100% line coverage means every line ran during tests. It does NOT mean:
- every line was *asserted* on
- every branch was exercised
- the assertions are meaningful

A suite can hit 100% line coverage and catch zero real bugs (assertion-free
tests, over-mocking). Conversely, a suite at 60% can be excellent if that 60%
covers every critical path.

**The metric that matters: coverage of critical paths.**

## Risk ranking

Rank untested code by what breaks if it's wrong, not by file size.

### 🔴 Critical — test these first

- **Authentication & authorization** — token validation, permission checks,
  session handling
- **Money** — pricing, charges, refunds, currency conversion, tax, rounding
- **Data loss** — deletes, cascades, migrations, overwrites
- **Error handling** — the catch blocks, retries, fallbacks (the paths that
  only run when something's already wrong)
- **Security boundaries** — input validation, sanitization, escaping, rate limits
- **Concurrency** — locks, race conditions, idempotency keys
- **State machines** — every transition, especially illegal ones

### 🟡 Important — test these next

- Core business logic and calculations
- API request/response contracts
- Data transformations and mappers
- Integration points between modules
- Validation rules

### 🟢 Low — optional

- Pure formatters and display helpers
- Getters / setters with no logic
- Config objects
- Generated code
- Thin pass-through wrappers

## Boundary analysis

For each function worth testing, the high-value cases are at the boundaries:

| Input type | Cases to cover |
|-----------|----------------|
| Numbers | 0, negative, max, off-by-one around limits |
| Strings | empty, whitespace-only, very long, unicode, injection chars |
| Collections | empty, single element, exactly-at-limit, over-limit |
| Optional/nullable | present, absent, null vs undefined |
| Dates | epoch, far future, DST boundary, timezone edges |
| Money | zero, smallest unit, rounding edge (0.005), negative |

The happy path needs 1–2 tests. The boundaries and error paths need the rest —
that's where bugs live.

## Mapping source to tests

To find gaps, map each source file to its test file by the project's naming
convention:

```
src/auth/token.ts        → src/auth/token.test.ts      (co-located)
src/auth/token.ts        → __tests__/auth/token.test.ts (sibling dir)
app/services/user.py     → tests/services/test_user.py  (mirror tree)
internal/order/order.go  → internal/order/order_test.go (Go convention)
```

A source file with no corresponding test file is an immediate gap. A test file
that exists but doesn't assert on a given exported function is a partial gap —
detect by checking which symbols the test actually exercises.

## Reporting

Report gaps ranked by risk, with the specific untested path named:

```
Test Gap Report: src/

🔴 CRITICAL — untested, high risk:
  src/auth/token-refresh.ts:42  refreshToken()
    → no test for the expired-token branch (the one that matters)
  src/payments/charge.ts:88     applyRefund()
    → no test for partial refund or refund-exceeds-charge

🟡 IMPORTANT:
  src/orders/state-machine.ts   transitionTo()
    → 3 of 6 transitions untested (PENDING→CANCELLED never tested)

🟢 LOW (optional):
  src/utils/format-date.ts      formatRelative() — pure, low risk

Critical path coverage: 11/14 (79%)
Recommendation: cover the 3 critical gaps before adding any 🟢 tests.
```

## What good coverage looks like

- Every 🔴 critical path has a test for its happy case AND its failure case.
- Boundary conditions are covered for any function doing arithmetic, parsing,
  or limits.
- Error handling paths are tested (deliberately trigger the error).
- You stop adding tests when the remaining uncovered code is genuinely 🟢 —
  not when a coverage percentage hits an arbitrary target.
