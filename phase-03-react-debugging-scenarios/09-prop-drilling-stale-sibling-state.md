# Prop Drilling Causing Stale Sibling State

## Quick Reference

| Cause | Mechanism | Fix |
|---|---|---|
| Sibling state duplicated instead of lifted | Each sibling holds its own local copy of conceptually shared state, updated independently | Lift the state to the nearest common ancestor; siblings read via props, update via a callback passed down |
| State lifted, but only passed one direction | Parent holds state, passes it down to sibling A, but sibling B (which should also reflect it) isn't wired to receive it at all | Pass the same state (and, if needed, the same updater) down to every sibling that needs to read or write it |
| Deeply drilled prop, mutated at a leaf, but an intermediate component memoized on stale props | A `memo`'d component in the drilling path doesn't re-render because its own props (not the drilled value itself, but a sibling prop) look unchanged, silently stopping propagation | Ensure `memo` comparisons account for every prop that must propagate, or restructure so the value doesn't have to pass through a memoized intermediate at all |
| Two independent event handlers each computing/updating "the same" derived value separately | No shared source of truth; each call site can drift out of sync with the other | One source of truth for the value; every consumer derives from it, no independent copies |

## The Scenario

"There's a page with a filter sidebar and a results table as siblings under one parent. Selecting a filter option is supposed to immediately update the results table. Most of the time it works — but sometimes, especially after switching between two different filter categories quickly, the table shows results for a filter that's no longer selected in the sidebar. They're visibly out of sync. Find out why and fix it."

## Clarifying Questions

- **Where does the "currently selected filter" state actually live — in the sidebar component itself, in the results table component itself, in both (independently), or lifted to their common parent?** This is the single most important thing to establish first — the symptom described (sidebar and table visibly disagreeing about what's selected) is the direct signature of state that's *supposed* to be one shared value but is actually stored as two independent copies that can each be updated without the other necessarily following.
- **When a filter is selected, what exactly happens — does the sidebar call a callback passed down from the parent (which updates the one shared state and re-renders both children with the new value), or does the sidebar update its own local state and separately call something to notify the table (e.g., firing a custom event, or calling an imperative method via a ref) rather than going through a shared prop?** An event-based or imperative "notify the sibling" mechanism, instead of a shared-state-via-parent model, is a common alternative root cause — it can appear to work for a single change but is prone to exactly the kind of missed/out-of-order update described when changes happen quickly.
- **Is any component in the path between the shared state and the results table wrapped in `React.memo`, and if so, what does its prop comparison look like?** A `memo`'d intermediate component that doesn't correctly include the filter-related prop in whatever makes it decide to re-render is a subtle but real way for a *correctly* lifted, *correctly* passed piece of state to still fail to visibly propagate to the sibling that needs it.
- **Does "switching between two filter categories quickly" suggest anything async is involved** — for instance, does selecting a filter trigger a fetch for updated results, where the *fetch* itself might resolve out of order (a version of the autocomplete race condition, layered on top of a UI-sync bug)? Worth ruling in or out explicitly, since the fix differs substantially depending on whether this is a pure state-architecture bug or has an async race component mixed in.
- **Is the app using any global/shared state mechanism (Context, Redux, Zustand) for this, or is it plain component state and props?** Changes the specific place to look, though the conceptual bug (two things that should share one source of truth instead each holding an independently-updatable copy) is the same regardless of mechanism.

## Approach & Trade-offs

**Prop drilling itself is not the bug — it's a legitimate (if sometimes cumbersome) way to share data down a tree. Duplicated, independently-updatable state is the bug.** It's worth being precise about this distinction in an interview: "prop drilling" typically refers to passing a value through several layers of components that don't themselves use it, purely to get it to a descendant that does — that's an ergonomics complaint (verbose, brittle to refactor), not a correctness bug on its own. The actual correctness bug in this scenario is that the filter's "currently selected" value isn't a single, shared piece of state being read by both the sidebar and the table — it's being tracked independently in more than one place, which structurally allows them to disagree, and *that's* what produces the observed desync, especially under rapid changes where timing can determine which of the two independent copies "wins" at any given moment.

**The general fix: identify the nearest common ancestor of every component that needs to read or write the shared value, and make that ancestor the single source of truth.** Both the sidebar (which sets the filter) and the results table (which reads the filter to know what to display) need access to the same value — so that value should live in state owned by their common parent, passed down to the sidebar as `(selectedFilter, onSelectFilter)` and to the table as `selectedFilter` (read-only, from its perspective). This is "lifting state up," and it's the standard resolution whenever two or more siblings need to stay in sync about something — there is no way for two *independent* pieces of state to guarantee they stay synchronized without an explicit synchronization mechanism between them (which is strictly more complex, and more failure-prone, than simply not duplicating the state in the first place).

**Why an event-based "notify the sibling" pattern (instead of shared lifted state) is fragile specifically under rapid changes.** If the sidebar manages its own local "selected filter" state and, on change, dispatches a custom event or calls an imperative ref method to tell the table "hey, update to X" — rather than the table deriving its displayed filter from the same shared prop the sidebar's selection is also derived from — there are now two independent representations of "what's selected" (the sidebar's own state, and whatever the table decided to do in response to the last notification it received) that are kept in sync only by the discipline of every relevant change firing a notification, and every notification being received and applied in the correct order. Rapid changes (the "switching quickly" detail in the prompt) are exactly the condition under which an ordering assumption like this is most likely to break — if two change-notifications fire close together and are handled asynchronously or out of strict order (React batching, or an async gap introduced by whatever the notification mechanism is), the table can end up reflecting an earlier notification than the sidebar's current actual state, producing precisely the described desync. Lifting the state removes this entire class of risk, because there's no "notification" step at all — both components render from the exact same state value on every render, so they're incapable of disagreeing about what that value currently is.

**A `memo`'d intermediate component silently blocking propagation is a subtler variant worth naming even if it's not the actual bug here.** If the shared state is correctly lifted and passed down, but happens to pass through an intermediate component wrapped in `React.memo` with a custom (or default shallow) comparison that doesn't account for the filter prop changing — because, say, the filter value is nested inside a larger object prop and the comparison only checks top-level reference equality of a *different* field — that intermediate component can bail out of re-rendering even though the value it's supposed to forward down has genuinely changed, effectively "swallowing" the update partway down the tree. This produces a symptom that looks identical from the outside (sidebar and table disagree) but has a completely different root cause and fix (fixing or removing the memo comparison, not touching where state lives) — which is why confirming exactly *where* the state lives and *how* it propagates, rather than assuming based on the visible symptom alone, matters.

**When prop drilling's ergonomics (not its correctness) becomes the actual complaint worth addressing.** If the common ancestor for a shared value is many layers above both consumers, and several intermediate components need to forward the prop through without using it themselves, that's a legitimate case for reaching for Context (to avoid threading the prop through every intermediate layer) — but it's worth being clear that Context, in this role, solves an *ergonomics* problem (not manually threading props through uninterested intermediates), not a *correctness* problem — the correctness fix is "one shared source of truth," which is equally achievable via drilled props or via Context; Context just changes how that one source of truth's value physically gets from the state owner to the consumer.

## Solution

Reproducing the bug — sidebar and table each hold their own copy of "selected filter":

```tsx
function FilterSidebar({ onSelect }: { onSelect: (filter: string) => void }) {
  const [selected, setSelected] = useState('all'); // BUG: local copy #1

  const handleClick = (filter: string) => {
    setSelected(filter);
    onSelect(filter); // notifies the parent/table separately — two things to keep in sync now
  };

  return (
    <ul>
      {['all', 'active', 'archived'].map(f => (
        <li key={f} onClick={() => handleClick(f)} className={f === selected ? 'active' : ''}>{f}</li>
      ))}
    </ul>
  );
}

function ResultsTable({ selectedFilter }: { selectedFilter: string }) {
  const [appliedFilter, setAppliedFilter] = useState('all'); // BUG: local copy #2

  useEffect(() => {
    setAppliedFilter(selectedFilter); // "catches up" to whatever the parent last told it — asynchronously, via an effect
  }, [selectedFilter]);

  const rows = useMemo(() => getRows(appliedFilter), [appliedFilter]);
  return <table>{/* render rows */}</table>;
}

function FilterPage() {
  const [filter, setFilter] = useState('all'); // a THIRD copy, in the parent
  return (
    <>
      <FilterSidebar onSelect={setFilter} />
      <ResultsTable selectedFilter={filter} />
    </>
  );
}
```

Three independent copies of conceptually one value (`selected` in the sidebar, `filter` in the parent, `appliedFilter` in the table, synced to `filter` only through an effect that runs *after* render, one render behind). Rapid clicks between filter categories can easily produce a sequence where the sidebar's own `selected` has already moved to "archived," the parent's `filter` has updated to "archived," but the table's `appliedFilter` is still catching up from an earlier effect run reflecting "active" — a window during which the sidebar visibly shows one thing and the table visibly shows another.

Fix — one source of truth, lifted to the shared parent, both children derive directly from it with no local copies and no effect-based catch-up:

```tsx
function FilterSidebar({ selected, onSelect }: { selected: string; onSelect: (filter: string) => void }) {
  return (
    <ul>
      {['all', 'active', 'archived'].map(f => (
        <li key={f} onClick={() => onSelect(f)} className={f === selected ? 'active' : ''}>{f}</li>
      ))}
    </ul>
  );
}

function ResultsTable({ selectedFilter }: { selectedFilter: string }) {
  const rows = useMemo(() => getRows(selectedFilter), [selectedFilter]); // derived directly, same render, no lag
  return <table>{/* render rows */}</table>;
}

function FilterPage() {
  const [filter, setFilter] = useState('all'); // the ONLY copy
  return (
    <>
      <FilterSidebar selected={filter} onSelect={setFilter} />
      <ResultsTable selectedFilter={filter} />
    </>
  );
}
```

Clicking a filter calls `setFilter` once, in the one place the value lives — both `FilterSidebar` and `ResultsTable` re-render in the same update, reading the exact same `filter` value, with no intermediate local state and no effect-based synchronization step that could lag behind or process updates out of order.

> **Check yourself:** In the fixed version, if `getRows(selectedFilter)` were an expensive synchronous computation (not shown here as async), would there be any scenario, even under very rapid clicking, where the sidebar and table could visibly disagree about the selected filter? Why does removing the two local copies of state eliminate that possibility structurally, rather than just making it less likely?

## Root Cause

The root cause is representing what is conceptually *one* piece of shared state as multiple, independently-updatable copies, synchronized (if at all) through an indirect mechanism — an effect, an event, an imperative call — rather than by simply having every consumer read from the same single source. Any synchronization mechanism between independent copies introduces a window (and, under rapid or out-of-order updates, a real possibility) for the copies to disagree; removing the duplication removes the window entirely.

## Gotchas

**"It works fine for a single, slow filter change" masking the bug during casual testing.** A single click, with time to fully settle before the next one, gives any effect-based catch-up mechanism time to run and converge before it's visibly relevant — the bug only reveals itself under rapid sequential changes, which is exactly the condition the scenario names and exactly the condition casual manual testing is least likely to naturally reproduce.

**Treating "lift the state up" as the answer without actually removing the now-redundant local copies.** Adding a lifted `filter` state to the parent while *leaving* the sidebar's local `selected` state and the table's local `appliedFilter` state in place (perhaps now also being kept vaguely in sync with the new lifted value via yet another effect) doesn't fix the bug — it just adds a third copy to the two that already existed, and the same order-of-updates fragility remains. The fix requires deleting the redundant local state entirely, not layering a "more correct" copy on top.

**Confusing "prop drilling is inconvenient" with "prop drilling caused this bug."** As covered in Approach & Trade-offs, drilling itself isn't what broke synchronization here — reaching immediately for Context as "the fix" without first correcting the actual duplicated-state architecture would preserve the bug while changing how the (still-duplicated) values get passed around.

**Introducing a `useEffect`-based "catch-up" sync specifically to bridge two pieces of state that shouldn't have been separate in the first place**, rather than treating the presence of such a sync effect as a signal to eliminate one of the two states. Effects that exist purely to copy one piece of state into another are almost always a sign that the copy shouldn't exist as separate state at all (a close relative of the pattern discussed in the infinite-render-loop scenario).

## Follow-up Questions

**Q (High): Explain precisely why an effect-based "sync state B to reflect state A" pattern is inherently one render behind, and why that lag is exactly what produces the visible desync described in this scenario.**

Answer: `useEffect` runs *after* React has committed a render to the DOM — it's explicitly a post-render side effect, not something that happens synchronously as part of computing what to render. So if `ResultsTable` derives its displayed rows from `appliedFilter` (its own local state) and an effect updates `appliedFilter` in response to a `selectedFilter` prop change, the *first* render after `selectedFilter` changes still shows the *old* `appliedFilter` value (since the effect hasn't run yet — effects run after that render commits), and only the *subsequent* render (triggered by the effect's own `setAppliedFilter` call) reflects the new value. Under a single, isolated filter change with time to settle, this one-render lag is invisible to a human eye. Under rapid changes, this lag means the table's displayed state, at any given moment, may correspond to an *earlier* selected filter than what the sidebar currently shows — and if changes happen fast enough relative to React's render/commit/effect cycle, several "generations" of catch-up can be in flight, in principle even applying out of order depending on exactly how the effect handles overlapping updates (with no explicit ordering/cancellation logic here, there isn't a guarantee later effects always "win" over earlier ones in every edge case).

The trap: describing the fix as "the effect just needs to run faster" or "add a loading state" — the issue isn't effect speed, it's that an effect-based sync is *structurally* always at least one render behind the value it's syncing from, which is a category of problem no amount of speed fixes; only removing the indirection (deriving directly during render, or lifting the single source of truth) eliminates it.

---

**Q (High): The fix lifts state to the common parent. What's the general rule for *where*, in a component tree, shared state should live, and how do you find that location for an arbitrary pair of components that need to stay in sync?**

Answer: The general rule is: state should live at the lowest common ancestor of every component that needs to either read or write it — "lowest" meaning as close to the components that need it as possible, while still being high enough in the tree to be a genuine ancestor of *all* of them. Concretely, this is found by tracing each relevant component's ancestor chain up to the root and finding the first point where those chains converge; state placed any higher than that point is unnecessarily distant (forcing prop drilling through intermediates that don't need it), and state placed any lower (in only one of the sibling branches) is what recreates the exact duplication bug this scenario demonstrates, since a state defined inside one sibling's subtree is, by construction, invisible to the other sibling without some indirect synchronization mechanism.

The trap: describing the rule only as "put it in the parent" without the general "lowest common ancestor" framing — a candidate should be able to apply this to an arbitrary, deeper tree (state needed by a component three levels deep in one branch and two levels deep in a sibling branch), not just the two-sibling case given in this specific scenario.

---

**Q (High): Would this bug be possible if the filter state were instead managed through a global state library (Redux, Zustand) rather than component state and props? Why or why not?**

Answer: The specific *mechanism* that caused this bug (two or more independently-`useState`-managed local copies, synchronized via a lagging effect) becomes structurally impossible in the same form, because both `FilterSidebar` and `ResultsTable` would subscribe to and derive from the *same* single store value directly (via a selector), with no local copies of the filter value existing anywhere to fall out of sync with each other — an update to the store is, from each subscriber's perspective, a single source of truth changing, not a notification that needs to be independently caught up on. That said, a *different*, structurally analogous bug is still possible if the codebase reintroduces duplication anyway — e.g., a component that copies a store value into local `useState` "for convenience" (to allow local editing before committing back to the store) and doesn't correctly re-sync that local copy if the store value changes from elsewhere — which is the same underlying mistake (duplicated, independently-updatable copies of one conceptual value) wearing a different outfit, just moved one layer further from where it's most commonly introduced. The library removes the *most common* way to accidentally introduce this bug, but doesn't make the underlying discipline (don't duplicate shared state) unnecessary.

The trap: claiming a state library "solves" this category of bug categorically — it removes the most common trigger (independent component-local `useState` copies of what should be shared state) but doesn't prevent someone from reintroducing the same duplication pattern manually on top of the library, which is worth naming as the more complete, structurally-aware answer.

---

**Q (Medium): If the sidebar and table are in genuinely different, distant parts of a large component tree — not simple siblings — and lifting state all the way to their true common ancestor would mean drilling it through a dozen intermediate layers, what would you actually do?**

Answer: This is the legitimate case for Context (or a lightweight external store) — not because prop drilling would be "incorrect," but because it becomes an ergonomics and maintainability burden (a dozen intermediate components that don't use the value themselves, all needing to accept and forward it as a prop, and all needing to be touched again if the shape of that prop ever changes). A `FilterContext.Provider` placed at the actual common ancestor, with `FilterSidebar` and `ResultsTable` each calling `useContext(FilterContext)` directly, delivers the exact same single-source-of-truth guarantee (no duplicated copies, since both read from the same Provider value) while removing the need for every intermediate component in between to know about or forward the filter value at all. The correctness property (one shared value, not fixed by React Context specifically) is identical to the props-based lifted-state fix — Context is chosen here purely to solve the *distance* problem, which is real, but distinct from the *duplication* problem this whole scenario is actually about.

The trap: treating "the components are far apart" as license to skip past the actual correctness principle (single source of truth) — Context still needs to be structured as one Provider owning one value, read directly by both consumers; sprinkling multiple `useState` calls across the tree and trying to reconcile them via Context updates would reproduce the exact same bug this scenario is built around, just spread across a wider tree.

---

**Q (Medium): How would you write an automated test that specifically catches this class of bug — sidebar and table disagreeing under rapid sequential changes — rather than only under a single, slow change?**

Answer: A React Testing Library test that renders the full parent component (not the sidebar or table in isolation, since the bug is about their *interaction*), fires several filter-selection events in quick succession without awaiting/settling between them (e.g., `fireEvent.click` on two or three different filter options back-to-back, synchronously, before any `act()`/`waitFor` flush occurs), and then asserts that the sidebar's visually-active filter and the table's currently-rendered rows agree on the same filter value, would directly exercise the failure condition — since it specifically avoids giving any intermediate effect-based catch-up time to settle between changes, which is precisely the condition under which the original bug (built on an effect-based sync) would surface a mismatch, and which the fixed (single-source-of-truth, same-render-derivation) version would pass regardless of how rapidly the events fire, since there's no catch-up step to race against in the corrected version at all.

The trap: writing a test that only clicks one filter, awaits full settlement, then asserts — that kind of test would pass against both the buggy and fixed versions equally, since it doesn't exercise the specific rapid-sequential-change condition the bug depends on, giving false confidence that the component is correctly tested when the actual regression path remains uncovered.

---

## Self-Assessment

- [ ] Can distinguish "prop drilling" (an ergonomics concern) from "duplicated state" (the actual correctness bug) in this scenario
- [ ] Can explain precisely why an effect-based state-sync pattern is structurally one render behind the value it syncs from
- [ ] Can apply the "lowest common ancestor" rule to find where shared state should live, for an arbitrary tree shape
- [ ] Can explain why a global state library removes the most common trigger for this bug without eliminating the underlying discipline needed to avoid it
- [ ] Can justify choosing Context specifically to solve tree-distance, while still keeping the same single-source-of-truth correctness principle
- [ ] Can design a test that exercises rapid sequential updates specifically, not just a single settled change

---
*Next: Error Boundary Not Catching an Error — Why — a scenario about the limits of a React mechanism, specifically the categories of errors Error Boundaries are structurally incapable of catching, and why.*
