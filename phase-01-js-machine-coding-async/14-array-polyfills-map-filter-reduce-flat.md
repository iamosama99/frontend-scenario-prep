# Array Method Polyfills: map/filter/reduce/flat

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Attach to `Array.prototype` | Define as non-enumerable methods via `Object.defineProperty`, guarded by existence checks | Naively assigning `Array.prototype.map = function() {...}` works but makes the polyfill enumerable, so it leaks into `for...in` loops over arrays — a classic, real-world jQuery-era bug source |
| Sparse array holes are skipped | Check `i in this` before invoking the callback, not just iterate `0..length` | `map`/`filter`/`forEach` all skip holes in sparse arrays (`[1, , 3]`) per spec — a naive `for` loop invokes the callback with `undefined` for holes, which is observably different |
| `reduce` without initial value uses first element as accumulator | Special-case the zero-initial-value call signature, and throw on an empty array with no initial value | This is the most commonly gotten-wrong branch of `reduce` — get the index bookkeeping wrong here and everything after it shifts by one |
| `flat`'s depth is recursive, not one level fixed | Recurse only while `depth > 0`, decrementing per level, defaulting to `depth = 1` | `flat()` with no argument flattens exactly one level; `Infinity` fully flattens; this parameterization is easy to hardcode wrong |

## The Scenario

"Implement `Array.prototype.map`, `.filter`, `.reduce`, and `.flat` from scratch — as polyfills attached to the prototype, not standalone functions — matching native behavior exactly, including their less obvious edge cases like sparse arrays and `reduce` with no initial value."

## Clarifying Questions

- **Should these be attached to `Array.prototype` (true polyfills) or written as standalone utility functions taking an array as an argument?** The word "polyfill" implies the former, but I'd confirm — attaching to the prototype has real considerations a standalone function doesn't (enumerability, guarding against double-definition if the native method already exists, and using `this` instead of a parameter).
- **Do they need to correctly skip holes in sparse arrays** (e.g., `[1, , 3].map(x => x * 2)` should produce `[2, , 6]`, preserving the hole, not `[2, NaN, 6]` or `[2, undefined, 6]`)? This is genuine, spec-mandated behavior for `map`/`filter`/`forEach`/`reduce`, and it's the kind of edge case that separates "knows the common case" from "has actually read the spec" — I'd confirm it's in scope, since it changes the iteration logic from a plain `for` loop to one that checks property existence first.
- **For `reduce`, should calling it on an empty array with no initial value throw, matching native behavior** (`TypeError: Reduce of empty array with no initial value`), or is silently returning `undefined` acceptable for this exercise? I'd implement the spec-accurate throwing behavior by default, since it's a real, commonly-hit runtime error in production code (e.g., `arr.reduce((a, b) => a + b)` on an array that turned out to be empty) and worth demonstrating awareness of.
- **For `flat`, what should the default depth be if no argument is passed, and should `Infinity` be supported for full flattening?** Native `flat()` defaults to depth `1` (not full flattening) — this default is a common source of confusion (a deeply nested array only appears "half-flattened" if depth isn't explicitly bumped or set to `Infinity`), so I'd confirm this default matches what's expected.

## Approach & Trade-offs

Each polyfill is attached as a non-enumerable property on `Array.prototype`, guarded by `if (!Array.prototype.map)` (or defined under a different name like `myMap` to avoid overwriting the native implementation entirely, which I'd default to for a learning/demo context so native methods remain available for comparison in the same file).

**Why `Object.defineProperty` instead of direct assignment.** `Array.prototype.map = function () {...}` works functionally, but direct property assignment creates an *enumerable* property by default — meaning `for (const key in someArray)` would now iterate over `'map'` as if it were an own/inherited enumerable key, alongside the array's actual indices. This was a real, painful class of bugs in the pre-ES5 era when libraries polyfilled `Array.prototype` carelessly, and it's exactly why native built-ins are defined with `enumerable: false`. Using `Object.defineProperty(Array.prototype, 'myMap', { value: fn, writable: true, configurable: true, enumerable: false })` avoids reintroducing that bug class.

**Sparse array handling — the detail most implementations skip.** `[1, , 3]` has a "hole" at index 1 — it's not `undefined` stored at that index, the index simply doesn't exist as an own property. Native `map`, `filter`, `forEach`, and `reduce` all specifically check `hasOwnProperty` (or the `in` operator against `this`) before invoking the callback for a given index, and skip indices that don't exist as own properties — `map` still preserves the hole in the *output* array (the result is also sparse at that index), while `filter` simply never considers that index for inclusion. A naive `for (let i = 0; i < arr.length; i++) callback(arr[i], i, arr)` loop doesn't distinguish "index has no own property" from "index exists with value `undefined`," calling the callback with `undefined` for holes — which is observably wrong (extra callback invocations that shouldn't happen, and for `map`, a dense result where a sparse one was expected).

**`reduce`'s no-initial-value branch is the trickiest control flow.** When no initial value is passed, the *first* element of the array becomes the initial accumulator, and iteration starts from index 1, not 0. When the array is empty and no initial value was given, this is a `TypeError` — there's no possible way to produce a result, since there's no initial value and no first element to fall back to. Handling both branches correctly (with-initial-value starts folding from index 0; without-initial-value starts folding from index 1, seeded by index 0) without an off-by-one is the main place implementations go wrong. I explicitly branch on `arguments.length` (a regular `function`, not an arrow function, to access `arguments`) to distinguish "no second argument passed" from "second argument passed as `undefined`" — these are different: `reduce(fn, undefined)` *does* have an initial value (it's `undefined`), while `reduce(fn)` does not.

**`flat`'s recursive depth parameter**, defaulting to `1`, recursing into any element that's itself an array while `depth > 0`, decrementing depth on each recursive call — implemented via `reduce` internally (a nice demonstration that `flat` can be built compositionally on top of the other polyfills already written, rather than as an independent primitive).

## Solution

```javascript
// --- myMap ---
function myMap(callback, thisArg) {
  if (this == null) throw new TypeError('Array.prototype.myMap called on null or undefined');
  if (typeof callback !== 'function') throw new TypeError(callback + ' is not a function');

  const result = new Array(this.length); // preserves sparseness by construction
  for (let i = 0; i < this.length; i++) {
    if (i in this) { // skip holes — this is the sparse-array-correct check
      result[i] = callback.call(thisArg, this[i], i, this);
    }
  }
  return result;
}

// --- myFilter ---
function myFilter(callback, thisArg) {
  if (this == null) throw new TypeError('Array.prototype.myFilter called on null or undefined');
  if (typeof callback !== 'function') throw new TypeError(callback + ' is not a function');

  const result = [];
  for (let i = 0; i < this.length; i++) {
    if (i in this && callback.call(thisArg, this[i], i, this)) {
      result.push(this[i]);
    }
  }
  return result;
}

// --- myReduce ---
function myReduce(callback, ...initialArg) {
  if (this == null) throw new TypeError('Array.prototype.myReduce called on null or undefined');
  if (typeof callback !== 'function') throw new TypeError(callback + ' is not a function');

  const hasInitial = initialArg.length > 0;
  let accumulator = hasInitial ? initialArg[0] : undefined;
  let startIndex = 0;
  let accumulatorSet = hasInitial;

  if (!accumulatorSet) {
    // No initial value: find the first existing index to seed the accumulator.
    while (startIndex < this.length && !(startIndex in this)) startIndex++;
    if (startIndex >= this.length) {
      throw new TypeError('Reduce of empty array with no initial value');
    }
    accumulator = this[startIndex];
    startIndex++;
  }

  for (let i = startIndex; i < this.length; i++) {
    if (i in this) {
      accumulator = callback(accumulator, this[i], i, this);
    }
  }
  return accumulator;
}

// --- myFlat ---
function myFlat(depth = 1) {
  if (this == null) throw new TypeError('Array.prototype.myFlat called on null or undefined');

  return myReduce.call(this, (flat, item) => {
    if (Array.isArray(item) && depth > 0) {
      flat.push(...myFlat.call(item, depth - 1));
    } else {
      flat.push(item);
    }
    return flat;
  }, []);
}

for (const [name, fn] of Object.entries({ myMap, myFilter, myReduce, myFlat })) {
  Object.defineProperty(Array.prototype, name, {
    value: fn,
    writable: true,
    configurable: true,
    enumerable: false, // critical — keeps `for...in` over arrays from seeing these
  });
}
```

```javascript
[1, 2, 3].myMap((x) => x * 2);              // [2, 4, 6]
[1, 2, 3, 4].myFilter((x) => x % 2 === 0);   // [2, 4]
[1, 2, 3, 4].myReduce((sum, x) => sum + x);  // 10 — no initial value, starts from index 0's value
[1, [2, [3, [4]]]].myFlat();                 // [1, 2, [3, [4]]] — depth 1 only
[1, [2, [3, [4]]]].myFlat(Infinity);         // [1, 2, 3, 4] — fully flattened

const sparse = [1, , 3];
sparse.myMap((x) => x * 2);                  // [2, <1 empty item>, 6] — hole preserved

[].myReduce((a, b) => a + b);                // throws TypeError: Reduce of empty array with no initial value

for (const key in [1, 2, 3]) console.log(key); // logs '0','1','2' only — myMap etc. don't leak in
```

> **Check yourself:** Why does `myReduce` need to check `initialArg.length > 0` (via rest params) rather than simply checking `initialValue !== undefined` to decide whether an initial value was provided?

## Gotchas

**Treating holes the same as `undefined` values.** `[1, , 3].map(x => x * 2)` on native arrays produces `[2, <hole>, 6]` — the hole is preserved, and the callback is never even invoked for that index. A naive `for` loop over `0..length` invoking the callback unconditionally treats the hole as `this[1] === undefined`, invoking the callback with `undefined` and producing `[2, NaN, 6]` (or `[2, undefined, 6]` if the callback doesn't do arithmetic) — both spec-incorrect and observably different from native behavior if anything downstream distinguishes holes from explicit `undefined`.

**Getting `reduce`'s initial-value detection wrong.** Checking `if (initialValue !== undefined)` instead of checking argument count (`arguments.length` or a rest-parameter length check) incorrectly treats `arr.reduce(fn, undefined)` (which **does** supply an initial value — it's just `undefined`) the same as `arr.reduce(fn)` (which does **not**) — these have different starting indices and different empty-array behavior, and conflating them is a subtle, hard-to-spot-in-testing bug (`undefined` as an initial value is a rare but valid use, e.g., reducing over a chain of state transitions that starts from "no prior state").

**Forgetting `reduce` on an empty array with no initial value must throw.** A polyfill that just returns `undefined` in that case looks reasonable and won't crash — until it silently produces a wrong answer at a call site that assumed `reduce` would loudly fail on unexpectedly-empty input (a real, common defensive-programming pattern: relying on `reduce`'s throw as an implicit assertion that the array wasn't empty).

**Hardcoding `flat`'s recursion to fully flatten, ignoring the `depth` parameter.** `[1,[2,[3]]].flat()` (no argument) should only flatten **one level**, yielding `[1, 2, [3]]`, not `[1, 2, 3]` — a very common mistake is writing a fully-recursive flatten by default and only supporting `Infinity` as if it were the only valid mode, missing that partial-depth flattening is the actual default and a real, intentional part of the API.

**Assigning polyfills as enumerable properties on `Array.prototype`.** `Array.prototype.myMap = function () {...}` (direct assignment) creates an enumerable property, which means every `for...in` loop over *every array in the program* now also yields `'myMap'` as a "key" — a correctness bug for any code (including third-party code) that uses `for...in` over arrays (itself generally discouraged for arrays, but still occurs, especially in older codebases), and exactly the class of real-world bug that motivated `enumerable: false` on native prototype methods.

## Follow-up Questions

**Q (High): Why do `map`, `filter`, and `reduce` need to explicitly check for holes in sparse arrays, and what's the actual observable difference if you don't?**

Answer: A "hole" in a sparse array (`[1, , 3]`) is an index within the array's `length` range that has no own property at all — as opposed to an index that exists with the value `undefined`. The distinction matters because `map`/`filter`/`forEach`/`reduce` are all specified to *skip* indices that don't exist as own properties, calling the callback only for indices that genuinely have a value (including an explicit `undefined`). Concretely: `[1, , 3].map(x => x * 2)` produces `[2, <1 empty item>, 6]` — the hole persists in the result, and the callback runs exactly twice, not three times. A naive `for (let i = 0; i < arr.length; i++)` loop that indexes directly (`arr[i]`) can't distinguish a hole from a real `undefined` (both read as `undefined` via bracket access) and will invoke the callback for every index unconditionally — producing a *dense* result and one extra callback invocation, which is wrong both in shape (dense vs. sparse output) and in side-effect count (if the callback has side effects, the invocation count itself is observably different).

The trap: assuming `arr[i] === undefined` is sufficient to detect a hole — it's not, because a real value of `undefined` at an existing index reads identically via bracket access; the correct check is `i in arr` (or `Object.prototype.hasOwnProperty.call(arr, i)`), which tests for *property existence*, not *value*.

---

**Q (High): Walk through exactly how `reduce` behaves differently when called with vs. without an initial value, including on an empty array.**

Answer: With an initial value (`arr.reduce(fn, initial)`), the accumulator starts as `initial`, and the callback is invoked once per element starting from index 0, folding each element into the accumulator in turn — so an empty array simply returns `initial` unchanged, with zero callback invocations. Without an initial value (`arr.reduce(fn)`), the *first element* of the array becomes the initial accumulator value with no callback invocation for it, and folding begins from index 1 — so a single-element array returns that element unchanged, again with zero callback invocations; a two-element array invokes the callback exactly once (folding element 1 into the accumulator seeded from element 0). Critically, calling `reduce` with no initial value on an **empty** array is a `TypeError` (`Reduce of empty array with no initial value`) — there's no first element to seed the accumulator and no supplied initial value, so there's no way to produce any result at all, and the spec chooses to fail loudly rather than return `undefined` silently.

The trap: describing "no initial value" as functionally equivalent to "initial value defaults to `undefined`" — it isn't; those are two genuinely different call signatures with different starting indices and different empty-array behavior, and native `reduce` distinguishes them by *argument count*, not by the initial value's actual value.

---

**Q (Medium): How would you implement `flat(depth)` in terms of `reduce`, and why does the default depth matter?**

Answer: `flat` can be expressed compositionally as a `reduce` that, for each element, either pushes it directly into the accumulator (if it's not an array, or `depth` has been exhausted) or recursively flattens it one level shallower and spreads *that* result in (if it's an array and `depth > 0`) — `arr.reduce((flat, item) => Array.isArray(item) && depth > 0 ? [...flat, ...flatten(item, depth - 1)] : [...flat, item], [])`. The default depth of exactly `1` (not `Infinity`, and not `0`) matters because it's a genuine, spec-chosen middle ground: `flat()` with no argument flattens *one* level of nesting only, leaving deeper structure intact — `[1, [2, [3]]].flat()` yields `[1, 2, [3]]`, not `[1, 2, 3]`. This surprises people who assume "flatten" means "fully flatten" by default; full flattening requires explicitly passing `Infinity`.

The trap: implementing `flat` as always-fully-recursive and treating the `depth` parameter as an afterthought/edge case, when partial-depth flattening (specifically depth-1-by-default) is the actual primary, spec-mandated behavior — getting this backwards means the "default" case is wrong, not just an edge case.

---

**Q (Medium): Why is it important to use `Object.defineProperty` with `enumerable: false` when attaching a polyfill to `Array.prototype`, instead of plain assignment?**

Answer: Plain property assignment (`Array.prototype.myMap = fn`) creates a property with `enumerable: true` by default — and enumerable properties on a prototype are visited by `for...in` loops over *any* instance of that prototype, including every array anywhere in the running program, not just arrays the polyfill author controls. This means any code — including third-party library code the author has no visibility into — that iterates an array with `for...in` (itself a discouraged pattern for arrays specifically because of exactly this kind of prototype-pollution risk, but one that still appears in real, especially older, codebases) would suddenly also "see" `'myMap'` as if it were an array index or key. This was a genuine, painful source of bugs in the pre-ES5/jQuery era when libraries extended built-in prototypes carelessly — Prototype.js's early `Array.prototype` extensions were a famous example, and it's part of why extending built-in prototypes at all became broadly discouraged practice. `Object.defineProperty` with `enumerable: false` (matching how native methods are actually defined) avoids reintroducing that specific failure mode.

The trap: treating `Object.defineProperty` as unnecessary ceremony ("assignment does the same thing") — functionally the method works identically either way when called directly (`arr.myMap(fn)`), but the *enumerability* difference is a real, observable, and historically significant bug source that only manifests in code paths (`for...in` over arrays) the polyfill's own tests likely never exercise, making it an easy thing to miss without knowing to specifically check for it.

---

**Q (Low): How would you polyfill `Array.prototype.flatMap`, and why is it more than just `map` followed by `flat`?**

Answer: `flatMap(callback)` is semantically equivalent to `.map(callback).flat(1)` (specifically depth 1, always — `flatMap` doesn't accept a depth argument), and can be implemented either literally as that two-step composition or, for a minor performance win, as a single pass that avoids constructing the intermediate mapped array before flattening it (`reduce`-based, pushing each mapped result directly, spreading if it's an array). The reason it exists as its own method rather than requiring `.map().flat(1)` everywhere is primarily ergonomic/performance — a very common pattern (map each item to zero-or-more output items, e.g., splitting each string in an array into its words and producing one flat array of all words) previously required the two-call chain and an intermediate throwaway array; `flatMap` does it in one call and, in engine implementations, can avoid materializing that intermediate structure.

The trap: implementing `flatMap` as literally `this.myMap(cb).myFlat(2)` or with the wrong hardcoded depth — the depth is always exactly `1`, not configurable and not deeper, which is easy to get wrong if conflating it with generic `flat`'s configurable depth parameter.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement `map`, `filter`, `reduce`, and `flat` as prototype polyfills from memory, including sparse-array hole handling
- [ ] Can explain precisely why `i in this` (not `this[i] !== undefined`) is the correct hole check
- [ ] Can walk through `reduce`'s with-vs-without-initial-value branches and the empty-array throw condition without hesitation
- [ ] Can explain `flat`'s default depth of 1 and implement it via `reduce` composition
- [ ] Can explain why `enumerable: false` matters when extending `Array.prototype`, with a concrete `for...in` example

---
*Next: call/apply/bind Polyfills — moves from re-implementing array iteration methods to re-implementing the mechanism that controls `this` binding itself, one level closer to the language's core semantics.*
