<!-- rafsanulhasan/HypermediaEngine:.github/skills/spec-driven-development/SKILL.md | https://github.com/rafsanulhasan/HypermediaEngine/blob/HEAD/.github/skills/spec-driven-development/SKILL.md -->

---
name: spec-driven-development
description: Manages the full lifecycle of a feature specification — from collaborative drafting to enforcement during implementation and testing.
invoker: requirement-analyst
model: claude-sonnet-4-6
tools: Read, Write, Edit, Glob, Grep, Bash, TodoWrite
---

Manages the full lifecycle of a feature specification: collaborative drafting with architect and engineer review, finalization to a persistent spec file, and enforcement handoff to software-engineer and sqa-engineer. Only the `requirement-analyst` agent may invoke this skill.

---

## Phase 0 — Guard

1. Verify the caller is `requirement-analyst`. If not, halt immediately and output:
   > "This skill may only be invoked by the requirement-analyst agent."
2. Read `.claude\CLAUDE.md`
3. Invoke `Skill("manage-memory", args: "requirement-analyst")` to load persistent memory
4. Read relevant source files and any existing specs under `docs/specs/` identified from memory or task input

---

## Phase 1 — Spec Drafting (Collaborative)

1. `requirement-analyst` drafts the initial specification from previously elicited requirements.
2. Consult `software-architect` to review the draft spec for:
   - Architectural alignment with the existing system
   - Technical feasibility
   - Clear and correct component boundary assignments
3. Consult `system-engineer` to review the draft spec for:
   - Low-level design concerns
   - SOLID, DRY, and YAGNI compliance
   - Data shape conformance — all request/response shapes must use `{ data, error }`
4. Collect all feedback from both reviewers and revise the spec.
5. Repeat consultation rounds until both `software-architect` and `system-engineer` explicitly sign off.

---

## Phase 2 — Spec Finalization

1. Derive the `<feature-slug>` from the feature name in kebab-case (e.g., `user-registration`).
2. Write the finalized spec to `docs/specs/<feature-slug>.spec.md`.
3. The spec file MUST include all of the following sections:

   ### Overview
   What the feature does and why it is being built.

   ### Acceptance Criteria
   Numbered, testable criteria. Each AC must be independently verifiable.
   Example format:
   ```
   AC-1: Given X, when Y, then Z.
   AC-2: ...
   ```

   ### Component Boundaries
   Which component, service, or module owns each part of the feature. Describe ownership explicitly — do not leave ambiguity.

   ### Data Shapes
   All request and response data shapes. Every shape must use the `{ data, error }` envelope:
   ```json
   {
     "data": { ... },
     "error": null
   }
   ```

   ### Out of Scope
   Explicit list of things that will NOT be addressed by this feature. Required — cannot be omitted.

   ### Open Questions
   Any unresolved questions that remain after the review cycle. May be empty if none remain, but the section must be present.

4. Validate the written file is readable and well-formed before proceeding.

---

## Phase 3 — Enforcement Handoff

Output the following summary block. `software-engineer` and `sqa-engineer` MUST reference this before beginning any work:

```
--- SPEC ENFORCEMENT HANDOFF ---

Spec file: docs/specs/<feature-slug>.spec.md

Acceptance Criteria IDs:
  AC-1, AC-2, ... (list all IDs from the finalized spec)

IMPORTANT: All implementation and tests must trace to an AC in this spec.
Do not implement anything not covered by the spec.
Any deviation from the spec requires re-invoking `spec-driven-development`
to update the spec first — do not diverge silently.

--- END HANDOFF ---
```

---

## Adherence Rules (for software-engineer and sqa-engineer)

These rules are enforced by the handoff block and must be followed by downstream agents:

- **Before writing any code**, `software-engineer` must read the spec at `docs/specs/<feature-slug>.spec.md`.
- Every implemented behavior must map to a numbered AC. If an AC has no corresponding implementation, flag it explicitly.
- **Before writing any test**, `sqa-engineer` must read the spec and trace each test case to an AC ID in the test's comments or description.
- Any deviation from the spec — whether discovered during implementation or testing — requires re-invoking `spec-driven-development` to update the spec first. Silent divergence is not permitted.
