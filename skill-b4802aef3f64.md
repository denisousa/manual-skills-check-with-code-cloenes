<!-- chuansao258-888/personal-coding-skills:skills/feature-implementer/SKILL.md | https://github.com/chuansao258-888/personal-coding-skills/blob/HEAD/skills/feature-implementer/SKILL.md -->

---
name: feature-implementer
description: Use when implementing a cross-reviewed, approved feature plan or named phase. Enforce task-branch, finding, prerequisite, Architecture Contract, boundary, cleanup, and verification gates; document conformance as pending independent cross-review rather than self-accepting or committing early.
---

# Feature Implementer

Use this skill after the user has approved a feature plan or asks to start a named phase.

The goal is to implement the requested phase faithfully, keep changes scoped, add meaningful tests, and update the implementation document as the source of delivery history.

## Core Rules

- Implement the approved phase, not the whole feature roadmap unless asked.
- Do not start a planned phase when the latest applicable review recommendation
  is `APPLY-REVIEW-FIX` or `REJECT`, or when a blocking Finding remains `OPEN`.
- Do not apply review findings inside feature implementation. Route them through
  `$apply-review-fix`, then require `$cross-review` closure.
- Treat the phase's Goal, Acceptance IDs, Allowed, Not Allowed, Deferred,
  Cleanup obligations, verification, and prerequisites as the execution
  contract.
- Read the plan, implementation document, current code, and related tests before editing.
- Pass the Task Branch Gate before editing and record the branch decision.
- Read repository glossary/context files and relevant ADRs when they exist, and use their language in code, tests, and documentation.
- Read repository environment/configuration docs before changing environment variables, provider settings, feature flags, or runtime defaults. Treat any local config file as secret-bearing and do not quote real values.
- Prefer existing project patterns over new abstractions.
- Do not add endpoints, debug surfaces, permissions, migrations, dependencies, or UI scope that the plan does not require.
- If the latest user instruction changes the approved phase goal, boundary, or
  acceptance, stop and update/re-review the plan before coding; do not use a
  stale `ACCEPT` verdict for the changed target.
- If the plan, implementation document, review result, or latest user instruction contains unresolved `Open Questions`, `Open Decisions`, `Options`, `Alternatives`, `TBD`, or similar blocking choices, stop and ask the user for clarification before editing code, tests, schemas, prompts, or project documentation.
- If the implementation needs a new product or architecture decision, stop and ask.
- Preserve unrelated user changes in the worktree.
- Do not commit before independent phase acceptance. When repository policy
  requires automatic phase checkpoints, the accepting `$cross-review` performs
  the scoped commit after all gates are green. Never push or open a PR unless
  the user explicitly asks.
- Update the implementation document with concrete files, behavior, tests, and verification results.
- Mark completed coding as `IMPLEMENTED_PENDING_REVIEW`; only `$cross-review`
  may mark the phase accepted or its findings closed.

## Standards Dependency

Before creating, moving, naming, validating, or reviewing files, follow
repository-local standards from `AGENTS.md`, `ENGINEERING_STANDARDS.md` or its
equivalent, README, contributor docs, and nearby existing files. Read the
project knowledge/pitfall ledger in `AGENTS.md` before choosing commands or
patterns.

## Task Branch Gate

At the start of every distinct repository task:

1. Run `git status --short --branch` and `git branch --show-current`.
2. Derive the expected task branch from the latest goal and accepted plan.
   Implementation, review-fix, and re-review for the same feature stay on that
   branch.
3. Continue when it matches. When the worktree is clean and the branch is the
   default or unrelated, create or switch automatically using the repository
   convention, falling back to `feat/<task-slug>`.
4. If unrelated dirty changes would move with the switch, do not stash, reset,
   discard, or silently carry them. Prefer a separate worktree from the intended
   base when safe; otherwise stop and ask the user.

Do not create a fresh branch for each phase of one approved task.

## Original Intent Gate

Before coding, compare the selected phase and current plan text against the latest explicit user goal, the original problem statement, confirmed decisions, and relevant review history.

- Do not implement a plan revision that appears to satisfy a finding by weakening, deferring, or removing a user-requested capability.
- Treat scope narrowing, acceptance-gate weakening, warn-only replacement of a requested hard gate, or removal of tuning/configuration/user-visible support as blocking unless the user approved it.
- If the approved plan and the latest user intent appear to diverge, stop and ask which direction to follow before editing code.
- Record any user-approved scope change in the implementation document before relying on it.

## Review Acceptance Gate

Before editing a planned phase, assemble a phase execution contract:

```text
Plan version/path
Target phase
Goal
Stable Acceptance IDs
Allowed
Not Allowed
Deferred
Cleanup obligations
Verification
Prerequisite phases/checkpoints
Latest review recommendation
Open Finding IDs
```

Proceed only when:

- the latest applicable plan review recommendation is `ACCEPT`, unless the user
  explicitly waives review for this phase;
- every blocking plan-review Finding is `CLOSED` by `$cross-review`;
- no unresolved product/architecture/security/data/provider decision remains;
- every prerequisite phase is accepted, and any repository-required checkpoint
  commit exists;
- the target phase has stable acceptance IDs and explicit boundaries, or those
  can be copied unambiguously from an older approved plan.

If the recommendation is `APPLY-REVIEW-FIX`, stop and use
`$apply-review-fix`. If it is `REJECT`, return to `$feature-planner` or the user.
If an older approved plan lacks stable IDs/boundary labels but is otherwise
unambiguous, record a read-only phase execution contract in the implementation
document before coding; do not invent new requirements.

## Implementation Simplicity

Code should be simple, readable, and proportionate to the phase.

- Prefer clear control flow with early returns over deep nesting when it improves readability.
- Avoid speculative abstractions, pass-through wrappers, generic frameworks, or helper layers that only serve one trivial caller.
- Keep methods, components, DTOs, and services focused on one coherent responsibility.
- Reuse existing project patterns and helper APIs before inventing new ones.
- Make complex logic local and named; extract only when it reduces real duplication or hides meaningful complexity behind a stable interface.
- Keep null, error, fallback, and budget paths explicit rather than hidden in clever conditionals.
- Add comments only for non-obvious reasoning, invariants, or trade-offs.
- If a phase requires complexity, record why in the implementation document and protect it with tests.

## Code Style Consistency

Match the style of the surrounding code before introducing a new pattern.

- Follow existing package/module placement, class/component names, method names, DTO/type names, test names, and file organization.
- Keep backend conventions consistent for the repository's framework, service boundaries, controllers/routes, repositories/data access, config objects, DTOs/types, exceptions, logging, and tests.
- Keep frontend conventions consistent for components, hooks, types, imports, state handling, styling, icons, test utilities, and file naming.
- Follow existing formatting and import ordering. Run formatter, lint, typecheck, or build when the project provides a relevant command.
- Do not mix competing styles in the same area, such as different assertion styles, DTO construction styles, error response patterns, or CSS/component patterns.
- Do not reformat unrelated files or churn existing code only for style unless the user explicitly asks for a style cleanup.
- If no clear style exists, choose the simplest style that matches nearby code and document the choice when it affects future work.

## Architecture Conformance Gate

Before editing, extract the phase's `Architecture Contract` and applicable
repository rule IDs. Confirm the business capability owner, behavior/state
owner, authoritative rule location, dependency direction, public surface,
failure/fallback semantics, and cleanup boundary against the current code.

During implementation:

- Keep one authoritative implementation for each changed business rule.
- Do not cross module internals, reverse an approved dependency, introduce a
  cycle, or expose a new public surface unless the plan explicitly allows it.
- Do not add an interface, factory, strategy, provider, registry, pipeline,
  adapter, bridge, context, base class, retry, cache, fallback, or compatibility
  layer without a present, phase-specific justification.
- Keep legal empty results, business rejection, system failure, timeout, and
  degradation semantically distinguishable.
- Remove superseded code, rules, config, tests, and comments named by Cleanup so
  parallel authority cannot survive the phase.

Before handoff, record `Architecture Conformance Evidence` keyed by rule ID:

```text
Ownership and authoritative rule
Existing / added dependency edges
Public API and data-model change
Transaction / concurrency / idempotency impact
Failure and fallback semantics
New abstractions and justification
Complexity delta
Cleanup and stale-reference search
Deviations: None | explicit approved decision
```

If conformance requires a new architecture decision or a prohibited surface,
stop and return to `$feature-planner`; do not improvise a redesign.

## Workflow

1. Pass the Task Branch Gate, identify the target phase, and run the Review
   Acceptance Gate.
2. Extract the phase execution contract and Architecture Contract, then map
   each Acceptance ID and engineering-standard rule ID to planned
   code/test/manual evidence.
3. If the user asks to continue, verify, or close a phase, first check whether it
   is already implemented or still has OPEN findings before writing code.
4. Read relevant backend, frontend, prompt, migration, and test files only after
   review, prerequisite, and Open Questions gates are clear.
5. Make an implementation checklist keyed by Acceptance ID and Cleanup
   obligation when the phase is non-trivial.
6. Implement one tracer-bullet vertical slice at a time inside `Allowed`.
7. Add or update a failing/regression test first when practical.
8. Edit the smallest set of files needed; do not touch `Not Allowed` or
   `Deferred` surfaces.
9. Run targeted tests first.
10. Run broader build or regression checks when contracts, shared code, or
    frontend types changed.
11. Complete named Cleanup and search for stale references.
12. Manually test local UI/app flows when visible behavior changed.
13. Update durable, verified project knowledge/pitfalls in project-level
    `AGENTS.md`; exclude secrets, raw logs, temporary failures, and phase history.
14. Update the implementation document with per-Acceptance-ID evidence,
    Architecture Conformance Evidence, boundary audit, cleanup, commands/results, and
    `IMPLEMENTED_PENDING_REVIEW` status.
15. Report what changed, what passed, and what remains manual; hand off to
    `$cross-review` without claiming acceptance.

## Open Questions Gate

Before implementation starts, inspect the source plan, matching implementation document, current phase text, review result, and latest user message for sections or fields named `Open Questions`, `Open Decisions`, `Questions`, `Options`, `Alternatives`, `Decision Needed`, `TBD`, `To Clarify`, or equivalent.

- If any item is unresolved and could affect product behavior, architecture, API, data model, permissions, dependencies, provider/model choice, UX, tests, rollout, or implementation scope, do not implement. Ask the user the blocking question directly.
- If the current phase contains multiple viable options, recommended-but-unconfirmed choices, or "choose one" alternatives, do not choose on the user's behalf. Ask the user to confirm the option before editing.
- If an Open Question is already answered by a newer explicit user decision, update or cite that decision before continuing.
- If an Open Question is explicitly marked non-blocking/deferred and does not affect the current phase, record that assumption in the implementation document and continue.
- Do not treat a plan as approved for implementation when its Open Questions or Options sections contain unresolved blocking items.

Implementation document Open Questions are binding blockers. Do not start implementation while the implementation document still contains unresolved blocking questions, even if the plan looks approved or the requested phase seems obvious. The only allowed work before clarification is to ask the user, or to update the implementation document with a newer explicit answer from the user.

## Success Criteria Gate

Before coding, confirm the phase has measurable success criteria:

- Stable Acceptance IDs or an unambiguous legacy mapping
- Expected behavior
- Edge cases
- Tests to add or update
- Build/typecheck/lint expectations
- Manual localhost checks for UI/app flows
- Allowed / Not Allowed / Deferred boundaries
- Architecture Contract and applicable repository rule IDs
- Cleanup obligations
- Prerequisite phase/review/checkpoint gates
- Explicit non-goals

If the approved plan does not provide enough success criteria for safe implementation, update the implementation document or ask the user before coding.

## TDD And Minimal Change

For bugs and behavior changes, prefer red-green-refactor:

1. Add a test that fails for the current behavior.
2. Implement the smallest fix.
3. Confirm the test passes.
4. Refactor only after tests are green.

If TDD is not practical, say why in the implementation record and still verify the behavior.

Prefer tests through public interfaces and observable behavior. Do not add brittle tests that only mirror private implementation details unless there is no better seam and the limitation is documented.

## Vertical Slice Discipline

For multi-layer work, complete a narrow path through the necessary layers before expanding scope. A good slice is independently verifiable and can be reviewed on its own.

Avoid building all backend, then all frontend, then all tests as separate horizontal phases unless the approved phase explicitly requires infrastructure groundwork.

## Backend Expectations

When touching backend code:

- Keep DTO, service, facade, controller, repository, migration, and prompt boundaries consistent with existing structure.
- Match existing naming, package layout, constructor/factory style, exception style, logging style, and test style in the touched backend area.
- Wire configuration through properties and policy objects instead of hardcoding planned values.
- When adding or changing config, update the repository's config files and environment documentation consistently without exposing secret values in logs or responses.
- Avoid needless nesting, indirection, or pass-through services. A new seam should earn its keep through reuse, testability, isolation, or a clear domain boundary.
- Keep feature flags and availability checks enforced at every externally reachable path.
- Validate inputs before provider, model, tool, or persistence calls.
- Keep internal trace, prompt, tool argument, and provider payload data sanitized when exposed to users or logs.
- Add unit tests for domain logic and integration-style tests for cross-component contracts when needed.

## Frontend Expectations

When touching frontend code:

- Follow existing components, hooks, types, styling, and test conventions.
- Match existing import style, file naming, component composition, state management, and test utilities in the touched frontend area.
- Keep status, content, done, error, and trace events distinct when handling SSE or streaming state.
- Bind pending state to turn/session identifiers when the backend contract provides them.
- Render empty, loading, error, disabled, and completed states where the user would naturally encounter them.
- Add component tests for user-visible state and contract-sensitive event handling.
- For visible UI changes, run the app locally when practical, exercise the changed flow, and check browser console errors.

## Test Expectations

Tests should prove the behavior that matters:

- Exact payloads and metadata for API, SSE, DTO, tool, and prompt contracts.
- Budget, limit, fallback, timeout, malformed input, disabled, unavailable, and wrong-turn paths.
- Privacy expectations for logs, traces, tool arguments, and provider responses.
- Environment/config binding, defaults, missing values, and invalid values when configuration changed.
- Regression cases for every review finding or previously identified risk.
- Public interface or user-observable behavior for the current slice.

Avoid tests that only prove mocks were called when the contract depends on the data that was passed.

## Layered Verification

Verify from narrow to broad:

1. Targeted tests for changed behavior.
2. Full relevant backend or frontend suite when shared contracts changed.
3. Lint, typecheck, or build when project conventions require it.
4. Manual localhost/browser checks for UI or app-flow changes.
5. Review `git diff` for unintended files, secrets, debug logs, and scope creep.

If a broad check fails because of a pre-existing issue, record the unrelated failure separately and still prove the changed behavior.

## Implementation Document Update

For each completed phase, update the relevant implementation document with:

- Phase status: `IMPLEMENTED_PENDING_REVIEW` until independent acceptance
- Plan/phase version and latest accepted review reference
- Acceptance-ID-to-evidence matrix
- Boundary audit: Allowed touched; Not Allowed/Deferred untouched
- Task branch decision
- Architecture Conformance Evidence keyed by repository rule IDs
- Cleanup obligations completed and stale-reference search
- Files created or modified
- Important implementation details
- Implementation simplicity notes, including any justified complexity or avoided abstraction
- Code style notes when a new or ambiguous local convention was followed
- Tests added or updated
- Verification commands and results
- Known failures or pre-existing blockers
- Manual E2E checks still pending

Do not mark a phase accepted yourself. After implementation and verification,
record `IMPLEMENTED_PENDING_REVIEW`; only a subsequent `$cross-review` may
record acceptance and, when repository policy requires it, create the scoped
phase checkpoint commit.

## Skill Handoff

After implementation:

- Use `$cross-review` with the phase execution contract, Acceptance-ID evidence,
  boundary audit, cleanup record, and verification results.
- Use `$apply-review-fix` if review returns concrete findings.
- Use `$module-introduction` when the completed module needs onboarding, architecture walkthrough, or interview documentation.
- Return to `$feature-planner` if implementation reveals a new product, architecture, dependency, data, security, or acceptance decision that was not planned.

## Final Response

Keep the final response concise:

- Summary of implementation
- Tests/builds run
- Document updated
- Remaining manual checks or next phase
