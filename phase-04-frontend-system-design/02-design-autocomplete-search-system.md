# Design an Autocomplete / Search-as-you-type System

## Quick Reference

| Decision Point | Choice | Why |
|---|---|---|
| Request lifecycle | Debounce keystrokes (~150–250ms) + `AbortController` cancellation of the previous in-flight request | Debouncing cuts request volume; cancellation (not just ignoring stale responses) frees the browser/server from doing work for a query the user already moved past |
| Stale-response ordering | A monotonically increasing request-sequence number, applied results only accepted if they're from the latest request | Network responses can resolve out of order; naive "last resolved wins" re-introduces the exact race condition this exists to prevent |
| Perceived latency | Client-side prefix cache (`Map<string, Suggestion[]>` or a trie) rendered instantly while the authoritative network request is in flight | Typing is faster than round-trip latency — showing a locally-cached superset immediately, then reconciling with the server result, hides network latency entirely for repeat/refined prefixes |
| Composing input methods (CJK, etc.) | Suppress querying while `compositionstart`…`compositionend` is active | Firing a request per intermediate IME keystroke sends garbage partial-character queries and wastes the debounce window on noise |
| Result relevance | Merge multiple ranked sources (personal history, trending, server full-text match) behind one ranking/dedup step, not three separate UI sections bolted together | A senior-level scenario expects a coherent single ranked list, not naively concatenated sources with duplicate or conflicting entries |

## The Scenario

"Design the search-as-you-type experience for our product's main search bar — think Google's or Amazon's search suggestions. As the user types, a dropdown shows relevant suggestions, updating live with each keystroke. It needs to feel instant, handle a flaky/slow network gracefully, and not show suggestions for a query the user has already changed their mind about. Walk me through the full system, client-side."

## Clarifying Questions

- **Are suggestions purely server-computed (a full-text/relevance search backend), purely client-side (filtering a small pre-loaded dataset), or a blend — e.g., instant client-side personal history plus a server round trip for broader results?** This changes the entire architecture: a small, bounded dataset (say, a settings-search box with 200 possible entries) can be filtered entirely client-side with no network involved and no race conditions to solve at all; a large corpus needs a server round trip, which is where debouncing, cancellation, and out-of-order response handling become necessary.
- **What's the acceptable latency budget, and is there a requirement for the dropdown to show *something* instantly even before the network responds?** This decides whether a client-side cache/prefix-matching layer is worth building (showing cached results immediately, then reconciling with the authoritative network response) versus a simpler "show a loading spinner until the request resolves" approach being acceptable.
- **Does the suggestion list need to blend multiple sources — recent personal searches, trending/popular queries, and live server matches — or is it a single source?** Multiple sources raise a real design question around merging/de-duplicating and ranking them into one coherent list, not just concatenating three separately-fetched arrays into three visually separate sections.
- **What alphabets/input methods need to be supported — is this English-only, or does it need to handle CJK (Chinese/Japanese/Korean) input via IME composition, right-to-left scripts, or emoji/multi-codepoint characters?** IME composition specifically changes the keystroke-handling logic (composing keystrokes shouldn't trigger a query per intermediate character), and RTL affects layout/highlighting logic, not just visual mirroring.
- **Is there a minimum query length before suggestions appear, and should previously-selected/previously-typed queries be remembered locally across sessions?** Affects both request volume (no point querying on a single character for a broad corpus) and whether `localStorage`/IndexedDB persistence of recent searches is in scope.
- **Should selecting a suggestion (or the queries typed) feed back into any analytics/ranking signal** — i.e., does this system need to report telemetry that eventually improves ranking, or is it purely a read path with no feedback loop? This affects whether the design needs an event-logging concern threaded through selection, not just fetching and rendering.

## Approach & Trade-offs

**Debounce keystrokes, but cancel — don't just ignore — the previous in-flight request, because these solve different problems.** Debouncing (waiting ~150–250ms of typing silence before firing a request) reduces how many requests get sent in the first place, which matters for both client and server load. But even with debouncing, a user can pause long enough to trigger a request, then keep typing before it resolves — at that point, the *already-sent* request is still doing real network and server work for a query that's now stale. `AbortController` lets the client actually cancel that request (the browser stops waiting on it, and well-behaved servers/CDNs can stop processing it), rather than merely disregarding its response when it eventually arrives — the latter still wastes the round trip's resources even if the UI correctly ignores the result.

**A request-sequence guard is required even with cancellation, because cancellation isn't instant or guaranteed, and out-of-order resolution is still possible.** Suppose request A (for "ca") is in flight, gets debounce-triggered to be superseded by request B (for "cat") which cancels A — but A's abort doesn't guarantee A's promise rejects before B's response arrives, and in some transport/proxy setups, a "cancelled" request can still resolve with a cached/already-computed response. The robust fix is orthogonal to cancellation: tag every request with a locally-incrementing sequence number when it's fired, and when a response arrives, only apply it to the UI if its sequence number matches the *latest* request that's been fired — any earlier-sequence response, whether cancelled or not, whether it resolves in whatever order, is simply discarded on arrival. This is the same fix used in [Race Condition in Fetch (Autocomplete Overwrite Bug)](../phase-03-react-debugging-scenarios/03-race-condition-fetch-autocomplete.md) — this system-design scenario is the "build the whole system with this baked in from the start" version of that debugging scenario.

**A client-side cache (prefix → results) exists to hide network latency, not to replace the network round trip.** Typing "café" one character at a time produces a sequence of prefixes ("c", "ca", "caf", "café") where each subsequent request's true, authoritative result depends on the live server/corpus — but a cheap, useful optimization is caching each prefix's last-seen server response locally (in memory for the session, or in `IndexedDB`/`localStorage` across sessions for very common queries), and instantly rendering the cached superset for a prefix the user has already queried before doing anything else, while a fresh network request is still fired and reconciles the list once it resolves. This makes backspacing (re-typing a prefix already seen this session) feel instantaneous, and makes forward-typing feel faster because a plausible-if-slightly-stale list appears before the network round trip completes — as long as the UI never presents cached results as final/authoritative without an in-flight reconciliation happening behind them.

**IME composition needs to be treated as a first-class input state, not an edge case bolted on.** For CJK and other composed-input scripts, the browser fires a sequence of `input` events for intermediate, not-yet-finalized characters while a user is composing a syllable/character via their input method, bracketed by `compositionstart` and `compositionend` events. Querying on every intermediate `input` event during composition sends a stream of garbage partial-character requests and burns through the debounce window on noise that was never a real query the user intended — the correct behavior is to suppress query-firing entirely between `compositionstart` and `compositionend`, and only treat the input as a completed, query-worthy value once composition ends (or the user isn't composing at all, for scripts without this mechanism).

**Merging multiple suggestion sources into one ranked list is a real design problem, not a rendering afterthought.** If suggestions blend personal search history, trending queries, and live server matches, naively rendering three separately-labeled sections (which is what an unstructured approach tends to produce) is a weaker answer than merging them behind a single ranking/dedup step — e.g., personal history for an exact-prefix match ranks above a generic trending query, and if the same string appears in both trending and server results, it's shown once, annotated with whichever context is most relevant (or the highest-priority source wins). This merge step is naturally where source-specific latency also gets reconciled — personal history (often local/cached, near-instant) can render first, with server results merging in as they arrive, rather than the whole dropdown waiting on the slowest source.

## Solution

**Core input-handling hook** — debounce, cancellation, sequence-guarding, and composition-awareness together:

```tsx
function useAutocomplete(query: string) {
  const [results, setResults] = useState<Suggestion[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const seqRef = useRef(0);
  const abortRef = useRef<AbortController | null>(null);
  const cacheRef = useRef(new Map<string, Suggestion[]>());

  const debouncedQuery = useDebouncedValue(query, 200);

  useEffect(() => {
    if (debouncedQuery.length < MIN_QUERY_LENGTH) {
      setResults([]);
      return;
    }

    // Instant cached render while the authoritative request is in flight.
    const cached = cacheRef.current.get(debouncedQuery);
    if (cached) setResults(cached);

    abortRef.current?.abort();
    const controller = new AbortController();
    abortRef.current = controller;
    const mySeq = ++seqRef.current;

    setIsLoading(true);
    fetchSuggestions(debouncedQuery, controller.signal)
      .then((data) => {
        if (mySeq !== seqRef.current) return; // superseded — discard regardless of arrival order
        cacheRef.current.set(debouncedQuery, data);
        setResults(data);
      })
      .catch((err) => {
        if (err.name !== 'AbortError') reportError(err);
      })
      .finally(() => {
        if (mySeq === seqRef.current) setIsLoading(false);
      });
  }, [debouncedQuery]);

  return { results, isLoading };
}
```

**IME composition guard**, applied at the input layer before anything reaches the debounced query state:

```tsx
function SearchInput({ onQueryChange }: { onQueryChange: (v: string) => void }) {
  const [value, setValue] = useState('');
  const isComposing = useRef(false);

  return (
    <input
      value={value}
      onChange={(e) => {
        setValue(e.target.value);
        if (!isComposing.current) onQueryChange(e.target.value);
      }}
      onCompositionStart={() => { isComposing.current = true; }}
      onCompositionEnd={(e) => {
        isComposing.current = false;
        onQueryChange((e.target as HTMLInputElement).value); // fire once composition finalizes
      }}
    />
  );
}
```

**Merging multiple ranked sources into one list:**

```ts
function mergeSuggestions(
  personal: Suggestion[],
  trending: Suggestion[],
  server: Suggestion[]
): Suggestion[] {
  const bySignature = new Map<string, Suggestion>();
  // Priority order: personal history > server match > trending —
  // later sources in this loop only fill in if not already present.
  for (const source of [personal, server, trending]) {
    for (const s of source) {
      if (!bySignature.has(s.text)) bySignature.set(s.text, s);
    }
  }
  return Array.from(bySignature.values());
}
```

> **Check yourself:** Explain, without looking above, why `AbortController` cancellation alone is not sufficient to prevent a stale response from overwriting the UI, and what the sequence-number check adds that cancellation doesn't guarantee.

## System Architecture

```
Keystroke → SearchInput (composition-aware)
          → debounce (~200ms)
          → check local cache for this prefix → render instantly if present (non-authoritative)
          → fire network request (AbortController-tracked, sequence-tagged)
          → response arrives → sequence check → merge with personal/trending sources → dedup/rank
          → render dropdown → keyboard nav (↑/↓/Enter/Esc) → selection → telemetry event
```

The cache and the network request are not mutually exclusive stages — they run concurrently, with the cache providing an immediate, possibly-stale render and the network request being what's actually authoritative once it resolves. This is the same "optimistic-then-reconcile" shape used for optimistic UI updates elsewhere in the app, applied to a read path instead of a write path.

## Scaling Considerations

**Request volume at scale is a real cost, not just a UX nicety** — a product with a large active user base firing one request per keystroke without debouncing multiplies request volume by average word length; debouncing plus a minimum query length (e.g., no query fires below 2–3 characters) meaningfully cuts backend load, and should be treated as a backend-cost concern the frontend is responsible for controlling, not purely a "make typing feel smooth" concern.

**The prefix cache should have a bounded size and simple eviction (LRU), not grow unbounded for the session** — a long, exploratory search session can generate many distinct prefixes; capping the cache (a few hundred entries is typically ample for a single session) with LRU eviction keeps memory bounded without materially hurting the hit rate for the actually-common case (backspacing/re-typing a recent prefix).

**Highlighting matched substrings needs to be script-aware, not a naive `indexOf`/character-index slice** — for scripts with multi-codepoint graphemes (emoji, some accented characters, combining marks), naive string index slicing can split a grapheme cluster in half, corrupting the rendered text; using a grapheme-aware segmentation (e.g., `Intl.Segmenter`) for highlighting is the correct-by-construction approach rather than assuming one JS string index equals one visual character.

**Analytics/telemetry on selection should be fire-and-forget and never block the navigation/selection action itself** — logging "user selected suggestion X for query Y" should not be awaited before acting on the selection; it's sent via `navigator.sendBeacon` or a non-blocking fetch so a slow or failed analytics call never delays or breaks the actual user-facing action.

## Gotchas

**Debouncing without cancellation, believing it "solves" the race condition.** Debouncing reduces *how often* requests fire; it does nothing about the case where a fired request is slow and a subsequent (also debounce-cleared) request resolves first — the classic overwrite bug is still fully possible with debouncing alone and no sequence/cancellation guard.

**Trusting `AbortController` cancellation as sufficient on its own, without a sequence-number check.** Aborting is a best-effort signal to stop unneeded work; it isn't a hard guarantee that a cancelled request's `.then()` never runs or that its data can't still arrive — the sequence check is the actual correctness guarantee, and cancellation is the resource-saving optimization layered on top of it.

**Firing queries during IME composition.** Easy to miss entirely if development and testing only happen in a language without composed input — this silently produces a broken (noisy, laggy, occasionally wrong) experience specifically for CJK and similar-script users, a whole user segment, not a rare edge case.

**Presenting cached prefix results as if they were final, with no visual indication a fresher network result may still be reconciling.** If the cache and network can disagree (a genuinely dynamic corpus, e.g., real-time inventory or trending queries), showing zero loading indication risks the user acting on stale data that a moment later silently changes underneath them — some lightweight in-progress indicator (even subtle) keeps this honest.

**Naively concatenating multiple suggestion sources instead of merging/deduping them.** Produces visible duplicate entries or an incoherent list ordering that reads as unpolished — merging behind one ranking/dedup step is what separates a "senior system design" answer from "I fetched three things and rendered three lists."

## Follow-up Questions

**Q (High): Walk through exactly why `AbortController.abort()` on the previous request is not, by itself, a complete fix for the stale-response race condition.**

Answer: `abort()` signals the browser (and, if the server/infrastructure respects the signal, the server) to stop processing that specific request, and causes the associated `fetch` promise to reject with an `AbortError` in the common case — which is genuinely useful for saving wasted work. But it's a best-effort signal, not an atomicity guarantee: depending on timing, proxies, caching layers, or how far along the request already was, an aborted request's response can still arrive and resolve rather than reject, or a race can exist between the abort taking effect and the response already being in transit. A monotonic sequence number checked at the moment a response is about to be applied to the UI is the actual correctness guarantee — it doesn't matter whether the earlier request was aborted, resolved, or errored; if its sequence number isn't the current latest, its result is discarded, full stop, regardless of the exact mechanics of how or whether the abort "worked."

The trap: treating `AbortController` as if it were a transactional guarantee ("the aborted request can never affect the UI") rather than a best-effort optimization — the distinction matters because relying on abort alone leaves a residual, hard-to-reproduce race condition that only shows up under specific timing/network conditions, exactly the kind of bug that's hard to catch in testing and shows up as a rare production report.

---

**Q (High): How would you design the system so that typing "caf" then quickly backspacing to "ca" then retyping "café" feels instantaneous, without waiting on three separate network round trips?**

Answer: This is precisely what the prefix cache is for — each of "ca", "caf", and (once typed) "café" gets cached by its own key the first time each is queried; backspacing back to an already-cached prefix ("ca") renders instantly from cache with no network request needed at all (assuming the debounce/minimum-length logic still allows re-querying it fresh in the background if freshness matters, but the *render* doesn't wait on that). Retyping forward to "café" a second time similarly hits the cache instantly if it was already fetched moments earlier. The key design point is that the cache key is the exact prefix string, and lookups are checked before any network request is fired, so previously-seen prefixes in the same session pay zero network latency on re-encounter.

The trap: only caching the *final* query's result rather than every intermediate prefix along the way — that only helps if the user retypes the exact same final string, and does nothing for the much more common backspace-and-adjust pattern, which is the actual scenario being tested here.

---

**Q (High): The interviewer says: "Assume the corpus can change in near-real-time (e.g., live inventory levels affecting search results). Does the prefix cache become a liability?"**

Answer: Yes, in a specific way — an aggressively long-lived cache can serve a stale result for a prefix whose *true* answer has since changed, and if the UI treats cached results as final rather than provisional, a user could act on stale information (e.g., see "in stock" for an item now out of stock). The fix isn't to abandon the cache, but to treat it strictly as a "render something instantly while the authoritative request reconciles" mechanism — always still firing the live network request in parallel/behind the cached render (never skipping the network call just because a cache hit existed) and giving the cache a short TTL appropriate to how fast the underlying data actually changes, rather than caching for the full session unconditionally.

The trap: either dropping the cache entirely "to be safe" (losing the real latency-hiding benefit for the common, non-time-sensitive case) or keeping it exactly as designed for a static corpus without adjusting TTL/reconciliation behavior for a dynamic one — the correct answer adapts the caching strategy to the data's actual volatility rather than picking one extreme.

---

**Q (Medium): How should keyboard navigation (arrow keys, Enter, Escape) interact with the async, potentially-still-loading suggestion list?**

Answer: Arrow-key highlighting should move through whatever's *currently rendered* (which may be the cached, non-authoritative list while a network request is still resolving) — the highlighted index is just an index into the current results array, and if that array is replaced by a fresher (reconciled) result set mid-navigation, the highlighted index should be reset or clamped to the new array's bounds rather than pointing at a now-different item at the same index. Enter selects whatever's currently highlighted at the moment it's pressed (using whatever data is current at that instant, cached or authoritative), and Escape closes the dropdown and, typically, should also cancel any in-flight request for the now-abandoned interaction (tying back into the same `AbortController` mechanism used for supersession).

The trap: keeping a stale highlighted index pointing at a stale array position after the results array is replaced — if the new array is shorter, or reordered, the previously-highlighted position may now silently correspond to a completely different (or nonexistent) suggestion, which is a subtle bug that only shows up when results actually change while a user is mid-navigation.

---

**Q (Medium): How would you extend this design to log which suggestion a user ultimately selects, in a way that could eventually improve ranking — without that logging path affecting the perceived responsiveness of selection itself?**

Answer: The selection action (navigating to the chosen result, closing the dropdown, updating the input) should happen immediately and be entirely independent of whether or how the analytics event is sent — the telemetry call (`{ query, selectedSuggestion, position, source, timestamp }`) is fired via `navigator.sendBeacon` (designed exactly for "fire this on the way out, don't block anything on its completion or success") or a non-awaited `fetch`, so a slow, failed, or even entirely dropped analytics call has zero effect on the user-facing selection flow. Architecturally, this keeps "act on the user's selection" and "record telemetry about the selection" as two independent, unordered side effects of the same event, rather than a pipeline where one depends on the other completing.

The trap: awaiting the analytics call (even briefly) before completing the selection/navigation action — this couples user-facing responsiveness to an entirely separate concern (whether the logging backend is fast or even reachable right now), which is exactly the kind of coupling a senior design should avoid on principle, not just as an optimization.

---

**Q (Low): If this needs to support right-to-left languages, what actually changes beyond mirroring the layout?**

Answer: Layout mirroring (the dropdown, icons, and text alignment flipping for RTL locales, generally handled by `dir="rtl"` and logical CSS properties rather than hardcoded `left`/`right`) is the visible part, but substring highlighting logic also needs care — if highlighting is implemented via character-index slicing assuming left-to-right visual order, it needs to instead operate on the underlying logical string (which doesn't change order internally for RTL — RTL is a rendering/bidi concern, not a change to string indexing) and let the browser's bidi algorithm handle visual presentation, rather than the application trying to manually reverse anything itself.

The trap: assuming RTL support means manually reversing strings or indices in application code — the browser's bidi rendering already handles visual order correctly given properly-directioned markup and unmodified logical string content; manual reversal in JS is both unnecessary and actively wrong.

---

## Self-Assessment

- [ ] Can explain why debouncing alone doesn't prevent the stale-response race, and why cancellation alone doesn't either
- [ ] Can describe the sequence-number guard and why it's the actual correctness mechanism, independent of abort/cancellation
- [ ] Can design a prefix cache that hides latency without presenting stale data as authoritative, and explain when it needs a TTL
- [ ] Can explain why IME composition needs special handling and what breaks without it
- [ ] Can design a merge/dedup step for combining multiple ranked suggestion sources into one coherent list
- [ ] Can reason about keyboard-navigation state staying consistent as the underlying results array changes asynchronously

---
*Next: Design an E-commerce Product Listing + Filters Page — shifts from a single fast-moving input stream to a page with many simultaneous, interacting pieces of state (filters, sort, pagination, URL sync), where the central question becomes "where does each piece of state live and how does it stay in sync with the URL."*
