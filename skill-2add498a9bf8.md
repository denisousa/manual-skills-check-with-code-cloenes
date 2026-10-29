<!-- deadhand777/cs_cv:.claude/skills/writing-plans/SKILL.md | https://github.com/deadhand777/cs_cv/blob/HEAD/.claude/skills/writing-plans/SKILL.md -->

---
name: writing-plans
description: >
  Phase 2 of drklein-craft. Turns an approved spec into a concrete implementation
  plan with bite-sized tasks, TDD steps, exact file paths, and actual code.
  Use when you have an approved spec from brainstorming and need to plan the
  implementation before writing code.
user-invocable: true
---

# Writing Plans

Turn an approved spec into a step-by-step implementation plan that a subagent
can execute without ambiguity.

**Announce at start:** "Using drklein-craft:writing-plans to create the implementation plan."

**Save plans to:** `docs/status/YYYY-MM-DD-<feature>-plan.md`

---

## Checklist

1. Scope check
2. Map file structure
3. Write tasks (bite-sized, TDD)
4. Self-review
5. Dev reviews plan
6. Handoff to `agent-driven-dev`

---

## Step 1: Scope Check

Read the spec. Is this one system or multiple independent systems?

- **Single system** → proceed
- **Multiple independent systems** → split into separate plans, one per system. Each plan must produce working, testable software on its own.

---

## Step 2: Map File Structure

Before defining tasks, map out which files will be created or modified.

- Each file should have one clear responsibility
- Prefer smaller, focused files over large ones
- Files that change together should live together
- In existing codebases, follow established patterns
- Flag any files that touch security-boundaries concerns

This structure drives the task decomposition.

---

## Step 3: Write Tasks

Each task is a self-contained unit of work. Each step within a task is one action (2-5 minutes).

### Task Granularity

A step is ONE of these:
- Write a failing test
- Run the test and verify it fails
- Write the minimal code to make it pass
- Run the test and verify it passes
- Commit

### Plan Document Format

```markdown
# [Feature] Implementation Plan

**Goal:** One sentence
**Architecture:** 2-3 sentences
**Tech Stack:** Key technologies
**Spec:** docs/specs/YYYY-MM-DD-<topic>.md
**Security:** Any security-boundaries considerations

---

### Task 1: [Component Name]

**Files:**
- Create: `exact/path/to/file.ts`
- Modify: `exact/path/to/existing.ts:42-58`
- Test: `tests/exact/path/to/test.ts`

- [ ] **Step 1: Write failing test**

\`\`\`typescript
describe('specific behavior', () => {
  it('should do X when Y', () => {
    const result = functionName(input);
    expect(result).toBe(expected);
  });
});
\`\`\`

- [ ] **Step 2: Run test, verify failure**

Run: `npm test -- tests/path/to/test.ts`
Expected: FAIL — "functionName is not defined"

- [ ] **Step 3: Implement**

\`\`\`typescript
export function functionName(input: InputType): OutputType {
  return expected;
}
\`\`\`

- [ ] **Step 4: Run test, verify pass**

Run: `npm test -- tests/path/to/test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

\`\`\`bash
git add tests/path/to/test.ts src/path/to/file.ts
git commit -m "feat: add specific feature"
\`\`\`
```

### No Placeholders

Every step must contain actual content. These are **plan failures** — never write:

- "TBD", "TODO", "implement later"
- "Add appropriate error handling"
- "Write tests for the above" (without actual test code)
- "Similar to Task N" (repeat the content)
- Steps that describe what to do without showing how
- References to types or functions not defined in any task

---

## Step 4: Self-Review

After writing the complete plan, review it:

1. **Spec coverage:** Skim each requirement in the spec. Can you point to a task that implements it? List any gaps.
2. **Placeholder scan:** Search for any of the patterns from the "No Placeholders" section. Fix them.
3. **Type consistency:** Do function names, types, and method signatures match across tasks? A function called `clearItems()` in Task 3 but `clearAllItems()` in Task 5 is a bug.

Fix issues inline. If a spec requirement has no task, add the task.

---

## Step 5: Dev Reviews Plan

Ask the developer to review:

> "Plan written to `docs/status/YYYY-MM-DD-<feature>-plan.md`. Please review and let me know
> if you want any changes before we start implementation."

Wait for approval. If changes are requested, apply them and re-run the self-review.

---

## Step 6: Handoff

Once the plan is approved, invoke `agent-driven-dev`:

> "Plan approved. Moving to drklein-craft:agent-driven-dev to execute the implementation."

---

## Principles

- **Exact file paths always.** No guessing, no "somewhere in src/".
- **Actual code in every step.** If a step changes code, show the code.
- **Exact commands with expected output.** The implementer should know what success looks like.
- **TDD is the default.** Write the test first, verify it fails, then implement.
- **DRY, YAGNI.** No speculative abstractions, no premature optimization.
- **Security check.** Flag any task that touches restricted areas per `security-boundaries`.
