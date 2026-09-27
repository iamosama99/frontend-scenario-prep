# Memory Leak From Uncleaned Subscriptions

## Quick Reference

| Leak Source | Mechanism | Fix |
|---|---|---|
| Event listener added without removal | `window.addEventListener` in an effect/mount, no matching `removeEventListener` | Return a cleanup function from `useEffect` that removes exactly what was added |
| Interval/timeout not cleared | `setInterval`/`setTimeout` started without a matching `clearInterval`/`clearTimeout` | Store the id, clear it in the effect's cleanup |
| Third-party subscription (WebSocket, observable, store) | `.subscribe()` called without holding/calling the returned unsubscribe handle | Capture the unsubscribe function, call it in cleanup |
| Detached DOM node retained by closure | A ref or captured DOM node held onto by a still-alive closure after the component unmounts | Null out refs / avoid capturing DOM nodes in long-lived closures beyond what's needed |
| Growing cache/module-level array | Pushing to a module-scope array/Map on every mount with nothing ever removing entries | Scope the collection to component lifetime, or explicitly remove entries on cleanup |

## The Scenario

"Users are reporting that after leaving this page open and navigating around the app for a while — opening and closing this same modal repeatedly, say — the tab gets sluggish and eventually the browser tab's memory usage climbs steadily in the dev tools. Nothing crashes outright, it just gets slower and slower. Find where the leak is and fix it."

## Clarifying Questions

- **Does the component in question set up any subscriptions to something outside React's own lifecycle — event listeners on `window`/`document`, `setInterval`/`setTimeout`, WebSocket connections, a third-party store's `.subscribe()`, an `IntersectionObserver`/`ResizeObserver`, or a custom event emitter?** Any of these can outlive the component if not explicitly torn down, since none of them are automatically garbage-collected just because the component that created them unmounted — they hold references (directly, or via a closure) that keep the component's whole tree from being freed even after it's removed from the DOM.
- **Is the component being mounted and unmounted repeatedly (a modal opening/closing, a route being navigated to and away from), or is it a single long-lived component that just accumulates state over time?** This determines whether the leak is "N times the size of one component's subscriptions, growing with N mounts" (the modal case) or "one component's own internal accumulation growing unboundedly the longer it stays mounted" (e.g., an array being pushed to on every event with nothing ever trimmed) — different root causes, same symptom.
- **Does `useEffect` in the suspect component return a cleanup function at all, and if so, does it actually undo everything the effect set up — every listener added, every interval started, every subscription made?** A cleanup function existing but only partially undoing the effect's side effects is a very common miss — for example, removing one of two listeners added, or clearing an interval but not calling an unsubscribe.
- **Is the memory growth visible in the Chrome DevTools Memory tab as a straightforward heap size increase over time, and does taking heap snapshots before/after repeated mount-unmount cycles show retained instances of the component (or its DOM nodes) that should have been garbage collected?** This is the actual diagnostic step, not a guess — comparing two heap snapshots and filtering for "objects allocated between snapshot 1 and snapshot 2" that are still retained is how you'd actually locate the leak in practice, and I'd want to walk through doing that rather than eyeballing the code for a "probably right" answer.
- **Is this observed in development (with Strict Mode's double-mount) or exclusively in production?** Strict Mode intentionally mounts, unmounts, and remounts components once in development specifically to help surface exactly this class of bug (an effect whose cleanup doesn't properly undo its setup) — worth confirming the leak isn't purely a dev-mode artifact of something otherwise idempotent, though a component that actually fails Strict Mode's mount/unmount/remount cycle cleanly is, by design, one that has a real cleanup bug.

## Approach & Trade-offs

**Every effect that touches something outside React's own reactive system needs a mental "who owns this, and who's responsible for releasing it" pass.** React manages the lifecycle of its own component tree and the DOM nodes it renders — when a component unmounts, React removes its DOM nodes and lets them be garbage collected. But anything a component's effect *reaches outside* of React to set up — a global event listener, a timer, a subscription to some external store or socket — is not automatically torn down by React just because the component unmounted. React only guarantees to *call* an effect's cleanup function; it's the developer's responsibility to make sure that cleanup function actually undoes everything the effect set up. A leak is what happens when this responsibility isn't fully discharged: something external keeps a reference alive (directly, or via a closure that captured component state/props/DOM refs) that would otherwise have let the garbage collector reclaim everything the unmounted component was holding onto.

**Why a leaked subscription drags the whole component instance with it, not just the subscription itself.** A closure passed to `.subscribe()`, `addEventListener`, or `setInterval` typically references things from the component's scope — state setters, props, local variables, sometimes DOM refs. As long as that closure is reachable (which it is, for as long as whatever it was registered with holds a reference to it — the event target, the interval's internal registry, the store's subscriber list), everything the closure closes over is also reachable, and therefore *not* eligible for garbage collection, even though the component itself has been unmounted and removed from the DOM tree. This is why a single un-cleaned-up subscription can effectively leak an entire component instance's closure scope, not just "a small callback" — the actual memory cost is often much larger than the leak's origin suggests.

**Diagnosing this for real means using heap snapshots, not code review alone.** Code review can spot an obviously missing cleanup function, but confirming an actual leak (versus a false alarm, or confirming the fix actually worked) means: open Chrome DevTools' Memory tab, take a heap snapshot, trigger the suspected leaky action several times (open/close the modal N times), force a GC (the trash-can icon), take a second snapshot, and use the "Comparison" view to see what's still retained that shouldn't be. Retained objects matching the leaking component's constructor name, with a "Retainers" chain leading back to something like a global `window` event listener list, directly names both the leak's existence and its root cause — this is the actual professional workflow, and being able to describe it (not just recite "use `useEffect` cleanup") is a stronger signal than reciting the fix pattern alone.

**Symmetric setup/teardown as the actual discipline, not just "remember to return a cleanup function."** The bug rarely comes from *forgetting* a cleanup function exists — it comes from the cleanup function not being a precise mirror of the setup: adding two listeners but removing one, subscribing to a store but discarding the returned unsubscribe handle without calling it, starting an interval whose id is captured in a variable that then gets rebound or lost before cleanup can reference it. The discipline that actually prevents this class of bug is treating "everything the effect body adds, the returned cleanup function removes — one to one" as close to a hard rule, and reviewing new effects specifically by checking that symmetry rather than eyeballing "is there a return statement."

## Solution

Reproducing the bug — a component that subscribes to a global event bus and starts a polling interval, with no cleanup at all:

```tsx
function LiveTicker({ symbol }: { symbol: string }) {
  const [price, setPrice] = useState<number | null>(null);

  useEffect(() => {
    const handlePriceUpdate = (data: { symbol: string; price: number }) => {
      if (data.symbol === symbol) setPrice(data.price);
    };
    priceEventBus.on('update', handlePriceUpdate); // BUG: never calls priceEventBus.off

    const intervalId = setInterval(() => {
      pingServerForFreshness(symbol);
    }, 5000); // BUG: never cleared

    // no return — no cleanup function at all
  }, [symbol]);

  return <div>{symbol}: {price ?? 'Loading...'}</div>;
}
```

Every time this component mounts (e.g., a modal containing it is opened), a new `handlePriceUpdate` closure is registered on `priceEventBus` and a new interval starts. Unmounting the component (closing the modal) removes it from the DOM, but the event bus still holds a reference to `handlePriceUpdate` — which closes over `setPrice`, which closes over the (now-orphaned) component's fiber — and the interval keeps firing forever, calling `pingServerForFreshness` for a component that no longer exists. Opening and closing the modal ten times leaves ten leaked listeners and ten leaked intervals, all still running.

Fix — symmetric setup and teardown:

```tsx
function LiveTicker({ symbol }: { symbol: string }) {
  const [price, setPrice] = useState<number | null>(null);

  useEffect(() => {
    const handlePriceUpdate = (data: { symbol: string; price: number }) => {
      if (data.symbol === symbol) setPrice(data.price);
    };
    priceEventBus.on('update', handlePriceUpdate);

    const intervalId = setInterval(() => {
      pingServerForFreshness(symbol);
    }, 5000);

    return () => {
      priceEventBus.off('update', handlePriceUpdate); // exact mirror of the `.on` call above
      clearInterval(intervalId);
    };
  }, [symbol]);

  return <div>{symbol}: {price ?? 'Loading...'}</div>;
}
```

Every mount's effect now has a matching teardown, run automatically by React before the effect re-runs (if `symbol` changes) and on unmount — no listener or interval outlives the component instance that created it.

A subtler version — a subscription-returning API (common with observables/state libraries), where the return value itself *is* the unsubscribe function:

```tsx
useEffect(() => {
  const subscription = externalStore.subscribe(state => {
    setLocalState(state.relevantSlice);
  });
  return () => subscription.unsubscribe(); // easy to forget when `.subscribe()`'s return value looks optional
}, []);
```

> **Check yourself:** If `handlePriceUpdate` in the fixed version were defined *outside* the effect (e.g., as a component-level function, not recreated inside the effect body) but the effect still called `priceEventBus.on('update', handlePriceUpdate)` and `off` with what looks like the same reference, could a subtle version of this bug still occur depending on how `handlePriceUpdate` is defined? What would you check?

## Root Cause

The root cause is an asymmetry between what an effect sets up and what its cleanup function tears down — anything reaching outside React's own managed lifecycle (global listeners, timers, external subscriptions) needs an explicit, matching teardown, because nothing about unmounting a component automatically severs references held by external systems the component's effects registered with.

## How to Prevent This Class of Bug

**Treat "does this effect's cleanup function exactly undo what the effect body did" as a required review question for every new `useEffect`**, not just a nice-to-have — specifically checking each thing added (listener, timer, subscription) has a corresponding line in the cleanup removing it.

**Run components through a manual mount/unmount/remount stress test during development** — React 18 Strict Mode already does one cycle of this automatically in development for exactly this reason; deliberately mounting and unmounting a suspect component many times in a row (a simple test harness, or just opening/closing a modal repeatedly) while watching the Memory tab is a cheap, direct way to catch leaks before they reach production.

**Prefer APIs and patterns that make forgetting cleanup harder, not just possible to remember.** A custom hook like `useEventListener(target, event, handler)` that internally guarantees the effect/cleanup pairing centralizes the correct pattern in one reviewed place, rather than relying on every call site remembering to write the pairing correctly from scratch every time.

## Gotchas

**A cleanup function that exists but is asymmetric — removes one of two listeners, clears one of two intervals.** Looks like the bug is fixed (there *is* a cleanup function, and *some* things get cleaned up) while still leaking the remainder.

**Passing a freshly-created inline function to both `.on()` and `.off()`, where the two references aren't actually the same function.** `element.addEventListener('click', () => foo())` followed by `element.removeEventListener('click', () => foo())` does *not* remove the original listener — these are two different function objects, and `removeEventListener` requires reference equality with what was originally passed to `addEventListener`. The handler needs to be a named, stored reference used identically in both calls.

**Relying on garbage collection to "eventually" clean up a leaked closure once nothing else references it — while something still does.** As long as the external subscription/listener list holds the closure, it (and everything it closes over) remains reachable — GC only reclaims genuinely unreachable memory; a leak, by definition, is a reference that keeps something reachable longer than intended, and no amount of waiting fixes that on its own.

**Only testing a component's happy-path single mount, never its unmount-and-remount behavior**, especially for components that live inside modals, tabs, or conditionally-rendered routes that get mounted and unmounted repeatedly over a session.

**Assuming a memory leak "isn't a big deal" because it's slow to manifest.** In a long-running single-page app session (a dashboard left open all day, an admin panel), a slow steady leak from a frequently-remounted component compounds over hours into real, user-visible degradation — the "it just gets slower" symptom described in the scenario is exactly that compounding effect.

## Follow-up Questions

**Q (High): Explain precisely why a component that has actually unmounted (removed from the DOM, no longer rendering) can still be retained in memory due to an uncleaned subscription — walk through the reference chain.**

Answer: The subscription target (an event bus, `window`, a store) holds a reference to the handler function that was registered with it — that reference is what keeps the handler reachable from the JS garbage collector's perspective, since the subscription mechanism's internal list of subscribers is itself reachable from a long-lived root (`window` is always reachable; a module-level event bus instance is reachable for the lifetime of the module). That handler function, being a closure, holds references to everything from its enclosing scope that it uses — commonly the component's state setters, and via those, the component's fiber/instance data structures that React uses internally. So the reachability chain is: long-lived root → subscription's internal subscriber list → handler closure → captured component-scope references → effectively the whole component instance's retained data. React removing the component's DOM nodes from the visible tree does nothing to break this chain, because the chain doesn't go through the DOM at all — it goes through the subscription mechanism, entirely outside React's management.

The trap: describing this vaguely as "the component doesn't get garbage collected" without being able to name the actual reference chain that keeps it reachable — the specific mechanism (closures capturing scope, held alive by an external subscriber list) is what demonstrates real understanding of *why* this happens, not just that it does.

---

**Q (High): Walk through how you'd actually confirm, using browser DevTools, that a suspected component is leaking — not just review the code and guess.**

Answer: Open Chrome DevTools' Memory panel, select "Heap snapshot," and take an initial snapshot as a baseline. Trigger the suspected leak path several times in a controlled way — for a modal, open and close it, say, 10 times — then force garbage collection (the trash-can/broom icon in the Memory panel, which requests an immediate GC pass) and take a second snapshot. Switch the second snapshot's view to "Comparison" against the first — this shows objects allocated between the two snapshots that are *still present* after the forced GC, which (for a properly-cleaned-up component) should be close to zero for that component's constructor/class name, and (for a leaking one) shows a nonzero, often exactly-proportional-to-10 count of retained instances. Clicking into one of those retained objects and inspecting its "Retainers" panel shows the actual reference chain keeping it alive — tracing that chain back typically lands directly on the offending subscription or listener, naming the exact fix needed rather than requiring a guess.

The trap: describing this only as "I'd check if `useEffect` has a cleanup function" — that's a code-review step, not a diagnostic one, and doesn't actually confirm whether a leak exists, how large it is, or where in a large codebase it's coming from when the leak isn't in code you're already looking at.

---

**Q (High): `element.addEventListener('click', handler)` in an effect, with `element.removeEventListener('click', handler)` in the cleanup — but the leak persists. What's a likely reason this specific pairing still fails, given both calls reference `handler`?**

Answer: The most likely cause is that `handler` isn't actually the *same function reference* at both call sites, despite looking identical in the code — commonly because `handler` is defined as an inline arrow function inside the effect body on every effect run (fine, as long as the *same* closure instance is captured by both the `addEventListener` and the `removeEventListener` call within that same effect execution — which it should be, if both reference the same local variable) versus a more subtle version where `handler` is redefined outside the effect at the component's top level as a plain function declaration that gets recreated on every render (e.g., `const handler = () => {...}` written directly in the component body, not memoized) and the effect closes over "whichever `handler` was current when the effect last ran" — if the effect's dependency array doesn't include `handler` and doesn't recreate it, this specific pairing risk mostly resolves itself within one effect execution; the actual classic failure mode is code that defines the handler fresh in *two separate places* (once for `.addEventListener`, again for `.removeEventListener`, e.g., across two different effects or lifecycle hooks) — `removeEventListener` uses strict reference equality against what was passed to `addEventListener`, so two structurally-identical-looking-but-distinct function objects will not match, and the removal silently does nothing (no error is thrown — it just fails to find a match and no-ops).

The trap: assuming "the code looks like it should cancel out" is sufficient without checking that both sides are referencing the literal same function object — `removeEventListener`'s reference-equality requirement is a specific, checkable fact, not a stylistic nicety, and reasoning about this bug requires actually tracing whether the two `handler` mentions resolve to the same object.

---

**Q (Medium): Does React's automatic cleanup of DOM nodes on unmount mean refs pointing to those DOM nodes are also automatically nulled out / made eligible for GC?**

Answer: The DOM nodes themselves become eligible for garbage collection once nothing else references them and they're removed from the document — React removing them from the tree is necessary but not sufficient; if something *else* (a `ref` object that a long-lived closure captured a copy of, or a variable stored in a module-level array) still holds a direct reference to that DOM node, the node remains reachable and therefore not collected, exactly analogous to the subscription case, just with a DOM node as the retained object instead of component state. React itself does not proactively "null out" ref `.current` values pointing at DOM nodes purely because a component unmounted — for a `ref` created with `useRef` and only ever read from within the component's own render/effects, this is a non-issue since the ref object itself becomes unreachable once the component instance is gone; it only becomes a real leak if that ref (or the DOM node it points to) was captured by something with a longer lifetime than the component.

The trap: assuming React automatically "cleans up" every ref on unmount in some blanket sense — the actual guarantee is much narrower (React updates certain ref props to `null` for `ref` callback patterns during unmount in specific cases) and isn't a general substitute for making sure nothing else is independently holding onto the same DOM node or its containing closure.

---

**Q (Medium): Is a `console.log` reference held inside a long-running `setInterval` callback (never cleared) considered a memory leak, even if the interval's logic itself doesn't obviously grow any data structure?**

Answer: Yes — an uncleaned interval is a leak regardless of what its callback body does internally, because the interval's continued existence itself keeps the closure (and everything it captured) alive indefinitely, whether or not that closure's *own* logic accumulates additional memory over time. Even a callback that does nothing but `console.log('tick')` still retains whatever scope it closed over (commonly, indirectly, the entire component instance it was defined inside), and the interval itself continues consuming a small amount of scheduler overhead forever. The "does it grow unboundedly" question is really about *severity/rate* of the leak, not about *whether* it qualifies as one — a component leaking a fixed, small closure ten times (from ten mount/unmount cycles) is a smaller-impact leak than one that also grows an internal array unboundedly, but both are genuine leaks in the sense of "memory retained longer than the logical lifetime of what created it," and the fix (clear the interval in cleanup) is identical either way.

The trap: treating "leak" as synonymous with "unboundedly growing" — a leak is fundamentally about retained-longer-than-intended, and a fixed-size leak repeated across many mount/unmount cycles (the "open and close a modal repeatedly" scenario from the prompt) is exactly how a fixed-size-per-instance leak becomes a visibly growing total.

---

## Self-Assessment

- [ ] Can trace the exact reference chain (subscriber list → closure → component scope) that keeps an unmounted component's memory reachable
- [ ] Can describe the heap-snapshot-comparison DevTools workflow for confirming a leak, not just reciting "add a cleanup function"
- [ ] Can identify the reference-equality requirement behind `removeEventListener` failing silently when handler references don't match
- [ ] Can write a `useEffect` whose cleanup function is an exact, symmetric mirror of everything the effect body registers
- [ ] Can explain why a leak's severity (fixed-size vs. unboundedly growing) doesn't change whether it qualifies as a leak

---
*Next: Context Causing App-wide Re-renders — a performance-flavored bug that, unlike this one, doesn't leak memory but instead wastes render cycles across an entire subtree due to a single shared Context value.*
