# Deep Clone — Circular References & Special Types

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Recursive structural copy | Walk the object graph, cloning each nested value | `JSON.parse(JSON.stringify(x))` silently drops functions, `undefined`, and `Date`s become strings |
| Circular reference safety | A `WeakMap<original, clone>` tracks what's already been cloned | Without it, an object referencing itself causes infinite recursion → stack overflow |
| Shared reference preservation | The same `WeakMap` also ensures two properties pointing at the *same* nested object still point at the same clone | Naive recursion clones shared references twice, silently breaking object identity assumptions elsewhere in the app |
| Special types | `Date`, `RegExp`, `Map`, `Set` each need type-specific reconstruction | A generic "copy own enumerable keys" loop produces a broken/empty clone for all four |

## The Scenario

"Write a `deepClone` function that does a structural deep copy of an arbitrary nested object — arrays, plain objects, and also correctly handles `Date`, `RegExp`, `Map`, and `Set`. It also needs to not blow up on objects with circular references. Don't just tell me `structuredClone()` exists — assume we need this to run somewhere that doesn't have it, or assume the interviewer wants to see you build it."

## Clarifying Questions

- **What counts as "needs to be supported"?** Confirming the exact type list up front (plain objects, arrays, `Date`, `RegExp`, `Map`, `Set`, primitives) — because each unsupported type (functions, `Symbol`s, class instances, DOM nodes) needs an explicit, stated decision rather than silent mishandling.
- **What should happen to functions encountered inside the object graph?** Functions can't be meaningfully cloned (you can't recreate a closure's captured scope) — the sane default is to keep the same function reference in the clone, and I'd say so explicitly rather than let it be an unstated gap.
- **Does it need to preserve custom prototypes / class instances, or is "plain object and array" enough?** This changes whether `Object.create(Object.getPrototypeOf(obj))` is needed versus a plain `{}`.
- **Should circular references produce a clone that *also* has the identical circular shape** (i.e., `clone.self === clone`, not `clone.self === original`)? Yes — that's the actual point of testing circular references; a clone that merely avoids crashing but loses the cycle's shape is only half-correct.

## Approach & Trade-offs

The mechanism is recursive: for each value, branch on its type. Primitives (`string`, `number`, `boolean`, `null`, `undefined`, `bigint`, `symbol` used as a value) return as-is — they're immutable, so "cloning" them is a no-op by definition.

For anything reference-typed (object, array, `Date`, `RegExp`, `Map`, `Set`), the critical structural decision is a `WeakMap` keyed by the *original* reference, storing the clone as it's created — **before** recursing into that value's children. This single data structure solves two problems at once:

1. **Circular references** — if node A contains a reference back to itself (directly or through a cycle), the recursive call for that back-reference finds A already in the `WeakMap` and returns the existing (possibly still-being-populated) clone instead of recursing infinitely.
2. **Shared references** — if two different properties both point at the same nested object, both encounters find the same clone in the `WeakMap` and reuse it, so the clone's object graph has the same aliasing structure as the original, not two independent copies that have silently diverged.

`WeakMap` specifically (not a plain `Map`) is chosen because it doesn't prevent the original objects from being garbage collected once the clone function returns and nothing else references them — the map's keys don't keep the originals alive.

I considered `structuredClone()` as "the real answer" for production code — it's the platform-native structured clone algorithm (Node 17+, all modern browsers), and it already handles circular refs, `Date`, `RegExp`, `Map`, `Set`, and more (`ArrayBuffer`, typed arrays) correctly. But the interview ask is specifically to build the mechanism, both to test whether the candidate understands *why* `JSON.parse(JSON.stringify())` fails, and because `structuredClone()` has its own gaps worth knowing (no functions, and it throws on some values like DOM nodes in certain contexts).

## Solution

```javascript
function deepClone(value, seen = new WeakMap()) {
  // Primitives (including null) — return as-is, nothing to clone.
  if (value === null || typeof value !== 'object') {
    return value;
  }

  // Circular / shared reference — return the clone already in progress.
  if (seen.has(value)) {
    return seen.get(value);
  }

  // --- Special types need type-specific reconstruction ---

  if (value instanceof Date) {
    return new Date(value.getTime());
  }

  if (value instanceof RegExp) {
    return new RegExp(value.source, value.flags);
  }

  if (value instanceof Map) {
    const clone = new Map();
    seen.set(value, clone); // register BEFORE recursing, to catch cycles through this Map
    for (const [k, v] of value) {
      clone.set(deepClone(k, seen), deepClone(v, seen));
    }
    return clone;
  }

  if (value instanceof Set) {
    const clone = new Set();
    seen.set(value, clone);
    for (const item of value) {
      clone.add(deepClone(item, seen));
    }
    return clone;
  }

  if (Array.isArray(value)) {
    const clone = [];
    seen.set(value, clone);
    for (let i = 0; i < value.length; i++) {
      clone[i] = deepClone(value[i], seen);
    }
    return clone;
  }

  // Plain object (or class instance) — preserve the prototype chain.
  const clone = Object.create(Object.getPrototypeOf(value));
  seen.set(value, clone);
  for (const key of Reflect.ownKeys(value)) { // includes Symbol keys
    clone[key] = deepClone(value[key], seen);
  }
  return clone;
}
```

```javascript
// Circular reference example
const a = { name: 'a' };
a.self = a;

const clone = deepClone(a);
console.log(clone.self === clone); // true — the cycle is preserved in the clone's own graph
console.log(clone.self === a);     // false — it's not pointing back at the original

// Shared reference example
const shared = { count: 1 };
const original = { x: shared, y: shared };
const cloned = deepClone(original);
console.log(cloned.x === cloned.y); // true — both still point at the SAME cloned object
```

> **Check yourself:** Why must `seen.set(value, clone)` happen *before* recursing into that value's children, rather than after the clone is fully built?

## Why Not Just `JSON.parse(JSON.stringify(obj))`?

It's the answer every candidate reaches for first, and it's worth stating precisely why it fails, rather than just "it's bad":

- **Functions and `undefined` values are silently dropped** — `JSON.stringify({ fn: () => {}, x: undefined })` produces `"{}"`.
- **`Date` becomes a string**, not a `Date` — `JSON.parse(JSON.stringify(new Date()))` gives you a plain string, and calling `.getTime()` on it throws.
- **`Map` and `Set` serialize to `{}`** — `JSON.stringify(new Map([['a', 1]]))` produces `"{}"`, silently losing all data.
- **Circular references throw synchronously** — `JSON.stringify` throws `TypeError: Converting circular structure to JSON` rather than handling it at all.
- **`NaN` and `Infinity` become `null`** — silent, hard-to-trace data corruption.

## Gotchas

**Registering the clone in `seen` too late.** If you build the full clone object (recursing into every child) *before* calling `seen.set(value, clone)`, a circular reference inside that same object will recurse infinitely before the map entry ever gets a chance to short-circuit it — the map has to be populated with a (possibly still-empty) clone *before* recursing into children, specifically so a cycle back to the same node finds it already registered.

**Only solving circular refs, not shared refs.** A common shallow fix is checking `seen` only to prevent stack overflow, but using a fresh clone every time a value is *first* encountered from a different path — if the same map correctly returns the cached clone whenever the identical original reference is seen again (from any path, not just a literal cycle back to an ancestor), shared-reference preservation falls out for free. Missing this means two properties that pointed at the same object before cloning silently point at two different, now-independent objects after — a real bug if app code relies on referential equality (e.g., a memoized selector, or `===` comparisons in a reducer).

**Prototype loss.** Using `{}` instead of `Object.create(Object.getPrototypeOf(value))` turns class instances into plain objects — `clone instanceof MyClass` becomes `false`, `instanceof` checks and prototype methods elsewhere in the app silently stop working on the cloned copy.

**`Date` spread/property-copy instead of reconstruction.** `{ ...someDate }` or a generic key-loop over a `Date` produces `{}` — `Date`'s value lives in an internal slot, not in enumerable own properties, so it has to be reconstructed via `new Date(value.getTime())`.

**`RegExp` losing flags.** `new RegExp(value.source)` alone drops `g`/`i`/`m`/etc. — both `source` and `flags` need to be passed.

**Functions can't be truly cloned.** Deciding to keep the same function reference is the correct default, but it should be a stated decision (documented behavior), not a silent side effect of "the generic loop happened to skip them."

## Follow-up Questions

**Q (High): How do you handle circular references in a recursive deep clone, and why does a naive implementation stack-overflow?**

Answer: A naive recursive `deepClone` that just walks `Object.keys(value)` and recurses into each one, with no memory of what's already been visited, will — when it hits a self-referencing or mutually-referencing structure — recurse into the same object again, which recurses into itself again, forever, until the call stack is exhausted (`RangeError: Maximum call stack size exceeded`). The fix is a `WeakMap` that maps each original object to its in-progress clone, populated *before* recursing into that object's children; when the recursive walk reaches a reference back to an object already in the map, it returns the existing clone instead of recursing again, terminating the cycle.

The trap: candidates sometimes propose "just track visited objects in an array and check `includes()`" — this works but is O(n) per lookup (O(n²) overall for a graph with n reference-typed nodes) and, more importantly, using an array (or a plain object/`Map`) instead of a `WeakMap` means the tracking structure itself holds strong references to every object in the graph, which can leak memory if the clone function is called on large, long-lived structures and the tracking map isn't scoped to a single call (as it correctly is here, via the `seen` parameter).

---

**Q (High): Why is `JSON.parse(JSON.stringify(obj))` insufficient for deep cloning?**

Answer: `JSON.stringify` can only represent the subset of JavaScript values that map onto JSON's type system — objects, arrays, strings, numbers, booleans, and `null`. Everything else is either dropped (`function`, `undefined`, `Symbol`), coerced destructively (`Date` → ISO string, losing the `Date` type; `NaN`/`Infinity` → `null`), or serialized to an empty, data-losing shape (`Map`/`Set` → `{}`, since neither has own enumerable string-keyed properties). On top of the type coverage gap, `JSON.stringify` throws synchronously on any circular reference rather than handling it, making it unusable on self-referencing structures at all, independent of the type-coverage issue.

The trap: an answer that says "it's slow" and stops there — performance is a secondary concern; the primary, interview-relevant reason it's wrong is *correctness* (silent data loss and unhandled types), which is a much more dangerous failure mode than slowness because it doesn't announce itself.

---

**Q (High): How do you preserve shared references between two properties pointing at the same nested object — not just avoid infinite recursion?**

Answer: The same `WeakMap` that prevents infinite recursion on cycles also solves this, as long as it's checked (and populated) by *original object identity*, not by path. When the clone function encounters `original.x`, it clones the shared object and registers `seen.set(sharedObj, clonedShared)`. When it later encounters `original.y`, which points at the *same* `sharedObj`, the `seen.has(sharedObj)` check finds it and returns the *same* `clonedShared`, so `clone.x === clone.y` remains `true`, matching `original.x === original.y`. Without this, a clone implementation that only checks "have I seen this object on the current path" (to catch cycles) but starts fresh for objects reached via a different path will produce two independent clones for `x` and `y`, silently breaking any code downstream that relies on referential equality between them (a very real bug — e.g., a React component memoized on reference equality, or a Redux reducer comparing `prevState.x === nextState.x`).

The trap: implementing cycle detection with a path-scoped "currently on stack" set instead of a whole-call-scoped map — that correctly prevents infinite recursion but doesn't achieve reference-sharing preservation, which is a distinct and easy-to-miss requirement.

---

**Q (Medium): How would you extend deepClone to support `Map`, `Set`, `Date`, and `RegExp`?**

Answer: Each needs type-specific reconstruction because none of them expose their internal state as generically-enumerable own properties the way a plain object does. `Date` reconstructs via `new Date(value.getTime())` (copying the internal timestamp). `RegExp` reconstructs via `new RegExp(value.source, value.flags)` (both the pattern and the flags, since flags aren't part of `source`). `Map` and `Set` need their own branch that creates a new empty instance, registers it in the `seen` map *before* iterating (to support cycles/shared refs through the collection itself), then iterates the original's entries/values and recursively clones each one into the new collection — critically, `Map` keys can themselves be objects and need to be deep-cloned too, not just the values.

The trap: cloning a `Map`'s or `Set`'s *contents* shallowly (e.g., `new Map(value)`) — this copies the entries but doesn't deep-clone object values/keys inside them, so mutating a cloned `Map`'s value object still mutates the original's.

---

**Q (Medium): What are the limitations of the native `structuredClone()` API, and when would you still write your own deep clone?**

Answer: `structuredClone()` (available in modern browsers and Node 17+) implements the HTML structured clone algorithm and correctly handles far more than a hand-rolled version typically would out of the box — circular references, `Date`, `RegExp`, `Map`, `Set`, typed arrays, `ArrayBuffer`, even `Blob`/`File` in browser contexts — with less code and fewer bugs than a custom implementation. Its real limitations: it **cannot clone functions** (throws a `DataCloneError`), can't clone DOM nodes in most contexts, doesn't preserve the prototype chain of custom class instances (they come back as plain objects, losing `instanceof`), and can't clone values with property accessors (getters/setters) — it clones the *current value*, not the accessor. You'd still write a custom clone when you need to preserve class identity/prototypes, when you need functions to survive (even as shared references), or when you're targeting an environment without `structuredClone` support (some older runtimes, or certain sandboxed/worker contexts with restrictions).

The trap: presenting `structuredClone()` as strictly superior with no caveats — in a senior interview, naming its specific gaps (prototype loss, no functions) is exactly the depth signal being tested for, especially since it's the first thing many candidates reach for as "the modern answer" without knowing its edges.

---

**Q (Low): How do you handle cloning objects with custom prototypes or class instances?**

Answer: Use `Object.create(Object.getPrototypeOf(value))` to create the clone's shell instead of a plain `{}` literal, so the clone inherits from the same prototype as the original (preserving `instanceof` checks and access to prototype methods), then copy/clone the instance's own enumerable properties onto that shell as usual. This handles the common case of simple class instances whose state lives entirely in own properties, but it does **not** re-run the class's constructor — for classes that maintain invariants only enforceable through construction logic (e.g., private fields via `#`, which aren't reachable through `Object.getPrototypeOf`/property enumeration at all), this approach silently fails to clone that private state, and a more targeted per-class clone/`clone()` method becomes necessary.

The trap: assuming `Object.create` + property copy is a universal solution — it breaks down specifically for private class fields (`#field`), which are invisible to `Reflect.ownKeys` and any other reflection-based enumeration, by design.

---

**Q (Low): How does deep cloning interact with `Symbol`-keyed properties?**

Answer: `Object.keys()` and `for...in` both skip `Symbol`-keyed properties entirely, so a clone implementation built on either will silently drop any data stored under a `Symbol` key. `Reflect.ownKeys(value)` (or `Object.getOwnPropertyNames(value).concat(Object.getOwnPropertySymbols(value))`) includes both string and symbol keys, which is why the reference solution above uses `Reflect.ownKeys` rather than `Object.keys`. Whether this matters in practice depends on whether the codebase actually uses `Symbol`s as property keys (common for library-internal "private-ish" properties, like React's internal fiber fields, or Well-Known Symbols like `Symbol.iterator` used to make an object iterable) — a clone that drops a `Symbol.iterator` implementation, for instance, would silently make the cloned object non-iterable even though the original was.

The trap: not knowing that `Object.keys`/`for...in` skip symbols at all — this is a genuinely obscure-enough JS detail that even experienced engineers miss it, which is exactly why it's a good depth-probing follow-up.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement a recursive deepClone with a `WeakMap` for cycle detection from memory
- [ ] Can explain why registering the clone in the map must happen *before* recursing into children
- [ ] Can explain the distinction between "avoiding infinite recursion" and "preserving shared references" and why the same fix solves both
- [ ] Can name at least four concrete ways `JSON.parse(JSON.stringify())` silently corrupts data
- [ ] Can state two real limitations of `structuredClone()`
- [ ] Can explain why `Date`/`RegExp`/`Map`/`Set` each need type-specific reconstruction rather than a generic property loop

---
*Next: Deep Equal — the natural counterpart: instead of copying a structure, recursively comparing two structures for value equality, with its own set of edge cases (NaN, key order, circular refs).*
