<!-- filio623/agent-skills:skills/tdd/SKILL.md | https://github.com/filio623/agent-skills/blob/HEAD/skills/tdd/SKILL.md -->

---
name: tdd
description: Implement functionality using test-driven development. Use when the user asks for TDD, red-green-refactor, test-first implementation, bug reproduction tests, or behavior-preserving refactors with safety checks.
license: MIT
metadata:
  author: Matt Pocock
  source: https://github.com/mattpocock/skills/tree/main/tdd
---

# TDD

Use a test-first loop to implement or fix behavior.

## Workflow

1. Identify the smallest observable behavior to add or fix.
2. Find the appropriate test location and existing test style.
3. Write or update a failing test first.
4. Run the narrowest relevant test command and confirm the expected failure.
5. Implement the smallest change that makes the test pass.
6. Run the test again.
7. Refactor only after the test is green.
8. Repeat for the next behavior.

## Rules

- Test behavior through public boundaries where possible.
- Avoid testing implementation details unless the module has no stable public boundary.
- Prefer one clear failing reason per test.
- For bugs, first reproduce the bug with a failing test.
- For refactors, add characterization tests before changing structure when coverage is weak.

## Output

Keep the user informed of:

- The behavior under test.
- The failing assertion.
- The implementation change.
- The verification command and result.
