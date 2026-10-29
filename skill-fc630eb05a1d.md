<!-- jimmymalhan/codereview-pilot:.claude/skills/e2e-orchestrator/SKILL.md | https://github.com/jimmymalhan/codereview-pilot/blob/HEAD/.claude/skills/e2e-orchestrator/SKILL.md -->

---
name: e2e-orchestrator
description: Orchestrate 4 phases (Discover, Plan, Implement, Verify) with 5-10 subagents per phase. Use when automating full E2E, "run everything", or coordinating Create→Handle→Run. Invoke with /e2e-orchestrator phase|full|run.
disable-model-invocation: true
argument-hint: [phase|full|run]
---

## Execution Standard (Apply to Every Skill)

- **Phases**: Define sub-agents in **four to five phases** in every skill (e.g. DISCOVER → PLAN → IMPLEMENT → VERIFY → DELIVER).
- **Auto-execute**: NEVER wait for user. Proceed through phases automatically. Never pause. Never ask "shall I proceed?". Agents spawn and run; no approval between phases.

## Phase 1: DISCOVER
### Sub-Agent: `BusinessScout` (model: haiku)
- **Tools**: Glob, Grep, Read
- **Prompt**: Identify all roles needed (Router? FE? BE? QA?). List entry points. Check what's already done. Determine server status.
- **Output**: `{ roles_needed[], entry_points[], skip_roles[], server_status }`
- **Gate**: >= 1 role identified

## Phase 2: PLAN
### Sub-Agent: `PhaseArchitect` (model: sonnet)
- **Prompt**: Map roles to 4 phases. Define subagent prompts per phase. Set parallel vs sequential. Apply cost-guardrails tier.
- **Output**: `{ phase_plan[{phase, agents[], parallel: boolean, model_tier}] }`
- **Gate**: 4 phases defined

## Phase 3: IMPLEMENT
### Sub-Agent: `PhaseRunner` (model: haiku)
- **Tools**: All
- **Prompt**: Execute phases in order. **Within Phase 3 (Review & Critique): spawn CodeReviewer, APIValidator, EvidenceReviewer, QAReviewer, Critic ALL IN PARALLEL.** Do NOT run one after another. Use `agent-task-assignment` skill. Collect outputs. Between phases: check gates.
- **Output**: `{ phase_results[{phase, agent_outputs[], gate_pass}] }`
- **Gate**: current phase complete AND gate passes

## Phase 4: VERIFY
### Sub-Agent: `FlowVerifier` (model: haiku)
- **Tools**: Bash, Read
- **Prompt**: Run `npm test`. Start server (`npm start`). Smoke test localhost:3000. Capture evidence. Score confidence.
- **Output**: `{ tests_pass, localhost_ok, smoke_results[], confidence }`
- **Gate**: tests pass AND localhost works (or server not needed)

## Phase 5: DELIVER
### Sub-Agent: `HandoffPackager` (model: haiku)
- **Prompt**: Create PR if requested. Update CHANGELOG, CONFIDENCE_SCORE. Output real localhost URL and PR link only. Notify user of server status and resume instructions.
- **Output**: `{ pr_url, localhost_url, confidence, server_status, resume_instructions }`
- **Gate**: links are real and verified

## Contingency
IF any phase fails completely → contingency L2 (simplify: skip optional roles). IF rate limited mid-flow → contingency L5 (save state per phase, resume later).

## Server Lifecycle
Phase 4 MUST check server. Phase 5 MUST report server status to user. If laptop may sleep → save phase state.

---

# E2E Orchestrator Skill

**Purpose**: Run the whole business end-to-end—creation, handling, and execution—by role.

**When to use**: Automating full workflows, running everything, or coordinating multi-role flows.

## End-to-End Flow by Phase

### Phase 1: Discovery & Classification
1. **Router** – Classify incident
2. **Explore** – Scout codebase
3. **Frontend Scope** – Identify UI touchpoints
4. **Backend Scope** – Identify API/pipeline touchpoints
5. **Risk ID** – List unknowns, assumptions, risks

### Phase 2: Implementation & Evidence
1. **Retriever** – Gather evidence
2. **Skeptic** – Produce alternatives
3. **Frontend Implementer** – Build UI (use `frontend-engineer`)
4. **Backend Implementer** – Build API (use `backend-engineer`)
5. **Verifier** – Validate claims, score confidence

### Phase 3: Review & Critique – Ten-Pass Verification (Required)

Run `ten-pass-verification`: 10 distinct checks. **Spawn ALL 5 agents in parallel** — CodeReviewer, APIValidator, EvidenceReviewer, QAReviewer, Critic. Do NOT sequence. See `agent-task-assignment` skill.

1. **CodeReviewer** – DRY, style, guardrails, no console.log
2. **APIValidator** – API contract, request/response, error format
3. **EvidenceReviewer** – file:line valid, no invented APIs/fields
4. **QAReviewer** (qa-engineer) – npm test pass, coverage, critical flows
5. **Critic** – Quality gate (confidence >= 0.70, all 6 output contract fields)

**Parallel spawn**: All 5 run simultaneously. Each posts `gh pr comment`. Aggregated pass → proceed. Any block → fix → re-run failed only.

### Phase 2.5 / 3: Live Watchdog (When run-the-business)

When `run-the-business` mode: Add live-watchdog phase after Execute.
- Poll CI (`gh run list`), health (`curl localhost:3000/health`)
- On error → `error-detector` → `fix-pr-creator` → `self-fix`
- Integrate `fix-pr-creator` for automatic fix PR flow

---

## Phase 4: Run & Verify
1. **npm install** – Dependencies
2. **npm test** – All tests passing (local + CI)
3. **npm start** – Server at http://localhost:3000
4. **Manual smoke** – Form submit, results display, error recovery
5. **Update docs** – CONFIDENCE_SCORE, CHANGELOG
6. **PR flow** – Use `pr-push-merge` skill: push, CI green, localhost + PR links
7. **Merge** – Only after user approval; use `conflict-resolution` if main moved
8. **Skills self-update** – Use `skills-self-update` after any fix; commit skill changes

## Always Learning

After every fix, review, or stakeholder feedback:
- Run `skills-self-update`
- Add lesson to relevant SKILL.md (backend-reliability, ui-quality, feedback-log, etc.)
- Commit skill change with fix
- Do not respawn; update in same turn to save credits

## Cost Optimization

- Grep first, targeted reads
- Skip completed work
- 1 lead + 4 subagents default; add specialists only when justified
- Haiku for Explore, QA; Sonnet for Plan, Review
- Batch discovery; one Explore per phase

## Workflow Skills (From User Prompts)

| Prompt Type | Skill | Purpose |
|-------------|-------|---------|
| Plan mode, checklist | `plan-and-execute` | Break work, skip-if-done, 4 subagents |
| Branch permissions | `branch-permissions` | Auto-accept on feature, ask on main |
| Project structure | `project-structure` | Shared vs local, .gitignore |
| PR, push, merge | `pr-push-merge` | Commit→push→CI→localhost→PR→merge |
| Push hard | `push-hard` | No permission asks, auto-execute |
| Stakeholder feedback | `stakeholder-feedback` | CEO, EM, frustrated user iteration |
| Conflict resolution | `conflict-resolution` | Stash, pull main, resolve |
| Feedback log | `feedback-log` | Incorporate common feedback, update guardrails |
| UI premium | `ui-premium-checklist` | Product story, frontend-backend alignment |
| Backend full | `backend-full-checklist` | Phase 0/1/2, backend gate before UI |

## Commands to Run Everything

```bash
# Full E2E
npm install && npm test && npm start

# Pipeline (diagnosis)
node src/local-pipeline.js
# or: node src/run.js

# API health
curl http://localhost:3000/health
```

## Role Skills (Create → Handle → Run)

| Role | Create | Handle | Run |
|------|--------|--------|-----|
| **Router** | Add classification patterns | Classify incident | Pipeline intake |
| **Retriever** | Add retrieval/verification | Gather evidence | Verify citations |
| **Skeptic** | Add counter-analysis | Produce alternatives | Detector API |
| **Verifier** | Add verification rules | Validate + score | Verify/detect/score APIs |
| **Critic** | Add quality gates | APPROVE/REJECT | Pipeline output |
| **Frontend** | Add pages/components | Build UI | localhost:3000 |
| **Backend** | Add routes/handlers | Build API | npm start, health |
| **QA** | Add tests | Review gaps | npm test |

## Idea → Production (Master Skill)

For full flow from idea to production with explicit handoffs, use `idea-to-production`:
- Idea (you) → Execute (Claude, auto) → HANDOFF 1: Merge (you approve) → HANDOFF 2: Deploy (you approve)
- Reference: [Claude Code docs](https://code.claude.com/docs/en/skills), [costs](https://code.claude.com/docs/en/costs.md)

## Invocation

`/e2e-orchestrator [phase|full|run]`

- `phase` – Run specific phase (1–4)
- `full` – Run all 4 phases
- `run` – Just execute (npm test, npm start, smoke)

## Related Skills

- All role skills: `router`, `retriever`, `skeptic`, `verifier`, `critic`, `frontend-engineer`, `backend-engineer`, `qa-engineer`
- Workflow: `plan-and-execute`, `pr-push-merge`, `push-hard`, `conflict-resolution`
- Structure: `project-structure`, `branch-permissions`
- Feedback: `stakeholder-feedback`, `feedback-log`
- Learning: `skills-self-update` – update skills when issues fixed
- Master flow: `idea-to-production` – Idea → Execute → HANDOFF 1 (merge) → HANDOFF 2 (deploy)
- `evidence-proof` – Proof before done
- `pr-automation` – PR creation after E2E pass
