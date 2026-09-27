# Why Is This Component Re-rendering Constantly?

## Quick Reference

| Cause | Mechanism | Fix |
|---|---|---|
| New object/array/function identity every render | Inline `{}`, `[]`, or arrow functions created fresh each render, passed as props | Hoist constants outside the component; wrap with `useMemo`/`useCallback` when identity must be stable across renders |
| Context value object recreated every render | `<Ctx.Provider value={{ a, b }}>` — a new object literal every render | Memoize the provider's `value` with `useMemo` |
| `React.memo` on the wrong layer | Memoizing a component whose parent still re-renders it with new prop identities | Fix the identity churn at the source; `memo` alone doesn't help if props aren't referentially stable |
| Unnecessary state colocated too high | State that only one deep child cares about lives in a shared ancestor | Push state down to the component that owns it, or split into a separate subtree |
| Reading React DevTools "why did this render" wrong | Assuming a highlighted render = a bug | Confirm it's actually expensive/visible before treating it as a problem — not every render is worth chasing |

## The Scenario

"Here's a dashboard component — profiler says it's re-rendering on every single keystroke in a search box that's nowhere near it in the tree, and it's noticeably janky. Nothing about its own props or state looks like it should be changing. Find out why, and fix it without just throwing `React.memo` at everything."

## Clarifying Questions

- **What does the component actually render, and is the re-render itself the problem, or is it a *slow* render that happens to be triggered too often?** A component re-rendering 60 times a second is harmless if it renders in under a millisecond — the actual complaint is jank, and jank comes from expensive work happening at the wrong frequency, not from render-count in isolation. I want to know if profiling shows a long "self time" per render before assuming the fix is "render less."
- **Is the search box's state (and the dashboard's re-render) connected through Context, a shared parent, or a global store (Redux/Zustand)?** Each has a different failure mode — Context re-renders every consumer on any value change regardless of what part of the value a consumer reads; a shared parent re-renders every child unless those children are memoized with stable props; a store-based subscription re-renders only components that select the changed slice, if selectors are used correctly.
- **Are props being passed down as object/array/function literals, or are they primitives?** This is usually the actual root cause when "nothing about its props looks like it changed" — the *values* may be equal, but the *references* aren't, and React's default reconciliation (and `memo`'s shallow comparison) cares about reference equality, not deep equality.
- **Has anyone already tried wrapping things in `React.memo`, and did it help?** If `memo` was already tried and didn't fix it, that's a strong signal the culprit is upstream — a new value or callback identity being generated on every parent render defeats `memo` immediately, and jumping straight to `useMemo`/`useCallback` everywhere without finding *which* prop is unstable is how people end up "memoizing everything" without understanding why any of it works.
- **Is this in development mode with Strict Mode on?** React 18 Strict Mode intentionally double-invokes render (and effects) in development to surface side-effect bugs — a component appearing to render twice in dev, once in prod, is expected behavior, not a bug, and conflating the two wastes debugging time.

## Approach & Trade-offs

**Don't start with `React.memo` — start with the Profiler.** The instinct to reach for `memo` immediately is understandable but backwards: `memo` only prevents a re-render when the *parent* re-renders but props are referentially unchanged. If the actual problem is that the parent is re-rendering with genuinely new prop identities every time, wrapping the child in `memo` does nothing — its shallow prop comparison will correctly determine the props "changed" (new object reference) and re-render anyway. The React DevTools Profiler, with "record why each component rendered" enabled, tells you directly: was it a prop change, a state change, a context change, or a parent re-render with no prop/state/context change at all (the "hooks changed" vs "the parent rendered" distinction matters here).

**Chase the re-render to its actual trigger, not its symptom.** If a `SearchBox` re-render is somehow causing a sibling `Dashboard` to re-render, the two are connected through something shared — most commonly a Context provider sitting above both, or a shared parent whose own state includes something the search box also touches (search text living in a common ancestor, for instance). The fix target is wherever that connection is, not the `Dashboard` itself. Memoizing `Dashboard` might mask the symptom if its own props happen to be stable, but if the actual issue is a Context value being reconstructed as a new object every render, every consumer of that Context re-renders regardless of `memo` on the consumer, because Context propagation bypasses the parent-child prop-equality check entirely — it re-renders every subscribed consumer on any new Provider `value` reference, full stop.

**Referential equality vs. deep equality — this is the crux of nearly every "why is this re-rendering" bug.** JavaScript's `===` compares objects, arrays, and functions by reference, not contents. `{ a: 1 } === { a: 1 }` is `false`. Every time a component function runs, any object/array/function literal written inline in its JSX or body is a *brand-new* value, even if its contents are identical to last render's. `React.memo`'s default comparator uses `Object.is` per prop (essentially `===`), so a new object literal passed as a prop always fails that comparison and forces the child to re-render, regardless of `memo`. The fix isn't "compare deeply instead" (that trades a cheap reference check for an expensive recursive one, and doesn't scale) — it's to *stop generating new references* when the underlying value hasn't actually changed, via `useMemo` for values and `useCallback` for functions, or by hoisting genuinely static values outside the component entirely so they're created once, ever.

**`useMemo`/`useCallback` are not free, and not always the right call.** They add their own overhead (running the memoization comparison, retaining the previous value) and are only a net win when (a) the value/function is passed to something that actually does reference-equality checks that matter (a memoized child, a `useEffect` dependency array, a Context value) and (b) recomputing it is more expensive than the memoization bookkeeping itself. Wrapping every function in `useCallback` "just in case" is a common overcorrection — if a callback is only ever used inline in the same component's JSX (e.g., an `onClick` on a plain `<button>`), there's no downstream consumer doing an equality check, so `useCallback` adds overhead for zero benefit.

**Sometimes the right fix is state colocation, not memoization at all.** If the search box's typed value is being stored in a state variable that lives in a shared ancestor of both the search box and the dashboard — because "it seemed convenient to keep it near the top" — every keystroke re-renders that ancestor and, by default, every descendant of it, dashboard included. Moving that state down into the `SearchBox` component itself (or a small wrapper that owns just the input and its state) means keystrokes only re-render the subtree that actually needs to see them. This is often a bigger win than any amount of memoization, because it removes the re-render at its source rather than making the unwanted re-render cheap.

## Solution

Reproducing the bug — state colocated too high, plus an unstable Context value, both feeding into an unrelated component:

```tsx
// BEFORE: search state lives in the app shell, dashboard is a sibling
function AppShell() {
  const [query, setQuery] = useState('');
  const theme = { mode: 'dark', accent: 'blue' }; // new object every render

  return (
    <ThemeContext.Provider value={theme}>
      <SearchBox value={query} onChange={setQuery} />
      <Dashboard /> {/* re-renders on every keystroke, and isn't even wired to `query` */}
    </ThemeContext.Provider>
  );
}
```

Every keystroke updates `query` in `AppShell`, which re-renders `AppShell` — and by default, every non-memoized descendant, including `Dashboard`, even though `Dashboard` never reads `query`. Separately, `theme` is a new object literal every render, so *any* `ThemeContext` consumer re-renders on every `AppShell` render regardless of whether `mode`/`accent` actually changed — compounding the problem.

Fixing the state colocation first — this alone removes most of the unnecessary re-renders:

```tsx
// AFTER: search state moves into its own component, scoped to what needs it
function AppShell() {
  const theme = useMemo(() => ({ mode: 'dark', accent: 'blue' }), []); // stable reference

  return (
    <ThemeContext.Provider value={theme}>
      <SearchBoxContainer />
      <Dashboard />
    </ThemeContext.Provider>
  );
}

function SearchBoxContainer() {
  const [query, setQuery] = useState('');
  // keystrokes now only re-render this small subtree
  return <SearchBox value={query} onChange={setQuery} />;
}
```

`Dashboard` no longer re-renders on keystrokes at all — not because it's memoized, but because nothing it depends on changes anymore. The `theme` object is memoized to a stable reference (`useMemo` with an empty dependency array, since it's genuinely static here), so `ThemeContext` consumers don't re-render spuriously either.

Where `memo` *does* earn its place — a component that legitimately receives new data periodically, but whose expensive rendering shouldn't repeat when a sibling changes for unrelated reasons:

```tsx
const Dashboard = React.memo(function Dashboard({ widgets }: { widgets: Widget[] }) {
  // expensive chart rendering, etc.
  return <div>{widgets.map(w => <Widget key={w.id} data={w} />)}</div>;
});

// The parent must also avoid passing a new `widgets` array reference on every render
// if `widgets` didn't conceptually change — e.g., derive it with useMemo from source state,
// not by mapping/filtering inline in JSX on every render.
```

> **Check yourself:** If `Dashboard` is wrapped in `React.memo` but its parent still passes `widgets={data.filter(d => d.active)}` inline in JSX, will `memo` prevent the re-render when an unrelated sibling's state changes? Why or why not?

## Gotchas

**Wrapping a component in `memo` without fixing the props feeding it.** This is the single most common miss — `memo` is treated as a fix rather than a diagnostic dead-end that reveals where the *real* unstable reference lives.

**Assuming Context consumers re-render only when the specific field they read changes.** They don't — Context re-renders every consumer of a Provider on any change to the `value` reference, even if a given consumer only destructures one field out of a larger value object. Splitting one large Context into several smaller, narrowly-scoped Contexts (or using a selector-based state library) is the actual fix when this granularity matters.

**Confusing Strict Mode's intentional double-render/double-effect behavior (dev only) with a real bug.** Burning time debugging a "double re-render" that only happens in development and disappears in a production build is a wasted detour — check the build mode before treating dev-only double-invocation as the culprit.

**Treating every profiler-highlighted render as something to eliminate.** A component that re-renders often but cheaply (a `<span>{count}</span>`) is not a performance problem. Chasing render-count to zero everywhere is itself a time sink with no user-facing payoff — the bar is "does this cause visible jank," not "does this render exist."

**Not distinguishing `useCallback`'s cost from its benefit.** Slapping `useCallback` on every function "for optimization" without a downstream consumer that does reference-equality checks on it adds overhead (the hook itself, the dependency array comparison) for no measurable gain — sometimes making things marginally slower, not faster.

## Follow-up Questions

**Q (High): `React.memo` didn't stop the re-renders. Walk through your diagnostic process for figuring out why.**

Answer: First, confirm what `memo` actually checks — a shallow, per-prop `Object.is` comparison against the previous render's props. If the child still re-renders, one of three things is true: (1) at least one prop's *reference* changed even if its logical value didn't (most likely — an inline object/array/function literal, or a value derived via `.map()`/`.filter()`/spread inline in the parent's JSX, which creates a new reference every parent render); (2) the child has its own internal state or Context subscription causing the re-render independent of props entirely, in which case `memo` was never going to help since `memo` only gates the parent-driven path; or (3) a custom comparison function was passed to `memo` and has a bug (e.g., comparing the wrong fields, or a typo causing it to always return `false`). I'd use the Profiler's "why did this render" panel to see exactly which of these it is rather than guessing, then trace whichever unstable prop it flags back to where it's constructed in the parent.

The trap: proposing to just add `useMemo`/`useCallback` everywhere upstream without first confirming, via the Profiler, which specific prop is actually unstable — that's treating the symptom with a shotgun instead of finding the actual reference that's churning.

---

**Q (High): Explain, precisely, why `<Ctx.Provider value={{ a, b }}>` written inline is a performance bug even if `a` and `b` never change.**

Answer: `{{ a, b }}` is an object literal evaluated fresh every time the enclosing component function runs — even if `a` and `b` hold the exact same primitive values as last render, the object *wrapping* them is a new reference (`{} !== {}`). React's Context propagation compares the Provider's `value` by reference (not deep equality) to decide whether to notify consumers of a change; a new reference on every render means every subscribed consumer re-renders every single time the Provider's parent re-renders, regardless of whether the data inside actually changed. The fix is `useMemo(() => ({ a, b }), [a, b])` — this returns the *same* object reference across renders as long as `a` and `b` (compared by `Object.is`, so fine for primitives) haven't changed, so consumers correctly skip re-rendering when nothing relevant changed.

The trap: saying "wrap it in `useMemo`" without being able to explain *why* — specifically, that Context's bailout mechanism is reference equality on the whole `value`, not a per-field comparison, which is also why splitting a large Context object into multiple smaller Contexts is sometimes the better fix when different consumers care about different fields.

---

**Q (High): A parent re-renders 10 children on every state update, but only 1 child's data actually changed. What are the options, and what would you actually reach for first?**

Answer: Options, roughly in order of how surgical they are: (1) colocate the state closer to whichever child actually owns it, so the update no longer touches the parent at all — the most effective fix when it's applicable, since it removes the re-render trigger rather than mitigating its blast radius; (2) wrap the 9 unaffected children in `React.memo` with stable prop references, so they bail out of re-rendering even though the parent re-renders — appropriate when the children genuinely need to stay siblings under one parent and their own render cost is non-trivial; (3) use a state management approach with selector-based subscriptions (Zustand, Redux with `useSelector`, Jotai atoms) so components subscribe directly to the specific slice of state they care about, bypassing the parent-child re-render chain entirely — worth it when this pattern repeats across many places in the app, overkill for a single isolated case. I'd reach for (1) first if the state's "natural home" genuinely is closer to the single affected child (which is often the actual design smell being surfaced), and only reach for (2) when the state legitimately needs to live where it is for other reasons.

The trap: jumping straight to "wrap everything in `memo`" as a universal answer without considering that the state's *location* might be the actual design issue — `memo` is a mitigation for re-renders you can't avoid, not a substitute for avoiding them when you can.

---

**Q (Medium): Does calling `setState` with the same value it already holds trigger a re-render?**

Answer: No — React bails out early if the new state value is `Object.is`-equal to the current value, for a given state variable set via `useState`/`useReducer`, *before* re-rendering the component. This applies per state update call, though it's worth noting the component function will still have been scheduled to run once (React calls the component to check, then bails on committing if nothing changed) — for expensive components this "call but don't commit" behavior is itself sometimes relevant, but the DOM is not touched and children are not re-rendered as a result. This bailout does not apply to Context value changes, which always propagate if the value's reference changed, even if it's "equal" by some deep-equality standard.

The trap: assuming this bailout also means "if I call `setQuery(query)` with the unchanged string, absolutely nothing happens at all" — the component function itself is still invoked to perform the comparison; what's avoided is the render being committed and propagated to children, not the call to the function.

---

**Q (Medium): How does React 18 Strict Mode's double-invocation of render functions in development relate to this kind of bug, if at all?**

Answer: It doesn't directly cause re-render bugs, but it can make diagnosing them confusing if you're not accounting for it — in development, Strict Mode intentionally renders components (and re-runs certain effects, mount/cleanup/mount) twice in a row to help surface side effects that aren't idempotent (a common real bug class: an effect that pushes to an array or increments a counter as a side effect will visibly double up under Strict Mode, which is the point — it's designed to expose exactly that class of bug). This is invisible in a production build. If someone is debugging "why does this render twice" using `console.log` inside a component body during development, they need to first rule out Strict Mode's intentional double-invocation before concluding there's an actual extra-render bug.

The trap: "fixing" a perceived double-render bug by disabling Strict Mode instead of first confirming whether the doubling even exists in production — this removes a diagnostic tool without addressing the underlying question.

---

**Q (Low): Would using `useReducer` instead of several `useState` calls change how many times a component re-renders?**

Answer: Not inherently — both trigger a re-render on any state change (subject to the same `Object.is` bailout described above). What `useReducer` changes is how *related* updates are batched conceptually: with several independent `useState` calls representing pieces of what's logically one state object, it's easier to accidentally trigger multiple separate state-update calls (and, historically outside of React 18's automatic batching, multiple renders) for what's really one logical transition; a single `dispatch` to a reducer that updates several fields at once is naturally one state transition, one render. In React 18+, automatic batching largely closes this gap for updates that happen within the same synchronous event handler regardless of `useState` vs `useReducer`, so the practical re-render-count difference is smaller than it used to be — the bigger benefit of `useReducer` here is conceptual clarity about what constitutes one atomic state transition, not raw render-count savings.

The trap: claiming `useReducer` "reduces re-renders" as a blanket performance fact — in modern React with automatic batching, the render-count difference is often negligible; the real benefit is elsewhere (colocated transition logic, easier testing of state transitions in isolation).

---

## Self-Assessment

- [ ] Can explain why `React.memo` fails silently when fed an inline object/array/function-literal prop, referencing `Object.is`
- [ ] Can state precisely why Context re-renders every consumer on any `value` reference change, independent of which field a consumer reads
- [ ] Can use the Profiler's "why did this render" output to distinguish a parent-driven re-render from a state/context-driven one
- [ ] Can justify choosing state colocation over `memo`/`useMemo` when a state's "natural home" is closer to one child than to a shared ancestor
- [ ] Can explain why not every profiler-highlighted re-render is a bug worth fixing
- [ ] Can distinguish Strict Mode's dev-only double-invocation from a genuine re-render bug

---
*Next: Stale Closure in useEffect/useCallback — a different flavor of "the component isn't behaving right," where the render count is fine but the *data* the component acts on is frozen in time.*
