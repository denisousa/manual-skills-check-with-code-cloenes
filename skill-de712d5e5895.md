<!-- kashman001/ai-workspace-template:skills/onboard-repo/SKILL.md | https://github.com/kashman001/ai-workspace-template/blob/HEAD/skills/onboard-repo/SKILL.md -->

---
name: onboard-repo
description: Use when bringing a repository into the workspace — record its identity and auth in the registry, build/wire a graphify code index (with a manual-scan fallback), and derive committed code-structure / design / api docs under docs/repo-context/<repo>/ so agents and developers can pull repo knowledge on demand. Trigger when adding a repo, or when the user says "onboard this repo".
---

# onboard-repo

You are **onboarding a repository** into this workspace. Capture its identity, build a
queryable index, and produce the committed reference docs other agents will pull on demand.

Vendor-neutral: any agent runtime can follow these steps. Claude Code also exposes this as
the `/onboard-repo` slash command (`.claude/commands/onboard-repo.md`, a thin wrapper around
this skill). Required input — the **repo name** (the registry key + `docs/repo-context/<name>/`
folder); optional second input — the **repo path** (defaults to the workspace root for a
single-repo workspace, else `repos/<name>/`).

## When to use
- Adding a new repo to the workspace (hosted or local).
- Refreshing an already-onboarded repo after material code change (`--refresh`).
- When the user says "onboard this repo".

## Prerequisites
- `scripts/onboard-repo.sh` (mechanical scaffolding) and the `docs/repo-context/_templates/`.
- graphify is **optional** — its absence triggers the manual-scan fallback, not a failure.
- Auth: `docs/runbooks/authentication.md` (verify-only; never log in or store secrets here).

## Steps

Do these in order, concisely (reference artifacts by path — don't duplicate code):

1. **Record repo info.** Run `scripts/onboard-repo.sh <repo-name> [repo-path]`. Then open
   `docs/repos-registry.md` and fill the new entry's placeholders (Host/Clone URL/Auth
   method/Language-stack/Build-test-run/etc.). For a local repo use `Host: local`, `Clone
   URL: n/a`.
2. **Verify host auth.** Confirm access to the repo's host per `docs/runbooks/authentication.md`
   (e.g. `gh auth status`). Verify only — do not log in or write secrets. Local repos: skip.
3. **Set up indexing.** Read the script's `Status:` line. If `graphify=fallback` and you want
   live queries, build the graph: run `/graphify` (Claude Code) or the graphify CLI in the repo
   path — this is an API-cost step, so do it deliberately. If graphify can't be used (not
   installed / unsupported), proceed with a manual scan; the committed docs below are produced
   either way.
4. **Write `code-structure.md`.** Fill `docs/repo-context/<repo>/code-structure.md` —
   directory map, modules & responsibilities, entrypoints, key flows. Use graphify queries
   (`graphify query/path/explain`, or the graphify MCP tools) when live; else a manual scan.
   Keep it high-level (not line-level) so it ages well.
5. **Derive `design.md`.** Fill architecture overview, components & boundaries, data flow,
   key decisions/trade-offs, known constraints. This holds the "why" graphify can't recover —
   keep graphify-derivable facts in code-structure.md, not here.
6. **Derive `api.md`.** Detect the repo type (service / library / CLI), fill the matching
   section(s), and delete the inapplicable ones.
7. **Wire discoverability.** Confirm the registry entry's "Covered by context docs" points at
   the folder; add the repo to `docs/repo-context/README.md`'s index; note it under
   `CONTEXT.md` → Covered Repos.
8. **Flag for human review.** `design.md` and `api.md` are *your reading* of the code, not
   mechanically extracted — tell the user to review the three docs in the commit/PR diff
   before trusting them.

To refresh later: `scripts/onboard-repo.sh <repo-name> --refresh` (re-stamps provenance +
runs `graphify update .`), then re-derive any docs whose content has drifted.

## Outputs
- A completed `docs/repos-registry.md` entry (identity + auth + stack + commands).
- A graphify graph (live query path) or a documented fallback.
- `docs/repo-context/<repo>/{code-structure,design,api}.md`, provenance-stamped.
- Discoverability wired (registry "covered", README index, CONTEXT.md) + a review nudge.

End by listing the three docs for the user to review, and whether graphify is live or fallback.
