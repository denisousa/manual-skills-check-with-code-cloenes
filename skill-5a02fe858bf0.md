<!-- bowerbird-app/flatpack:.github/skills/rails-refactoring/SKILL.md | https://github.com/bowerbird-app/flatpack/blob/HEAD/.github/skills/rails-refactoring/SKILL.md -->

---
name: rails-refactoring
description: Improve Rails maintainability and convention alignment without changing behavior.
---

# Rails Refactoring Skill

## Use when

- Requests focus on code quality, duplication reduction, naming, structure, or convention alignment.
- The user wants safe refactors with minimal behavior risk.

## Goals

1. Improve readability and reduce duplication.
2. Follow Rails conventions for naming and structure.
3. Keep changes small and safe.
4. Preserve public APIs and behavior.

## Focus areas

- Rails conventions (naming, file placement, autoloading, concerns)
- Model health (validations, callbacks, scopes, associations)
- Service boundaries (move domain logic out of controllers/models when appropriate)
- Query performance (N+1, scopes, eager loading)
- Test maintainability (duplication, helpers, fixtures/factories)
- Configuration hygiene (initializers, engine setup, env-specific config)

## Workflow

1. Identify smallest set of files causing maintenance friction.
2. Propose or implement minimal refactors at root cause.
3. Keep behavior unchanged unless explicitly requested.
4. Add or update targeted tests if behavior-adjacent code changed.
5. Run `bundle exec rubocop -A` to auto-correct style offenses after all code changes.

## Guardrails

- No large rewrites unless explicitly requested.
- No unrelated cleanup.
- Keep explanations plain and concise.
