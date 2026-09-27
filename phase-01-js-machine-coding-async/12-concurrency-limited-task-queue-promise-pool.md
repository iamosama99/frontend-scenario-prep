# Concurrency-limited Task Queue (Promise Pool)

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Bounded concurrency | Track an active-worker count; start a new task only when `active < limit` | Firing all tasks with `Promise.all(tasks.map(fn))` runs them fully in parallel — fine for 5 requests, catastrophic for 5,000 (browser connection limits, rate limits, memory) |
| Self-refilling workers | Each worker, on finishing one task, immediately pulls the next from a shared queue | This is what keeps exactly `limit` tasks in flight at all times, rather than processing in fixed batches with idle gaps between them |
| Order-preserving results | Store results by original index, not by completion order | Tasks complete in unpredictable order under concurrency; callers almost always want results back in the order they submitted, not completion order |
| Per-task failure isolation | One task's rejection doesn't stop the others (unless explicitly required to) | A pool processing 100 uploads shouldn't abandon the other 99 because one failed — but this needs to be a stated design decision, not an accident |

## The Scenario

"We need to upload 500 files, but we can't fire all 500 requests at once — the browser will throttle us and the server will reject the burst. Write a function that runs a list of async tasks with a concurrency limit of N at a time, and returns all the results once everything's done."

## Clarifying Questions

- **Should the function fail fast on the first rejected task, or let all tasks run to completion regardless of individual failures?** This mirrors the `Promise.all` vs. `Promise.allSettled` distinction, and it's not obvious which is wanted here — for a batch upload, I'd lean toward "let everything run, collect both successes and failures," since abandoning 400 pending uploads because upload #12 failed is usually worse than finishing the batch and reporting which ones failed. I'd confirm this explicitly rather than assume.
- **Do results need to preserve the original task order, or is completion order acceptable?** Since tasks complete at different times under concurrency, "task 3 finishes before task 1" is normal — but callers almost always want `results[i]` to correspond to `tasks[i]`, not to be in whatever order things happened to finish. I'd confirm, but default to order-preserving since that's what `Promise.all` does and what most callers expect by convention.
- **Is the task list known up front (an array), or can tasks be added dynamically while the pool is running** (e.g., a producer streaming new upload requests in as the user selects more files)? A fixed-array version is simpler; a dynamically-fed queue needs an `add()` method and different lifecycle semantics (when is "done"?). I'd ask, but default to the fixed-array version matching "we need to upload 500 files" (a known, upfront list) unless told the list grows live.
- **What should the concurrency limit actually be, and is it based on a known constraint** (browser's per-origin connection limit, typically 6, or a server-side rate limit)? Worth surfacing since the "right" number isn't arbitrary — it should be informed by whatever the actual bottleneck is (HTTP/1.1 browsers cap concurrent connections per origin at ~6; HTTP/2 multiplexes over one connection so the limit becomes more about server-side rate limits or client CPU/memory for concurrent processing).

## Approach & Trade-offs

The mental model is a fixed-size pool of "workers," each pulling from a shared queue and immediately grabbing the next task the moment it finishes one — not fixed batches processed sequentially.

**Why not `Promise.all` in chunks of N (batching)?** A naive-but-plausible first approach: split tasks into groups of `N`, `await Promise.all()` on each group sequentially. This *does* bound concurrency to `N`, but wastes time — if 5 of 6 tasks in a batch finish quickly and the 6th is slow, the other 5 workers sit idle waiting for that batch to fully complete before the *next* batch starts, even though there's more work available immediately. True concurrency-limiting keeps exactly `N` tasks in flight continuously, refilling a finished slot with the next queued task immediately rather than waiting for the whole current group to drain.

**The worker-pool pattern**: spin up exactly `min(limit, tasks.length)` "workers" (as async functions, not actual OS threads — this is cooperative, not parallel, concurrency), each running a loop: pull the next task index off a shared cursor, run it, store its result at the correct index, repeat until the queue is exhausted. All workers run "simultaneously" in the sense of being interleaved by the event loop (since each `await` yields control), which is exactly what bounds real concurrency to `N` — there are only ever `N` of these loops alive at once, each holding at most one task's promise unresolved at a time.

**Result ordering via a pre-sized array + index tracking**, not `.push()` — same principle as `Promise.all`: since tasks can complete in any order, storing by original index (`results[i] = value`) rather than appending is what guarantees `results` comes back in submission order regardless of completion order.

**Failure handling as an explicit, stated choice**: I implement the "let everything finish, collect both results and errors" variant by default (mirroring `Promise.allSettled`'s shape: each result slot is `{ status: 'fulfilled', value }` or `{ status: 'rejected', reason }`), since that matches the batch-upload use case best, but I'd note the fail-fast variant is a small modification (reject the overall promise immediately on the first task rejection, without waiting for in-flight tasks — though "in-flight tasks already started" can't actually be cancelled unless the task functions themselves support an `AbortSignal`, which is worth flagging as a limitation either way).

**A shared mutable cursor (`nextIndex`) rather than `.shift()`-ing an array**, because `.shift()` is O(n) per call (every remaining element re-indexes) — for a queue of hundreds or thousands of tasks, an incrementing index into the original array is O(1) per pull and avoids needless array mutation entirely.

## Solution

```javascript
async function runWithConcurrencyLimit(tasks, limit) {
  const results = new Array(tasks.length);
  let nextIndex = 0;

  async function worker() {
    while (nextIndex < tasks.length) {
      const currentIndex = nextIndex++; // claim this index before awaiting anything
      try {
        const value = await tasks[currentIndex]();
        results[currentIndex] = { status: 'fulfilled', value };
      } catch (reason) {
        results[currentIndex] = { status: 'rejected', reason };
      }
    }
  }

  const workerCount = Math.min(limit, tasks.length);
  const workers = Array.from({ length: workerCount }, () => worker());
  await Promise.all(workers); // waits for all worker loops to drain the queue

  return results;
}
```

```javascript
const uploadTasks = files.map((file) => () => uploadFile(file)); // array of thunks, not promises

const results = await runWithConcurrencyLimit(uploadTasks, 6);

const failed = results.filter((r) => r.status === 'rejected');
console.log(`${results.length - failed.length}/${results.length} uploads succeeded`);
```

> **Check yourself:** Why does `nextIndex++` need to happen synchronously, *before* the `await` on that line — what race condition would occur if it happened after the task resolved instead?

## Why Tasks Must Be Passed as Functions, Not Promises

A detail easy to get wrong in the calling code, not just the pool implementation: `tasks` must be an array of **functions that return promises when called** (`() => uploadFile(file)`), not an array of already-created promises (`uploadFile(file)`). This is because a promise, once created, starts running immediately — `uploadFile(file)` fires the upload the instant that expression executes, regardless of whether anything is `await`-ing it yet. If the caller built `const tasks = files.map(uploadFile)` (an array of live promises, all already in flight), by the time `runWithConcurrencyLimit` even starts, all 500 uploads would already be running — the concurrency limit would have no effect at all, since there's nothing left to *defer*. Wrapping each task in a thunk (`() => uploadFile(file)`) is what lets the pool control *when* each task actually starts, which is the entire mechanism the concurrency limit depends on.

## Gotchas

**Passing already-invoked promises instead of task functions.** As explained above, this silently defeats the entire purpose of the utility — the tasks all start immediately regardless of the stated limit, and the bug is easy to miss in testing with a small number of tasks (where firing everything at once doesn't visibly cause problems) but causes exactly the browser-throttling/server-rejection failure mode the concurrency limit was built to prevent, once the task count is large.

**Chunked `Promise.all` batching instead of a true worker pool.** As discussed above, this bounds concurrency correctly but leaves workers idle between batches whenever task durations are uneven — for 500 uploads of varying file sizes, this can meaningfully slow down total completion time compared to a pool that immediately refills any finished slot.

**Using `.push()` for results instead of index-based storage.** Produces results in *completion* order rather than *submission* order — passes tests where all tasks take equal time, breaks the moment durations vary, which for a batch upload is basically always (files differ in size).

**Claiming the task index *after* awaiting instead of before.** If `nextIndex++` happens after `await tasks[nextIndex]()` rather than capturing `currentIndex = nextIndex++` before the `await`, multiple workers can race to read the same `nextIndex` value simultaneously (since JS's single-threaded model still allows this race across separate async function invocations interleaved at `await` boundaries) — leading to two workers processing the same task while another task never gets picked up. The fix is claiming (incrementing) the index synchronously, in the same tick, before any `await` yields control.

**One task's rejection silently aborting the entire batch, when that wasn't the intent.** If task execution isn't wrapped in its own `try/catch` inside the worker loop, a single rejected task propagates up through `Promise.all(workers)` and rejects the whole pool immediately — the *other* 499 in-flight and not-yet-started uploads' outcomes are simply never reported (though already-started ones do keep running in the background, orphaned, since nothing cancels them), which is rarely what "upload 500 files, tell me the outcome of the batch" actually wants.

## Follow-up Questions

**Q (High): Why is a true worker-pool pattern better than processing tasks in fixed-size batches (chunked `Promise.all`)?**

Answer: Both approaches correctly bound the *maximum* number of concurrent tasks to `N`, but batching bounds it in a way that leaves capacity idle whenever tasks within a batch finish at different times — the batch as a whole only advances once *every* task in it (including the slowest) completes, so a single slow task in a batch of 6 blocks the other 5 slots from picking up new work even though they're free. A true pool has each worker independently and immediately pull the next task the instant it finishes its current one, so exactly `N` tasks are in flight at (almost) all times, with no artificial synchronization points between "batches" — for a workload with variable task duration (which describes almost all real-world I/O, including file uploads of different sizes), this measurably reduces total wall-clock time to complete the full set.

The trap: presenting chunked `Promise.all` as an equivalent, simpler alternative without acknowledging the idle-capacity cost — it's a reasonable *simpler* answer if asked to write something quickly, but conflating it with "as good as" a real pool misses a meaningful practical difference that's exactly what this scenario is designed to test.

---

**Q (High): How do you guarantee that `nextIndex++` doesn't cause two workers to process the same task, given that this is all running on a single JS thread?**

Answer: Even though JavaScript is single-threaded, multiple `async function` worker loops running "concurrently" are actually interleaved cooperatively at `await` boundaries — control only switches between them when one hits an `await` (or otherwise yields, e.g., via a microtask). The key correctness requirement is that reading and incrementing `nextIndex` (`const currentIndex = nextIndex++`) happens as a single, uninterrupted synchronous statement, with no `await` in between the read and the increment — since JS guarantees synchronous code runs to completion without another task interleaving mid-statement, this makes the claim atomic *in practice*, even without a real mutex/lock (which JS doesn't need for this, precisely because there's only one thread actually executing synchronous code at any instant). If the increment happened *after* an `await` (e.g., reading `nextIndex` before calling the task, but only incrementing after the task resolves), two workers could both read the same `nextIndex` value before either increments it, both process the same task, and the task at the *next* index would never get claimed by anyone.

The trap: assuming any use of concurrency in JS automatically needs explicit locking (mutex-style) borrowed from multi-threaded languages — the actual requirement is much narrower and specifically about ensuring no `await` sits between reading and mutating shared state that determines "which task is next," which is a JS-idiomatic way of thinking about this that differs from how the same problem is solved in a genuinely multi-threaded runtime.

---

**Q (Medium): How would you modify this to support cancelling all remaining (not-yet-started) tasks if one task fails — a "fail fast" mode instead of "run everything to completion"?**

Answer: Add a shared `cancelled` flag (or an `AbortController` whose signal is checked) that any worker sets to `true` immediately upon catching a rejection in fail-fast mode; each worker's `while` loop condition then also checks `!cancelled` in addition to `nextIndex < tasks.length`, so once one worker observes a failure, all workers stop claiming *new* tasks on their next loop iteration. The overall promise should then reject with that first error (often via a manually-created promise that's rejected explicitly, since `Promise.all(workers)` alone wouldn't propagate a flag-based cancellation as a rejection on its own). Critically, this only stops tasks that *haven't started yet* — tasks already in flight when the failure occurs can't be forcibly stopped unless the task functions themselves accept and respect an `AbortSignal` (e.g., `fetch` does), so "fail fast" in a promise-pool context usually means "stop starting new work," not "instantly halt everything already running."

The trap: claiming that cancellation can stop already-in-flight async work by default — without the task functions explicitly supporting cancellation (via `AbortSignal` or similar), a promise that's already been created and is awaiting some I/O operation will run to completion (or its own failure) regardless of what the pool "decides" afterward; the pool can only choose not to start new work and not to *wait* for/use the result, not force the operation itself to stop.

---

**Q (Medium): How is this problem related to `Promise.all`, and what would a from-scratch `Promise.all` implementation share with this pool's structure?**

Answer: `Promise.all` is, structurally, the *unbounded*-concurrency special case of this same problem — it fires every task immediately (concurrency limit of "however many there are") and collects results by index once everything settles, using the same "pre-sized results array + per-index assignment + a counter of remaining unsettled items" pattern this pool uses internally. The concurrency-limited pool generalizes that by adding a gate: instead of starting all tasks at construction time, it starts only `limit` at a time and has each one, upon finishing, trigger the start of the next queued one — `Promise.all` can be thought of as this same pool with `limit = tasks.length` (i.e., no gating at all, every worker starts immediately). Recognizing this connection is a good way to demonstrate that these "different" machine-coding problems (custom Promise, `Promise.all` polyfill, concurrency-limited pool) aren't actually separate ideas — they're variations on the same "track N pending async operations and aggregate their outcomes" primitive.

The trap: treating this as an unrelated new algorithm rather than recognizing it as `Promise.all` with an added admission-control gate — missing that connection means re-deriving index-based result ordering and per-task error isolation from scratch instead of recognizing them as the same techniques already used in the `Promise.all` polyfill scenario.

---

**Q (Low): How would you determine a sensible concurrency limit in practice, e.g., for browser-side file uploads?**

Answer: The limit should be informed by the actual bottleneck, not picked arbitrarily. For HTTP/1.1, browsers cap concurrent connections *per origin* at roughly 6 (this varies slightly by browser) — exceeding that doesn't add real parallelism, it just queues excess requests at the browser's network layer, so a limit meaningfully above 6 provides no benefit and just adds bookkeeping overhead for connections that immediately queue anyway. For HTTP/2 (which multiplexes many logical streams over one physical connection), the practical limit shifts to server-side capacity (rate limits, how many concurrent uploads the backend can actually process) or client-side resource usage (memory held by in-flight file reads/encodings) rather than a hard browser connection cap. In practice, I'd start with a conservative default (matching the browser's per-origin limit, ~6, for HTTP/1.1 contexts) and make it configurable, ideally informed by measuring actual throughput at different limits rather than guessing.

The trap: picking an arbitrary "reasonable-sounding" number without being able to justify it against a concrete constraint (browser connection limits, server rate limits, memory) — the interview-worthy answer names the actual bottleneck the limit is protecting against, not just "some number that isn't too high or too low."

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement a true concurrency-limited worker pool (not chunked batching) from memory
- [ ] Can explain why tasks must be passed as thunks/functions, not already-invoked promises
- [ ] Can explain why claiming `nextIndex` must happen synchronously before any `await`, and what race condition results otherwise
- [ ] Can articulate why worker-pool beats chunked `Promise.all` for uneven task durations
- [ ] Can connect this problem back to `Promise.all`'s own internal structure (index-based results, counter of remaining items)
- [ ] Can explain the real limitation of "cancelling" already-in-flight tasks without `AbortSignal` support in the task functions

---
*Next: Pub/Sub System — shifts from managing a pool of async tasks to managing a registry of subscribers reacting to published events, a different but related pattern for decoupling producers from consumers.*
