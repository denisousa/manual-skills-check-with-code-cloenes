<!-- yossichakim/weather-information-api:.codex/skills/contract-driven-react-frontend/SKILL.md | https://github.com/yossichakim/weather-information-api/blob/HEAD/.codex/skills/contract-driven-react-frontend/SKILL.md -->

---
name: contract-driven-react-frontend
description: Build and maintain a responsive React, Vite, and TypeScript frontend against a verified OpenAPI backend contract. Use for authentication flows, current-weather search, authenticated task CRUD, API client architecture, project-specific UI systems, accessibility, responsive behavior, and frontend validation.
---

# Contract-Driven React Frontend

## Purpose

Build and maintain a production-style portfolio frontend from the repository's real API contract, not invented mock response shapes. Treat `backend/openapi.yaml` as the single API contract source of truth. Verify important behavior against the backend implementation and tests before relying on it.

## Mandatory Inputs

Before implementing frontend work:

1. Inspect `AGENTS.md`, `PLANS.md`, and `backend/openapi.yaml`.
2. Inspect relevant backend routes, controllers, middleware, services, and tests.
3. Inspect existing frontend files when `frontend/` already exists.
4. Run `git branch --show-current` and `git status --short`.
5. Stop before editing when the branch is unexpected or unrelated working-tree changes could overlap the task.

## Guardrails

- Do not modify backend business logic during a frontend task.
- Do not alter API paths, methods, request bodies, response bodies, authentication behavior, status codes, or Prisma data.
- Do not edit real `.env` files.
- Never place API keys, tokens, passwords, or other secrets in frontend source code.
- Do not commit, push, merge, or create a pull request unless explicitly requested.
- Do not use fake production data to hide an incomplete integration.
- Do not add a large generic component library.
- Do not add Redux or another heavy state-management library unless demonstrated application complexity makes it necessary.
- Do not silently weaken TypeScript, ESLint, tests, or accessibility checks.
- Do not overwrite the completed backend execution plan in `PLANS.md`.
- Do not claim responsive or browser validation was performed unless it was actually performed.
- Preserve existing behavior and unrelated user changes.

## Execution Planning

For a broad frontend build, create a dedicated plan before implementation. Preserve the completed root `PLANS.md` and create or maintain:

`docs/plans/frontend-execution-plan.md`

Use `PLANS.md` as the structure and quality standard. Include:

- current state
- desired outcome
- affected files
- frontend architecture
- API integration strategy
- visual direction
- milestones
- validation commands
- accessibility requirements
- risks
- decisions
- progress checklist

Keep the plan current as implementation progresses. Record material decisions and mark milestones only after their validation completes.

## Frontend Technology

Use:

- React
- Vite
- TypeScript
- native `fetch`
- React hooks and context where shared state is justified
- Vitest
- React Testing Library
- ESLint
- custom CSS with CSS variables and reusable layout primitives

Prefer a small dependency set. Do not require a routing library for a single-screen application unless multiple real routes are intentionally designed.

## Recommended Architecture

Prefer a readable feature-oriented structure:

```text
frontend/
  src/
    api/
    components/
    features/
      auth/
      weather/
      tasks/
    hooks/
    types/
    styles/
    App.tsx
    main.tsx
```

Require:

- one centralized API client
- one normalized API error type
- typed request and response models derived from `backend/openapi.yaml`
- feature-specific components
- reusable UI primitives only where they remove real duplication
- separation between API communication, state management, and presentation

Keep transport details, bearer authorization, response parsing, and error normalization out of presentation components.

## Local API Connectivity

The backend currently does not configure browser CORS. For local development:

- Configure the Vite development proxy from `/api` to `http://localhost:3000`.
- Use `VITE_API_BASE_URL`.
- Default the local API base to `/api`.
- Create only a safe `frontend/.env.example` containing documented placeholders.
- Never create or edit a real `.env`.

For a future separate-origin production deployment, report that backend CORS configuration is required as a separate, explicitly scoped backend change. Do not add CORS to the backend during a frontend-only task.

## Authentication Behavior

Support:

- registration
- login
- logout
- bearer-token API requests
- session restoration through `GET /api/auth/me`
- an application-level restoring-session state
- clearing invalid, expired, or revoked tokens after a `401`
- clearing local session state after logout
- useful authentication errors that do not expose sensitive details

The backend exposes bearer JWT authentication rather than cookie authentication. Store only the access token required by the current contract. Keep token access inside a small session or API abstraction; do not spread `localStorage` calls through components.

Ensure startup distinguishes session restoration from an unauthenticated state so protected content does not flash incorrectly. Route authenticated requests through the centralized API client.

## Weather Feature

Implement current-weather search by city with:

- trimmed city input
- loading state
- empty initial state
- validation state
- city-not-found state
- provider or server-error state
- successful weather presentation

Display only information returned by the API:

- city
- state when available
- country
- current temperature
- feels-like temperature
- minimum and maximum temperature
- humidity
- conditions
- wind speed

Do not invent forecasts, precipitation probability, sunrise data, hourly data, or other unsupported fields.

## Task Feature

Implement complete authenticated task functionality:

- list tasks
- create a task
- update a task
- change status
- delete a task
- filter by status
- filter by category
- combine filters
- loading state
- empty state
- validation errors
- authentication errors
- server errors
- delete confirmation

Handle the contract difference explicitly in the API or domain-mapping layer:

- Response statuses are `PENDING` and `CHECKED`.
- Request and filter values are `pending` and `checked`.

Do not spread status conversion logic across UI components. Keep filters synchronized with the requests they produce and prevent stale responses from replacing newer state where concurrent requests are possible.

## Product Design Direction

Follow the design direction in `AGENTS.md`. Create an atmospheric weather operations interface that feels modern, calm, intelligent, distinctive, polished, and portfolio-ready. Avoid a generic admin-dashboard appearance.

Prefer:

- a strong weather search area
- a current-weather hero panel
- a focused task workspace
- a restrained header
- a desktop two-column composition
- a clear stacked mobile composition
- weather-inspired gradients created with CSS
- subtle atmospheric layers
- restrained glass effects
- strong typography
- meaningful hierarchy
- condition-aware accent styling only when derived from returned weather conditions

Do not rely on external background photographs. Do not allow decorative effects to reduce contrast, readability, performance, or usability.

## Accessibility and Responsive Behavior

Require:

- semantic HTML
- real form labels
- visible keyboard focus
- keyboard-operable controls
- meaningful button names
- appropriate `aria-live` regions for asynchronous status messages
- accessible dialogs or confirmations
- sufficient contrast
- usable touch targets
- no color-only status communication
- support for `prefers-reduced-motion`
- desktop and mobile viewport verification

Preserve logical focus after dialogs, mutations, and authentication transitions. Ensure responsive layouts do not hide required actions or force horizontal scrolling.

## Testing

Add focused behavior tests for:

- API error normalization
- login success and failure
- session restoration
- weather loading, success, and error states
- task loading and empty states
- task creation
- task status updates
- task deletion
- status and category filtering
- expired or invalid authentication handling

Use request mocking appropriate for `fetch`, such as MSW or another small, maintainable approach. Verify user-observable behavior rather than implementation details. Keep mock payloads contract-valid and use them for deterministic tests, not as a substitute for real integration.

## Validation

Require frontend package scripts named:

- `lint`
- `typecheck`
- `test`
- `build`

Before completing a frontend task, run from `frontend/`:

```bash
npm run lint
npm run typecheck
npm test
npm run build
```

Also run the existing backend test suite from `backend/` to prove frontend work did not disturb the backend:

```bash
npm test
```

When browser tooling is available, verify at desktop and mobile widths. Record the actual viewport sizes and scenarios checked. Always inspect:

```bash
git diff
git status --short
```

Report unavailable tooling and failed commands accurately. Do not describe an unrun check as passing.

## Definition of Done

A frontend task is complete only when:

- it uses the real backend contract
- authentication works through the documented endpoints
- weather search works through the documented endpoint
- task CRUD and filters work through the documented endpoints
- TypeScript contains no unexplained errors
- loading, empty, success, validation, authentication, and server-error states exist
- the interface is responsive and accessible
- lint, typecheck, tests, and production build pass
- backend tests still pass
- no secret or real environment file is staged
- the execution plan is current
- the final report lists changed files, commands, passed checks, failed checks, and remaining risks
- no commit or push was performed
