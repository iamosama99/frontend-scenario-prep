# Testing a Race-condition-prone Async Component

## Quick Reference

| Technique | Mechanism | Why it's the right call |
|---|---|---|
| Control the race, don't hope for it | Hand-resolved promises (deferreds) so the test decides response order | Races are timing-dependent; tests must make timing deterministic |
| Assert the invariant | "UI always reflects the *latest* request," regardless of arrival order | Tests the contract, not the implementation (AbortController vs. ignore flag) |
| Fake timers for debounce/retry | `vi.useFakeTimers()` + `advanceTimersByTimeAsync` | No real sleeping; exact control over time |
| Test cleanup | Unmount mid-flight; assert no state update / no leak | The other half of async correctness |

## The Scenario

"We have a search box that fetches results as you type. Users sometimes see results for an old query after typing a new one. We've fixed it — or think we have. How do you write a test that would have caught this, and would fail if someone reintroduced the bug?"

## Clarifying Questions

- **How is the fix implemented — AbortController, a request-id/ignore flag, or a library (React Query, switchMap)?** I want tests agnostic to this, but I need to know whether aborts are observable (a `signal` I can check) or purely ignored.
- **Is there debounce?** If so I need fake timers, and the race exists *after* the debounce fires for two queries close together.
- **What should the UI show for the superseded request — nothing, a stale-while-revalidate list, a spinner?** Defines the exact assertion.
- **What about errors from the stale request?** An aborted/late failing request must not clobber a newer success.
- **Can responses be out of order in practice?** Yes — any real network can reorder — so "just await sequentially" tests don't model reality.

## Approach & Trade-offs

**The bug is about ordering, so the test must own ordering.** A normal mocked fetch resolves in call order after a tick; that can never reproduce "response 1 arrives after response 2." I need promises I resolve manually, in the adversarial order: type "a" → request A (pending), type "ab" → request B (pending), resolve B, then resolve A. Correct UI shows B's results; the buggy version shows A's.

**Assert the invariant, not the mechanism.** "After both settle in any order, the screen shows the results for the latest input." This passes whether the fix is AbortController, an ignore flag, or `useQuery` — so refactoring the fix doesn't break the test. A secondary, mechanism-specific test (that the first request's signal is aborted) is optional and clearly labeled as an implementation detail.

**Real timers vs. fake timers.** Real `setTimeout` waits make debounce tests slow and flaky. Fake timers give determinism but have a classic hazard: they don't flush promise microtasks unless you use the async variants. Trade-off accepted; use `advanceTimersByTimeAsync`.

**Prove the test can fail.** A race test that passes on the buggy code is worthless. I'd temporarily revert the fix (or mutation-test it) and watch the test go red before trusting it.

## Solution

### The component under test

```tsx
function Search() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<string[]>([]);

  useEffect(() => {
    if (!query) { setResults([]); return; }
    const ctrl = new AbortController();
    const t = setTimeout(async () => {
      try {
        const res = await fetch(`/api/search?q=${query}`, { signal: ctrl.signal });
        setResults(await res.json());
      } catch (e) {
        if ((e as Error).name !== 'AbortError') throw e;
      }
    }, 300);
    return () => { clearTimeout(t); ctrl.abort(); };
  }, [query]);

  return (/* input + <ul> of results */);
}
```

### A deferred helper

```ts
function deferred<T>() {
  let resolve!: (v: T) => void, reject!: (e: unknown) => void;
  const promise = new Promise<T>((res, rej) => { resolve = res; reject = rej; });
  return { promise, resolve, reject };
}
```

### The out-of-order test

```tsx
it('shows results for the latest query even if responses arrive out of order', async () => {
  vi.useFakeTimers();
  const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });

  const a = deferred<Response>(), b = deferred<Response>();
  const fetchMock = vi.fn()
    .mockImplementationOnce(() => a.promise)   // "a"
    .mockImplementationOnce(() => b.promise);  // "ab"
  vi.stubGlobal('fetch', fetchMock);

  render(<Search />);
  const input = screen.getByRole('searchbox');

  await user.type(input, 'a');
  await vi.advanceTimersByTimeAsync(300);      // request A in flight
  await user.type(input, 'b');
  await vi.advanceTimersByTimeAsync(300);      // request B in flight
  expect(fetchMock).toHaveBeenCalledTimes(2);

  // Adversarial order: newer first, older last
  await act(async () => { b.resolve(json(['apple banana'])); });
  await act(async () => { a.resolve(json(['avocado'])); });

  expect(screen.getByText('apple banana')).toBeInTheDocument();
  expect(screen.queryByText('avocado')).not.toBeInTheDocument();
});
```

Note: if the component aborts A, the mock should honor the signal (reject with `AbortError`) — in that case resolving `a` later is a no-op; the test still passes, which is exactly the point: it asserts the outcome.

### A stale failure must not clobber a newer success

```tsx
it('ignores an error from a superseded request', async () => {
  // A pending then B resolves ok, then A rejects with a network error
  ...
  expect(screen.queryByRole('alert')).not.toBeInTheDocument();
  expect(screen.getByText('apple banana')).toBeInTheDocument();
});
```

### Unmount mid-flight

```tsx
it('does not update state after unmount', async () => {
  const errSpy = vi.spyOn(console, 'error').mockImplementation(() => {});
  const d = deferred<Response>();
  vi.stubGlobal('fetch', vi.fn(() => d.promise));
  const { unmount } = render(<Search />);
  await user.type(screen.getByRole('searchbox'), 'a');
  await vi.advanceTimersByTimeAsync(300);
  unmount();
  await act(async () => d.resolve(json(['x'])));
  expect(errSpy).not.toHaveBeenCalled();
});
```

> **Check yourself:** Why would `mockResolvedValueOnce` twice never catch this bug, and what exactly do you change so the test is capable of failing?

## Gotchas

- **Sequentially-resolving mocks.** They resolve in call order and so can never produce the race; the test passes on buggy code.
- **A test that was never red.** Always verify it fails against the pre-fix code.
- **Fake timers + promises.** `advanceTimersByTime` (sync) doesn't flush async work; use the `Async` variant and `act`.
- **Asserting on the abort mechanism only.** Locks the test to AbortController and misses the ignore-flag/React Query implementations of a correct fix.
- **Forgetting the symmetric cases.** Late *errors*, late *empty* responses, and unmount-in-flight are separate bugs.
- **Real-timer `waitFor` with large timeouts.** Masks slowness and flakes under load.

## Follow-up Questions

**Q (High): How do you make a timing-dependent bug deterministic in a test?**

Answer: Remove real time and real network from the equation. Replace them with things the test resolves explicitly: deferred promises for responses, fake timers for debounce/retry. Then the test script literally says "request 2 resolves before request 1," which is the failing interleaving.

The trap: adding `setTimeout` delays to mocks to "simulate" slowness, which is just a differently-flaky test.

**Q (High): How do you know the test actually protects against the regression?**

Answer: Make it fail first: revert the fix (or comment out the abort/ignore logic) and confirm red. For critical code, mutation testing (Stryker) automates this. A test that never failed is a hypothesis, not a guard.

The trap: shipping a green test written after the fix with no red phase.

**Q (Medium): Should the test assert that `abort()` was called?**

Answer: Optionally, as a separate narrowly-scoped test — aborting is valuable (saves bandwidth) — but the primary test should assert the user-visible invariant so alternative correct implementations pass. Over-specifying the mechanism creates brittle tests.

The trap: only testing the mechanism, so a regression that bypasses it via another path goes unnoticed.

**Q (Medium): What changes if the data layer is React Query / SWR?**

Answer: The library handles staleness via query keys, so my tests shift from "is the race handled" (trust but verify with one out-of-order test) to testing my key construction and loading/error rendering. The deferred-promise technique still applies at the network boundary through MSW with controlled delays.

The trap: assuming a library means zero race tests are needed — misconfigured keys reintroduce it.

**Q (Low): Where would a real browser test add value over this one?**

Answer: Verifying real fetch abort semantics, real input event sequencing (IME composition, paste), and actual network throttling. jsdom's fetch/abort is mocked, so one Playwright test with `route` delays on the first request covers integration reality.

The trap: believing mocked-fetch tests prove real-network behavior.

## Self-Assessment

- [ ] Can write the `deferred` helper and use it to force out-of-order resolution
- [ ] Can state the invariant under test in one sentence
- [ ] Can use fake timers correctly with async work and `userEvent`
- [ ] Can explain why a test must be shown to fail before being trusted
- [ ] Can list the sibling race tests: late error, unmount-in-flight, double submit
- [ ] Can say what's mechanism vs. behavior and which to assert

---
*Next: Mocking the Network Layer for Integration Tests (MSW) — formalizes the boundary-mocking approach used throughout this phase.*
