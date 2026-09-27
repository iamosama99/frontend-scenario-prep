# Long Tasks Blocking the Main Thread

## Quick Reference

| Long Task Source | How You Spot It | Fix |
|---|---|---|
| One large synchronous computation (sort/filter/transform on a big array) | A single wide "Task" block in the Performance panel, one function dominating it | Break into chunks (yield between batches), or move off-thread (Web Worker) |
| Rendering a very large list/tree in one commit | Long task correlates with a component mount/update, "Recalculate Style"/"Layout" heavy | Virtualize the list, or paginate/incrementally render |
| Third-party script doing synchronous initialization | Long task's call stack roots in a vendor script, not app code | Defer initialization, load `async`, or negotiate lazy-init with the vendor |
| A `setState` update cascading into a large re-render tree | Long task's call stack shows React commit/reconciliation work | Memoize, split state so updates affect a smaller subtree, batch updates |
| A tight synchronous loop with no yield point (e.g., JSON parsing/stringifying a huge payload) | Task duration scales directly with payload size | Stream/parse incrementally, move to a worker, or reduce payload size |

## The Scenario

"Users say the app 'freezes' for a second or two right after clicking certain buttons — clicks register late, animations stutter, sometimes the whole tab looks unresponsive. Chrome's own Performance panel is flagging 'Long Tasks' during these moments. Explain what a long task actually is, find the one(s) causing this, and fix them."

## Clarifying Questions

- **Is "freezes" specifically about input not registering (clicks feel delayed/ignored) or about visual stutter (animations pausing, scroll janking) during the same window — or both?** Both symptoms share the same root cause (the main thread being occupied and unable to process new work, whether that's an input event handler or a paint), but confirming both are present (rather than just one) helps confirm the diagnosis is "main thread blocked" rather than something narrower, like a single animation-specific issue.
- **Which specific button/action triggers this, and does it correlate with an amount of data** (a button that filters/sorts a large list, submits a form that re-renders a big table, opens a modal with a large dataset)? A long task whose duration scales with data size points at an unoptimized synchronous operation over that data (sort, filter, transform, render) as the direct cause — worth confirming this correlation before assuming a fixed-cost cause (a third-party script, a fixed-size computation).
- **Does the Performance panel's Long Task entry show the call stack rooted in application code, a UI framework's internals (e.g., React reconciliation), or a third-party script?** This determines who owns the fix — application-level fixes (chunking, virtualization) if it's app code; framework-level tuning (memoization, splitting state to limit re-render scope) if it's framework reconciliation cost; escalation/negotiation with a vendor, or self-hosting and deferring their script, if it's a third party.
- **How long is "a second or two" precisely, per the trace** — is a single task genuinely ~1-2 seconds long, or is it actually a rapid sequence of many shorter tasks (each individually under the 50ms long-task threshold, but back-to-back with no yield) that add up to a similarly-blocked-feeling window? These look similar to a user but have different fixes: one enormous task needs to be chunked/broken up; many small tasks in a row without yielding to the browser between them needs the code restructured to actually yield (not just made individually smaller) so the browser gets a chance to process input/paint between them.
- **Is this reproducible consistently on a fast dev machine, or does it need CPU throttling to see clearly?** If it's *already* visibly bad on a fast dev machine, the underlying task is severe (likely to be much worse for real users on average/weaker hardware); if it only shows up under throttling, that calibrates how urgently it needs fixing relative to the team's typical device/traffic profile.

## Approach & Trade-offs

**A "long task" has a precise technical definition, and knowing it explains why the specific 50ms threshold matters, not just that "long tasks are bad."** Any task on the main thread running longer than 50ms is classified as a Long Task (surfaced via the `PerformanceObserver` Long Tasks API and flagged in DevTools) — the 50ms figure isn't arbitrary: it's derived from the "Response, Process, Present" budget where user input needs to be handled within roughly 100ms to feel instantaneous, and 50ms leaves headroom for the remaining pipeline work (rendering the response) within that budget. A single task longer than this blocks *everything* else queued on the main thread — pending input events, scheduled rendering work, other JS callbacks — for its entire duration, because JavaScript execution on the main thread is single-threaded and non-preemptive: once a task starts running, the browser cannot interrupt it partway through to process a higher-priority pending click, no matter how urgent that click is.

**The core fix pattern across almost every cause is the same idea: yield control back to the browser periodically, rather than running one uninterrupted block of work.** Whether the long task is a big array transformation, a large render, or heavy JSON processing, the fix generally involves breaking the work into smaller chunks and inserting yield points between them (via `setTimeout(fn, 0)`, the newer `scheduler.yield()`/`isInputPending()` APIs, or React's own concurrent-rendering time-slicing) so the browser gets a chance to process a pending click or paint a frame between chunks, even though the *total* wall-clock time to finish all the work might be similar or even slightly longer than running it as one block. This is a genuinely different goal than "make the computation faster" — a chunked version that still takes 1.5s total but yields every 40ms feels far more responsive than an unchunked 1.2s block, because input and paint get interleaved throughout rather than queued up entirely behind the computation.

**Distinguish "the work is fundamentally too slow and needs an algorithmic/architectural fix" from "the work is fine in total but needs to be chunked/moved off-thread."** Sorting a genuinely enormous array with an inefficient algorithm needs a better algorithm or a smaller working set (pagination, virtualization) before chunking even becomes relevant — chunking a fundamentally-too-slow O(n²) operation just spreads the same excessive total cost across more, still-too-many interruptions to the user's session, rather than fixing the actual problem. I'd profile to determine whether the *total* cost is reasonable (then chunking/yielding is the right fix, purely for responsiveness) or unreasonable for what's being computed (then the real fix is algorithmic or reduces the data volume first).

**Web Workers are the strongest fix when the work is genuinely CPU-heavy and doesn't need synchronous DOM access, since it removes the work from the main thread entirely rather than just interleaving it more finely.** Chunking still consumes the same total main-thread time, just spread out (better for responsiveness, same total main-thread cost); a worker moves the actual computation to a separate thread, so the main thread is free for the *entire* duration, at the cost of message-passing overhead (data must be serialized/cloned across the boundary, which itself isn't free for very large payloads) and the inability to touch the DOM directly from within the worker.

## Solution — the diagnostic + fix walkthrough

**Step 1 — record a trace of the exact repro (click the button, capture the freeze).** DevTools → Performance → record → click → stop. Long Tasks appear as task blocks with a red flag in the top-right corner if they exceed 50ms.

**Step 2 — inspect the flagged task's call stack.** Expanding it might show:

```
Task duration: 1420ms  [LONG TASK — flagged]
Call stack:
  onClick (ProductTable.tsx:42)
    → applyFiltersAndSort (ProductTable.tsx:58)
        → products.filter(...).sort(...)   ← 1380ms of the 1420ms total
```

A single synchronous `.filter().sort()` over a large in-memory array, run entirely within one click handler, with nothing yielding control back to the browser until it completes.

**Step 3 — check whether the total cost itself is reasonable for the data size**, e.g. confirm the array size (10,000 rows) and whether the filter/sort logic itself is efficient (no accidental O(n²) comparator, no redundant re-computation per element) — assume here the algorithm is fine, it's just a genuinely large synchronous unit of work for the main thread to do in one uninterrupted block.

**Step 4 — fix by chunking the work with yield points**, using the browser's scheduling primitives so the browser can interleave input handling and paint between chunks:

```ts
// Before: one long synchronous block
function applyFiltersAndSort(products: Product[], filters: Filters) {
  return products.filter(p => matchesFilters(p, filters)).sort(compareProducts);
}

// After: chunked, yielding between batches
async function applyFiltersAndSortChunked(
  products: Product[],
  filters: Filters,
  chunkSize = 500
): Promise<Product[]> {
  const filtered: Product[] = [];
  for (let i = 0; i < products.length; i += chunkSize) {
    const chunk = products.slice(i, i + chunkSize);
    filtered.push(...chunk.filter(p => matchesFilters(p, filters)));
    // Yield to the browser between chunks — lets a pending click/paint through
    await new Promise(resolve => setTimeout(resolve, 0));
  }
  return filtered.sort(compareProducts); // sort is comparatively cheap once filtered down; chunk it too if not
}
```

`setTimeout(resolve, 0)` schedules the continuation as a new macrotask, which lets the browser process anything else queued (including pending input and a paint) before running the next chunk — the modern equivalent, where available, is `scheduler.yield()` from the Prioritized Task Scheduling API, purpose-built for exactly this pattern with better scheduling semantics than a `setTimeout(0)` hack.

**Step 5 — for genuinely CPU-heavy, DOM-independent work, prefer a Web Worker over chunking** when the total cost is large enough that even chunked main-thread time meaningfully competes with other work:

```ts
// worker.ts
self.onmessage = (e) => {
  const { products, filters } = e.data;
  const result = products.filter(p => matchesFilters(p, filters)).sort(compareProducts);
  self.postMessage(result);
};

// main thread
const worker = new Worker('worker.ts');
worker.postMessage({ products, filters });
worker.onmessage = (e) => setFilteredProducts(e.data);
```

This keeps the main thread completely free during the computation — no interleaving needed, since the work isn't happening there at all — at the cost of the serialization overhead of passing `products` across the worker boundary (real for very large datasets, and worth measuring rather than assuming is negligible).

**Step 6 — re-trace and confirm no Long Task flag appears for the same interaction, and specifically verify a click during the (previously blocking) window now registers promptly** — the actual user-facing symptom (delayed clicks) is the thing to re-validate, not just the absence of a red flag in the trace.

> **Check yourself:** In the chunked version above, why does the fix use `await new Promise(resolve => setTimeout(resolve, 0))` inside the loop instead of, say, `await Promise.resolve()` (a microtask) — what would break if a microtask were used instead?

## Gotchas

**Chunking work with a microtask-based yield (`Promise.resolve()`/`queueMicrotask`) instead of a macrotask-based one, and finding the freeze persists.** Microtasks all run to completion before the browser yields control back for rendering/input — chaining `await Promise.resolve()` between chunks still runs every chunk back-to-back within the same overall task, without ever giving the browser an opportunity to paint or process input in between; only yielding via a macrotask (`setTimeout`, `MessageChannel`, or `scheduler.yield()`) or by returning control to the event loop lets the browser interleave other work.

**Chunking a fundamentally inefficient algorithm and calling it fixed**, when the real issue is (for example) an O(n²) comparator function run during sort — chunking spreads the same excessive total main-thread time across more, smaller blocking windows, which can improve responsiveness somewhat but leaves the underlying inefficiency (and its poor scaling as data grows further) unaddressed.

**Moving work to a Web Worker without accounting for serialization cost on very large payloads.** `postMessage` structured-clones the data by default — for a sufficiently large array/object, the clone itself can take a non-trivial amount of time on both ends of the boundary, which can offset some of the benefit; `Transferable` objects (like `ArrayBuffer`) avoid a full clone where the data can be represented that way, worth considering for genuinely large payloads.

**Confusing "many short tasks in rapid succession" with "one long task" when reading the trace**, and applying a chunking fix to something that's already technically broken into small pieces but simply run without any actual yield between them (e.g., a `for` loop calling `requestAnimationFrame` recursively but each iteration's callback itself does too much work, or a sequence of several separate function calls each under 50ms but with zero idle time between them) — the fix here isn't "make it smaller," since it may already be nominally chunked; the fix is ensuring genuine yield points exist between the pieces.

**Only testing the fix on a fast dev machine and seeing the long-task flag disappear, without also checking under CPU throttling.** A borderline task (55ms, just over threshold on a fast machine) can shrink under a superficial fix to just under 50ms on the dev machine while still being genuinely long (200ms+) on real user hardware — the fix needs validating under a representative throttle, same as any other performance fix in this phase.

## Follow-up Questions

**Q (High): Define, precisely, what makes a task a "Long Task" — what's actually being measured, and why does JavaScript's single-threaded execution model make this matter so much?**

Answer: A Long Task is any task on the browser's main thread event loop that runs for 50 milliseconds or longer without yielding — this includes any synchronous JS execution triggered by an event (a click handler), a scheduled callback, a promise continuation, or any other unit of work the event loop processes as a single, uninterruptible turn. It matters specifically because of JavaScript's main-thread execution model: within a single realm, JS execution is single-threaded and non-preemptive, meaning once a task begins running, the browser's engine cannot pause it partway through to handle something else more urgent (a pending click, a scheduled paint) — it must run to completion (or to its next explicit yield point) before the event loop can move on to the next queued item. This is why a single 1.5-second synchronous computation doesn't just "make that computation slow" — it makes the *entire page* unresponsive to anything for that full 1.5 seconds, since every other pending main-thread activity is queued up strictly behind it with no way to interleave.

The trap: describing a long task vaguely as "something that takes a while" without connecting it to the specific consequence of the single-threaded, non-preemptive execution model — the reason a 1.5s task is categorically worse than "the page feels 1.5s slower" is that it blocks *everything else* queued on the main thread for that entire window, not just the operation itself.

---

**Q (High): Explain why `await new Promise(resolve => setTimeout(resolve, 0))` yields control back to the browser between loop iterations, while `await Promise.resolve()` does not — trace through the event loop mechanics.**

Answer: `setTimeout`, even with a 0ms delay, schedules its callback as a macrotask — the browser's event loop processes exactly one macrotask per iteration, and critically, after a macrotask finishes (and before starting the next queued macrotask), the browser has the opportunity to run pending rendering work (style/layout/paint) and process pending input events. `Promise.resolve()`'s `.then()` continuation (which is what `await` desugars to) is a microtask, and the event loop's rule is that *all* currently-queued microtasks are drained completely before the event loop proceeds to rendering or the next macrotask — meaning a loop that `await`s only microtasks between iterations keeps the entire chain running within what is, from the browser's rendering/input-processing perspective, still effectively one continuous unit of work, since microtasks never yield to rendering or input in between. So chunking with microtask-only yields technically produces more, smaller pieces of JS execution, but doesn't achieve the actual goal (letting the browser paint or handle input between chunks), since the browser never gets an opportunity to do so until the whole microtask chain drains.

The trap: assuming any `await` inside a loop is sufficient to "yield" in the sense that matters for responsiveness — `await` only defers to the microtask queue by default, and microtask draining specifically does not include an opportunity for rendering or input processing, which is the actual property needed here.

---

**Q (High): You've confirmed via the trace that a Long Task's call stack is dominated by React's own reconciliation/commit work following a `setState` call, not application logic like a sort or filter. How does the fix differ from the chunking approach used for a raw JS computation?**

Answer: For work inside React's own rendering pipeline, manually chunking with `setTimeout`/worker offloading isn't directly applicable the same way, since the expensive work (reconciliation, committing DOM changes) has to happen on the main thread through React's own scheduler, not as an arbitrary JS function that can be sliced up freely. The more applicable fixes are: reducing the *scope* of what needs to re-render on that `setState` (splitting state so the update only affects a smaller subtree, rather than one `setState` at a high level cascading into a huge re-render of a large tree — see [[05-context-causing-app-wide-re-renders]] in Phase 3 for the general shape of this problem), memoizing expensive child subtrees (`React.memo`, `useMemo` for expensive derived values) so they're skipped during a re-render they don't actually need to participate in, and, where available, leaning on React's own concurrent features (`useTransition`/`startTransition`) to mark the update as non-urgent, which lets React's scheduler interrupt and yield during that work in favor of more urgent updates (like the input the user is trying to make), rather than a raw JS chunking loop trying to reimplement the same idea manually.

The trap: applying a generic "wrap it in `setTimeout` chunks" fix to framework-internal rendering work — React's own scheduler already has purpose-built primitives (`startTransition`, concurrent rendering) for exactly this class of problem, and manually chunking around React's commit phase either doesn't apply cleanly or fights against the framework's own scheduling rather than using it.

---

**Q (Medium): A long task is caused by a third-party analytics script's synchronous initialization code, which your team doesn't own or control. What are your actual options?**

Answer: Options, roughly in order of how much control they require: (1) Load the script `async`/`defer` if not already, so at minimum it doesn't block HTML parsing/other resource discovery — this doesn't fix the long task itself once the script executes, but limits its blast radius on unrelated critical-path work. (2) Delay loading/initializing the script until after the page's initial critical interactions are done (e.g., initialize it on `requestIdleCallback`, or after a short delay, or after the user's first meaningful interaction) — trading "the cost happens later, when the user is less likely to be trying to click something at that exact moment" for "the cost still exists somewhere." (3) Check whether the vendor offers a lighter-weight or asynchronous initialization mode/API — many analytics vendors do, and it's worth an explicit ask if the default snippet is heavier than necessary. (4) As a last resort, if the vendor's script is genuinely unavoidable and unfixable from the loading side, self-host and monitor it specifically, and treat its cost as a known, tracked trade-off in the page's performance budget, revisited if a better-behaved alternative vendor becomes viable. What's generally not a real option is silently ignoring it — an unowned script contributing a real long task still degrades the actual user experience regardless of whose code caused it, and users don't distinguish "our bug" from "a vendor's bug" when judging the page as unresponsive.

The trap: treating "we don't own that code" as a reason to stop investigating — the fix set is more constrained than for first-party code, but there's still real leverage (loading strategy, timing, vendor negotiation) short of rewriting the vendor's script, and it's worth exhausting that leverage rather than shrugging it off as untouchable.

---

**Q (Medium): Would code-splitting the JS bundle (from the earlier bundle-size scenario in this phase) ever cause a long task where one didn't exist before?**

Answer: Yes, indirectly — a dynamically `import()`ed chunk, once its network fetch resolves, still needs to be parsed, compiled, and executed on the main thread just like any other script, and if that chunk itself contains a large synchronous initialization (registering many components, running setup logic) it can itself register as a long task at the moment it's evaluated, even though the *original* problem (a bloated eagerly-loaded bundle) was fixed. This is a case where fixing one bottleneck (load-time bundle size) can surface or relocate another (a long task, now occurring later, at the point the split chunk is evaluated, rather than earlier during the initial bundle's evaluation) — worth explicitly re-checking for long tasks after a code-splitting change, rather than assuming code-splitting is purely additive good with no new failure surface of its own.

The trap: treating code-splitting purely as a load-time-only optimization with no runtime-performance interaction — the split chunk's evaluation cost doesn't disappear, it moves to whenever that chunk is dynamically loaded and executed, and if that moment coincides with user interaction (e.g., importing a large chunk in response to a click, then immediately using it), it can itself present as a newly-visible long task at that trigger point.

---

**Q (Low): If long tasks are inherently about main-thread blocking, does moving to a framework/runtime with true multi-threaded rendering (rather than a single main thread) eliminate this class of bug entirely?**

Answer: It significantly changes the shape of the problem but doesn't eliminate the underlying constraint entirely — even architectures that move more rendering work off the traditional single main thread still generally need a single thread (or a tightly coordinated small set of them) to actually apply committed changes to the DOM and handle input events, since the DOM API itself is not thread-safe/is bound to one thread in current browser engines; multi-threaded approaches typically move the *preparation* work (diffing, computing what needs to change) off-thread while the final DOM application step remains a main-thread, effectively-serial operation. So this class of problem shrinks (there's less main-thread work per update, since expensive computation moved elsewhere) but the fundamental "something synchronous and large enough on the thread that talks to the DOM will still block that thread" constraint persists in some form — the fix categories in this scenario (reduce total cost, chunk/yield, move genuinely DOM-independent computation off-thread) remain the right mental model even as the specific thread doing the "final" work shifts.

The trap: assuming a more sophisticated rendering architecture is an unconditional fix for main-thread blocking in general — it reduces the *amount* of work forced onto the thread that owns the DOM, but doesn't remove the existence of that thread or its susceptibility to being blocked by a sufficiently large synchronous operation still running on it.

---

## Self-Assessment

- [ ] Can state the precise definition of a Long Task (50ms+ on the main thread) and explain why non-preemptive single-threaded execution makes it block everything else
- [ ] Can explain why a macrotask-based yield (`setTimeout`) actually lets the browser interleave rendering/input while a microtask-based yield (`Promise.resolve()`) does not
- [ ] Can distinguish "the total work is too slow, needs an algorithmic fix" from "the total work is fine, needs chunking/yielding for responsiveness"
- [ ] Can identify when a Web Worker is the better fix over chunking (CPU-heavy, DOM-independent work) versus when it isn't worth the serialization overhead
- [ ] Can name React-specific fixes (`startTransition`, memoization, narrowing re-render scope) for a long task rooted in reconciliation, distinct from raw-JS chunking

---
*Next: High INP / Unresponsive Interactions — the Core Web Vital that Long Tasks directly explain the mechanism behind; this scenario applies the same diagnostic tools to the specific metric Google measures and scores.*
