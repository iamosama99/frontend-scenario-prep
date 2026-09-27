# Custom Promise From Scratch

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| States are one-way | `pending → fulfilled` or `pending → rejected`, never back, never both | Once settled, a promise's outcome is immutable — this is what makes `.then` callbacks safe to call more than once without re-running side effects |
| Callbacks always run async | Queue callbacks and flush via `queueMicrotask` (or `setTimeout(fn, 0)` fallback) even if already settled | `.then` must never call its callback synchronously — otherwise execution order becomes caller-dependent and unpredictable, breaking the guarantee that `.then(fn)` runs after the current synchronous stack |
| `.then` returns a new promise | Chaining works by creating and returning a *fresh* `MyPromise` from every `.then` call | This is what allows `.then().then().then()` to chain — each link's resolution depends on unwrapping whatever the previous callback returned |
| Thenable/promise unwrapping | If a callback returns another promise (or thenable), recursively adopt its eventual state instead of resolving to the promise object itself | Prevents `Promise<Promise<value>>` nesting — this is the trickiest and most-tested part of a correct implementation |

## The Scenario

"Implement a `MyPromise` class from scratch — no using the native `Promise` internally. It needs to support `.then(onFulfilled, onRejected)`, `.catch()`, and `.finally()`, and needs to behave correctly when a `.then` handler is attached *after* the promise has already settled, and when a handler returns another promise."

## Clarifying Questions

- **Does it need to pass the full Promises/A+ spec, or just cover the common interview-tested behaviors?** The full spec has ~872 conformance tests covering edge cases like thenables with getters that throw, or calling `resolve` and `reject` multiple times. For an interview, I'd confirm the bar is "correct core semantics" (states, async callback execution, chaining, thenable adoption) rather than full spec compliance, and mention what I'm consciously simplifying.
- **Should `.then` support being called multiple times on the same promise?** Yes, implicitly — a promise can have many `.then` handlers attached (e.g., multiple parts of an app awaiting the same fetch), so the internal callback storage needs to be an array, not a single slot.
- **What happens if `resolve` is called after the promise already settled, or if both `resolve` and `reject` are called?** Native promises make settlement a one-time, first-call-wins event — every call after the first no-ops. I'd implement that explicitly rather than let it be undefined behavior, since it's a common way executor code accidentally resolves twice (e.g., in a `try/catch` alongside a callback-based API).
- **Should the executor's synchronous throw be caught and treated as a rejection?** Native `Promise` wraps the executor in an implicit `try/catch` — if the executor throws synchronously, the promise rejects with that error rather than the throw propagating up and crashing the caller. I'd implement this since it's core to why `new Promise((resolve, reject) => { throw new Error() })` is a documented, commonly-relied-on pattern.

## Approach & Trade-offs

The core design has four pieces: a state machine, a callback queue, an async execution mechanism, and value-unwrapping logic for `.then`'s return value.

1. **State**: three possible values — `pending`, `fulfilled`, `rejected` — plus a `value`/`reason` slot. Transition is one-directional and one-time: once `state !== 'pending'`, further `resolve`/`reject` calls are ignored. I store this as a single `state` string plus a `value` field (used for both the fulfillment value and rejection reason, discriminated by `state`) rather than separate value/error fields, to make the "only one is meaningful" invariant structural rather than convention-based.

2. **Callback storage as arrays, not single slots**: because `.then` can be called multiple times on the same promise (before or after settlement), I keep `onFulfilledCallbacks` and `onRejectedCallbacks` as arrays. When the promise settles, every queued callback fires, in registration order.

3. **The "settle now vs. settle later" branch inside `.then`**: this is the crux of the implementation. If the promise is still `pending` when `.then` is called, the callback is queued and will run later, when `resolve`/`reject` eventually fires. If the promise has *already* settled by the time `.then` is called, the callback can't run synchronously right there — it still has to be deferred to a microtask, otherwise `promise.then(cb)` would sometimes run `cb` synchronously (already-settled case) and sometimes asynchronously (still-pending case), which is exactly the inconsistent-timing bug the Promise spec exists to prevent. So both branches end up scheduling the callback via `queueMicrotask`, just from different trigger points.

4. **`.then` returns a new promise, and the return value gets "unwrapped"**: this is what makes chaining and thenable-adoption work. `.then(onFulfilled)` doesn't just call `onFulfilled` and stop — it returns a *new* `MyPromise`, and that new promise's resolution depends on what `onFulfilled` returns:
   - If `onFulfilled` returns a plain value, the new promise resolves with that value.
   - If `onFulfilled` returns another `MyPromise` (or any thenable — object with a `.then` method), the new promise must *adopt* that promise's eventual state, not resolve with the promise object itself — recursively, since the returned promise could itself resolve to another promise.
   - If `onFulfilled` throws, the new promise rejects with the thrown error.
   
   I chose to write a dedicated `resolvePromise(newPromise, returnedValue, resolve, reject)` helper for this rather than inlining it, because the same unwrapping logic is needed in three places (`.then`'s fulfillment path, its rejection path, and technically nowhere else since `.catch`/`.finally` are implemented in terms of `.then`) — and because getting this one function exactly right is genuinely the hardest part, isolating it makes it easier to reason about and test independently.

5. **`.catch` and `.finally` as thin wrappers over `.then`**: `.catch(onRejected)` is just `.then(undefined, onRejected)` — no new logic needed. `.finally(onFinally)` is more subtle: it needs to run `onFinally` regardless of outcome, *without* seeing the value/reason (its return value is ignored for resolution purposes unless it throws or returns a rejected promise, which propagates), and it needs to pass the original value/reason through unchanged to the next link in the chain — I implement this by wrapping in `.then(value => { onFinally(); return value }, reason => { onFinally(); throw reason })`.

I chose to build execution ordering on `queueMicrotask` rather than `setTimeout(fn, 0)` because that's what real promises use (microtask queue, not macrotask/timer queue) — using `setTimeout` would give the wrong relative ordering versus real promises and versus `async/await` if the two were ever mixed in the same test, which is exactly the kind of subtle-but-checkable detail that separates a working toy from a spec-faithful one.

## Solution

```javascript
const PENDING = 'pending';
const FULFILLED = 'fulfilled';
const REJECTED = 'rejected';

class MyPromise {
  #state = PENDING;
  #value = undefined;
  #onFulfilledCallbacks = [];
  #onRejectedCallbacks = [];

  constructor(executor) {
    const resolve = (value) => this.#settle(FULFILLED, value);
    const reject = (reason) => this.#settle(REJECTED, reason);

    try {
      executor(resolve, reject);
    } catch (err) {
      // Executors that throw synchronously reject the promise — matches native behavior.
      reject(err);
    }
  }

  #settle(state, value) {
    // First settlement wins; subsequent resolve/reject calls are no-ops.
    if (this.#state !== PENDING) return;

    // If resolve(thenable) is called, adopt the thenable's eventual state
    // instead of settling with the thenable object itself.
    if (state === FULFILLED && value && (typeof value === 'object' || typeof value === 'function') && typeof value.then === 'function') {
      value.then(
        (v) => this.#settle(FULFILLED, v),
        (r) => this.#settle(REJECTED, r)
      );
      return;
    }

    this.#state = state;
    this.#value = value;

    const callbacks = state === FULFILLED ? this.#onFulfilledCallbacks : this.#onRejectedCallbacks;
    callbacks.forEach((cb) => queueMicrotask(cb));
    this.#onFulfilledCallbacks = [];
    this.#onRejectedCallbacks = [];
  }

  then(onFulfilled, onRejected) {
    // Non-function handlers are "transparent" — value/reason passes through unchanged.
    const handleFulfilled = typeof onFulfilled === 'function' ? onFulfilled : (v) => v;
    const handleRejected = typeof onRejected === 'function' ? onRejected : (r) => { throw r; };

    return new MyPromise((resolve, reject) => {
      const runFulfilled = () => {
        try {
          resolve(handleFulfilled(this.#value));
        } catch (err) {
          reject(err);
        }
      };
      const runRejected = () => {
        try {
          resolve(handleRejected(this.#value)); // a caught rejection "recovers" into a resolution
        } catch (err) {
          reject(err);
        }
      };

      if (this.#state === FULFILLED) {
        queueMicrotask(runFulfilled);
      } else if (this.#state === REJECTED) {
        queueMicrotask(runRejected);
      } else {
        // Still pending — queue for whenever settle() eventually fires.
        this.#onFulfilledCallbacks.push(runFulfilled);
        this.#onRejectedCallbacks.push(runRejected);
      }
    });
  }

  catch(onRejected) {
    return this.then(undefined, onRejected);
  }

  finally(onFinally) {
    return this.then(
      (value) => { onFinally(); return value; },
      (reason) => { onFinally(); throw reason; }
    );
  }

  static resolve(value) {
    if (value instanceof MyPromise) return value;
    return new MyPromise((resolve) => resolve(value));
  }

  static reject(reason) {
    return new MyPromise((_, reject) => reject(reason));
  }
}
```

```javascript
new MyPromise((resolve) => setTimeout(() => resolve(1), 100))
  .then((v) => v + 1)
  .then((v) => new MyPromise((resolve) => setTimeout(() => resolve(v + 1), 50))) // returns a promise
  .then((v) => console.log(v)); // logs 3, after ~150ms total

new MyPromise((_, reject) => reject('boom'))
  .then((v) => console.log('never runs', v))
  .catch((err) => console.log('caught:', err)); // logs "caught: boom"

console.log('sync code runs first'); // always logs before any .then callback, even for an already-resolved promise
```

> **Check yourself:** Why does `.then` need to schedule via `queueMicrotask` even in the branch where the promise has *already* settled, instead of just calling the handler synchronously in that case?

## Why `resolve`/`reject` Adopt Thenables Recursively

The subtlest correctness requirement is in `#settle`: when `resolve(value)` is called and `value` is itself thenable (has a callable `.then`), the promise must not settle to `FULFILLED` with that thenable as its value — it must instead wait for the thenable to settle, and adopt *that* result. This has to be recursive because the thenable could itself resolve with another thenable, arbitrarily deep (`resolve(promiseA)` where `promiseA` resolves to `promiseB`, which resolves to `promiseC`, which resolves to `42` — the outer promise should eventually settle with `42`, not with `promiseA`). The implementation above gets this "for free" because `#settle` calls itself (via `value.then(v => this.#settle(FULFILLED, v), ...)`), so each layer of thenable-wrapping triggers another pass through the same unwrapping check.

This is also exactly why `Promise.resolve(Promise.resolve(Promise.resolve(42)))` flattens to a promise that resolves to `42`, not a triple-nested promise — a detail that trips up anyone who assumes `Promise.resolve` on an existing promise "wraps" it another level.

## Gotchas

**Calling handlers synchronously when the promise is already settled.** The most common broken implementation checks `if (state === FULFILLED) { onFulfilled(value); }` directly inside `.then`, without a `queueMicrotask` wrapper. This looks correct in simple tests but breaks the invariant that `.then` callbacks *never* run before the current synchronous execution context finishes — code like `promise.then(() => console.log('a')); console.log('b');` must always log `b` then `a`, regardless of whether the promise was already settled when `.then` was called.

**Forgetting that `resolve`/`reject` are one-time events.** An executor that calls `resolve(1)` and then later (e.g., in a `catch` block or a delayed callback) calls `reject(err)` must have the second call silently ignored — the promise already settled as fulfilled with `1`. Implementations that don't guard `#settle` with a `state !== PENDING` check will let a later call overwrite an earlier settlement, which native promises never do.

**Not catching synchronous throws inside the executor.** `new Promise((resolve, reject) => { throw new Error('x') })` must produce a *rejected* promise, not an uncaught exception that crashes the surrounding code. This requires wrapping the executor invocation itself in `try/catch` inside the constructor — easy to forget since it's not inside `.then` or `#settle` where most of the attention goes.

**Resolving with a thenable without recursively adopting its state.** `resolve(anotherPromise)` must make the outer promise track the *inner* promise's eventual outcome, not immediately fulfill with the inner promise object as the value — otherwise `.then(v => v.then(...))` becomes necessary at every call site, defeating the entire purpose of chaining.

**`.finally` accidentally consuming or transforming the value.** A naive `.finally` implementation like `then(onFinally, onFinally)` runs the callback but also (a) passes the settled value/reason *into* `onFinally` when the callback signature doesn't expect an argument, and (b) whatever `onFinally` returns becomes the new chain's value, silently replacing the original one. The correct version explicitly re-returns (or re-throws) the original value/reason after calling `onFinally`, ignoring its return value except when it throws or returns a rejected promise (which does propagate, per spec).

## Follow-up Questions

**Q (High): Why must `.then` callbacks always execute asynchronously, even when the promise has already settled by the time `.then` is called?**

Answer: Consistency of execution order is the guarantee being protected. If `.then`'s timing depended on whether the promise was already settled — synchronous when settled, asynchronous when pending — then the exact same calling code could behave differently purely based on a timing race the caller doesn't control (e.g., whether a network request happened to resolve before or after `.then` was attached). Promises solve "callback hell" partly by making ordering *predictable*: all `.then` callbacks are guaranteed to run after the current synchronous stack completes, full stop, regardless of when the promise actually settled. This is enforced via the microtask queue — `queueMicrotask` (or historically, an internal job queue) — which always drains after the current synchronous execution but before the event loop moves to the next macrotask (timers, I/O, rendering).

The trap: implementing the "already settled" branch of `.then` as a direct synchronous call to the handler, which passes naive tests (single `.then` call, nothing else happening concurrently) but breaks under composition — e.g., logging statements interleave unpredictably depending on promise state at attach time, which is a real, hard-to-debug production bug class if it ever leaked into actual promise-polyfill code.

---

**Q (High): How does promise chaining actually work — specifically, why does `.then` need to return a *new* promise rather than the same one?**

Answer: Each `.then` call represents one more transformation step, and each step can have a different outcome than the one before it (the handler might return a new value, throw, or return another promise) — so each step needs its own independent state machine to track "has *this* step's result been determined yet." If `.then` returned `this` (the original promise), every link in the chain would share one state, and the chain couldn't represent "step 1 succeeded but step 2's handler threw" as a distinct outcome from step 1's outcome. Returning a fresh `MyPromise` from `.then`, whose resolution is driven by calling the handler and feeding its result (unwrapped, if thenable) into that new promise's `resolve`, is what makes `.then().then().then()` behave as a sequential pipeline where each stage can independently succeed, fail, or hand off to another async operation.

The trap: describing chaining vaguely as "it just calls the next `.then`" without being able to explain that the mechanism is specifically "each `.then` allocates a new promise and resolves/rejects it based on the current handler's outcome" — that's the part that explains why returning a promise from inside a `.then` handler correctly "flattens" instead of creating nested promises.

---

**Q (High): What happens if a `.then` handler returns another promise, and how do you implement that unwrapping correctly?**

Answer: The new promise created by `.then` must not resolve with the returned promise as its value — it must recursively adopt whatever that returned promise eventually resolves or rejects with. Mechanically, this means: when the handler's return value is thenable, call `.then()` on it with callbacks that in turn resolve/reject the *outer* new promise — and because that returned promise's own resolution could itself be another thenable, the same check needs to apply again at that point, which naturally happens if `resolve`/`#settle` itself contains the thenable check (as in the reference implementation above), rather than only checking once at the top level of `.then`.

The trap: checking `if (result instanceof MyPromise)` only in `.then` and forgetting that `resolve()` itself (called from anywhere, including a plain executor) can also receive a thenable and needs the same unwrapping — the check belongs in the single choke point where values become "the settled value" (i.e., inside `#settle`/`resolve`), not scattered across every call site that might produce a value.

---

**Q (Medium): How would `Promise.all` be implemented on top of this `MyPromise` class?**

Answer: `Promise.all(promises)` returns a new promise that resolves with an array of all results, in the original order, once every input promise fulfills — or rejects immediately with the reason of the *first* promise that rejects. Implementation: wrap each input in `MyPromise.resolve(p)` (to tolerate plain values mixed with promises), track a results array pre-sized to the input length and a counter of remaining unsettled promises; for each promise at index `i`, call `.then(value => { results[i] = value; if (--remaining === 0) resolve(results); }, reject)`. Storing by index (not pushing) is what preserves original order despite promises settling in arbitrary completion order — a common bug is pushing to a results array as promises resolve, which produces results in *completion* order rather than *input* order.

The trap: forgetting the by-index assignment and using `.push()` instead, which passes tests where all promises take equal time but silently scrambles output order the moment resolution times vary — exactly the situation `Promise.all` exists to handle correctly.

---

**Q (Medium): Why does the executor function run synchronously, immediately, inside the `Promise`/`MyPromise` constructor — unlike everything else about promises, which is asynchronous?**

Answer: The executor's job is to *initiate* the asynchronous work (start the timer, fire the network request, subscribe to the event) — that initiation needs to happen immediately so the async operation actually begins as soon as the promise is constructed, not deferred to some later microtask. Only the *settlement notification* (calling registered `.then` handlers) is deferred; the executor itself is plain synchronous code that happens to usually contain calls to genuinely async APIs (`setTimeout`, `fetch`, etc.) — but even a fully synchronous executor like `new Promise(resolve => resolve(1))` runs its body immediately, and it's specifically the *handler invocation* from `.then` that's queued as a microtask, not the executor.

The trap: assuming "promises are asynchronous" applies uniformly to every part of a `Promise` — it's specifically the resolution/handler-calling machinery that's deferred; the executor body itself executes eagerly and synchronously the moment `new Promise(...)` runs, which is why `console.log` statements placed directly in an executor (outside any callback) appear before any `.then` output.

---

**Q (Low): How would you add `Promise.race` and `Promise.any` to this implementation, and what's the semantic difference between them?**

Answer: `Promise.race(promises)` settles (fulfilled or rejected) as soon as the *first* input promise settles, with that promise's outcome — implemented by attaching `.then(resolve, reject)` to every input promise and letting whichever fires first determine the outer promise's fate (later settlements are no-ops since the outer promise's own `#settle` already enforces first-call-wins). `Promise.any(promises)` is more selective: it resolves with the first *fulfillment*, but only rejects if *all* inputs reject — collecting rejection reasons into an `AggregateError` if every promise fails. Implementation: attach `.then(resolve, err => { errors[i] = err; if (--remaining === 0) reject(new AggregateError(errors)); })` to each — structurally similar to `Promise.all`'s counter pattern, but tracking rejections instead of fulfillments, and inverting which outcome triggers immediately versus which requires all-inputs-in.

The trap: implementing `Promise.any` as effectively `Promise.race` with a "skip rejections" comment bolted on, without tracking a rejection counter — that leaves no path to ever reject when every single input promise fails, silently leaving the returned promise pending forever instead of rejecting with an `AggregateError` as spec requires.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement a `MyPromise` class with correct state machine, async callback execution, and chaining from memory
- [ ] Can explain precisely why `.then` callbacks must always be deferred via the microtask queue, even for an already-settled promise
- [ ] Can explain why `.then` must return a new promise, and how thenable return values get recursively unwrapped
- [ ] Can implement `.catch` and `.finally` in terms of `.then` without look-up
- [ ] Can sketch `Promise.all`/`Promise.race`/`Promise.any` on top of this implementation
- [ ] Can explain why the executor runs synchronously while handler invocation does not

---
*Next: Retry With Exponential Backoff — moves from building the promise primitive itself to a common real-world consumer of promises: resilient retry logic for flaky async operations.*
