<!-- JohnPlummer/jp-plugins:plugins/dev-workflow/skills/review/SKILL.md | https://github.com/JohnPlummer/jp-plugins/blob/HEAD/plugins/dev-workflow/skills/review/SKILL.md -->

---
name: review
description: |
  Single-pass, standards-aware, diff-oriented code review. Reviews whatever is in the
  diff regardless of author (plugin-built or hand-written) against the plugin philosophy
  plus the resolved standards cascade (external folder/repo -> repo, repo wins). Use for
  the CI review action and as a light local pass. NOT the heavy multi-agent
  separation-of-judgment review (that is the build workflow's Review phase, local-only).
---

# review (the diff-oriented review engine)

One review engine, three triggers - same philosophy and standards everywhere:

- **`/implement` phase 5**: reviews the code the build just produced (heavy, local).
- **`/review` standalone (local)**: your manual changes before pushing (light or heavy).
- **CI action**: any PR diff, on every PR (this skill, single-pass, comment-only).

This skill is the **single-pass** form. It runs in CI (the Workflow tool is NOT available
there) and locally as a quick standards pass. The heavy, multi-agent,
separation-of-judgment review lives in `workflows/build-isolated-tdd.js` (Review phase) and
JP's `code-review` skill - both local-only. Determinism here comes from the fixed criteria +
loaded philosophy/standards, not from orchestration.

## Scope (what to review)

Resolve the target from the argument:

- **`owner/repo/pull/N`** (CI) - review the PR diff. Get it with
  `gh pr diff N --repo owner/repo` (or the GitHub tools the action provides).
- **No argument / a ref range** (local) - review the branch's commits ahead of upstream plus
  uncommitted changes (`git diff @{upstream}...` + working tree), or the passed `base...head`.

Review only the diff and the code it touches. Do not review the whole repo.

## Resolve standards (the cascade, most-specific wins)

Three layers, merged by topic; the repo overrides the external source on any conflict:

1. **Philosophy (always on, never overridable).** Read
   `${CLAUDE_PLUGIN_ROOT}/docs/philosophy.md`. These are values, not style - they outrank
   standards but cover different ground.
2. **External standards (org/team/user).** If `$DEV_WORKFLOW_STANDARDS_PATH` is set, that directory is the checked-out standards source. Read its index - `common-llms.md` if present, otherwise `README.md`, whichever the corpus uses - and load only the files matching the languages/domains in the diff (Go, TS, .NET, testing, database, etc.). If the index carries a "Loading Strategy by Task Type" table, select pages through it rather than by eye. Index paths may be written `standards/<topic>/<file>` - resolve them relative to the standards root (strip a leading `standards/`). If the var is unset, or the directory has neither index file, skip this layer and say so in the output (local-folder-only users get a repo-only review; CI must set it for parity).
3. **Repo standards (win on conflict).** Read the repo's `CLAUDE.md` (and any nested
   `CLAUDE.md` under changed paths - they apply only to files beneath them) and, if present,
   a repo `REVIEW.md`. `REVIEW.md` is review-only instruction and takes precedence over the
   default criteria below (severity recalibration, nit caps, skip rules, repo-specific
   checks). When a repo standard and an external standard address the same topic, the repo
   one replaces it; unrelated standards stack.

## Review dimensions

Lead with correctness; everything else is secondary. For each:

1. **Correctness**: logic errors, broken edge cases, races, unscoped queries, data/PII
   leaks, regressions. The priority. A behaviour claim needs a `file:line` citation in the
   diff, not an inference from naming.
2. **Testable by default**: code behaviour changed with no corresponding test change. Flag
   it (philosophy #2: no code change without a test change).
3. **Simplicity (YAGNI/DRY)**: over-build beyond what the change needs, duplication, dead
   code, a hand-rolled thing the codebase already has. Bias toward the smaller diff.
4. **Small / reversible**: an oversized or mixed-concern diff; a hard-to-reverse change
   (public API shape, data format, migration) shipped without a flag, expand-then-contract,
   or a named rollback (philosophy #4).
5. **Standards conformance**: violations of the resolved standards. Repo wins on conflict.

Do **not** try to assess separation of judgment from a diff - that is a build-time property,
not visible in the artefact. Review what is in front of you.

## Severity

Match the familiar markers so findings read the same as the managed reviewer:

- 🔴 **Important**: a bug that should be fixed before merge: breaks behaviour, leaks data,
  or blocks a rollback. Standards violations that break behaviour land here.
- 🟡 **Nit**: minor; style, naming, a small simplification, a non-behavioural standards
  violation. Worth fixing, not blocking.
- 🟣 **Pre-existing**: a real bug in code the PR touches but did not introduce. Note it,
  don't gate on it.

`REVIEW.md` overrides this calibration when present.

## Output

**Before posting, refute each finding.** If you cannot point to the exact `file:line` and
state how the code misbehaves, drop it. False positives cost the author a round trip.

- **CI**: post findings as **inline review comments** on the diff lines (the action
  provides `gh` + a write `github_token`). Open the review body with a one-line tally
  (e.g. `2 important, 3 nits`), or lead with "No blocking issues" when there are none.
  **Single pass. Never apply fixes in CI**: comment only. Cap inline nits at 5 by default
  (or the `REVIEW.md` cap); summarise the rest as a count. If a PR was already reviewed,
  suppress repeat nits and post new Important findings only.
- **Local**: print findings grouped by severity with `file:line` and a one-line tally.
  Offer to fix; do not auto-apply. For a heavier local pass, defer to the build workflow's
  Review phase or `code-review`.

## What this skill is NOT

- Not the heavy multi-agent separation-of-judgment review (Workflow tool, local-only).
- Not a fixer in CI (comment-only; CI never pushes changes).
- Not the managed Code Review service (that runs on Anthropic infra; this runs in your
  Actions minutes via `claude-code-action`).
- Not a whole-repo audit - diff-scoped only.
