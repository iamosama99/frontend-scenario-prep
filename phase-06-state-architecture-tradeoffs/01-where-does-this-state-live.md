# Where Does This State Live? (Server/Client/URL/Form Sort)

## Quick Reference

| State Category | Lives In | Tell-tale Sign |
|---|---|---|
| Server state (mirrors a remote resource) | Query cache (React Query/SWR/RTK Query) or a thin fetch+cache layer | It's "owned" by the backend; the client has a copy that can go stale |
| URL state (drives what's on screen, shareable/bookmarkable) | Router (search params, path segments) | Refreshing the page or sharing the link should reproduce the same view |
| Session/UI state (ephemeral, this-tab-only) | Local component state / a lightweight client store | Lost on refresh with no complaints from anyone |
| Form state (in-progress input, not yet committed) | Local state or a form library (React Hook Form, Formik) | Uncommitted, has its own validation/dirty/touched lifecycle distinct from the "real" data |
| Derived state | Not stored at all — computed at render/selector time | If you can compute it from other state, storing it separately is a bug waiting to desync |

## The Scenario

"Here's a product page: a data table of orders, with filters (status, date range, search text), sortable columns, pagination, a selected-rows checkbox set for bulk actions, and a slide-over panel that opens to show order details when you click a row. Before you write any code — where does each piece of state for this page live, and why? Walk me through your reasoning for each piece."

## Clarifying Questions

- **Should filter/sort/pagination state survive a page refresh, and should it be shareable via link (e.g., a support agent pasting a filtered URL to a teammate)?** This is the single question that determines whether filters belong in the URL or in local component state — if "someone should be able to bookmark or share this exact filtered view," the answer is URL, full stop; if it's meant to reset every visit, local state is simpler and there's no reason to add URL complexity.
- **Is the order data itself something other parts of the app also need (e.g., a global order count badge, a related widget on another page), or is it scoped entirely to this table?** If nothing else needs it, a page-local fetch is fine; if it's shared, it belongs in a query cache keyed consistently so multiple consumers reuse the same request instead of each independently re-fetching.
- **Does "selected rows" need to survive navigating away and back (e.g., select on page 1, paginate to page 2, come back and selections are still checked), or is it fine for selection to reset per session/page load?** This affects whether selection is derived from row IDs (survivable across pagination, requires more careful state shape) versus row indices (simpler, but breaks the moment the underlying data changes or reorders) — the answer changes the shape of the state, not just where it lives.
- **Is the slide-over panel's open/closed state and which order it's showing something that should be a URL-addressable route (e.g., `/orders?panel=order-123`) or purely transient UI state?** If a support workflow involves "here's a direct link to this open order's detail panel," that's URL state; if it's just a UI convenience that's fine to lose on refresh, it's local.
- **How is data fetched — is there an existing data-fetching/caching layer (React Query, SWR, Redux with RTK Query) already in the codebase, or is this greenfield?** The answer changes "how" server state is stored, though not the underlying classification logic — I'd rather reuse whatever pattern is already established than introduce a second competing approach.

## Approach & Trade-offs

**The core discipline here is classifying every piece of state into one of four buckets *before* reaching for any specific tool** — server state, URL state, session/UI state, and form state — because reaching for one general-purpose store (shove everything into Redux, or worse, one giant `useState` object) is the single most common mid-level mistake on this exact scenario. Each bucket has a different lifecycle, a different "source of truth," and a different natural home, and conflating them creates real bugs, not just style violations: server data kept in a global client store desyncs from the backend; UI-only state pushed into the URL clutters links and triggers unnecessary re-renders on unrelated components subscribed to the router; filters kept in local state can't be shared or survive a refresh when the product actually needs that.

**Server state (the order rows themselves) is not "state I own," it's a cached copy of something the backend owns** — so it belongs in a dedicated server-cache layer (React Query/SWR) rather than in `useState`/Redux, specifically because that layer already solves the hard parts this scenario will eventually need: request deduplication (filters changing rapidly firing overlapping requests), staleness/revalidation, loading/error states per query key, and cache invalidation after a bulk action mutates the data. Modeling it as "just data I fetched into a `useState`" means re-implementing all of that by hand, badly, under interview time pressure.

**Filters, sort, and pagination are URL state, not component state, because this is a data table a user would reasonably want to bookmark, share, or navigate back to with the same view intact** — the URL is the one piece of client "storage" that survives a refresh, a share, and a back-button press for free, without any extra persistence code. The trade-off to state explicitly: URL state is slightly more code to wire up (reading/writing search params instead of `setState`) and the encoding (dates, arrays of statuses) needs a convention, but the alternative — local state that resets on refresh — actively breaks the "share a filtered view with a teammate" workflow this kind of internal tool almost always needs eventually, even if not asked for on day one.

**Selected rows for bulk actions is session/UI state — it's this-tab, this-session, transient, and nobody expects it to survive a refresh** — I'd keep it as local state (a `Set<orderId>` for O(1) lookup) at the level of the table component, not lifted into global state, because nothing outside this page's tree needs to read or react to it. The one design decision that matters here: keying selection by row *ID*, not row *index*, so that paginating away and back, or the underlying data reordering after a mutation, doesn't silently select the wrong rows.

**The slide-over panel's open state is a judgment call, and I'd default to URL state (a search param like `?order=123`) unless told otherwise** — because "click a row, see its detail, want to share a direct link to that specific order's panel with a teammate" is such a common support/ops workflow for exactly this kind of page that I'd rather build it URL-addressable from the start than retrofit it later; the cost of doing so (reading one more search param, an effect or router listener that opens the panel when it's present) is small relative to the workflow it unlocks.

**Form state, if this page later adds an inline edit form inside the detail panel, is deliberately kept separate from both the server cache and the URL** — it's a third distinct lifecycle (dirty/touched/validation, uncommitted local edits that may be discarded), and mixing it into the query cache (mutating cached server data optimistically before the form is even submitted) or the URL (encoding in-progress edits as search params) would be conflating "what the server says" / "what view am I looking at" with "what am I currently typing," which are different questions with different answers at different times.

## Solution — the classification walkthrough

**1. Server state — order rows, fetched via the current filters/sort/pagination as query parameters:**

```tsx
function useOrders(filters: OrderFilters, sort: SortState, page: number) {
  return useQuery({
    queryKey: ['orders', filters, sort, page], // cache key encodes everything the result depends on
    queryFn: () => fetchOrders({ filters, sort, page }),
    keepPreviousData: true, // avoids a loading flash when only the page number changes
  });
}
```

**2. URL state — filters, sort, pagination, and the open detail panel, all as search params:**

```tsx
function useOrdersUrlState() {
  const [searchParams, setSearchParams] = useSearchParams();

  const filters: OrderFilters = {
    status: searchParams.get('status')?.split(',') ?? [],
    dateFrom: searchParams.get('from') ?? undefined,
    dateTo: searchParams.get('to') ?? undefined,
    search: searchParams.get('q') ?? '',
  };
  const sort: SortState = {
    field: (searchParams.get('sortField') as SortField) ?? 'createdAt',
    direction: (searchParams.get('sortDir') as 'asc' | 'desc') ?? 'desc',
  };
  const page = Number(searchParams.get('page') ?? '1');
  const openOrderId = searchParams.get('order'); // slide-over panel state

  function setFilters(next: Partial<OrderFilters>) {
    setSearchParams((prev) => {
      const merged = { ...filters, ...next };
      prev.set('status', merged.status.join(','));
      // ...set remaining params
      prev.set('page', '1'); // changing filters resets pagination — a real, easy-to-miss bug otherwise
      return prev;
    });
  }

  return { filters, sort, page, openOrderId, setFilters /* , setSort, setPage, openOrder, closeOrder */ };
}
```

**3. Session/UI state — selected rows, keyed by ID, scoped to the table:**

```tsx
function OrdersTable({ orders }: { orders: Order[] }) {
  const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set());

  function toggleRow(id: string) {
    setSelectedIds((prev) => {
      const next = new Set(prev);
      next.has(id) ? next.delete(id) : next.add(id);
      return next;
    });
  }
  // selection survives across re-renders of `orders` (e.g. a background refetch) because it's keyed by id, not index
}
```

**4. Derived state — nothing new is stored for "are all visible rows selected" (the header checkbox's indeterminate state):**

```tsx
const allVisibleSelected = orders.length > 0 && orders.every((o) => selectedIds.has(o.id));
const someVisibleSelected = orders.some((o) => selectedIds.has(o.id));
// computed at render time — storing this separately would require manually keeping it in sync with `selectedIds` and `orders`
```

> **Check yourself:** If a teammate suggested "let's just put everything — filters, selection, the open panel — into one Redux slice for consistency," what specifically would you say breaks, for each of the three?

## Gotchas

**Putting filters in local `useState` because "it's simpler" and only discovering the shareable-link requirement in a later sprint.** Retrofitting URL state onto filters that were built as local state means threading router reads/writes through code that wasn't structured for it, plus fixing every place that assumed synchronous `setState` semantics instead of URL-driven re-renders.

**Selecting rows by array index instead of ID.** The instant the underlying `orders` array reorders (a background refetch after a mutation, a sort change) or a new page of different rows loads into the same index range, index-based selection silently points at the wrong rows — a bug that often isn't caught until a bulk action is performed against the wrong records.

**Storing "is this order currently being edited" as a boolean flag on the cached server-state object itself** (e.g., mutating the React Query cache to add an `isEditing: true` field to an order). This conflates server state with UI/form state — a background refetch or cache invalidation can silently overwrite or lose that flag, since it's not really "server data," it's UI state that happens to be attached to a server-state object.

**Forgetting to reset pagination when filters change.** A user filters to a narrower set of results while sitting on page 4 of the old, larger result set — without resetting `page` to 1 on filter change, they land on an empty or out-of-range page and it looks like the filter returned nothing.

**Encoding complex filter state (arrays, date ranges) into the URL without a clear, tested serialization convention.** `status=pending,shipped` needs a consistent split/join convention handled in exactly one place — ad hoc `JSON.stringify` into a URL param produces ugly, fragile, sometimes-invalid URLs and breaks the "shareable link" goal that was the whole reason to use URL state.

## Follow-up Questions

**Q (High): Why not just put all of this — orders, filters, selection, panel state — into a single Redux (or Zustand) store for a consistent mental model?**

Answer: Because "consistent mental model" is optimizing for the wrong thing here — the four categories have genuinely different lifecycles and correctness requirements, and a single store either has to reinvent the specific machinery each one needs (cache invalidation and staleness semantics for server state, URL-sync for filters, none of that for ephemeral selection) or, more commonly, just skips that machinery, producing bugs: server data goes stale because nobody wired up refetching logic that a query library gives for free; filters don't survive a refresh because nobody built the URL-sync layer a router already provides; selection state gets needlessly persisted or synced somewhere it doesn't need to be. Using the tool that's purpose-built for each category (query cache, router, local state) isn't inconsistency, it's matching the tool to the actual shape and lifecycle of the data — the "consistency" that matters is architectural (always classify state the same way), not literally "one store for everything."

The trap: treating "fewer state management tools" as inherently simpler — it often produces *more* code, not less, once you manually rebuild what a query library or router already solved, and it produces subtler bugs from missing that machinery.

---

**Q (High): The bulk-action bar needs to show "3 orders selected" and also needs to know the *full order objects* for those 3 selections (e.g., to check they're all in a status that supports the bulk action). How would you get from `selectedIds: Set<string>` to the actual order objects, and where would that logic live?**

Answer: I'd derive it, not store it separately — `const selectedOrders = orders.filter(o => selectedIds.has(o.id))`, computed either inline at render or memoized with `useMemo` if `orders` and `selectedIds` are both stable references and the computation is nontrivial (large lists). This stays correct automatically as `orders` refetches or `selectedIds` changes, with no manual sync step; storing a second, separate `selectedOrderObjects` array would require remembering to update it every time either source changes, and would drift the moment someone updates one but not the other — a classic derived-state bug. One nuance worth surfacing: if a selected order is currently on a page that's since been paginated away from client-side cache (React Query evicted it, or it's simply not in the currently-fetched page), `selectedOrders` derived from only the currently-loaded page's data would silently drop that selection from the count — worth explicitly deciding whether cross-page selection needs its own small cache of "selected order summaries" (id + minimal display fields) captured at selection time, separate from the full live order data.

The trap: reaching for a second piece of stored state to hold "the selected objects" instead of deriving them — this is the single most common source of the "state got out of sync" family of bugs, and the fix is almost always "delete the redundant state and compute it instead."

---

**Q (High): A teammate argues the slide-over panel should be pure local UI state (`isOpen`, `selectedOrderId` in a `useState`), not a URL param, because "it's just a UI detail, keeping it out of the URL keeps the URL clean." How do you respond?**

Answer: I'd push back on the premise that it's "just a UI detail" — the actual test is whether a user would ever want to reproduce, bookmark, or share this exact view, and for an order-detail panel in what's described as an internal ops/support-style table, that's a very plausible workflow ("here's the order that's stuck, look at tab 2 of the detail panel"). If the product genuinely has no such workflow and the team has made that call deliberately (not just "it's simpler to skip"), local state is a reasonable choice, and I'd defer to that — this is a product judgment call, not a pure architecture rule, and I'd want to ask rather than assume either way. What I would push back on regardless is *deciding this implicitly by whichever was easier to code first* — it's a five-minute conversation with the product owner, and retrofitting URL-addressability onto an already-shipped local-state panel is meaningfully more work than building it URL-addressable from the start once the team decides they want it.

The trap: treating this as a settled architecture rule ("panels are always/never URL state") instead of the actual decision criterion, which is a product question about shareability — a strong answer names the criterion and asks the clarifying question rather than asserting a universal rule in either direction.

---

**Q (Medium): How would you handle the case where a user has a filtered/sorted URL open in one tab, performs a bulk action that changes several orders' statuses, and now some of the visible rows no longer match the active filter — do they just vanish from the table?**

Answer: This is fundamentally a cache-invalidation question, not a new state-category question — after the bulk action's mutation succeeds, I'd invalidate (or optimistically update) the React Query cache for the `['orders', filters, sort, page]` key(s) currently in view, which triggers a refetch against the current filters and naturally removes rows that no longer match — that's the expected, correct behavior for a live filtered view, not a bug to work around. The UX nuance worth calling out explicitly: silently having rows disappear immediately after a user's own bulk action can be disorienting ("did my action even work, or did something break?") — a brief, deliberate UI acknowledgment (a toast confirming the action, or a short transition/highlight before removal) closes that gap without changing the underlying state-management answer.

The trap: treating "rows disappearing after a mutation" as itself a bug to be prevented (e.g., by freezing the filtered view) rather than recognizing it as correct behavior for a live filter that needs a UX affordance, not an architecture fix.

---

**Q (Medium): If this page needs to support "restore my exact previous session" (user closes the tab, comes back next day, wants filters/sort/page/selection all restored) — which of these four state categories would you change, if any?**

Answer: Filters, sort, and pagination are already covered for free if they're in the URL *and* the user is returning via a bookmarked/previously-shared link — but "closes the tab and comes back later expecting the same URL" only works if something actually preserved that URL (browser history/bookmark), which isn't guaranteed for every return path (e.g., they navigated away within the same SPA session and lost the search params, or opened a fresh tab from a bare bookmark to the page root). If the requirement is specifically "restore state even without a URL carrying it," that pushes filters/sort/page into `localStorage` as a *fallback default* read on initial mount (URL params still win if present, since a shared link should override a stale local default) — not a replacement for URL state, an addition alongside it. Selection, by contrast, I'd deliberately *not* persist across a full session restore — reviving "these 3 rows were checked yesterday" as if it's still an intentional, current action is more likely to cause an accidental bulk action against stale intent than to be a helpful convenience.

The trap: reaching for `localStorage` as a blanket solution for "remember everything," including selection state, where persisting a destructive/committing action's staged state across sessions is actively risky, not just unnecessary.

---

**Q (Low): Would your answer change if this were a mobile app (React Native) instead of a web page — is there still a "URL" bucket?**

Answer: The categories still apply, but the "URL state" bucket's natural home shifts to whatever the platform's equivalent shareable/restorable navigation state is — React Navigation's deep-linking and route params serve a similar role to URL search params (a specific screen + params can still be deep-linked to, and React Navigation persists/restores navigation state across app restarts if configured to), so filters/sort/pagination would still live in route params rather than a generic global store, for the same reasoning (shareability, restorability) — just without a literal address bar exposing it to the user. Server state and form state reasoning is unchanged; session/UI state (selection) still stays local.

The trap: concluding "there's no URL on mobile, so this whole category disappears and everything becomes local/global state" — the underlying reason for the category (shareable, restorable, navigation-addressable state) still exists on mobile, it just has a different concrete mechanism.

---

## Self-Assessment

- [ ] Can name the four state categories unprompted and classify a new piece of state into one of them within a few seconds
- [ ] Can explain, for each category, the specific bug that results from putting it in the wrong bucket (not just "it's bad practice")
- [ ] Can justify keying row selection by ID rather than index, and describe the failure mode of index-based selection
- [ ] Can explain why derived state (selected order objects, indeterminate checkbox) should be computed, not stored
- [ ] Can articulate the actual decision criterion for "does this go in the URL" (shareability/bookmarkability) rather than a blanket rule
- [ ] Can reason through cache invalidation after a mutation affecting a filtered view without treating disappearing rows as a bug

---
*Next: State Design: Filters + Saved Views + Real-time Counters — builds directly on this classification by adding a fifth wrinkle (persisted "saved view" configurations that snapshot URL state) and a sixth (server-pushed real-time data layered on top of a paginated/filtered view).*
