# Stale Closure in useEffect/useCallback

## Quick Reference

| Concept | Mechanism | Fix |
|---|---|---|
| Stale closure | A function created during render N captures that render's variable bindings; if it's invoked later (a timer, an event listener, an async callback) after state has moved to render N+1, it still sees N's values | Reference the latest value via the functional updater form (`setState(prev => ...)`), a `ref`, or by including the value in the effect's dependency array so the effect re-runs and recreates the closure |
| `useEffect` dependency array lying | Omitting a value the effect actually reads (often to "avoid extra runs") | Include every reactive value the effect reads; if that causes an effect to re-run too often, that's a signal to restructure the effect, not to lie about its dependencies |
| `useCallback` with stale deps | A memoized callback keeps referencing old state because its dependency array is empty or incomplete | Either include the correct dependencies (accepting the callback identity changes) or use the functional updater form to avoid needing the value as a dependency at all |
| ESLint `react-hooks/exhaustive-deps` | Statically flags effects/callbacks that read a value not listed in their dependency array | Treat every rule violation as a real signal for stale closures until proven otherwise — don't disable the rule to silence a legitimate warning |

## The Scenario

"There's a component with a 'like' button — clicking it should toggle the like count based on the *current* count. It works the first time. After that, it seems to always add exactly 1 to whatever the count was on the very first render, no matter how many times you click, or it visibly lags behind by a click or two. Find the bug."

## Clarifying Questions

- **Is the increment happening inside an event handler directly, or inside a callback that was set up once — e.g., inside a `useEffect` that attaches an event listener, a `setTimeout`/`setInterval`, or a memoized `useCallback` passed down and invoked later?** A closure captured once (in an effect that runs on mount, or a callback memoized with an incomplete dependency array) and then invoked repeatedly is exactly the shape of a stale-closure bug; a handler that's freshly created every render generally isn't susceptible to this in the same way.
- **What does the dependency array look like on the relevant `useEffect`/`useCallback`/`useMemo`, and does it match what the function body actually reads?** This is almost always where the bug lives — I'd want to see it directly rather than reason abstractly about "it worked once and now it doesn't."
- **Is the count being updated via `setCount(count + 1)` or `setCount(prev => prev + 1)`?** This distinction is the crux of the bug in the vast majority of cases like this — the former bakes in whatever `count` the closure captured at creation time; the latter always receives React's latest pending state regardless of what the closure captured.
- **Does the bug reproduce in a fresh reload every time, or does it depend on how many times you've clicked before reproducing?** Helps confirm whether it's a first-click-succeeds-then-freezes pattern (classic single stale closure from a one-time effect) versus an off-by-one/lag pattern (classic batching-plus-stale-capture combination, common with rapid clicks).
- **Is this in Strict Mode / development, where effects run twice?** Double-invocation of an effect in development can make an already-present stale closure bug more visibly wrong (or coincidentally mask it) versus a production build — worth ruling out before concluding the observed behavior is exactly what production does.

## Approach & Trade-offs

**What a closure actually captures.** Every time a component function runs, it creates an entirely new set of local variable bindings — `count` on render 3 is a genuinely different binding than `count` on render 4, even though they're conceptually "the same" state variable across the app's lifetime. A function defined during a given render (an event handler, an effect callback, anything) closes over *that render's* bindings specifically. If a closure is created once — say inside a `useEffect(() => { ... }, [])` that sets up an interval — and never recreated, every future invocation of that interval's callback still sees the `count` binding from whatever render created it, frozen forever, even though the component has re-rendered many times since and `count` has moved on.

**Why `setCount(count + 1)` is the classic version of this bug.** `count + 1` is evaluated using the `count` binding available to the closure *at the time the closure was created* — not at the time it's invoked. If that closure is the one attached in a mount-only effect, `count` is permanently whatever it was on mount (commonly `0`), so every subsequent invocation computes `0 + 1 = 1` and calls `setCount(1)` — hence "always resets to 1" rather than incrementing. Even in the simpler case of a plain event handler (recreated fresh every render, so not usually stale) doing several `setCount(count + 1)` calls in a row within the same handler, all three calls read the *same* `count` binding for that render and each independently computes `count + 1`, so three calls collapse into a single +1 rather than +3 — a related but distinct bug (batched multiple reads of one stale-within-a-single-render value, versus a closure stale *across* renders).

**Why `setCount(prev => prev + 1)` fixes it.** The functional updater form doesn't reference the closure's captured `count` at all — React calls the updater function at the moment it processes the state update, passing in whatever the *actual current pending state* is at that point (correctly chained even across multiple queued updates in the same batch). This sidesteps the staleness problem entirely, because the function no longer depends on which render's closure it happens to be running inside — it always operates on live state. This is the generally-preferred fix specifically *when the new state is purely a function of the previous state* — it doesn't help when the effect also needs some other current prop/value that genuinely isn't derivable from previous state alone.

**When the functional updater isn't enough — the dependency array as the real fix.** If an effect needs to read some other current value (not just "increment based on previous state" but, say, "post the current `count` to an API," or "compare `count` against some current `threshold` prop"), the functional updater trick doesn't apply, because the value needed isn't a function of the state being *set* — it's an independent value the effect needs to read. In that case, the actual fix is to include that value in the effect's dependency array, so the effect is torn down and re-created (with a fresh closure over the current bindings) every time that value changes. This is precisely what `react-hooks/exhaustive-deps` is checking for, and why disabling it to silence a warning, rather than fixing the underlying closure, just hides the bug instead of resolving it.

**`useCallback`'s dependency array is the same bug wearing a different hat.** A `useCallback(fn, [])` memoizes `fn` once, and if `fn`'s body reads a piece of state or a prop, that reference is frozen at whatever it was during the render that created the memoized version — every future invocation of that (referentially stable) callback still executes against those frozen bindings, even though the callback identity itself didn't change (which is misleadingly reassuring — "it's memoized, so it must be up to date" is exactly backwards; memoization freezes staleness, it doesn't prevent it). The fix is the same principle: either include the actual dependency and accept a new callback identity when it changes, or use the functional-updater trick if it's purely a function of previous state, or reach for a `ref` when neither applies cleanly (see the follow-up on `useEventCallback`-style patterns).

## Solution

Reproducing the bug — a "like" button whose count is updated from inside a one-time effect (contrived to isolate the pattern, but structurally identical to a WebSocket handler, an IntersectionObserver callback, or any listener set up once):

```tsx
function LikeButton() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const onKeyPress = (e: KeyboardEvent) => {
      if (e.key === 'l') {
        setCount(count + 1); // BUG: `count` is frozen at whatever it was on mount (0)
      }
    };
    window.addEventListener('keypress', onKeyPress);
    return () => window.removeEventListener('keypress', onKeyPress);
  }, []); // empty deps — this closure is created exactly once, over render-0's `count`

  return <button onClick={() => setCount(count + 1)}>Likes: {count}</button>;
}
```

Pressing "l" always sets the count to `1`, no matter how many times it's pressed — every invocation of `onKeyPress` uses the same frozen `count` binding from the initial render.

Fix 1 — functional updater, the right call here since the new value is purely a function of the previous one:

```tsx
useEffect(() => {
  const onKeyPress = (e: KeyboardEvent) => {
    if (e.key === 'l') {
      setCount(prev => prev + 1); // always operates on the true current state
    }
  };
  window.addEventListener('keypress', onKeyPress);
  return () => window.removeEventListener('keypress', onKeyPress);
}, []); // safe to keep empty — the effect no longer reads `count` at all
```

Fix 2 — correct dependency array, needed when the effect reads something that genuinely isn't derivable from previous state (e.g., logging the current count alongside some other current prop):

```tsx
function LikeButtonWithLogging({ userId }: { userId: string }) {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const onKeyPress = (e: KeyboardEvent) => {
      if (e.key === 'l') {
        setCount(prev => prev + 1);
        logEvent('like', { userId, currentCount: count }); // reads `count` directly — can't avoid the dependency
      }
    };
    window.addEventListener('keypress', onKeyPress);
    return () => window.removeEventListener('keypress', onKeyPress);
  }, [userId, count]); // effect re-subscribes whenever either value changes, so the closure is never stale
  // trade-off: the listener is torn down and re-attached on every count change —
  // acceptable for a keypress listener, potentially worth a ref-based pattern if
  // the subscribe/unsubscribe cost were expensive (see follow-up on refs).

  return <button onClick={() => setCount(prev => prev + 1)}>Likes: {count}</button>;
}
```

> **Check yourself:** In the "Fix 2" version, why does including `count` in the dependency array — rather than using `prev` inside `logEvent` — correctly solve the staleness, while still being a legitimate design trade-off (re-subscribing the listener) worth naming out loud in an interview?

## Root Cause

The root cause is a mismatch between *when* a closure is created (once, at effect-setup time, or once, at `useCallback` memoization time) and *when* it's invoked (later, potentially many renders after the state it references has moved on). React's rendering model — a new function call, new bindings, every render — makes this an inherent risk any time a function outlives the render that created it, which is exactly what timers, event listeners, subscriptions, and memoized callbacks all do by design.

## How to Prevent This Class of Bug

**Never disable `react-hooks/exhaustive-deps` to silence a warning without understanding why it's firing.** The rule exists specifically to catch this bug class statically, before it ships. A warning firing because "the effect would re-run too often if I fixed it properly" is a signal to restructure the effect (functional updater, splitting into multiple effects, moving non-reactive values into a `ref`) — not a signal to add an eslint-disable comment.

**Default to the functional updater form for any `setState` call that only needs the previous value.** `setCount(prev => prev + 1)` is strictly safer than `setCount(count + 1)` and costs nothing extra — making it the default habit removes an entire category of potential staleness before it can occur, rather than relying on remembering to fix it after the fact.

**Treat "I need the current value of something inside a long-lived callback, but including it as a dependency causes churn I don't want" as a `ref` problem, not a dependency-array problem.** A `ref` that's updated every render (`useEffect(() => { ref.current = value; })`, no dependency array — deliberately run every render) but read from inside a stable, rarely-recreated callback gives access to the always-current value without forcing the callback itself to be recreated. This pattern (sometimes wrapped as a custom `useEventCallback`/`useLatest` hook) is the standard escape hatch when the exhaustive-deps-correct answer would cause an effect to churn (re-subscribe/unsubscribe) more than is acceptable.

## Gotchas

**Fixing the visible symptom (the effect that sets up the listener) while missing a second stale closure nearby.** It's common for a component to have more than one place reading stale state — e.g., a `useCallback`-memoized handler passed to a child, sitting right next to the fixed effect, with the identical bug. Fixing one instance and declaring victory without checking for others is a shallow fix.

**"It works when I test it slowly but breaks under rapid clicks."** This is usually not a *new* bug but the same stale-closure/batching issue made more visible — rapid interactions increase the chance of a closure created several renders ago still being live and invoked, surfacing what a single slow click might not expose.

**Believing `useCallback`/`useMemo` memoization implies correctness.** A memoized function is exactly as stale as its dependency array allows — memoization is orthogonal to correctness; it only affects *identity stability*, not *which bindings the function body sees*.

**Adding a dependency, watching the effect now re-run "too often," and reflexively removing it again instead of restructuring.** This is the failure mode `exhaustive-deps` is trying to prevent — removing the dependency reintroduces the exact staleness bug that was just fixed; the correct response to "this re-runs too often now" is almost always a functional updater or a `ref`, not reverting the dependency array.

## Follow-up Questions

**Q (High): Explain exactly why `setCount(count + 1)` inside a `useEffect(() => {...}, [])` produces "always resets to 1" rather than a correct increment, in terms of closures and React's render model.**

Answer: `useEffect(fn, [])` runs `fn` exactly once, after the initial mount. The `fn` passed in is a closure created during that first render, and it captures the `count` binding as it existed *then* — `0`. Because the dependency array is empty, this effect never re-runs, so this closure — and its captured `count === 0` — is never recreated, no matter how many times the component subsequently re-renders with a new `count` value. Every future invocation of whatever this effect set up (an event listener, in the earlier example) calls `setCount(count + 1)` using that permanently-frozen `count`, i.e., `setCount(0 + 1)`, i.e., `setCount(1)`, every single time — which is why the displayed count appears to "reset to 1" repeatedly rather than incrementing.

The trap: describing the bug as "React isn't updating `count`" — React *is* updating `count` correctly; the bug is that a specific closure, created once, never gets a chance to see the update, because nothing recreated it. Misattributing the bug to React's state mechanism instead of to closure semantics is the shallow version of this answer.

---

**Q (High): When is `setState(prev => ...)` not sufficient to fix a stale closure, and what's the correct pattern instead?**

Answer: The functional updater form only has access to the *previous value of the specific state variable being updated* — it doesn't give access to other current props, other state variables, or anything else the closure might need. If a stale closure needs to read some other current value (a prop, a different piece of state, the current value of a ref) at the time it's invoked — not just "the previous version of the value I'm setting" — the functional updater doesn't help, because there's nothing to hand it the other value. In that case, the two real options are: include that value in the effect's/callback's dependency array (accepting that the effect will tear down and re-run, or the callback will get a new identity, whenever it changes), or store the value in a `ref` that's updated every render and read the `ref.current` inside the closure — refs are explicitly exempt from the closure-staleness problem because reading `ref.current` always reads the *current* mutable value, regardless of which render's closure is doing the reading (a ref's identity is stable across renders, but its `.current` content is mutable and shared).

The trap: reaching for the functional updater as a universal fix for "stale closure" bugs generically — it's specifically a fix for stale *state-being-set*, not stale *anything the closure reads*. A candidate who applies it correctly to the "increment count" case but doesn't recognize its limits when a different value is needed hasn't fully understood why it works.

---

**Q (High): Why does the `react-hooks/exhaustive-deps` ESLint rule sometimes suggest a dependency that, when added, causes an effect to run far more often than seems desirable — and what should you do about it, rather than disabling the rule?**

Answer: The rule is doing exactly its job — it's telling you the effect genuinely reads a value that changes on a schedule you may not have consciously accounted for, and any time that value changes without the effect re-running, the effect's closure is stale with respect to it. If adding the dependency causes churn that seems excessive (e.g., an effect re-subscribing a WebSocket connection every time an unrelated-feeling piece of state changes), the correct response is to ask *why* the effect depends on that value in the way it currently does — often the fix is splitting one effect that does several unrelated things into multiple effects with tighter, more accurate dependency sets, or moving a value that doesn't need to trigger a re-run into a `ref` (if the effect only needs its *latest* value at invocation time, not to *react* to its changes). Disabling the rule for that one line removes the compiler's ability to warn about this specific bug class going forward, on that specific effect, permanently — a much worse trade than the few minutes it takes to restructure the effect correctly.

The trap: treating "the linter is being annoying" and "the linter caught a real bug" as though they're commonly two different situations — in practice, the overwhelming majority of `exhaustive-deps` violations that people are tempted to suppress are the latter, and the rule has very few legitimate false-positive cases (the primary sanctioned exception being `useRef`-held values that are read but genuinely shouldn't trigger the effect, which the rule's own documentation, and the pattern of moving such values into a ref, addresses directly rather than via suppression).

---

**Q (Medium): Does this bug class exist in class components, or is it unique to hooks?**

Answer: The underlying issue — a callback capturing a value that later becomes stale — exists in a related form with class components too, though it manifests differently. A class component's methods typically read `this.state` at call time rather than closing over a snapshot, since `this` is a mutable reference to the whole instance, so a class method invoked later generally *does* see current state (there's no per-render closure the way hooks create). Where classes get an analogous bug is with `setState`'s own batching semantics — calling `this.setState({ count: this.state.count + 1 })` multiple times in a row within one handler suffers the same "reads a value that hasn't been updated yet within this batch" problem, for which class components have their own functional-updater equivalent: `this.setState(prevState => ({ count: prevState.count + 1 }))`. So the *specific* "closure captured over a stale render's bindings, and that closure outlives the render" bug is a hooks-and-closures phenomenon fairly specific to how function components work; the *general* "read the value before it's been updated" bug has always existed and has always had a functional-updater fix in both paradigms.

The trap: claiming "this is a hooks-only bug" without qualification — the *closure-capturing-a-render's-bindings* mechanism is specific to function components, but the broader "you read stale state and computed a new value from it" failure mode is older than hooks and shows up in different clothing in classes too.

---

**Q (Medium): Would using `useRef` instead of `useState` for the count avoid this problem entirely? What would you give up?**

Answer: Storing the count in a `ref` (`countRef.current`) does sidestep closure staleness, since reading `ref.current` always gets the live current value regardless of which render's closure does the reading — but it comes at the cost of losing React's reactivity for that value entirely: mutating a ref does not trigger a re-render, so the UI would not update to reflect the new count unless something else forces a re-render afterward (e.g., also calling a dummy `setState` to force one, which is an anti-pattern once you're doing it purely to trigger a render around a ref-held value that logically should have been state all along). Refs are the right tool when a value needs to be read with up-to-date freshness inside a long-lived closure but *does not* need to drive rendering (e.g., tracking "is the component still mounted" for an async cleanup check, or holding the latest callback for an event-emitter-style pattern) — they're the wrong tool when the value is meant to be displayed or otherwise drive what's rendered, which a visible "Likes: N" counter clearly is.

The trap: proposing refs as a general-purpose fix for stale closures without acknowledging the re-render trade-off — a candidate who says "just use a ref" for a value that's supposed to be visibly displayed hasn't thought through what refs actually don't do.

---

## Self-Assessment

- [ ] Can explain, precisely, why a closure created inside a mount-only effect captures that render's bindings permanently
- [ ] Can identify when the functional updater form (`prev => ...`) fully resolves a staleness bug versus when it doesn't
- [ ] Can articulate the `ref`-based escape hatch for reading a current value inside a long-lived callback without triggering re-subscription
- [ ] Can justify never disabling `exhaustive-deps` to silence a real warning, and can describe the restructuring options instead
- [ ] Can distinguish this hooks-specific closure-staleness mechanism from the more general "read-before-update" batching bug that also exists in class components

---
*Next: Race Condition in Fetch (Autocomplete Overwrite Bug) — a different consequence of the same underlying theme: an async callback outliving the render (or even the component) that created it, and acting on stale assumptions about what's still current.*
