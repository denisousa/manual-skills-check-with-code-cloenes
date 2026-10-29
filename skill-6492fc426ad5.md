<!-- SigurdHall/skills:skills/thinking-planning/skill-design/SKILL.md | https://github.com/SigurdHall/skills/blob/HEAD/skills/thinking-planning/skill-design/SKILL.md -->

---
name: skill-design
description: Design, critique, simplify, or improve Codex skills and reusable agent workflows. Use when the user wants to make a skill, lage en skill, create or update SKILL.md files, organize skill folders, convert legacy commands into skills, decide what belongs in references/scripts/assets, improve trigger descriptions, reduce skill bloat, or make skills better for advanced agents.
---

# Skill Design

Use this when making repo-managed Codex skills. Prefer compact, high-leverage
instructions over teaching the model what it already knows.

Core rules:

- Treat `description` as the routing surface. Include concrete trigger words,
  use cases, and boundaries.
- Keep `SKILL.md` short. Put only workflow, decision rules, hazards, and output
  expectations that materially change behavior.
- Use high freedom for thinking, planning, writing, critique, and analysis
  skills. Do not over-script advanced agents.
- Use tighter steps only for fragile formats, tool workflows, APIs, encodings,
  filesystem operations, or domain rules with real failure modes.
- Move long examples, policy detail, schemas, and source notes to `references/`.
- Add `scripts/` only when deterministic or repeated code is better than
  retyping instructions.
- Add `assets/` only for reusable inputs that are copied into outputs.
- Do not add README, changelog, quick-reference, or broad process notes inside
  a skill folder.

Review checklist:

1. Would the trigger fire for the right tasks and avoid adjacent tasks?
2. Does every paragraph justify its context cost?
3. Is the skill adding knowledge or constraints the agent would not infer?
4. Are references discoverable without being loaded by default?
5. Are examples concise and distinct, not a tutorial?
6. Are validation steps concrete when the skill edits files or code?
7. Does the skill avoid duplicating related skills?

For advanced agents, a good skill is usually a sharp lens plus a few guardrails,
not a full manual.

Related skills: [grill-me](../grill-me/SKILL.md) for pressure-testing skill
scope and assumptions, and [prompt-engineering](../../coding/prompt-engineering/SKILL.md)
when the task is mainly prompt wording rather than reusable skill design.
