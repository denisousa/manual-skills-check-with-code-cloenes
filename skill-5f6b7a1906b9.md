<!-- Ruzenie/RUI-agentskills:RUI-agentskills/RUI-claude-en/skills/ui-fullflow-orchestrator/SKILL.md | https://github.com/Ruzenie/RUI-agentskills/blob/HEAD/RUI-agentskills/RUI-claude-en/skills/ui-fullflow-orchestrator/SKILL.md -->

---
name: ui-fullflow-orchestrator
description: Master orchestration skill for end-to-end UI workflow. Connects selection, aesthetics, token generation, implementation, refactor, acceptance, and self-review into an executable pipeline with unified reports.
---

# UI Fullflow Orchestrator

Goal: upgrade 17 existing skills from "collaborative" to "serially executable".

## SSOT

- `../contracts/fullflow-handoff.md`
- `../contracts/quality-gates.md`
- `../contracts/reasoning-visibility.md`
- `../README.md`
- `../ui-codegen-master/SKILL.md`
- `../style-scope-guard/SKILL.md`
- `../svg-canvas-icon-engine/SKILL.md`
- `../references/AI_UI_Skill_Design_Document.md`
- Frontend development specification document (Chinese source file in repo)

## One-Click Pipeline Script

```bash
bash skills/ui-fullflow-orchestrator/scripts/run_fullflow_pipeline.sh \
  --brief "SaaS dashboard with data readability and strong primary CTA" \
  --style-target "Dashboard header and core card area" \
  --scope-file "src/pages/dashboard.css" \
  --icon-mode auto \
  --icon-style outline \
  --framework react \
  --project-type saas-modern \
  --priority performance \
  --priority accessibility \
  --density comfortable \
  --auto-complete \
  --refactor-threshold 200 \
  --render-threshold 30 \
  --duplicate-threshold 3 \
  --props-depth-threshold 3 \
  --acceptance-level strict
```

Default output directory (current workspace): `Ruiagents/<timestamp>/` (override with `--workspace-root` or `--out-dir`)

Artifacts:
- `flow.input.json`
- `requirements.summary.json`
- `requirements.prd.md`
- `requirements.questions.md`
- `style-profile.yaml`
- `style.scope.lock.json`
- `style.scope.checklist.md`
- `style.scope.validation.json`
- `icon.manifest.json` (on demand)
- `icon.spec.md` (on demand)
- `icon.sprite.svg` (on demand)
- `icon.need.analysis.json`
- `selector.recommend.json`
- `selector.evaluate.json`
- `aesthetic.score.json`
- `tokens.json`
- `tokens.css`
- `flow.metrics.json`
- `plugin.hooks.phase4.before.json`
- `plugin.hooks.phase5.after.json`
- `stage.status.json`
- `phase4.refactor.report.json`
- `phase4.refactor.report.md`
- `phase5.acceptance.report.json`
- `phase5.acceptance.report.md`
- `quality.gates.md`
- `gate-validation-report.json`
- `flow.state.json`
- `self-eval.scorecard.json`
- `optimization.plan.md`
- `decision.trace.md` (user-visible decision trace summary)
- `fullflow.report.md`

Status query:

```bash
bash skills/ui-fullflow-orchestrator/scripts/flow_status.sh --format table
```

## Orchestration Flow

Follow the five-phase lifecycle from meta standards:
- Phase 1 requirement analysis and design exploration
- Phase 2 architecture planning and component design
- Phase 3 code generation and implementation
- Phase 4 self-review and refactor
- Phase 5 acceptance and delivery

0. `app-workspace-guide`: read current workspace baseline
1. `requirements-elicitation-engine`: generate requirement draft, question list, style profile
2. `style-scope-guard` (optional): lock style-only change boundary and output scope lock when style-boundary constraints are required
3. `ui-selector-pro`: generate recommendation shortlist
4. `ui-selector-pro`: run quantified evaluation for shortlist
5. `ui-aesthetic-coach`: score brief aesthetics and suggest direction
6. `ui-aesthetic-coach`: generate design tokens (JSON/CSS)
7. `svg-canvas-icon-engine`: generate icon-system artifacts on demand
8. `ui-generation-workflow-runner`: enter implementation
9. `ui-component-extractor` + `frontend-standards-enforcer`: structure and standards optimization
10. `ui-acceptance-auditor` + `ui-self-reviewer`: quality closure
11. `ui-codegen-master`: final delivery convergence

## Rules

- Pipeline outputs must follow `fullflow-handoff.md`.
- Stage status and quality gates must follow `quality-gates.md`.
- Reasoning visibility must follow `reasoning-visibility.md`: output auditable decision summaries only, not private reasoning details.
- Fullflow must-pass skill chain (fixed 9):
  - `requirements-elicitation-engine`
  - `ui-codegen-master`
  - `ui-selector-playbook`
  - `ui-aesthetic-coach`
  - `ui-aesthetic-generator`
  - `ui-generation-workflow-runner`
  - `ui-acceptance-auditor`
  - `ui-self-reviewer`
  - `ui-agent-workspace`
- Chat visibility (mandatory): when running `$ui` fullflow, the assistant must present each phase in chat as "Phase -> skills used -> stage status/key artifacts", not only a final aggregate summary.
- Alias resolution (mandatory): when user inputs `$ui`, it must always be treated as `$ui-fullflow-orchestrator` even if no `ui` entry appears in the available-skill list; do not fallback to "missing same-name skill" logic.
- Explicit-invocation lock (mandatory): when user explicitly inputs `$ui` or `$ui-fullflow-orchestrator`, routing must not downgrade or switch to `$ui-codegen-master` or any other single-skill flow.
- Missing-parameter exception (mandatory): under explicit invocation with incomplete parameters, only ask follow-up questions and wait; downgrading execution is forbidden.
- Parameter-completion interaction (chat mode): when user only provides `$ui "<brief>"` and misses `framework` or `project-type`, ask follow-up questions first and wait for user answers before execution.
- Follow-up fields are fixed to two items: `target frontend framework`, `project-type label`.
- Validation failure (script mode): only fail immediately when directly running `run_fullflow_pipeline.sh` with missing required arguments.
- `style-scope-guard` is not mandatory: require successful `style.scope.lock.json` only when `style_target` or `scope_file` is provided; otherwise continue in optional mode.
- `run_phase4_refactor.sh` now performs four default refactor checks: file line count, render-logic line count, repeated pattern count, and props-forwarding depth (all thresholds are overridable via pipeline args).
- Unified config: defaults are loaded from `.rui-config.yaml` (JSON-compatible YAML); explicit CLI args always take precedence.
- `run_phase5_acceptance.sh` now auto-detects and attempts `lint/typecheck/test/a11y/lighthouse` scripts (run when available, mark as skipped otherwise).
- Quality gate validator: `quality-gate-validator/scripts/validate_gates.py` generates/refreshes `gate-validation-report.json`.
- Style linkage validation: `style-scope-guard/scripts/validate_scope_change.py` writes `style.scope.validation.json`; use `install_precommit_hook.sh` to install pre-commit validation.
- When `icon-mode=auto|on` and icon requirements are detected, `icon.manifest.json` must be generated.
- If `requirements.summary.json` has `completeness_score < 70`, mark requirement-completeness risk in report.
- Must output quantitative scorecard (`self-eval.scorecard.json`) and optimization plan (`optimization.plan.md`) as iteration entry.
- If script output conflicts with document heuristics, keep script output first and explain manual correction rationale in report.

### Chat follow-up template (when framework/project-type is missing)

1. What is the target frontend framework? (for example: `react` / `vue` / `angular` / `svelte`)
2. What is the project-type label? (for example: `saas-modern` / `dashboard` / `marketing-site` / `admin-console`)

## Output Contract

1. Pipeline execution result (success/failure)
2. Artifact directory and report path
3. Recommended library + recommended direction
4. Stage status summary (five phases)
5. Stage-by-stage skill list (format: `Phase X: skill-a, skill-b`)
6. Must-pass skill status (fixed 9, each marked as passed/pending)
7. Next skill chain to execute
