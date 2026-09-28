# Nested Scroll Container Overflow Trap

## Quick Reference

| Symptom | Root Cause | Fix |
|---|---|---|
| Scrolling an inner panel also scrolls the page behind it once the inner panel hits its scroll limit | Scroll chaining — the browser hands off leftover scroll delta to the next ancestor scroll container by default | `overscroll-behavior: contain` (or `none`) on the inner scroll container |
| A modal/drawer's background page scrolls when the user scrolls inside the modal on mobile | Same scroll-chaining mechanism, plus touch scroll momentum making it worse on iOS | `overscroll-behavior: contain` on the modal's scrollable region, and/or locking body scroll while the modal is open |
| Sticky element inside a scrollable panel doesn't stick, or sticks relative to the wrong boundary | The sticky element's nearest scrolling ancestor is the inner panel, not the outer page — same principle as [[02-sticky-header-broken-mobile-safari]] | Confirm which ancestor is actually the intended scroll boundary and place the sticky element inside that one specifically |
| Nested scroll container's content is clipped/cut off unexpectedly | An ancestor has `overflow: hidden` for an unrelated reason (clipping a decoration), inadvertently also clipping the nested scrollable area | Scope the clipping `overflow: hidden` more narrowly, or use `clip-path` on just the specific element needing it |
| Horizontal scroll container inside a vertical-scrolling page causes the whole page to scroll horizontally on trackpad/gesture | Ambiguous scroll gesture direction not resolved to the intended container | `overscroll-behavior-x: contain` on the horizontal container, and/or `touch-action` tuning for touch-specific gesture handling |

## The Scenario

"We have a chat app: a sidebar with a scrollable conversation list, and a main panel with a scrollable message thread. Both are independently scrollable — that part works. The bug: when a user scrolls to the bottom of the message thread and keeps scrolling (e.g., with a trackpad, after the last message), the whole page starts scrolling instead of just stopping. Same thing happens with a settings modal — scroll to the end of its content, keep scrolling, and the page underneath moves. Fix this so each scrollable region contains its own scroll and never leaks to whatever's behind or around it."

## Clarifying Questions

- **Is this observed primarily on trackpad/mouse wheel, touch (mobile), or both?** Scroll chaining exists on both input types, but the specific browser mechanisms and historical workarounds differ — touch-based scroll chaining on iOS Safari in particular has its own quirks (momentum/rubber-banding scroll continuing to propagate even after `overscroll-behavior` support existed in other contexts), so confirming which input types are affected changes how thoroughly the fix needs to be tested.
- **Does the app currently target only modern evergreen browsers, or does it need to support browsers predating `overscroll-behavior` (Safari added support later than Chrome/Firefox)?** `overscroll-behavior` is the correct, minimal modern fix, but if legacy Safari (pre-16) or another older browser is in scope, a JS-based fallback (intercepting wheel/touch events and calling `preventDefault()` conditionally at scroll boundaries) may need to be layered in for those specific browsers.
- **Are there any of these scrollable regions where scroll chaining is actually desired** — e.g., a small inline scrollable code block inside an article, where continuing to scroll past it should naturally continue scrolling the page, versus the chat thread and modal, where it clearly shouldn't? This matters because the fix (`overscroll-behavior: contain`) shouldn't be applied blindly to every `overflow: auto` element in the app — some nested scroll areas are intentionally "leaky" by design, and over-applying containment everywhere can itself become a UX regression (a small scrollable table that traps scroll and forces the user to move their cursor off it just to keep reading the page).
- **When the settings modal is open, should the page behind it be scrollable at all via any means (keyboard, screen reader virtual cursor), or should it be fully inert while the modal is open?** This is related to, but distinct from, the scroll-chaining fix — a modal that correctly contains scroll chaining via `overscroll-behavior` can still have its background scrollable through other means (keyboard `Page Down`, a screen reader's own navigation) unless the background is also given `inert` or `aria-hidden` + focus-trapping, which is a completeness question about modal behavior beyond just the wheel/touch scroll-chaining bug reported.
- **Is virtualization in use for the long message thread or conversation list (e.g., react-window, react-virtual)?** Virtualized lists manage their own scroll container and content sizing in a way that can interact with `overscroll-behavior` and native scroll-chaining differently than a plain `overflow: auto` div with all items rendered — worth confirming since the fix might need to be applied at a different DOM level (the virtualization library's outer scroll container specifically) than expected.

## Approach & Trade-offs

**The root cause is scroll chaining, a deliberate, spec'd default browser behavior — not a bug in the literal sense, but a default that's frequently wrong for app-like UI (modals, panels, chat threads) even though it's often exactly right for document-like content.** When a user scrolls inside a nested scrollable element and that element reaches the end of its own scrollable range, the browser's default behavior is to hand off any remaining scroll delta (the part of the gesture that didn't get "consumed" by the inner element, since it had no more room to scroll) to the next ancestor scroll container up the chain — continuing to scroll the page behind a modal, or the sidebar behind a scrolled-out message thread, is that handoff working exactly as designed. This default exists because it's often genuinely desirable — a small scrollable `<pre>` code block embedded in an article shouldn't "trap" a reader's scroll gesture; naturally continuing to scroll the surrounding article once the code block's content is exhausted is the right, expected behavior there. The task isn't "scroll chaining is broken, disable it everywhere" — it's identifying specifically which scroll containers in this app are conceptually self-contained regions (a modal, a chat thread meant to feel like its own scrollable "pane") where chaining is wrong, and containing scroll only at those boundaries.

**`overscroll-behavior: contain` (or `none`) is the direct, purpose-built CSS property for this — reach for it first, not a JS wheel/touch event workaround, because it's declarative, handles both wheel and touch input uniformly, and doesn't have the input-lag or `preventDefault()`-correctness pitfalls a manual JS implementation risks.** Before `overscroll-behavior` existed, the standard workaround was a wheel/touchmove event listener that checks whether the scroll container is at its boundary and, if so, calls `event.preventDefault()` to stop the browser from handing off the remaining delta — functionally similar in intent, but with real downsides: it requires careful boundary-detection math (checking `scrollTop`/`scrollHeight`/`clientHeight` precisely, including subpixel/rounding edge cases that differ across browsers), it can introduce a perceptible input lag if the JS runs on a busy main thread (a jank-prone chat app rendering new messages is exactly the kind of place this could bite), and getting `preventDefault()` timing subtly wrong can break the *inner* element's own scrolling, not just fail to prevent chaining. `overscroll-behavior: contain` does the equivalent job as a pure CSS declaration, evaluated by the browser's own scroll-handling logic rather than user-land JS running on every scroll event, and needs zero manual boundary math.

**`contain` vs. `none` is a real distinction worth choosing deliberately, not defaulting to whichever comes to mind first.** `overscroll-behavior: contain` prevents scroll chaining to the next ancestor, but *keeps* the element's own default overscroll effects within itself (e.g., the rubber-band/bounce effect at the boundary on platforms that have one, or a "glow" overscroll indicator) — it contains the *chaining*, not the visual overscroll affordance itself. `overscroll-behavior: none` goes further, disabling both the chaining *and* the element's own overscroll effect entirely. For a chat message thread, `contain` is usually right — the user still gets normal platform-native feedback that they've hit the end of the scrollable content (rubber-banding on iOS/macOS, for instance), it just doesn't leak to the page behind it. For a case where even the local overscroll effect is undesirable (e.g., a custom-styled component where a native rubber-band effect would look visually broken against a bespoke design), `none` is the more complete containment.

**Modal background scroll containment needs to be addressed at two levels, not one, because `overscroll-behavior` on the modal's own scrollable content only solves the wheel/touch scroll-chaining path — it doesn't make the background inert to every other means of scrolling it.** Beyond the immediate reported bug (scroll chaining via wheel/touch), a fully correct modal implementation also needs to prevent the background page from scrolling via keyboard (`Page Down`/`Space` while focus is somehow still reachable on background content) or programmatically — which is really a focus-management and `inert`/`aria-hidden` question, not a scroll-CSS one. It's worth naming as a related-but-distinct requirement rather than conflating it with the `overscroll-behavior` fix, since a candidate who only proposes `overscroll-behavior` for the modal has correctly fixed the specific reported symptom but may not have delivered a fully robust modal (background scroll containment is exactly the kind of adjacent requirement covered more fully in the accessible-modal scenario in Phase 9).

## Solution

**Step 1 — contain scroll on the message thread panel, addressing the primary reported bug directly:**

```css
.message-thread {
  overflow-y: auto;
  overscroll-behavior-y: contain; /* stop scroll chaining to the page once this panel hits its boundary */
}
```

```css
.conversation-sidebar {
  overflow-y: auto;
  overscroll-behavior-y: contain; /* same fix, same reasoning, independently scoped */
}
```

Each scrollable region gets its own `overscroll-behavior: contain` — this isn't a single global fix, since each container independently needs to stop propagating its own leftover scroll delta upward, regardless of what its specific ancestor chain looks like.

**Step 2 — contain the settings modal's scrollable content:**

```css
.modal-content {
  overflow-y: auto;
  overscroll-behavior-y: contain;
  max-height: 80vh; /* a bounded height is required for overflow to trigger at all */
}
```

**Step 3 — address the broader "is the background inert while the modal is open" question, since `overscroll-behavior` alone doesn't cover keyboard/programmatic scroll paths:**

```css
body.modal-open {
  overflow: hidden; /* prevents the page itself from being scrollable by any means while a modal is open */
}
```

```tsx
useEffect(() => {
  if (isModalOpen) {
    document.body.classList.add('modal-open');
    return () => document.body.classList.remove('modal-open');
  }
}, [isModalOpen]);
```

Locking `body` scroll while the modal is open is a stronger, more complete guarantee than `overscroll-behavior` alone — it makes the background page non-scrollable through *any* input method for the duration the modal is open, which is usually the actual intended UX for a modal (the background isn't meant to be interactive at all while a modal has focus), rather than relying on scroll-chaining containment to merely make background scrolling *harder to trigger accidentally*.

**Step 4 — for a horizontal-scrolling sub-component (e.g., a horizontally-scrolling row of chat attachments) nested inside the vertically-scrolling thread, contain the horizontal axis specifically so ambiguous diagonal trackpad gestures don't bleed into vertical page scroll:**

```css
.attachment-row {
  display: flex;
  overflow-x: auto;
  overscroll-behavior-x: contain;
}
```

> **Check yourself:** If `.message-thread` has `overscroll-behavior-y: contain` but its parent, `.chat-main-panel`, also happens to be independently scrollable (e.g., due to a layout bug giving it its own unwanted `overflow-y: auto`), would the fix still work as expected? Reason through what "contain" actually stops versus what a genuinely unrelated second scroll container elsewhere in the tree could still do.

## Gotchas

**Applying `overscroll-behavior: contain` globally (e.g., on `body` or a top-level wrapper) instead of scoped to each specific inner scroll container that needs it** — `overscroll-behavior` needs to be set on the container whose chaining should be stopped, not on an ancestor; setting it in the wrong place either does nothing (the chaining already happened before reaching that ancestor) or has unintended broader effects on legitimate top-level page overscroll behavior (e.g., breaking a desired pull-to-refresh gesture on the whole page, if that's a feature the app relies on).

**Forgetting the axis-specific variants (`overscroll-behavior-x`/`-y`) when only one axis needs containment**, and reaching for the shorthand `overscroll-behavior` (which applies to both axes) on a container that's only scrollable in one direction, unintentionally also affecting the other axis's default behavior in a way that wasn't the actual goal (usually harmless since the other axis isn't scrollable anyway, but worth being precise about, especially in the horizontal-attachment-row case where both a horizontal and a vertical ancestor scroll container coexist nearby).

**Treating `overscroll-behavior: contain` on a modal's content as sufficient modal-scroll-isolation on its own**, without the separate `body`-scroll-lock (or `inert`) layer — leaves the background page scrollable via keyboard or programmatic means even though wheel/touch scroll chaining is correctly contained, an incomplete fix for what a "properly modal" experience actually requires.

**Not testing on actual iOS Safari specifically for the touch/momentum case** — historically, iOS Safari's rubber-band/momentum scrolling had its own quirks around scroll chaining that differed from desktop wheel-based chaining, and `overscroll-behavior` support arrived in Safari later than in Chromium/Firefox; a fix validated only via desktop trackpad testing (itself a valid input type to test, but not the only one) can miss a real remaining gap on an older-but-still-supported iOS Safari version.

**Applying containment indiscriminately to every scrollable element in the app, including ones where chaining is actually the desired, expected behavior** (an inline code block, a small scrollable table within a long article) — over-applying `overscroll-behavior: contain` everywhere "to be safe" can create a worse UX than the original bug in those cases, where a user naturally expects their scroll gesture to continue into the surrounding page once a small embedded scrollable region is exhausted.

## Follow-up Questions

**Q (High): Explain exactly what "scroll chaining" is as a browser default behavior, and why it's not simply a bug even though it produces the symptom described in this scenario.**

Answer: Scroll chaining is the default behavior where a scroll gesture (wheel, touch, trackpad) applied to a nested scrollable element continues to be dispatched to that element's ancestor scroll containers once the nested element has no more room to scroll in the gesture's direction — rather than a scroll gesture being "consumed entirely" by whichever element it started on, any leftover, unconsumed portion of the gesture propagates upward through the scroll-container ancestor chain until something absorbs it or the top of the document is reached. This exists because it's frequently the *correct* default for ordinary document-flow content — a reader scrolling through an article who encounters a small embedded scrollable element (a code block, an embedded iframe, a small table) generally expects their continued scroll gesture to keep moving them through the article once that embedded element's content is exhausted, not to feel "stuck" requiring them to move their pointer off the embedded element first. The behavior becomes undesirable specifically for UI that's meant to feel like a self-contained, app-like region (a modal, a message thread panel meant to feel like its own pane) — the scenario's bug isn't that scroll chaining is broken, it's that the default (chaining enabled) is the wrong choice for these specific containers, which is exactly what `overscroll-behavior: contain` exists to override on a per-element, opt-in basis.

The trap: describing this as "a browser bug" or "unexpected behavior" rather than the deliberate spec'd default it actually is — understanding it as an intentional, sometimes-correct default that needs selective overriding (not a universal bug needing suppression everywhere) is what leads to correctly scoping the fix rather than applying it indiscriminately, per the last Gotcha above.

---

**Q (High): What's the practical difference between `overscroll-behavior: contain` and `overscroll-behavior: none`, and which would you choose for the chat message thread specifically, and why?**

Answer: Both values stop scroll chaining to ancestor scroll containers — the leftover scroll delta at a boundary no longer propagates upward in either case. The difference is what happens *locally*, within the contained element itself, once its scroll boundary is reached: `contain` still allows the browser's native overscroll effect for that element (e.g., the rubber-band bounce on macOS/iOS, or a glow/bounce indicator on other platforms) to play out within the element's own bounds, whereas `none` suppresses that local overscroll effect entirely as well, leaving the scroll simply stop dead at the boundary with no visual affordance. For the chat message thread, `contain` is the better choice — it gives the user the same native, familiar "I've hit the end" feedback (a small bounce) they'd expect from any scrollable region on their platform, while still correctly preventing that leftover gesture from leaking into the page behind it; `none` would work equally well for the "stop chaining" requirement but produces a slightly less polished, more abrupt-feeling stop that isn't necessary to achieve the actual bug fix.

The trap: treating `contain` and `none` as interchangeable synonyms for "fix scroll chaining," missing that they're both valid fixes for the *specific reported bug* (chaining to the page) but produce different local UX, which is a legitimate, additional design decision on top of the correctness fix — a complete answer distinguishes the two rather than picking one arbitrarily.

---

**Q (Medium): If `overscroll-behavior` weren't available (say, supporting a much older browser where it's unsupported), how would you implement equivalent scroll containment with JavaScript, and what specifically makes that implementation harder to get right than the CSS version?**

Answer: The JS equivalent listens for `wheel` (and `touchmove`, for touch input) events on the scrollable container, and on each event, checks whether the container is currently at a scroll boundary in the direction the gesture is trying to move (e.g., for downward scroll, whether `scrollTop + clientHeight >= scrollHeight`, accounting for subpixel/rounding tolerance since these values aren't always exact integers across browsers) — if so, calls `event.preventDefault()` to stop the browser from handing the gesture off to the next ancestor, and otherwise lets the event proceed normally so the container's own scrolling isn't broken. What makes this harder to get right: the boundary check has to run on every scroll event (a potential performance concern if the handler does non-trivial work and the container scrolls frequently, especially relevant in a chat app that might also be receiving/rendering new messages concurrently), the check itself is easy to get subtly wrong at the exact boundary (off-by-one/rounding errors that either fail to prevent chaining right at the edge, or worse, prevent the container's own normal scrolling near the edge), touch events specifically require care around passive event listener defaults (`{ passive: false }` needs to be explicitly set for `preventDefault()` to have any effect on `touchmove`, which itself has scroll-performance implications since non-passive touch listeners can't be optimized by the browser the same way), and the whole implementation needs separate handling for wheel vs. touch vs. potentially keyboard-driven scroll, whereas `overscroll-behavior` handles all of these uniformly as a single declarative CSS property evaluated by the browser's own native scroll-handling code.

The trap: describing only the wheel-event listener and boundary check without mentioning the touch-specific `{ passive: false }` requirement or the performance/correctness risk of running boundary math on every scroll event — these are the specific details that make the JS fallback meaningfully more fragile and effortful than the one-line CSS fix, and are what an interviewer is checking for when asking "what makes it harder," not just "how would you do it."

---

**Q (Medium): The conversation sidebar and message thread are both independently scrollable with `overscroll-behavior: contain` correctly applied to each. A user reports that scrolling within the sidebar sometimes also scrolls the message thread panel next to it. Is this the same bug, and does the same fix apply?**

Answer: No — this is a different bug with a different cause, even though the symptom ("scrolling one region moves another") sounds superficially similar to scroll chaining. Scroll chaining specifically propagates *up the ancestor chain* (a nested scrollable element handing off to its own container), not sideways between unrelated sibling elements — the sidebar and message thread are siblings, not ancestor/descendant, so there's no scroll-chaining relationship between them at all, and `overscroll-behavior` has no mechanism that would cause one sibling's scroll to affect another regardless of its value. The actual cause is more likely one of: both containers sharing a common scrollable ancestor that's also, unexpectedly, scrollable itself (and scroll chaining from *both* siblings independently reaching that shared ancestor, which would look like "scrolling one affects the layout near the other" without them directly affecting each other); a JS scroll-sync feature (intentional or accidental) explicitly linking the two containers' scroll positions; or, more mundanely, a CSS layout bug where the two "independent" containers aren't actually two separate scroll containers at all (e.g., one is visually positioned to look separate but is actually the same overflow container as the other, or has `overflow: visible` inherited unexpectedly, meaning content isn't actually clipped/scrolled independently the way it appears to be).

The trap: assuming any two elements affecting each other's scroll must be a scroll-chaining case and reaching for `overscroll-behavior` again as a reflexive fix — chaining specifically requires an ancestor/descendant scroll-container relationship; a sibling-to-sibling interaction needs a structurally different diagnosis (shared ancestor, explicit JS coupling, or a layout bug), and proposing the same fix without recognizing this distinction signals pattern-matching on the symptom rather than understanding the actual mechanism.

---

**Q (Low): Does `overscroll-behavior: contain` have any effect on keyboard-driven scrolling (e.g., `Page Down`, arrow keys, `Space`) reaching an ancestor once the focused scrollable element hits its boundary?**

Answer: `overscroll-behavior` is specified in terms of scroll actions generally (its scope in the spec covers the scroll chaining and default overscroll-effect behaviors triggered by scroll input, which in practice is primarily wheel/touch/trackpad gesture-driven scrolling) — browser behavior for keyboard-driven scrolling at a boundary is a related but separately-governed area, and its interaction with `overscroll-behavior` has historically been less consistently specified/implemented across browsers than the wheel/touch case. In practice, keyboard scroll behavior at a contained element's boundary depends more directly on standard focus and `tabindex` semantics — if an element is focused and scrollable, `Page Down`/arrow keys typically scroll it, and once at its boundary, whether focus (and thus subsequent key presses) "moves on" to affect an ancestor is governed by normal DOM focus/event bubbling rules rather than being uniformly covered by `overscroll-behavior` the same way wheel/touch chaining explicitly is. This is exactly why the modal-background-scroll-lock approach (Step 3 above, using `body { overflow: hidden }` or a more complete `inert`/focus-trap solution) is the more robust guarantee for background inertness — it doesn't rely on `overscroll-behavior` covering every possible input method, it removes the background from the interactive/scrollable surface entirely for the duration the modal is open.

The trap: assuming `overscroll-behavior: contain` is a complete, input-method-agnostic guarantee that "nothing about scrolling this element ever affects anything outside it" — it's specifically strongest for wheel/touch gesture-driven scroll chaining, which is what this scenario's reported bug is actually about, but a fully robust modal implementation (as distinct from just fixing the reported wheel/touch bug) still needs the separate body-scroll-lock/inert layer to cover keyboard and other interaction paths completely.

---

## Self-Assessment

- [ ] Can define scroll chaining precisely and explain why it's a deliberate default, not a bug, in the general case
- [ ] Can apply `overscroll-behavior: contain`/`none` correctly scoped to the specific container that needs it, not a shared ancestor
- [ ] Can explain the practical difference between `contain` and `none` and choose deliberately between them
- [ ] Can explain why modal background scroll containment needs a body-lock/inert layer beyond just `overscroll-behavior`
- [ ] Can describe the JS wheel/touch-event fallback and name its specific correctness/performance pitfalls versus the CSS solution
- [ ] Can distinguish a genuine scroll-chaining bug (ancestor/descendant) from a sibling-to-sibling scroll-coupling bug with a different root cause

---
*Next: Dark Mode & Print Stylesheet Edge Cases — closes out Phase 8 by moving from interaction/scroll correctness to rendering-context correctness, where the same page needs to look right under color-scheme and output-medium conditions that are easy to forget to test at all.*
