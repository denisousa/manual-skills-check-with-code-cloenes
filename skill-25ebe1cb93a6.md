<!-- mmynk/dotfiles:config/ai/skills/elevate/SKILL.md | https://github.com/mmynk/dotfiles/blob/HEAD/config/ai/skills/elevate/SKILL.md -->

---
name: elevate
description: Raise a code change to a high engineering bar — correctness first, then craft and robustness. A wrapper that runs the built-in /code-review (bugs) and /simplify (reuse/simplification/efficiency/altitude) to inherit their machinery, then layers a principle-grounded review (SRP, DRY, DDD domain purity, API/interface design, naming, failure-mode robustness, data-evolution safety) drawing on Clean Code, The Pragmatic Programmer, Designing Data-Intensive Applications, Domain-Driven Design (Evans), and The Staff Engineer's Path. Use on demand ("/elevate", "elevate this", "review to a high bar") before a CR, or when you want more than simplify's quality-only pass.
user-invocable: true
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - Grep
  - Glob
  - Skill
  - Agent
---

# /elevate — raise a change to a high engineering bar

`/simplify` cleans up quality. `/code-review` finds bugs. `/elevate` does both **and**
holds the change to the standard a senior/staff engineer would in review: correct first,
then well-crafted, then robust under real-world conditions.

It is a **wrapper**, not a fork. It *invokes* the built-in `/code-review` and `/simplify`
skills at runtime rather than copying their logic, so whatever those skills become,
`/elevate` inherits automatically. It then adds only the dimensions neither of them
covers — the principle-grounded craft and robustness layer below.

Priority order is fixed: **correctness > robustness > craft.** A beautiful function that
returns the wrong answer fails elevate. Never trade a correctness fix for a style one.

---

## Phase 0 — Scope the diff

Get the change under review (same rule `/simplify` uses):
- `git diff @{upstream}...HEAD`, falling back to `git diff main...HEAD` / `git diff HEAD~1`.
- If there are uncommitted changes or the range is empty, also `git diff HEAD`.
- If a PR number, branch, or path was passed as an argument, review that instead.

If the diff only touches comments/docs/renames/formatting with no runtime surface, say so
and stop — there is nothing to elevate. Match effort to scope: a one-file logic tweak does
not need the full fleet; a cross-cutting change does.

## Phase 1 — Inherit: correctness (primary), then craft

Run the two built-ins **in order**, to completion, before the elevate layer:

1. **Correctness — invoke `/code-review` at high effort** (this is the most important pass;
   `/elevate` exists to not ship bugs). Let it surface and, where it applies fixes, fix
   correctness defects. Treat its confirmed findings as blocking.
2. **Craft baseline — invoke `/simplify`.** Inherit its reuse / simplification / efficiency /
   altitude pass and its applied fixes.

Do not re-implement what these do. If either is unavailable in the session, note it and
substitute the equivalent dimension in Phase 2 yourself, but prefer delegation.

## Phase 2 — Elevate: the principle layer (parallel agents)

Re-diff after Phase 1 (fixes may have moved lines), then launch review agents **in a single
message so they run concurrently.** Each agent gets the diff, one dimension, and returns
findings as `{file, line, summary, principle, cost, fix}` — `principle` names the rule and
its source so the finding carries authority, `cost` is the concrete failure or maintenance
tax. Skip a dimension if the diff plainly can't trigger it, and say you skipped it.

Run only the dimensions that Phase 1 did **not** already cover. The catalog:

### 1. Correctness deepening (only if `/code-review` was unavailable, or the change is subtle)
Concurrency, boundary/off-by-one, null/empty, error-path arithmetic, invariant violations.
Grounds: *DDIA* (correctness under concurrency and partial failure). Usually already covered
by Phase 1 — don't duplicate; add only genuinely new failure scenarios with concrete inputs.

### 2. Robustness & failure modes
What happens on the unhappy path? Unhandled exceptions, silent catch-and-swallow, missing
timeouts/retries/idempotency, resource leaks, degradation behavior, log-and-continue vs
fail-fast chosen wrongly. Flag "happy-path-only" code.
Grounds: *DDIA* (faults are the norm, not the exception; design for partial failure),
*Pragmatic Programmer* ("crash early", assertive programming, don't outrun your headlights).

### 3. Cohesion & responsibility (SRP)
Does each unit do one thing? Flag methods/classes that mix levels of abstraction, do
resolution + I/O + formatting at once, or would need to change for more than one reason.
Grounds: *Clean Code* (functions do one thing, one level of abstraction per function),
*DDD* (a model element should have a single clear responsibility in the domain).

### 4. Duplication & the right abstraction (DRY — applied with judgment)
Flag *knowledge* duplication (the same rule/decision encoded twice), not coincidental
similarity. Equally, flag the opposite failure: a premature or wrong abstraction forcing
unrelated cases together. Prefer "duplicate until the abstraction is obvious."
Grounds: *Pragmatic Programmer* (DRY = every piece of knowledge has one authoritative
representation; also the rule of three), *Clean Code*.

### 5. Domain purity & layering (DDD)
Does domain logic stay in the domain layer, free of I/O and infrastructure types? Flag
persistence/DTO/framework beans leaking into domain signatures, business rules stranded in
controllers/handlers, anemic models where behavior belongs on the entity. Check that names
match the ubiquitous language.
Grounds: *DDD* (Evans) — layered architecture, keep the domain isolated; entities and value
objects own their invariants.

### 6. Interface & API design
Is each signature honest and minimal? Flag: parameters a method can derive itself (thread
`site`, derive `siteId` locally — don't pass both); booleans that should be enums; primitive
obsession where a value object would enforce an invariant; `Optional` used as a *return* (good)
vs as a *parameter* (avoid); leaky return types; easy-to-misuse ordering of like-typed args.
Grounds: *Clean Code* (few arguments, no flag args), *Pragmatic Programmer* (design by
contract; make interfaces easy to use right and hard to use wrong), *Effective Java* spirit.

### 7. Naming & legibility
Do names reveal intent without a comment? Flag misleading, vague, or encoded names, and
comments that narrate *history/bugs* instead of stating the *invariant* the code holds
(see the user's CLAUDE.md comment rule — enforce it, don't restate it). Prefer a worked
example in javadoc for non-obvious logic.
Grounds: *Clean Code* (intention-revealing names; comments explain why/invariant, code
explains how).

### 8. Change-amplification & evolvability
Will this change be cheap to evolve? Flag high coupling / low cohesion that amplifies future
change, schema/wire formats without a forward/backward-compatibility story, breaking a
published contract without versioning, or a special case bolted onto shared infrastructure
where the mechanism should generalize (altitude — coordinate with what `/simplify` found).
Grounds: *DDIA* (evolvability; schema evolution and compatibility), *Staff Engineer's Path*
(design for the change you'll need next; reduce blast radius), *Pragmatic Programmer*
(good-enough, reversible decisions, orthogonality).

## Phase 3 — Apply and report

Dedup findings that point at the same line/mechanism (across Phase 1 and Phase 2). Apply each
surviving fix directly, honoring the priority order — correctness fixes always land; a craft
fix that would risk behavior, sprawl well beyond the diff, or that you judge a false positive
gets **skipped with a one-line reason**, not argued with.

Push back, don't rubber-stamp: if a "principle" would make the code worse here (a forced
abstraction, a stream that hurts readability, an SRP split that fragments a cohesive unit),
say so and cite the tradeoff — matching the user's standing preference to weigh engineering
merit over dogma, and to prefer `final`/immutable locals and locally-derived values.

Close with a tight summary grouped by source and severity:
- **Correctness** (from `/code-review` + dimension 1) — what was wrong, what was fixed.
- **Robustness** (dimension 2) — failure modes hardened.
- **Craft** (from `/simplify` + dimensions 3–8) — quality/design improvements, each tagged
  with the principle that motivated it.
- **Skipped** — findings deliberately not applied, with reasons.

If the change was already clean at this bar, say that plainly rather than inventing work.
