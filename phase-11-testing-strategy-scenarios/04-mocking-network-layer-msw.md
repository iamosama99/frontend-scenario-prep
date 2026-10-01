# Mocking the Network Layer for Integration Tests (MSW)

## Quick Reference

| Decision | Mechanism | Why it's the right call |
|---|---|---|
| Mock at the network boundary | MSW intercepts at the request level (Service Worker in browser, interceptors in Node) | App code, fetch client, serialization all run for real |
| One handler set, many contexts | Shared `handlers.ts` reused by unit/integration tests, Storybook, and local dev | Single source of truth for API fixtures |
| Defaults + per-test overrides | `server.use(...)` for the test, `resetHandlers()` in `afterEach` | Tests stay isolated; happy path is the default |
| Fail loudly on the unexpected | `onUnhandledRequest: 'error'` | An un-mocked call is a bug in the test, not something to ignore |

## The Scenario

"Our integration tests currently mock `fetch` with `jest.fn()` in every file, and half of them mock the `api.ts` module instead. Tests pass, but we keep shipping bugs where the real request is wrong — wrong URL, missing header, bad JSON parsing. A teammate suggested MSW. Convince me, and show me how you'd set it up."

## Clarifying Questions

- **What runs the tests — Jest/Vitest with jsdom, Playwright component tests, Cypress?** MSW has Node (`msw/node`) and browser (`msw/browser`) integrations; the runner decides which.
- **REST, GraphQL, or both?** MSW has handlers for both; affects handler shape.
- **Is there an OpenAPI/GraphQL schema or types we can derive fixtures from?** If so, generate typed handlers/mocks to prevent fixture drift.
- **Do we also want mocks for local dev and Storybook?** That's the case for a shared handler library.
- **Is there auth/cookies/CSRF behavior that matters to the code under test?** Real request semantics (headers, credentials) are part of what we gain.

## Approach & Trade-offs

**The problem with `jest.fn()` on fetch or mocking `api.ts`.** The mock is on *our side of the line*. Anything between the component and the mock — URL construction, query-string encoding, headers, body serialization, response parsing, error mapping — is never executed, so the test cannot fail for the exact bug class we keep shipping. Worse, mocks encode the author's belief about the API and silently drift from reality.

**MSW moves the seam to the network.** The app issues a genuine `fetch`; MSW intercepts the outgoing request and returns a mocked response. Everything above the wire runs for real. The same handlers work in Node tests, in the browser for Storybook/dev, and conceptually mirror a real server.

**Trade-offs and honest limits.**

- *Setup cost* — a server, handlers, lifecycle wiring. Paid once.
- *Fixtures can still drift* from the real API. MSW doesn't fix that alone; pair with generated types from the schema and a small number of contract/E2E tests against the real backend.
- *It's still not the real network* — no real CORS, TLS, or latency behavior. CORS bugs won't appear in jsdom tests.
- *Over-mocking* — hundreds of per-test handlers recreate the old problem. Keep a sensible default handler set and override only what the test is about.

**Default vs. override.** Defaults represent "a healthy backend." Each test overrides the single thing it cares about (a 500, an empty list, a slow response). That keeps tests readable: the override *is* the test's premise.

## Solution

### Handlers (shared)

```ts
// test/handlers.ts
import { http, HttpResponse, delay } from 'msw';
import type { Product } from '../src/api/types';

export const handlers = [
  http.get('/api/products', ({ request }) => {
    const q = new URL(request.url).searchParams.get('q') ?? '';
    return HttpResponse.json<Product[]>(db.products.filter(p => p.name.includes(q)));
  }),
  http.post('/api/cart/items', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: crypto.randomUUID(), ...body }, { status: 201 });
  }),
];
```

### Server and lifecycle

```ts
// test/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';
export const server = setupServer(...handlers);

// test/setup.ts (setupFilesAfterEach)
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => { server.resetHandlers(); /* + queryClient.clear() */ });
afterAll(() => server.close());
```

### Per-test overrides — errors, slowness, request assertions

```tsx
it('shows an error state and retry on 500', async () => {
  server.use(http.get('/api/products', () => new HttpResponse(null, { status: 500 })));
  render(<ProductList />);
  expect(await screen.findByRole('alert')).toHaveTextContent(/something went wrong/i);
  // retry succeeds after restoring defaults
  server.resetHandlers();
  await userEvent.click(screen.getByRole('button', { name: /retry/i }));
  expect(await screen.findByText('Widget')).toBeInTheDocument();
});

it('sends the right request', async () => {
  let captured: Request | undefined;
  server.use(http.post('/api/cart/items', ({ request }) => {
    captured = request.clone();
    return HttpResponse.json({ id: '1' }, { status: 201 });
  }));
  render(<AddToCart sku="A1" />);
  await userEvent.click(screen.getByRole('button', { name: /add/i }));
  await waitFor(() => expect(captured).toBeDefined());
  expect(captured!.headers.get('content-type')).toBe('application/json');
  expect(await captured!.json()).toEqual({ sku: 'A1', qty: 1 });
});

it('shows a skeleton while loading', async () => {
  server.use(http.get('/api/products', async () => { await delay('infinite'); }));
  render(<ProductList />);
  expect(screen.getByTestId('skeleton')).toBeInTheDocument();
});
```

### Reuse in the browser

```ts
// .storybook/preview.ts or src/mocks/browser.ts
import { setupWorker } from 'msw/browser';
import { handlers } from '../test/handlers';
export const worker = setupWorker(...handlers);
```

### Test isolation with a data-fetching cache

```tsx
function renderWithProviders(ui: ReactElement) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>);
}
```

Fresh client per test and `retry: false` so error tests don't wait through backoff.

> **Check yourself:** Name three classes of bug a `jest.fn()` fetch mock cannot catch that MSW can, and one that MSW still cannot.

## Gotchas

- **Shared cache between tests.** React Query/SWR cache leaking across tests makes results order-dependent; new client per test.
- **Forgetting `resetHandlers`.** A `server.use` override leaking into the next test is a classic order-dependent flake.
- **`onUnhandledRequest: 'warn'` (default).** Un-mocked requests pass through or just log; use `'error'` so missing mocks fail loudly.
- **Retries making error tests slow.** Disable retries in the test client.
- **Relative URLs in Node.** jsdom needs a base URL (`testEnvironmentOptions.url`) or absolute handler URLs.
- **Fixture drift.** Handlers return what someone *thought* the API returns; generate from schema/types and add contract tests.
- **Putting assertions inside handlers.** Failures inside the interceptor surface as confusing network errors; capture the request and assert in the test body.

## Follow-up Questions

**Q (High): Why MSW over mocking `fetch` or the API module?**

Answer: It tests the real client path — URL, headers, serialization, parsing, error mapping — because the mock sits at the network, not in our code. It's implementation-agnostic (swap fetch for axios and tests keep passing), reusable across Jest, Storybook, and dev, and expresses behavior as "the server responds with X," which reads like the real world.

The trap: "It's just fancier mocking." The point is *where* the seam is.

**Q (High): How do you keep mocked responses from drifting away from the real API?**

Answer: Generate types (and ideally handlers/fixtures) from the OpenAPI or GraphQL schema so a breaking change fails compilation; share a single handler set rather than ad-hoc fixtures; and keep a few contract tests or E2E tests against the real backend/staging. Consumer-driven contracts (Pact) if teams are separate.

The trap: assuming MSW handlers are "the truth."

**Q (Medium): How do you test loading, error, and empty states?**

Answer: Override the default handler per test: `delay('infinite')` for loading, a 4xx/5xx or `HttpResponse.error()` for failures, an empty array for empty. The default is the healthy path; the override is the premise of the test.

The trap: testing only success because errors are "hard to trigger."

**Q (Medium): What about E2E — should Playwright use MSW too?**

Answer: Sparingly. For E2E I prefer real backend for first-party calls and `page.route` (or MSW in-browser) for third parties and hard-to-trigger failures. Using MSW everywhere in E2E recreates the "tests pass, prod broken" problem at a higher cost.

The trap: mocking the entire backend in E2E and calling it end-to-end.

**Q (Low): Can MSW test CORS or real network failures?**

Answer: No. It intercepts before the real network stack, so CORS preflight, TLS, and actual timeouts aren't exercised. Those need a real browser against a real (or dev) server.

The trap: claiming MSW covers "all network concerns."

## Self-Assessment

- [ ] Can explain the seam difference: module mock vs. fetch mock vs. network-level mock
- [ ] Can set up `setupServer`, lifecycle hooks, and `onUnhandledRequest: 'error'` from memory
- [ ] Can write per-test overrides for 500, empty, slow, and network-error cases
- [ ] Can capture and assert on an outgoing request correctly
- [ ] Can describe how to prevent fixture drift
- [ ] Can state what MSW can't test (CORS, real latency, TLS)

---
*Next: Visual Regression False Positives — Handling Them — another place where a test layer's signal-to-noise ratio decides whether anyone trusts it.*
