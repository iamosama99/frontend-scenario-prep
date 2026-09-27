# Context Causing App-wide Re-renders

## Quick Reference

| Cause | Mechanism | Fix |
|---|---|---|
| One large Context, many unrelated fields | Any field changing re-renders every consumer, even ones reading unrelated fields | Split into multiple, narrowly-scoped Contexts by change frequency/concern |
| Provider `value` recreated every render | New object/array literal passed to `value` on every parent render | `useMemo` the value object, keyed on its actual dependencies |
| High-frequency state in Context (mouse position, form input) | Every update re-renders the entire consumer subtree, not just what visually needs it | Keep high-frequency state local, or out of React state entirely, and only surface derived/settled values via Context |
| `memo` on consumers, without fixing the Provider | Doesn't help — Context propagation bypasses the parent-child shallow-prop-equality check `memo` relies on | Fix at the Provider (splitting/memoizing value); `memo` is not effective against Context-driven re-renders the way it is against prop-driven ones |
| `useContext` selector pattern absent | Every consumer re-renders on any Context change, no way to subscribe to "just this field" | Use `useSyncExternalStore`-based selection, or a state library, when this granularity matters |

## The Scenario

"We have a `UserContext` that holds the current user's profile, their notification count, their theme preference, and a few UI flags — basically 'stuff related to the current user.' It's used all over the app. Someone just noticed that updating the notification count every few seconds (a polling badge) is causing the *entire* app, including totally unrelated parts of the UI like a settings panel three levels away, to visibly re-render and briefly lag. Diagnose it and propose a fix, including trade-offs between the options."

## Clarifying Questions

- **What does the `UserContext.Provider`'s `value` prop actually look like in the code — is it a single object literal constructed inline in the provider component's render, or something already memoized?** This is almost always where to look first — Context's re-render mechanism is based on the `value` reference changing, so if it's a fresh object every render (even one where only `notificationCount` logically changed), *every* consumer re-renders regardless of which field they actually read.
- **How many distinct "concerns" does this one Context actually bundle together, and do they change at meaningfully different frequencies?** `theme` (changes rarely, maybe once per session), `user profile` (changes rarely, once per login), and `notificationCount` (changes every few seconds via polling) are very different change-frequency profiles bundled into one object — that mismatch is usually the actual design issue, independent of any memoization fix.
- **Do the affected consumers (like the distant settings panel) actually read `notificationCount` at all, or do they only read `theme`/`user`?** If they don't read the changing field at all, this is squarely the "Context re-renders every consumer on any value change, not just consumers of the changed field" behavior — confirming this rules out "maybe they're supposed to re-render" and confirms it's a bug, not a feature.
- **Is the notification count polling implemented as `setInterval` + `setState` inside whatever component owns the Context provider, or pushed via a WebSocket/subscription?** Doesn't change the fix, but matters for where the state update actually originates and whether there's an opportunity to decouple it earlier in the chain.
- **Is there a reason all of this state needs to be *one* Context specifically (shared update transaction, tightly coupled derived values), or was it bundled together mostly for convenience of "put related things in one place"?** If nothing actually requires them to update atomically together, that's a strong signal splitting is the right fix rather than trying to preserve one Context and just memoizing harder.

## Approach & Trade-offs

**Context's re-render granularity is "any value change, all consumers" — there is no built-in per-field subscription.** This is the single fact underlying the whole bug: `useContext(UserContext)` subscribes a component to *the entire Provider*, not to any particular field within its value. When the Provider re-renders with a new `value` reference, React re-renders every component that calls `useContext` on that Context, full stop — it does not inspect which specific properties of the value object a given consumer actually destructures and used in its own render. This is different from props, where a child only re-renders if the specific props passed to *it* change (and `memo` can intercept that) — Context propagation is a separate mechanism that bypasses the parent-child prop-equality check entirely. This is why wrapping the distant settings panel in `React.memo` does nothing here: `memo` guards against unnecessary re-renders triggered by a parent re-rendering with unchanged props; it does nothing to prevent a re-render triggered by a Context this component subscribes to directly via `useContext`.

**Splitting one Context by change-frequency/concern is usually the more durable fix than trying to memoize around a single Context.** `theme`, `user profile`, and `notificationCount` have almost nothing to do with each other structurally — they're bundled only because they're all "current-user-related." Splitting them into `ThemeContext`, `UserProfileContext`, and `NotificationCountContext` (three separate providers, which can still be nested together in one `<AppProviders>` wrapper component for convenience at the composition root) means a component that only reads `theme` subscribes only to `ThemeContext`, and is structurally incapable of re-rendering when `notificationCount` changes, because it never subscribed to that Context at all. This isn't a memoization trick — it's removing the coupling that caused the problem in the first place. The trade-off is more Providers to wire up and more Context imports scattered around, a modest ergonomic cost for a real correctness/performance win.

**Memoizing the Provider's `value` is necessary but not sufficient on its own.** Even after splitting Contexts, each individual Provider's `value` still needs `useMemo` if it's ever constructed as an object literal (which, with more than one field per Context, it usually is) — otherwise the split doesn't fully pay off, since `NotificationCountContext`'s own `value` object might still be recreated on unrelated parent re-renders even when `notificationCount` itself hasn't changed. `useMemo(() => ({ count }), [count])` ensures the reference is stable across renders where `count` hasn't changed, which matters because even within `NotificationCountContext`'s own consumers, an unrelated parent re-render shouldn't force a Context-driven re-render if the actual notification count value is unchanged.

**Keeping genuinely high-frequency state (think: every-few-seconds polling, or worse, high-frequency events like mouse position/scroll) out of Context-consumed-by-many entirely.** Even a perfectly split, perfectly memoized `NotificationCountContext` still means *every* consumer of it re-renders every time the count changes — which is correct and expected if there are only one or two small consumers (a badge icon), but becomes its own problem if many components across the app end up subscribing to it "just in case." For genuinely high-frequency values, the more scalable pattern is to keep the raw high-frequency state as close to where it's needed as possible (or in a ref, or in an external store outside React state) and only surface a deliberately-throttled, settled, or derived value through Context when broader consumption is needed — trading raw freshness for reduced re-render fanout, which for a notification badge (a count, not something requiring sub-second precision) is almost always an acceptable trade.

**The selector pattern, for when per-field subscription is genuinely needed at scale.** Some state libraries (Redux via `useSelector`, Zustand, Jotai) solve this problem structurally by letting a component subscribe to a *derived slice* of shared state — re-rendering only when that specific slice's value changes, computed via a selector function, using `useSyncExternalStore` under the hood rather than plain Context propagation. This is the right reach when an app has many pieces of shared state with genuinely fine-grained, high-churn subscription needs across a large component tree — it's a heavier architectural commitment than "just split the Context," appropriate once Context-splitting alone stops being practical (dozens of independent pieces of shared state, or components needing to combine slices from several of them with memoized derived values).

## Solution

Reproducing the bug — one Context, one un-memoized value object, mixing low- and high-frequency state:

```tsx
type UserContextValue = {
  profile: { name: string; id: string };
  theme: 'light' | 'dark';
  notificationCount: number;
};

const UserContext = createContext<UserContextValue | null>(null);

function UserProvider({ children }: { children: React.ReactNode }) {
  const [profile] = useState({ name: 'Ada', id: 'u1' });
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const [notificationCount, setNotificationCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      fetchNotificationCount().then(setNotificationCount);
    }, 5000);
    return () => clearInterval(id);
  }, []);

  // BUG: new object literal every render — and every render of THIS provider
  // (triggered by notificationCount ticking every 5s) forces every consumer,
  // anywhere in the tree, to re-render — including ones that only read `theme`.
  return (
    <UserContext.Provider value={{ profile, theme, notificationCount }}>
      {children}
    </UserContext.Provider>
  );
}

function SettingsPanel() {
  const { theme } = useContext(UserContext)!; // doesn't read notificationCount at all — re-renders anyway
  return <div className={theme}>Settings...</div>;
}
```

Fix — split by concern, memoize each Provider's value:

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');
const UserProfileContext = createContext<{ name: string; id: string } | null>(null);
const NotificationCountContext = createContext<number>(0);

function AppProviders({ children }: { children: React.ReactNode }) {
  const [profile] = useState({ name: 'Ada', id: 'u1' });
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const [notificationCount, setNotificationCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      fetchNotificationCount().then(setNotificationCount);
    }, 5000);
    return () => clearInterval(id);
  }, []);

  return (
    <UserProfileContext.Provider value={profile}> {/* primitive-ish, stable across notification ticks */}
      <ThemeContext.Provider value={theme}>
        <NotificationCountContext.Provider value={notificationCount}>
          {children}
        </NotificationCountContext.Provider>
      </ThemeContext.Provider>
    </UserProfileContext.Provider>
  );
}

function SettingsPanel() {
  const theme = useContext(ThemeContext); // only subscribed to ThemeContext now
  return <div className={theme}>Settings...</div>; // never re-renders on notification count changes
}

function NotificationBadge() {
  const count = useContext(NotificationCountContext); // isolated to just this concern
  return <span className="badge">{count}</span>;
}
```

`profile`, `theme`, and `notificationCount` are passed directly as each Provider's `value` (not wrapped in an object) since each is now independently a primitive or a stably-referenced object — no `useMemo` needed for the primitives; `profile` itself would need `useMemo`/stable construction if it were derived inline rather than held in its own `useState`. `SettingsPanel` now structurally cannot re-render due to a notification count change, because it never subscribes to `NotificationCountContext` at all.

> **Check yourself:** If `profile` in the fixed version were instead computed inline every render as `{ name: user.name, id: user.id }` (spread/derived from some other state, not held directly in its own `useState`), what would break, and what's the one-line fix?

## Gotchas

**Splitting Contexts but forgetting to also memoize a still-object-shaped `value`.** If any of the split Contexts still carries more than one field as an object, the split alone doesn't fully solve the problem unless that object's reference is also stabilized with `useMemo` — half a fix.

**Wrapping consumers in `React.memo` and expecting it to help.** As covered above, `memo` doesn't intercept Context-driven re-renders — this is one of the most common wrong turns taken when first encountering this bug, because `memo` genuinely is the right tool for the *prop-driven* version of unnecessary re-renders, and it's easy to over-generalize that fix to a mechanism it doesn't apply to.

**Splitting Contexts along the wrong axis** — e.g., splitting by "how the data is used" rather than "how often it changes," ending up with a Context that still mixes a rarely-changing field with a frequently-changing one under a different name. The axis that actually matters for this bug is change frequency (and, secondarily, consumer overlap), not topical grouping.

**Assuming the fix requires abandoning Context altogether in favor of a state library.** For a moderate number of genuinely independent concerns, split-Context is usually sufficient and simpler than introducing a new dependency — reaching for Redux/Zustand purely to solve *this specific* bug, without a broader need for their other capabilities, is often more machinery than the problem warrants.

**Not re-checking whether the *Provider component itself* re-renders for unrelated reasons**, which — even with per-field Contexts — still recomputes and re-passes each Provider's value; if the provider component's own parent re-renders it for an unrelated reason and any of the values are recomputed inline rather than read from stable state/refs, the splitting work can be partially undone.

## Follow-up Questions

**Q (High): Why doesn't wrapping a Context consumer in `React.memo` prevent it from re-rendering when the Context value changes?**

Answer: `React.memo` intercepts re-renders that are triggered by *the component's parent re-rendering and passing it props* — its comparison function checks whether the props passed down are shallowly equal to last time, and skips re-rendering if so. A component subscribed to a Context via `useContext` is not being re-rendered because its parent passed it new props; it's being re-rendered because React's Context propagation mechanism directly notifies every subscribed consumer when the Provider's `value` reference changes — a completely separate code path from the parent-child prop-passing/reconciliation `memo` hooks into. `memo` has no visibility into, and no ability to intercept, a Context subscription's own re-render trigger; the component will re-render regardless of what `memo` wrapper surrounds it, because the re-render isn't coming from its props at all.

The trap: proposing `React.memo` as a fix for Context-driven re-renders — this is exactly the kind of answer that sounds plausible (`memo` prevents re-renders, so wrap it in `memo`) but reveals not understanding that Context propagation and prop-based reconciliation are two structurally different re-render trigger mechanisms.

---

**Q (High): Concretely, why does splitting one Context into several actually solve the problem, rather than just relocating it?**

Answer: Splitting works because Context subscription is per-Context, not per-field — a component that calls `useContext(ThemeContext)` is only ever notified of `ThemeContext` Provider value changes; it has no subscription relationship whatsoever to `NotificationCountContext`, and therefore cannot be re-rendered by a change to it, structurally, not just by convention. Before splitting, every consumer of the single combined `UserContext` was, by definition, subscribed to *all* of `profile`, `theme`, and `notificationCount` changing together, because they were all packaged into one `value` reference — there was no way for a consumer to express "I only care about `theme`" to the Context mechanism itself. Splitting doesn't optimize away unnecessary re-renders through comparison or memoization tricks; it removes the coupling that made a `theme`-only consumer accidentally depend on `notificationCount` in the first place, at the subscription level.

The trap: describing the fix vaguely as "it's more efficient" without being able to name specifically *why* — the precise mechanism is that Context subscription grain is per-Provider, and splitting Providers changes what a given `useContext` call is actually subscribed to.

---

**Q (High): You've split the Contexts, but the settings panel still re-renders occasionally when nothing it cares about seems to have changed. What would you check next?**

Answer: First, check whether the Context it *does* legitimately subscribe to (say, `ThemeContext`) has a `value` that's genuinely stable across the renders in question — if `theme` itself hasn't changed but the Provider still passes a *new* value each time (e.g., if the value were wrapped as `{ theme }` inline rather than passed directly, recreating the object every render regardless of whether `theme`'s actual value changed), that alone would still trigger the re-render even post-split. Second, check whether `SettingsPanel`'s parent is re-rendering for unrelated reasons and `SettingsPanel` itself isn't wrapped in `memo` — a Context split fixes Context-driven re-renders specifically; it does nothing about ordinary parent-driven re-renders via props, which is a separate, still-valid case for `memo` if `SettingsPanel`'s own props are otherwise stable. Third, use the Profiler's "why did this render" panel directly rather than guessing — it will distinguish "rendered because of a Context change" from "rendered because parent re-rendered" definitively.

The trap: assuming that because a Context split was done, *all* future re-renders of that component must be Context-related — the split fixes one specific re-render *cause*; it doesn't make the component immune to the ordinary parent-driven re-render path that `memo` (a different tool) addresses.

---

**Q (Medium): How would `useSyncExternalStore`-based state (or a library like Zustand) solve this differently from split Contexts, at a mechanical level?**

Answer: These approaches maintain state entirely outside React's own render tree (in an external, mutable store object), and each subscribing component calls a `selector` function (e.g., `useStore(state => state.notificationCount)`) that the library uses to derive exactly the slice that component cares about; the library's internal subscription tracks, per component, only the selector's *output* — and only triggers that specific component's re-render when the selector's output actually changes (typically via a shallow-equality check on the selected value across store updates), regardless of how many other unrelated fields also changed in the same store update. This achieves genuinely per-field (or per-derived-value) subscription granularity without needing to physically split state into separate Context objects at all — one single store can back arbitrarily many independent selector-based subscriptions, each isolated from unrelated field changes, which is a finer and more flexible granularity than Context splitting can achieve without an impractical number of separate Contexts.

The trap: describing this as "basically the same as splitting Context, just with different syntax" — the mechanical difference (selector-based output comparison vs. per-Context subscription) is what allows these libraries to scale to dozens or hundreds of independent pieces of state without a proliferation of Provider components, which Context-splitting alone can't do gracefully past a certain point.

---

**Q (Medium): If `notificationCount` needs to be read by many different, scattered components across the app, does isolating it into its own Context actually reduce the total number of re-renders that occur when it changes?**

Answer: No — and this is worth stating clearly rather than implying the split is a silver bullet: every component that legitimately reads `notificationCount` will still, correctly, re-render every time it changes, regardless of whether it's in its own Context or bundled with others — that's expected, necessary behavior, not a bug. What the split eliminates specifically is components that *don't* read `notificationCount` being needlessly dragged along for the ride. If the actual problem were that *too many components legitimately need this value and it changes too often*, splitting the Context doesn't address that at all — the fix in that case is elsewhere: reducing the update frequency itself (polling less often, or pushing updates via a mechanism that only fires on real changes rather than a fixed interval), or restructuring which components actually need live access to the count versus a periodically-refreshed snapshot.

The trap: presenting Context-splitting as a general performance win for "a value that changes often" — it only helps the *unrelated-consumer* portion of the re-render fanout; consumers that genuinely need the value still pay the re-render cost every update, and that part of the problem, if it's excessive, needs a different fix (reduce update frequency, or restructure ownership of that state).

---

## Self-Assessment

- [ ] Can explain precisely why Context subscription is per-Provider, not per-field, and why that's the root mechanism behind this bug
- [ ] Can explain concretely why `React.memo` does not intercept Context-driven re-renders
- [ ] Can design a Context split by change-frequency/concern rather than by superficial topical grouping
- [ ] Can identify when a Context split isn't enough on its own (still-object-shaped, unmemoized `value`) and needs `useMemo` too
- [ ] Can describe, mechanically, how a selector-based store achieves finer subscription granularity than Context splitting
- [ ] Can distinguish "unnecessary re-renders from unrelated consumers" (fixed by splitting) from "necessary re-renders from too-frequent legitimate updates" (not fixed by splitting)

---
*Next: Key Prop Misuse — State Bleeding Between List Items — a different but related identity problem: not about Context subscriptions, but about React's reconciliation losing track of which list item is which.*
