<!-- Lixuhang987/medium-powers:skills/use-case-driven-development/SKILL.md | https://github.com/Lixuhang987/medium-powers/blob/HEAD/skills/use-case-driven-development/SKILL.md -->

---
name: use-case-driven-development
description: Use when implementing any feature or bugfix, before writing implementation code
---

# Use-Case-Driven Development: Integration Tests First

## Overview

Start from an approved writing plan.
Confirm the plan already defines the use case flow, core data structures, and interface contracts.
Then write an integration test that proves the flow works end to end.
Finally, implement the minimum amount of code required to make the current use case pass.

**Core principle:** You do not need to write fine-grained unit tests for every internal function, but you must control the core data structures, interface boundaries, and complete flow of the use case.

The goal of testing is not to prove that “a certain function is implemented correctly,” but to prove that:

> The input, state changes, interface calls, data flow, and output of a real use case are coherent.

## When to Use

Use this for:

- New features
- Bug fixes
- Core flow refactoring
- Interface definition changes
- Data structure changes

Fine-grained unit tests may still be kept when:

- A bug is hard to reproduce reliably through integration tests
- A module is core infrastructure reused by many flows

## Iron Rule

Do not start implementation from this skill unless a writing plan exists and is usable.

The writing plan, not this skill, owns detailed planning and contract definition. Before writing implementation code, confirm the plan clearly defines:

- The use case trigger and expected result
- The existing flow to reuse or extend
- The core data structures and interface contracts
- The use-case-level integration test to create first
- The implementation tasks needed to make that test pass

## Development Loop

**PLAN CHECK → INTEGRATION TEST → IMPLEMENT → VERIFY → REFACTOR**

------

## 1. PLAN CHECK — Confirm the Writing Plan Exists

This skill does not create the product spec, use case map, or interface contracts. Those are writing-plans work.

Before continuing, locate the relevant implementation plan. If no plan exists, stop and invoke `medium-powers:writing-plans` first.

The plan must include:

- User or system trigger
- Expected result or observable effect
- Existing flow inventory
- Core data structures and interface contracts
- Use case map showing how data moves through functions or interfaces
- Path or near-code description for the integration test to create
- Any planned negative test skip, if the only available test would assert removed or unsupported behavior

If the plan is missing one of these, stop and repair the plan before writing tests or implementation code.

------

## 2. PLAN FIT — Keep Work Inside the Plan

Use the plan as the boundary for this execution pass.

Confirm:

- The current task maps to exactly one planned use case or one explicit step in that use case
- No extra behavior is being added beyond the spec and plan
- Any needed interface or data structure change is already described in the plan
- The planned test can be made runnable before production code changes

If implementation reveals that the plan is wrong, update the plan first. Do not silently redesign inside implementation.

------

## 3. INTEGRATION TEST — Write Use-Case-Level Tests

The test should cover a complete flow, not an internal function.

A good test focuses on:

- The real entry point
- Real core modules
- As few mocks as possible
- Mocking external uncontrollable systems, not your own business logic
- Verifying final state and key side effects
- Positive supported behavior, not proving removed behavior is gone

```ts
test('creates a session and generates an assistant message after the user submits a prompt', async () => {
  const app = createTestApp({
    agentRunner: fakeAgentRunner({
      response: 'Hello, I can help you.'
    })
  });

  const result = await app.submitPrompt({
    workspaceId: 'workspace-1',
    prompt: 'Hello'
  });

  const session = await app.sessions.getById(result.sessionId);

  expect(session.workspaceId).toBe('workspace-1');
  expect(session.status).toBe('completed');
  expect(session.messages).toEqual([
    { role: 'user', content: 'Hello' },
    { role: 'assistant', content: 'Hello, I can help you.' }
  ]);
});
```

This test verifies the complete use case: entry point, session creation, runner invocation, message writing, and final state.

```ts
test('appendMessage works', async () => {
  const repo = mockRepo();
  await appendMessage(repo, 'session-1', { role: 'user', content: 'hi' });
  expect(repo.appendMessage).toHaveBeenCalled();
});
```

This test only verifies that a mock was called. It does not prove that the flow actually works.

### Negative Test Rule

Do not write tests whose main assertion is that old behavior, a removed feature, or an unsupported path no longer works.

If the plan's test target would become that kind of negative test, you may skip creating it. Record that the negative test was skipped and reference only the relevant plan description. Do not invent a broader negative-test suite.

------

## 4. IMPLEMENT — Write the Minimum Code

Only implement the code required to make the current use case integration test pass.

Do not implement ahead of time:

- Parameters that may be needed in the future
- Complex abstractions
- Extra configuration options
- Branches not covered by the current use case
- Premature framework-style abstractions

It is acceptable to start with a simple, direct, replaceable implementation.

------

## 5. VERIFY — Validate That the Flow Passes

Run the integration test for the corresponding use case.

Confirm that:

- The integration test passes
- Core data structures are not temporarily hacked
- Interface boundaries do not leak implementation details
- Failures can be located at a specific stage in the flow
- The test does not over-mock your own business code

If the test fails, fix the implementation first.
If the test is hard to write, first check whether the interface boundaries or data structures are unclear.

------

## 6. REFACTOR — Refactor After Green

Refactoring is only allowed after the integration test passes.

Allowed:

- Extract helpers
- Improve naming
- Split modules
- Simplify data structures
- Adjust interface boundaries
- Remove duplicate logic

Not allowed:

- Adding new behavior without adding a new use case test
- Breaking interface contracts for implementation convenience
- Turning an integration test into a test that only verifies mock calls

## Test Granularity Principle

The default testing granularity is:

> One integration test group per use case

Instead of:

> One test per function

## Mocking Principle

Only mock system boundaries. Do not mock the core business flow.

## Final Rules

The core data structures are clear.
The interface contracts are clear.
The use case flow is clear.
The integration test proves that the flow works.

Otherwise, it is not done.
