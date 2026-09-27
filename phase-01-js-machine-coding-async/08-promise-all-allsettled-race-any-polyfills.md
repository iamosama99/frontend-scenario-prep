# Promise.all / allSettled / race / any Polyfills

## Quick Reference

| Method | Resolves when | Rejects when | Result shape |
|---|---|---|---|
| `all` | Every input settles successfully | The **first** input rejects (short-circuits) | Array of resolved values, in input order |
| `allSettled` | Every input has settled (success or failure) — never rejects | Never | Array of `{status, value}` / `{status, reason}` objects, in input order |
| `race` | The **first** input to settle, resolves | The **first** input to settle, if that one rejects | Whatever that single first-settled input produced |
| `any` | The **first** input to *fulfill* | **Every** input rejects | The single fulfilled value, or an `AggregateError` of all rejection reasons |

## The Scenario

"Implement `Promise.all`, `Promise.allSettled`, `Promise.race`, and `Promise.any` from scratch — no using the native versions. I want to see that you actually understand the settlement semantics of each, not just that you can make the happy path work."

## Clarifying Questions

- **Do these need to accept any iterable, or just arrays?** The real `Promise.all` etc. accept any iterable (not just arrays) — I'd implement against arrays for simplicity unless asked to generalize, but I'd name that as a scoping decision up front rather than silently narrowing the spec.
- **Do the inputs need to be actual Promises, or can they be a mix of promises and plain values?** The native versions accept *any* value in the iterable — non-promise values are treated as already-resolved. I need `Promise.resolve(item)` to normalize each input before attaching `.then()`, otherwise a plain value like `5` (which has no `.then()` method) breaks the implementation.
- **What's the expected behavior on an empty input array for each of the four?** This is worth stating explicitly rather than discovering by accident: `Promise.all([])` resolves immediately with `[]`; `Promise.allSettled([])` resolves immediately with `[]`; `Promise.race([])` never settles at all (it just hangs forever — that's correct, spec-defined behavior, not a bug); `Promise.any([])` rejects immediately with an `AggregateError` (no candidate could possibly fulfill).
- **Does `Promise.any`'s `AggregateError` need to be the real global `AggregateError`, or a close approximation?** I'd use the real one if the environment provides it (Node 15+, all modern browsers), with a small fallback shape (`{errors: [...]}`) if it doesn't — but I'd default to assuming it's available.

## Approach & Trade-offs

All four share a common shape: wrap every input through `Promise.resolve(item)` to normalize non-promise values into promises, then attach a `.then(onFulfilled, onRejected)` to each and drive a **new**, manually-constructed `Promise` (via the `Promise` constructor's `executor(resolve, reject)` pattern) that settles according to each method's specific rule. What differs between the four is *when* the outer promise settles and *what shape* the settled value takes — that's the whole design space.

**The single most important shared implementation detail:** for `all` and `allSettled`, results must be written into a **pre-sized results array by index**, not accumulated via `.push()`. Because the input promises can resolve in any order (a later-index promise can settle before an earlier-index one), collecting results via `.push()` inside each `.then()` callback would produce an array ordered by *resolution time*, not by *input position* — which violates the actual contract both methods guarantee (results always correspond to input order, regardless of which one finished first).

**The empty-array edge case is a common structural bug, not just a trivia fact.** A common `all`/`allSettled` implementation that starts a `remaining` counter at `promises.length` and resolves the outer promise once `remaining` hits `0` inside each `.then()` callback will, for an empty input array, never have any `.then()` callback fire at all — meaning the counter starts at `0` but the "resolve when it hits 0" logic never actually runs, because it's *only* checked from inside a callback that, for zero inputs, never gets scheduled. The fix is checking the empty case explicitly, up front, before attaching any `.then()` handlers.

## Solution

```javascript
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    const items = Array.from(promises);
    if (items.length === 0) {
      resolve([]);
      return;
    }

    const results = new Array(items.length);
    let remaining = items.length;

    items.forEach((item, index) => {
      Promise.resolve(item).then(
        (value) => {
          results[index] = value; // BY INDEX — preserves input order regardless of resolution order
          remaining -= 1;
          if (remaining === 0) resolve(results);
        },
        (reason) => reject(reason) // first rejection short-circuits everything
      );
    });
  });
}
```

```javascript
function promiseAllSettled(promises) {
  return new Promise((resolve) => {
    const items = Array.from(promises);
    if (items.length === 0) {
      resolve([]);
      return;
    }

    const results = new Array(items.length);
    let remaining = items.length;

    items.forEach((item, index) => {
      Promise.resolve(item).then(
        (value) => {
          results[index] = { status: 'fulfilled', value };
          remaining -= 1;
          if (remaining === 0) resolve(results);
        },
        (reason) => {
          results[index] = { status: 'rejected', reason };
          remaining -= 1;
          if (remaining === 0) resolve(results); // note: resolve, never reject
        }
      );
    });
  });
}
```

```javascript
function promiseRace(promises) {
  return new Promise((resolve, reject) => {
    // No counting needed — whichever settles first (fulfilled OR rejected) wins,
    // and forwards directly. The rest are simply ignored once the outer promise settles.
    for (const item of promises) {
      Promise.resolve(item).then(resolve, reject);
    }
  });
}
```

```javascript
function promiseAny(promises) {
  return new Promise((resolve, reject) => {
    const items = Array.from(promises);
    if (items.length === 0) {
      reject(new AggregateError([], 'All promises were rejected'));
      return;
    }

    const errors = new Array(items.length);
    let remaining = items.length;

    items.forEach((item, index) => {
      Promise.resolve(item).then(
        (value) => resolve(value), // first FULFILLMENT wins — reject the rest silently
        (reason) => {
          errors[index] = reason;
          remaining -= 1;
          // Only reject once EVERY input has rejected — not on the first one.
          if (remaining === 0) {
            reject(new AggregateError(errors, 'All promises were rejected'));
          }
        }
      );
    });
  });
}
```

> **Check yourself:** Why does `promiseAll` write into `results[index]` instead of `results.push(value)` — construct a concrete example with three promises where `.push()` would produce a visibly wrong result.

## Gotchas

**Using `.push()` instead of index assignment for `all`/`allSettled`.** If `promises[0]` takes 100ms to resolve, `promises[1]` takes 10ms, and `promises[2]` takes 50ms, resolution order is `[1], [2], [0]` — a `.push()`-based collector would produce `results = [value1, value2, value0]`, silently reordered relative to the caller's input array. This is exactly wrong for `Promise.all`'s actual contract (`results[i]` corresponds to `promises[i]`, always) and is the single most common bug in hand-written implementations.

**The empty-array edge case, structurally.** As detailed above — a counter-based "resolve when `remaining === 0`" implementation that only checks the counter from *inside* a `.then()` callback will never resolve for an empty input, because with zero items, no callback is ever scheduled to perform that check. This has to be handled as an explicit early-return before the loop, not left to "the loop naturally handles zero iterations correctly" (it doesn't, for this specific pattern).

**Confusing `any`'s rejection behavior with `all`'s.** A very common mistake: implementing `any` by rejecting on the *first* rejected input (that's `all`'s rule, inverted incorrectly) instead of only rejecting after *every* input has rejected. `any`'s entire purpose is "give me whichever succeeds first, and only give up if literally everything fails" — rejecting early defeats that purpose (e.g., `any([fastFailingPromise, slowSucceedingPromise])` should still resolve with the slow success, not reject early because of the fast failure).

**Forgetting to normalize non-promise values with `Promise.resolve(item)`.** If the input array contains a plain value (e.g., `[fetchData(), 5, anotherFetch()]`), calling `.then()` directly on `5` throws (`5.then is not a function`). `Promise.resolve(item)` handles both cases uniformly — wrapping a non-promise value in an already-resolved promise, and passing an existing promise through unchanged (well, technically wrapping it in a way that's externally indistinguishable from passing it through).

**`Promise.race([])` never settling is correct behavior, not a bug to "fix."** A candidate who adds special-case handling to make `race` resolve/reject on an empty array (e.g., resolving with `undefined`) is deviating from the actual spec — an empty race genuinely has no candidate that could possibly "win," so it correctly hangs forever, matching the real `Promise.race`'s documented behavior.

**A classic `var`-in-a-loop closure bug when hand-rolling the index tracking.** Using `var index` in a `for` loop (rather than `forEach`'s per-iteration parameter, or a `let`-scoped `for` loop) and referencing `index` inside an async `.then()` callback captures the *same* `var` binding across all iterations — by the time any callback actually fires, the loop has finished and `index` holds its final value for every callback, corrupting every result into the same slot. This is a deservedly famous JavaScript gotcha independent of promises specifically, but it surfaces here naturally.

## Follow-up Questions

**Q (High): Implement `Promise.all`. Walk through why you must track results by index instead of pushing, given promises can resolve out of order.**

Answer: Each input is wrapped with `Promise.resolve(item)` and given a `.then()` handler that, on fulfillment, writes the resolved value into a pre-allocated `results` array at that promise's *original* index — not via `results.push(value)`. This matters because the promises in the input array can settle in any order relative to each other (a network call that happens to be fast can resolve before one that started earlier but is slower), and `Promise.all`'s contract guarantees the output array's order matches the *input* array's order, unconditionally, regardless of completion timing. A `.push()`-based collector's output order instead reflects *completion* order, which — for any input array whose promises don't happen to resolve in the same order they were listed — produces a result array that's silently, structurally wrong (right values, wrong positions), which is worse than an obvious crash because it can pass casual testing with fast, uniformly-ordered mock promises and only fail with real, variably-latent network calls.

The trap: testing only with mock promises that all resolve near-instantly and in the same order they're listed — this hides the exact bug the by-index-vs-push distinction exists to catch, since near-simultaneous resolution rarely exposes ordering issues; a correct test needs promises with deliberately staggered, out-of-order resolution times (e.g., via different `setTimeout` delays) to actually exercise the bug.

---

**Q (High): What's the difference in rejection behavior between `Promise.all` and `Promise.allSettled`? When would you use each?**

Answer: `Promise.all` **short-circuits** — the outer promise rejects as soon as the *first* input promise rejects, immediately, without waiting for any of the other still-pending promises to settle (they continue running in the background, but their eventual results/errors are simply never observed by this particular `Promise.all` call). `Promise.allSettled` **never rejects** — it always resolves, once every input has settled one way or the other, with an array where each entry explicitly records whether that specific input fulfilled (`{status: 'fulfilled', value}`) or rejected (`{status: 'rejected', reason}`). Use `all` when the operation is only meaningful if *everything* succeeds and you want to fail fast the moment any part fails (e.g., loading three required pieces of data to render a page — if any one is missing, there's no point waiting for the others). Use `allSettled` when partial success is meaningful and you need to know the outcome of *every* operation regardless of whether others failed (e.g., sending analytics events to five different tracking providers — one provider being down shouldn't prevent you from knowing which of the other four succeeded).

The trap: using `Promise.all` for a batch of independent operations where partial failure is actually fine, and then being surprised that one failing item causes the *entire* batch's results to be discarded (since `all` rejects and gives you nothing, not "the results of everything that succeeded plus an error for the one that didn't") — that's precisely the gap `allSettled` exists to fill.

---

**Q (High): Implement `Promise.race`. How does it differ from `Promise.any` in terms of what "wins"?**

Answer: `Promise.race` settles — resolving or rejecting — based on whichever input promise settles **first**, period, regardless of whether that first settlement is a fulfillment or a rejection; it simply forwards that first outcome (value or reason) directly to the outer promise. `Promise.any` specifically waits for the first **fulfillment** — a rejection from any individual input is *not* enough to settle `any`; it only rejects the outer promise once **every single input** has rejected, at which point it produces an `AggregateError` collecting all of the rejection reasons. Implementation-wise, `race` is a plain `.then(resolve, reject)` on every input with no counting logic at all — whichever calls `resolve`/`reject` on the outer promise first simply wins, and the `Promise` constructor guarantees only the first call to either has any effect (subsequent calls are silently no-ops); `any` requires a counter tracking how many inputs have rejected so far, only calling the outer `reject` once that counter reaches the total input count.

The trap: conflating the two — "race" sounds like it should mean "first successful one wins," which is actually `any`'s behavior; `race`'s actual semantics (first to settle *at all*, win or lose) are less intuitive from the name alone and worth stating explicitly to avoid confusing them under interview pressure.

---

**Q (Medium): How does `Promise.any` handle rejections, and what is `AggregateError`? Implement it.**

Answer: `Promise.any` must track every rejection it receives (rather than acting on the first one) because it needs to distinguish "some inputs failed but at least one is still pending or has succeeded" (not yet a final outcome) from "literally all inputs have failed" (the only condition under which `any` itself should reject). It does this with a counter (or an equivalent) tracking how many rejections have been observed, incrementing on each rejected input, and only calling the outer promise's `reject` once that count equals the total number of inputs — at which point it constructs an `AggregateError`, a built-in `Error` subtype (added alongside `Promise.any` itself) whose `.errors` property holds an array of all the individual rejection reasons, in the same order as the original input array, giving the caller visibility into *why* every candidate failed rather than just knowing that they all did.

```javascript
try {
  await promiseAny([fetchA(), fetchB(), fetchC()]);
} catch (err) {
  console.log(err instanceof AggregateError); // true
  console.log(err.errors); // [reasonA, reasonB, reasonC] — all three, in order
}
```

The trap: implementing "reject on the first failure" (that's `all`'s behavior) instead of correctly waiting for *all* inputs to fail — this is the single most common `any` implementation bug, precisely because it's the same shape of logic as `all`'s rejection handling, just needing to be triggered on the opposite outcome (rejection count reaching total, not fulfillment count).

---

**Q (Medium): What happens with `Promise.all`/`race`/`any` on an empty array — what's the spec behavior, and why does a naive counter-based implementation get it wrong?**

Answer: Per spec: `Promise.all([])` resolves immediately with `[]` (vacuously, "all of zero promises" are trivially all fulfilled); `Promise.allSettled([])` likewise resolves immediately with `[]`; `Promise.race([])` **never settles** — there's no candidate that could possibly be "first," so the returned promise simply stays pending forever, which is correct, spec-defined, and not something to "fix"; `Promise.any([])` **rejects immediately** with an `AggregateError` containing zero errors, since there's no candidate that could possibly fulfill. The reason a naive counter-based implementation (`let remaining = items.length; ...resolve when remaining === 0...`) silently breaks on an empty array specifically is structural: the "resolve when the counter hits zero" check typically lives *inside* the `.then()` callback attached to each item — but with zero items, that callback is never attached to anything and therefore never runs, so the check that would trigger resolution never executes at all, even though the counter technically already started at zero. The outer promise is left permanently pending — silently hanging, with no error, which is a particularly hard bug to notice in casual testing (it doesn't crash; it just never resolves).

The trap: assuming a `remaining === 0` check "naturally" covers the zero-input case because zero trivially equals zero — the bug isn't in the *value* of the check, it's in the check never being *reached* at all when there's no callback to run it from; the fix requires an explicit early-return for the empty case, checked *before* entering the loop that attaches callbacks.

---

**Q (Low): How do you make your polyfill accept any iterable, not just arrays, and handle non-promise values mixed in?**

Answer: Accepting any iterable (not just arrays) means the input parameter should be consumed via `Array.from(iterable)` (or spread, `[...iterable]`) up front, which works uniformly for arrays, `Set`s, `Map` values/entries, generator results, or any other object implementing the iterable protocol (`Symbol.iterator`) — converting to a concrete array once at the start also conveniently gives you `.length` and index access for the rest of the implementation, rather than needing to handle iterables specially throughout. Handling non-promise values mixed into that iterable is what `Promise.resolve(item)` already solves uniformly — it's specified to return its argument unchanged if the argument is already a genuine native Promise, and to wrap it in a new, already-fulfilled promise otherwise (including for "thenables" — objects with a `.then()` method that aren't true Promise instances, which get properly assimilated rather than just wrapped opaquely).

The trap: trying to special-case "is this a Promise" with an `instanceof Promise` check and branching logic, rather than uniformly calling `Promise.resolve(item)` on every item regardless of type — the uniform approach is both simpler and more correct, since it also correctly handles thenables from other Promise implementations/polyfills that wouldn't pass an `instanceof Promise` check but are still meant to be treated as promise-like.

---

**Q (Low): How would you implement a `Promise.allSettled`-like utility with a concurrency limit — capping the number of simultaneously in-flight promises?**

Answer: This changes the problem from "start every input immediately and wait for all of them" to "start only `N` at a time, and start the next one as soon as any currently-running one finishes" — which means the input can no longer be a pre-built array of already-started promises (since a promise begins executing the moment it's created, not when you `.then()` it); instead, the input needs to be an array of promise-returning *functions* (thunks), so execution can genuinely be deferred until a concurrency slot is free. A common implementation runs `N` "worker" loops concurrently (via `Promise.all` over `N` async functions), where each worker repeatedly pulls the next not-yet-started thunk from a shared index/queue, awaits it, records its settled result at the correct original index, and loops until the shared queue is exhausted — naturally capping in-flight work at `N` without any explicit semaphore/counting logic beyond "how many worker loops did I start."

The trap: trying to retrofit concurrency limiting onto the existing `allSettled` implementation by keeping the input as already-created promises and only "waiting" on `N` at a time — this doesn't actually work, because by the time you have promise *objects* in hand, whatever they represent (e.g., a fetch call) has already started executing; true concurrency limiting requires deferring the *start* of each unit of work, which means the input has to be functions/thunks, not promises.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement `Promise.all` with correct by-index result ordering, from memory
- [ ] Can implement `Promise.allSettled` and explain why it never rejects
- [ ] Can implement `Promise.race` and explain why it needs no counting logic at all
- [ ] Can implement `Promise.any` and explain why it rejects only after *all* inputs reject, using `AggregateError`
- [ ] Can state, precisely, the empty-array behavior for all four and why the counter-based bug happens
- [ ] Can explain why `Promise.resolve(item)` is needed to normalize mixed promise/non-promise inputs

---
*Next: Custom Promise From Scratch — having polyfilled the combinators, the natural next step is building `Promise` itself (states, `.then()` chaining, microtask scheduling) from first principles.*
