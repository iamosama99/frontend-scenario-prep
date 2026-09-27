# Autocomplete / Typeahead — Debounced & Cancellable

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Debounce keystrokes | Reset a timer on every `input` event; only fetch after quiet period | Stops firing a network request per keystroke |
| Cancel in-flight requests | `AbortController` per request, `.abort()` the previous one before starting a new one | Frees the network/server from doing work whose result is already useless |
| Stale-response race | A monotonically increasing request id/token; ignore any response whose id isn't the latest issued | Debounce + abort alone don't guarantee response *order* — an older request can still resolve after a newer one |
| Keyboard navigation | Track a `activeIndex` in JS state, move it on Arrow keys, don't let `input` re-fetch on navigation keys | Mouse-free selection is a baseline requirement, not a nice-to-have |
| ARIA combobox pattern | `role="combobox"` + `aria-expanded` + `aria-controls` + `aria-activedescendant` on the input, `role="listbox"`/`role="option"` on results | Screen reader users get the same "here's what's highlighted" signal sighted users get from CSS |

## The Scenario

"Build me a search-as-you-type box — type a city name, get a dropdown of matching suggestions from an API. I want it to feel fast and not hammer our backend. Also, use your keyboard to move up and down the list and hit Enter to pick one. Make sure it's usable with a screen reader too."

## Clarifying Questions

- **Is there a minimum number of characters before we fetch?** Almost always yes — firing a search API call on a single keystroke (especially the first character) returns a huge, useless result set and wastes a round trip. I'd default to a 2- or 3-character minimum and confirm the number with product/design, since it's a UX decision as much as a technical one.
- **What's the expected debounce delay, and is it fixed or should it adapt to typing speed?** A fixed 200–300ms delay is the standard baseline. This matters because too short defeats the purpose (still fires on every keystroke of a fast typist) and too long feels laggy — I'd rather state the number and be corrected than silently guess.
- **Can the API return results out of order relative to when requests were sent?** This is the single most important question in the whole scenario. If the answer is "assume no" (unrealistic on a real network) the design is simpler; if "yes, assume nothing about response order" (the honest answer), then debounce and `AbortController` alone are *not* sufficient — I need a request-id guard too. I'd state this explicitly rather than let it surface as a "found in code review" bug.
- **Should the widget be a generic, reusable component, or one-off wired to this specific city-search endpoint?** Affects whether I parameterize the fetcher, the render-item function, and the minimum-query-length as props/config, versus hardcoding them — I'd lean generic since typeahead is the kind of pattern that gets reused across a codebase.
- **Does it need full ARIA combobox compliance, or is "looks right visually" good enough?** The prompt says "usable with a screen reader," which means yes — I'd implement the WAI-ARIA combobox pattern rather than a visually-styled div-soup that happens to look like a dropdown.

## Approach & Trade-offs

There are three layers of the same underlying problem — "don't act on stale input" — and it's worth separating them explicitly, because conflating them is the most common way candidates under-solve this scenario:

1. **Reducing request volume** — debounce. Don't fire a fetch on every keystroke; wait for a quiet period.
2. **Cancelling wasted work** — `AbortController`. Even with debouncing, a slow request for query "Lon" can still be in flight when the user has already typed "London" and a new request has fired. Aborting the previous request's underlying `fetch` call frees the browser connection and lets the server (if it respects the abort signal) stop doing wasted work.
3. **Guarding against out-of-order responses** — a request id/token. This is the layer most candidates miss, and it's independent of the first two. Even *with* debounce and `AbortController`, there's a scenario where the old request has already left the browser and can't be recalled by an abort signal on some transports, or the abort races with the response arriving — the network is fundamentally not ordered. The only airtight fix is: stamp every outgoing request with an incrementing id, and when a response comes back, check "is this response's id still the most recent one I issued?" If not, discard it, no matter what `AbortController` did or didn't manage to cancel.

I chose to implement all three rather than relying on any one of them, because each solves a different failure mode:
- Debounce alone still lets two requests race (fast typer skips the debounce window twice in a row, in-flight order isn't guaranteed).
- `AbortController` alone (no debounce) still fires a request per keystroke — abort reduces *waste* but not *volume*.
- The id-guard alone (no debounce/abort) is correct but wasteful — you'd fire and complete every request, just ignore most of the responses; that's needless load on the backend.

For keyboard navigation, the key design decision is that **Arrow/Enter/Escape are handled entirely client-side against already-fetched results** — they never trigger a new debounce cycle or a new fetch. This is worth calling out explicitly because a naive implementation that treats "any keydown" as "user is still typing, restart the debounce timer" breaks arrow-key navigation (the dropdown would refetch or flicker every time you press ArrowDown). The fix is checking `event.key` and only routing character-producing keys (and a few others like Backspace) into the debounce/fetch path — navigation keys go through a separate, synchronous handler that only touches local UI state.

For the ARIA layer, I'm following the WAI-ARIA APG combobox pattern (listbox popup variant) rather than inventing my own — `aria-activedescendant` specifically exists so that keyboard focus can stay on the `<input>` (required, since moving DOM focus to each list item would kick the user out of typing mode) while still telling assistive tech which option is "virtually" focused.

## Solution

Start with markup — the ARIA wiring has to exist from the beginning, not bolted on after:

```html
<div class="typeahead">
  <input
    type="text"
    role="combobox"
    aria-expanded="false"
    aria-controls="city-listbox"
    aria-autocomplete="list"
    aria-activedescendant=""
    autocomplete="off"
    id="city-input"
  />
  <ul role="listbox" id="city-listbox" hidden></ul>
</div>
```

State the widget needs to track, all in closure/module scope (no framework state manager here — plain JS):

```javascript
const input = document.getElementById('city-input');
const listEl = document.getElementById('city-listbox');

let debounceTimer = null;
let currentController = null; // AbortController for the in-flight request
let requestId = 0;            // monotonically increasing — the stale-response guard
let results = [];
let activeIndex = -1;         // -1 = nothing highlighted
const MIN_QUERY_LENGTH = 2;
const DEBOUNCE_MS = 250;
```

The `input` handler — debounce, then fetch with both an abort signal and a captured request id:

```javascript
input.addEventListener('input', (e) => {
  const query = e.target.value.trim();

  clearTimeout(debounceTimer);

  if (query.length < MIN_QUERY_LENGTH) {
    closeList();
    return;
  }

  debounceTimer = setTimeout(() => runSearch(query), DEBOUNCE_MS);
});

async function runSearch(query) {
  // Cancel whatever was previously in flight — its result is now irrelevant.
  if (currentController) currentController.abort();
  currentController = new AbortController();

  const thisRequestId = ++requestId; // stamp this request before awaiting anything
  setLoadingState();

  try {
    const res = await fetch(`/api/cities?q=${encodeURIComponent(query)}`, {
      signal: currentController.signal,
    });
    const data = await res.json();

    // THE critical guard: even if abort() didn't manage to cancel this in
    // time, or the transport doesn't honor AbortSignal, a response for an
    // older query must never overwrite a newer one's results.
    if (thisRequestId !== requestId) return;

    renderResults(data, query);
  } catch (err) {
    if (err.name === 'AbortError') return; // expected — a newer request superseded this one
    if (thisRequestId !== requestId) return; // stale error, same guard applies to failures too
    renderError();
  }
}
```

Rendering with substring highlighting and full ARIA state sync:

```javascript
function renderResults(items, query) {
  results = items;
  activeIndex = -1;

  if (items.length === 0) {
    listEl.innerHTML = '<li role="option" aria-disabled="true">No matches</li>';
  } else {
    listEl.innerHTML = items
      .map((item, i) => `
        <li role="option" id="opt-${i}" aria-selected="false">
          ${highlightMatch(item.name, query)}
        </li>
      `)
      .join('');
  }

  listEl.hidden = false;
  input.setAttribute('aria-expanded', 'true');
}

function highlightMatch(text, query) {
  const idx = text.toLowerCase().indexOf(query.toLowerCase());
  if (idx === -1) return escapeHtml(text);
  return (
    escapeHtml(text.slice(0, idx)) +
    `<mark>${escapeHtml(text.slice(idx, idx + query.length))}</mark>` +
    escapeHtml(text.slice(idx + query.length))
  );
}

function escapeHtml(str) {
  const div = document.createElement('div');
  div.textContent = str;
  return div.innerHTML;
}
```

Keyboard navigation — deliberately routed through a *separate* handler from the debounce path, and only reacting to non-character keys:

```javascript
input.addEventListener('keydown', (e) => {
  if (listEl.hidden) return;

  switch (e.key) {
    case 'ArrowDown':
      e.preventDefault(); // don't move the text cursor
      moveActive(1);
      break;
    case 'ArrowUp':
      e.preventDefault();
      moveActive(-1);
      break;
    case 'Enter':
      if (activeIndex >= 0) {
        e.preventDefault();
        selectResult(results[activeIndex]);
      }
      break;
    case 'Escape':
      closeList();
      break;
    // Any other key (letters, Backspace, etc.) falls through untouched —
    // the `input` event listener above handles those via debounce.
  }
});

function moveActive(delta) {
  const optionEls = listEl.querySelectorAll('[role="option"]:not([aria-disabled])');
  if (optionEls.length === 0) return;

  activeIndex = (activeIndex + delta + optionEls.length) % optionEls.length; // wraps both ends

  optionEls.forEach((el, i) => {
    el.setAttribute('aria-selected', String(i === activeIndex));
    el.classList.toggle('is-active', i === activeIndex);
  });

  input.setAttribute('aria-activedescendant', optionEls[activeIndex].id);
  optionEls[activeIndex].scrollIntoView({ block: 'nearest' });
}

function selectResult(item) {
  input.value = item.name;
  closeList();
  // Fire a custom event / callback here for the consumer to act on the pick.
}

function closeList() {
  listEl.hidden = true;
  listEl.innerHTML = '';
  input.setAttribute('aria-expanded', 'false');
  input.setAttribute('aria-activedescendant', '');
  activeIndex = -1;
  clearTimeout(debounceTimer);
  if (currentController) currentController.abort();
}
```

> **Check yourself:** If a user types "Lo", pauses long enough for a request to fire, then quickly types "ndon" before that request resolves — trace exactly which mechanisms (debounce, abort, request-id) touch this sequence, and what would happen if the request-id guard were the *only* one implemented.

## Loading, Empty, and Error States

- **Loading**: show a spinner/skeleton *inside* the listbox region (not replacing it) so `aria-expanded`/`aria-controls` stay wired to a stable region; announce via a visually-hidden `aria-live="polite"` region ("Loading results…") since a screen reader won't otherwise notice a spinner appearing.
- **Empty**: render a single non-interactive `role="option" aria-disabled="true"` row ("No matches for '…'") rather than just hiding the list — an empty dropdown with no feedback reads as broken.
- **Error**: distinguish "no results" from "the request failed" — a network failure should say so and ideally offer retry, not silently look like zero matches.

## Gotchas

**Debounce without an id-guard "usually" works, which is worse than always failing.** The race only manifests when an *older* request happens to resolve *after* a newer one — on a fast, stable network in a demo, this basically never happens, so a candidate can ship code that looks correct and only breaks in production under variable network conditions (mobile, throttled connections, server-side load spikes). This is exactly the kind of bug interviewers plant on purpose.

**Treating every `keydown` as "restart the debounce timer."** If the input handler and the debounce logic aren't keyed specifically off events that change the *text value*, pressing ArrowDown fires a redundant fetch and/or visually flickers the list closed-then-reopened.

**Forgetting to abort on close/blur/selection, not just on the next keystroke.** If the user picks a result or dismisses the dropdown while a request is in flight, that request should be aborted too — otherwise it can still resolve and (without the id-guard) repaint a dropdown the user already closed.

**`aria-activedescendant` alone isn't enough — focus must stay on the `<input>`.** A common mistake is moving actual DOM focus to the `<li>` elements on ArrowDown, which breaks continued typing and violates the combobox pattern; the input keeps focus the entire time, and `aria-activedescendant` is the *only* signal for which option is "selected."

**Not handling `AbortError` as an expected, silent case.** If the `catch` block treats every rejected fetch (including ones the code itself aborted) as a real error, the UI briefly flashes an error state on every keystroke that supersedes a prior request — an obviously broken user experience that's easy to miss if you only test slow typing.

## Follow-up Questions

**Q (High): Walk through the exact race condition where an older, slower request resolves after a newer, faster one — and why `AbortController` alone doesn't fully solve it.**

Answer: Say the user types "Lon" (request A fires) then quickly types "London" (request B fires, and the code calls `A.abort()` first). `AbortController.abort()` signals cancellation, but it doesn't guarantee the browser or server stops processing instantly, nor does every layer of a request pipeline necessarily honor the signal promptly (a slow reverse proxy, a fetch polyfill, or a server that has already fully computed its response before the abort reaches it). If A's response arrives and is processed *after* B's — despite the abort call — and there's no additional guard, A's (stale, "Lon"-matching) results overwrite B's (current, "London"-matching) results on screen, showing the user results for text they've since changed. The fix layered on top of abort is a monotonically increasing request id: stamp each request before it's sent, and when any response arrives, compare its stamped id against the *current* latest-issued id — if they don't match, discard the response outright, regardless of what the abort signal did.

The trap: saying "`AbortController.abort()` guarantees the aborted request's promise never resolves with data" — it typically rejects with an `AbortError`, which is the *common* case, but relying on that as an absolute guarantee across all fetch implementations, polyfills, and network stacks is exactly the assumption that turns into an intermittent production bug. The id-guard is what makes correctness independent of that assumption.

---

**Q (High): Why must keyboard navigation (ArrowUp/ArrowDown) not restart or interact with the debounce timer at all?**

Answer: Debounce exists to rate-limit *fetches triggered by changing the query text*. Arrow-key navigation doesn't change the query text — it changes which already-fetched result is highlighted, which is a purely local, synchronous UI state update. If the `keydown` handler doesn't distinguish navigation keys from character-input, either of two bugs appears: (1) pressing ArrowDown re-triggers the debounce/fetch cycle, causing a flicker or duplicate network request for a query the user didn't actually change, or (2) worse, the results list gets wiped and rebuilt mid-navigation, breaking `activeIndex` and any in-progress selection. The fix is routing character-producing input through the `input` event (which naturally only fires when the value changes) and handling Arrow/Enter/Escape in a separate `keydown` listener that only touches `activeIndex` and ARIA attributes, never the fetch pipeline.

The trap: implementing navigation inside the same handler that triggers fetches and gating it with an `if` that "happens to" work for the keys tested in the demo, but doesn't generalize — e.g., forgetting that Backspace, Delete, and Tab all fire `keydown` too and need to be excluded from the navigation branch without also being excluded from normal typing.

---

**Q (High): How would you implement the ARIA combobox pattern correctly here, and why does focus stay on the `<input>` the entire time instead of moving to the highlighted option?**

Answer: The `<input>` carries `role="combobox"`, `aria-expanded` (toggled true/false as the popup opens/closes), `aria-controls` (pointing at the listbox's id), and `aria-activedescendant` (updated to the `id` of the currently-highlighted `<li role="option">` on every Arrow key press). The popup itself is `role="listbox"` containing `role="option"` children. Focus deliberately never leaves the `<input>` — if ArrowDown moved actual DOM focus to an `<li>`, the user would no longer be typing-focused, breaking the ability to keep filtering by typing while browsing results, and it would also require re-implementing character input handling on a non-input element. `aria-activedescendant` exists specifically to solve this: it lets a screen reader announce "option 3 of 8, London" as if focus moved there, while real DOM/keyboard focus stays put on the input.

The trap: implementing the pattern with `tabindex` on each `<li>` and actually moving `.focus()` between them — this is the "roving tabindex" pattern, which is correct for a *different* widget (tabs, toolbars) but wrong here specifically because it breaks continuous typing, which is the entire premise of a combobox.

---

**Q (Medium): What's the right minimum query length, and why does firing a request on a 1-character query cause real problems beyond just "extra network traffic"?**

Answer: Beyond wasted requests, a 1-character query against most backends returns a huge, often-truncated result set that's useless for the user to scan and expensive for the backend to compute (large index scans, more data serialized and transferred). It also increases the *odds* of the stale-response race, since more requests are in flight over the same short typing burst. A minimum of 2–3 characters is the common default, but the real answer is "ask the API/product owner" — it depends on the dataset (a 2-letter country-code search is meaningfully different from a 2-letter freeform text search).

The trap: treating this as a purely client-side UX nicety ("stops the dropdown from looking cluttered") rather than recognizing it as also a load-shedding mechanism for the backend — the minimum length is often driven by the API team, not the frontend team's preference.

---

**Q (Medium): How would you debounce the *loading* indicator itself, so a fast response doesn't cause a 1-frame flash of a spinner?**

Answer: Introduce a short delay (commonly ~150–200ms) before showing the loading state, and cancel that delay if the response arrives before it elapses — so a request that resolves quickly never shows a spinner at all, while a genuinely slow request still gives the user feedback. This is a distinct timer from the input debounce (which delays the *request*), applied to the *loading UI* specifically, and it's a common polish detail interviewers notice by its absence rather than ask about directly.

The trap: conflating this with the input debounce and thinking one timer serves both purposes — they solve different problems (rate-limiting requests vs. avoiding UI flicker) and typically need separate timers with separate durations.

---

**Q (Medium): How would you make this component reusable across different data sources (cities, users, products) rather than hardcoded to one API?**

Answer: Parameterize the fetcher function (`(query, signal) => Promise<Item[]>`), the render/label function (`(item) => string` for both the input value on selection and the display text), and the minimum length/debounce delay as configuration passed into a constructor or factory function, keeping the debounce/abort/request-id machinery generic and untouched by what's actually being searched. The DOM structure and ARIA wiring stay identical regardless of data source — only the fetch call and the item-to-text mapping change.

The trap: hardcoding the API endpoint string or the shape of the response object (`item.name`) directly inside the debounce/fetch logic, which then requires copy-pasting the entire widget for the next use case instead of reusing it with different config.

---

**Q (Low): How does this pattern change if the matching should happen client-side against an already-fetched, complete dataset (e.g., a small fixed list of 50 countries) instead of hitting an API per keystroke?**

Answer: Drop the debounce/abort/request-id machinery entirely (there's no network race to guard against) and instead filter the in-memory array synchronously on every `input` event — the remaining logic (keyboard navigation, ARIA wiring, substring highlighting) is identical. The main new concern becomes performance of the filter itself on very large in-memory lists (thousands of entries), which might warrant debouncing the *render* (not a network request) if filtering is expensive, or a more efficient search structure (e.g., a prefix trie) if the dataset is large enough to matter.

The trap: keeping the debounce timer "just in case" even though there's no network request to rate-limit — for a synchronous, in-memory filter, debouncing usually just adds a perceptible ~250ms of unnecessary lag on every keystroke, with no benefit.

---

**Q (Low): What accessibility gap remains even with a fully correct ARIA combobox implementation, and how do you address it?**

Answer: Screen reader users get no feedback when the *result count* changes after a fetch unless it's explicitly announced — `aria-expanded`/`aria-activedescendant` tell them what's highlighted, but not "8 results found" or "no results" as a state change. Pairing the widget with a visually-hidden `aria-live="polite"` region that's updated with a short status message ("8 results available", "No results", "Loading results") after each fetch closes this gap without any visual change to sighted users.

The trap: assuming the ARIA combobox role attributes alone are a complete accessibility solution — the *structural* roles communicate what an option is and which is active, but live regions are a separate, additive mechanism needed for announcing asynchronous state changes as they happen.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement debounced input handling that only fetches after a quiet period and a minimum query length
- [ ] Can explain why debounce, `AbortController`, and a request-id guard each solve a *different* failure mode and why all three are needed together
- [ ] Can implement the ARIA combobox pattern (`role="combobox"`, `aria-expanded`, `aria-activedescendant`, `role="listbox"`/`option`) from memory
- [ ] Can explain why focus must stay on the input during keyboard navigation instead of moving to each option
- [ ] Can implement substring highlighting safely (without introducing an HTML injection bug via unescaped user input)
- [ ] Can describe loading/empty/error states and why "no feedback" is a common but real bug

---
*Next: Infinite Scroll List — the next data-fetching-driven UI pattern, this time triggered by scroll position instead of keystrokes.*
