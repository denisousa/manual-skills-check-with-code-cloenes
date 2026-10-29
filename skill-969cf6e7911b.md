<!-- kwiggen/claude-code-plugin:skills/tdd/SKILL.md | https://github.com/kwiggen/claude-code-plugin/blob/HEAD/skills/tdd/SKILL.md -->

---
name: tdd
description: |
  Test-driven development with vertical slices. Enforces behavior-focused tests
  through public interfaces, one red-green-refactor cycle at a time. Use when
  writing tests, adding test coverage, or building features test-first.
---

# TDD — Vertical-Slice Test-Driven Development

Write tests that verify behavior through public interfaces, not implementation
details. One red-green-refactor cycle at a time. Never horizontal slicing.

## Core Principle

Tests describe WHAT the system does, not HOW it does it. Code can change
entirely; tests should not break unless behavior changes.

## Before Starting

1. **Read CLAUDE.md** and existing test files to understand the project's test patterns, frameworks, and conventions
2. **Identify the behavior** — What user-visible or caller-visible behavior are we building or changing?
3. **Confirm the interface** — What public API, endpoint, or function signature will callers use?
4. **Check for existing tests** — Understand what's already covered so you don't duplicate

## The Vertical Slice Loop

For each behavior, execute ONE complete cycle before starting the next:

### RED — Write a Failing Test

Write ONE test that describes ONE behavior through the public interface.

**Good test checklist:**
- [ ] Describes behavior, not implementation ("user can checkout with valid cart")
- [ ] Uses only the public interface — no reaching into private state
- [ ] Would survive an internal refactor without changes
- [ ] Reads like a specification a non-engineer could understand
- [ ] Has a clear, behavior-describing name

**Run the test. Confirm it fails. Confirm it fails for the RIGHT reason** (missing behavior, not a syntax error or import issue).

### GREEN — Minimal Code to Pass

Write the minimum code that makes the test pass. No more.

- No speculative features
- No "while I'm here" cleanup
- No generalization beyond what this single test demands
- If you're writing code no test requires, stop

**Run the test. Confirm it passes.**

### REFACTOR — Clean Up Under Green Tests

With all tests passing, improve the code:

- Extract duplication
- Improve names
- Simplify logic
- Deepen modules (concentrate complexity behind smaller interfaces)

**Run all tests after each refactoring step. Never refactor on red.**

### Then Repeat

Pick the next behavior. Write the next failing test. Continue.

## Anti-Patterns to Avoid

### Horizontal Slicing (the biggest one)

**WRONG:** Write all tests first, then all implementations.

```
RED:   test1, test2, test3, test4, test5
GREEN: impl1, impl2, impl3, impl4, impl5
```

**RIGHT:** One complete vertical slice at a time.

```
RED->GREEN->REFACTOR: test1->impl1->cleanup
RED->GREEN->REFACTOR: test2->impl2->cleanup
RED->GREEN->REFACTOR: test3->impl3->cleanup
```

Horizontal slicing fails because you're predicting what the implementation will
need before you've written it. Tests end up testing imagined behavior, not actual
behavior. The implementation then contorts to satisfy the tests instead of the
tests documenting what the implementation does.

### Implementation-Coupled Tests

**The diagnostic:** Your test breaks when you refactor, but behavior hasn't
changed. If you rename an internal function and tests fail, those tests were
testing implementation, not behavior.

**Symptoms:**
- Mocking internal collaborators instead of using real ones
- Spying on private methods to confirm they were called
- Reaching into private state (`obj._items`, `obj.internal`)
- Asserting on intermediate data structures
- Tests that break when you change HOW but not WHAT

**Example — brittle vs. durable:**

```
// BAD: Coupled to implementation
test("checkout", () => {
  const cart = new Cart();
  cart._items.push({ id: 1, price: 10 });
  const spy = jest.spyOn(cart, "_calculateTax");
  cart.checkout();
  expect(spy).toHaveBeenCalled();
});

// GOOD: Tests behavior through public interface
test("user can checkout with valid cart", () => {
  const cart = new Cart();
  cart.add({ id: 1, price: 10 });
  const receipt = cart.checkout();
  expect(receipt.total).toBe(11);
});
```

### Over-Mocking

Mock at boundaries (external APIs, third-party services), not between your own
modules. If you're mocking a class you own, ask whether you can use the real
thing instead. If you can't, ask whether the interface between the two is too
coupled.

**When to mock:**
- External HTTP APIs (use recorded fixtures or a test server)
- Time-dependent behavior (use a clock abstraction)
- Non-deterministic inputs (use seeded randomness)

**When NOT to mock:**
- Your own database (use a test database or in-memory equivalent)
- Your own internal modules (use the real implementation)
- Anything where the mock's behavior can diverge from production

## Test Naming

A test name should read as a behavior specification:

**Good:** `user can checkout with valid cart`, `returns error when email is invalid`, `expires session after 30 minutes of inactivity`

**Bad:** `test_checkout_handler`, `testValidation`, `it should work`

The test file is documentation. Someone reading only the test names should understand what the system does.

## Ending the Cycle

When all identified behaviors have passing tests:

1. Run the full test suite — confirm nothing else broke
2. Review the test names — do they read as a specification?
3. Check coverage of the new code — are there untested paths that represent real behaviors?
4. If the project has a linter or type checker, run those too
