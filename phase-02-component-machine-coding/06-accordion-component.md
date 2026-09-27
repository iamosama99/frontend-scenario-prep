# Accordion Component

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| `aria-expanded` | On the trigger `<button>`, toggled `true`/`false` | The single source of truth for "is this section open" for assistive tech and CSS alike |
| `aria-controls` | On the trigger, pointing at the panel's ID | Tells assistive tech which region this button expands, since visual proximity isn't perceivable |
| Single-open exclusivity | Not "close others as a side effect of opening one" — a real constraint enforced centrally | Prevents inconsistent state if items can also be toggled programmatically, not just by click |
| Animating unknown height | Animate `max-height` off `scrollHeight`, or CSS Grid `grid-template-rows: 0fr → 1fr` | `height: auto` cannot be animated — the browser can't interpolate to/from a keyword |
| Controlled vs. uncontrolled | Does the component own open/closed state, or does a parent pass `openItems` + `onChange`? | Determines whether the accordion can be driven by external state (URL, form validation) |

## The Scenario

"Build an accordion — a list of headers, click one to expand its panel below it. Support both a mode where only one section can be open at a time, and a mode where multiple can be open simultaneously. I also want the expand/collapse to actually animate, not just snap open."

## Clarifying Questions

- **In single-open mode, is closing all sections ever allowed, or must exactly one always be open?** This determines whether clicking the currently-open header should close it (toggle-to-empty allowed) or should be a no-op (always-one-open, like a classic FAQ accordion that defaults its first item open and never lets you close down to zero). Both are legitimate; I'd default to allowing full close since it's the more common expectation, but I'd confirm because it changes the click handler's logic, not just a config flag.
- **Should single-open be implemented as "opening one closes the others as a side effect," or as a hard mutual-exclusivity constraint?** This sounds like a semantic quibble but it isn't — if a section can also be opened programmatically (e.g., "jump to and expand section 3" from a search result), a "side effect of click" implementation that only closes others inside the click handler won't enforce exclusivity for that programmatic path, and you'll end up with two sections open at once via a code path nobody thought to update. I'd model `openItems` as a `Set` (or a single value in single-mode) and make every mutation path go through one function that enforces the constraint, not through the click handler specifically.
- **Does the animation need to work when a panel's content height is dynamic (e.g., text that reflows on window resize while the panel is open)?** If yes, that rules out hardcoding pixel heights in the animation and pushes toward either the `scrollHeight`-driven approach or CSS Grid's `fr`-unit trick, both of which tolerate content whose height isn't known until render time.
- **Is this component controlled or uncontrolled** — does it manage its own open/closed state internally, or does a parent own `openItems` and pass it in along with a change callback? This affects whether the accordion can be driven by state living outside it (a URL query param that should auto-open a specific FAQ, form-level "all sections must be visited" validation), which an uncontrolled, self-contained component can't support without an escape hatch.
- **Do keyboard users need Arrow-key navigation between headers, or is Tab-to-each-header plus Enter/Space sufficient?** The ARIA Disclosure pattern doesn't mandate arrow-key support (accordion headers are ordinary buttons in a normal Tab sequence, unlike tabs), but some accordion implementations add optional Up/Down/Home/End for parity with other composite widgets — worth confirming expected scope before over- or under-building.

## Approach & Trade-offs

**Trigger and panel wiring mirror the ARIA Disclosure pattern**, not the tabs pattern from the previous scenario — there's no `role="tablist"`/`role="tab"` here. Each header is a real `<button>` with `aria-expanded` (true/false) and `aria-controls` pointing at its panel's ID; the panel itself typically just needs `role="region"` with `aria-labelledby` back to the button if it should be independently discoverable by landmark navigation, though a plain `<div>` is acceptable when the accordion isn't meant to be a page-level navigation aid. Because triggers are ordinary buttons in normal document flow (not a composite single-tab-stop widget like tabs), Tab moves through every header individually — no roving tabindex needed here, which is a deliberate and meaningful difference from the tabs pattern.

**Single-open vs. multiple-open as a real constraint, not a side effect.** I model state as `openItems: Set<id>` regardless of mode. In multi-open mode, toggling an id just adds/removes it from the set. In single-open mode, every mutation goes through one function — `setOpen(id, isOpen)` — that, when opening an id, first clears the set (or replaces it with a singleton) before adding the new one; when a UI needs to open item 3 from *outside* a click (a search-result deep link, for example), it still calls the exact same function, so exclusivity is impossible to violate no matter which code path triggers a state change. This is the difference between "closing others is something the click handler happens to also do" (fragile — every future call site has to remember to replicate that logic) and "the state container itself cannot represent more than one open item in single-mode" (robust — the constraint lives in one place).

**Animating height when content height is unknown ahead of time.** `height: auto` cannot be transitioned — CSS transitions interpolate between two numeric values, and `auto` isn't a number the browser can pick a start or end point for mid-animation, so `transition: height 0.3s` combined with toggling `height: auto` either does nothing or snaps instantly, depending on the browser. Two real fixes: (1) measure the panel's actual content height via `element.scrollHeight` (which reports the full content height even while the element is visually clipped to less) and animate `max-height` (or `height`) from `0` to that measured pixel value, then optionally clear the inline height back to `auto` once the transition ends so subsequent content reflows/resizes aren't stuck at a stale pixel number; or (2) use CSS Grid: wrap the panel's content in a grid row track set to `grid-template-rows: 0fr`, transition that to `1fr` on open — `fr` units *are* animatable and the browser computes the actual rendered height per frame from the content's intrinsic size, so there's no manual measurement at all. I'd default to the Grid approach for new code since it needs no JS-side height measurement or `transitionend` cleanup, but the `scrollHeight` approach is the one every candidate should be able to produce from memory since it works in every browser without relying on `fr`-unit-on-`grid-template-rows` support for older targets.

**Controlled vs. uncontrolled API design.** An uncontrolled accordion (state lives inside the component, e.g., a `Set` in a closure or class field) is simpler to drop in and is the right default for a component with no need to be driven by outside state. A controlled accordion instead exposes `getState()`/`setState(openItems)`-style hooks (or, in a framework, accepts `openItems` + `onChange` as props and renders purely from what it's given) — the trade-off is that a controlled component pushes the responsibility of actually updating and re-passing state onto the caller for every interaction, in exchange for letting the caller be the single source of truth (so a URL param, or a "must open every section before Submit is enabled" validation rule, can legitimately own and drive the open/closed state instead of it being locked inside the widget).

## Solution

Markup:

```html
<div class="accordion">
  <h3>
    <button aria-expanded="false" aria-controls="panel-1" id="header-1">
      What's your return policy?
    </button>
  </h3>
  <div id="panel-1" role="region" aria-labelledby="header-1" class="panel" hidden>
    <div class="panel-inner">30 days, unworn, with a receipt.</div>
  </div>

  <h3>
    <button aria-expanded="false" aria-controls="panel-2" id="header-2">
      Do you ship internationally?
    </button>
  </h3>
  <div id="panel-2" role="region" aria-labelledby="header-2" class="panel" hidden>
    <div class="panel-inner">Yes, to most countries — rates vary at checkout.</div>
  </div>
</div>
```

Wrapping each button in a heading (`<h3>`) is deliberate — it lets screen reader users jump between accordion sections via heading navigation, which is how a lot of AT users skim a page.

State + exclusivity, single function as the only mutation path:

```javascript
class Accordion {
  #root;
  #mode; // 'single' | 'multiple'
  #openItems = new Set();

  constructor(root, { mode = 'multiple' } = {}) {
    this.#root = root;
    this.#mode = mode;
    this.#root.querySelectorAll('button[aria-controls]').forEach((btn) => {
      btn.addEventListener('click', () => this.toggle(btn));
    });
  }

  toggle(button) {
    const isOpen = button.getAttribute('aria-expanded') === 'true';
    this.#setOpen(button, !isOpen);
  }

  // The ONE path every state change goes through — click, or a future
  // programmatic call like accordion.open('panel-3'), both land here.
  #setOpen(button, shouldOpen) {
    const panelId = button.getAttribute('aria-controls');

    if (this.#mode === 'single' && shouldOpen) {
      // Enforce exclusivity centrally — not as a click-handler side effect.
      this.#root.querySelectorAll('button[aria-expanded="true"]').forEach((other) => {
        if (other !== button) this.#collapse(other);
      });
    }

    shouldOpen ? this.#expand(button, panelId) : this.#collapse(button);
  }

  #expand(button, panelId) {
    const panel = document.getElementById(panelId);
    button.setAttribute('aria-expanded', 'true');
    this.#openItems.add(panelId);
    animateOpen(panel);
  }

  #collapse(button) {
    const panelId = button.getAttribute('aria-controls');
    const panel = document.getElementById(panelId);
    button.setAttribute('aria-expanded', 'false');
    this.#openItems.delete(panelId);
    animateClose(panel);
  }
}
```

The `scrollHeight`-based animation, including the reset-to-`auto` cleanup that keeps later resizes correct:

```javascript
function animateOpen(panel) {
  panel.hidden = false;
  const targetHeight = panel.scrollHeight; // full content height, even while clipped
  panel.style.height = '0px';
  panel.style.overflow = 'hidden';
  // Force a layout flush so the browser registers the 0px start point
  // before we change it — otherwise the two style writes get batched
  // and there's nothing to transition from.
  panel.offsetHeight; // eslint-disable-line no-unused-expressions
  panel.style.transition = 'height 0.25s ease';
  panel.style.height = `${targetHeight}px`;

  panel.addEventListener('transitionend', function onEnd() {
    panel.style.height = 'auto'; // let it reflow naturally after animating
    panel.style.overflow = '';
    panel.removeEventListener('transitionend', onEnd);
  }, { once: true });
}

function animateClose(panel) {
  const startHeight = panel.scrollHeight;
  panel.style.height = `${startHeight}px`; // pin current height before animating from it
  panel.style.overflow = 'hidden';
  panel.offsetHeight;
  panel.style.transition = 'height 0.25s ease';
  panel.style.height = '0px';

  panel.addEventListener('transitionend', function onEnd() {
    panel.hidden = true;
    panel.style.transition = '';
    panel.removeEventListener('transitionend', onEnd);
  }, { once: true });
}
```

> **Check yourself:** Why does `animateClose` need to first set `height` to the *current* pixel value (`scrollHeight`) before transitioning to `0px`, instead of just transitioning straight from whatever the panel's current style is?

## Gotchas

**Trying to transition `height: auto` directly.** The most common first attempt — `panel.style.height = panel.classList.contains('open') ? 'auto' : '0'` inside a CSS-transitioned class toggle — does nothing visually, because the browser has no numeric start/end pair to interpolate; the change is either instant or entirely skipped depending on engine.

**Forgetting the layout-flush before starting the transition.** Setting `height: 0px` and `height: <target>px` back-to-back in the same tick, with no forced reflow between them, gets batched by the browser into a single style recalculation — there's no "before" state to animate from, so it snaps instead of transitioning. Reading a layout-triggering property (`offsetHeight`, `getBoundingClientRect()`, etc.) between the two writes forces the flush.

**Single-open enforced only inside the click handler.** If exclusivity logic lives in the click event listener rather than in the state-mutation function itself, any other code path that opens a section (a deep link, a "expand all" button that's supposed to be disabled in single-mode, a test) can silently produce two simultaneously open sections in a mode that's supposed to forbid it.

**Leaving `height` pinned to a stale pixel value after opening.** If the open-transition's `transitionend` handler doesn't reset `height` back to `auto`, the panel is stuck at whatever pixel height it happened to be when it finished animating — if the window resizes, or the content itself reflows (an image loads, a font swap changes line count), the panel either clips new content or leaves dead whitespace, because it's no longer sized to its actual content.

**Missing `aria-controls`/mismatched IDs when panels are dynamically generated** (e.g., accordion items built from a data array) — the same class of bug as in the tabs component: it fails silently for assistive tech only, so it's invisible without a dedicated audit.

## Follow-up Questions

**Q (High): Why can't you animate `height: auto` with a CSS transition, and what are the two standard workarounds?**

Answer: CSS transitions work by interpolating a property between two numeric (or otherwise interpolable) values over time; `auto` is a keyword the browser resolves to "whatever height the content needs," not a fixed number, so there's no defined numeric start or end point to interpolate between — browsers either skip the transition entirely or snap instantly. The two standard workarounds: measure the content's actual height via `element.scrollHeight` (which returns full content height even while the element is clipped/collapsed) and animate a real pixel value (`height` or `max-height`) from `0` to that number, cleaning up back to `auto` after the transition so later reflows aren't pinned to a stale number; or use CSS Grid's `grid-template-rows`, transitioning from `0fr` to `1fr` on a wrapper — `fr` units on grid tracks are genuinely animatable, and the grid layout algorithm computes the actual pixel height live from content, so no JS measurement step is needed at all.

The trap: proposing `max-height: 9999px` as a fixed large value to transition to/from instead of measuring `scrollHeight` — this "works" visually but the transition's perceived duration becomes wrong (a CSS transition's timing is computed against the full delta, from 0 to 9999px, even though the panel only ever visually reaches 200px — so a 0.3s transition to an actual 200px content height finishes in a small, non-obvious fraction of the stated duration, making the animation feel abrupt or inconsistent across panels of different content heights).

---

**Q (High): How do you enforce single-open exclusivity so it can't be violated by a future code path, not just the click handler?**

Answer: Route every state mutation — regardless of trigger (click, a programmatic "open section 3" call, a keyboard shortcut, a test) — through one function that owns the invariant, e.g., `setOpen(id, shouldOpen)`. When that function is asked to open an id while in single-mode, it closes every other currently-open id *inside itself*, before or as part of opening the new one, so the constraint is structurally impossible to bypass — there is no other way to mutate the open state. Compare this to putting "close the others" logic inside the click event listener only: that's correct for clicks, but a second entry point (a deep link that calls something like `panel.expand()` directly) that doesn't happen to also call the "close others" logic will produce two simultaneously-open sections in a mode that's supposed to forbid it, and the bug only surfaces once that second entry point actually gets used in production.

The trap: describing the fix as "just also close the others when opening one" without addressing *where* that logic lives — the correct answer is specifically about centralizing the mutation path, not merely remembering to add the close-others step somewhere.

---

**Q (High): What's the difference between a controlled and uncontrolled accordion, and when does it matter?**

Answer: An uncontrolled accordion owns its open/closed state internally — a `Set` in a closure, class field, or component instance state — and the caller only ever tells it "toggle this" or reads its current state after the fact; it's simpler to embed and requires no state-wiring from the parent. A controlled accordion instead renders purely from state the caller owns and passes in (`openItems`) plus a callback the caller must invoke to actually change that state (`onChange`) — the accordion component itself holds no state of its own for "what's open." This matters whenever something *outside* the accordion needs to be the source of truth for which sections are open: a URL query parameter that should deep-link directly to an expanded FAQ item, a wizard-style form that disables "Submit" until every section has been expanded (visited) at least once, or state that needs to survive the accordion component being unmounted and remounted. An uncontrolled accordion can't support any of those without bolting on an escape-hatch API; a controlled one supports them for free, at the cost of every interaction requiring a round-trip through the parent's state update.

The trap: presenting controlled as strictly "better" — for a simple, self-contained FAQ widget with no external state dependency, controlled is pure overhead (the parent has to wire up state and a change handler for no actual benefit); the right answer names the concrete external-state scenario that justifies the extra wiring, not a blanket preference.

---

**Q (Medium): Why wrap each accordion trigger button in a heading element (`<h3>`, etc.), and does the heading level matter?**

Answer: Screen reader users commonly navigate a page by jumping between headings (many screen readers bind a single key, like `H`, to "next heading") rather than reading linearly — wrapping each trigger in a heading makes every accordion section independently reachable that way, the same as it would be for a sighted user visually scanning down a list of bold section titles. The heading level does matter insofar as it should reflect the accordion's actual position in the page's heading outline (an accordion nested under an `<h2>` section should use `<h3>` for its items, not skip to `<h4>` or drop back to `<h2>`), since assistive tech also uses heading *level* to convey document structure/hierarchy, not just presence.

The trap: treating the heading wrapper as a nice-to-have rather than a real accessibility requirement — the WAI-ARIA Disclosure/Accordion pattern explicitly recommends this, and its absence is a common, specific finding in accessibility audits of hand-rolled accordions.

---

**Q (Medium): How would you support "expand all" / "collapse all" without breaking the single-open mode's invariant?**

Answer: "Expand all" is only meaningful in multiple-open mode — in single-open mode, it's a contradiction with the mode's own constraint, so the correct behavior is to disable or hide that control entirely when in single-open mode, not to silently reinterpret it as "expand the first one" or similarly paper over the conflict. Implementation-wise, "expand all" iterates every trigger and calls the same centralized `setOpen(id, true)` path used everywhere else, rather than a separate bespoke code path — reusing the one function that owns the invariant means there's no risk of a second, slightly-different opening mechanism drifting out of sync with the first.

The trap: implementing "expand all" as a special-cased loop that directly manipulates `aria-expanded` and panel visibility without going through the same state-mutation function as click-to-toggle — this duplicates logic (including the animation triggering) and is exactly the kind of second code path that later causes the single-open-mode exclusivity bug described above.

---

**Q (Medium): What accessibility issue arises if the panel's `hidden` attribute is removed/added in JS but the height animation's `transitionend` listener never fires (e.g., the element had `display: none` applied via a class before the transition could run)?**

Answer: If `hidden` (which applies `display: none`) is toggled at the same time an attempted CSS transition is supposed to run, the transition never plays at all — `display: none` removes the element from the render tree instantly, giving the browser no frames in which to interpolate anything, and any `transitionend` listener waiting to do cleanup (like resetting `height` back to `auto`) never fires, since no transition ever started. The panel either snaps open/closed with no animation, or — worse — is left in an inconsistent style state (a pinned pixel height with `display: none` layered on top) that causes a visible jump on next legitimate open. The fix is sequencing the `hidden` removal to happen *before* the height transition starts (on open) and only after the height transition completes (on close), exactly as shown in `animateOpen`/`animateClose` above.

The trap: applying `hidden` and starting the height transition in the same synchronous block without sequencing them correctly — a subtle ordering bug that only shows up as "the animation doesn't play" rather than throwing any error, making it easy to ship unnoticed.

---

**Q (Low): How would Arrow-key navigation between accordion headers work, and is it required by the ARIA spec?**

Answer: It's not required — the WAI-ARIA Authoring Practices' Accordion pattern treats headers as ordinary buttons navigable via Tab, unlike the Tabs pattern's mandatory roving-tabindex arrow-key model — but some implementations add it for consistency with other composite widgets: ArrowDown/ArrowUp move focus between headers (not open/close anything by themselves), Home/End jump to the first/last header, and Enter/Space toggle the currently focused header, mirroring native `<button>` behavior that would apply anyway. If added, it would be implemented the same way as the tabs component's keydown handler — intercepting arrow keys on the container, moving `.focus()` between header buttons — but without roving tabindex, since headers remain independently Tab-reachable regardless.

The trap: assuming arrow-key support is mandatory for every accordion to be "accessible" — citing it as a hard requirement when the actual ARIA APG pattern lists it as optional/author's-choice is a specific, checkable factual error.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can write correct accordion markup (`aria-expanded`, `aria-controls`, heading-wrapped triggers) from memory
- [ ] Can explain why `height: auto` can't be CSS-transitioned and implement the `scrollHeight` workaround
- [ ] Can implement single-open exclusivity as a centralized state constraint, not a click-handler side effect
- [ ] Can explain the controlled vs. uncontrolled trade-off with a concrete scenario that requires controlled
- [ ] Can explain why forcing a layout flush (reading `offsetHeight`) between two style writes is necessary
- [ ] Can state why heading-wrapped triggers matter for screen reader heading navigation

---
*Next: Accessible Combobox / Dropdown — another disclosure pattern, but combined with a text input and a much larger ARIA surface area.*
