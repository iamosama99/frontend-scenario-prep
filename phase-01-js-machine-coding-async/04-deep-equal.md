# Deep Equal

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Structural comparison | Recursively compare values instead of references | `{a:1} === {a:1}` is `false` with `===`; deep equal needs to say `true` |
| `NaN` handling | Use `Object.is`-style semantics for the primitive fast path | `NaN !== NaN` under `===`, but a testing/memoization utility usually needs `NaN` to equal `NaN` |
| Key-order independence | Compare by *set* of keys, not by iteration order | `{a:1,b:2}` and `{b:2,a:1}` must be equal — order is not semantically meaningful for objects |
| Circular reference safety | Track visited **pairs** (not single objects, since this compares two graphs) | Without it, two mutually-referencing structures cause infinite recursion, same failure mode as deep clone |

## The Scenario

"We need a `deepEqual(a, b)` utility — it's going into a testing helper and also being considered for a custom `useMemo`-style hook's dependency comparison. It needs to handle nested objects and arrays, and correctly deal with `NaN` — right now our naive version says `deepEqual(NaN, NaN)` is `false`, which is breaking a test."

## Clarifying Questions

- **Should `NaN` equal `NaN`?** The bug report says yes — this is asking for `Object.is`-like semantics for the primitive comparison, not raw `===`, specifically because `===` treats `NaN` as never equal to itself, which is almost never the semantically useful behavior for a *value* comparison utility (as opposed to `===`'s IEEE-754-correct-but-inconvenient behavior).
- **Should `+0` and `-0` be considered equal?** `Object.is` treats them as *distinct* (`Object.is(0, -0)` is `false`), but most deep-equal implementations (Jest's `toEqual`, Lodash's `isEqual`) treat them as *equal*, matching `==`/`===`'s behavior for this one case specifically. I'd confirm which behavior is wanted — for a "for testing" utility, matching Jest's convention (treat `+0`/`-0` as equal, but `NaN` as equal to itself) is the safer default since it matches what engineers already expect from `toEqual`.
- **Do keys with `undefined` values need special handling?** — e.g., is `{a: 1, b: undefined}` deeply equal to `{a: 1}`? This is a real edge case with disagreement between libraries; I'd default to "no, not equal" (`Object.keys` includes `b` in the first but not the second, so key sets differ) unless told otherwise, and flag it as a decision either way.
- **Does it need to handle circular references?** Since this shares infrastructure with the deep clone problem, I'd ask, but even if not explicitly required, a "seen pairs" guard is cheap insurance against a stack overflow on unexpected input (e.g., comparing two DOM-adjacent objects that happen to be self-referencing).

## Approach & Trade-offs

The comparison branches by type, mirroring deep clone's structure but comparing instead of copying:

1. **Reference/primitive fast path first**: if `a === b`, they're equal (covers identical primitives and identical object references immediately — no need to recurse into two references to the same object). But `===` alone gets `NaN` wrong, so the fast path actually needs `Object.is(a, b) || a === b` — using `Object.is` first correctly handles `NaN` (`Object.is(NaN, NaN)` is `true`), and layering `=== ` after it (or adjusting `Object.is`'s `+0`/`-0` behavior) depends on which `0` convention was chosen above.
2. **Type mismatch → not equal**: if `typeof a !== typeof b`, or one is an array and the other isn't, or their constructors differ meaningfully (e.g., a `Date` vs. a plain object), short-circuit to `false` immediately.
3. **Arrays**: compare `length` first (cheap early exit), then compare element-by-element at matching indices.
4. **Objects**: compare `Object.keys(a).length === Object.keys(b).length` first (cheap early exit that also catches the "`b` has an extra key" case), then for every key in `a`, confirm `b` has that key (`hasOwnProperty`, not just `b[key] !== undefined`, to correctly distinguish "key present with value `undefined`" from "key absent") and that the values are recursively equal.
5. **Circular references**: unlike deep clone (which tracks one object → its clone), equality has to track **pairs** of objects currently being compared, since a cycle only matters relative to *both* structures being walked in parallel — if the recursive call reaches the same `(a, b)` pair again, treat it as equal-so-far (the cycle "closes" identically on both sides) rather than recursing forever.

I chose to short-circuit on length/key-count mismatches before doing the more expensive per-element/per-key recursive comparison — this is a meaningful optimization for testing utilities that get called on large fixtures repeatedly, and it's a cheap thing to point out as evidence of thinking about the hot path, not just correctness.

## Solution

```javascript
function deepEqual(a, b, seen = new WeakMap()) {
  // Fast path: identical reference, or primitives that are strictly equal.
  // Object.is handles NaN === NaN correctly; we then also treat +0/-0 as equal
  // (matching common testing-library convention, unlike raw Object.is).
  if (Object.is(a, b)) return true;
  if (a === 0 && b === 0) return true; // +0 vs -0, both already failed Object.is above

  // If either isn't an object (or is null), and they weren't caught above, they differ.
  if (typeof a !== 'object' || a === null || typeof b !== 'object' || b === null) {
    return false;
  }

  // Type-specific comparisons for values that aren't "plain structural" objects.
  if (a instanceof Date || b instanceof Date) {
    return a instanceof Date && b instanceof Date && a.getTime() === b.getTime();
  }
  if (a instanceof RegExp || b instanceof RegExp) {
    return a instanceof RegExp && b instanceof RegExp &&
      a.source === b.source && a.flags === b.flags;
  }

  const aIsArray = Array.isArray(a);
  const bIsArray = Array.isArray(b);
  if (aIsArray !== bIsArray) return false;

  // Circular reference guard: track pairs currently being compared.
  if (seen.has(a) && seen.get(a) === b) return true;
  seen.set(a, b);

  if (aIsArray) {
    if (a.length !== b.length) return false;
    for (let i = 0; i < a.length; i++) {
      if (!deepEqual(a[i], b[i], seen)) return false;
    }
    return true;
  }

  if (a instanceof Map && b instanceof Map) {
    if (a.size !== b.size) return false;
    for (const [key, val] of a) {
      if (!b.has(key) || !deepEqual(val, b.get(key), seen)) return false;
    }
    return true;
  }

  if (a instanceof Set && b instanceof Set) {
    if (a.size !== b.size) return false;
    for (const val of a) {
      // Set membership isn't index-based, so this is O(n) per item — acceptable
      // for typical fixture sizes, worth flagging for very large sets.
      if (![...b].some((bVal) => deepEqual(val, bVal, seen))) return false;
    }
    return true;
  }

  // Plain object comparison.
  const aKeys = Object.keys(a);
  const bKeys = Object.keys(b);
  if (aKeys.length !== bKeys.length) return false;

  for (const key of aKeys) {
    if (!Object.prototype.hasOwnProperty.call(b, key)) return false;
    if (!deepEqual(a[key], b[key], seen)) return false;
  }

  return true;
}
```

```javascript
deepEqual(NaN, NaN);                        // true  — the bug from the scenario, now fixed
deepEqual({ a: 1, b: 2 }, { b: 2, a: 1 });  // true  — key order doesn't matter
deepEqual([1, [2, 3]], [1, [2, 3]]);        // true
deepEqual({ a: 1 }, { a: 1, b: undefined }); // false — key sets differ (see gotcha below)

const x = { name: 'x' };
x.self = x;
const y = { name: 'x' };
y.self = y;
deepEqual(x, y); // true — structurally identical cycles
```

> **Check yourself:** Why does the circular-reference guard here track *pairs* `(a, b)` in a `WeakMap`, rather than tracking `a` alone the way deep clone's `seen` map does?

## Gotchas

**`NaN !== NaN` under `===`, but should usually be equal under deepEqual.** This is the literal bug in the scenario — a deepEqual built on raw `===` for its primitive fast path will say two objects containing `NaN` in the same position are unequal, which breaks any test asserting a computed value that happens to be `NaN` (astonishingly common with things like `0/0` in derived calculations). `Object.is` fixes this specific case.

**Key-order sensitivity from using `JSON.stringify` as a shortcut.** A common "clever" wrong answer is `JSON.stringify(a) === JSON.stringify(b)` — this appears to work on simple cases but fails the moment key order differs (`{a:1,b:2}` vs `{b:2,a:1}` stringify to different strings despite being equal objects), and inherits all of `JSON.stringify`'s other blind spots (drops functions/`undefined`, mangles `Date`/`Map`/`Set`, throws on circular structures) — the same problems as using it for deep clone.

**The `undefined`-valued-key ambiguity.** Whether `{a:1, b: undefined}` should equal `{a:1}` is genuinely ambiguous across libraries (Jest's `toEqual` treats them as equal by default; `toStrictEqual` does not) — an implementation using plain key-count comparison (as above) treats them as *not* equal, since `Object.keys` includes `b`. This is worth stating as an explicit design decision, not something to get caught off guard by.

**Comparing by reference instead of value for `Date`/`RegExp`.** Two different `Date` objects representing the same instant are never `===` to each other and aren't plain objects either, so a generic object-key comparison on a `Date` compares zero enumerable own properties (since `Date`'s value lives in an internal slot) and would incorrectly report *any* two `Date`s as equal — the opposite failure mode from what you'd expect. They need an explicit `instanceof Date` branch comparing `.getTime()`.

**Circular references need pair-tracking, not single-object tracking.** Reusing deep clone's `WeakMap<original, clone>` pattern naively (tracking just `a`) doesn't work for equality, because the question isn't "have I cloned this object before" — it's "am I already in the middle of comparing exactly this `(a, b)` pair," since the same `a` could legitimately be compared against many different `b`s across a large structure (e.g., inside an array of otherwise-unrelated objects).

## Follow-up Questions

**Q (High): How does deepEqual differ semantically from `===` and from a shallow/reference equality check?**

Answer: `===` (and `Object.is`) compare object *references* — two distinct objects with identical contents are never `===`-equal, only two bindings to the literal same object are. Shallow equality (what `React.memo`'s default comparator or a naive "compare top-level keys" check does) compares one level deep: it checks that all top-level keys have `===`-equal values, but doesn't recurse — so `{a: {x:1}}` and `{a: {x:1}}` are shallow-*unequal* because the two `a` values are different object references, even though their contents match. Deep equal recurses all the way down the structure, comparing every nested primitive by value, so those two objects are deep-equal even though neither `===` nor shallow equality would say so.

The trap: conflating "shallow equal" with "deep equal" — this distinction is exactly what React's `useMemo`/`useCallback` dependency arrays rely on (they're shallow, by reference, on purpose, for performance) and it's a very common point of confusion that separates candidates who've internalized *why* dependency arrays behave the way they do from those who've just memorized "put your deps in the array."

---

**Q (High): How do you handle `NaN` when comparing values deeply, given `NaN !== NaN` under strict equality?**

Answer: Use `Object.is(a, b)` instead of `a === b` for the primitive comparison — `Object.is` is specified to treat `NaN` as equal to itself (`Object.is(NaN, NaN) === true`), unlike `===`, which follows IEEE-754 semantics where `NaN` compares unequal to everything including itself. The one adjustment needed on top of `Object.is` is `+0`/`-0`: `Object.is(0, -0)` is `false` (they're distinguishable by `Object.is` even though `0 === -0` is `true`), and most deep-equal implementations intentionally treat `+0`/`-0` as equal to match `===`'s behavior for that specific pair, layering an explicit `a === 0 && b === 0` check after the `Object.is` check to catch that case.

The trap: reaching for `Object.is` and stopping there without realizing it *introduces* a new discrepancy (`+0` vs `-0`) relative to `===`/most testing conventions — using `Object.is` correctly fixes `NaN` but requires a deliberate follow-up decision about zero.

---

**Q (High): Why must two objects with the same keys/values but different key order still be considered equal, and how do you implement that correctly?**

Answer: Object property order is not semantically meaningful in JavaScript for the purpose of value equality — `{a:1, b:2}` and `{b:2, a:1}` represent the same logical value, and any equality check treating them as different would be surprising and wrong for essentially every real use case (deserialized JSON from different sources, objects built by different code paths, etc.). The correct implementation compares by *set membership*, not by parallel iteration: check that both objects have the same number of own enumerable keys (a cheap early-exit), then for every key in `a`, confirm `b` has that exact key (via `hasOwnProperty`, not just truthiness) and that the corresponding values are recursively equal — this is order-independent by construction, since it never iterates both objects "in lockstep."

The trap: implementing comparison via `JSON.stringify(a) === JSON.stringify(b)`, which — because `JSON.stringify` serializes object keys in their enumeration order — is accidentally key-order-*sensitive*, breaking this exact requirement while looking correct on hand-picked test cases where the key order happens to match.

---

**Q (Medium): How do you detect and handle circular references in a deep equality check, as opposed to a deep clone?**

Answer: Deep clone tracks a single map from each original object to its clone, because the relationship being memoized is "this object → its one clone." Deep equal is comparing *two* independent graphs against each other, so the thing that needs to be memoized is "this specific pair of objects is already being compared" — tracked as, e.g., a `WeakMap` from `a` to `b` (or a `Set` of `a`-`b` pair identifiers). When the recursive walk reaches a cycle — `a`'s traversal leads back to the same `a` it started from, and correspondingly `b`'s traversal leads back to the same `b` — the guard recognizes the pair has already been "entered" and returns `true` (equal-so-far) rather than recursing infinitely, letting the rest of the comparison (any non-cyclic parts of the structure) still complete normally.

The trap: reusing deep clone's single-object `WeakMap` pattern unchanged — it doesn't generalize to a two-argument comparison, since the same object `a` might need to be compared against many different, unrelated `b` values across a larger structure, and a single-object map can't represent "paired with *this specific* other object" versus "paired with a different one."

---

**Q (Medium): What's the trap with comparing `{a: undefined}` vs. `{}` — are they deeply equal, and why does it matter for React prop comparisons?**

Answer: Whether these are "equal" is a convention choice, not an objectively correct answer — `Object.keys({a: undefined})` returns `['a']` while `Object.keys({})` returns `[]`, so a key-count-based comparison (as in this implementation) reports them as *not* equal, since the key sets differ even though the accessible value at `a` is `undefined` in both effectively-accessed cases. This matters concretely for React: if a component receives `props = {a: undefined}` on one render and `props = {}` on the next (e.g., because a parent conditionally spreads an object), a strict deep-equal-based `React.memo` comparator would treat these as different props and re-render, even though `props.a` reads as `undefined` either way and the component's rendered output is identical — a subtle, hard-to-spot cause of "why is this memoized component still re-rendering?"

The trap: assuming there's one universally "correct" answer here — the interview-worthy part is recognizing it's a real ambiguity with production consequences (unnecessary re-renders, or test assertions that pass/fail depending on which library convention is in play), not confidently asserting either behavior as objectively right.

---

**Q (Low): How would you extend deepEqual to compare `Map` and `Set` instances?**

Answer: `Map` comparison needs to check `size` matches first (cheap early exit), then for every key in `a`, confirm `b` has that exact key (via `b.has(key)`, which for object keys relies on the same reference unless you also want to deep-compare keys themselves — usually `Map` key comparison is left as reference-equality, since deep-comparing keys means you can no longer use `b.has()`/`b.get()` directly and have to fall back to O(n²) pairwise comparison) and that the values compare deeply equal via `deepEqual(a.get(key), b.get(key))`. `Set` comparison is inherently more expensive because set membership isn't indexed the way array position is — for each item in `a`, you have to check whether *some* item in `b` is deeply equal to it (since sets are unordered and items aren't retrievable by a key), which is O(n²) in the worst case for sets of non-primitive values, versus O(n) for `Map`/array comparison.

The trap: assuming `Set` comparison can be done with the same per-index or per-key efficiency as arrays/`Map`s — the unordered, non-indexed nature of `Set` genuinely changes the complexity class of the comparison for object-valued sets, which is worth naming rather than glossing over.

---

**Q (Low): How does deep equality checking relate to how `React.memo`/`useMemo` dependency arrays actually work?**

Answer: React's built-in comparisons (`React.memo`'s default prop comparator, `useMemo`/`useCallback`/`useEffect` dependency arrays) are all **shallow**, by reference, by deliberate design choice — not deep — because a deep comparison on every render would itself be a performance cost proportional to the size of the data being compared, potentially larger than the cost of just re-rendering. This is why the common advice is "don't put a new object/array literal in a dependency array on every render" (`{ id }` created fresh each render is never `===` to the previous render's `{ id }` even if `id` is unchanged) — the fix is either memoizing that object itself, extracting the primitive value directly into the dependency array, or, rarely, deliberately opting into a deep comparison via a custom hook (`useDeepCompareMemo`) when the performance trade-off is acceptable for a specific case with genuinely deep, infrequently-changing data.

The trap: recommending "just use a deep-equal-based memo comparator everywhere" as a general fix for unnecessary re-renders — this trades one performance problem (re-rendering) for a potentially worse one (a full deep traversal on every single render to *decide whether* to re-render), and the better fix in most real cases is structural (stabilizing references at the source) rather than comparison-strategy-based.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement deepEqual with correct `NaN` handling from memory
- [ ] Can explain why key-count + `hasOwnProperty` checks are needed instead of `JSON.stringify` comparison
- [ ] Can explain why circular-reference tracking for equality needs *pairs*, not single objects, unlike deep clone
- [ ] Can articulate the `{a: undefined}` vs `{}` ambiguity and its real-world React consequence
- [ ] Can explain, precisely, the difference between `===`, shallow equality, and deep equality
- [ ] Can explain why React's dependency arrays are intentionally shallow, not deep

---
*Next: Curry, Compose & Pipe — shifting from comparing/copying data structures to composing functions, a different flavor of closures-and-edge-cases machine coding.*
