# LRU Cache

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| O(1) get/put | `Map` (insertion-order-preserving) instead of a plain object or array scan | A linear scan to find the least-recently-used item on every eviction turns an O(1) cache into O(n) per operation — defeats the point of caching |
| "Used" = re-inserted | On `get`/`put` of an existing key, delete then re-set it | `Map` iterates in insertion order; re-inserting moves a key to the "most recently used" end without a separate linked list |
| Eviction on overflow | On `put`, if at capacity and key is new, delete the *first* key in iteration order | The first key in a `Map`'s iteration order is, by construction, the least recently touched — that's the eviction target |
| Capacity is fixed at construction | Passed once, validated (`> 0`), immutable afterward | An LRU cache's entire purpose is bounding memory; a cache that can silently grow past its stated limit isn't doing its job |

## The Scenario

"Implement an `LRUCache` class with a fixed capacity, supporting `get(key)` and `put(key, value)` in O(1) time. When the cache is full and a new key is inserted, evict the least recently used entry. Both `get` and `put` count as 'using' a key."

## Clarifying Questions

- **Does `get` on a missing key return `undefined`, `-1`, or throw?** This is a real API design choice with precedent both ways — LeetCode's classic version returns `-1` (implying integer values only), but a general-purpose cache used with arbitrary value types can't safely use a sentinel like `-1` (a cached value might legitimately *be* `-1`). I'd default to `undefined` for a general-purpose cache, and confirm if this is meant to match a specific existing interface (e.g., mimicking `Map`'s `get` semantics).
- **What happens on `put` for a key that already exists?** Does it update the value and *also* count as a "use" (moving it to most-recently-used)? Yes, in every standard LRU definition — updating a value is itself an access, and should refresh its recency. I'd confirm this rather than assume, since a subtly wrong implementation ("update value but don't move it") only breaks under access patterns that hit that exact case.
- **What's the capacity, and what happens if it's 0 or negative?** A capacity of 0 is a degenerate case worth explicitly deciding (probably: cache holds nothing, every `put` immediately gets evicted — or, arguably, throw a construction error since a zero-capacity cache is likely a caller bug). I'd validate at construction time rather than let it produce confusing behavior later.
- **Does this need to be thread-safe / concurrency-safe?** JavaScript is single-threaded for synchronous code, so this doesn't apply the way it would in Java/Go — but I'd flag it briefly to show awareness, and note that if this cache were later used to memoize *async* operations (caching in-flight promises), a different set of race conditions around concurrent `get`s for the same missing key would need consideration (see the memoization-with-TTL scenario).

## Approach & Trade-offs

The naming "LRU" already tells you the eviction policy — the interesting part is achieving O(1) for both `get` and `put`, which is where most naive-but-plausible implementations fail.

**Why not a plain object + separate timestamp tracking?** A common first instinct is `{ [key]: { value, lastUsed: Date.now() } }`, evicting by scanning for the minimum `lastUsed` on overflow. This works but is O(n) per eviction (must scan every entry to find the minimum), and `Date.now()` has limited resolution — two accesses in the same millisecond tie, ambiguating true recency order. This is the naive-but-plausible trap: it looks correct and passes small tests, but doesn't meet the O(1) requirement and has a subtle correctness gap under high-frequency access.

**Why a `Map` alone is enough — no manual doubly linked list needed.** In LeetCode-style solutions this problem is often solved with a hand-rolled doubly linked list + hash map for O(1) removal from arbitrary positions. In JavaScript specifically, this complexity is unnecessary: a native `Map` already preserves *insertion order* during iteration, and critically, `Map.prototype.delete` followed by `Map.prototype.set` for the same key moves that key to the *end* of the iteration order (re-inserting it). That's exactly the "move to most-recently-used position" operation a linked list would otherwise be needed for — and `Map.prototype.get`/`set`/`delete` are all O(1) (amortized, hash-table-backed) regardless of map size. So the "least recently used" entry is always whatever key currently comes *first* in the map's iteration order, retrievable via `map.keys().next().value`.

**The core operations**:
1. `get(key)`: if absent, return `undefined` (or the agreed sentinel). If present, delete-then-reinsert to mark it most-recently-used, then return the value.
2. `put(key, value)`: if the key already exists, delete it first (so the re-insertion below correctly moves it to the end rather than leaving a stale position) — then, if at capacity and this is a genuinely *new* key, evict the first (least-recently-used) key before inserting. Finally, set the key/value, which places it at the most-recently-used end.

I chose to implement this as a class wrapping a private `Map` (via `#cache`) rather than subclassing `Map` directly — subclassing `Map` and overriding `get`/`set` is tempting but risks subtle bugs if any native `Map` method that isn't overridden (like `forEach` or the iterator protocol) is called and doesn't go through the LRU-aware logic; composition keeps the LRU semantics contained to exactly the two methods that need them.

## Solution

```javascript
class LRUCache {
  #capacity;
  #cache = new Map();

  constructor(capacity) {
    if (!Number.isInteger(capacity) || capacity <= 0) {
      throw new RangeError('Capacity must be a positive integer');
    }
    this.#capacity = capacity;
  }

  get(key) {
    if (!this.#cache.has(key)) return undefined;

    // Accessing a key counts as "using" it — move it to the most-recently-used
    // end by deleting and re-inserting (Map preserves insertion order).
    const value = this.#cache.get(key);
    this.#cache.delete(key);
    this.#cache.set(key, value);
    return value;
  }

  put(key, value) {
    if (this.#cache.has(key)) {
      // Existing key: remove first so the re-insertion below places it
      // correctly at the most-recently-used end, not at its old position.
      this.#cache.delete(key);
    } else if (this.#cache.size >= this.#capacity) {
      // New key, cache full: evict the least-recently-used entry —
      // by construction, that's the first key in iteration order.
      const lruKey = this.#cache.keys().next().value;
      this.#cache.delete(lruKey);
    }

    this.#cache.set(key, value);
  }

  get size() {
    return this.#cache.size;
  }
}
```

```javascript
const cache = new LRUCache(2);

cache.put(1, 'a');
cache.put(2, 'b');
cache.get(1);          // 'a' — and 1 is now most-recently-used
cache.put(3, 'c');      // capacity full, 2 is LRU (1 was just touched) → evicts key 2
cache.get(2);           // undefined — was evicted
cache.put(4, 'd');      // 1 is now LRU (3 was inserted after) → evicts key 1
cache.get(1);           // undefined — was evicted
cache.get(3);           // 'c'
cache.get(4);           // 'd'
```

> **Check yourself:** Why does `put` on an *existing* key need to `delete` it before re-`set`ting, even though `Map.prototype.set` on an existing key updates its value in place without an explicit delete?

## Why `Map` Solves This Without a Manual Linked List

This is worth dwelling on because it's the detail that distinguishes "knows the LRU algorithm" from "knows this specific language's tools well enough to implement it cleanly." The textbook LRU cache design (in a language without an order-preserving hash map) is a hash map for O(1) lookup **plus** a doubly linked list for O(1) removal-from-middle and O(1) move-to-end — the hash map stores `key → node reference`, and the linked list maintains actual recency order, because a plain hash map's iteration order is unspecified or insertion-order-only-in-some-languages.

JavaScript's `Map` specifically *guarantees* insertion-order iteration (per spec, unlike, say, older versions of Python's `dict` or Java's `HashMap`), and its `delete`+`set` combo already gives O(1) "move this key to the most-recently-used position." That collapses the two-data-structure design into one. This is genuinely language-specific — the same clean solution isn't available in a language whose native hash map doesn't preserve insertion order, which is exactly why the canonical LeetCode solution (language-agnostic) teaches the linked-list approach — it's worth explicitly naming this as "here's the general algorithm, and here's the JS-specific simplification" rather than presenting the `Map`-based version as if it were the only or most fundamental approach.

## Gotchas

**Using a plain object instead of `Map`.** Plain objects (`{}`) don't guarantee key iteration order for all key types the way `Map` does (integer-like string keys get sorted numerically ahead of insertion order, per spec) — this makes "first key = least recently used" unreliable the moment numeric-looking keys are involved, a subtle trap since it works fine with string keys like `'a'`, `'b'` but silently breaks with keys like `'1'`, `'2'`.

**Forgetting that `put` on an existing key must also refresh recency.** A naive `put` that only checks `if (!cache.has(key))` before evicting, and otherwise does a plain `cache.set(key, value)` without first deleting, ends up *not* moving the key to the most-recently-used position (because `Map.set` on an existing key updates the value in place but does **not** change its position in iteration order) — this passes tests where every key is only put once, but silently gives wrong eviction order the moment a key is updated more than once.

**Not counting `get` as a "use."** Some incorrect implementations only reorder on `put`, treating `get` as a read-only operation that doesn't affect eviction order — but the entire point of "least recently *used*" (not "least recently *inserted*") is that reads count too; skipping this makes the cache behave like a FIFO queue with a capacity limit, not an actual LRU cache, and this bug is easy to miss because it only manifests under access patterns where a `get` is what should have "saved" a key from eviction.

**Off-by-one on the eviction check.** Checking `size > capacity` instead of `size >= capacity` before inserting a *new* key evicts one entry too late — by the time the check runs, the new key hasn't been inserted yet, so the correct condition for "is the cache full and about to overflow" is `size >= capacity`, not `size > capacity`.

**Assuming `Map.prototype.delete` + `.set` is expensive.** It's tempting to think re-inserting on every `get` is wasteful, but both operations are amortized O(1) on V8 and other engines' `Map` implementations (hash-table-backed) — this is not a performance concern worth "optimizing away" with a more complex data structure, and reaching for a hand-rolled doubly linked list in JavaScript specifically usually adds complexity without a real performance win.

## Follow-up Questions

**Q (High): Why does this problem require O(1) `get` and `put`, and how does JavaScript's `Map` achieve that without a manual doubly linked list?**

Answer: An LRU cache's value proposition is being fast — if `get`/`put` were O(n), you could just as well use an unordered array and linearly scan for both lookup and eviction, and the "cache" would provide no meaningful speed advantage over just doing the underlying expensive operation directly in many cases. O(1) is achieved because `Map` is hash-table-backed (O(1) average-case `get`/`set`/`delete` by key, same as a plain object) **and** because `Map` specifically preserves insertion order during iteration (guaranteed by spec) — so "move this key to the most-recently-used position" reduces to `delete` then `set` (two O(1) hash operations), and "find the least-recently-used key" reduces to reading the first entry in iteration order (O(1) via `.keys().next().value`, since `Map`'s iterator gives entries in that guaranteed order without needing to scan). No pointer manipulation is needed because the `Map`'s internal ordering *is* the linked structure a hand-rolled solution would otherwise build explicitly.

The trap: describing the classic hash-map-plus-doubly-linked-list design as if it's required in every language, without recognizing that JavaScript's `Map` already provides the ordering guarantee that the linked list exists to provide in languages that lack it — missing this shows unfamiliarity with what `Map` actually guarantees versus a plain object.

---

**Q (High): What's the difference between LRU (Least Recently Used) and LFU (Least Frequently Used) eviction, and when would you prefer one over the other?**

Answer: LRU evicts based on *recency* — the item that hasn't been touched for the longest time goes first, regardless of how often it was used historically. This works well when access patterns have temporal locality (recently accessed items are likely to be accessed again soon), which describes most real caching scenarios (recently viewed products, recently opened files, recent API responses). LFU evicts based on *frequency* — the item accessed the fewest total times goes first, regardless of when those accesses happened. LFU can outperform LRU when there's a stable set of "hot" items that are used constantly but with occasional long gaps between individual accesses (LRU would incorrectly evict a hot-but-momentarily-idle item), but LFU has its own failure mode: an item that was accessed heavily once, long ago, can accumulate a frequency count high enough to never get evicted even though it's now cold ("cache pollution" from a one-time burst) — mitigated in practice with frequency decay over time. LFU is also more expensive to implement correctly in O(1) (requires tracking both frequency and, within a frequency tier, recency, for tie-breaking).

The trap: treating LRU as universally "the right" eviction policy — it's a strong general default for temporal-locality workloads, but conflating it with "the only sensible caching strategy" misses that the right policy is workload-dependent, which is precisely the kind of trade-off framing that signals seniority.

---

**Q (Medium): How would you add TTL (time-to-live) expiration on top of this LRU cache, and what new complexity does that introduce?**

Answer: Each entry needs an associated expiry timestamp (`Date.now() + ttl`) stored alongside the value — e.g., `Map<key, { value, expiresAt }>`. The tricky part is *when* expiration is checked: lazy expiration (check `expiresAt` only when a key is accessed via `get`, and delete-and-treat-as-miss if expired) is simple and avoids background work, but means expired entries can linger in memory (and count against capacity) until someone happens to access them. Active expiration (a periodic sweep via `setInterval`, or a min-heap keyed by expiry time to always know the next entry to expire) keeps memory tighter but adds real complexity and a background timer to manage (including cleaning it up to avoid leaks). Combining TTL with LRU also raises a design question worth surfacing explicitly: does an expired-but-not-yet-evicted entry still count toward the LRU capacity limit, and does checking (but not finding-expired) an entry still count as "using" it for recency purposes? Most implementations say no to both — an expired entry is treated as already gone.

The trap: bolting on TTL by only checking expiry in `get` without ever reclaiming space from expired entries proactively — under a workload with many put-once-never-read entries, expired garbage can fill the entire cache capacity, evicting live, frequently-used entries to make room for entries nobody is actually re-fetching, silently degrading hit rate.

---

**Q (Medium): How would you make this LRU cache emit an eviction event or hook, e.g., for cache-hit/miss metrics or cleanup of evicted resources (like revoking an object URL)?**

Answer: Add an optional `onEvict(key, value)` callback (or an `EventEmitter`-style `.on('evict', ...)` interface, connecting this to the pub/sub and event emitter scenarios) invoked at the exact point an entry is removed due to capacity overflow — importantly, *only* on capacity-driven eviction, not on every `delete` (e.g., if the cache also exposes an explicit `.delete(key)` for manual removal, that's a different semantic event than "evicted because full"). This is genuinely useful in practice: if cached values hold resources that need explicit cleanup (revoking a `URL.createObjectURL` blob URL, closing a WebSocket, cancelling an in-flight request tied to that cache entry), failing to hook eviction means those resources leak silently every time the cache evicts something, since nothing else in the system is watching for it.

The trap: assuming garbage collection alone handles cleanup — GC reclaims JS-managed memory when the value becomes unreachable, but it does *not* know to call `.close()`, `.revokeObjectURL()`, or cancel a subscription; any external-resource cleanup that a cached value's lifecycle implies must be handled explicitly at the point of eviction, not left to GC.

---

**Q (Low): How would this design change for a distributed cache (e.g., Redis-backed) shared across multiple server instances, versus this single-process, single-map implementation?**

Answer: The single biggest change is that "read, check capacity, evict, write" is no longer atomic across concurrent callers — multiple processes/requests could race on the same key or on triggering eviction simultaneously, requiring either the distributed store's own atomic primitives (Redis's `EXPIRE`, its native LRU/LFU eviction policies configurable per-instance, or Lua scripts for compound check-then-act operations) rather than a JS-level `Map`. Additionally, recency tracking that's trivial in-process (relying on `Map`'s iteration order) doesn't exist as a primitive in most distributed stores — Redis, for instance, implements its own approximate LRU (sampling a subset of keys rather than tracking exact global recency, for performance reasons at scale) rather than exact LRU, which is a real, intentional trade-off worth knowing exists.

The trap: assuming a distributed cache is "the same algorithm, just further away" — the concurrency and atomicity concerns, and the fact that production systems often use *approximate* rather than exact LRU for performance, are qualitatively different problems from the single-process version, and glossing over that gap in a system design context is a common tell of only having solved the LeetCode-style version.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement an O(1) `get`/`put` LRU cache using `Map` from memory, without a manual linked list
- [ ] Can explain exactly why `Map`'s insertion-order guarantee eliminates the need for a doubly linked list in JavaScript specifically
- [ ] Can state the LRU vs. LFU trade-off and when each is preferable
- [ ] Can identify all five gotchas above without re-reading them
- [ ] Can sketch how TTL expiration would layer on top of this design and name the lazy-vs-active expiration trade-off

---
*Next: Concurrency-limited Task Queue (Promise Pool) — another bounded-resource problem, this time bounding simultaneous in-flight async operations rather than cached memory.*
