<!-- AlbertoBasalo/express-boilerplate:.agents/skills/coding-type-script/SKILL.md | https://github.com/AlbertoBasalo/express-boilerplate/blob/HEAD/.agents/skills/coding-type-script/SKILL.md -->

---
name: coding-type-script
description: "Best practices for clean and maintainable code in TypeScript. To be used for writing TypeScript code that is easy to read, maintain, and follows industry standards."
---
# TypeScript Coding Skill

- Configuration: Type script is configured with strict typing and ES module syntax at `tsconfig.json`.
- Lint: all code must be linted with `npm run lint` before committing.

## Clean code principles

### Variables and naming
- Must name variables and functions descriptively.
- Should use named constants instead of magic numbers or strings.
### Functions and complexity
- Should keep functions small and focused on a single task.
- Should avoid deeply nested structures.
- Should use early returns to minimize indentation.
- Should avoid negative conditionals (e.g., `!isOk`); prefer handling the positive case first.
- Avoid blank lines in the middle of functions (use comments instead or create helper functions).
### Classes and modules
- Should avoid primitive obsession by defining types.
- Should favor composition over inheritance.
- Should keep dependencies to a minimum.
- May use adapter pattern to decouple from external systems.
- Should keep shared utilities/types in a dedicated shared module.
### Error handling and comments
- Should handle errors with meaningful messages and contextual handling.
- Should write comments to explain the "why" behind complex logic, not the "what".
### General principles
- Must keep solutions simple and avoid over-engineering.
- Should keep code DRY by reusing logic where applicable.

## TypeScript specific guidelines

- TypeScript file naming conventions: `<domain>.<technology>.ts`
- Must use ES modules (`import`/`export`) instead of CommonJS.
- Should favor named exports over default exports.
- Must use file names in `kebab-case.{pattern}.ts` format. Example: `user-login.service.ts`.
- Must use strict typing and avoid `any`.
- Should place reusable `types` in dedicated files; may colocate tiny local types.
- Should use `as const` for constant values to infer literal types.
- Should place reusable `interfaces` in dedicated files; may colocate tiny local contracts.
- Should avoid `null` and `undefined`; prefer using the `Result` pattern (`Ok` and `Err` types).
- Should leverage utility types (e.g., `Partial`, `Pick`, `Omit`) where they improve clarity.
- Should use async/await; add try/catch when adding contextual error handling or recovery.
