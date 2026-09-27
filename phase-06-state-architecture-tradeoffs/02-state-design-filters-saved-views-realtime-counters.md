# State Design: Filters + Saved Views + Real-time Counters

## Quick Reference

| Piece | Category | Storage | Key Design Choice |
|---|---|---|---|
| Active filters | URL state | Search params | Single source of truth for "current view" |
| Saved views | Server state (per-user, persisted) | Query cache, backed by an API resource | A saved view is a *named snapshot* of URL state, not a live-synced copy |
| "Currently on saved view X" indicator | Derived | Computed by comparing active URL state to each saved view's snapshot | Never stored as its own flag — it'd desync the moment either side changes |
| Real-time counters (e.g., "12 new results") | Ephemeral client state, sourced from a subscription | Local state / a small store, updated by websocket/SSE events | Deliberately decoupled from the paginated list itself — see Approach |

## The Scenario

"Extend the orders table from before: users can now save their current filter/sort configuration as a named 'view' (e.g., 'Overdue — East Region') and switch between saved views from a dropdown. Separately, product wants a live counter badge — '12 new orders match your current filters' — that updates in real time via a websocket feed as new orders come in, without forcing a full table refresh. Design the state for both features."

## Clarifying Questions

- **When a user saves a view, is it a snapshot frozen at save time, or does it stay "live" — e.g., if I later edit the underlying saved view's date range, does that change what URL state loads when someone selects it?** This determines whether a saved view is genuinely just persisted URL-state data (a snapshot, immutable until explicitly edited) or something with its own separate editing/versioning lifecycle — I'd assume snapshot-until-edited unless told otherwise, since that matches how most "saved search" features behave.
- **Are saved views personal (per-user) or shared across a team?** This changes where they're stored (a per-user preference resource vs. a shared, ACL'd resource) and whether concurrent edit conflicts are a real concern — shared saved views that multiple people can rename/edit introduce a much bigger scope (who can edit, conflict handling) than personal ones.
- **For the real-time counter — does "new orders match your current filters" mean the server is doing filter evaluation per-message and only notifying me about genuinely-matching new orders, or is the client receiving all new-order events and filtering client-side?** This is the single biggest design fork: server-side filtered push (I get a `count` or a stream of only-matching new records) is far simpler and more scalable than the client receiving every new-order event across the whole system and evaluating potentially-complex filter logic against each one — I'd push hard for the former.
- **When the counter says "12 new," and the user clicks it, what's the expected behavior — does it insert those 12 rows into the currently-viewed page in place, prepend them, or does it just trigger a full refetch of the current filtered/sorted/paginated query?** This affects whether I need to reconcile new real-time data against an existing paginated result set (non-trivial, especially with sorting) or can treat "click to see new results" as "just refetch," which is dramatically simpler and usually what's actually wanted.
- **Should the real-time counter keep counting while the user is actively viewing/interacting with the table, or should it pause while they're mid-action (e.g., filling out a bulk-edit form) to avoid data shifting under them?** A counter that silently changes underlying data while someone's mid-workflow is a real UX risk (selected-row bulk actions targeting rows that have since changed) — I'd want an explicit answer rather than assuming.

## Approach & Trade-offs

**A saved view is not a new state category — it's server-persisted data whose *payload* is a snapshot of URL state from Scenario 1.** The temptation is to build a parallel, bespoke "saved view" state machine; the better framing is: a saved view is just a named record `{ id, name, filters, sort }` stored via the same server-state layer as everything else (React Query mutation to create/update/delete, query to list). The only genuinely new logic is (a) applying a saved view = writing its stored filter/sort payload into the URL (reusing the exact `setFilters`/`setSort` functions from Scenario 1, not a separate code path), and (b) determining whether the *current* URL state matches any saved view, for UI purposes (highlighting the active view in the dropdown).

**That "is a saved view currently active" indicator must be derived, never stored as a flag, because it has two independent sources of truth that can each change at any time** — the active URL state (user tweaks a filter manually) and the saved view's own definition (someone edits/deletes it). If I stored `activeViewId` as its own piece of state, set only at the moment a view is selected, it would go stale the instant the user changes any filter afterward — the dropdown would still show "Overdue — East Region" highlighted as active even though the user has since changed the date range and it's no longer actually that view. The correct approach is a cheap equality check on every render (or memoized): compare current `{filters, sort}` against each saved view's stored payload, and whichever matches (if any) is "active" — self-correcting by construction, with no separate state to desync.

**The real-time counter is the more interesting design problem, and the key decision is to keep it fully decoupled from the paginated table's query, not trying to merge live data into an existing paginated/sorted result set.** Reconciling incoming real-time events directly into an already-fetched, sorted, paginated list is genuinely hard to get right — where does a new matching row get inserted given the current sort order and page boundary, does it push another row off the visible page, does it change the total count shown for pagination — and almost never worth solving for what's actually being asked (a *notification* that new data exists, not a live-merging feed). Instead: a lightweight subscription (websocket/SSE) maintains its own small piece of state — just a count (or a small buffer of new-item summaries) — entirely separate from the table's query cache, and the "12 new orders" badge is driven by that separate state. Clicking it doesn't try to splice data in; it triggers exactly the same refetch mechanism a manual filter change would (invalidate the `['orders', filters, sort, page]` query), which is a solved problem already, then resets the real-time counter to zero. This trades "instant, seamless live-merge" for "simple, correct, and consistent with how every other data change on this page already works" — the right trade for almost any real product requirement phrased as "notify me new stuff exists," versus something explicitly demanding live-merging (a chat feed, discussed in scenario coverage elsewhere), which is a different, harder problem.

**The websocket subscription's lifecycle needs to track the *current* filters, since "matches your current filters" implies the server needs to know what to filter against** — meaning the subscription itself needs to be re-established (or its server-side filter criteria updated) whenever the URL-state filters change, not just set up once on mount. This is a real piece of design, not an afterthought: an effect that tears down and re-opens the subscription (or sends an updated "subscribe with these criteria" message over an existing connection) whenever `filters` changes, mirroring how the query's `queryKey` already changes to trigger a refetch.

## Solution — the state design

**1. Saved views — server state, reusing the existing filter/sort shape:**

```tsx
type SavedView = {
  id: string;
  name: string;
  filters: OrderFilters;
  sort: SortState;
};

function useSavedViews() {
  return useQuery({ queryKey: ['savedViews'], queryFn: fetchSavedViews });
}

function useSaveView() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (view: Omit<SavedView, 'id'>) => createSavedView(view),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['savedViews'] }),
  });
}
```

**2. Applying a saved view — reuses Scenario 1's URL-state setters directly, no new code path:**

```tsx
function applySavedView(view: SavedView, setFilters: (f: OrderFilters) => void, setSort: (s: SortState) => void) {
  setFilters(view.filters);
  setSort(view.sort);
}
```

**3. "Is this view currently active" — derived, computed on every render, not stored:**

```tsx
function useActiveViewId(savedViews: SavedView[], currentFilters: OrderFilters, currentSort: SortState): string | null {
  return useMemo(() => {
    const match = savedViews.find(
      (v) => isEqual(v.filters, currentFilters) && isEqual(v.sort, currentSort)
    );
    return match?.id ?? null;
  }, [savedViews, currentFilters, currentSort]);
}
```

**4. Real-time counter — fully separate subscription-backed state, decoupled from the table query:**

```tsx
function useNewOrdersCounter(filters: OrderFilters) {
  const [newCount, setNewCount] = useState(0);

  useEffect(() => {
    setNewCount(0); // reset when filters change — the old count no longer means anything against a new filter set
    const socket = subscribeToNewOrders(filters, {
      onMatch: () => setNewCount((c) => c + 1),
    });
    return () => socket.close(); // re-subscribes with updated criteria whenever `filters` changes
  }, [filters]);

  return newCount;
}

function NewOrdersBadge({ filters }: { filters: OrderFilters }) {
  const newCount = useNewOrdersCounter(filters);
  const queryClient = useQueryClient();

  if (newCount === 0) return null;

  return (
    <button
      onClick={() => {
        queryClient.invalidateQueries({ queryKey: ['orders'] }); // same refetch path as any filter change
        setNewCount(0); // handled via a ref/callback into the hook in the real version
      }}
    >
      {newCount} new order{newCount > 1 ? 's' : ''} — click to refresh
    </button>
  );
}
```

> **Check yourself:** Why does resetting `newCount` to 0 inside the effect's setup (keyed on `filters`) matter, specifically — what would silently go wrong if that reset line were removed?

## Data Model

**Saved view (server resource):**
```
SavedView {
  id: string
  ownerId: string       // or teamId, if shared views are in scope
  name: string
  filters: { status: string[], dateFrom?: string, dateTo?: string, search: string }
  sort: { field: string, direction: 'asc' | 'desc' }
  createdAt: string
  updatedAt: string
}
```
Deliberately does *not* store `page` — reapplying a saved view resets to page 1, since "page 4 of whatever the result set happens to be today" isn't a meaningful part of a saved *view's* definition, even though page is part of the URL state generally.

## Gotchas

**Storing `activeViewId` as a piece of state set at selection time, rather than deriving it.** The moment the user manually tweaks any filter after selecting a view, the dropdown keeps showing the old view as "active" — a visible, confusing bug that's entirely avoided by computing the match on every render instead.

**Building the real-time feature by having the client subscribe to *all* new-order events and filter them client-side against potentially complex filter logic.** This duplicates filter-evaluation logic between server (for the actual paginated query) and client (for the real-time stream), risking the two falling out of sync, and doesn't scale if order volume is high — pushing filter evaluation to the server so the client only receives already-matching events (or a count) is both simpler and more correct.

**Trying to splice real-time events directly into the existing paginated/sorted query result.** Where a new item lands relative to current sort order, whether it displaces something off the current page, and how it affects a displayed total count are all genuinely tricky to get right — for a "12 new — click to refresh" requirement, this complexity buys nothing; a decoupled counter plus a refetch-on-click is simpler and matches what was actually asked for.

**Forgetting to reset or re-establish the real-time subscription's filter criteria when the user changes filters.** A stale subscription still using the old filter criteria produces a counter that's telling the user about the wrong thing entirely — "12 new" that doesn't actually correspond to what's currently on screen.

**Not handling saved-view deletion or edit gracefully when it's currently the "active" view.** If a view is deleted while active, does the URL state (which is independent, already-applied filters) just keep working with no indication the view no longer exists? Usually yes for filters already applied — but the dropdown needs to handle "the previously active view is now gone" without crashing on a stale reference.

## Follow-up Questions

**Q (High): Why treat the real-time counter as fully decoupled from the paginated table query instead of trying to keep the table "live"?**

Answer: Because "live" in the sense of seamlessly merging server-pushed events into an already-paginated, sorted result set is a materially harder problem than what's actually being asked — a notification badge — and solving the harder problem buys nothing here: where a new row should visually land given the current sort, whether it should push a row off the bottom of the current page, how it affects a "total: 240 results" count shown for pagination, all need real design decisions with no obviously-correct answer, versus a decoupled counter-plus-refetch which reuses machinery (the query invalidation path) that already exists and is already correct. If the actual product requirement were instead "this should feel like a live chat/activity feed, not a paginated table," that's a signal the feature itself should be modeled differently (an appended/prepended live list, not a paginated grid) — not that the current design is wrong for what was asked.

The trap: over-engineering a live-merge solution because it sounds more impressive or "more real-time," when the simpler decoupled design better matches the stated requirement and has fewer edge cases.

---

**Q (High): Two saved views happen to have byte-for-byte identical filter/sort payloads (e.g., a user accidentally saved the same view twice under different names). What does your `useActiveViewId` derivation return, and is that a problem?**

Answer: As written, `.find()` returns the first match in array order, so it'd non-deterministically (well, deterministically by array order, but arbitrarily from a user's perspective) highlight whichever of the two duplicate-payload views happens to come first — showing one specific view as "active" while the other, equally-matching one isn't highlighted, which could look like a bug to a user who expects both to be highlighted or is confused about which one is "really" active. Practically, I'd treat this as a low-priority cosmetic edge case rather than something requiring an architecture change — it doesn't represent incorrect underlying state (the URL state itself is still completely correct), just an ambiguous *display* of which name to show for it, and I'd either leave it as "shows the first match" (fine for almost every real case) or, if it mattered more, change the affordance to "N views match this configuration" rather than picking one arbitrarily.

The trap: treating this as a sign the derived-state approach is flawed and reaching for a stored flag "to make it deterministic" — the ambiguity is inherent to the data (two views really are identical), not a flaw in deriving rather than storing; storing wouldn't resolve the ambiguity, just hide it behind whichever view happened to be clicked last.

---

**Q (High): A saved view is shared across a team, and another teammate renames it while the current user has it selected/active. What should happen on the current user's screen?**

Answer: Nothing needs to happen to the *applied filters* — the URL state is already a plain, independent copy of the filter/sort values, not a live reference to the saved view record, so a rename doesn't change what's currently displayed or filtered. What *should* update, on the next fetch/refetch of the saved-views list (whether via polling, a websocket event, or simply the next time that query naturally refetches), is the *label* shown in the dropdown for that view — and the "is this view active" derivation still correctly matches by filter/sort payload, not by name, so it keeps highlighting correctly even through the rename. This is actually a good demonstration of why treating saved views as "just a persisted payload plus metadata" rather than a special stateful entity pays off: a rename is a boring metadata update to an already-independent copy of data, not a synchronization problem.

The trap: assuming a rename requires actively pushing an update to every client currently "using" that view — there's no live coupling to break, because applying a view already copied its payload into independent URL state at selection time.

---

**Q (Medium): How would you implement "undo my last filter change" given filters live in the URL — is browser back/forward sufficient, or does it need dedicated undo logic?**

Answer: Browser back/forward is largely sufficient and is the reason URL state is attractive here — since every `setSearchParams` call (assuming it's not using `replace: true`) pushes a new history entry, the back button naturally steps through the filter change history for free, with zero custom undo code. The nuance to flag: if the implementation used `replace` for search-param updates (a common choice to avoid flooding history with every single keystroke in a search box), back/forward would skip over intermediate states — so the actual design decision is which URL updates should push a new history entry (a filter/sort/view change — a meaningful navigation) versus replace the current one (live-typing into a search box before it's "settled," where every keystroke as a separate history entry would make back/forward nearly unusable). Getting that split right gives usable undo via native browser navigation without writing any bespoke undo stack — a good example of URL state's back/forward "for free" benefit, distinct from the more general undo/redo architecture problem covered later in this phase.

The trap: assuming URL state automatically means back/forward undo works perfectly with no further thought — it depends on correctly choosing push vs. replace per update, and getting that wrong (e.g., replacing on every keystroke of a *filter dropdown selection*, not just free-text typing) silently breaks the undo experience.

---

**Q (Medium): The real-time subscription needs the *server* to evaluate whether a new order matches the current filters. What has to happen on re-subscribe every time filters change, and what's the failure mode if that's implemented sloppily (e.g., debounced too aggressively, or not re-subscribing at all)?**

Answer: On every filter change, the client needs to either open a new subscription scoped to the new filter criteria (simplest, if the transport supports cheap reconnects) or send an "update my subscription criteria" message over a persistent connection (more efficient for a connection-heavy transport, avoiding reconnect overhead on every filter tweak) — either way, the server-side filter-matching logic needs the current criteria, not stale criteria from before the change. The failure mode of not re-subscribing at all: the counter keeps counting matches against the *old* filters indefinitely, so "3 new orders" shown to the user doesn't correspond to what's actually visible under their current filters — actively misleading, worse than showing no counter at all. Over-aggressive debouncing of the re-subscribe (e.g., waiting 2 seconds after the last filter change before updating the subscription) is a reasonable trade-off for avoiding subscription churn while a user is actively adjusting multiple filters in quick succession, as long as the counter is visibly reset/hidden during that debounce window rather than showing a stale count as if it were current.

The trap: treating the subscription as "set up once on mount" — filters are dynamic client-owned state, and the subscription's server-side criteria has to track it exactly the same way the query's `queryKey` does.

---

**Q (Low): Would a saved view ever need optimistic UI when saving, given it's just a small metadata record?**

Answer: It's a reasonable, low-risk candidate for optimistic UI — add the new view to the visible list immediately on save (before the server confirms), since the failure mode of a save request is rare and low-stakes (worst case, a brief flash of an "unsaved" view that then errors and is removed with a toast), unlike a bulk order-status mutation where an incorrect optimistic assumption could mislead someone into acting on wrong data. I'd still want a rollback path (remove it from the local cache and show an error toast) if the mutation fails, using the same optimistic-update-with-rollback pattern covered in the next scenario, just applied to a much lower-stakes piece of data.

The trap: either skipping optimistic UI reflexively ("it's just a save, no need") when it's cheap and genuinely improves perceived responsiveness, or over-engineering elaborate conflict handling for what's a low-contention, low-stakes personal-preference-style resource.

---

## Self-Assessment

- [ ] Can explain why a saved view is server state whose payload happens to be a URL-state snapshot, not a new state category
- [ ] Can justify deriving "is this view active" instead of storing it, and name the specific bug that storing it causes
- [ ] Can explain why the real-time counter is deliberately decoupled from the paginated query, and what problem that avoids solving
- [ ] Can describe what has to happen to the real-time subscription when filters change, and the failure mode if it doesn't
- [ ] Can reason about push vs. replace history semantics for URL state changes in the context of back-button-driven undo

---
*Next: Optimistic Update With Rollback — moves from "where does state live" into "what happens to that state during the gap between a user action and server confirmation," using the bulk-action and saved-view mutations introduced here as the motivating cases.*
