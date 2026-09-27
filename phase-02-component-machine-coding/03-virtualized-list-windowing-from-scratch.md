# Virtualized List (Windowing) From Scratch

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Render only visible items | Compute a visible index range from `scrollTop` + container height; render just that slice | 10,000 real DOM nodes is the actual bottleneck (layout, paint, memory) — windowing keeps rendered nodes proportional to viewport, not data size |
| Preserve real scrollbar behavior | A tall "phantom" spacer sized to `totalItemCount * rowHeight`; render window is absolutely positioned/translated inside it | The scrollbar has to behave as if all items exist, even though only a handful are in the DOM |
| Overscan | Render a few extra items above/below the visible range | Fast scrolling would otherwise show blank gaps for the one frame before new items render |
| Scroll listener perf | Throttle position reads/writes via `requestAnimationFrame` | Scroll fires at high frequency; recomputing and re-rendering on every raw event risks janking the very thing you're optimizing |
| Variable row height | Measure rendered rows, cache real heights, estimate unmeasured ones, correct scroll position when an estimate is wrong | Fixed-height math (`index * rowHeight`) breaks completely once rows can differ in height |
| Accessibility trade-off | `aria-setsize`/`aria-posinset` on rendered rows; screen readers still can't reach non-rendered items | Virtualization is a genuine, unavoidable trade-off against assistive tech, not something ARIA fully fixes |

## The Scenario

"We have a list with tens of thousands of rows — think a spreadsheet-like data grid or a huge log viewer — and rendering them all is killing performance. I want you to build a virtualized list from scratch, no libraries. Start with fixed-height rows, then let's talk about what changes if rows have variable height."

## Clarifying Questions

- **Is the container a fixed, known height, or does it need to size itself to available space?** Determines whether the visible-range math can assume a static viewport height or needs to react to resize (`ResizeObserver`) as well as scroll.
- **Fixed or variable row height — and if variable, is it known ahead of time (e.g., from data) or only knowable after rendering (text that wraps differently)?** This is the single biggest fork in the design. Fixed height makes index-range math a simple division; unknown-until-rendered variable height requires a measure-cache-estimate-correct loop that's an order of magnitude more complex, and I'd want to confirm which case actually matters before investing in the harder one.
- **Does the list need to support scrolling to an arbitrary index programmatically (e.g., "jump to row 5000")?** If yes, that needs to work even for rows that have never been rendered/measured yet, which is a real constraint on the variable-height design (you can't scroll precisely to an unmeasured row without first estimating its position).
- **What interaction does this list need — is it read-only display, or does it need focus/selection/keyboard navigation on rows?** Virtualization interacts directly with focus management: a focused row that scrolls out of the rendered window and gets unmounted loses focus entirely, which needs an explicit answer (keep it in the DOM? move focus elsewhere? re-focus by index when it re-enters?).
- **How important is screen reader usability for this specific list, given that virtualization is a real accessibility trade-off?** I'd rather surface this upfront than let it be discovered later — a screen reader user fundamentally cannot "arrow down through" items that don't exist in the DOM yet, and that's not something ARIA attributes alone resolve.

## Approach & Trade-offs

**The core mechanism, at a high level:** instead of rendering all N items, render only the small subset whose vertical position falls inside (or near) the current viewport, and reposition that small subset as the user scrolls — while making the *scrollbar* and *scroll range* behave exactly as if all N items were really there. This requires two coordinated pieces: (1) a full-height "phantom"/spacer element whose height equals what the total content height would be if everything were rendered (so the browser's native scrollbar and scroll range are correct), and (2) an absolutely-positioned (or `transform: translateY(...)`-shifted) inner container holding only the currently-visible rows, positioned at the correct offset within that phantom space.

I chose `transform: translateY()` over adjusting `top`/`margin-top` for positioning the rendered window, because `transform` is compositor-friendly (doesn't trigger layout/reflow the way changing `top` on a non-`position: absolute` element would, and is cheaper than repeatedly reflowing `margin-top`) — this matters specifically because this positioning update happens on every scroll-driven re-render, so its cost compounds.

**Computing the visible range (fixed height case):** given `scrollTop`, `containerHeight`, and a constant `rowHeight`, the first visible index is `Math.floor(scrollTop / rowHeight)` and the number of visible rows is `Math.ceil(containerHeight / rowHeight)` — this is a pure, cheap calculation, no DOM measurement needed, which is exactly why the fixed-height case is so much simpler than the variable-height case (where "what's the height of row N" isn't answerable without having rendered row N at least once).

**Overscan (rendering a buffer beyond the strictly-visible range):** without it, fast scrolling exposes a real gap — the browser paints the next frame before JS has finished computing and rendering the newly-visible rows, showing a blank flash at the leading edge of scroll direction. Rendering, say, 3–5 extra rows above and below the computed visible range absorbs that timing gap at the cost of a few extra DOM nodes — a trade-off overwhelmingly worth taking, since the whole point of virtualization is capping node count at "viewport-sized," and a small constant overscan doesn't meaningfully change that order of magnitude.

**Throttling the scroll handler via `requestAnimationFrame` rather than firing full recompute-and-render logic on every raw `scroll` event:** `scroll` fires far more often than the display can usefully paint distinct states, so batching "read `scrollTop`, compute range, re-render" into an rAF callback (only queuing one per frame, ignoring intermediate scroll events within that frame) avoids doing multiple full recomputations for scroll deltas the user will never actually perceive as distinct frames.

**Fixed height vs. variable/dynamic height — the real trade-off to narrate.** Fixed-height windowing is a solved, cheap problem: array-index math, no measurement, no correction. Variable-height windowing is fundamentally harder because you can't know a row's real height until it's been rendered and measured (unless the data itself carries a reliable height, which is rare for anything involving text/wrapping) — so the design has to *estimate* unmeasured rows' heights (using an average of measured rows, or a supplied default), track a growing cache of *actual* measured heights as rows get rendered for the first time, and — the genuinely tricky part — *correct* the scroll position/rendered range when a previously-estimated row turns out, once measured, to have a meaningfully different real height than assumed (otherwise the phantom spacer's total height, and therefore the scrollbar and computed offsets for every row after it, is subtly wrong until corrected). I'd build the fixed-height version first, confirm it's actually correct, and only then layer in the variable-height complexity — building both at once from scratch invites conflating two different classes of bugs.

## Solution — Fixed Row Height

Markup: an outer scroll container with a fixed height, a tall spacer inside it, and a positioned inner window inside the spacer.

```html
<div id="scroll-container" style="height: 500px; overflow-y: auto; position: relative;">
  <div id="phantom" style="position: relative;">
    <div id="window" style="position: absolute; top: 0; left: 0; right: 0;"></div>
  </div>
</div>
```

Config and state:

```javascript
const ROW_HEIGHT = 40;
const OVERSCAN = 4;
const items = generateItems(50000); // the full dataset — never all rendered at once

const container = document.getElementById('scroll-container');
const phantom = document.getElementById('phantom');
const windowEl = document.getElementById('window');

phantom.style.height = `${items.length * ROW_HEIGHT}px`; // makes the scrollbar behave correctly
```

Computing the visible range and rendering it, throttled via `requestAnimationFrame`:

```javascript
let rafId = null;

function onScroll() {
  if (rafId !== null) return; // a render is already queued for this frame
  rafId = requestAnimationFrame(() => {
    renderWindow();
    rafId = null;
  });
}

function renderWindow() {
  const scrollTop = container.scrollTop;
  const containerHeight = container.clientHeight;

  const firstVisible = Math.floor(scrollTop / ROW_HEIGHT);
  const visibleCount = Math.ceil(containerHeight / ROW_HEIGHT);

  const startIndex = Math.max(0, firstVisible - OVERSCAN);
  const endIndex = Math.min(items.length, firstVisible + visibleCount + OVERSCAN);

  // Position the rendered slice at its true offset within the phantom's full height.
  windowEl.style.transform = `translateY(${startIndex * ROW_HEIGHT}px)`;

  const fragment = document.createDocumentFragment();
  for (let i = startIndex; i < endIndex; i++) {
    const row = document.createElement('div');
    row.style.height = `${ROW_HEIGHT}px`;
    row.setAttribute('role', 'listitem');
    row.setAttribute('aria-setsize', String(items.length)); // total logical count
    row.setAttribute('aria-posinset', String(i + 1));        // 1-based position
    row.textContent = items[i].label;
    fragment.appendChild(row);
  }
  windowEl.replaceChildren(fragment); // clear + repopulate in one operation
}

container.addEventListener('scroll', onScroll);
renderWindow(); // initial paint
```

> **Check yourself:** Why does the phantom element need an explicit `height: ${items.length * ROW_HEIGHT}px`, rather than just letting it size to its (small) rendered content — what specifically would break about scrolling if it didn't have that explicit height?

## Solution — Variable/Dynamic Row Height

The fixed-height math (`index * ROW_HEIGHT`) no longer works — offsets have to be computed from a running total of actual (or estimated) heights. This needs three pieces of state beyond the fixed-height version: a `measuredHeights` cache (index → real height, once known), an `estimatedHeight` default (used for anything not yet measured), and offset computation that walks the cache instead of multiplying.

```javascript
const measuredHeights = new Map(); // index -> measured height (px)
const ESTIMATED_HEIGHT = 40;       // best-guess default for unmeasured rows

function getHeight(index) {
  return measuredHeights.get(index) ?? ESTIMATED_HEIGHT;
}

// Total height up to (not including) `index` — needed to position it correctly.
function getOffsetForIndex(index) {
  let offset = 0;
  for (let i = 0; i < index; i++) offset += getHeight(i);
  return offset;
}
```

That linear walk is fine for a first pass but O(n) per lookup — worth naming as a real scaling problem before shipping it (see the follow-up on this below); a production implementation typically keeps a prefix-sum array (or a Fenwick/BIT-style structure) that updates incrementally as heights are measured, rather than re-summing from zero every time.

Rendering with measurement — the key addition is measuring each rendered row's *actual* height after it's in the DOM, and correcting the total/estimated heights if reality differs from the estimate:

```javascript
function renderWindow() {
  const scrollTop = container.scrollTop;
  const containerHeight = container.clientHeight;

  // Find the first visible index by walking cumulative height (binary search
  // over a maintained prefix-sum array in a real implementation).
  const startIndex = findIndexAtOffset(scrollTop);
  let endIndex = startIndex;
  let accumulated = 0;
  while (endIndex < items.length && accumulated < containerHeight) {
    accumulated += getHeight(endIndex);
    endIndex++;
  }

  const renderStart = Math.max(0, startIndex - OVERSCAN);
  const renderEnd = Math.min(items.length, endIndex + OVERSCAN);

  windowEl.style.transform = `translateY(${getOffsetForIndex(renderStart)}px)`;

  const fragment = document.createDocumentFragment();
  const rowsToMeasure = [];
  for (let i = renderStart; i < renderEnd; i++) {
    const row = document.createElement('div');
    row.textContent = items[i].label; // variable-length text -> variable wrapped height
    row.dataset.index = i;
    fragment.appendChild(row);
    rowsToMeasure.push([i, row]);
  }
  windowEl.replaceChildren(fragment);

  // Measure AFTER attaching to the DOM — height is unknown before layout.
  let totalHeightChanged = false;
  for (const [index, row] of rowsToMeasure) {
    const realHeight = row.getBoundingClientRect().height;
    const previouslyAssumed = getHeight(index);
    if (Math.abs(realHeight - previouslyAssumed) > 0.5) {
      measuredHeights.set(index, realHeight);
      totalHeightChanged = true;
    }
  }

  if (totalHeightChanged) {
    // The phantom's total height, and every offset after the changed rows,
    // is now stale — recompute and, if the user's current scroll position
    // was based on the old (wrong) estimate, correct scrollTop so the
    // content under their eyes doesn't visibly jump.
    updatePhantomHeight();
  }
}

function updatePhantomHeight() {
  let total = 0;
  for (let i = 0; i < items.length; i++) total += getHeight(i);
  phantom.style.height = `${total}px`;
}
```

## Gotchas

**Letting the phantom element size to its actual (small) content instead of the full logical height.** Without an explicit height reflecting *all* items, the scrollbar reflects only the handful of rendered rows — scrolling reaches the "end" almost immediately, and the whole premise of "scroll through 50,000 items" breaks.

**Using `top`/`margin-top` instead of `transform: translateY()` to position the rendered window.** Changing `top` on a positioned element (or `margin-top`) triggers layout recalculation; `transform` is handled by the compositor and avoids that cost — a difference that's negligible once, but compounds significantly across every scroll-driven re-render.

**No overscan, or too much overscan.** Zero overscan shows visible blank flashes during fast scroll (the one-frame gap between scroll and render). Excessive overscan (rendering hundreds of extra rows "to be safe") defeats the entire purpose of virtualization by letting rendered node count creep back toward the un-virtualized baseline.

**Recomputing full cumulative offsets from scratch on every render in the variable-height case.** The naive `getOffsetForIndex` shown above is O(n) per call and O(n) per render if called repeatedly — for tens of thousands of rows this reintroduces a real performance problem, just one frame later than the original "render everything" problem. A real implementation needs an incrementally-maintained prefix-sum structure, not a fresh linear scan per scroll event.

**Focus loss when a focused row scrolls out of the rendered window.** If row 500 has DOM focus and the user scrolls far enough that row 500 gets unmounted (no longer in the rendered slice), focus silently falls back to `<body>` with no warning — a real, easily-missed bug for any virtualized list that supports row interaction/selection.

**Treating `aria-setsize`/`aria-posinset` as a full accessibility fix.** These attributes correctly announce "item 47 of 50,000" to a screen reader for a row that *is* rendered, but they don't let a screen reader user actually reach or read item 30,000 without first triggering enough scroll/render cycles to get there — which typically requires sighted, mouse-driven scrolling, not standard screen reader navigation gestures. This is a real, acknowledged limitation, not something to paper over.

## Follow-up Questions

**Q (High): Why is a tall phantom/spacer element necessary, and what specifically would break in the scrollbar/scroll behavior without it?**

Answer: The browser computes scrollbar size and scrollable range from the actual rendered content's height inside the scroll container — if only the handful of currently-visible rows are ever in the DOM, the container's content height is just `visibleRowCount * rowHeight`, which the browser reports as "the entire scrollable content," making the scrollbar reach its end almost immediately and making it impossible to scroll to item 40,000 since, as far as the browser is concerned, there's no more content past a few dozen rows. The phantom element's explicit height (`totalItemCount * rowHeight`, or the running cumulative sum in the variable-height case) gives the browser a scrollable range that matches the *logical* dataset size, while the actually-rendered rows (a small window positioned via `transform` inside that phantom space) are what's cheap to paint.

The trap: describing the phantom as "just a visual spacer" without connecting it to the browser's actual scrollbar-size computation — the phantom isn't decorative, it's the mechanism that makes native scroll behavior (scrollbar thumb size, scroll range, even things like scrollbar-driven "scroll to roughly 60% through the list") correct despite almost nothing being rendered.

---

**Q (High): Walk through what changes, mechanically, when row heights become variable and unknown until render — why can't you just keep using `index * rowHeight`?**

Answer: `index * rowHeight` assumes every row has the identical, known-in-advance height, which lets you compute any row's offset with pure arithmetic and no DOM interaction. Once heights vary (e.g., wrapped text of different lengths) and aren't reliably known ahead of time, there's no formula — you only learn a row's real height by rendering it and calling something like `getBoundingClientRect().height` after layout. This forces a fundamentally different architecture: maintain a cache of *measured* heights (populated lazily, only for rows that have actually been rendered at least once), use a *default estimate* for anything unmeasured (needed to compute a phantom height and offsets for rows far from the current scroll position that have never been rendered), and — the hard part — when an estimated row is finally rendered and its real height turns out to differ from the estimate, every offset for rows *after* it is now wrong until recomputed, and if the user's current scroll position was itself based on stale offsets, the visible content can visibly jump unless the correction also adjusts `scrollTop` to compensate.

The trap: proposing "just measure every row upfront before rendering anything" — this defeats the purpose of virtualization entirely (it requires rendering, or at least laying out, every row at least once, which is exactly the cost windowing exists to avoid) and doesn't even work for content whose height can depend on container width or dynamic content loaded later.

---

**Q (High): What's the accessibility trade-off inherent to virtualization, and why isn't `aria-setsize`/`aria-posinset` a complete fix?**

Answer: `aria-setsize` and `aria-posinset` correctly tell a screen reader, for a row that *is currently rendered*, its logical position and the total logical count ("item 47 of 50,000") — this is genuinely useful and better than leaving those attributes off. But it doesn't solve the underlying problem: a screen reader user navigating linearly (arrow-key/next-item navigation within the list's accessibility tree) can only reach items that exist in the DOM at that moment; items 48 through 50,000 don't exist yet, so there's no way to "arrow down" to them the way a sighted user can scroll to them. In practice, a screen reader user is dependent on the same scroll-triggered rendering a sighted user gets, which is a fundamentally different (and typically worse) interaction model than linear keyboard/AT navigation through a fully-rendered list. This is a real, unavoidable trade-off of virtualization — not a gap that more ARIA attributes closes.

The trap: presenting `aria-setsize`/`aria-posinset` as "making the virtualized list accessible" full stop — the honest answer is that virtualization and full assistive-technology navigability are in genuine tension, and the right response in an interview is naming that trade-off explicitly (and discussing when it's acceptable — e.g., a visually-dense data grid where an alternative accessible view/export might be the real solution — rather than claiming ARIA attributes alone resolve it).

---

**Q (Medium): Why should the scroll handler be throttled via `requestAnimationFrame`, and what's wrong with `setTimeout`-based throttling for this specific use case?**

Answer: `scroll` events can fire far more frequently than the display refreshes, so recomputing the visible range and re-rendering on every raw event does redundant work for frames the user will never see distinct output for. `requestAnimationFrame`-based throttling (queue at most one pending recompute-and-render per animation frame, ignoring additional `scroll` events until that frame's callback runs) naturally aligns the work to the browser's actual paint cadence, doing exactly as much work as there are frames to show, no more. A `setTimeout`-based throttle instead ties the update rate to an arbitrary millisecond interval that has no inherent relationship to the display's refresh rate — pick too short an interval and you're back to near-every-event overhead; pick too long and scrolling visibly lags behind the actual scroll position, since updates are now capped below the display's frame rate rather than aligned to it.

The trap: treating `requestAnimationFrame` and `setTimeout(fn, 16)` as roughly equivalent "since 16ms is about one frame at 60fps" — a hardcoded interval doesn't adapt to variable refresh rates (90Hz, 120Hz displays) or to frames where the browser is already busy, whereas `requestAnimationFrame` is scheduled by the browser specifically to align with whatever the actual next paint opportunity is.

---

**Q (Medium): How would you support programmatically scrolling to an arbitrary index (e.g., "jump to item 8000") in the variable-height case, where most rows have never been measured?**

Answer: Compute a best-effort target scroll offset using `getOffsetForIndex(targetIndex)` against the *current* cache-plus-estimates (measured heights where known, the default estimate for everything else), scroll there immediately, then render the window at that position — which will, for the first time, measure the rows now newly rendered near the target. If those measurements differ meaningfully from the estimates that were used to compute the jump target, apply a correction pass: recompute the offset for `targetIndex` using the now-more-accurate heights and adjust `scrollTop` again if it drifted enough to matter (a small, imperceptible drift usually isn't worth correcting; a large one is). This means a single "jump to index" interaction can involve one initial best-effort scroll plus a follow-up micro-correction, rather than a single perfectly-accurate jump — which is an inherent consequence of not knowing real heights until render.

The trap: assuming the first estimate-based jump is exact — for a list with meaningfully variable row heights and a jump target far from anything previously rendered/measured, the based-on-estimates offset can be noticeably off, and a robust implementation acknowledges and corrects for that rather than treating the estimate as ground truth.

---

**Q (Medium): What happens to a focused or selected row when it scrolls out of the rendered window, and how would you handle it?**

Answer: By default, nothing graceful — if row 500 currently holds DOM focus and scrolling causes it to fall outside the rendered index range, its DOM node is removed/replaced (via `replaceChildren` or similar), and focus reverts to `document.body` with no visual or programmatic indication of where it went, which is a jarring, broken-feeling experience for keyboard users. A deliberate handling strategy is needed: options include keeping a focused/selected row's index tracked in state independent of the DOM (so it can be re-focused if it scrolls back into range), moving focus to a sensible fallback (e.g., the list container itself, with `tabindex="-1"`) when the focused row is about to be unmounted, or — for selection specifically (as opposed to focus) — keeping selection state entirely in JS/data (not DOM-attribute-based) so it survives rows being unmounted and remounted, and is correctly reflected (e.g., a `aria-selected`/checked visual state) whenever a previously-selected row happens to re-render.

The trap: not handling this at all and only discovering it when asked directly — this is a very natural follow-up precisely because "make it virtualized" and "keep it interactive" pull against each other, and a candidate who's only thought about the read-only rendering case usually hasn't considered it.

---

**Q (Low): How would horizontal virtualization (a very wide table with many columns) differ from vertical row virtualization?**

Answer: The mechanism is symmetric — compute a visible column-index range from `scrollLeft` and container width, render only those columns, position the rendered slice via `transform: translateX(...)` inside a phantom whose width equals the full logical row width — but it introduces the added complication that visible *rows* and visible *columns* are now independent axes, so a full 2D virtualized grid needs to compute and render the intersection of the visible row range and visible column range, not just one axis at a time, and has to handle sticky/frozen columns or headers (common in spreadsheet-like UIs) as a separate exception to the general windowing logic, since those need to always render regardless of horizontal scroll position.

The trap: assuming horizontal virtualization is "the same code with x/y swapped" without accounting for the combinatorial nature of a 2D grid — a naive port of the 1D logic to two independent axes without handling their intersection (and any frozen rows/columns) tends to either double-virtualize incorrectly or fail to account for cells that should always be rendered regardless of scroll position.

---

**Q (Low): Why is `getOffsetForIndex`'s naive O(n) linear-scan implementation a real problem at scale, and what data structure fixes it?**

Answer: For a list of 50,000 items, computing a single row's offset by summing every preceding row's height is 50,000 additions per call — and this function is called on essentially every scroll-driven render, potentially once per rendered row per frame, which reintroduces an O(n) (or worse) cost into the exact hot path virtualization exists to keep cheap. A prefix-sum array (recomputed incrementally, not from scratch, whenever a height changes) turns offset lookup into O(1); if heights change frequently enough that maintaining a plain prefix-sum array's suffix (everything after a changed index) is itself too costly to recompute repeatedly, a Fenwick tree (binary indexed tree) supports both point updates and prefix-sum queries in O(log n), which scales better under frequent height corrections.

The trap: treating this as a premature optimization not worth mentioning — for the scale this scenario explicitly describes (tens of thousands of rows), an O(n) offset lookup called per-render is a genuine, measurable regression, not a hypothetical one; naming the fix (even without implementing the Fenwick tree in full) is what signals the candidate has actually reasoned about the scaling behavior of their own solution.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement fixed-height virtualization (phantom spacer + `transform`-positioned window + index-range math) from memory
- [ ] Can explain why the phantom element needs an explicit full-height, and what breaks in scroll/scrollbar behavior without it
- [ ] Can explain the measure-cache-estimate-correct loop required for variable row heights, including why offsets after a corrected row become stale
- [ ] Can explain why `transform: translateY()` is preferred over `top`/`margin-top` for positioning the rendered window
- [ ] Can explain overscan's purpose and the cost/benefit of choosing its size
- [ ] Can articulate the genuine, unresolved accessibility trade-off virtualization creates for screen reader users, not just cite `aria-setsize`/`aria-posinset` as a fix

---
*Next: Accessible Modal With Focus Trap — shifting from data-heavy list patterns to interaction/accessibility-heavy widget patterns, which dominate the rest of this phase.*
