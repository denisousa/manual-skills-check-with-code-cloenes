<!-- JamesMorales04/diamond-birthday:.opencode/skills/integration-gate-checklist/SKILL.md | https://github.com/JamesMorales04/diamond-birthday/blob/HEAD/.opencode/skills/integration-gate-checklist/SKILL.md -->

---
name: integration-gate-checklist
description: >
  Concise integration gate checklist for local diff review, completed integration,
  pre-finalization check, post-integration review, and reviewing current changes.
  Use when reviewing uncommitted changes, a completed integration, or preparing
  for finalization. Provides a lightweight read-only checklist.
license: MIT
metadata:
  author: project
  version: '1.0'
---

# Integration Gate Checklist

A read-only, reusable checklist for local diff and completed-integration review. This checklist complements @integration-validator but does not replace it.

## When to Use

Trigger when reviewing:

- Current uncommitted changes (local diff)
- A completed integration before finalization
- Pre-finalization readiness
- Post-integration quality check

## Checklist

### 1. Changed-area identification

- [ ] Run `git status` and `git diff --name-only` to identify all changed files.
- [ ] Group changes by domain: frontend, docs, ops, tests.

### 2. Validator output consumption

- [ ] Check if @validator has run and review its output.
- [ ] Note any NOT_AVAILABLE commands and whether targeted checks filled the gap.
- [ ] Confirm no broad build/test/lint validation is being duplicated.

### 3. Cross-cutting risks

- [ ] Missing content or data file integrity issues.
- [ ] Missing tests for changed behavior.
- [ ] Documentation drift for changed contracts or workflows.
- [ ] Duplicated code or weak naming.

### 4. Follow-up routing

- [ ] Every actionable finding names a required executor_agent.
- [ ] No finding is left without a routing target.
- [ ] Orchestrator will not self-implement remediation.

### 5. Finalization safety

- [ ] Determine Finalization safety: `SAFE` | `BLOCKED` | `RISK_KNOWN`.
- [ ] `SAFE`: no blocking findings, all validations passed.
- [ ] `BLOCKED`: Critical/High findings or required validation could not run.
- [ ] `RISK_KNOWN`: no blocking findings but Medium findings are explicitly deferred with rationale.
