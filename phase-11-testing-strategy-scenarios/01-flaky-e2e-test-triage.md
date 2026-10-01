# Flaky E2E Test — Triage and Fix

## Quick Reference

| Step | Mechanism | Why it's the right call |
|---|---|---|
| Measure first | Pull CI history: failure rate, which step, which browser/shard, since which commit | "Flaky" is a hypothesis until you have a rate and a pattern — fixing by vibes wastes days |
| Classify the cause | Race on async UI state · shared/leaked test data · environment (CPU starvation, network) · real product bug | Each class has a different fix; retrying only hides all four |
| Fix at the root | Web-first assertions that auto-wait on the *user-visible condition*, isolated data per test, deterministic network | Replaces "wait N ms and hope" with "wait until true, up to a timeout" |
| Contain while fixing | Quarantine (tagged, still running, non-blocking) with an owner and a deadline | Keeps main green without deleting signal or normalizing red builds |

## The Scenario

"We have a checkout E2E test that fails maybe one run in six on CI. It never fails locally. People have started just hitting 're-run failed jobs' and moving on. You've just been handed it. What do you do?"

## Clarifying Questions

- **What is the actual failure rate, and is it the same step every time?** A test failing at the same assertion 90% of the time is a race at one spot; failures scattered across different steps suggest environmental starvation or shared state. It also tells me whether "one in six" is real or folklore.
- **When did it start — can we bisect against a commit, a dependency bump, a CI runner change?** A flake that appeared with a specific PR is often a real regression (a new async path, a removed loading state) rather than test noise.
- **Does it fail only on CI, only in one browser, or only when run in parallel with other tests?** "Only in parallel" points at shared data or shared server state; "only on CI" points at resource constraints (slower CPU makes races wider) or network differences.
- **Do we have traces/videos/screenshots from failed runs?** If not, step zero is turning on trace-on-first-retry so the next failure is diagnosable rather than a mystery.
- **What does the test touch — real backend, staging, or mocked network? Shared accounts or fresh data?** Shared test users and carts are the most common source of order-dependent flakes.
- **Is it blocking merges, and what's the team's tolerance for a quarantine?** Determines whether I contain first or fix inline.

## Approach & Trade-offs

**Don't start by adding retries.** Retries convert a failing signal into a slow-but-green one. They're a legitimate *mitigation* (Playwright's `retries: 2` in CI, plus trace-on-first-retry to harvest evidence), but as a fix they hide real bugs and compound: a test that flakes at 17% passes ~99.5% of the time with 2 retries, so the suite looks healthy while the underlying race — which may be a real user-facing bug — remains.

**Get evidence before theorizing.** I'd enable traces on failure, then run the test in a loop under stress: `--repeat-each=50` with `--workers` high and CPU throttled. Reproducing the flake locally under CPU pressure is the single most valuable move, because most flakes are races whose window is widened by a slow machine.

**Classify, then fix by class:**

1. **Async race against UI state** (most common). The test acts before the app is ready or asserts before it settles. Fix: assert on the condition a user would perceive, using auto-waiting assertions — never `waitForTimeout`.
2. **Shared or leaked state.** Two tests use the same user/cart, or a previous test leaves data behind. Fix: each test creates its own data via API in setup and cleans up (or uses a unique namespace), and tests are order-independent.
3. **Network/third-party nondeterminism.** A real payment sandbox or analytics call that's occasionally slow. Fix: stub what you don't own at the network layer; keep one thin contract test against the real thing, separately.
4. **A real bug.** Double-submit, a stale cache, a race in the app itself. Here the test is the *only* thing noticing. Fix the product, not the test.

**Trade-off: determinism vs. realism.** Mocking everything makes E2E fast and stable but erodes the reason E2E exists. My rule: mock what you don't control and what's slow/nondeterministic (third parties, email, payments), keep your own frontend↔backend path real.

**Quarantine, don't delete or ignore.** If it's blocking the team while I investigate, move it to a quarantined tag: still runs, still reports, doesn't gate merges, has a named owner and an expiry date. Deleting loses coverage; leaving it gating teaches everyone to ignore red.

## Solution

### The usual culprit — a fixed sleep and a stale read

```ts
// ❌ Flaky: races the network and the render
test('places an order', async ({ page }) => {
  await page.goto('/checkout');
  await page.click('text=Place order');
  await page.waitForTimeout(2000);               // hope 2s is enough on CI
  expect(await page.locator('.confirmation').count()).toBe(1); // snapshot, no retry
});
```

Two bugs: the sleep is either too long (slow suite) or too short (flake), and `count()` is a one-shot read that doesn't retry.

### Fixed — wait on the user-visible outcome

```ts
test('places an order', async ({ page }) => {
  await page.goto('/checkout');

  const placeOrder = page.getByRole('button', { name: 'Place order' });
  await placeOrder.click();

  // Auto-retrying assertion: polls until true or the timeout elapses
  await expect(page.getByRole('heading', { name: /order confirmed/i })).toBeVisible();
  await expect(page).toHaveURL(/\/orders\/\w+/);
});
```

### Isolated data per test

```ts
// fixtures.ts
export const test = base.extend<{ user: TestUser }>({
  user: async ({ request }, use) => {
    const user = await createUserViaApi(request, { email: `e2e+${crypto.randomUUID()}@example.test` });
    await use(user);
    await deleteUserViaApi(request, user.id);   // teardown even on failure
  },
});
```

### Deterministic third party, real first party

```ts
await page.route('**/api.paymentprovider.com/**', route =>
  route.fulfill({ status: 200, json: { status: 'succeeded', id: 'pi_test_123' } })
);
```

### Config: retries as mitigation with evidence capture

```ts
// playwright.config.ts
export default defineConfig({
  retries: process.env.CI ? 1 : 0,
  use: { trace: 'on-first-retry', video: 'retain-on-failure' },
  // Fail the build if a test only passed on retry — flaky ≠ passing
  failOnFlakyTests: !!process.env.CI,
});
```

### Reproduce locally under pressure

```bash
npx playwright test checkout.spec.ts --repeat-each=50 --workers=8
# plus CPU throttling in the test via CDP to widen race windows
```

> **Check yourself:** Without looking, what are the four root-cause classes of E2E flake, and which single fix technique addresses the most common one?

## Gotchas

- **`waitForTimeout` / `sleep` as the fix.** It's a bet against CI speed; it makes the suite slower *and* still flaky.
- **One-shot reads that don't retry** (`count()`, `textContent()`, `isVisible()` then `expect(boolean)`). Use auto-retrying locator assertions.
- **Retries that mask real bugs.** A test that passes on retry is a test that failed; surface it.
- **Treating "never fails locally" as exculpatory.** It means your laptop is faster than CI — the race exists in both.
- **Selectors tied to CSS/DOM structure.** A re-render that briefly detaches the element causes "element not attached" flakes; role/label locators plus actionability checks avoid this.
- **Animations and transitions.** Clicking a moving element; disable animations in test mode.
- **Dismissing a real bug as flake.** Always ask "could a fast user hit this race?" before "how do I make the test tolerate it?"

## Follow-up Questions

**Q (High): Why not just configure retries and move on?**

Answer: Retries trade signal for green. They're fine as short-term mitigation and as an evidence-gathering mechanism (trace on first retry), but they hide races that may be real product bugs, slow down CI, and let the flake rate silently grow. I'd keep retries at 1, fail the build or at least report any test that needed a retry, and track flaky-rate as a metric with an owner.

The trap: "Retries are fine" or "retries never." The senior answer separates mitigation from fix and keeps the signal visible.

**Q (High): How do you decide whether the flake is a test bug or a product bug?**

Answer: Ask whether a real user could hit the same timing. If clicking before a handler is attached, or submitting twice, breaks the app, that's a product bug the test accidentally found — fix the app (disable the button while pending, attach handlers before showing the control). If the failure only happens because the test is faster/slower than any human (asserting during a transition), it's a test bug. Traces and videos make this distinction obvious.

The trap: reflexively "fixing" the test and shipping a real double-submit bug.

**Q (Medium): How would you stop flaky tests from accumulating across a large suite?**

Answer: Measure and own it. Track per-test flake rate from CI history, auto-quarantine tests over a threshold with an owner and expiry, fail PRs that introduce new tests which flake under `--repeat-each` (a "burn-in" step for new/changed tests), and make the testing guidelines explicit: no fixed sleeps, isolated data, web-first assertions. Treat the quarantine list size as a team health metric.

The trap: only reacting test-by-test with no systemic prevention.

**Q (Medium): When is mocking the network in an E2E test the wrong call?**

Answer: When it removes what the test exists to verify. If the point is "frontend and our backend agree on the contract," mocking our own API makes the test pass while production is broken. Mock third parties and nondeterministic edges; cover the contract either with real backend runs or contract tests.

The trap: "Mock everything for speed," producing an E2E suite that tests nothing real.

**Q (Low): Why can slower CI runners cause failures that never reproduce locally?**

Answer: Races have timing windows; a slower CPU, noisy neighbors, or cold caches widen them. Parallel workers also contend for the same backend/db. Reproduce by throttling CPU, raising worker count, and repeating the test many times.

The trap: concluding "CI is just flaky" instead of reproducing the widened window.

## Self-Assessment

- [ ] Can list the four flake root-cause classes and the fix for each, unprompted
- [ ] Can explain why `waitForTimeout` is wrong and what replaces it
- [ ] Can write a fixture that provisions and tears down isolated test data
- [ ] Can explain retries as mitigation vs. fix and how to keep the signal visible
- [ ] Can describe how to reproduce a CI-only flake locally under load
- [ ] Can distinguish a product bug from a test bug using the "could a real user hit this" test

---
*Next: "How Would You Test This Component?" Exercise — moves from fixing a bad test to deciding up front what to test, at which layer, for a given component.*
