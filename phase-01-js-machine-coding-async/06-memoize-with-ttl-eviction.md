# Memoize With TTL & Cache Eviction

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Cache key | Stringify (or otherwise normalize) the arguments into a lookup key | Object/array arguments aren't usable as `Map` keys by value — `{a:1}` never equals another `{a:1}` by reference |
| TTL (time-to-live) | Store `{value, expiresAt}`; check `Date.now() < expiresAt` on read | Without it, a memoized "get current user" call would cache forever, serving stale data indefinitely |
| LRU eviction | A `Map` re-inserts the accessed key to the end on every read, so the *first* key is always the least-recently-used one | Caps memory use — without a bound, a memoized function called with ever-changing arguments grows the cache forever |
| Async memoization | Cache the **pending Promise**, not just the resolved value | Prevents duplicate concurrent network calls for the same in-flight argument (the "thundering herd" / dogpile problem) |

## The Scenario

"We have an expensive `getUserProfile(userId)` function that hits our API — it gets called from multiple components on the same page, often with the same `userId`, within milliseconds of each other. Write a `memoize` wrapper for it. It needs a TTL so we don't serve stale profiles forever, and it needs a max cache size with LRU eviction so it doesn't grow unbounded if we memoize something with high-cardinality arguments."

## Clarifying Questions

- **`getUserProfile` returns a Promise — does memoization need to prevent duplicate *in-flight* calls, or only avoid re-calling for already-resolved results?** This is the question that changes the whole design. The scenario explicitly says "called from multiple components... within milliseconds of each other" — that's describing calls that overlap *before* the first one has resolved, so simply caching resolved values isn't enough; the cache needs to store and return the **pending promise itself** so concurrent callers share one in-flight request instead of firing duplicates.
- **What should happen if the cached promise rejects?** A rejected promise shouldn't stay cached and keep returning the same error forever — I'd evict a rejected entry immediately so the next call gets a fresh attempt, rather than caching failures with the same TTL as successes.
- **How should the cache key be derived for a single primitive argument like `userId`?** Straightforward — the primitive value itself (or `String(userId)`) works directly as a `Map` key. I'm flagging this because the *general* version of this problem (multiple, possibly-object arguments) needs a real answer, but this specific case is simple.
- **Is `maxSize` a hard cap that evicts on insert, or a target the cache "usually" stays under?** Hard cap — every insert past the limit evicts exactly one entry (the least-recently-used) before or as part of adding the new one, to keep a predictable memory ceiling.

## Approach & Trade-offs

**Keying:** for this specific scenario (single primitive `userId` argument), the key is just the argument itself. For the general case of arbitrary arguments, `JSON.stringify(args)` is the common pragmatic choice — it's not perfect (see Gotchas), but it's good enough for the vast majority of real memoization needs and dramatically simpler than a structural-equality-based cache lookup.

**LRU via `Map`'s insertion order:** JavaScript's `Map` iterates keys in insertion order, and — critically — re-inserting an existing key (`map.delete(key); map.set(key, value)`) moves it to the *end* of that iteration order. This means: on every cache *read* that hits, delete-and-re-set the entry to mark it "recently used," and on every cache *write* that would exceed `maxSize`, evict `map.keys().next().value` — the first key in iteration order is, by construction, the one that's gone the longest without being touched. This gives O(1) LRU semantics without a separate doubly-linked-list structure, which is the classic textbook LRU implementation — the `Map` *is* the linked list, using its native ordering guarantee.

**TTL, checked lazily on read:** rather than a background timer sweeping expired entries proactively (which costs CPU even when the cache isn't being read), each entry stores its own `expiresAt` timestamp, and every `get` checks it before returning — an expired entry is deleted and treated as a cache miss. This trades a small amount of "wasted" memory (an expired-but-unread entry sits in the cache until something happens to read or evict it) for zero background CPU cost, which is the right trade-off for a UI-layer cache where entries are typically read again soon if they matter at all.

**Async-safe memoization — caching the Promise, not the resolved value:** the cache stores whatever `fn(...)` returns *immediately*, synchronously, before it resolves — which, for an async function, is the Promise object itself. A second call arriving while the first is still pending finds that same Promise already in the cache and returns it directly, so both callers are `await`-ing the exact same underlying network request rather than triggering two. On rejection, the entry is removed from the cache so the next call gets a clean retry instead of replaying the same failure.

## Solution

```javascript
function memoize(fn, { ttl = Infinity, maxSize = Infinity } = {}) {
  const cache = new Map(); // key -> { value, expiresAt }

  function getKey(args) {
    // Simple, pragmatic default. See Gotchas for its limitations.
    return JSON.stringify(args);
  }

  return function memoized(...args) {
    const key = getKey(args);
    const now = Date.now();

    if (cache.has(key)) {
      const entry = cache.get(key);
      if (entry.expiresAt > now) {
        // Cache hit — mark as most-recently-used by re-inserting.
        cache.delete(key);
        cache.set(key, entry);
        return entry.value;
      }
      cache.delete(key); // expired
    }

    const value = fn.apply(this, args); // may be a Promise — cached as-is, unresolved

    // If it's a promise, evict on rejection so failures aren't cached.
    if (value && typeof value.then === 'function') {
      value.catch(() => cache.delete(key));
    }

    cache.set(key, { value, expiresAt: now + ttl });

    if (cache.size > maxSize) {
      const oldestKey = cache.keys().next().value; // first key = least recently used
      cache.delete(oldestKey);
    }

    return value;
  };
}
```

```javascript
const getUserProfile = memoize(fetchUserProfileFromAPI, { ttl: 60_000, maxSize: 100 });

// Three components mount within the same tick, all requesting the same user:
const p1 = getUserProfile(42);
const p2 = getUserProfile(42);
const p3 = getUserProfile(42);

console.log(p1 === p2 && p2 === p3); // true — ONE network request, three callers share it
```

> **Check yourself:** Why does caching the *Promise itself*, synchronously, before it resolves, solve the "multiple components call this within milliseconds" problem — and why would caching only the *resolved value* (inside a `.then()`) fail to solve it?

## Gotchas

**`JSON.stringify` as a cache key has real limitations.** It's order-sensitive (`{a:1,b:2}` and `{b:2,a:1}` produce different keys despite being "the same" arguments semantically — mirroring the same issue from the deep-equal scenario), it silently drops `undefined`/functions from the key entirely, and it throws on circular references. For the scenario as given (a single primitive `userId`), none of this bites — but it's worth naming as a known limitation of the general-purpose version rather than presenting `JSON.stringify` as a universally correct keying strategy.

**Caching a pending promise, but forgetting to handle rejection.** If a rejected promise stays in the cache for its full TTL, every caller during that window gets the *same* rejected promise replayed — including callers whose retry logic assumes a fresh attempt is being made. The fix (evicting on `.catch()`) is easy to forget precisely because the success path "just works" without it, and the bug only shows up under failure conditions that are less likely to be hit during casual manual testing.

**LRU correctness via `Map` re-insertion is easy to get subtly wrong.** The critical, easy-to-miss detail is that a cache **hit** must also re-insert the entry (delete then set) to mark it as recently used — a naive implementation that only reorders on *write* (not on *read*) will evict entries that are actually being read frequently, just never re-written, which defeats the entire purpose of LRU (evict what's *unused*, not what's merely old).

**Memoizing impure functions produces wrong answers, not just wasted cache space.** If the wrapped function's result can legitimately differ for the same arguments over time for reasons *other* than the TTL boundary — e.g., `getUserProfile` after the user has updated their own name — memoization is deliberately serving stale data until the TTL expires. That's the correct, intended trade-off *here* (the scenario explicitly wants a TTL specifically to bound this staleness), but it's worth being explicit that memoization is fundamentally a staleness-vs-performance trade-off, not a free win.

**Unbounded cache growth without `maxSize`.** If `getUserProfile` is called with a wide, effectively-unbounded range of `userId`s (e.g., a support tool that looks up arbitrary users), a memoize without `maxSize` grows forever, holding every distinct user's profile in memory for the app's entire lifetime — a slow, easy-to-miss memory leak that only shows up after the app has been open a long time.

## Follow-up Questions

**Q (High): How do you generate a cache key for functions called with object/array arguments, and what are the limitations of `JSON.stringify` for this?**

Answer: `JSON.stringify(args)` serializes the arguments array into a string that can serve as a `Map`/object key, and it's the pragmatic default for most real memoization needs because it's simple and handles nested primitives/arrays/objects correctly for the *common* case. Its limitations: it's sensitive to key insertion order (two structurally-equivalent objects with keys in different orders produce different string keys, causing spurious cache misses), it silently omits `undefined` values and functions from the serialized output (two argument sets that differ only in an `undefined`-valued key would collide into the same cache key), it throws on circular references, and it can't distinguish some type differences that don't round-trip through JSON (e.g., a `Map` argument and a plain object with the same enumerable-looking shape). For most memoization of API calls keyed on IDs or simple filter objects, these limitations don't matter in practice; for a general-purpose memoize utility, they're worth naming as known, accepted trade-offs.

The trap: presenting `JSON.stringify` as if it were a correct-by-construction general solution rather than a pragmatic, limitation-bearing default — a senior answer names the specific ways it can produce spurious cache misses (or, worse, false collisions) rather than treating it as a solved problem.

---

**Q (High): How would you implement LRU eviction efficiently (O(1)) using just a `Map`?**

Answer: JavaScript's `Map` guarantees iteration in insertion order, and re-inserting a key that already exists (`delete` then `set`) moves it to the end of that order — this native behavior is exactly what an LRU cache needs, without building a separate doubly-linked-list-plus-hashmap structure by hand. The implementation: on every cache **read** that hits, `delete` and re-`set` the entry (moving it to "most recently used" position); on every **write** that would push the cache past `maxSize`, evict via `cache.keys().next().value` (the *first* key in iteration order, i.e., whichever key has gone longest without being read or re-written) followed by `cache.delete()` on that key. Both the read-side reordering and the write-side eviction are O(1) operations (`Map` operations are O(1) average case), matching the O(1) complexity of a hand-rolled linked-list-based LRU, but with dramatically less code.

The trap: forgetting the read-side reordering step and only reordering on write — this produces a cache that evicts based on *insertion* recency rather than *access* recency, which technically compiles and often "looks correct" on cache-miss-heavy test cases, but fails the actual LRU contract the moment a frequently-*read*, rarely-*rewritten* entry needs to survive eviction.

---

**Q (High): How do you handle TTL expiry — checking on write (proactive/background) vs. checking on read (lazy)? What's the trade-off?**

Answer: A **proactive** approach runs a background timer (or a scheduled sweep) that periodically scans the cache and deletes any entry whose `expiresAt` has passed, independent of whether anything is currently reading that entry — this keeps the cache's actual size closer to its "live" size at all times, at the cost of continuous background CPU work (and the operational complexity of managing that timer's lifecycle, including clearing it if the memoized function/cache is ever torn down, to avoid a dangling interval). A **lazy** approach (used in the solution above) does no background work at all — it simply checks `expiresAt` at the moment of a `get`, deleting the entry then if it's stale — trading a small amount of "wasted" memory (a stale entry can sit in the cache, taking up space, until something happens to read or evict it) for zero background CPU cost. For a UI-layer memoization cache — where entries that are never read again also don't matter (nothing depends on them being promptly reclaimed) — lazy expiry is almost always the better trade-off; proactive expiry earns its cost in systems where memory pressure from stale-but-unread entries is a real operational concern (e.g., a very-high-cardinality server-side cache).

The trap: presenting one approach as universally correct — the right answer names both, states the trade-off explicitly (CPU cost vs. memory-holding cost), and picks based on the actual deployment context rather than by default habit.

---

**Q (Medium): How do you memoize an async function so that concurrent calls with the same argument share one in-flight request instead of firing duplicate network calls?**

Answer: The critical move is caching what `fn(...)` returns **synchronously and immediately** — which, for an `async function` or any function returning a `Promise`, is the `Promise` object itself, *before* it has settled — rather than waiting for the promise to resolve and only then caching the resolved value inside a `.then()` callback. Because the cache is populated synchronously on the very first call, any subsequent call that arrives *before* that promise has settled (even microseconds later) finds the same pending `Promise` already in the cache and returns that same reference — so both callers end up `await`-ing (or `.then()`-ing) the identical underlying request, and the network layer only ever sees one actual call for that argument during that window. If you instead cached the resolved value only after `.then()` fires, every call that arrives *before* that `.then()` runs would still see a cache miss (since nothing's been stored yet) and would kick off its own independent, duplicate request — completely failing to solve the "multiple components call this within milliseconds" problem the scenario describes.

The trap: writing `cache.set(key, await fn(...))` (or an equivalent `.then()`-based caching) instead of `cache.set(key, fn(...))` — the `await`ed version delays the cache write until *after* resolution, reopening the exact race window the whole feature exists to close, and it's an easy mistake because it "looks" more correct (caching the "real," resolved value) while being functionally wrong for the concurrent-call use case.

---

**Q (Medium): What happens if a memoized async function's promise rejects — should the error be cached? Why or why not?**

Answer: No — a rejected promise generally should **not** stay cached for its full TTL, because doing so means every caller during that window receives the exact same failure, including callers whose surrounding logic expects a fresh attempt (e.g., a retry button, or a component that just remounted after the transient failure that caused the original rejection has since resolved itself — a flaky network blip, a momentarily-overloaded backend). The fix is attaching a `.catch()` to the cached promise that evicts its own cache entry on rejection, so the *next* call after a failure gets a genuinely fresh attempt rather than replaying the cached error. This does reopen the "duplicate concurrent calls" problem specifically for the failure case — if ten callers were all awaiting the same failed promise, all ten see the rejection (correctly, since they were genuinely waiting on the same request), but the *next* new call after that isn't held back by a stale cached failure.

The trap: leaving rejected promises cached with the same TTL as successes "for simplicity" — this silently converts a transient, likely-recoverable failure into a much longer-lived one from the caller's perspective, which is a materially worse failure mode than not memoizing at all.

---

**Q (Low): How does this memoize implementation interact with a function that captures mutable external state — why is memoization unsafe for impure functions?**

Answer: Memoization's entire correctness argument rests on the premise "given the same arguments, this function always produces the same result" (referential transparency) — if the wrapped function's behavior depends on anything *other* than its arguments (a module-level variable, the current time in a way not captured by TTL, `Math.random()`, DOM/global state), memoizing it means the *first* call's result gets served to every subsequent call with matching arguments, even after the underlying state that actually determines the "real" answer has changed — and this staleness isn't bounded or made visible the way TTL-based staleness is, since nothing about the cache "knows" the underlying state changed. This is a strictly worse failure mode than simply not memoizing, because it's silent — the function still returns *a* value, just possibly the wrong one, with no error or warning signal.

The trap: assuming "add a TTL" fully solves the impurity problem — TTL bounds *time-based* staleness (the answer might change over time even for identical inputs, in ways the cache doesn't know about) but does nothing for staleness caused by *other* mutable inputs that aren't reflected in the TTL clock at all — the two are genuinely different problems that happen to look similar.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement memoize with TTL and LRU eviction, using a `Map`'s insertion-order behavior, from memory
- [ ] Can explain why LRU eviction requires reordering on *read*, not just on write
- [ ] Can explain precisely why caching the Promise synchronously (not the resolved value) solves the concurrent-duplicate-call problem
- [ ] Can explain why a rejected promise should be evicted rather than cached for the full TTL
- [ ] Can name at least two concrete limitations of `JSON.stringify` as a cache key strategy
- [ ] Can explain why memoization is unsafe for impure functions, distinct from the TTL staleness trade-off

---
*Next: Custom EventEmitter (on/off/once/emit) — moving from caching function results to building a pub/sub-style listener registry, with its own closures-and-mutation-during-iteration gotchas.*
