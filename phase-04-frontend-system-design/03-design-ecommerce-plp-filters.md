# Design an E-commerce Product Listing + Filters Page

## Quick Reference

| Decision Point | Choice | Why |
|---|---|---|
| Source of truth for filter/sort/page state | The URL's query string, not component state | Product listing pages must be shareable, bookmarkable, back-button-navigable, and crawlable — state that only lives in memory disappears on refresh/share/new-tab and can't be indexed by search engines |
| Pagination style | Numbered pages (or "Load more" appending pages), not pure infinite scroll, as the default for a commerce grid | Numbered pages preserve a stable, shareable, indexable URL per page and a predictable back-button return-to-exact-scroll-position; pure infinite scroll trades this away for a marginal scroll-continuity gain that matters less on a grid than on a feed |
| Applying a filter change | Update the URL (triggering a re-fetch keyed off the new query string), not a client-only re-filter of an already-fetched page | Real catalogs are too large to ship entirely to the client for local filtering; filtering must happen server-side, with the URL as the request's parameters |
| Rapid filter interactions (e.g., a price slider) | Debounce the URL/fetch update, but update the visible control instantly | The slider itself must feel responsive on every drag frame; the expensive part (re-fetching results) only needs to happen once the user settles on a value |
| Facet option counts (e.g., "Red (12)") | Server-computed and returned alongside results for the *current* filter combination | Client-side count computation would require the full unfiltered dataset locally, which doesn't scale; only the server, which has the full catalog, can compute "how many results if I also add this facet" |

## The Scenario

"Design the product listing page for an e-commerce site — a grid of products with filters (category, price range, brand, rating, in stock) down the side, a sort dropdown, and pagination. Filters need to be combinable, the page needs to be shareable and work correctly with the browser back button, and it should feel fast even though the underlying catalog has millions of items. Walk me through the architecture."

## Clarifying Questions

- **Should the exact filter/sort/page combination be reflected in a shareable, bookmarkable URL** — e.g., can a user send a teammate a link to "shoes, under $50, 4 stars and up, sorted by price," and land on that exact view? This is the central architectural decision for this scenario: it determines whether the URL query string is the source of truth for this state (with components reading from and writing to it) or whether it's acceptable for filters to live purely in component/session state and reset on refresh or share — the former is table stakes for essentially every real commerce site and drives most of the rest of the design.
- **How large is the catalog, and is filtering/faceting expected to happen client-side against a fully-loaded dataset, or does every filter change require a server round trip?** A catalog of a few hundred items could plausibly be shipped to the client and filtered/sorted locally; a catalog of millions cannot — this determines whether "apply filter" means "re-run an in-memory `.filter()`" or "issue a new network request with new query parameters," which changes essentially every subsequent design decision (loading states, debouncing, request cancellation).
- **Do filter options need to show live result counts (e.g., "Running shoes (128)", "Under $50 (43)") that update as other filters are applied?** If so, those counts must come from the server for the *current* combination of already-applied filters (a facet count is inherently relative to what else is selected), which is a materially harder backend/frontend contract than filters with no counts shown at all.
- **Is SEO a requirement for these pages** — should search engines be able to index, say, `/shoes?brand=nike&sort=price-asc` as its own crawlable page with server-rendered product content? This affects whether the initial page load (at least) needs to be server-rendered with the filtered result set already present in the HTML, versus a client-only fetch-after-mount being acceptable.
- **What's the expected interaction pattern for pagination — numbered pages, "Load more" button, or continuous infinite scroll?** Each has different trade-offs around URL shareability (a specific page of infinite-scrolled results is much harder to link to precisely) and back-button behavior (returning to exactly where a user was scrolled to), worth surfacing rather than assuming.
- **Should applying a new filter reset pagination back to page 1, and does changing the sort order need to preserve which filters are active?** These interact — a specific filter combination is state that should generally persist across a sort change, but the current page number generally should not persist across a *filter* change (the result set underneath page 3 has changed), which is a subtlety worth calling out explicitly rather than leaving implicit.

## Approach & Trade-offs

**The URL query string is the single source of truth for filters, sort, and page — not React state that happens to be synced to the URL as an afterthought.** The temptation is to hold `selectedFilters` in `useState`/a store and separately push it to the URL as a side effect "for shareability" — but that ordering (state first, URL second) creates two sources of truth that can drift, and makes the back button unreliable (browser back changes the URL but doesn't automatically revert whatever local state was derived from the *previous* URL unless that sync is built extremely carefully in both directions). The more robust architecture treats the URL as authoritative: reading `useSearchParams` (or equivalent) directly to derive what filters are active and what to fetch, and filter/sort/page-changing UI writes to the URL (`router.push`/`replace` with new query params) rather than to component state — the URL change itself is what triggers re-fetching, and the back button "just works" because navigating back is, definitionally, changing the URL to a previous value, which naturally re-derives the correct filter state and re-fetches.

**`replace` vs. `push` on the history stack matters and is a real design decision, not a detail.** Every keystroke on a price slider shouldn't create a new history entry (a user hitting "back" once after adjusting a slider ten times shouldn't have to hit back ten times to leave the page) — so continuous/rapid adjustments update the URL via `history.replaceState`-equivalent (no new history entry) until the user "commits" a change (releasing the slider, selecting a checkbox), which uses `pushState`-equivalent (a real, back-button-navigable history entry). Getting this wrong in either direction produces a visibly broken back button — too many entries (from `push`-ing every intermediate value) or too few (from `replace`-ing even meaningful, distinct filter selections a user would reasonably want to back out of one at a time).

**Server-side filtering/faceting for any catalog beyond trivially small, because facet counts and full-catalog search fundamentally require server-side computation.** Client-side filtering only works if the client has the full relevant dataset already in memory, which doesn't hold for a real catalog (millions of SKUs) — every filter/sort/page change is a new request with the current URL's query parameters translated into query parameters/a query body for the backend's search/catalog service. This has a real cost (network latency on every filter tweak, versus instant client-side re-filtering for a small in-memory dataset) that's worth naming explicitly as the trade-off being accepted for correctness at scale.

**Debounce the network-triggering update, but never debounce the visible control's own responsiveness.** For a continuous input like a price-range slider, the slider's visual position must update on every single drag frame (60fps, no debounce) — what gets debounced is the resulting URL/fetch update, which should only fire once the value has settled (a short delay after the last drag frame, or explicitly on drag-end/release) rather than firing a network request per pixel of drag movement. This is the same debounce-vs.-responsiveness split used in machine-coding debounce scenarios, applied here to filter UI specifically rather than a raw input handler.

**Facet counts require a specific request shape: "counts for every other facet, given the currently-applied ones."** This is worth naming as a non-trivial backend contract, not just a frontend rendering detail — showing "Under $50 (43)" next to an unchecked price filter means the server computed, for the *currently applied* brand/rating/category filters, how many matching products would remain if price-under-$50 were *also* applied — this is inherently a per-facet, conditional count, not a static count of the whole catalog. The frontend's job is simply to render whatever counts the server returns alongside the current result set and re-fetch this whole structure (results + updated facet counts) on every filter change — it should not attempt to compute or estimate these counts itself from the current page's visible results, which represent only the current page, not the full matching set.

**Pagination as numbered/"load more" pages rather than pure infinite scroll, as the sensible default for a commerce grid (contrasted deliberately with the Design a News Feed scenario's choice).** A feed is consumed more passively and continuously (scrolling is the entire interaction model); a product listing page is more often used with an explicit "compare item 3 against item 40" or "share this page of results" intent, where a specific, stable, linkable page number is genuinely valuable in a way it typically isn't for a social feed. This is worth stating as a deliberate, context-dependent choice rather than a universal rule — the same engineer might correctly choose infinite scroll for one product and numbered pagination for another, and should be able to articulate *why* the two scenarios differ (shareability/discreteness of "a page" vs. continuous passive consumption) rather than picking one pattern reflexively for every list.

## Solution

**Deriving filter state from the URL, and updating it — using Next.js-style `useSearchParams`/`useRouter`, though the pattern generalizes to any router:**

```tsx
function useProductFilters() {
  const searchParams = useSearchParams();
  const router = useRouter();
  const pathname = usePathname();

  const filters = useMemo(() => ({
    category: searchParams.get('category') ?? undefined,
    brand: searchParams.getAll('brand'), // multi-select facet
    minPrice: searchParams.get('minPrice') ? Number(searchParams.get('minPrice')) : undefined,
    maxPrice: searchParams.get('maxPrice') ? Number(searchParams.get('maxPrice')) : undefined,
    minRating: searchParams.get('minRating') ? Number(searchParams.get('minRating')) : undefined,
    sort: searchParams.get('sort') ?? 'relevance',
    page: Number(searchParams.get('page') ?? '1'),
  }), [searchParams]);

  function updateFilters(patch: Partial<typeof filters>, options?: { replace?: boolean }) {
    const next = new URLSearchParams(searchParams.toString());
    for (const [key, value] of Object.entries(patch)) {
      if (value === undefined || value === '') next.delete(key);
      else if (Array.isArray(value)) {
        next.delete(key);
        value.forEach((v) => next.append(key, String(v)));
      } else {
        next.set(key, String(value));
      }
    }
    // A filter change resets pagination — the underlying result set changed.
    if (!('page' in patch)) next.delete('page');

    const url = `${pathname}?${next.toString()}`;
    options?.replace ? router.replace(url) : router.push(url);
  }

  return { filters, updateFilters };
}
```

**The price slider — instant visual response, debounced URL commit, explicit history handling:**

```tsx
function PriceRangeFilter({ min, max, onCommit }: {
  min?: number; max?: number; onCommit: (min: number, max: number) => void;
}) {
  const [draft, setDraft] = useState<[number, number]>([min ?? 0, max ?? 500]);
  const debouncedDraft = useDebouncedValue(draft, 400);

  // Runs on every drag frame — purely visual, no network involvement.
  useEffect(() => { /* render `draft` in the slider's thumbs */ }, [draft]);

  // Fires only once dragging has settled — this is what triggers the URL/fetch update.
  useEffect(() => {
    onCommit(debouncedDraft[0], debouncedDraft[1]);
  }, [debouncedDraft]);

  return <RangeSlider value={draft} onChange={setDraft} min={0} max={500} />;
}

// Usage: intermediate drag updates use `replace` (no history spam);
// the settled/committed value is what the parent's `updateFilters` call
// ultimately pushes to history via a real navigation entry.
```

**Fetching results + facet counts together, keyed by the URL's current filter state:**

```tsx
function useProductResults(filters: ProductFilters) {
  return useQuery({
    queryKey: ['products', filters],
    queryFn: ({ signal }) => fetchProducts(filters, signal), // cancels the previous in-flight fetch on filter change
    placeholderData: keepPreviousData, // show the previous grid, dimmed, instead of a blank flash while refetching
  });
}

interface ProductResults {
  items: Product[];
  totalCount: number;
  facets: {
    brand: { value: string; count: number }[];
    rating: { value: number; count: number }[];
    // "count" here is server-computed relative to every OTHER currently-applied filter
  };
}
```

> **Check yourself:** Without looking above, explain why treating the URL as the single source of truth (rather than syncing a separate piece of component state to it) is what makes the browser's back button work correctly here, and what specifically breaks if state-then-URL-sync is used instead.

## State Architecture

| State | Lives In | Why |
|---|---|---|
| Active filters, sort, page | URL query string | Must be shareable, bookmarkable, back/forward-navigable, and (if SEO matters) crawlable |
| In-progress slider drag value | Local component state | Must update at 60fps with zero network/URL involvement; committing it is a separate, debounced step |
| Fetched results + facet counts | Server-state cache (React Query/SWR), keyed by the full filter object | This is server-owned data with its own staleness/loading lifecycle, distinct from client-only UI state |
| "Filters panel expanded/collapsed" (mobile) | Local/session UI state, not URL | Purely a UI-chrome preference with no shareability requirement — putting it in the URL would just add noise to every link |

## Gotchas

**Syncing component state to the URL as an afterthought instead of deriving state from the URL directly.** Works fine until the back button is tested — the URL changes on back navigation, but nothing then re-derives the "selected filters" state from it if the sync was only ever built one-directionally (state → URL), leaving the UI showing filters that no longer match the URL/fetch that just happened.

**Debouncing the slider's visual thumb position itself, not just the resulting fetch.** Produces a visibly laggy slider (thumb appears to lag behind the pointer) — only the expensive, network-triggering side of the interaction should be debounced; the input's own visual feedback must stay perfectly synchronous with pointer movement.

**Using `pushState`-equivalent navigation for every intermediate value during a continuous drag.** Fills the back-button history with dozens of near-identical entries from one slider interaction, making "go back" require many presses to actually leave the filter state behind — intermediate updates should use `replace`, with only the settled/committed value creating a real history entry.

**Not resetting the page number when a filter changes.** Landing on "page 4" of a result set that, after adding a new filter, might only have 2 pages total produces an empty or broken page — any filter change should reset pagination to page 1, while a *sort* change on the same filter set can reasonably preserve which page number the user was on (arguable, but worth having an explicit answer rather than an accidental one).

**Computing facet counts client-side from the current page's visible items.** The current page is a small slice of the total matching set — counting "Nike: 3" from only the 20 products currently rendered is not the same number as the true count across all matching products, and presenting it as if it were is simply incorrect; facet counts must come from the server, computed against the full matching set.

**Showing a blank/empty grid while refetching after a filter change, instead of keeping the previous results visible (dimmed/with a loading indicator) until the new ones arrive.** A full blank-then-repopulate flash on every filter tweak reads as janky; keeping the previous result set visible (`keepPreviousData`-style) while the new request is in flight, then swapping once it resolves, is a materially smoother-feeling default.

## Follow-up Questions

**Q (High): Why does deriving filter state directly from the URL (rather than syncing separately-held component state to the URL) matter specifically for the back button, and not just as a style preference?**

Answer: The back button's actual mechanism is "change the URL to a previous value in history" — nothing more. If the filter UI's displayed state is read directly from the current URL on every render, a back-button-triggered URL change automatically and correctly re-derives what filters should show and what should be fetched, with zero additional wiring, because "the URL changed" and "re-derive state from the URL" are the same code path used on every navigation, not a special case for back specifically. If instead filter state lives in `useState` and is *pushed to* the URL as a side effect, back-button navigation changes the URL but there's no guarantee anything re-reads that new URL back into the component state unless a `popstate`/URL-change listener is separately, correctly wired to re-sync in the reverse direction — miss that, and the UI silently shows stale filters that don't match what's actually in the address bar or what would be fetched on a fresh load of that same URL.

The trap: describing this as "just remembering to add a listener for back navigation" — the deeper point is that deriving-from-URL makes correct back-button behavior a natural consequence of the architecture (one direction of data flow, always read from source of truth), whereas state-then-sync makes it a separately-maintained special case that's easy to get subtly wrong or forget entirely.

---

**Q (High): A user applies three filters in quick succession (checking three brand checkboxes back to back). Should this fire three separate network requests, or be batched into one?**

Answer: This depends on the interaction model established for the checkboxes specifically — discrete, deliberate clicks (as opposed to a continuous drag like the price slider) can reasonably fire a request per click if each one is a meaningful, intentional state change the user might want to see reflected immediately (and might want in their back-button history individually) — but if checkbox interactions are expected to happen in a rapid burst (a "select all in this category" pattern, or fast keyboard-driven toggling), a short debounce (much shorter than the price slider's, since these are discrete rather than continuous, e.g., 100–150ms) around the resulting fetch avoids firing three nearly-simultaneous, mostly-wasted requests where only the third's result actually matters. Either is defensible; what's not defensible is not having considered it at all — the same request-cancellation and sequence-guarding principles from the autocomplete scenario apply here too, since a fast-changing filter state can produce out-of-order responses just as fast-changing search input can.

The trap: assuming filters are inherently "different" from a search-input race condition and therefore don't need the same cancellation/sequence-guard treatment — any UI where user input can trigger multiple in-flight requests for what's conceptually "the current state" is subject to the same class of race condition regardless of whether the input is a text box or a set of checkboxes.

---

**Q (High): How would you keep the facet counts accurate and non-misleading as a user is applying filters one at a time — specifically, should a facet's own count include or exclude itself from the "currently applied" set it's conditioned on?**

Answer: The standard, correct approach is that each facet group's counts are computed against every *other* currently-applied filter, but not against filters within that same facet group — e.g., if "Nike" and "Adidas" are both brand options, the count shown next to "Adidas" should reflect "how many results if Adidas were added, given the current category/price/rating filters" (excluding any currently-selected brand filters from that specific calculation), otherwise selecting "Nike" would make every other brand's count collapse toward zero (since almost nothing is both Nike and Adidas), which would be misleading — the point of showing "Adidas (17)" is "if you'd chosen Adidas *instead*/*in addition*, depending on whether brand is single- or multi-select," not "here's what's left after Nike is already locked in and can't be changed." Whether multiple values within one facet are OR'd together (multi-select: "Nike or Adidas") or mutually exclusive is itself a product decision that changes this math, and should be confirmed rather than assumed.

The trap: implementing counts as strictly "given every filter currently selected, including this facet group" — technically simpler to compute, but produces the confusing near-zero-count effect described above the moment more than one option in the same facet group could otherwise coexist.

---

**Q (Medium): Should the initial page load (first visit to a filtered URL, e.g., via a shared link) be server-rendered, or is a client-side fetch-after-mount acceptable?**

Answer: This traces back to the SEO/crawlability requirement clarified up front — if these filtered URLs need to be indexable (a real, common e-commerce SEO strategy, since "waterproof hiking boots under $100" as its own crawlable, well-ranked page is valuable), the initial HTML response needs the actual product grid already present, meaning server-side rendering (or an equivalent pre-render/ISR strategy) reading the same URL query parameters that the client-side filter logic reads, hitting the same backend search/catalog API before the response is sent. If SEO isn't a requirement (e.g., an internal tool, or a product that's fine relying entirely on client-side navigation for all filter states), a client-fetched approach with a loading skeleton on first paint is simpler and avoids maintaining a server-rendering code path that mirrors the client one.

The trap: treating this as a universal "always SSR product pages" rule without connecting it back to the specific requirement (SEO/crawlability) that makes it necessary — the correct answer is conditional and should be stated as such, not recited as a best practice independent of the actual product need.

---

**Q (Medium): A user filters, scrolls halfway down page 2, then clicks into a product detail page and later hits browser back. What should they see?**

Answer: They should land back on the exact same filtered/sorted/paginated URL they left from, ideally scrolled to approximately the same position they were at — the URL-as-source-of-truth architecture gets the filter/sort/page part of this right "for free" (navigating back restores the same URL, which re-derives the same filters and re-fetches or serves from cache the same result set), but scroll position specifically requires either the browser's native scroll restoration (`history.scrollRestoration`, which works reasonably well for this exact case in most modern browsers if not fought against) or, if the app manages scroll manually (common in SPA routing setups), explicitly capturing and restoring scroll offset keyed to that history entry. This is worth naming as an additional, distinct piece of state (scroll position) beyond the filter/sort/page state already covered by the URL.

The trap: assuming "the URL is correct on back navigation" automatically means "the page looks exactly as it did" — scroll position is a separate concern from query-string state and needs its own explicit handling (or explicit reliance on native browser behavior) rather than being assumed to come along for free.

---

**Q (Low): How would this design change for a small, fixed catalog (a few hundred items) where the entire dataset could reasonably be sent to the client?**

Answer: With a small enough dataset, it becomes legitimate to fetch the full (or a reasonably large superset of the) product list once, and perform filtering, sorting, and even facet-count computation entirely client-side against that in-memory set — no network round trip per filter change, and facet counts become a simple client-side `.filter().length` computation per option rather than a server contract. The URL-as-source-of-truth principle for shareability/back-button behavior still fully applies (that's about UX correctness, independent of dataset size) — what changes is only whether "apply this filter" means "re-run a local `Array.filter`" or "issue a network request," which is purely a function of whether the full relevant dataset can reasonably live in client memory.

The trap: assuming a smaller dataset removes the need for the URL-driven architecture too — shareability, bookmarkability, and back-button correctness are UX requirements independent of where the filtering computation happens, and a correct answer keeps the URL-as-source-of-truth design regardless of catalog size, only relaxing the "must hit the server per change" requirement.

---

## Self-Assessment

- [ ] Can explain why the URL (not synced component state) needs to be the single source of truth for filter/sort/page state, and what specifically breaks on back-button navigation otherwise
- [ ] Can articulate the `replace` vs. `push` history distinction for continuous vs. discrete filter interactions
- [ ] Can design the debounce split between a control's own visual responsiveness and the network-triggering side effect
- [ ] Can explain why facet counts must be server-computed relative to the *other* currently-applied filters, and the "counting within the same facet group" subtlety
- [ ] Can justify numbered pagination vs. infinite scroll as a context-dependent choice, not a universal default, contrasting this scenario against the News Feed scenario
- [ ] Can reason through what additional state (scroll position) needs handling beyond what URL-derived filter state covers for granted

---
*Next: Design a Chat Application (WhatsApp Web-style) — moves from request/response-shaped state (fetch results for the current URL) to a persistent, bidirectional real-time connection, where the central new concerns become message ordering/delivery guarantees, optimistic sends, and reconnection handling.*
