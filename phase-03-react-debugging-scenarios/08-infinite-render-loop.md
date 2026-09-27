# Infinite Render Loop

## Quick Reference

| Cause | Mechanism | Fix |
|---|---|---|
| `setState` called unconditionally during render (not in an event handler/effect) | Every render triggers a state update, which triggers another render, forever | Never call `setState` directly in the render body outside of the specific "adjusting state during render" pattern — and even then, only conditionally |
| `useEffect` with no dependency array, setting state unconditionally | Runs after every render, sets state, triggers another render, runs again | Add a correct dependency array; guard the state update with a condition if needed |
| `useEffect` dependency is a new object/array/function every render | Effect's dependency array never "settles" — looks unchanged in code but is a new reference every time, so the effect re-runs every render | Memoize the dependency, or depend on primitive values extracted from it instead of the whole object |
| Derived state stored in `useState` and synced via effect, both changing each other | An effect updates state A based on state B; another effect (or the same one) updates B based on A | Compute derived values directly during render instead of storing them in separate state synced via effects |

## The Scenario

"This page just froze the tab — React's console is spamming 'Maximum update depth exceeded.' Find where the infinite loop is coming from and fix it, and explain in general terms what kinds of code patterns tend to produce this, so it doesn't come back somewhere else in this codebase."

## Clarifying Questions

- **Does the "Maximum update depth exceeded" warning include a component stack trace, and what's at the top of it?** React's own error message for this specific failure almost always names the component whose `setState` call is the immediate trigger — that's the fastest way into the actual code, rather than scanning the whole page's components blind.
- **Is the state update happening inside a `useEffect`, directly in the component's render body (not inside any handler or effect at all), or inside an event handler that itself gets re-invoked in a loop** (e.g., an `onScroll`/`onResize` handler that itself triggers a re-render that re-attaches/re-fires)? Each has a different specific mechanism, even though the visible symptom (the browser tab freezing, the same warning) looks identical.
- **If it's in a `useEffect`, what does the dependency array contain, and are any of those dependencies objects, arrays, or functions constructed inline in the render** (rather than primitives, or memoized references)? An effect whose dependency array *looks* stable in the code but actually receives a new reference every render is one of the most common non-obvious causes of this bug — it's not obviously "missing a dependency array," it's a dependency array that never evaluates as unchanged.
- **Is there a piece of state being kept in sync with another piece of state (or a prop) via an effect — i.e., "when A changes, set B," possibly in both directions?** Two-way synchronization implemented via effects going in a cycle (A's effect sets B, B's effect sets A) is a classic structural cause, and the fix is usually architectural (stop storing the derived value as separate state at all) rather than a small patch.
- **Does the loop happen immediately on mount, or only after some specific user interaction?** An immediate loop on mount points strongly at the render-body or mount-effect versions of this bug; a loop that only starts after an interaction narrows it toward a handler or an effect keyed off state that the interaction itself changes.

## Approach & Trade-offs

**The general shape of every infinite render loop: a render causes a state update, and that state update is guaranteed (or accidentally guaranteed) to cause another render whose conditions for triggering the same update are unchanged.** There are a few structurally different ways to arrive at that shape, and diagnosing this bug is mostly about figuring out *which* shape is present, because the fix differs meaningfully between them.

**Shape 1 — `setState` called unconditionally in the render body itself.** Calling a state setter directly in a component's function body (not inside a `useEffect`, not inside an event handler) executes it *during* render — React (in modern versions) detects same-render state updates and will re-render immediately to reflect them, and if that update is unconditional (always sets a value, regardless of what it currently is), every subsequent render repeats the same unconditional call, looping forever. The one sanctioned exception to "never call setState during render" is the documented "adjusting state during render" pattern for derived state resets (e.g., resetting a piece of state when a prop like an id changes) — and even that pattern is written with an explicit *condition* guarding the call (`if (prevId !== id) { setPrevId(id); setState(initial); }`), specifically so it only fires when something has actually changed, not on every render unconditionally.

**Shape 2 — a `useEffect` with a missing or incorrect dependency array, setting state unconditionally.** `useEffect(() => { setCount(count + 1); })` with *no* dependency array runs after every single render (mount and every re-render), and if it unconditionally updates state, every run triggers a re-render, which triggers the effect again (no dependency array means "run after every render," which includes the render caused by the effect's own last state update). Even `useEffect(() => { setX(...) }, [x])` where the effect *sets the very thing it depends on* is a self-triggering loop unless the setter is conditioned on the value actually needing to change (e.g., only setting it if it's currently different).

**Shape 3 — a dependency array containing a reference that's never stable, even though it "looks" correct in the code.** This is the least obvious version, and worth walking through explicitly: `useEffect(() => { setFiltered(items.filter(predicate)); }, [items, predicate])` looks like a normal, safely-scoped effect — but if `predicate` (or `items`, if it's derived inline in the parent, e.g., `<Child items={data.filter(d => d.active)} />`) is a new function/array reference constructed inline on every parent render, the dependency array *never* evaluates as "unchanged" between renders, because a fresh reference always fails the `Object.is` comparison React uses to decide whether to re-run the effect — so the effect re-runs every render, and if it sets state (even to a "logically" unchanged value, since state setters don't know or care that the *content* is the same, only that a new object reference was computed), that triggers another render, which reconstructs a new `predicate`/`items` reference again, forever.

**Shape 4 — two pieces of state synchronized bidirectionally through effects.** State A's effect sets state B when A changes; state B's effect sets A when B changes (directly, or transitively through several components/effects) — this creates a genuine update cycle where each side's change is a legitimate trigger for the other, and the only thing stopping an infinite loop is if the values eventually stop actually changing (e.g., if both converge to the same value and further updates become no-ops due to `Object.is` equality) — a design that works "by accident" until some input pattern causes the values to never converge, at which point it becomes a live infinite loop. The much more robust fix here is architectural: recognize that one of the two states is *derived* from the other and shouldn't be stored as independent state with an effect syncing it at all — it should be computed directly during render (or via `useMemo` if the computation is expensive), which has no update cycle because it's not a separate reactive update at all, just a value recalculated as part of the same render pass.

**Diagnosis in practice starts from the console's own stack trace, not from re-reading the whole component top to bottom.** React's "Maximum update depth exceeded" error includes a component stack indicating which `setState` call is the immediate trigger — that's the fastest entry point. From there, the question is specifically: is this call inside render, inside an effect, or inside a handler; if inside an effect, what's in the dependency array, and is every one of those dependencies actually stable across renders where its logical value hasn't changed.

## Solution

Reproducing Shape 1 — unconditional `setState` in the render body:

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  setCount(count + 1); // BUG: runs on every render, unconditionally, forever
  return <div>{count}</div>;
}
```

Fix — move it into an event handler, where it belongs, or (if genuinely needed during render) guard it with a real condition:

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

Reproducing Shape 3 — a dependency that's a fresh reference every render, even though the effect "looks" correctly scoped:

```tsx
function FilteredList({ items }: { items: Item[] }) {
  const [filtered, setFiltered] = useState<Item[]>([]);

  useEffect(() => {
    setFiltered(items.filter(i => i.active)); // BUG: `items` from the parent is a new array reference every render
  }, [items]);

  return <ul>{filtered.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
}

// Parent:
function Parent({ data }: { data: Item[] }) {
  // `data.filter(...)` constructs a brand-new array every render — even if `data` itself hasn't changed
  return <FilteredList items={data.filter(d => d.visible)} />;
}
```

Every render of `Parent` constructs a new `items` array reference (even when `data` and the filter's outcome are identical to last time), which makes `FilteredList`'s effect dependency array never stabilize, so the effect re-runs every render, calling `setFiltered` with a new array every time, triggering another render, forever.

Fix — the derived value doesn't need to be separate state or an effect at all; compute it directly during render (memoized, since filtering could be nontrivially expensive):

```tsx
function FilteredList({ items }: { items: Item[] }) {
  const filtered = useMemo(() => items.filter(i => i.active), [items]);
  // still re-computes if `items` is a new reference every render, but at least
  // it no longer triggers a *separate* setState-driven render on top of the one
  // already happening — no update loop, because there's no separate reactive state involved at all
  return <ul>{filtered.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
}

// And, addressing the actual root cause in the parent — stabilize `items` itself:
function Parent({ data }: { data: Item[] }) {
  const visibleData = useMemo(() => data.filter(d => d.visible), [data]);
  return <FilteredList items={visibleData} />;
}
```

Removing the `useState`/`useEffect` pairing for `filtered` entirely is the structural fix — there's no longer any state update triggered as a side effect of rendering, so there's no possibility of a render→setState→render cycle for this value at all; `useMemo` recomputes a value as part of the same render, not as a follow-up render of its own.

> **Check yourself:** In the first fixed version (`useMemo` in `FilteredList` alone, without fixing `Parent`), is the infinite loop actually gone, or just less severe? What's the difference between "recomputing `filtered` every render" (which `useMemo` without a stable `items` still does) and "triggering an additional render every time" (which the original `useState`/`useEffect` version did)?

## Root Cause

Every variant traces back to the same structural mistake: a state update that's supposed to happen only when something meaningfully changes is instead wired to fire on every render — either because it's unconditional, because its guarding dependency array never stabilizes (a fresh reference every time), or because it's part of a bidirectional sync where nothing structurally guarantees convergence.

## How to Prevent This Class of Bug

**Ask, for every `useEffect` that calls a state setter, "is there a real-world sequence of renders where this dependency array doesn't change, but the effect still runs anyway" — and separately, "is there a sequence where it changes on every single render."** The second question is really asking whether every dependency is either a primitive or a genuinely memoized reference — object/array/function dependencies constructed inline anywhere upstream are the recurring culprit.

**Prefer computing derived values directly during render (plain calculation, or `useMemo` if expensive) over storing them in `useState` synced via `useEffect`.** This is close to a general rule: if a value can be computed purely from existing props/state, it very rarely needs to be its own piece of `useState` kept in sync via an effect — that pattern is exactly what creates the possibility of an update cycle, and removing the separate state removes the possibility entirely, not just makes it less likely.

**Treat "Maximum update depth exceeded" as a message worth reading carefully, not just an indication to add a dependency array somewhere.** The accompanying stack trace names the actual component and, often, whether it's a render-phase or effect-phase update — using that information directly is faster than guessing.

## Gotchas

**Fixing the crash by wrapping the offending `setState` call in a `setTimeout`.** This "fixes" the immediate synchronous infinite loop (spreading it out over time so the browser doesn't freeze in one tick) but doesn't fix the underlying unconditional-update problem — it becomes a much slower, still-infinite loop, continuously consuming CPU and battery in the background, often without an obvious visible symptom, which is arguably worse than the crash, since it can go unnoticed for a long time.

**Adding a dependency array to a previously-array-less `useEffect`, but getting the array wrong** (missing a value that's actually read, or including an object/array that's a fresh reference every render) — technically "adding a dependency array" without actually fixing the loop, since an incorrect array can still fail to stabilize.

**Treating every instance of this bug as the same shape.** Applying the "just add `useMemo`" fix (correct for Shape 3) to a Shape 1 bug (unconditional `setState` directly in the render body, with no effect involved at all) doesn't address it — `useMemo` doesn't prevent a directly-called `setState` from running every render; the actual fix there is removing the unconditional call or guarding it, not memoizing something.

**Bidirectional state sync "working" in casual testing because the specific inputs tested happen to converge quickly**, masking a design that isn't actually guaranteed to terminate for all inputs — this is a particularly dangerous version because it can pass code review and ship, only to freeze in production under an input pattern that wasn't tested.

## Follow-up Questions

**Q (High): Explain precisely why `useEffect(() => { setX(...) }, [])`  — empty dependency array — does *not* cause an infinite loop, while `useEffect(() => { setX(...) })` — no dependency array at all — does.**

Answer: An empty dependency array tells React "this effect has no reactive dependencies; run it exactly once, after the initial mount, and never again" — so even though it calls a state setter, that setter call happens exactly once, and while it does trigger one additional re-render (to reflect the new state), the effect itself doesn't run again after that, because there's nothing in its (empty) dependency list to have changed. No dependency array at all means "run this effect after *every* render, with no comparison against anything" — every re-render (including the one caused by this effect's own last state update) causes the effect to run again, and if it always sets state, it always triggers another render, which runs the effect again, indefinitely. The presence and contents of the dependency array is what determines "run once" vs. "run after every render, forever, if it self-triggers" — omitting the array entirely is a fundamentally different (and, for anything setting state, almost always incorrect) instruction to React than providing an empty one.

The trap: describing both as "basically the same, just with or without brackets" — the difference (run-once vs. run-after-every-render) is not a minor stylistic nuance; it's the entire distinction between a safe pattern and a guaranteed infinite loop the moment the effect body sets state unconditionally.

---

**Q (High): A `useEffect`'s dependency array contains a dependency that is an object, and the effect only re-runs when a nested field of that object actually changes — yet the loop still occurs. Why might that be, even though "the object's meaningful contents aren't changing"?**

Answer: React's dependency comparison for `useEffect` uses `Object.is` (essentially reference/primitive equality) on each item in the array — it does not perform a deep comparison of an object's fields. If the object passed as a dependency is reconstructed (a new object literal, even with identical field values) anywhere upstream on every render — a very common case when a parent passes an inline-constructed object as a prop, or when the object is derived via `.map()`/spread/destructuring assignment freshly each render — then `Object.is(oldObjectRef, newObjectRef)` is `false` every time, regardless of whether the *fields* inside are unchanged, so React considers the dependency "changed" and re-runs the effect every render, even though nothing meaningful, from the perspective of someone reading the object's contents, actually changed. The fix is either to memoize the object at its source (`useMemo`, keyed on its actual primitive inputs) so its reference is stable when its logical contents haven't changed, or — often simpler and more robust — to depend on the object's individual primitive fields directly in the dependency array (`[obj.a, obj.b]`) rather than the object itself, sidestepping reference-equality concerns entirely since primitives compare by value.

The trap: assuming `useEffect`'s dependency comparison is "smart" about object contents in some way — it's a shallow, per-item reference/`Object.is` comparison, full stop; any perceived "it should know the contents didn't change" expectation is a misunderstanding of the mechanism, not a React limitation to work around with a deep-equality library (though a `useDeepCompareEffect`-style custom hook, doing exactly that deep comparison, is a legitimate, if less common, way to explicitly opt into that different behavior when truly needed).

---

**Q (High): Walk through why a bidirectional state-sync design (state A's effect updates state B, state B's effect updates state A) is fundamentally riskier than a one-directional derived-state pattern, even if it happens to work for the inputs you've tested.**

Answer: A one-directional derived pattern (state B computed directly from state A during render, or via a `useMemo` keyed on A) has no possibility of a cycle by construction — there's exactly one source of truth (A) and one derived read of it, and nothing ever "writes back" from the derived value to the source. A bidirectional sync, by contrast, creates a genuine dependency cycle: A's effect writing to B is itself something B's effect is watching, and B's effect writing to A is something A's effect is watching — whether this actually loops forever depends entirely on whether the *values themselves* stabilize (converge to a fixed point where each side's effect, seeing an unchanged value, doesn't fire an update) for the specific range of inputs exercised. This is inherently fragile: it's correct for exactly as long as every input scenario happens to converge, and there's no structural guarantee that will remain true as the component evolves or as edge-case inputs (which happened not to be tested) are eventually hit — a change to either effect's logic, or a new caller passing an input that doesn't happen to converge, can silently turn "seems to work" into "freezes the tab," without the code that broke it looking obviously wrong at a glance.

The trap: defending a working bidirectional sync as "fine, since it hasn't looped in practice" — the interviewer is testing whether the candidate recognizes that "hasn't looped yet" and "structurally cannot loop" are different guarantees, and that only the latter is actually safe to ship.

---

**Q (Medium): Does calling a state setter with the exact same value it already holds (e.g., `setCount(5)` when `count` is already `5`) inside an effect still risk an infinite loop?**

Answer: No, for primitive state — React's setters bail out of scheduling a re-render if the new value is `Object.is`-equal to the current value, so `setCount(5)` when `count` is already `5` is a no-op with respect to triggering another render, which is exactly what prevents certain effect patterns (an effect that "corrects" a value back to some canonical form, but only actually changes anything on the first correction) from looping forever — the second and subsequent runs see the setter called with a value equal to current state, and the update is skipped. This bailout does *not* extend the same way to object/array state, where `setState({ ...sameFields })` produces a new *reference* even if every field is logically identical — `Object.is` on two different object references is `false` regardless of their contents, so this specific safety net does not protect against unconditional object/array state updates in an effect, which is exactly why Shape 3 above is a real, common bug despite this bailout existing for primitives.

The trap: over-generalizing the primitive-equality bailout to imply "React always figures out if nothing really changed" — it only does so for values it can cheaply compare by identity/`Object.is`; anything object- or array-shaped needs the *code* to ensure reference stability itself (via memoization or by not reconstructing it unnecessarily), since React isn't going to deep-compare it for you.

---

**Q (Medium): How would `React.StrictMode` interact with debugging this specific bug — would it make an infinite loop more or less obvious?**

Answer: Strict Mode's development-only double-invocation of render and certain effects doesn't fundamentally change whether an infinite loop occurs — a genuinely self-triggering update cycle loops regardless of Strict Mode, since the looping condition (an unconditional or improperly-guarded state update) is present either way; Strict Mode's double-render/double-effect behavior might make an *already-present* loop manifest slightly faster or produce a marginally different-looking error trace (since each logical render/effect cycle is being run twice), but it isn't the *cause* of a loop that wouldn't otherwise exist, and disabling Strict Mode would not "fix" a genuine infinite-loop bug — it would just remove one of the tools (the deliberate double-invocation designed to surface exactly this class of non-idempotent-effect bug) that makes bugs like this easier to catch early in development.

The trap: concluding "disable Strict Mode, it's causing double-rendering which is causing the loop" — Strict Mode's double-invocation is bounded (it runs twice, not infinitely) and specifically exists to help surface effects that aren't safe to run more than once; a component that loops under Strict Mode has a real bug that would also eventually loop in production under the right conditions, just possibly less predictably.

---

## Self-Assessment

- [ ] Can name and distinguish the distinct structural shapes of this bug (render-body setState, dependency-less effect, unstable-reference dependency, bidirectional sync) rather than treating it as one undifferentiated failure mode
- [ ] Can explain precisely why an empty dependency array is safe while no dependency array at all is not, for an effect that sets state
- [ ] Can explain why `useEffect`'s dependency comparison is shallow/reference-based, and how that specifically causes a "looks stable but isn't" dependency to cause a loop
- [ ] Can articulate why computing a derived value directly during render (or via `useMemo`) structurally cannot produce this bug, unlike storing it as separate state synced via an effect
- [ ] Can explain the `Object.is` bailout for primitive state updates and why it does not extend to object/array state
- [ ] Can read a "Maximum update depth exceeded" stack trace and use it to locate the actual offending `setState` call rather than guessing

---
*Next: Prop Drilling Causing Stale Sibling State — a different failure mode again: not a loop, not extra renders, but state that's technically correct somewhere in the tree yet fails to reach a sibling that needs to react to it.*
