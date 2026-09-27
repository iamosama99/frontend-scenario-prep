# Event Loop Output Prediction — Tricky Async

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| One call stack, drained to empty first | All synchronous code runs to completion before any queue is touched | This is the anchor fact — no async callback, of any kind, runs while there's still synchronous code executing |
| Microtasks fully drain between each macrotask | After the call stack empties, **every** queued microtask runs (including ones queued by other microtasks) before the next macrotask starts | This is why promise chains can "starve" timers — a microtask that queues another microtask keeps winning against a pending `setTimeout` |
| Macrotask queue order ≠ delay order | `setTimeout(fn, 0)` doesn't mean "run immediately" — it means "run as a macrotask, after the current stack and all microtasks, and after the browser's own minimum-delay clamp" | A `setTimeout(fn, 0)` scheduled before a `Promise.resolve().then(fn2)` still runs *after* `fn2` — timer callbacks are always macrotasks, promise callbacks are always microtasks, and microtasks always win the race for "next thing to run" |
| `async`/`await` is sugar over promises + microtasks | Everything after an `await` is scheduled exactly like a `.then()` callback on that awaited value | An `async function`'s body runs synchronously up to the first `await`, then the rest resumes as a microtask — tracing `async/await` output requires mentally desugaring it back into `.then()` chains |

## The Scenario

"Here's a snippet of code mixing `setTimeout`, native promises, and `async/await`. Before running it, tell me exactly what gets logged, and in what order — and explain *why*, referencing the call stack, the microtask queue, and the macrotask (task) queue specifically."

## Clarifying Questions

Unlike the other machine-coding scenarios in this phase, there's no code to write here — the interviewer hands you a snippet and expects a trace. The "clarifying questions" that matter are about the snippet itself and the runtime, not requirements:

- **Is this running in a browser or in Node.js?** The core model (call stack → microtasks → macrotasks) is the same, but the *macrotask queue specifics* differ: browsers interleave rendering/other task sources between macrotasks in ways Node doesn't, and Node has additional queue types (`process.nextTick`, which runs *before* other microtasks including promise callbacks, and separate phases for timers/I/O/`setImmediate`) that don't exist in browsers at all. If a snippet includes `process.nextTick` or `setImmediate`, confirming the runtime is essential — the answer genuinely differs.
- **Are there any promise rejections in the snippet, and if so, are they ever caught?** An unhandled rejection doesn't stop execution (unlike a synchronous throw), but it does produce a separate, asynchronously-reported warning/event (`unhandledrejection` in browsers, a process warning in Node) — worth explicitly noting if relevant rather than silently ignoring that a rejection occurred.
- **Does the snippet use `setTimeout` with a non-zero delay that's comparable to, or overlaps with, other async work's timing?** If delays are meaningfully different (e.g., 0 vs. 1000ms), the relative order is unambiguous. If delays are the same or the snippet doesn't specify, I'd note that among multiple macrotasks with equal scheduled delay, they run in the order they were *scheduled*, not some other tiebreak.

## Approach & Trade-offs

Tracing this kind of snippet reliably comes down to mentally running a specific, fixed algorithm rather than "reading it like normal code top to bottom." The algorithm:

1. **Run all synchronous code first, top to bottom, exactly as written** — including the synchronous portions of `async function` bodies (everything before their first `await`) and executor functions passed to `new Promise(...)` (which run **immediately and synchronously**, not deferred). Anything that gets logged during this pass happens before *anything* else, no exceptions.
2. **While running that synchronous code, whenever a `.then`/`.catch`/`.finally` callback, an `async function`'s post-`await` continuation, or a `queueMicrotask` callback gets scheduled, it goes into the microtask queue** — it does not run yet, no matter how "immediate" it looks (`Promise.resolve().then(fn)` does not run `fn` synchronously, ever).
3. **Whenever `setTimeout`/`setInterval` schedules a callback, it goes into the macrotask (task) queue**, tagged with its target fire time — it is *never* eligible to run until the call stack is empty **and** the entire microtask queue has been fully drained.
4. **Once the initial synchronous run finishes (call stack empty), drain the microtask queue completely** — run the oldest queued microtask, and if running it schedules *more* microtasks, those get appended to the same queue and also run before moving on — the engine does not proceed to the macrotask queue until the microtask queue is entirely empty, however many rounds that takes.
5. **Only then, take the single oldest-eligible macrotask off the macrotask queue and run it** — and critically, after that *one* macrotask finishes, go back to step 4 and fully drain the microtask queue again (including any microtasks the macrotask itself just scheduled) before taking the *next* macrotask. Microtasks are drained between **every single** macrotask, not just once at the very end.
6. Repeat step 5 until both queues are empty.

**The single most load-bearing fact to internalize**: microtasks (promise callbacks, `async/await` continuations, `queueMicrotask`) always run before the next macrotask (any `setTimeout`/`setInterval` callback), *regardless of the order they were scheduled in relative to each other* — a `setTimeout(fn, 0)` scheduled first, followed by a `Promise.resolve().then(fn2)` scheduled second, still logs `fn2` before `fn`, every time, because the entire microtask queue is required to fully drain before the engine is even allowed to look at the macrotask queue.

**For `async/await` specifically**, the trace-friendly mental model is: mentally rewrite `await someValue` as `return someValue.then(rest_of_function_as_a_callback)`. Everything in the `async function` *before* the first `await` runs synchronously, immediately, as part of whatever called that function — this is a commonly missed detail (people assume the entire async function body is "deferred," when only the *continuation after `await`* is). Everything *after* an `await` runs as a microtask, scheduled at the point the awaited value settles.

## Worked Example

```javascript
console.log('1: sync start');

setTimeout(() => console.log('2: setTimeout'), 0);

Promise.resolve().then(() => console.log('3: promise 1'));

async function asyncFn() {
  console.log('4: asyncFn start (sync)');
  await null;
  console.log('5: asyncFn after await (microtask)');
}
asyncFn();

Promise.resolve().then(() => {
  console.log('6: promise 2');
}).then(() => {
  console.log('7: promise 2 chained');
});

console.log('8: sync end');
```

**Predicted output, traced step by step:**

1. **Synchronous pass** (nothing here waits, so it all runs top to bottom without interruption):
   - `console.log('1: sync start')` → logs `1`.
   - `setTimeout(...)` registers its callback in the macrotask queue with delay 0 — does *not* run yet.
   - `Promise.resolve().then(...)` registers its callback in the microtask queue — does *not* run yet.
   - `asyncFn()` is called: its body starts running *synchronously* — logs `4: asyncFn start (sync)` — then hits `await null`, which suspends the function and schedules its continuation (everything after the `await`) as a microtask. Control returns to the caller immediately; `asyncFn()`'s promise is still pending.
   - The second `Promise.resolve().then(...)` registers its callback in the microtask queue.
   - `console.log('8: sync end')` → logs `8`.
   - Call stack is now empty. Synchronous pass logged, in order: `1, 4, 8`.

2. **Drain microtask queue** (in the order those callbacks were queued: promise-1's `.then`, asyncFn's post-`await` continuation, promise-2's first `.then`, then whatever that `.then` itself schedules):
   - Promise 1's `.then` callback runs → logs `3: promise 1`.
   - `asyncFn`'s continuation runs → logs `5: asyncFn after await (microtask)`.
   - Promise 2's first `.then` callback runs → logs `6: promise 2`, and its return value schedules the *next* `.then` in the chain (`7`) as a **new** microtask, appended to the now-still-draining queue.
   - Since the microtask queue isn't considered empty until *no more* microtasks are pending (including ones just added), the just-scheduled continuation runs next → logs `7: promise 2 chained`.
   - Microtask queue is now fully empty.

3. **Macrotask queue**: only one macrotask is pending — the `setTimeout` callback. It runs → logs `2: setTimeout`.

**Final output order: `1, 4, 8, 3, 5, 6, 7, 2`.**

> **Check yourself:** Why does `6` log before `2`, even though the `setTimeout` was scheduled (in source order) *before* either promise chain — and why does `7` log before `2` as well, even though `7` wasn't even scheduled until *after* the synchronous pass had already finished?

## Why `setTimeout(fn, 0)` Doesn't Mean "Run Immediately"

This is the single most common misconception this scenario is designed to surface. `setTimeout(fn, 0)` schedules `fn` as a **macrotask** with a *requested* delay of 0ms — it does not mean "run right now" or even "run next." Two things stand between it and execution: first, the entire current synchronous call stack must finish; second — and this is the part people miss — the *entire* microtask queue must be fully drained, potentially across multiple rounds (as step 5 above shows, promise chains can keep re-populating the microtask queue with more work), before the event loop is even allowed to check the macrotask queue for the next thing to run. Additionally, browsers clamp minimum timer delay (historically 4ms after a certain nesting depth, per the HTML spec) — so "0ms" was never a literal guarantee even in isolation. The mental shift required is: `setTimeout(fn, delay)` schedules *a minimum time after which `fn` becomes eligible to run, once the stack and microtask queue are both clear* — not a guarantee of exactly when it runs.

## Gotchas

**Assuming promise executor code is deferred.** `new Promise((resolve, reject) => { console.log('runs sync!'); resolve(1); })` — the function passed to `new Promise(...)` runs **synchronously and immediately**, in the exact position it appears in the code. Only the `.then`/`.catch` *callbacks* attached to that promise are deferred to the microtask queue. A very common tracing mistake is assuming anything "promise-related" is automatically asynchronous.

**Assuming an `async function`'s entire body is deferred.** As shown in the worked example, everything before the first `await` runs synchronously, in place, the moment the function is called — it's only the code *after* `await` that gets deferred (as a microtask, scheduled once the awaited value settles). Calling an `async function` and assuming nothing happens until "later" misses any `console.log` calls or side effects positioned before its first `await`.

**Assuming microtasks queued during microtask-draining wait for the next "round."** If a `.then` callback itself calls another `.then` (chaining), or calls `queueMicrotask`, that newly queued microtask runs **before** the engine moves on to the macrotask queue — not after. The drain step doesn't run "one pass" of the queue as it existed at the start; it runs until the queue is *actually* empty, however many new entries get added along the way. This is exactly how a sufficiently adversarial promise chain can indefinitely "starve" a pending `setTimeout` from ever running (a real, if rare, gotcha in production: an unbounded recursive microtask scheduling loop can make timers never fire).

**Conflating `Promise.resolve().then(fn)` timing with `queueMicrotask(fn)` timing.** They're both microtasks and, for ordering purposes relative to `setTimeout`, behave identically — but they're not always perfectly interchangeable in every engine/edge case (e.g., `queueMicrotask` doesn't involve promise state machinery at all, which matters if you're specifically testing promise-chain-length-related behavior) — for the purposes of this kind of trace, though, treating them as equivalent-priority microtasks is correct.

**Forgetting Node.js has an additional, higher-priority queue: `process.nextTick`.** In Node specifically (not browsers), `process.nextTick(fn)` callbacks run **before** the regular microtask queue (promise callbacks) is processed, and Node also fully drains the `nextTick` queue between *each* microtask-queue item, not just once — a snippet mixing `process.nextTick` and promises in Node produces a different, more nuanced order than the same snippet using only promises, and treating them as equivalent is a common, specifically-Node mistake.

## Follow-up Questions

**Q (High): Explain, precisely, why microtasks always run before the next macrotask — what in the event loop's algorithm actually enforces this?**

Answer: The event loop's per-iteration algorithm is specifically: take one task (macrotask) from the macrotask queue and run it to completion; *then*, before doing anything else — before rendering, before considering the next macrotask — fully drain the microtask queue, including any microtasks newly added during that draining; only once the microtask queue is completely empty does the loop move on to select the next macrotask. This ordering is a deliberate design choice (not an accident of implementation) specifically so that promise-based code has a consistent, predictable point at which "all currently pending reactions have been processed" before anything else (a timer, another task) gets a chance to run and potentially observe inconsistent intermediate state. The consequence is that no matter how many microtasks get scheduled, and no matter how many *new* microtasks those schedule in turn, all of them execute before the earliest-queued macrotask gets its turn — macrotasks are only ever considered once the microtask queue has nothing left in it, however long that takes.

The trap: describing this as "microtasks just have higher priority" without being able to explain the actual mechanism (full draining, including newly-added items, happens *between every single macrotask*, not once globally) — the deeper, more precise version of the answer is what a strong candidate volunteers, since the shallow version invites a harder follow-up that exposes the gap.

---

**Q (High): If you have `setTimeout(fn, 0)` scheduled before `Promise.resolve().then(fn2)` in source order, which runs first, and why — and does this ever change?**

Answer: `fn2` always runs first, regardless of source order, because `setTimeout` always schedules a macrotask while `.then` always schedules a microtask, and the event loop guarantees the entire microtask queue drains before the next macrotask is even considered — source order of *scheduling* doesn't override queue-type priority. This is true even if the `setTimeout` delay were, hypothetically, negative or the browser somehow processed it "instantly" — it still has to wait for the current synchronous execution to finish and the microtask queue to fully empty first, and those two things are true of the promise callback but the promise callback doesn't additionally have to wait for the macrotask-queue-check step. The only way this relationship would flip is if `fn2`'s scheduling were itself somehow delayed past the point the macrotask queue was checked — which isn't possible in the scenario as stated, since both are scheduled synchronously and in the same synchronous pass.

The trap: reasoning about this by delay magnitude ("well, 0ms is basically instant, so maybe it's close") — the relevant fact isn't about *how small* the timeout delay is, it's a categorical distinction between two entirely different queues with a strict, delay-independent priority ordering between them.

---

**Q (Medium): How does Node.js's event loop differ from a browser's, specifically regarding `process.nextTick` and `setImmediate`?**

Answer: Node's event loop has more granular phases than a browser's (timers, pending callbacks, idle/prepare, poll, check, close callbacks), and layers two additional queue types on top of the standard microtask/macrotask model: `process.nextTick`'s queue, which is drained **before** the regular microtask (promise) queue, and drained again to completion after *each* microtask processed from that queue (making it even higher priority than promise callbacks) — and `setImmediate`, which schedules a callback for the "check" phase, conceptually similar to but distinct from `setTimeout(fn, 0)` (their relative order versus each other is actually not fully deterministic when scheduled from the top-level script, but *is* deterministic — `setImmediate` always wins — when both are scheduled from within an I/O callback). A snippet using `process.nextTick` in Node will show that callback's output ahead of *any* promise-based microtask output, which has no browser equivalent at all (browsers have no `process.nextTick`).

The trap: assuming Node.js and browser event loops are identical because they share the same underlying microtask/macrotask vocabulary — the phased structure and the existence of `process.nextTick` as an even-higher-priority queue than promises are genuinely Node-specific and change real answers to real trace questions if the runtime isn't specified.

---

**Q (Medium): Can a sufficiently pathological promise chain "starve" a `setTimeout` callback from ever running? Walk through how.**

Answer: Yes — since the microtask queue must be fully drained (including microtasks newly scheduled *during* that draining) before the engine will even look at the macrotask queue, a recursive pattern like `function loop() { Promise.resolve().then(loop); } loop();` schedules a new microtask from within a microtask, indefinitely, and the engine — following its own algorithm faithfully — never reaches a point where the microtask queue is empty, so it never proceeds to check the macrotask queue at all. Any `setTimeout` callback scheduled before or during this loop simply never runs, for as long as the recursive microtask scheduling continues; this is a real, documented starvation hazard, not a theoretical curiosity (it's cited as a cautionary pattern in engine implementer discussions specifically because naive recursive `.then` chains can accidentally produce it).

The trap: assuming the event loop has some fairness mechanism that guarantees macrotasks eventually get a turn regardless of microtask volume — it doesn't; the algorithm as specified genuinely prioritizes complete microtask-queue draining over macrotask progress, with no built-in circuit breaker, which is precisely why this starvation pattern is possible in practice, not just in theory.

---

**Q (Low): How would `queueMicrotask` differ in priority or behavior from `Promise.resolve().then(fn)`, if at all, for the purposes of trace prediction?**

Answer: For pure ordering-prediction purposes, they're equivalent — both schedule `fn` onto the same microtask queue, with the same "runs before the next macrotask, and before any macrotask, no matter how many microtasks get chained" priority. The practical difference is more about intent and minor mechanics: `queueMicrotask` schedules a callback directly with no promise machinery involved at all (no `.then` chaining semantics, no automatic error handling via rejection — a thrown error inside a `queueMicrotask` callback becomes an uncaught exception/reported error rather than a rejected promise that could be `.catch`-handled), whereas `Promise.resolve().then(fn)` necessarily goes through promise resolution machinery (including, technically, an extra microtask "hop" in some engines historically, for `thenable` resolution, though this detail has been optimized away in modern engines and isn't reliably observable). For trace-prediction interview purposes, treating them as same-priority microtasks is the correct and expected level of precision.

The trap: overclaiming a meaningful, always-observable ordering difference between the two — while there have been historical engine-specific micro-differences in extra microtask "hops" for promise resolution, presenting this as a reliable, spec-guaranteed distinction (rather than an implementation detail that's mostly been converged away) overstates certainty on a genuinely niche point.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can state the full event loop algorithm (stack → drain microtasks fully → one macrotask → drain microtasks fully → repeat) without hesitation
- [ ] Can trace a mixed `setTimeout`/`Promise`/`async-await` snippet and predict exact output order under time pressure
- [ ] Can explain precisely why `setTimeout(fn, 0)` never runs before a `Promise.resolve().then()` scheduled around the same time
- [ ] Can correctly identify which parts of an `async function` run synchronously vs. as a deferred microtask
- [ ] Can explain how a recursive microtask chain can starve `setTimeout` callbacks indefinitely
- [ ] Can name at least one concrete Node.js-vs-browser event loop difference (`process.nextTick`, phased loop structure)

---
*Next: this closes out Phase 1 (JavaScript Machine Coding & Async Mechanics). Phase 2 shifts from pure-logic/async-primitive problems to component-level machine coding — building actual interactive UI pieces (autocomplete, modals, data tables) where these async fundamentals (debouncing, race conditions, cancellation) get applied inside a rendering context for the first time.*
