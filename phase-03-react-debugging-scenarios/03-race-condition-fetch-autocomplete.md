# Race Condition in Fetch (Autocomplete Overwrite Bug)

## Quick Reference

| Concept | Mechanism | Fix |
|---|---|---|
| Out-of-order response race | Multiple in-flight requests resolve in a different order than they were sent; the last-to-*resolve* wins, not the last-*sent* | Ignore stale responses explicitly — via `AbortController`, a request-id/generation counter, or a cleanup-flag closure check |
| `AbortController` | Cancels the in-flight `fetch` itself, and the browser rejects it with an `AbortError` | Create one per request, abort the previous one before firing a new one, and abort in the effect's cleanup |
| Generation/id counter | A ref incremented per request; on resolve, compare the response's id to the latest id | Cheaper than aborting for non-fetch async work (e.g., a promise-based SDK with no cancel support); doesn't stop the network request, just ignores its result |
| Cleanup-flag closure | A boolean flag captured by the effect's closure, flipped in the cleanup function | The classic pre-`AbortController` pattern; still valid, slightly more boilerplate-y than a counter for multiple concurrent requests |

## The Scenario

"This autocomplete search box is misbehaving. If you type quickly — like typing 'react' one letter at a time — the results shown at the end sometimes don't match 'react' at all; they match 'r' or 're', as if an old, slower request came back after a newer, faster one and overwrote the correct results. Find the bug and fix it, and talk about which of the ways to fix this you'd actually reach for."

## Clarifying Questions

- **Is the fetch triggered directly in an `onChange` handler, or via a `useEffect` keyed off the debounced/raw query value?** This affects exactly where the fix needs to live — if it's in an effect, the effect's own cleanup function is the natural place to cancel/invalidate the previous request; if it's fired directly from an event handler with no effect involved, the cancellation bookkeeping needs to be managed some other way (a ref-held `AbortController`, most commonly).
- **Is there already a debounce on the input, and if so, is this race condition happening despite it, or is debounce itself misconfigured?** Debouncing reduces *how often* requests fire, which reduces the *frequency* of the race but does not eliminate it — even with a 300ms debounce, if request A (for "re") takes 800ms and request B (for "react", fired 300ms later) takes 200ms, B still resolves before A, and A can still land last and overwrite B's correct results. It's important not to conflate "debouncing" with "fixing the race" — they solve different problems.
- **What does the actual fetch/response-handling code look like — specifically, does it check *any* condition before calling `setState` with the response, or does every resolved promise unconditionally update state?** The bug is almost always "every promise that resolves, resolves into an unconditional `setState`" — confirming that's the shape narrows straight to the fix.
- **Does this need to also handle the case where the component unmounts while a request is in flight** (e.g., navigating away mid-search)? That's a closely related bug (calling `setState` on an unmounted component doesn't crash in modern React but is still wasted work and, in older React versions, produced a console warning) that the same fix (aborting/ignoring stale requests) also resolves as a side effect, worth mentioning since it demonstrates the fix's completeness.
- **Is the underlying API call cancellable at all** (a `fetch` request, which supports `AbortController`, versus a third-party SDK call or GraphQL client that may not expose cancellation)? This determines whether "actually cancel the network request" is even an available option, versus "let it complete but ignore the result," which is always available regardless of what the underlying call supports.

## Approach & Trade-offs

**The bug is about response *order*, not request *order*.** Requests are sent in the order the user types, but network conditions (server load, geographic routing, response payload size) mean there's no guarantee they *resolve* in that same order. If every `.then()`/`await` block unconditionally calls `setState(response)`, whichever response happens to arrive last — not whichever was requested last — wins and is what ends up displayed. This is the core mental model to state explicitly before touching any code: the fix isn't about "making requests happen in order" (they may need to be concurrent for latency reasons), it's about "making sure only the response belonging to the *most recently sent* request is allowed to update state."

**Three viable fixes, each with a different trade-off.** (1) `AbortController` — create a new controller per request, store it in a ref, call `.abort()` on the previous one before firing a new request. This is the most complete fix, because it doesn't just ignore the stale response — it actually cancels the underlying network request, saving bandwidth and server load, and the aborted `fetch` promise rejects with a distinguishable `AbortError` that a `catch` block can specifically ignore. It requires the underlying call to actually support cancellation (native `fetch` does; not every SDK does). (2) A generation/request-id counter — a `ref` incremented on every new request, with each in-flight request "remembering" the id it was issued; when a response resolves, compare its remembered id against the ref's *current* value, and only apply the response if they still match. This works regardless of whether the underlying call is cancellable, since it doesn't attempt to stop the request — it just ignores results that are no longer relevant. Trade-off: the stale request still completes and consumes bandwidth/server resources; only its *result* is discarded. (3) A boolean cleanup-flag closure inside a `useEffect` — the classic React-docs pattern: a local `let ignore = false` captured by the effect's closure, checked before calling `setState`, flipped to `true` in the effect's cleanup function (which React runs before the effect re-runs, or on unmount). Functionally similar to the counter approach for a single in-flight-request-at-a-time case, but slightly awkward to extend to genuinely concurrent (not superseding) requests.

**Which one to actually reach for.** For a `fetch`-based autocomplete specifically, `AbortController` is the best default — it's purpose-built for exactly this, it actually saves the wasted network round-trip (real cost savings, not just a correctness fix), and the abort-in-cleanup pattern composes cleanly with `useEffect`'s existing cleanup mechanism. The generation-counter approach becomes the right choice the moment the underlying async call isn't a cancellable `fetch` (a GraphQL client without built-in abort support, a WebSocket-based request/response bridge, some analytics SDK's promise-returning method) — in those cases "ignore the result" is the only lever available, so that's the one to pull.

**Debounce and race-condition handling are complementary, not substitutes for one another.** Debounce reduces the *number* of requests fired (good for reducing server load and avoiding firing a request per keystroke), but does not guarantee response ordering for whatever requests *do* fire — the fix for out-of-order resolution has to exist independently of, and in addition to, any debounce already in place. Presenting debounce as "the fix" for this bug in an interview is a common shallow answer that misses this distinction.

## Solution

Reproducing the bug — every resolved fetch unconditionally sets state:

```tsx
function Autocomplete() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<string[]>([]);

  useEffect(() => {
    if (!query) {
      setResults([]);
      return;
    }
    fetch(`/api/search?q=${encodeURIComponent(query)}`)
      .then(res => res.json())
      .then(data => setResults(data.results)); // BUG: no check that this response is still relevant
  }, [query]);

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <ul>{results.map(r => <li key={r}>{r}</li>)}</ul>
    </div>
  );
}
```

Typing "react" fires a request per keystroke (`r`, `re`, `rea`, `reac`, `react`); if the request for `r` takes longer than the request for `react` and resolves after it, `results` ends up showing matches for `r`, overwriting the correct `react` results that had briefly been correct a moment earlier.

Fix — `AbortController`, cancelling the previous request whenever a new one starts, and in the effect's cleanup:

```tsx
function Autocomplete() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<string[]>([]);

  useEffect(() => {
    if (!query) {
      setResults([]);
      return;
    }

    const controller = new AbortController();

    fetch(`/api/search?q=${encodeURIComponent(query)}`, { signal: controller.signal })
      .then(res => res.json())
      .then(data => setResults(data.results))
      .catch(err => {
        if (err.name === 'AbortError') return; // expected — a newer request superseded this one
        console.error('Search failed', err);
      });

    return () => controller.abort(); // runs before the effect re-fires on the next keystroke, and on unmount
  }, [query]);

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <ul>{results.map(r => <li key={r}>{r}</li>)}</ul>
    </div>
  );
}
```

Every keystroke's effect cleanup aborts *that* keystroke's in-flight request before the next one fires — so by the time "react" is typed, the requests for `r`, `re`, `rea`, and `reac` have all been aborted, and only `react`'s request is ever allowed to resolve into `setResults`.

Alternative — generation counter, for when the underlying call isn't cancellable:

```tsx
function AutocompleteWithNonCancellableSDK() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<string[]>([]);
  const latestRequestId = useRef(0);

  useEffect(() => {
    if (!query) {
      setResults([]);
      return;
    }

    const requestId = ++latestRequestId.current;

    searchSdk.search(query).then(data => {
      if (requestId !== latestRequestId.current) return; // a newer request has since been issued — discard
      setResults(data.results);
    });
  }, [query]);

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <ul>{results.map(r => <li key={r}>{r}</li>)}</ul>
    </div>
  );
}
```

The stale request for `r` still completes in the background (wasted work, unlike the `AbortController` version), but its result is discarded because `requestId` no longer matches `latestRequestId.current` by the time it resolves.

> **Check yourself:** In the `AbortController` version, why is checking `err.name === 'AbortError'` in the `catch` block necessary — what would the user visibly see if that check were missing and the catch block always ran `console.error` (or worse, set an error-state that rendered a "search failed" message)?

## Root Cause

The root cause is treating promise *resolution order* as if it were guaranteed to match request *initiation* order, and updating state unconditionally on resolution without checking whether the resolving request is still the one whose result should currently be reflected in the UI. Nothing about `fetch`, promises, or the browser guarantees FIFO resolution order for concurrent requests — that guarantee has to be built explicitly.

## Gotchas

**Debounce alone, presented as "the fix."** As covered above, debounce reduces frequency, not the possibility of out-of-order resolution — a debounced-but-unguarded effect can still race, just less often, which can make the bug intermittent and harder to catch in casual testing (a false sense of "it's fixed" from reduced frequency, not eliminated root cause).

**Forgetting the cleanup function runs on unmount too, and treating that as a special case needing separate handling.** It doesn't need separate handling — `AbortController`'s abort call (or the generation counter's comparison) naturally covers the unmount case for free, since an unmounted component's "current" request id/controller will never be the latest, or the component's own state setter calls simply become no-ops if the check is skipped correctly.

**Not distinguishing an aborted request's rejection from a genuine network failure in the `catch` block.** If `AbortError` isn't specifically excluded, a component might flash an "error searching" message to the user every time they type a new character quickly — a self-inflicted UX bug caused by the very code meant to fix the race.

**Applying the fix only to the "happy path" `then`, and missing that an aborted `fetch`'s `.json()` call can also throw/reject** — meaning the guard needs to be positioned to catch the abort regardless of which stage of the promise chain it interrupts, not bolted only onto the initial `fetch()` call's rejection.

**Assuming `AbortController.abort()` synchronously prevents any further code from running.** It causes the associated fetch promise to reject, asynchronously, going through the normal promise rejection path (a microtask) — abort doesn't halt execution instantly at the point of the call the way a synchronous exception would.

## Follow-up Questions

**Q (High): Walk through exactly why out-of-order response resolution is possible even though requests are sent strictly in order, and why debouncing doesn't eliminate the possibility.**

Answer: Each `fetch` call kicks off an independent round-trip whose duration depends on factors outside the client's control — server-side query cost (a broader/shorter search term might scan more rows or hit a slower code path than a longer, more specific one), network conditions, load balancer routing, even response payload size. There is no ordering guarantee between independent concurrent HTTP requests' completion times, only that each individual request/response pair is itself ordered. So it's entirely possible for request A (sent first, for "r") to still be in flight when request B (sent second, for "react") both fires and completes, with A completing after B. Debouncing only changes *how many* requests get fired and *how far apart* — it doesn't change the fact that whichever requests do fire are still independent and can still resolve out of order; a sufficiently large gap between two debounced requests makes the race statistically less likely to manifest, but does not make it structurally impossible, especially under variable network conditions (a slow request for an earlier query can still, on a bad day, take longer than a debounce window's worth of subsequent fast requests).

The trap: presenting debounce as sufficient on its own — an interviewer probing this scenario is very likely to follow up with "does debounce fully solve this?" specifically to check whether the candidate understands the distinction between reducing frequency and guaranteeing correctness.

---

**Q (High): Compare `AbortController` versus a generation-counter approach — when would you choose one over the other, concretely?**

Answer: `AbortController` is preferable whenever the underlying async operation supports cancellation (native `fetch` does natively; many HTTP client libraries — Axios, for instance — accept an abort signal or expose their own cancellation token) because it provides two benefits simultaneously: correctness (the stale response is prevented from ever reaching state) and efficiency (the actual network request is torn down, saving bandwidth on the client, and — if the server respects connection closure — potentially saving server-side compute that would otherwise complete a query nobody needs anymore). The generation-counter approach is the fallback specifically for async operations that don't expose any cancellation mechanism — a third-party SDK's promise-returning method, a GraphQL client without an abort integration, a `setTimeout`-wrapped simulated async call — where "stop the underlying work" simply isn't an available lever, so "let it finish, but discard the result" is the only remaining option. If both approaches are technically available, `AbortController` is strictly better because it avoids the wasted work the counter approach still incurs.

The trap: treating the two as interchangeable stylistic choices — the actual decision criterion is a concrete technical constraint (is the operation cancellable), not a matter of preference, and a candidate should be able to name that constraint directly rather than picking one arbitrarily.

---

**Q (High): This same component also needs to avoid calling `setState` after the component has unmounted (say, the user navigates away mid-search). Does the `AbortController` fix already handle this, or does it need separate code?**

Answer: It's already handled, for free, as a consequence of how `useEffect` cleanup works: when a component unmounts, React runs the current effect's cleanup function — which calls `controller.abort()` — before the component is actually torn down. The subsequent `catch` block sees an `AbortError` (correctly ignored) rather than attempting to call `setState` on an unmounted component. This is worth stating explicitly in an interview because it demonstrates that the fix isn't a narrow patch for one specific symptom (the visible overwrite bug) but a structurally correct solution that also happens to close a related, less visible bug (the unmounted-`setState` case) as a natural side effect of the same mechanism — modern React (18+) no longer warns about calling `setState` on an unmounted component, and doesn't crash either (the call is simply a no-op), but it's still wasted work worth avoiding, and older React versions did surface a console warning for it.

The trap: proposing a *second*, separate `isMounted` ref/flag specifically to guard against the unmount case, without recognizing that the abort-on-cleanup mechanism already subsumes it — that's solving an already-solved problem with redundant code, which suggests not having fully reasoned through what the existing fix covers.

---

**Q (Medium): If this fetch weren't inside a `useEffect` but instead called directly from the input's `onChange` handler, how would the fix change?**

Answer: The structural pattern is the same, but there's no effect cleanup to lean on, so the "cancel the previous request" bookkeeping has to be managed explicitly with a `ref` that persists across renders: store the current `AbortController` in a `useRef`, and at the start of the `onChange` handler, check if a previous controller exists and call `.abort()` on it before creating and storing a new one for the request about to be fired. Functionally equivalent to the effect-cleanup version — the effect's cleanup function was really just a convenient, automatically-invoked place to put "abort the previous one," and that logic can be relocated to the top of the handler directly when there's no effect providing that hook.

The trap: assuming the fix is somehow impossible or fundamentally different outside of `useEffect` — the actual cancellation mechanism (`AbortController`) is completely independent of where it's invoked from; only the *bookkeeping* of "when do I call abort on the previous one" needs to be relocated.

---

**Q (Medium): Suppose two requests for the exact same query string are accidentally fired in quick succession (not two different queries — a genuine duplicate). Does the fix handle that case the same way?**

Answer: Yes, identically — both the `AbortController` and generation-counter approaches operate purely on *request identity/recency*, not on whether the query strings differ. The first of the two duplicate requests gets aborted (or its result discarded) the moment the second one starts, exactly as it would for two different query strings — the fix doesn't need to special-case "are these actually the same search" because it's not trying to deduplicate identical requests; it's ensuring only the *most recently initiated* request's result is ever applied, which is correct behavior regardless of whether that most-recent request happens to be identical to an earlier one.

The trap: overcomplicating the answer by proposing query-string-based deduplication/caching as part of "the fix" — that's a legitimate, separate optimization (worth mentioning it exists, e.g., an LRU response cache keyed by query string) but conflating it with the race-condition fix itself misses that the race-condition fix doesn't care about query content at all, only about request recency.

---

**Q (Low): Would using React's `useTransition` or a data-fetching library like React Query/SWR change how you'd approach this?**

Answer: Data-fetching libraries like React Query and SWR handle exactly this race condition internally as part of their query-key-based caching model — a query keyed by the current search term automatically supersedes/cancels (or simply ignores the result of) a previous in-flight query for a different key when the key changes, because the library's internal bookkeeping is doing precisely the generation-counter/abort pattern described above, already battle-tested, without the component needing to hand-roll it. `useTransition` addresses a different, related-but-distinct concern — marking a state update as non-urgent so React can interrupt/deprioritize rendering it in favor of more urgent updates (like the input staying responsive) — it doesn't by itself solve response-ordering; it's about render scheduling priority, not network request lifecycle. In an interview, naming that a production codebase would likely reach for React Query/SWR rather than hand-rolling this is a reasonable pragmatic point, but understanding the hand-rolled version first is what demonstrates actually understanding *why* those libraries need this logic internally in the first place.

The trap: name-dropping React Query as "the fix" without being able to explain what it's doing under the hood — an interviewer who hears "just use React Query" as a complete answer to a debugging scenario is testing whether that's genuine understanding or a buzzword substitute for it.

---

## Self-Assessment

- [ ] Can explain why response resolution order isn't guaranteed to match request send order, independent of debouncing
- [ ] Can implement the `AbortController`-per-request pattern correctly, including the `AbortError` catch-block check
- [ ] Can implement the generation-counter fallback for non-cancellable async calls
- [ ] Can explain why the `AbortController` fix also transparently resolves the "unmounted component" `setState` concern
- [ ] Can articulate the concrete technical criterion (cancellable vs. not) for choosing between the two fix approaches
- [ ] Can explain what a data-fetching library is actually doing internally to avoid this class of bug

---
*Next: Memory Leak From Uncleaned Subscriptions — another case where an async or long-lived callback outlives what created it, this time causing a leak rather than incorrect data.*
