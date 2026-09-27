# Drag-and-Drop Sortable List

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Native DnD API | `draggable="true"` + `dragstart`/`dragover`/`drop`/`dragend` + `event.dataTransfer` | The browser-native way to implement drag reordering without a library |
| `preventDefault()` in `dragover`/`drop` | Must call it inside `dragover` or `drop` never fires at all | The single most common bug in this scenario — an element isn't a valid drop target by default |
| Drop index calculation | Compare pointer Y (or X) to the midpoint of each sibling's bounding rect | Determines "insert before" vs "insert after" the hovered item, not just "which item is hovered" |
| Touch support gap | Native HTML5 DnD largely doesn't fire on mobile touch | Forces a pointer-events (or Pointer Events API) reimplementation for any touch-friendly product |
| Optimistic update + rollback | Reorder the UI immediately, revert on persistence failure | Keeps the interaction feeling instant while still handling the backend rejecting the reorder |
| Keyboard alternative | "Move up"/"Move down" buttons, or a keyboard grab/drop mode per WAI-ARIA APG | Drag-and-drop has no native keyboard equivalent — an accessible list needs one built deliberately |

## The Scenario

"Build a reorderable list — think a playlist or a task list where users can drag items to change their order. Use native drag-and-drop, not a library. And make sure it actually persists — assume there's a `saveOrder(newOrder)` API call that can fail."

## Clarifying Questions

- **Does this need to work on touch devices, or is desktop-only acceptable for this pass?** This matters enormously because native HTML5 Drag and Drop largely doesn't fire drag events on mobile touch browsers — if touch support is required, that's not an extension of the same code, it's effectively a second implementation using pointer events instead.
- **What happens if `saveOrder` fails after the user has already seen the list reorder?** This determines whether I build optimistic UI (reorder immediately, roll back on failure) or pessimistic UI (wait for the server to confirm before visually reordering). Optimistic is almost always the right UX call for a reorder action, but it means I need to keep the previous order around to restore it.
- **Does this need to be accessible to keyboard-only or screen-reader users?** Native drag-and-drop has no keyboard equivalent at all — a mouse-only implementation completely locks out keyboard users. I'd want to know upfront whether an accessible alternative (button-based reordering, or a keyboard grab/move mode) is in scope, since it changes the shape of the markup from the start rather than being bolted on later.
- **Can items be dragged to any position in the list, or just swapped with adjacent siblings?** Free repositioning (drop anywhere in the list) requires computing an insertion index from pointer position; adjacent-swap-only is a much smaller feature. I'd assume free repositioning unless told otherwise, since that's what "drag to reorder" usually means.
- **Is there a maximum list size where per-frame DOM measurement (`getBoundingClientRect` on every sibling during drag) could become a performance concern?** For a short list this is a non-issue; for a list with hundreds of items rendered without virtualization, recalculating every sibling's rect on every `dragover` event could get expensive, and I'd want to know if that's a realistic scale.

## Approach & Trade-offs

The scenario explicitly asks for the *native* HTML5 Drag and Drop API rather than a pointer-events-based custom implementation or a library like SortableJS/dnd-kit — so the core mechanism is `draggable="true"` on each list item, plus the `dragstart` → `dragover` → `drop` → `dragend` event sequence, using `event.dataTransfer` to identify which item is being dragged.

The trickiest mechanical piece is that `dragover` and `drop` don't fire by default — an element is not a valid drop target unless its `dragover` handler calls `event.preventDefault()`. This is a deliberate browser default (most elements aren't drop targets, e.g. you don't want dropping a file onto a `<p>` to do anything by default), but it's the single most common reason a first attempt at this feature "does nothing" when a candidate drags an item and drops it.

For computing *where* to insert the dragged item, I don't just track "which item is currently hovered" — I compare the pointer's position against the vertical midpoint of the hovered item's bounding rect. If the pointer is above the midpoint, the dragged item should land *before* that sibling; if below, *after*. This is what makes the reorder feel like it's tracking the cursor smoothly rather than snapping awkwardly between only "before the first" and "after the last" states.

For touch, I'd flag upfront — not bury as an afterthought — that native HTML5 DnD is a poor fit. Most mobile browsers don't dispatch `dragstart`/`dragover`/`drop` for touch input at all (this is a long-standing, well-documented limitation, not an edge case). If the product needs to work on phones/tablets, the honest answer is: you don't extend this implementation, you build a parallel one using `pointerdown`/`pointermove`/`pointerup` (or a library that abstracts over both), because Pointer Events unify mouse, touch, and pen. I'd say this out loud rather than let the interviewer discover it later — it signals real experience with this feature's rough edges rather than naive optimism about "drag-and-drop just works everywhere."

For persistence, I chose **optimistic reordering with rollback** over waiting for server confirmation before updating the UI: the dragged item visibly lands in its new position the instant the user drops it (since that's what "drag to reorder" implies — instant, direct manipulation), and the `saveOrder` call happens in the background. If it fails, I revert the list to the pre-drag order and surface an error (a toast, tying back into the earlier scenario) rather than leaving the UI in a state that silently disagrees with the server's actual stored order.

Lastly — and I'd raise this as a first-class requirement, not a nice-to-have — native drag-and-drop is entirely mouse/touch-pointer-driven and has **no keyboard equivalent** built into the browser. A keyboard user physically cannot trigger `dragstart`. Meeting the WAI-ARIA APG's guidance here means shipping an alternative interaction alongside the drag handles: either explicit "Move up" / "Move down" buttons per item (simplest, most robust), or a keyboard-operable "grabbed" mode (press Enter/Space to grab an item, Arrow keys to move it, Enter/Space again to drop, Escape to cancel) that mirrors the visual drag semantics more closely. I'd default to the button approach for a first pass since it's simpler to get right and test, and mention the grabbed-mode pattern as the more polished option if there's time.

## Solution

Markup — each item needs `draggable="true"` and a stable identifier:

```html
<ul id="sortable-list">
  <li draggable="true" data-id="item-1">Buy milk</li>
  <li draggable="true" data-id="item-2">Walk the dog</li>
  <li draggable="true" data-id="item-3">Write report</li>
</ul>
```

`dragstart` — record which item is being dragged:

```javascript
const list = document.getElementById('sortable-list');
let draggedId = null;

list.addEventListener('dragstart', (event) => {
  const item = event.target.closest('li');
  if (!item) return;
  draggedId = item.dataset.id;
  event.dataTransfer.setData('text/plain', draggedId); // required for Firefox
  event.dataTransfer.effectAllowed = 'move';
  item.classList.add('dragging'); // visual feedback
});

list.addEventListener('dragend', (event) => {
  event.target.closest('li')?.classList.remove('dragging');
  draggedId = null;
});
```

`dragover` — this is where `preventDefault()` is mandatory, and where the insertion index gets computed from pointer position vs. sibling midpoints:

```javascript
list.addEventListener('dragover', (event) => {
  event.preventDefault(); // without this, 'drop' never fires — the #1 bug here
  event.dataTransfer.dropEffect = 'move';

  const afterElement = getElementAfterPointer(list, event.clientY);
  const dragging = list.querySelector('.dragging');
  if (!dragging) return;

  if (afterElement == null) {
    list.appendChild(dragging); // pointer is below every item — drop at the end
  } else {
    list.insertBefore(dragging, afterElement);
  }
});

function getElementAfterPointer(container, pointerY) {
  const items = [...container.querySelectorAll('li:not(.dragging)')];

  return items.reduce((closest, item) => {
    const rect = item.getBoundingClientRect();
    const midpoint = rect.top + rect.height / 2;
    const offset = pointerY - midpoint;

    // We want the first item whose midpoint is still BELOW the pointer
    // (offset < 0 means pointer is above this item's midpoint).
    if (offset < 0 && (closest === null || offset > closest.offset)) {
      return { offset, element: item };
    }
    return closest;
  }, null)?.element ?? null;
}
```

`drop` — finalize order and persist, with optimistic update + rollback:

```javascript
list.addEventListener('drop', async (event) => {
  event.preventDefault();

  const previousOrder = [...list.children].map((el) => el.dataset.id);
  // The DOM has already been reordered live during dragover — this IS the
  // optimistic update. We just need to capture the new order and persist it.
  const newOrder = [...list.children].map((el) => el.dataset.id);

  try {
    await saveOrder(newOrder);
  } catch (err) {
    // Rollback: rebuild the list in the pre-drag order.
    rebuildListOrder(list, previousOrderBeforeDrag);
    showToast('Could not save new order — reverted.', { variant: 'error' });
  }
});
```

> Note: capturing "previous order" accurately requires snapshotting it in `dragstart`, before any `dragover`-driven DOM moves happen — `previousOrder` above is illustrative; a real implementation stores the pre-drag snapshot once, in `dragstart`, not inside `drop`.

The accessible non-drag alternative — explicit move buttons, always present alongside the drag handle, not just for a "no-JS" fallback:

```html
<li draggable="true" data-id="item-2">
  <span class="drag-handle" aria-hidden="true">⠿</span>
  <span>Walk the dog</span>
  <button type="button" aria-label="Move up">↑</button>
  <button type="button" aria-label="Move down">↓</button>
</li>
```

```javascript
list.addEventListener('click', async (event) => {
  const button = event.target.closest('button[aria-label="Move up"], button[aria-label="Move down"]');
  if (!button) return;

  const item = button.closest('li');
  const direction = button.getAttribute('aria-label') === 'Move up' ? -1 : 1;
  const sibling = direction === -1 ? item.previousElementSibling : item.nextElementSibling;
  if (!sibling) return; // already at an edge — no-op

  const previousOrder = [...list.children].map((el) => el.dataset.id);
  if (direction === -1) {
    list.insertBefore(item, sibling);
  } else {
    list.insertBefore(sibling, item);
  }
  item.focus(); // keep focus on the moved item, don't let it get lost

  try {
    await saveOrder([...list.children].map((el) => el.dataset.id));
  } catch {
    rebuildListOrder(list, previousOrder);
    showToast('Could not save new order — reverted.', { variant: 'error' });
  }
});
```

> **Check yourself:** Without looking back, explain exactly why `drop` silently never fires if you omit `event.preventDefault()` from the `dragover` handler — what is the browser's default behavior being prevented?

## Gotchas

**Forgetting `preventDefault()` in `dragover`.** By default, no element is a valid drop target; without calling `preventDefault()` inside `dragover` (some browsers also require it in `drop`), the `drop` event simply never fires, and the cursor shows a "not allowed" icon during drag. This is the #1 stumbling block in this exact scenario.

**Not setting `dataTransfer.setData(...)` at all.** Firefox in particular requires *some* call to `setData` in `dragstart` for the drag to initiate properly; omitting it can silently break dragging in that browser while working fine in Chrome.

**Computing "which item is hovered" instead of "insert before or after."** Using only `event.target` to find the hovered `<li>` tells you which item the pointer is over, but not which *side* to insert on — without the midpoint comparison, the reorder logic either always inserts on one fixed side (items can't move past their neighbor correctly) or jitters.

**Assuming native DnD works on mobile.** This is worth stating explicitly and early — most mobile browsers don't fire `dragstart`/`dragover`/`drop` for touch, meaning a native-DnD-only implementation is desktop-only in practice, not "works everywhere, just clunkier on mobile."

**No accessible alternative at all.** A drag-only interface is a hard accessibility failure (WCAG 2.1 SC 2.1.1, Keyboard) — there is no way to trigger `dragstart` from a keyboard, and no browser provides a built-in keyboard equivalent for HTML5 DnD. This has to be designed in, not retrofitted.

**Losing focus after a reorder.** After a button-driven move (or a keyboard grab/drop move), if focus isn't explicitly restored to the moved item, a keyboard/screen-reader user loses their place in the list entirely.

**No rollback on persistence failure.** If `saveOrder` fails and the UI doesn't revert, the client and server now permanently disagree about the list's order until the next full reload — a subtle, hard-to-notice bug that erodes trust in the feature.

## Follow-up Questions

**Q (High): Why does `drop` never fire unless you call `preventDefault()` in `dragover` (or `drop` itself)? What's actually being prevented?**

Answer: The browser's default behavior for `dragover` is to indicate "this is not a valid drop target," which is why, absent any handler, dragging something over an arbitrary element shows a "no-drop" cursor and dropping does nothing. Calling `event.preventDefault()` inside the `dragover` handler overrides that default and tells the browser "this element accepts drops," which is a prerequisite for `drop` to fire at all when the user releases the pointer over it. It's a deliberate opt-in design, since most elements on a page (arbitrary text, images) aren't meant to be drop targets by default — e.g. dragging a file from the desktop onto a random `<div>` shouldn't do anything unless that `div` explicitly opts in.

The trap: describing this as "a browser bug" or "an annoying requirement" rather than understanding it as an intentional default that every valid drop target must explicitly opt out of.

---

**Q (High): Native HTML5 Drag and Drop has famously poor touch support. What would you actually do to ship a touch-friendly version of this feature?**

Answer: Reimplement the interaction using Pointer Events (`pointerdown`, `pointermove`, `pointerup`, plus `setPointerCapture` on the dragged element) instead of the HTML5 DnD event set, since Pointer Events unify mouse, touch, and pen input under one API and actually fire consistently on mobile touch browsers. This means manually tracking the "dragged" element's position (e.g., via `transform: translate(...)` following the pointer) and manually computing the same before/after insertion-index logic on `pointermove` that `dragover` gave for free in the native API — the reordering *logic* (midpoint comparison, insertion) carries over, but the event plumbing has to be rebuilt. In practice, most production apps just adopt a library (dnd-kit, SortableJS) that's already solved this dual-implementation problem, rather than hand-rolling both.

The trap: claiming native HTML5 DnD "mostly works" on mobile with minor tweaks — it doesn't; the drag events largely don't fire at all for touch in most mobile browsers, so this isn't a polish problem, it's a different implementation.

---

**Q (High): This feature has no keyboard equivalent by default. How do you fix that, concretely?**

Answer: Two viable patterns, per the WAI-ARIA Authoring Practices Guide: (1) explicit "Move up" / "Move down" (or "Move to position...") buttons rendered on every list item, which perform the same reorder logic as a completed drag, are trivially keyboard-focusable and screen-reader-labeled, and are the simplest to implement and test reliably; or (2) a keyboard-operable "grabbed" mode, where pressing Enter/Space on a focused, `draggable`-marked item toggles a "grabbed" state (announced via `aria-grabbed` or a live region), Arrow Up/Down then move the grabbed item within the list, and a second Enter/Space (or Escape to cancel) drops it — mirroring the visual drag gesture more closely but requiring more careful state management and clear instructions for the user. I'd default to the button approach for reliability and move to the grabbed-mode pattern only if the product explicitly wants drag-parity keyboard UX.

The trap: saying "just add `tabindex` to the draggable items" — that makes items focusable, but focus alone doesn't let a keyboard user *move* anything; there's still no key that triggers reordering without additional, deliberately built interaction logic.

---

**Q (Medium): Walk through how you compute whether the dragged item should be inserted before or after the hovered sibling.**

Answer: On every `dragover`, for each candidate sibling (excluding the item currently being dragged), get its bounding rect via `getBoundingClientRect()` and compute its vertical midpoint (`rect.top + rect.height / 2`). Compare the pointer's current `clientY` to that midpoint: if the pointer is above the midpoint, the dragged item should be inserted *before* that sibling; if below, *after* — practically, this is implemented by finding the first sibling (in document order) whose midpoint is still below the pointer, and inserting before it, falling back to appending at the end if no such sibling exists (pointer is below everything).

The trap: comparing against the sibling's `top` or `bottom` edge instead of its midpoint — that produces a "dead zone" or overly aggressive swap threshold that feels wrong compared to the expected behavior of crossing the midpoint of an item to swap past it.

---

**Q (Medium): Why optimistic reordering with rollback, rather than waiting for the server to confirm before updating the UI?**

Answer: Drag-and-drop is a direct-manipulation gesture — the entire point is that the item visibly follows the user's action in real time; introducing a round-trip delay before the reorder visually completes would make the interaction feel broken or laggy, since the user already "physically" placed the item during the drag. Optimistic UI updates the list immediately on drop and fires the persistence call in the background; if it fails, the list reverts to its pre-drag order (captured as a snapshot at `dragstart`, before any DOM mutations from `dragover`) and the user is notified, e.g., via a toast. The trade-off is added complexity — you need to keep the correct rollback snapshot and handle the failure path — in exchange for a UI that feels instant on the common (success) path.

The trap: implementing optimistic reordering but forgetting to actually snapshot the pre-drag order before `dragover` starts mutating the DOM, so "rollback" has nothing correct to roll back to.

---

**Q (Medium): How would you extend this to support reordering across two separate lists (e.g., a Kanban board's columns)?**

Answer: The `dragstart` still records the dragged item's id and, additionally, its source list; each list container gets its own `dragover`/`drop` handlers using the same midpoint-based insertion logic, but the `drop` handler now needs to handle the case where the target list differs from the source list — removing the item from the source list's data structure and inserting it into the target list's, then persisting both lists' new orders (or a single combined "move item X to list Y at position Z" API call, which is usually the cleaner backend contract). The visual reordering during `dragover` also has to move the DOM node across list boundaries live, not just within one list.

The trap: treating this as "just run the same single-list logic twice" — cross-list moves need an explicit concept of "remove from source, insert into destination" as one atomic operation, both in the DOM and in whatever state/backend model tracks list membership.

---

**Q (Low): Why call `setData('text/plain', ...)` in `dragstart` if you're not planning to actually drag data out of the browser (e.g., onto the desktop)?**

Answer: Some browsers, notably Firefox, require at least one `setData` call during `dragstart` for the drag operation to be considered valid and for subsequent drag events to fire correctly, even in an internal same-page reorder scenario where you don't actually need the transferred data (you already have the dragged item's id from a JS closure variable). It's effectively a cross-browser compatibility requirement rather than something the reorder logic depends on functionally.

The trap: assuming this line is purely decorative or copy-pasted boilerplate and can be safely omitted — omitting it can break dragging specifically in Firefox while appearing to work fine during Chrome-only testing.

---

**Q (Low): What performance concern arises from calling `getBoundingClientRect()` on every sibling during every `dragover` event, and how would you mitigate it for a very long list?**

Answer: `getBoundingClientRect()` forces a layout read, and calling it on every sibling on every `dragover` (which fires continuously as the pointer moves, similar to `mousemove` frequency) means potentially hundreds of layout reads per second for a long, un-virtualized list — this can cause visible jank during drag. Mitigations include throttling the `dragover` handler's expensive work (e.g., only recomputing every animation frame via `requestAnimationFrame` rather than on every raw event), caching sibling rects once at `dragstart` and only invalidating the cache when the DOM order actually changes, or virtualizing the list so only a bounded number of items exist in the DOM regardless of total list length.

The trap: treating this as a non-issue because "lists are usually short" — it's true for most lists, but it's the kind of thing that turns into a real production bug the first time this component gets reused for a genuinely long list, so it's worth naming even if not implementing it upfront.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement the full `dragstart`/`dragover`/`drop`/`dragend` sequence including `preventDefault()` placement
- [ ] Can explain why `drop` never fires without `preventDefault()` in `dragover`
- [ ] Can compute an insertion index from pointer position vs. sibling midpoints
- [ ] Can state, unprompted, that native HTML5 DnD doesn't work reliably on mobile touch and what the fix requires
- [ ] Can implement optimistic reordering with a correct rollback snapshot on persistence failure
- [ ] Can design and implement a keyboard-operable, accessible alternative to dragging

---
*Next: Toast / Notification Queue System — a different flavor of dynamic list, driven by a queue and timers instead of user manipulation.*
