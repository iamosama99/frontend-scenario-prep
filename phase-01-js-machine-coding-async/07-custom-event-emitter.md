# Custom EventEmitter (on/off/once/emit)

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| `on(event, fn)` | Push `fn` into a `Map<event, fn[]>` bucket | Multiple independent parts of an app can listen to the same event |
| `off(event, fn)` | Remove `fn` from that bucket by reference | Required for cleanup — the #1 real-world memory leak source |
| `once(event, fn)` | Wrap `fn` in a self-removing wrapper, but keep a pointer back to the *original* `fn` | So `off(event, fn)` still works even if the listener hasn't fired yet |
| `emit(event, ...args)` | Iterate a **snapshot** of the listener array, not the live array | A listener that adds/removes another listener mid-emit must not corrupt the iteration |

## The Scenario

"Implement an `EventEmitter` class from scratch — `on`, `off`, `once`, and `emit`, similar to Node's built-in `EventEmitter`. Support method chaining like `emitter.on('a', fn1).on('b', fn2)`. And I want to see you handle the case where a listener removes another listener while `emit` is running — that's a real bug we hit."

## Clarifying Questions

- **Can multiple listeners be registered for the same event?** Yes — this is the entire point of the pattern, and it means the internal storage per event has to be a collection (array or `Set`), not a single function reference.
- **Should `off(event, fn)` remove *one* matching listener, or all listeners matching that reference?** I'd remove all occurrences of that exact function reference for that event — registering the same function twice for the same event is unusual but not disallowed, and "remove this listener" most naturally means "this listener shouldn't run anymore, however many times it was registered."
- **For `once` — if I call `off(event, originalFn)` *before* the event has ever fired, does it need to actually prevent the listener from running?** Yes, and this is the sharpest edge case in the whole prompt: internally, `once` almost certainly needs to wrap `fn` in a new function (to implement the auto-removal logic), but the *caller* only ever has a reference to the original `fn`, not the wrapper — so `off` needs a way to find "the wrapper that wraps this original function" to remove it correctly.
- **What should happen if a listener throws?** I'd default to *not* letting one listener's exception stop the others from running — wrapping each individual listener invocation in its own `try/catch` so a bug in listener A doesn't silently prevent listener B (registered for the same event) from ever running, which would otherwise turn one buggy feature into an outage for unrelated features sharing the same event.

## Approach & Trade-offs

**Storage:** a `Map` from event name to an array of listener functions. A `Map` (rather than a plain object) sidesteps prototype-pollution-adjacent footguns (`event = "constructor"` colliding with `Object.prototype` members) and makes `.has()`/`.get()`/`.set()` explicit rather than relying on `in`/bracket-access ambiguity.

**The mutation-during-iteration bug, specifically:** if `emit` iterates the *live* array stored in the map (e.g., a plain `for (const fn of listeners)` over the actual stored array) and a listener calls `off()` on another listener for the same event mid-iteration, the array is mutated *while being iterated* — depending on which listener is removed relative to the iterator's current position, this can cause a listener to be silently skipped (if a listener *before* the current position is removed, every subsequent index shifts down by one, and the iterator — which advances by index — skips over what is now at the current index). The fix: `emit` takes a **shallow copy** of the listener array (`[...listeners]`) at the moment it starts iterating, and iterates that copy — any mutation to the live array during emit affects *future* emits, not the one currently in progress, which is both correct and matches how Node's own `EventEmitter` behaves.

**`once`'s self-removal, and why it needs to preserve a reference to the original function:** the naive approach — wrap `fn` in `(…args) => { off(event, wrapper); fn(...args); }` and register the wrapper — works for the "fires once, then removes itself" behavior, but breaks `off(event, fn)` called with the *original* function before it has fired, because the stored listener is the *wrapper*, not `fn`, so a reference-equality search for `fn` in the listeners array finds nothing. The fix is storing the original function as a property on the wrapper (`wrapper.originalListener = fn`) so `off` can check both direct matches and `.originalListener` matches when searching for what to remove.

## Solution

```javascript
class EventEmitter {
  #events = new Map(); // event name -> array of listener functions

  on(event, listener) {
    if (!this.#events.has(event)) {
      this.#events.set(event, []);
    }
    this.#events.get(event).push(listener);
    return this; // enables chaining
  }

  off(event, listener) {
    const listeners = this.#events.get(event);
    if (!listeners) return this;

    // Match either the listener itself, or a `once`-wrapper wrapping it.
    this.#events.set(
      event,
      listeners.filter((l) => l !== listener && l.originalListener !== listener)
    );
    return this;
  }

  once(event, listener) {
    const wrapper = (...args) => {
      this.off(event, wrapper);
      listener.apply(this, args);
    };
    wrapper.originalListener = listener; // so off(event, listener) still works pre-fire
    return this.on(event, wrapper);
  }

  emit(event, ...args) {
    const listeners = this.#events.get(event);
    if (!listeners || listeners.length === 0) return false;

    // Snapshot BEFORE iterating — mutation during emit must not affect this pass.
    const snapshot = [...listeners];

    for (const listener of snapshot) {
      try {
        listener.apply(this, args);
      } catch (err) {
        // One listener's failure shouldn't prevent the others from running.
        console.error(`Error in listener for event "${event}":`, err);
      }
    }
    return true;
  }
}
```

```javascript
const emitter = new EventEmitter();

function onLoginA() { console.log('A'); }
function onLoginB() {
  console.log('B');
  emitter.off('login', onLoginA); // removing another listener MID-emit
}
function onLoginC() { console.log('C'); }

emitter.on('login', onLoginA).on('login', onLoginB).on('login', onLoginC);
emitter.emit('login');
// Logs: A, B, C — all three still run this time, because emit snapshotted the
// array before onLoginB mutated it. A future emit('login') would only log B, C.
```

> **Check yourself:** Why does taking a shallow copy (`[...listeners]`) — rather than, say, iterating backward through the live array — correctly solve the mutation-during-iteration problem, and would iterating backward have also worked?

## Gotchas

**Iterating the live array during `emit`.** This is the exact bug named in the scenario. Without a snapshot, a listener that removes another listener registered *earlier* in the array shifts every subsequent listener's index down by one — if the iteration is a simple ascending `for` loop tracking an index, the listener that just shifted into the "already visited" index gets skipped entirely for this emit. Copying the array before iterating sidesteps the whole class of bug regardless of which listener gets removed or where.

**`once`'s wrapper breaking `off` for the original function.** Covered above — the fix (a `originalListener` back-pointer checked by `off`) is the kind of detail that separates "I implemented `once`" from "I implemented `once` correctly," and it's a very natural thing for an interviewer to specifically probe if a candidate's first pass doesn't handle it.

**One listener's exception silently killing the rest.** Without a `try/catch` around each individual listener invocation, a thrown error inside listener B propagates up through the `emit` loop and prevents listener C (registered for the *same* event, but with no relationship to B's bug) from ever running — turning an isolated bug in one feature into a cross-cutting outage for every other feature that happens to share the same event name.

**Memory leaks from listeners that are never removed.** The classic real-world version of this: a component subscribes via `emitter.on('someEvent', this.handleEvent)` in a constructor/mount hook but never calls `emitter.off('someEvent', this.handleEvent)` on teardown — the emitter (which usually outlives any individual component/view) keeps a strong reference to the listener function, which itself may close over `this`/the component instance, keeping the entire component (and everything *it* references) alive in memory indefinitely, even after it's been "removed" from the UI.

**Passing a fresh arrow function to `on()` and then trying to `off()` with a *different* arrow function.** `emitter.on('x', () => foo())` followed later by `emitter.off('x', () => foo())` does **not** remove the listener — these are two entirely different function objects that happen to look identical in source code; reference equality (`===`) is what `off` checks, and two separately-created arrow functions are never `===` to each other. This is an extremely common real-world bug, distinct from the `once`-specific wrapper issue.

## Follow-up Questions

**Q (High): How do you implement `once` so the listener is automatically removed after firing exactly once, while still supporting `off` being called with the original function reference before it fires?**

Answer: `once(event, listener)` creates a wrapper function that, when invoked, first calls `this.off(event, wrapper)` (removing *itself* from the listener list) and then calls the original `listener` — this handles the "fires once, then self-removes" half correctly on its own. The harder half — supporting `off(event, listener)` called with the *original* function reference *before* the wrapped listener has ever fired — requires that `off`'s search logic isn't just "does this array element `=== ` the function I was given," because the array actually contains the *wrapper*, not the original. The fix is attaching the original function as a discoverable property on the wrapper (e.g., `wrapper.originalListener = listener`) and having `off` check both `l === listener` and `l.originalListener === listener` when deciding what to filter out.

The trap: implementing `once` in a way that technically self-removes correctly, but never tests (or considers) the specific case of `off` being called with the original function *before* the event fires — this is exactly the kind of edge case that "works in the demo" but fails the first time real calling code tries to unregister a `once`-listener defensively during cleanup, before it's had a chance to fire.

---

**Q (High): What bug occurs if a listener removes another listener (or itself) during `emit`, and how do you fix it?**

Answer: If `emit` iterates the emitter's *live*, stored listener array directly, and a listener mutates that same array mid-iteration (by calling `off()` on another listener, including one registered earlier in the array), the iteration can skip listeners it should have run — concretely, removing an element shifts every subsequent element's index down by one, so an ascending index-based loop that has already processed index `i` will, after a removal at some index `< i`, find that the listener formerly at index `i+1` has shifted into index `i`, and the loop's next step (index `i+1`) skips over it. The fix is having `emit` take a shallow copy of the listener array *before* starting to iterate, and iterate that copy — any mutation to the live, stored array during the emit affects only *future* emits, leaving the array being iterated *right now* stable and unaffected.

The trap: fixing this by iterating the live array *backward* instead of forward — this actually does prevent the specific "skip a listener due to an earlier removal" failure mode (since removing an element ahead of the current backward-moving position doesn't affect indices already visited), but it's a fragile, direction-dependent fix that breaks again the moment someone adds a *new* listener mid-emit instead of removing one, or if a future refactor changes the iteration direction back — a snapshot-based fix is robust regardless of what kind of mutation happens or in which direction iteration proceeds.

---

**Q (High): How would you prevent one listener throwing an exception from stopping the rest of the listeners from executing?**

Answer: Wrap each individual listener's invocation (not the entire `emit` loop as a whole) in its own `try/catch`, so an exception thrown by one listener is caught and handled (logged, or re-emitted as a dedicated `'error'` event) without that exception propagating up through the loop and aborting the remaining iterations. This is the difference between wrapping the *loop body* per-iteration versus wrapping the *loop* as a single unit — wrapping the whole loop in one `try/catch` still stops at the first thrown error, since the `catch` only fires after control has already left the loop entirely.

The trap: "just wrap `emit` in a try/catch" — this catches the *first* listener's error but still terminates the loop before reaching subsequent listeners, which doesn't actually solve the stated problem (every other listener still fails to run); the fix specifically requires per-listener error isolation, not per-`emit`-call isolation.

---

**Q (Medium): How does forgetting to remove listeners cause real memory leaks in production applications?**

Answer: An event emitter is typically a long-lived object (a global app-wide event bus, a WebSocket connection wrapper, a shared data store) that outlives any individual component or view subscribing to it. When a component registers a listener (`emitter.on('event', this.handleEvent)`) and is later torn down (unmounted, navigated away from) without a corresponding `emitter.off('event', this.handleEvent)` call, the emitter's internal listener array keeps a live, reachable reference to that listener function — and if the listener is a bound method or a closure that captures `this` (the component instance) or other component-local variables, the entire retained object graph (the component instance, its state, any DOM nodes or large data it references) stays reachable from the GC's perspective via the emitter, even though the UI has "removed" the component. Repeated mount/unmount cycles of the same component type (e.g., navigating to and from a page repeatedly) compound this — each cycle adds another live listener, retaining another full copy of everything that component's closure touches, growing memory usage indefinitely.

The trap: assuming "the component was removed from the DOM" is the same as "the component is garbage-collectable" — DOM removal and JavaScript reachability are two separate mechanisms, and an event emitter (or any long-lived registry: timers, subscriptions, caches) is exactly the kind of thing that can keep a "removed" component reachable and therefore un-collectable.

---

**Q (Medium): How would you implement `emit` to indicate whether any listener actually handled the event?**

Answer: Have `emit` return a boolean — `true` if at least one listener was registered (and therefore invoked) for that event, `false` if there were zero listeners — computed simply by checking whether the listener array for that event exists and is non-empty before running the loop. This mirrors Node's actual `EventEmitter.prototype.emit` return value convention, and it's useful for callers that want to detect "nobody is listening for this" as a distinct, actionable case (e.g., logging a warning if a critical lifecycle event has no subscribers, which might indicate a wiring bug elsewhere in the app) rather than treating "zero listeners ran" and "one or more listeners ran, but none did anything interesting" as indistinguishable.

The trap: conflating "did any listener run" with "did any listener *handle* the event in some semantic sense" — a boolean based purely on listener *count* can't know whether a given listener's internal logic considered itself to have "handled" anything (that would require every listener to opt into a shared, richer contract, like returning a truthy value the emitter aggregates — a reasonable but meaningfully more involved extension beyond what Node's own convention provides).

---

**Q (Low): How would you add a `maxListeners` warning, similar to Node's `EventEmitter`, to help catch leak-prone code?**

Answer: Track a configurable threshold (Node's default is 10) and, in `on()`, after pushing the new listener, check whether that event's listener array length has crossed the threshold — if so, emit a warning (via `console.warn`, or a dedicated internal `'newListenerLeak'`-style signal) naming the event and the current listener count, on the theory that a single event legitimately having more than ~10 listeners is unusual enough to often indicate a bug (most commonly, the exact leak pattern described above: a component's mount hook registering a new listener on every mount, without a matching `off` on unmount, so the count climbs over the app's lifetime rather than staying constant). This is explicitly a heuristic, not a hard limit — it doesn't prevent registration past the threshold, it just surfaces a signal a developer can investigate.

The trap: implementing it as a hard cap that *throws* or silently refuses to register the 11th listener — Node's own implementation deliberately only *warns*, because there are legitimate use cases with many listeners on one event (e.g., a very popular application-wide event), and a hard cap would break those instead of just flagging the more common leak scenario for investigation.

---

**Q (Low): How would you make `emit` asynchronous — running listeners as microtasks instead of synchronously — and what would change?**

Answer: Instead of calling each listener directly inside the `emit` loop, wrap each call in a microtask (e.g., `queueMicrotask(() => listener.apply(this, args))`, or resolve a `Promise.resolve().then(...)` per listener) so listeners run after the current synchronous execution context finishes, but before the next macrotask (timer, I/O callback) runs. The main behavioral changes: `emit()` itself now returns before any listener has actually executed (so code immediately after an `emit()` call can no longer assume side effects from listeners have already happened — a real breaking change for any caller relying on synchronous ordering), and if listener execution order matters, it's preserved (microtasks queued in order run in that same order), but the *timing* relative to the rest of the synchronous call stack is fundamentally different, which affects debugging (stack traces from an async-emitted listener's error no longer show the original `emit()` call site as directly).

The trap: presenting this purely as an isolated technical change without naming the real semantic consequence — synchronous emit's biggest practical property (callers can rely on listeners having already run by the time `emit()` returns) is exactly what's given up, and that's a significant enough behavioral change that it isn't a safe drop-in replacement for existing synchronous-emit-based code without auditing every call site.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement `on`/`off`/`once`/`emit` with method chaining from memory
- [ ] Can explain exactly why `emit` must iterate a snapshot, not the live listener array
- [ ] Can explain the `once`-wrapper-plus-`originalListener`-backpointer trick and why it's needed for `off` to work pre-fire
- [ ] Can explain why per-listener `try/catch` (not a single `try/catch` around the whole loop) is required to isolate listener failures
- [ ] Can explain, with a concrete mechanism (not just "it leaks"), how forgetting `off()` causes real memory retention
- [ ] Can explain why two separately-created arrow functions with identical bodies are never `===`-equal, and why that matters for `off()`

---
*Next: Promise.all / allSettled / race / any Polyfills — moving from a synchronous listener registry to genuine async coordination primitives, with their own order-preservation and empty-input edge cases.*
