<!-- technanimals/geekmidas-skills:.cursor/skills/gkm-help/SKILL.md | https://github.com/technanimals/geekmidas-skills/blob/HEAD/.cursor/skills/gkm-help/SKILL.md -->

---
name: gkm-help
description: Use when the user runs /gkm-help or asks for a quick-reference of GeekMidas toolbox skills. One-shot display — not a persistent mode.
---

# GeekMidas toolbox — Quick Reference

**Tier 1 — always-on (injected every session):**

- `gkm-standing-rules` — no silent deploy; maintain worktree context; confirm before merge.
- `gkm-build-test-lint-gate` — `pnpm build && pnpm test && pnpm lint` before "done".

**Package skills (description-activated):**

- `gkm-constructs` — HTTP endpoints, cloud functions, cron jobs, event subscribers (e.get/e.post patterns).
- `gkm-client` — typed API client with React Query integration.
- `gkm-cli` — CLI workspace config, defineWorkspace, build/generate/dev commands.
- `gkm-envkit` — Zod-validated environment configuration with EnvironmentParser.
- `gkm-services` — service discovery and dependency injection.
- `gkm-db` — Kysely query builder, withTransaction, typed schema.
- `gkm-cache` — unified cache interface (InMemoryCache, UpstashCache).
- `gkm-storage` — AWS S3 abstraction, presigned URLs.
- `gkm-events` — RabbitMQ, SQS, SNS unified event pub/sub.
- `gkm-auth` — JWT token management, automatic refresh, server-side validation.
- `gkm-errors` — HTTP error factories (4xx/5xx).
- `gkm-rate-limit` — sliding window rate limiter, .rateLimit() middleware.
- `gkm-audit` — type-safe audit logging, KyselyAuditStorage.
- `gkm-telescope` — debugging dashboard for API inspection.
- `gkm-studio` — database browser for Kysely-backed schemas.
- `gkm-logger` — structured console logging with child loggers.
- `gkm-testkit` — KyselyFactory for test data factories and transaction isolation.
- `gkm-cloud` — SST resource utilities and stage-based infra config.
- `gkm-schema` — StandardSchema type inference (InferStandardSchema).
- `gkm-emailkit` — React email templates with SMTP client.
- `gkm-ui` — shadcn/ui-based React component library.

**Universal (always available):**

- `code-quality` — DRY, SRP, early returns, no `any`, Zod at boundaries.
- `naming-imports-exports` — camelCase/PascalCase/kebab-case conventions, named exports, `~` alias.

**Companion slash commands:**

- `/gkm-help` — this card.
- `/gkm-commit` — generate a Gitmoji + Conventional Commit message for staged changes.
- `/gkm-review` — review current branch / staged diff against all skills.

**Docs:** https://geekmidas.github.io/toolbox/