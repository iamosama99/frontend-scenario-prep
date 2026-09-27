# Infinite Scroll List

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Trigger fetch on scroll | `IntersectionObserver` watching a sentinel element at the list's end | No scroll-event listener on the main thread, no manual throttling math |
| Prevent duplicate fetches | An `isLoading` flag checked before firing, set before the request starts | Fast scrolling / observer firing twice can otherwise trigger overlapping page requests |
| Accessibility fallback | A visible "Load more" button, always present, that does the same fetch | Pure scroll-triggered loading breaks keyboard-only users, screen reader "find the footer" navigation, and users who scroll fast past the trigger zone |
| Cleanup | `observer.unobserve(sentinel)` / `observer.disconnect()` on teardown | An observer callback firing after the component/view is gone is a memory leak and a source of "setState after unmount"-style bugs |
| Layout shift | Reserve space (skeletons, fixed-height rows, or `min-height`) for incoming items | Popping in unsized content shifts everything below it — a CLS regression and a jarring UX |

## The Scenario

"We have a feed — think a social timeline or a product listing — and instead of pagination buttons, I want it to load more items automatically as the user scrolls near the bottom. Handle loading state, and don't fetch the same page twice if the user scrolls fast. Also, I care about accessibility — this can't be the *only* way to reach content further down."

## Clarifying Questions

- **Is the total item count known upfront, or do we keep fetching until the API tells us there's nothing left?** This determines whether there's a terminal "no more results" state to design for, and whether the sentinel should ever stop being observed. I'd assume the API returns something like `hasMore: boolean` or a `nextCursor` that's `null` at the end, and confirm.
- **Cursor-based or offset/page-number pagination?** Cursor-based is more robust against items being inserted/removed between fetches (offset pagination can skip or duplicate items if the underlying list changes while the user is scrolling); I'd ask which the backend supports rather than assume.
- **Does "don't fetch the same page twice" need to survive rapid scroll — i.e., firing the intersection callback multiple times before the first fetch resolves?** Yes, and this is the core race condition of the whole widget: a fast scroll (or a low `threshold`/generous `rootMargin`) can trigger the observer callback again before an in-flight request finishes, so a simple "did we fetch this page already" check isn't enough — the guard has to be "is a fetch currently in flight," checked and set synchronously before the async work starts.
- **What happens on scroll-back navigation — should the list preserve its scroll position and already-loaded items?** This matters a lot for the class of UI this pattern is usually used for (a feed a user opens, scrolls through, then navigates away from a specific item and hits back) — re-fetching from page 1 and losing the scroll position is a very common and very annoying real bug.
- **Is a "Load more" button acceptable as the *only* trigger for some users, or does it need to be genuinely equivalent for a screen reader user, not just present?** I'd push for it being a fully first-class, always-rendered control (not a hidden fallback that only appears if JS/observer support is missing), since "sighted mouse users get infinite scroll, everyone else gets a degraded experience" is a common failure mode this scenario is specifically testing for.

## Approach & Trade-offs

**Trigger mechanism: `IntersectionObserver` over a scroll-event listener.** A `scroll` event fires continuously and on the main thread — even throttled/`requestAnimationFrame`-batched, it still requires the browser to run JS on every scroll tick to check "has the sentinel entered the viewport," which is exactly the kind of work `IntersectionObserver` does natively, off the main thread's per-frame JS execution, only invoking a callback when the intersection state actually *changes*. Using `IntersectionObserver` means: no manual throttling code to get right, no risk of janking scroll performance on lower-end devices, and no math computing element positions relative to `scrollTop` by hand.

**Duplicate-fetch prevention: a synchronous in-flight flag, not a "have I fetched this page" set.** The naive fix — track fetched page numbers in a `Set` and skip if already present — doesn't actually solve the race, because the race is about *concurrent* fetches of the *same next page*, not re-fetching an *already-completed* page. The actual bug: the observer's callback can fire again (sentinel re-enters the viewport, or firing twice due to `threshold` edge behavior) while a fetch for page N is still in flight, before page N's items have been appended and the sentinel has moved further down — so a second fetch for page N (not page N+1) starts. The fix is a boolean `isLoading` flag, set to `true` synchronously the instant a fetch decision is made (before any `await`), and checked at the very top of the trigger handler — that's what actually closes the race window, since nothing async happens between the check and the flag being set.

**Infinite scroll alone vs. infinite scroll + "Load more" button, always both.** This is the trade-off the scenario explicitly calls out, and it's worth stating directly rather than defaulting to "just infinite scroll" out of habit: pure infinite scroll (a) traps keyboard users who can't easily "scroll" via Tab in the way a mouse-wheel scroll works, (b) breaks the "jump to the footer" pattern some users and some assistive tech rely on (the footer keeps receding as more content loads, functionally unreachable), and (c) can be jarring for screen reader users, whose linear reading order suddenly gets a pile of new content injected mid-stream with no announcement. A visible, keyboard-focusable "Load more" button that triggers the exact same fetch function isn't a compromise fallback bolted on for compliance — it's the more robust primary mechanism, with the `IntersectionObserver` as a progressive-enhancement convenience for users who benefit from it (which is most users, most of the time).

**Scroll position restoration on back-navigation** is a separate concern from the fetch/render loop itself — in a vanilla-JS single-page context this means persisting (in `sessionStorage`, or an in-memory cache keyed by route) both the already-fetched items *and* the scroll offset when navigating away, and replaying both when navigating back, rather than re-running the widget from an empty state. I'd flag this as scope to confirm rather than build reflexively — some products intentionally reset an infinite feed on re-entry.

## Solution

Markup — the sentinel is a real, empty element at the end of the list, and the "Load more" button is always rendered alongside it:

```html
<div class="feed-container">
  <ul id="feed-list" aria-live="polite"></ul>
  <div id="sentinel" style="height: 1px;"></div>
  <button id="load-more-btn" type="button">Load more</button>
  <p id="feed-status" role="status"></p>
</div>
```

State and config:

```javascript
const listEl = document.getElementById('feed-list');
const sentinel = document.getElementById('sentinel');
const loadMoreBtn = document.getElementById('load-more-btn');
const statusEl = document.getElementById('feed-status');

let isLoading = false;
let nextCursor = null; // null initially; becomes null again (terminal) when API says no more
let hasFetchedOnce = false;
```

The shared fetch function — both the observer and the button call *this same* function, so there's exactly one code path to get right:

```javascript
async function loadNextPage() {
  if (isLoading) return; // closes the race: set/checked synchronously, no await between them

  isLoading = true;
  loadMoreBtn.disabled = true;
  statusEl.textContent = 'Loading more items…';

  try {
    const res = await fetch(`/api/feed?cursor=${nextCursor ?? ''}`);
    if (!res.ok) throw new Error(`Fetch failed: ${res.status}`);
    const { items, nextCursor: newCursor } = await res.json();

    appendItems(items);
    nextCursor = newCursor;
    hasFetchedOnce = true;

    if (newCursor === null) {
      // Terminal state — no more pages. Stop observing and hide the button.
      observer.unobserve(sentinel);
      loadMoreBtn.hidden = true;
      statusEl.textContent = 'No more items.';
    } else {
      statusEl.textContent = `${items.length} more items loaded.`;
    }
  } catch (err) {
    statusEl.textContent = 'Failed to load more items.';
    renderRetryAffordance();
  } finally {
    isLoading = false;
    loadMoreBtn.disabled = false;
  }
}

function appendItems(items) {
  const fragment = document.createDocumentFragment();
  for (const item of items) {
    const li = document.createElement('li');
    li.className = 'feed-item';
    // A fixed min-height / skeleton class here mitigates layout shift —
    // see "Cumulative Layout Shift" below.
    li.textContent = item.title;
    fragment.appendChild(li);
  }
  listEl.appendChild(fragment);
}

function renderRetryAffordance() {
  const retryBtn = document.createElement('button');
  retryBtn.textContent = 'Retry';
  retryBtn.addEventListener('click', () => {
    retryBtn.remove();
    loadNextPage();
  }, { once: true });
  statusEl.appendChild(retryBtn);
}
```

The `IntersectionObserver`, watching the sentinel:

```javascript
const observer = new IntersectionObserver(
  (entries) => {
    const [entry] = entries;
    if (entry.isIntersecting) {
      loadNextPage();
    }
  },
  {
    root: null,        // viewport
    rootMargin: '200px', // start fetching a bit before the sentinel is actually visible
    threshold: 0,
  }
);
observer.observe(sentinel);
```

The always-present button, wired to the identical function:

```javascript
loadMoreBtn.addEventListener('click', loadNextPage);
```

Cleanup — critical if this widget is ever torn down (SPA route change, modal close) while a fetch could still be pending:

```javascript
function destroy() {
  observer.disconnect(); // stops watching, and any queued callback for this observer won't fire
  loadMoreBtn.removeEventListener('click', loadNextPage);
  // If using AbortController for the fetch too, abort any in-flight request here
  // so a resolved promise doesn't try to touch a torn-down DOM.
}
```

> **Check yourself:** If `rootMargin: '200px'` and the user scrolls extremely fast — fast enough that several intersection callback firings could theoretically queue up before the first `loadNextPage()` call even sets `isLoading = true` — walk through why the *synchronous* nature of the flag-check-and-set (no `await` between them) is what actually prevents a double-fetch, not the flag's mere existence.

## Scroll Position Restoration

When navigating away from the feed (e.g., clicking into an item's detail view) and back, re-running the widget from scratch means: re-fetching every page the user had already loaded, and starting scrolled at the top — both bad. The fix is treating the feed's loaded state as cache, not transient UI state:

```javascript
// On navigate-away: snapshot what's loaded and where the user was scrolled.
function saveFeedState() {
  sessionStorage.setItem('feedState', JSON.stringify({
    itemsHTML: listEl.innerHTML,
    nextCursor,
    scrollY: window.scrollY,
  }));
}

// On re-entry: if a snapshot exists, restore it instead of fetching page 1.
function restoreFeedState() {
  const saved = sessionStorage.getItem('feedState');
  if (!saved) return false;

  const { itemsHTML, nextCursor: savedCursor, scrollY } = JSON.parse(saved);
  listEl.innerHTML = itemsHTML;
  nextCursor = savedCursor;
  requestAnimationFrame(() => window.scrollTo(0, scrollY)); // after paint, so layout exists
  return true;
}
```

This is a reasonable default, not a universal one — some feeds (a live "latest posts" stream) intentionally want a fresh fetch on re-entry rather than a stale cached snapshot, which is exactly the kind of product decision worth confirming rather than assuming.

## Cumulative Layout Shift

Items that pop in with no reserved space push everything below them down, which is jarring mid-scroll and, if it happens above the viewport's visible content while the user is reading, actively disorienting (the content under their eyes jumps). Two mitigations, usually combined: (1) render a fixed-height skeleton/placeholder row *before* the real content is measured, so the space is reserved the instant it's known a new item is coming, and (2) size images and media with explicit `width`/`height` attributes (or `aspect-ratio` in CSS) so the browser reserves their box before the image finishes downloading, rather than collapsing to zero height and then jumping open.

## Gotchas

**Checking `isLoading` after an `await` instead of before it.** If the guard is structured as "start the fetch, then check/set the flag inside the `.then()`," the race window is wide open — any observer firing between the fetch call and the flag being set slips through. The flag must be set synchronously, in the same tick as the check.

**Not unobserving/disconnecting on teardown.** If the widget's DOM is removed (SPA navigation) but `observer.disconnect()` is never called, the observer keeps a reference to the sentinel element, and depending on how the surrounding framework/router tears things down, the callback can still fire and try to touch a list element that's no longer meaningfully attached — at minimum a wasted fetch, at worst an error from code assuming the DOM is still there.

**Treating the "Load more" button as a hidden JS-disabled fallback instead of an equal citizen.** A button that's `display: none` unless `IntersectionObserver` is unsupported technically satisfies "there's a fallback," but doesn't satisfy "this isn't the only way to reach content" for a user who has JS and IntersectionObserver but navigates by keyboard/screen reader and finds scroll-triggered loading unreliable or disorienting — the button should be visible and functional at all times, observer or not.

**Offset-based pagination page-skew when the underlying data changes mid-scroll.** If page 2 is fetched as "items 20–39" by numeric offset, and between page 1 and page 2's fetch, 5 new items were inserted at the top of the feed by other users, offset-based pagination either skips 5 items or duplicates 5 items depending on sort direction. Cursor-based pagination (opaque token pointing at "everything after this specific item") avoids this class of bug entirely.

**No terminal state handling.** Forgetting to stop observing (and hide/disable the "Load more" button) once the API reports no more pages means either an infinite loop of empty-page fetches, or (if guarded against re-fetching an already-exhausted cursor) a sentinel that sits there forever doing nothing useful while still being watched.

## Follow-up Questions

**Q (High): Why is `IntersectionObserver` preferred over a `scroll` event listener for this pattern, concretely — not just "it's more modern"?**

Answer: A `scroll` event fires at high frequency during scrolling (potentially every frame), and a naive handler computing "is the sentinel near the viewport" on every firing does synchronous layout reads (`getBoundingClientRect()`) on the main thread repeatedly during the exact activity (scrolling) where main-thread jank is most visible to the user. Even a throttled/`rAF`-batched scroll handler still requires the browser to run JS periodically during scroll and still requires manually computing intersection geometry by hand. `IntersectionObserver` does the intersection computation in the browser's own implementation (commonly off the main thread's per-frame budget), and only invokes the callback when the observed element's intersection state actually changes — meaning near-zero JS execution during the vast majority of scroll frames where nothing relevant has changed.

The trap: describing `IntersectionObserver` as "more accessible" or "the new API" without being able to name the actual mechanical difference (main-thread cost per scroll frame, manual geometry math vs. native computation) — interviewers use this question to check whether "use IntersectionObserver" is a memorized best practice or an understood one.

---

**Q (High): Walk through the exact race condition that causes a duplicate fetch of the same page, and why a `Set` of "already-fetched page numbers" doesn't fix it.**

Answer: The race isn't "did we already fetch page N and forget," it's "a second fetch *for page N* starts while the first fetch *for page N* is still in flight." This happens because the sentinel doesn't move until the newly-fetched items are appended to the DOM — so between the first intersection firing and the fetch actually completing, the sentinel is still in (or re-enters) the trigger zone, and if nothing prevents it, the observer callback fires again and calls `loadNextPage()` a second time, still asking for the same next cursor, because `nextCursor` hasn't been updated yet either (that only happens after the *first* fetch resolves). A `Set` tracking "pages already fully fetched" doesn't help because neither fetch has completed yet — from the `Set`'s perspective, page N was never marked as fetched, so both calls pass the check. The actual fix is a synchronous `isLoading` boolean, set to `true` the instant the decision to fetch is made (before the `await`), so the *second* call to `loadNextPage()` sees `isLoading === true` and returns immediately, regardless of what cursor/page state has or hasn't updated yet.

The trap: proposing "just check if the cursor changed" as the guard — the cursor is exactly the piece of state that hasn't changed yet during the race window, so any guard based on cursor/page identity fails for the same reason the `Set` approach fails; the guard has to be about *fetch-in-progress*, not *page-already-fetched*.

---

**Q (High): Why does pure infinite scroll create an accessibility problem, and what's the concrete fix — not just "add ARIA"?**

Answer: Several distinct issues, not one: (1) keyboard-only users (not using a mouse wheel) navigate via Tab/Page Down, and content that only loads on scroll-into-view via mouse-driven scrolling may not reliably trigger for keyboard-driven scrolling depending on implementation, and even when it does, there's no discoverable, focusable control representing "load more" for a keyboard user to intentionally invoke; (2) screen reader users navigating by heading/landmark (a very common navigation mode) expect to be able to reach a page's footer — a feed that keeps growing every time the viewport nears the bottom means the footer perpetually recedes and can become effectively unreachable; (3) content injected mid-stream without an announcement is disorienting for a screen reader user whose reading position is now surrounded by new, unannounced siblings. The concrete fix is providing a real, always-rendered, keyboard-focusable "Load more" button that performs the identical fetch, paired with an `aria-live="polite"` status region announcing what happened ("8 more items loaded") — not a visually-hidden fallback that only appears in a no-JS/no-IntersectionObserver-support branch.

The trap: answering with "add `aria-live` to the list" alone — a live region addresses the *announcement* problem but does nothing for the keyboard-trap or footer-unreachability problems, which require an actual alternative *interaction* mechanism (the button), not just better announcements of the existing one.

---

**Q (Medium): How would you restore scroll position and previously-loaded items when a user navigates back to this feed after clicking into a detail view?**

Answer: Treat the feed's already-fetched items and the user's scroll offset as cacheable state keyed to that view/route, persisted somewhere that survives the navigation (in-memory cache in a router-level store for an SPA, or `sessionStorage` for a full concept that needs to survive a tab-level scope) — on re-entry, check for a saved snapshot and, if present, restore the rendered items and scroll position instead of re-fetching from the first page. Restoring scroll position specifically needs to happen after the restored items have been laid out (e.g., inside a `requestAnimationFrame` after re-render), since `scrollTo()` before layout exists has nothing to scroll to yet.

The trap: assuming this is "free" simply by not destroying the DOM — in most real apps (client-side routers, tab/modal-based navigation) the feed's DOM *is* torn down and rebuilt on re-entry, so scroll/data restoration has to be deliberately implemented as a save/restore cache, not assumed to happen automatically via DOM persistence.

---

**Q (Medium): What's the CLS (Cumulative Layout Shift) risk in an infinite-loading list, and how do you mitigate it without knowing item content in advance?**

Answer: Every batch of newly-appended items that doesn't have pre-reserved space pushes subsequent content (and, if inserted above the current viewport for any reason, the currently-visible content itself) downward, which both hurts the CLS Core Web Vital and feels jarring to the user. Since new items' exact content/height often isn't known before the fetch resolves, the standard mitigation is rendering a fixed-size skeleton placeholder the moment loading starts (so space is reserved immediately, before content exists) and giving images/media explicit `width`/`height` or `aspect-ratio` so their box is reserved before the asset finishes downloading — the shift then happens once, predictably, at skeleton-render time (which is far less jarring than repeated, unpredictable shifts as each asset trickles in).

The trap: trying to eliminate all layout shift entirely — for genuinely variable-height, variable-content items, some shift when the skeleton resolves into real content is often unavoidable without knowing content ahead of time; the realistic goal is minimizing shift magnitude and frequency, not achieving zero.

---

**Q (Medium): Why is cursor-based pagination generally preferred over offset/page-number pagination for a feed that other actions can mutate concurrently?**

Answer: Offset-based pagination (`?page=2&pageSize=20`) identifies "page 2" purely by position in the current full ordering — if items are inserted or removed anywhere before that position between when page 1 and page 2 are fetched (very plausible in a live feed with other users posting), the position-based window shifts, causing either skipped items (something inserted pushes an item that should've been on page 2 back onto page 1, which has already been fetched) or duplicated items (a removal pulls an item from page 2 forward into what's now page 1's range, so it appears in both fetched sets). Cursor-based pagination anchors each page to an opaque pointer relative to a specific item ("everything after this item's id/timestamp"), which is stable regardless of insertions/removals elsewhere in the list, since it doesn't depend on absolute position.

The trap: dismissing this as a micro-edge-case — for a low-traffic internal tool it might be, but for anything resembling a live social feed or frequently-updated listing, this is a routinely-hit real bug, not a hypothetical.

---

**Q (Low): How would you adapt this pattern for a bidirectional feed (loading older items when scrolling down, but also newer items when scrolling up past the top, like a chat log)?**

Answer: Add a second sentinel/observer at the *top* of the list (in addition to the existing bottom one), with its own cursor tracking "the oldest loaded item's boundary going the other direction," and — critically — when prepending items above the current scroll position, adjust `scrollTop` by the newly-inserted content's height immediately after insertion so the user's visual position doesn't jump (the classic "load older messages" problem in chat UIs). The `isLoading` guard, terminal-state handling, and accessible-button-equivalent pattern all apply symmetrically to both directions.

The trap: forgetting the scroll-position compensation on prepend — appending at the bottom doesn't disturb the user's current scroll position, but *prepending* at the top does (everything the user is currently looking at gets pushed down by the height of the newly inserted content) unless the scroll offset is explicitly corrected in the same synchronous operation as the DOM insertion.

---

**Q (Low): What changes about this pattern in a framework like React, where you can't just mutate the DOM directly in event handlers?**

Answer: The mechanics stay the same conceptually — `IntersectionObserver` created in a `useEffect` observing a ref'd sentinel element, `isLoading` and `nextCursor` as component state (or a ref for the loading flag specifically, since it needs to be checked/set synchronously without waiting for a re-render to reflect it, which `useState` alone doesn't guarantee within the same synchronous tick), and cleanup (`observer.disconnect()`) returned from the `useEffect`. The main adaptation is that the loading guard often needs to be a `useRef` rather than `useState` precisely because state updates in React aren't synchronous/immediately-read-back — a `let isLoading` closure variable or a ref avoids a subtle version of the same race the plain-JS version guards against.

The trap: using `useState` for the `isLoading` guard and assuming `setIsLoading(true)` is immediately visible to the very next intersection callback firing — React batches and defers state updates, so relying on state (rather than a ref or module-level variable) for a same-tick race guard can reintroduce the exact duplicate-fetch bug this pattern exists to prevent.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement `IntersectionObserver`-based infinite loading with a sentinel element from memory
- [ ] Can explain exactly why the duplicate-fetch race happens and why the fix must be a synchronous in-flight flag, not a "seen pages" set
- [ ] Can explain the three distinct accessibility failures of pure infinite scroll and why a live-region announcement alone doesn't fix all of them
- [ ] Can implement a "Load more" button that shares the exact same fetch function as the observer, not a separate degraded path
- [ ] Can explain why cursor-based pagination is more robust than offset-based pagination under concurrent mutation
- [ ] Can name the CLS risk from unsized incoming content and at least one concrete mitigation

---
*Next: Virtualized List (Windowing) From Scratch — for when infinite scroll alone isn't enough because the DOM node count itself becomes the bottleneck.*
