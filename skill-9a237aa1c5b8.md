<!-- vicventum/shortly:.agents/skills/testing-library-course/SKILL.md | https://github.com/vicventum/shortly/blob/HEAD/.agents/skills/testing-library-course/SKILL.md -->

---
name: testing-library-course
description: "Trigger: Testing Library, userEvent, component test, integration test, frontend test. Enforce frontend Testing Library best practices: user-centric queries, accessible selectors, proper I/O mocking, AAA pattern, DAMP over DRY, and a11y automation."
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.1"
---

## When to Use

Use this skill when:
- Writing ANY new test for a frontend component
- Reviewing or refactoring existing frontend tests
- Setting up test infrastructure (custom renderers, factories, CI)
- Making mocking decisions — determining what to mock vs what to use real
- Building accessible components that need to be testable
- Handling time-dependent code, animations, or UI library components

---

## Testing Strategy (What to Test)

Follow a layered approach. The ROI pyramid for this project, ordered by impact:

### 1. Sections — Integration Tests (RTL + axe + factories + custom render)

**Highest ROI.** Each Section/page component represents a use case (login, create item, update profile). Test the orchestration: user interaction → store → API mock → UI feedback.

```js
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { axe } from 'vitest-axe'
import { toHaveNoViolations } from 'vitest-axe/matchers'

expect.extend({ toHaveNoViolations })

it('creates a new item', async () => {
  const user = userEvent.setup()
  const { container } = render(ItemCreateSection)
  await user.type(screen.getByLabelText(/title/i), 'My item')
  await user.click(screen.getByRole('button', { name: /save/i }))
  expect(await screen.findByText(/item created/i)).toBeInTheDocument()
  expect(await axe(container)).toHaveNoViolations()
})
```

One test per scenario (success, validation error, API error, empty state). No shallow rendering — exercise real children.

**Mock boundary:** Mock the **service** layer (`service-*.js`), not the providers. Services are the stable contract of each module — they represent business operations (`getItems`, `createItem`, etc.). Providers are an implementation detail and swapping them (fetch ↔ localStorage) must NOT break your tests.

```js
// ✅ Correcto — mockeás el servicio, que es el contrato estable
vi.mock('@/modules/links/_shared/api/services/service-items', () => ({
  getItems: vi.fn().mockResolvedValue([{ id: 1, title: 'Ejemplo' }]),
  createItem: vi.fn().mockResolvedValue({ id: 2 }),
}))

// ❌ No mockees el provider — es detalle de implementación intercambiable
vi.mock('@/modules/links/_shared/api/providers/provider-items-fetch', ...)
```

### 2. Service Integration Tests with MSW

**What:** Test the **service layer** (I/O boundary) by intercepting HTTP with MSW. This validates that the service correctly handles every possible response from the outside world — valid data, malformed data, HTTP errors, empty responses — without touching a real API.

**Why:** The service is the anti-corruption layer. It's where raw HTTP responses get validated against Zod schemas, transformed, and turned into clean domain data. If this layer is wrong, every Section test (Section 1) that mocks the service will give you false confidence.

**When to use this instead of Section 1:** You almost NEVER write ONLY a service test. Section 1 tests (Section + mocked service) give higher ROI because they cover the full frontend integration. Write service integration tests to complement Section 1 when:
- The service has complex transformation or validation logic
- You want to document how the service handles edge cases (malformed JSON, missing fields, wrong types)
- The provider uses different backends (fetch, axios, localStorage) and you need to verify each one works

**What NOT to test here:**
- Don't test the **provider** itself (URL, method, headers) — that's a unit test with spies
- Don't test the **hook** (TanStack Query cache, refetch, loading states) — that's a hook test
- Don't test the **client** (base URL config, auth interceptors) — that's a client unit test

#### Setup

Install MSW and create a shared test server:

```bash
npm install -D msw
```

```js
// tests/mocks/server.js
import { setupServer } from 'msw/node'
import { handlers } from './handlers'

export const server = setupServer(...handlers)
```

```js
// tests/mocks/handlers.js
import { http, HttpResponse } from 'msw'

export const handlers = [
  http.get('*/api/items', () => HttpResponse.json([])),
  http.post('*/api/items', () => HttpResponse.json({ id: '1' })),
]
```

```js
// vitest-setup.js (or jest-setup.js)
import { server } from '../tests/mocks/server'

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

#### What to Test (Service + MSW)

For each service, test these scenarios:

| Scenario | MSW behaviour | What the test verifies |
|----------|---------------|----------------------|
| **Happy path** | Returns valid data matching the Zod schema | Service returns parsed, validated data |
| **Schema mismatch** | Returns data with wrong types (e.g. `id` as number instead of string) | Service throws validation error — the anti-corruption layer works |
| **Missing fields** | Returns data missing a required field | Service rejects with schema validation error |
| **Extra fields** | Returns data with extra fields | Service silently passes (Zod strips unknowns by default). If schema uses `.strict()`, service rejects |
| **HTTP 404** | MSW responds with `{ status: 404 }` | Service/provider propagates the error |
| **HTTP 500** | MSW responds with `{ status: 500 }` with error body | Service throws or returns an error object |
| **Empty response** | MSW returns `[]` for lists, `null` for single items | Service handles gracefully — no crash |
| **Network error** | MSW responds with a network error | Service/provider throws a connection error |

These are the **minimum scenarios** every service should be tested against. Add more case-specific scenarios as needed — for example, a service that handles pagination should also test `?page=1` and `?page=invalid`, and a service that sends a request body should test payload serialisation.

#### Example: Testing a service with MSW

Given this service that validates against a Zod schema:

```js
// services/service-get-items.js
import { providerGetItems } from '../providers/provider-get-items'
import { itemSchema } from '../../types/api/Item.response'
import { utilCheckResponseSchema } from '../../../_core/utils/util-check-response-schema'
import { z } from 'zod'

export const serviceGetItems = async ({ signal }) => {
  const data = await providerGetItems({ signal })
  return utilCheckResponseSchema(data, z.array(itemSchema))
}
```

```js
// services/__tests__/service-get-items.integration.test.js
import { http, HttpResponse } from 'msw'
import { server } from '../../../tests/mocks/server'
import { serviceGetItems } from '../service-get-items'
import { generateItemList, generateItem } from '../../../factories/item'

describe('serviceGetItems integration', () => {
  it('returns validated items on success', async () => {
    const items = generateItemList(2)
    server.use(
      http.get('*/api/items', () => HttpResponse.json(items))
    )

    const result = await serviceGetItems({ signal: new AbortController().signal })

    expect(result).toHaveLength(2)
    expect(result[0]).toMatchObject({ id: expect.any(String), title: expect.any(String) })
  })

  it('rejects malformed responses (wrong types)', async () => {
    server.use(
      http.get('*/api/items', () =>
        HttpResponse.json([{ id: 123, title: 'Test' }]) // id is number, schema expects string
      )
    )

    await expect(
      serviceGetItems({ signal: new AbortController().signal })
    ).rejects.toThrow()
  })

  it('rejects responses with missing required fields', async () => {
    server.use(
      http.get('*/api/items', () =>
        HttpResponse.json([{ title: 'Test' }]) // missing id
      )
    )

    await expect(
      serviceGetItems({ signal: new AbortController().signal })
    ).rejects.toThrow()
  })

  it('returns an empty array for empty lists', async () => {
    server.use(
      http.get('*/api/items', () => HttpResponse.json([]))
    )

    const result = await serviceGetItems({ signal: new AbortController().signal })

    expect(result).toEqual([])
  })

  it('propagates HTTP 500 errors', async () => {
    server.use(
      http.get('*/api/items', () =>
        HttpResponse.json({ message: 'Server error' }, { status: 500 })
      )
    )

    await expect(
      serviceGetItems({ signal: new AbortController().signal })
    ).rejects.toThrow()
  })

  it('handles network errors', async () => {
    server.use(
      http.get('*/api/items', () => HttpResponse.error())
    )

    await expect(
      serviceGetItems({ signal: new AbortController().signal })
    ).rejects.toThrow()
  })
})
```

#### Why MSW instead of mocking the provider directly

Mocking `providerGetItems` with `vi.fn()` would test that the service can call a mocked function — it would NOT catch:
- Schema mismatches (the mock returns exactly what you tell it, not what the real server sends)
- HTTP error propagation
- Network error handling

MSW intercepts at the network level, so the entire `provider → client → fetch` chain runs for real — only the network is simulated. This catches issues that a direct mock would miss, while still being fast and deterministic.

#### What about providers that DON'T use HTTP (localStorage, IndexedDB)?

If a service uses a provider that reads from `localStorage` instead of HTTP, MSW is irrelevant. In that case, write a different kind of integration test:

```js
// providers/__tests__/provider-get-items-localstorage.unit.test.js
import { providerGetItemsLocalStorage } from '../provider-get-items-localstorage'

it('reads items from localStorage', () => {
  const items = [{ id: '1', title: 'Test' }]
  localStorage.setItem('items', JSON.stringify(items))

  const result = providerGetItemsLocalStorage()

  expect(result).toEqual(items)
})

it('returns an empty array when nothing is stored', () => {
  localStorage.removeItem('items')

  const result = providerGetItemsLocalStorage()

  expect(result).toEqual([])
})
```

These are unit tests for the **provider**, not service integration tests. The provider is the adapter — test its contract (read/write to the correct key, parse correctly, handle missing data).

---

### 3. Components with Stores — Integration Tests (RTL + custom render)

You test the store **IMPLICITLY** by testing the components that use it. Never test store actions, getters, or mutations in isolation — that doesn't generate user-observable results. What matters is: does the UI reflect the correct state?

**Set Zustand state for different scenarios** before rendering:

```js
import { useAuthStore } from '@/modules/auth/_shared/stores/store-auth'
import { useLinkStore } from '@/modules/links/_shared/stores/store-links'

// Simulate logged-in user
useAuthStore.setState({ user: { id: 1, name: 'Test User' } })
// Pre-populate data
useLinkStore.setState({ links: [{ id: 1, url: 'https://ejemplo.com' }] })

renderWithProviders(<LinkListSection />)
```

**Reset state between tests** to avoid leakage:

```js
afterEach(() => {
  useAuthStore.setState({ user: null })
  useLinkStore.setState({ links: [] })
})
```

**Mock at the service layer**, not the store. Let store actions call the mocked service naturally:

```js
vi.mock('@/modules/links/_shared/api/services/service-links', () => ({
  getLinks: vi.fn().mockResolvedValue([{ id: 1, url: 'https://ejemplo.com' }]),
}))

it('shows the list of links', async () => {
  useAuthStore.setState({ user: { id: 1 } })
  renderWithProviders(<LinkListSection />)
  expect(await screen.findByText('https://ejemplo.com')).toBeVisible()
})
```

**Which approach to use? Depends on how the store gets its data:**

| Scenario | Approach | Why |
|----------|----------|-----|
| Store state is **known/synchronous** (auth user, theme, feature flags) | `useAuthStore.setState(...)` before render | Set it and forget it — no async fetch involved |
| Store gets populated via an **API hook** that the component uses | Mock the **service**, not the store | The hook will call the service, fetch data, and populate the store naturally. If you `setState` first, the hook will overwrite it when the service resolves |
| Store state is **initially empty** and you only care about the UI after data loads | Mock the service + wait for UI | Let the full flow run: render → hook calls mocked service → store updates → UI re-renders. Use `findBy*` or `waitFor` to assert on the result |

```js
// ✅ The component uses useLinks() which internally calls getLinks →
//    mock the service, let the hook populate the store
vi.mock('@/modules/links/_shared/api/services/service-links', () => ({
  getLinks: vi.fn().mockResolvedValue([{ id: 1, url: 'https://ejemplo.com' }]),
}))

it('loads and displays links', async () => {
  renderWithProviders(<LinkListSection />)
  // No setState — the hook fetches, service is mocked, store updates naturally
  expect(await screen.findByText('https://ejemplo.com')).toBeVisible()
})

// ✅ Store holds synchronous auth state — setState is fine
it('shows create button when logged in', () => {
  useAuthStore.setState({ user: { id: 1, name: 'Test' } })
  renderWithProviders(<LinkListSection />)
  expect(screen.getByRole('button', { name: /create/i })).toBeVisible()
})
```

**Exception — direct store tests:** Only test a store in isolation if it contains complex validators or computed selectors with genuine business logic (e.g., `validateSlug()`, `calculateDiscount()`). Simple get/set stores never need their own test — they get validated through the components.

### 4. Core Components with State — Component Tests (RTL + axe)

Test core components that have **state, interaction, or composition logic**. The decision follows the ABCD+L architecture (see `component-architecture` skill):

| Class | Test? | Why |
|-------|-------|-----|
| **A** (Atom — `a/button`, `a/label`) | ❌ Never | Purely presentational. No logic to test. |
| **B** (Base — `b/modal`, `b/dropdown`) | ✅ If it has state or interaction | A `dropdown` that opens/closes, a `modal` with show/hide — yes. A `button` that just renders — no. |
| **C** (Composite — `c/modal-danger`) | ✅ If it has conditional logic | A `c-modal-danger` that hardcodes props only — no. One that conditionally renders sections — yes. |
| **D** (Design — `d/card-header`) | ✅ Always | Complex structures composing multiple sub-components. High value in verifying they render correctly. |
| **L** (Layout — `l/navbar`) | ⚠️ Only if it has state | A `l-navbar` with a mobile collapsible menu — yes. A static `l-main-section` — no. |

Unifying criterion: **if it has internal state, user interaction (click, type, hover), or non-trivial composition — test it**. The class prefix is a quick heuristic, not an absolute rule.

```js
it('toggles the dropdown on click', async () => {
  const user = userEvent.setup()
  const { container } = render(BDropdown, { props: { items: ['Edit', 'Delete'] } })
  await user.click(screen.getByRole('button', { name: /options/i }))
  expect(screen.getByText('Delete')).toBeVisible()
  expect(await axe(container)).toHaveNoViolations()
})
```

### 5. E2E per Module — Playwright, Happy Path Only

One happy-path flow per module. Edge cases and errors stay in integration tests.

| Module | E2E Flow |
|--------|----------|
| Auth | register → login → logout |
| Items | create item → see it in list → detail |
| Settings | update profile → see changes |

### 6. Legacy / Direct `fetch` in Components — Guard Clause

> ⚠️ **This is NOT the ideal scenario.** The 4 levels above assume HTTP calls go through a service/repository layer. That is the correct pattern and you should NOT look for this section unless you have to.

Before writing any Section test, check: **does this component call `fetch()` / `axios` directly without going through a service?**

If it does NOT — you have services, providers, or a repository layer — follow Section 1 above. Mock the service and move on.

If it DOES call `fetch()` directly (legacy code, no time to refactor), stop and ask:

> **"This component calls `fetch()` directly. The ideal fix is to extract those calls into a service/repository. But if you can't refactor right now, we can use MSW as a bridge. Which approach do you prefer?"**

| Option | When | How |
|--------|------|-----|
| **A — Extract service first** (recommended) | You can refactor safely | Move the `fetch()` call into a service, then mock the service following Section 1 |
| **B — MSW as bridge** | Legacy code, no test coverage yet, refactoring is risky | Intercept HTTP at the network level with MSW. You keep the `fetch()` in the component but MSW catches it and returns fake data |

**Why not mock `window.fetch` directly?** It's fragile — you couple the test to implementation details (which method, how `json()` is called, how headers are structured). MSW is better if you must intercept at the HTTP level, but extracting a service is always superior.

```js
// ✅ Option A — Extract service, then mock it (same as Section 1)
// service.js
export const getItems = () => fetch('/api/items').then(r => r.json())

// Test
vi.mock('../services/service-items', () => ({
  getItems: vi.fn().mockResolvedValue([{ id: 1 }])
}))

// ✅ Option B — MSW intercepts at network level
import { server } from '../mocks/server'
import { getItems } from '../mocks/handlers'

it('lists items', async () => {
  server.use(getItems([{ id: 1, title: 'Test' }]))
  render(<ItemsSection />)
  expect(await screen.findByText('Test')).toBeInTheDocument()
})
```

**Bottom line:** The presence of `fetch()` in a component is a code smell. The skill's default path (mock the service) assumes the architecture is healthy. This guard clause exists only for the cases where it isn't.

### 7. Custom Renderers — Avoiding Provider Boilerplate

**Problem:** Every integration test needs providers (router, toast, store reset, etc.). Repeating them in every test is tedious and fragile — if a provider changes, you update every file.

**Solution:** Create a single `test-utils.jsx` with a custom `renderWithProviders` that wraps Testing Library's `render` with all your global providers. Re-export everything from Testing Library so tests import from your utils instead.

```jsx
// test-utils.jsx
import { render } from '@testing-library/react'
import { MemoryRouter } from 'react-router-dom'
import { Toaster } from 'react-hot-toast'

function renderWithProviders(ui, { route = '/' } = {}) {
  return render(
    <MemoryRouter initialEntries={[route]}>
      <Toaster />
      {ui}
    </MemoryRouter>
  )
}

// Re-export everything from Testing Library
export * from '@testing-library/react'
export { renderWithProviders }
```

**Usage in tests:**

```jsx
// Import from your utils, not from @testing-library/react directly
import { renderWithProviders, screen, waitFor } from 'test-utils'

it('renders the link list', async () => {
  renderWithProviders(<LinkListSection />)
  expect(await screen.findByText('My links')).toBeVisible()
})
```

**Adding store state overrides:**

If a component needs specific store state, set it before rendering (see Section 3) or pass it through the custom render:

```jsx
function renderWithProviders(ui, { route = '/', preloadedState = {} } = {}) {
  // Apply Zustand state before rendering
  Object.entries(preloadedState).forEach(([storeName, state]) => {
    stores[storeName].setState(state)
  })

  return render(
    <MemoryRouter initialEntries={[route]}>
      <Toaster />
      {ui}
    </MemoryRouter>
  )
}
```

**Reset between tests** in your setup file:

```jsx
// vitest-setup.js
import { useAuthStore } from '@/modules/auth/_shared/stores/store-auth'
import { useLinkStore } from '@/modules/links/_shared/stores/store-links'

afterEach(() => {
  useAuthStore.setState({ user: null })
  useLinkStore.setState({ links: [] })
})
```

You can create multiple custom renders for different needs (`renderWithStore`, `renderWithRouter`, etc.), but one comprehensive `renderWithProviders` usually covers most cases.

### 8. Test Data with Factories (Fishery + Faker)

**Problem:** You hardcode the same test data across tests (`{ id: 1, name: 'Test User' }`). If the object structure changes, you update every file. And you always test with the same values — missing edge cases like emojis, long strings, or edge dates.

**Solution:** Create factories that generate random data. Each test run gets different values, increasing the chance of catching unexpected edge cases.

**Create a factory per domain entity:**

```js
// factories/user.js
import { Factory } from 'fishery'
import { name, internet } from 'faker'

const userFactory = Factory.define(({ sequence }) => ({
  id: sequence,
  name: `${name.firstName()} ${name.lastName()}`,
  email: internet.email(),
  avatar: internet.avatar(),
}))

export function generateUser(overrides = {}) {
  return userFactory.build(overrides)
}

export function generateUserList(count = 5) {
  return userFactory.buildList(count)
}
```

```js
// factories/link.js
import { Factory } from 'fishery'
import { internet, date } from 'faker'
import { generateUser } from './user'

const linkFactory = Factory.define(({ sequence }) => ({
  id: sequence,
  url: internet.url(),
  slug: `link-${sequence}`,
  createdAt: date.recent().toISOString(),
  user: generateUser(),  // nested factory
}))

export function generateLink(overrides = {}) {
  return linkFactory.build(overrides)
}

export function generateLinkList(count = 3) {
  return linkFactory.buildList(count)
}
```

**Usage in tests:**

```js
import { generateLinkList, generateLink } from 'factories/link'
import { generateUser } from 'factories/user'

// Random data — different every test run
const links = generateLinkList()
const link = generateLink({ slug: 'fixed-slug' })  // override specific field

// Mock services with factory data
vi.mock('@/modules/links/_shared/api/services/service-links', () => ({
  getLinks: vi.fn().mockResolvedValue(generateLinkList()),
}))

it('displays links', async () => {
  renderWithProviders(<LinkListSection />)
  expect(await screen.findByText(links[0].url)).toBeVisible()
})
```

**Override specific fields** when a test cares about a particular value — the rest stay random:

```js
it('uses the custom slug', () => {
  const link = generateLink({ slug: 'my-custom-slug' })
  expect(link.slug).toBe('my-custom-slug')
  // id, url, createdAt are still random
})
```

**Why this matters:** Random data in each run catches edge cases that hardcoded values never would — emojis in names, URLs with query params, dates across DST boundaries. And if the entity shape changes, you fix one factory instead of 20 tests.

**When to skip factories:** Simple primitive values (a string, a number) used in a single test don't need a factory. Use them for domain objects (user, link, comment) that appear in multiple tests.

## Hard Rules (Non-Negotiable)

These rules MUST be enforced in every frontend test. Deviations require explicit justification with a comment.

### 1. Accessible Queries Only

Use the priority pyramid. Never use `container.querySelector` or `data-test-id` unless ALL higher-priority options are impossible.

**Priority**: `*ByRole` > `*ByLabelText` > `*ByPlaceholderText` / `*ByDisplayValue` > `*ByText` > `*ByTitle` > `*ByTestId` (last resort).

`*ByRole` with `{name: /pattern/i}` checks the **accessible name** — text content, `aria-label`, `alt` attribute, etc.

### 2. userEvent Over fireEvent

MUST use `userEvent.setup()` for ALL user interactions. `fireEvent` is ONLY acceptable for native DOM events that `userEvent` cannot simulate.

```js
const user = userEvent.setup()
await user.click(button)
await user.type(input, 'text')
await user.selectOptions(select, ['option'])
```

Prefer per-test instances over `beforeEach`.

### 3. Never Test Implementation Details

- NO access to component instance, internal state, refs, or props
- NO checking internal method calls (`toHaveBeenCalled` on component methods)
- Test ONLY what the user sees, hears, or experiences

### 4. Full Render, Never Stub Children

- NEVER stub, mock, or shallow-render child components — it gives false confidence
- ALWAYS use Testing Library's `render()` which exercises the full component tree
- The test unit is the **use case** (container + children + repository), not an isolated component

### 5. Mock Only at the I/O Boundary

| Mock (external I/O) | Do NOT mock |
|---------------------|-------------|
| HTTP calls (`fetch`, axios) | Child components |
| User input / browser APIs | Pure logic functions (formatters, validators, time utils) |
| Repository / Service layer | Third-party libs with no side effects (dayjs, lodash) |
| `localStorage`, `navigator.clipboard` | UI library internals (Vuetify, DaisyUI) |

If you extract `fetch` into a repository, mock the repository with `mockResolvedValue`. Never mock `window.fetch` directly.

### 6. jest-dom Matchers Required (@testing-library/jest-dom)

Import `@testing-library/jest-dom/vitest` in your setup file to auto-register semantic matchers with Vitest.
Use `happy-dom` for the test environment — set `environment: 'happy-dom'` in `vitest.config.js` (faster than jsdom, no browser dependency).
Use semantic matchers. NO raw `toBeTruthy()`, `toBeDefined()`, or `!!value`.

```js
// ✅ Correct
expect(element).toBeInTheDocument()
expect(element).toHaveTextContent(/submit/i)
expect(element).toBeVisible()
expect(button).toBeDisabled()
expect(alert).toHaveClass(/error/i)

// ❌ Wrong
expect(element).toBeTruthy()
expect(element?.textContent).toBe('Submit')
```

### 7. One Scenario Per Test (AAA)

Follow Arrange-Act-Assert strictly. If you see multiple Act or Assert blocks, split into separate tests.

```js
describe('when user submits the form', () => {
  describe('and the name is empty', () => {
    it('shows a validation error', async () => {
      // Arrange
      render(ProfileForm)
      const user = userEvent.setup()

      // Act
      await user.click(screen.getByRole('button', { name: /submit/i }))

      // Assert
      expect(screen.getByRole('alert')).toHaveTextContent(/name is required/i)
    })
  })

  describe('and the name is valid', () => {
    it('submits successfully', async () => { /* ... */ })
  })
})
```

### 8. DAMP Over DRY in Tests

Self-contained tests with healthy duplication are preferred over shared abstractions.

- Avoid `beforeEach` for setup logic — use helper functions or Page Objects
- Avoid nested `describe` that shares mutable state
- Page Objects are acceptable for reducing duplication but keep them close to the test file
- If you extract helpers, they must return fresh state, not mutate shared state

### 9. No Snapshot Tests

DO NOT use `toMatchSnapshot` or `toMatchInlineSnapshot` for component output.

**Exception**: Legacy projects as a first coarse coverage layer, with explicit team waiver.

Prefer explicit assertions: `findByText(content).toBeInTheDocument()`

### 10. Include Accessibility Checks

Add `vitest-axe` to every integration test:

```js
import { axe } from 'vitest-axe'
import { toHaveNoViolations } from 'vitest-axe/matchers'
expect.extend({ toHaveNoViolations })

it('has no accessibility violations', async () => {
  const { container } = render(FormComponent)
  const results = await axe(container)
  expect(results).toHaveNoViolations()
})
```

Note: passing `toHaveNoViolations` is not 100% proof — some issues (skip-to-content, focus order) cannot be automated.

**When `toHaveNoViolations` fails: investigate the root cause in the component and FIX IT.** Do not remove the check. Common violations and their fixes:

| Violation | Root Cause | Fix |
|-----------|-----------|-----|
| `label-title-only` | Element has a `title` attribute but no visible label | Add `aria-label` or a `<span className='sr-only'>` with descriptive text inside a wrapping `<label>` |
| `color-contrast` | Text-to-background contrast ratio is too low | Adjust CSS colors in the component (check against WCAG AA ratios) |
| `aria-required-children` | ARIA role expects specific child roles | Review and correct the ARIA role hierarchy in the component |
| `button-name` | Button has no accessible name | Add text content, `aria-label`, or `aria-labelledby` to the button |
| `select-name` | Native `<select>` has no accessible label, `aria-label`, or `title` | Add `aria-label` to the `<select>` element |

`aria-label` is the minimum fix when you can't change the visual design. A `<span className='sr-only'>` is better when you want real text accessible to screen readers.

### 11. Handle Animations Without Slowing Tests

Animations are a silent test killer — a single 3 s transition per test compounds into minutes. Handle based on how the animation is implemented.

#### JS-controlled animations (e.g., `react-transition-group`, custom JS timers)

Set `process.env.TIME_SCALE_FACTOR = 0` in your test setup file. The component must read this env var:

```js
// vitest-setup.js
process.env.TIME_SCALE_FACTOR = 0

// Component
const duration = 3000 * (process.env.TIME_SCALE_FACTOR || 1)
```

Use `waitForElementToBeRemoved` as usual — animations are still async even at 0 s:

```js
await user.click(toggleButton)
await waitForElementToBeRemoved(
  screen.queryByRole('heading', { name: /subtitle/i })
)
```

Remove the custom `timeout` option — the animation completes instantly now.

#### CSS-driven animations (e.g., `react-transition-group`, CSS `transition` properties)

React Testing Library does NOT auto-stub any animation components — they render fully in tests. For CSS-based animations:

- **`react-transition-group`**: It's JS-controlled under the hood (it uses timeouts), so the `TIME_SCALE_FACTOR = 0` approach in the JS section above works directly.
- **Pure CSS transitions/animations**: With `happy-dom` they're effectively no-ops — DOM updates are instant regardless of CSS `transition-duration`. No extra work needed.
- **CSS animations that use `onTransitionEnd` / `onAnimationEnd`**: If your component logic depends on these events firing, use `vi.useFakeTimers()` and advance time with `vi.runAllTimers()` to trigger them.

#### When NOT to accelerate

If the test is specifically verifying animation timing or sequencing, keep real durations. Default: **accelerate**.

---

## Decision Gates

| Need | Prescribed Action |
|------|-------------------|
| Selecting a query | `*ByRole` > `*ByLabelText` > `*ByPlaceholderText` > `*ByText` > `*ByTitle` > `*ByTestId` |
| User interaction | `userEvent.setup()`, never `fireEvent` |
| Test environment | Use `happy-dom` (faster, no browser dependency). Set `environment: 'happy-dom'` in `vitest.config.js` |
| Component mounting | Full `render()`, never stub children |
| Component calls `fetch()` directly (legacy) | **Stop.** Ask user: extract a service (preferred) or use MSW as bridge. Never mock `window.fetch` |
| API call in component | Extract to repository → mock the repository |
| Mocking a dependency | Is it external I/O? → mock. Pure logic or low-cost lib? → use real |
| Where to mock (service/provider split) | Mock the **service** layer (`service-*.js`). Providers are implementation details — swapping them must NOT break tests |
| Asserting a mocked service function | Services with `(provider, options)` signature receive the provider + `{ signal, payload }`. A plain `{ id }` assertion misses the wrapping | Match as `expect.any(Function), expect.objectContaining({ payload: expect.objectContaining({ id }) })` |
| Random test data | See Section 8 — Fishery + Faker factories. Domain objects get factories, simple primitives use inline values |
| Memory issues in CI | Use `chance` instead of Faker, or disable file parallelism (`--no-file-parallelism`) + `--expose-gc` |
| Animations | See Rule 11. JS-controlled → `TIME_SCALE_FACTOR = 0` + `waitForElementToBeRemoved`. CSS-driven → `happy-dom` no-ops pure CSS, or use fake timers for `onTransitionEnd` events |
| UI library (Vuetify, DaisyUI) | Test YOUR integration, NEVER library internals |
| Time-dependent code (synchronous, no promises in between) | `vi.useFakeTimers()` + `vi.runOnlyPendingTimers()` or `vi.advanceTimersByTime(ms)` — cleanup in `afterEach` |
| Time-dependent code (with promises/microtasks: clipboard, mock fetch, async handlers) | `vi.useFakeTimers()` + `vi.advanceTimersByTimeAsync(ms)` inside `act(async () => { ... })` — the async version flushes microtasks in addition to timers. Do NOT use `waitFor` or `userEvent` while fake timers are active |
| Vanilla JS / legacy code | Generate DOM in test → use Testing Library queries. jQuery → E2E only |
| UI library non-semantic HTML | `ByTestId` only as absolute last resort |
| E2E vs integration | E2E: happy paths only. Edge cases and errors: integration tests |
| Test descriptions | Nested `describe("when...")` > `describe("and...")` for readable `--verbose` output |

---

## Execution Steps

1. **Identify the use case boundary** — find the container component that orchestrates the logic
2. **Set up providers** — create a custom `render()` with store, router, UI lib (see custom renderers). Set `environment: 'happy-dom'` in `vitest.config.js`
3. **Define test data** — use Fishery + Faker factories (see Section 8). Domain objects get a factory, simple primitives stay inline
4. **Write test with AAA**:
   - **Arrange**: render with factory data, call `userEvent.setup()`
   - **Act**: interact through user actions only (click, type, select)
   - **Assert**: use `@testing-library/jest-dom` semantic matchers, one assertion per test
5. **Add accessibility check** — `axe(container)` + `toHaveNoViolations`
6. **Use nested describe** — `describe("when...")` inside `describe("and...")` for structured output
7. **Verify mock boundary** — confirm you mock at the repository layer, not at `fetch`
8. **Handle async** — use `findBy*` or `waitFor` for async elements, never `sleep()`

   **⚠️ `waitFor` caveat — no side effects inside the callback.** `waitFor` retries the callback repeatedly (default: every 50 ms up to 1 s) until assertions pass. If you put side effects like `userEvent.click()` or state mutations inside, they execute on EVERY retry:

   ```js
   // ❌ Wrong — the click runs on every retry
   await waitFor(() => {
     const button = screen.getByRole('button', { name: /submit/i })
     userEvent.click(button)  // side effect inside waitFor
   })

   // ✅ Correct — wait for the element first, then act
   const button = await screen.findByRole('button', { name: /submit/i })
   await userEvent.click(button)
   ```

   Simple rule: if you need to **wait** before acting, use `findBy*`. If you need to **retry an assertion**, `waitFor` is fine — but assertions only inside.

### Test Cleanup Checklist
- [ ] No shared state between tests
- [ ] If using fake timers: `vi.useRealTimers()` in `afterEach`
- [ ] If polling/interval: verify timer is cleared on unmount (no ghost timers)
- [ ] No side effects inside `waitFor` callbacks (assertions only)
- [ ] Query methods correct: `getBy*` (assert exists + sync), `findBy*` (assert exists + async), `queryBy*` (assert does NOT exist)
  - 🚨 Common anti-pattern: `expect(screen.queryByText('x')).toBeInTheDocument()` → use `screen.getByText('x')` which throws if not found
  - `queryBy*` is only correct for `expect(...).not.toBeInTheDocument()` — you WANT it to return null without throwing

---

## Common Anti-Patterns (Flag These)

| Anti-Pattern | Why It's Wrong | Fix |
|-------------|---------------|-----|
| `container.querySelector('#foo')` | Tests implementation, bypasses accessibility | Use `getByRole` / `getByLabelText` |
| `setState` or `ref.current.value = 'x'` directly | Tests internal state, not user behavior | Interact via the UI (click, type) |
| `vi.mock('../../api')` in every test | Fragile, coupled to file paths | Extract to a service repository and mock that |
| `beforeEach` with 10+ lines of setup | Hides test context, breaks readability | Use factory + small helper functions |
| Snapshot tests (`toMatchSnapshot`) | Catches irrelevant markup changes | Use explicit assertions on behavior |
| `await sleep(1000)` in tests | Slow, flaky, nondeterministic | Use `findBy*` or fake timers |
| `queryByText(...) + toBeInTheDocument()` | `queryBy*` returns `null` silently — you're using a "does-not-exist" query to assert existence. Wrong semantic signal, worse error messages | Use `getByText(...)` for existence; `queryBy*` only for `expect(...).not.toBeInTheDocument()` |
| Side effects inside `waitFor` (click, type, state mutation) | `waitFor` retries the callback until assertions pass — side effects run N times, causing flaky tests or duplicate mutations | Use `findBy*` to wait for async elements; `waitFor` for assertions only |
| Testing library internals (Vuetify autocomplete) | Tests break on lib update | Test user flow, not lib internals |
| `vi.clearAllMocks()` without `vi.resetAllMocks()` | Mock behavior leaks between tests | Use `vi.resetAllMocks()` or set `restoreMocks: true` in vitest config |
| `fireEvent.click(button)` | Skips visibility and interactivity checks | Use `userEvent.setup()` |
| `findByText(...)` when the same text appears in multiple elements (toast + inline alert) | `document.querySelectorAll` returns multiple matches → `findByText` throws "Found multiple elements" | Use `findAllByText(...)` and assert `length >= 1`, or use a specific selector like `container.querySelector('.alert-error')` |
| `navigator.clipboard` spy assertion with `restoreMocks: true` | happy-dom's clipboard is a read-only getter. `Object.defineProperty` + `vi.fn()` may lose spy identity after `clearAllMocks` + `restoreMocks` | Assert on side effects instead (e.g., `findByText('Copied!')`), or use `vi.stubGlobal('navigator', { ...navigator, clipboard: { writeText: vi.fn() } })` |
| `expect(fn).toHaveBeenCalled()` on component methods | Tests implementation coupling | Assert on visible DOM changes instead |
| Native `<dialog>` + `getByRole('dialog')` when closed | A `<dialog>` without the `open` attribute is considered **hidden/inaccessible** by Testing Library. `getByRole('dialog')` throws `Unable to find an accessible element` | Use `getByRole('dialog', { hidden: true })` to find it even when closed |
| Native `<dialog>` + `userEvent.keyboard('{Escape}')` | happy-dom does NOT implement the native `<dialog>` ESC close behavior. No `close` event is dispatched automatically | Dispatch the event manually: `dialog.dispatchEvent(new Event('close'))`. The component must listen with `addEventListener('close', handleClose)` |
| `waitFor` / `findBy*` + `vi.useFakeTimers()` | `waitFor` uses `setTimeout` internally for polling. With fake timers, those timeouts never fire → the test hangs until the global timeout | Use `fireEvent.click()` (synchronous) + `vi.advanceTimersByTimeAsync()` + `act` instead of `userEvent` + `waitFor`. Or split the test: assertions with real timers first, then fake timers to advance |
| `userEvent.click()` + `vi.useFakeTimers()` | `userEvent` uses `setTimeout` internally to simulate hover, pointer, and browser events. With fake timers, the simulation never completes | Use `fireEvent.click()` (synchronous) + `await act(async () => { await vi.advanceTimersByTimeAsync(0) })` to flush both microtasks and re-renders |