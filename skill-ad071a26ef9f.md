<!-- idesofmarch3/checklist:.claude/skills/sweeping-tdd/SKILL.md | https://github.com/idesofmarch3/checklist/blob/HEAD/.claude/skills/sweeping-tdd/SKILL.md -->

---
name: sweeping-tdd
description: This skill should be used when the user says "sweep", "run all", "batch tdd", "full sweep", "sweep slice", or asks to run through all remaining tasks in MASTER_PLAN.md. Loops through every unchecked task executing the TDD cycle for each, with section-transition orchestration and error halting.
version: 1.0.0
---

# Sweeping TDD - Batch TDD Workflow

Loop through ALL remaining tasks in MASTER_PLAN.md without stopping after each one.

## When to Use

Trigger this skill when:
- User says "sweep" (the batch TDD trigger word)
- User says "run all", "batch tdd", "full sweep", "sweep slice"
- User says "sweep from [section name]" to start from a specific section
- User asks to run through all remaining tasks

NOT triggered by "go" — that remains `triggering-tdd` for single-task use.

## How It Differs from "go"

| | `go` (triggering-tdd) | `sweep` (this skill) |
|---|---|---|
| Scope | Single next task | All remaining tasks |
| Section arg | No | Yes (`sweep from X`) |
| Section close | Manual | Automatic |
| Loop | Stops after 1 task | Continues until done or failure |

## Phase 0: Parse Starting Point & Sweep Summary

1. Read `docs/MASTER_PLAN.md` (symlink to current plan).
2. **If user specified a starting point** (e.g., "sweep from steel thread", "sweep from section 3", "sweep from delete"):
   - Scan all `## Steel Thread` and `### Section N: ...` headers
   - Find the first header whose text contains the user's keyword (case-insensitive substring match)
   - Begin scanning for `- [ ]` from that header's line forward
3. **If no starting point specified:**
   - Begin scanning from the top of the file for the first `- [ ]` in Steel Thread or Test Checklist
4. Build an ordered task list: all unchecked `- [ ]` items from the starting point to the end of the file.
5. **Skip Exit Criteria checkboxes** — only process tasks under `## Steel Thread` and `## Test Checklist`.

Print a sweep summary before starting:

```
--- SWEEP PLAN ---
Starting from: [section name]
Remaining tasks: [N] across [M] sections
Sections: [list section names]
------------------
```

If all checkboxes are already marked `[x]`:
- Suggest: "All tasks complete. Run `/advancing-slices` to archive and start next slice."

## Phase 1-5: TDD Cycle Per Task (Loop)

For each unchecked task, execute the same TDD cycle as `triggering-tdd`:

### Identify

The task is already known from the ordered list. Print which task number (e.g., "Task 3/17") and which section is being worked on.

### Red (Write Failing Test)

Follow the test directory structure:
```
tests/
├── api/          # For src/api/ functions
├── hooks/        # For src/hooks/ custom hooks
├── components/   # For src/components/
├── lib/          # For src/lib/ utilities
└── fixtures/     # Reusable test data
```

Write a minimal failing test with Arrange-Act-Assert pattern. Run `npm run test:unit`. Confirm the test fails with a meaningful error (not a syntax error).

### Green (Implement)

Write **minimum code** to make the test pass. No extra features, no premature optimization, no refactoring yet. Run `npm run test:unit`. All tests must pass.

### Refactor (Optional)

Only if there's clear duplication or clarity issues. If refactoring:
1. Run tests before changes
2. Make one structural change
3. Run tests after
4. Commit with `[S]` prefix BEFORE behavioral commit

### Complete

1. Mark checkbox `[x]` in `docs/MASTER_PLAN.md`
2. Run `npm run test:unit` and `npm run lint` — both must pass
3. Commit with `[B]` prefix including issue number: `[B] Description (#NN)`
4. Check for section completion (see below)
5. **Move to next task immediately** — do not wait for user input

### Task Type: Commit Checkboxes

Some checkboxes in MASTER_PLAN start with `[B]` or `[S]` (e.g., `- [ ] [B] Add DraftCycle types...`). These are **commit checkboxes**, not test tasks. When you reach one:
- Do NOT start a new TDD red-green cycle
- Instead: run `npm run test:unit` and `npm run lint`, mark the checkbox `[x]`, and make the behavioral commit
- This is the natural point to check if the section is complete

## Section Transition Logic

After completing a task, check if it was the last unchecked `- [ ]` in its section (between `### Section N` and the next `---` divider or section header).

**If section is complete:**

1. Extract the issue number from the section header: `### Section N: Name (#NN)` → `NN`
2. Close the GitHub issue: `gh issue close NN`
3. Print a section-complete banner:

```
=== SECTION COMPLETE ===
Section: [Section Name] (#NN)
Tasks completed: [count]
Issue #NN closed.
Next: [Next Section Name] (#MM)
========================
```

4. Continue to the next section immediately.

**Steel Thread special case:**

The Steel Thread section has no associated GitHub issue. When all Steel Thread tasks are done, print:

```
=== STEEL THREAD COMPLETE ===
Foundation validated. Moving to Section 1.
=============================
```

## Error Halting

The sweep **must stop** (not continue to the next task) when:

- `npm run test:unit` fails during GREEN phase (implementation doesn't pass)
- `npm run test:unit` fails during COMPLETE phase (regression)
- `npm run lint` fails during COMPLETE phase
- A git commit fails (e.g., hook rejects it)

On halt, print:

```
--- SWEEP HALTED ---
Failed at: [task description]
Section: [section name] (#NN)
Phase: [RED|GREEN|REFACTOR|COMPLETE]
Error: [brief error description]
Progress: [X]/[N] tasks completed
--------------------
Fix the issue, then say "sweep" to resume from where you left off.
```

## Sweep Completion

When all tasks are done (no more `- [ ]` in Steel Thread or Test Checklist):

```
=== SWEEP COMPLETE ===
All [N] tasks completed across [M] sections.
All section issues closed.
Run /advancing-slices to archive this slice and generate the next one.
=======================
```

## Resumability

Because sweep always starts from the first unchecked `- [ ]` task, it is inherently resumable. If a session ends (context limit, error, user interrupt), a new "sweep" picks up exactly where it left off. No bookmark or state file is needed.

## Project-Specific Context

### Testing Framework

- **Vitest** for unit tests
- **React Testing Library** for component tests
- **MSW** for API mocking

### Test Commands

| Command | Use Case |
|---------|----------|
| `npm run test:unit` | Fast TDD feedback |
| `npm run test:unit -- --watch` | Watch mode |
| `npm run test:e2e` | Pre-deployment only |

### Common Test Patterns

For hooks:
```typescript
import { renderHook, waitFor } from '@testing-library/react';
import { useMyHook } from '@/hooks/useMyHook';

describe('useMyHook', () => {
  it('should return expected value', async () => {
    const { result } = renderHook(() => useMyHook());
    await waitFor(() => {
      expect(result.current.data).toBeDefined();
    });
  });
});
```

For components:
```typescript
import { render, screen } from '@testing-library/react';
import { MyComponent } from '@/components/MyComponent';

describe('MyComponent', () => {
  it('should render correctly', () => {
    render(<MyComponent />);
    expect(screen.getByRole('button')).toBeInTheDocument();
  });
});
```

### Commit Discipline

- `[B]` prefix for behavioral changes (features, tests passing)
- `[S]` prefix for structural changes (refactoring)
- Never mix structural and behavioral in the same commit
- Always include GitHub issue number: `[B] Description (#NN)`

## Quick Reference

| Phase | Action | Verification |
|-------|--------|--------------|
| 0 (once) | Parse start point, build task list | Sweep summary printed |
| Red | Write failing test | Test fails meaningfully |
| Green | Minimum implementation | All tests pass |
| Refactor | Clean up (optional) | Tests still pass, commit [S] |
| Complete | Mark checkbox, commit | [B] prefix, tests green |
| Transition | Close section issue | Banner printed |
| Loop | Next task | Until done or halted |
