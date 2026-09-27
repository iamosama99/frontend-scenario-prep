# Data Table — Sort, Filter, Paginate

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Pipeline order | `filter → sort → paginate`, always in that order, on the full dataset | Paginating first slices the dataset before filter/sort ever see most of it — you'd sort/filter only the current page, producing wrong results |
| Derived data | Recompute the pipeline from the raw dataset, don't mutate it in place | The raw dataset is the single source of truth; in-place mutation makes "reset filters" and "reset sort" impossible without re-fetching |
| Stable sort | Native `Array.prototype.sort` (ES2019+) is spec-stable, but ties still need an explicit tie-breaker for determinism | Two rows with equal sort-column values can visually "jump" between renders if you rely on stability alone across re-sorts by different criteria |
| Memoization | Cache each pipeline stage keyed on its own inputs (filters, sort, page) | Typing in an unrelated text filter shouldn't re-sort or re-slice work that hasn't changed |
| Accessible headers | `<button>` inside `<th>`, `aria-sort` on the `<th>` | Sortable columns need a real interactive control and must announce sort state to assistive tech |
| URL as state | Serialize `sortColumn`, `sortDirection`, `filters`, `currentPage` into the querystring | A shared/reloaded link must reproduce the exact view — state that only lives in memory is lost on refresh |

## The Scenario

"We need a data table for our admin dashboard — say, a list of orders. Users should be able to sort by clicking a column header, filter by a couple of fields like status and customer name, and page through the results. Can you build the logic for that? Assume the data's already loaded client-side as an array; you don't need to worry about a real backend."

## Clarifying Questions

- **Is sorting/filtering/pagination happening client-side on data we already have, or does each action need a new server request?** The prompt says data's already loaded client-side — that changes the entire architecture. If it were server-driven, "current page" would just be whatever the last response contained, and there'd be no local derived-data pipeline to reason about at all; the interviewer is deliberately scoping this to the client-side case.
- **Can multiple filters be active at once, and are they AND'd or OR'd together?** This determines whether the filter step is a single predicate or a composition of predicates. Real admin tables almost always AND multiple active filters (status = "shipped" AND customer contains "acme"), so I'd confirm that before writing the filter function's signature.
- **Is sort single-column or does it need multi-column (secondary sort) support?** Single-column click-to-sort is the common ask, but if the interviewer wants "sort by status, then by date within status," that's a materially different sort comparator, and I'd rather ask than guess.
- **Should the current sort/filter/page state survive a page reload or be shareable via link?** This is the URL-as-state requirement — if yes, state has to be serialized into the querystring rather than kept only in a JS variable or component state.
- **What happens to `currentPage` when a filter changes and the result set shrinks below the current page number?** A concrete edge case: if the user is on page 5 and a new filter leaves only 2 pages of results, page 5 is now out of range. I'd clamp back to a valid page (usually page 1, or `min(currentPage, lastPage)`) rather than rendering an empty page silently.

## Approach & Trade-offs

The central design decision is treating sort/filter/page as **derived state**, computed from one raw array plus a small set of "view" parameters — never mutating the raw array itself. That's what makes "clear filters" or "reset sort" trivial: you just reset the parameters and recompute, instead of trying to undo destructive mutations.

The second decision is pipeline **order**: filter, then sort, then paginate. This has to run in that exact sequence because each stage's output size feeds the next stage's semantics. If you paginate before filtering, you're filtering only the 10 rows already on the current page instead of the full dataset — the filtered result depends on which page you happened to be on, which is nonsensical. If you paginate before sorting, "page 2" shows arbitrary rows in arbitrary order rather than the 11th–20th row of a globally sorted list. So the only correct order is: reduce to matching rows (filter) → establish global order over the matches (sort) → slice a window out of that ordered result (paginate).

For sort stability, I lean on the fact that `Array.prototype.sort` has been spec-guaranteed stable since ES2019 — so if two rows compare equal on the active sort column, their *relative input order* is preserved rather than shuffled. That's good, but it's not the same as a deterministic secondary sort: stability only preserves whatever order the array happened to be in *before* this sort call, which itself might be leftover order from a previous sort by a different column. If the interviewer asks for "deterministic tie ordering" (e.g., ties broken by id, always), that has to be an explicit second comparison inside the comparator — stability alone doesn't give you that, it just avoids gratuitous reshuffling.

For performance, I'd avoid re-running the entire pipeline from scratch on every keystroke in an unrelated input. Practically: memoize each stage keyed on its own inputs, so filtering only re-runs when `filters` changes, sorting only re-runs when the filtered set or sort params change, and pagination is just a cheap `slice` that can run every time since it's O(pageSize), not O(n). This is a smaller-scale version of the classic "don't recompute what didn't change" instinct — with a dataset of a few thousand rows it's a micro-optimization, but it's the right instinct to demonstrate, and it becomes real once the dataset gets large or the filter predicate is expensive (e.g., a fuzzy string match across several fields).

An alternative I'd mention and reject: keeping three *separate* mutable copies of the array (a "filtered array," a "sorted array," a "paginated array") that each get mutated as state changes. I'd reject that because it invites the exact filter-before-paginate-order bugs above — it's too easy for one stage's cached output to go stale relative to another's inputs. A single `getVisibleRows()` derivation function that always runs filter → sort → paginate in order, given the current raw data and params, is simpler to reason about and to test.

## Solution

State shape first — this is the part interviewers actually grade closely:

```javascript
const state = {
  rawData: [],            // source of truth, never mutated
  filters: {               // AND'd together
    status: null,          // e.g. "shipped" | null (no filter)
    customerQuery: '',      // substring match, case-insensitive
  },
  sortColumn: 'date',
  sortDirection: 'desc',    // 'asc' | 'desc' | null (unsorted)
  currentPage: 1,
  pageSize: 10,
};
```

The filter stage:

```javascript
function applyFilters(data, filters) {
  return data.filter((row) => {
    if (filters.status && row.status !== filters.status) return false;
    if (
      filters.customerQuery &&
      !row.customerName.toLowerCase().includes(filters.customerQuery.toLowerCase())
    ) {
      return false;
    }
    return true;
  });
}
```

The sort stage — note the explicit tie-breaker on `id`, which is what makes ordering deterministic across re-renders even though native sort is already stable:

```javascript
function applySort(data, column, direction) {
  if (!direction) return data; // 'none' state — leave filtered order as-is

  const dir = direction === 'asc' ? 1 : -1;

  // Slice first: sort() mutates in place, and `data` here is the filtered
  // array from the previous stage — mutating it would corrupt a cached
  // reference if the caller (or a memo layer) is holding onto it.
  return [...data].sort((a, b) => {
    const primary = compareValues(a[column], b[column]);
    if (primary !== 0) return primary * dir;
    // Deterministic secondary order on ties, independent of `direction`,
    // so ties don't flip when the user toggles asc/desc.
    return a.id < b.id ? -1 : a.id > b.id ? 1 : 0;
  });
}

function compareValues(a, b) {
  if (typeof a === 'number' && typeof b === 'number') return a - b;
  return String(a).localeCompare(String(b));
}
```

The paginate stage, and the full pipeline tying it together in the correct order:

```javascript
function applyPagination(data, page, pageSize) {
  const start = (page - 1) * pageSize;
  return data.slice(start, start + pageSize);
}

function getVisibleRows(state) {
  const filtered = applyFilters(state.rawData, state.filters);
  const sorted = applySort(filtered, state.sortColumn, state.sortDirection);
  const paginated = applyPagination(sorted, state.currentPage, state.pageSize);

  return {
    rows: paginated,
    totalMatching: filtered.length,
    totalPages: Math.max(1, Math.ceil(filtered.length / state.pageSize)),
  };
}
```

Memoizing per-stage so an unrelated keystroke doesn't redo sort/filter work:

```javascript
function createDerivedTable(rawData) {
  let cache = { filters: null, filtered: null, sortColumn: null, sortDirection: null, sorted: null };

  return function getVisibleRows(state) {
    const filtersChanged = JSON.stringify(cache.filters) !== JSON.stringify(state.filters);
    if (filtersChanged) {
      cache.filtered = applyFilters(rawData, state.filters);
      cache.filters = state.filters;
    }

    const sortChanged =
      filtersChanged ||
      cache.sortColumn !== state.sortColumn ||
      cache.sortDirection !== state.sortDirection;
    if (sortChanged) {
      cache.sorted = applySort(cache.filtered, state.sortColumn, state.sortDirection);
      cache.sortColumn = state.sortColumn;
      cache.sortDirection = state.sortDirection;
    }

    // Pagination is O(pageSize) — cheap enough to always redo, no cache needed.
    const paginated = applyPagination(cache.sorted, state.currentPage, state.pageSize);
    return { rows: paginated, totalMatching: cache.sorted.length };
  };
}
```

Accessible sortable header, wired to update state and re-render:

```javascript
function renderSortableHeader(th, column, label, state, onSortChange) {
  const isActive = state.sortColumn === column;
  const ariaSort = isActive
    ? state.sortDirection === 'asc' ? 'ascending'
    : state.sortDirection === 'desc' ? 'descending'
    : 'none'
    : 'none';

  th.setAttribute('aria-sort', ariaSort);
  th.innerHTML = ''; // clear previous button on re-render

  const button = document.createElement('button');
  button.textContent = isActive ? `${label} ${state.sortDirection === 'asc' ? '▲' : '▼'}` : label;
  button.addEventListener('click', () => {
    const nextDirection =
      isActive && state.sortDirection === 'asc' ? 'desc' :
      isActive && state.sortDirection === 'desc' ? null :
      'asc';
    onSortChange(column, nextDirection);
  });

  th.appendChild(button);
}
```

Keeping state in the URL so a reload/share reproduces the view:

```javascript
function stateToQueryString(state) {
  const params = new URLSearchParams();
  params.set('sort', state.sortColumn);
  params.set('dir', state.sortDirection ?? '');
  params.set('page', String(state.currentPage));
  if (state.filters.status) params.set('status', state.filters.status);
  if (state.filters.customerQuery) params.set('q', state.filters.customerQuery);
  return params.toString();
}

function queryStringToState(search, defaults) {
  const params = new URLSearchParams(search);
  return {
    ...defaults,
    sortColumn: params.get('sort') ?? defaults.sortColumn,
    sortDirection: params.get('dir') || null,
    currentPage: Number(params.get('page')) || 1,
    filters: {
      status: params.get('status') || null,
      customerQuery: params.get('q') || '',
    },
  };
}

// On every state change: history.replaceState(null, '', `?${stateToQueryString(state)}`);
// On load: state = queryStringToState(window.location.search, defaults);
```

> **Check yourself:** If you paginate before filtering, what specifically breaks — walk through a concrete example with numbers, not just "it's wrong."

## Gotchas

**Wrong pipeline order.** Filtering after paginating means "page 2" shows the 11th–20th rows of the *unfiltered* set, then filters those 10 down further — you might end up with 3 visible rows on a page that's supposed to hold 10, and the total-pages count becomes meaningless. This is the single most common bug in this scenario.

**Stale `currentPage` after filters/sort change.** Changing a filter can shrink the result set below the current page number. Forgetting to clamp `currentPage` back into range produces an empty-looking table that looks broken even though the logic is technically "correct."

**Mutating the raw dataset in `sort()`.** `Array.prototype.sort` sorts in place. Calling it directly on `rawData` (or on a reference shared with a previous cached stage) silently corrupts the source of truth — always sort a copy (`[...data].sort(...)`).

**Relying on stability instead of an explicit tie-breaker.** Stability preserves *whatever order the array was already in*, which after a previous sort-by-different-column is not a meaningful tiebreak — it's leftover order. If the interviewer probes "what if two rows have the same date," a "sort is stable so it's fine" answer without an explicit secondary key is incomplete.

**`aria-sort` set on the wrong element.** It belongs on the `<th>`, not on the `<button>` inside it — screen readers look for it on the header cell.

**URL state drifting from UI state.** If you update `history.replaceState` in some code paths but not others (e.g., page-size change), a reload reproduces a different view than what the user was actually looking at. Every state-changing action needs to funnel through one function that updates both the in-memory state and the URL together.

## Follow-up Questions

**Q (High): Why must filtering happen before pagination, with a concrete numeric example?**

Answer: Say there are 100 rows, `pageSize = 10`, and a status filter matches only 25 of them. If you paginate first, "page 2" is rows 11–20 of the full 100-row set, and *then* applying the filter to just those 10 rows might leave you with, say, 3 matching rows — even though the correct "page 2" of the filtered 25-row result should show rows 11–20 of the *filtered* set (a full 10 rows), and there should only be 3 total pages, not 10. Filtering first establishes the correct universe (25 rows) before pagination slices a window out of it.

The trap: describing the bug only vaguely ("it shows wrong results") instead of walking through actual row/page numbers — the interviewer wants to see you reason about it mechanically, not just recite the rule.

---

**Q (High): `Array.prototype.sort` is spec-guaranteed stable now — does that mean you never need a manual tie-breaker?**

Answer: No. Stability guarantees that two elements comparing equal keep their *relative input order* — it says nothing about what that input order *means*. If the array's current order is itself leftover from a previous sort by a different column, "stable" just preserves that unrelated ordering for ties, which isn't deterministic from the user's perspective — toggling sort direction on a column with duplicate values can make tied rows appear to jump around depending on prior state. An explicit secondary sort key (commonly a stable unique id) guarantees the same tie order every time, regardless of history.

The trap: answering "sort is stable, so ties are handled" as if stability alone solves determinism — it solves *reshuffling*, not *meaningful* tie order.

---

**Q (High): How do you avoid re-running the whole filter/sort/paginate pipeline on every keystroke in an unrelated input?**

Answer: Split the pipeline into independently-memoized stages, each keyed on only the inputs that affect it: filtering is recomputed only when `filters` changes, sorting only when the filtered set or sort params change, and pagination — being O(pageSize), not O(n) — can just always re-run since it's cheap regardless. Practically this means caching each stage's last inputs and output, and comparing new inputs against the cache before redoing the work, rather than deriving everything from scratch inside one function every render.

The trap: over-engineering this for a small dataset (a few hundred rows) where the "inefficiency" is imperceptible — the interviewer wants to hear you reason about *when* this matters (large datasets or expensive predicates), not blindly add memoization everywhere.

---

**Q (Medium): How do you keep the table's state shareable via URL without causing a history-entry explosion as the user types in a filter?**

Answer: Use `history.replaceState` (not `pushState`) for most state updates, since sort/filter/page changes are refinements of the same "view," not new navigational destinations — `pushState` on every keystroke would flood browser history and make the back button useless. It's also worth debouncing the URL write for free-text filters specifically (not for sort/page clicks, which are discrete), so a fast typist doesn't trigger a `replaceState` call per keystroke.

The trap: using `pushState` for every change, which technically "works" but destroys back-button usability — a very common real-world regression.

---

**Q (Medium): What changes about this design if sorting/filtering/pagination must happen server-side instead of client-side?**

Answer: The derived-data pipeline concept doesn't go away, but who runs it changes — the client sends `sortColumn`, `sortDirection`, `filters`, and `page`/`pageSize` as request parameters, and the server returns only the current page's rows plus a total count; the client no longer holds the full dataset at all. This also changes debouncing strategy (filter changes should be debounced before firing a network request, not just before a local recompute) and means "total pages" comes from the server's count rather than a local `Math.ceil`. The URL-as-state requirement becomes even more valuable here, since it's what lets you reconstruct the exact request parameters on reload.

The trap: assuming client-side derived-state logic "just moves" to the server unchanged — the request/response shape and debouncing strategy both need to be redesigned around network latency.

---

**Q (Medium): How would you support a secondary sort — e.g., sort by status, then by date within status?**

Answer: Extend `sortColumn` from a single value to an ordered list of `{ column, direction }` pairs, and have the comparator iterate through them in order, returning the first non-zero comparison result and falling through to the next column on ties (with the deterministic id tie-breaker still last in the chain). UI-wise, this usually means shift-click to add a secondary sort column, similar to spreadsheet multi-column sort conventions.

The trap: trying to encode "sort by A then B" as two separate sequential `.sort()` calls — that doesn't work, because the second sort's stability only preserves order from the *first* sort if the second sort's comparator returns 0 for ties on the second column, which requires a combined comparator anyway; it's not meaningfully simpler than just writing the combined comparator directly.

---

**Q (Low): Why use `<button>` inside `<th>` rather than making the `<th>` itself clickable?**

Answer: A `<th>` isn't a native interactive element — attaching a click handler directly to it produces something that's mouse-clickable but invisible to keyboard users (not focusable, not in the tab order, no Enter/Space activation) and unannounced to screen readers as actionable. Wrapping a real `<button>` inside the `<th>` gets focusability, keyboard activation, and correct semantics for free, while `aria-sort` on the outer `<th>` (not the button) communicates the current sort state per the ARIA table pattern.

The trap: adding `role="button"` and a `tabindex`/keydown handler to the `<th>` itself to "make it accessible" — that's reinventing what a native `<button>` already provides correctly, with more surface area for bugs (e.g., forgetting Space vs. Enter handling).

---

**Q (Low): What's a reasonable approach if `rawData` is large enough (tens of thousands of rows) that even filtering becomes slow on every keystroke?**

Answer: Debounce the filter *input* itself (distinct from memoizing the pipeline stage) so the filter predicate only runs after the user pauses typing, not on every keystroke; for very large datasets, consider moving the expensive predicate off the main thread via a Web Worker so typing doesn't visibly jank, or moving the dataset server-side entirely once client-side filtering stops being viable. Virtualizing the rendered rows (only rendering the visible page's DOM) is a separate, complementary optimization — it doesn't help the filter/sort compute cost.

The trap: conflating "the pipeline is memoized" with "the pipeline is fast" — memoization avoids *redundant* recomputation, but the first computation on a genuinely large dataset can still be slow, and that's a different problem requiring debouncing or offloading, not caching.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can state the correct pipeline order (filter → sort → paginate) and explain with numbers why reordering breaks it
- [ ] Can explain why native stable sort doesn't eliminate the need for an explicit tie-breaker
- [ ] Can design a state shape that cleanly separates raw data from derived view parameters
- [ ] Can implement per-stage memoization so an unrelated filter keystroke doesn't re-sort
- [ ] Can build an accessible sortable header using `<button>` + `aria-sort` correctly placed
- [ ] Can serialize/deserialize table state to/from the URL querystring using `replaceState`

---
*Next: Drag-and-Drop Sortable List — from clicking to reorder (sort) to dragging to reorder, with a much larger accessibility gap to close.*
