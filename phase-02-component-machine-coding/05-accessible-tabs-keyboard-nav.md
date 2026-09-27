# Accessible Tabs — Keyboard Navigation

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| ARIA roles | `role="tablist"` (container) → `role="tab"` (each trigger) → `role="tabpanel"` (each content region) | Screen readers announce this as a tab widget instead of a generic list of buttons and divs |
| Tab↔panel linkage | Each tab has `aria-controls="panel-id"`; each panel has `aria-labelledby="tab-id"` | Lets assistive tech announce "Billing tab, 2 of 4, controls Billing panel" |
| Roving `tabindex` | Only the active tab is `tabindex="0"`; every other tab is `tabindex="-1"` | Tab key enters/exits the whole tablist in **one** stop, not once per tab; arrow keys move *within* it |
| Arrow keys | ArrowLeft/ArrowRight (or Up/Down for vertical tabs) move focus between tabs, wrapping at the ends | Matches native OS tab-strip behavior users already expect |
| Automatic vs. manual activation | Automatic: arrow key immediately switches the panel. Manual: arrow key only moves focus; Enter/Space activates | Manual is required when switching panels is expensive (network fetch, heavy re-render) |

## The Scenario

"Build a tabs component — a row of tab buttons and a content panel below that swaps based on which tab is selected. It needs to be keyboard accessible, so I want to see real ARIA attributes, not just `<div onclick>`. Plain JS and HTML, no framework."

## Clarifying Questions

- **Should arrow keys switch the panel immediately, or just move focus, requiring Enter/Space to activate?** This is the single most consequential design decision in the whole component, and WAI-ARIA explicitly names both as valid patterns. I'd default to automatic activation (arrow key switches immediately) since it's more common and feels snappier for cheap content, but I'd ask — if a tab panel does something expensive on activation (fires a network request, mounts a heavy chart), manual activation is the right call so a user tapping through tabs with the arrow keys doesn't fire five requests they didn't mean to commit to.
- **Is this a horizontal or vertical tablist?** It changes which arrow keys are the "next/previous" pair — Left/Right for horizontal, Up/Down for vertical (per the WAI-ARIA Authoring Practices) — and doing it backwards is a common, easily-caught mistake.
- **Do the tabs need to wrap (Right on the last tab goes to the first)?** Yes by convention — native OS tab strips and every reference implementation wrap, and a non-wrapping tablist feels broken to a keyboard user who expects a ring, not a line.
- **Can panel content contain form inputs that should preserve state when the user switches away and back?** This decides whether inactive panels can be safely removed from the DOM or must stay mounted and merely hidden — removing a panel with an in-progress, uncommitted text field silently discards the user's input.
- **Is there a maximum number of tabs, or could this list be dynamically generated (e.g., one tab per open document)?** If tabs can be added/removed at runtime, the roving-tabindex bookkeeping and `aria-controls`/`aria-labelledby` ID generation need to work for a dynamic list, not just a fixed markup block.

## Approach & Trade-offs

**Structure first.** Three ARIA roles map directly onto three visual pieces: the row of buttons is `role="tablist"`, each button is `role="tab"`, and each content region is `role="tabpanel"`. I use real `<button>` elements for tabs rather than `<div role="tab">` — buttons are focusable and keyboard-activatable by default, and even though roving tabindex overrides the native Tab-key stop-at-every-button behavior, starting from a real button means I don't have to hand-roll Enter/Space activation semantics I'd get for free otherwise, and screen reader + browser combinations are more consistent with real interactive elements under ARIA roles than with divs.

**Roving tabindex, and why not just leave every tab at `tabindex="0"`.** If every tab were natively focusable, pressing Tab from outside the widget would require pressing Tab once per tab to get past the whole tablist — for a tablist with eight tabs, that's eight Tab presses just to reach the content after it, which is exactly the kind of keyboard-trap-adjacent friction WCAG 2.4.3 (focus order) is meant to prevent. Roving tabindex fixes this: exactly one tab (the currently active/selected one) has `tabindex="0"` and is the single Tab stop for the whole widget; every other tab has `tabindex="-1"` (focusable programmatically, but skipped by sequential Tab navigation). Moving *within* the tablist becomes the arrow keys' job, not Tab's.

**Automatic vs. manual activation — the trade-off, not just the definitions.** Automatic activation (WAI-ARIA's simpler pattern) ties `aria-selected`, the roving tabindex, and the visible panel swap all to focus movement — arrow key press = new tab focused = new tab selected = new panel shown, all in one step. This is fine and fast when panel content is cheap to show. Manual activation decouples "which tab has focus" from "which tab/panel is selected" — arrow keys move focus and update visual highlight of the *focused* tab, but `aria-selected`/panel-swap only happen on Enter, Space, or (for mouse users) click. I'd pick manual specifically when a panel activation is expensive (a fetch, a heavy chart mount) so a user arrowing past three tabs to reach the fourth doesn't trigger three throwaway loads along the way.

**Hiding inactive panels: `hidden` attribute vs. `display:none` vs. removing from the DOM.** I used the native `hidden` attribute (equivalent to `display:none` under the hood, but semantic and free of any need for a CSS rule) rather than manually toggling a `display:none` class, because it's declarative and matches what `hidden` is for. Neither `hidden` nor `display:none` destroys the panel's DOM subtree or its state — an `<input>` inside an inactive panel keeps whatever the user typed, because the element is merely not rendered, not unmounted. Fully removing inactive panels from the DOM (and re-inserting on activation) would reset any live form state, uncommitted scroll position, or focus inside that panel — that's a real regression for any tab whose content is a form, so I'd only remove-from-DOM for tabs whose content is cheap to regenerate and has no state worth preserving (e.g., a tab that just re-renders a read-only list from a store on every mount).

## Solution

Markup — the ID wiring between tabs and panels is the part interviewers check most carefully:

```html
<div class="tabs">
  <div role="tablist" aria-label="Account settings">
    <button role="tab" id="tab-profile" aria-controls="panel-profile" aria-selected="true" tabindex="0">Profile</button>
    <button role="tab" id="tab-billing" aria-controls="panel-billing" aria-selected="false" tabindex="-1">Billing</button>
    <button role="tab" id="tab-security" aria-controls="panel-security" aria-selected="false" tabindex="-1">Security</button>
  </div>

  <div role="tabpanel" id="panel-profile" aria-labelledby="tab-profile" tabindex="0">…</div>
  <div role="tabpanel" id="panel-billing" aria-labelledby="tab-billing" tabindex="0" hidden>…</div>
  <div role="tabpanel" id="panel-security" aria-labelledby="tab-security" tabindex="0" hidden>…</div>
</div>
```

Note `tabindex="0"` on the panels themselves — without it, a keyboard user who activates a tab whose panel has no focusable content inside (e.g., static text) has nowhere for focus to logically land if you choose to move focus into the panel on activation; giving the panel itself a tab stop makes it announce and be reachable regardless of what's inside.

Behavior, built incrementally. First, the tab list and activation function:

```javascript
const tablist = document.querySelector('[role="tablist"]');
const tabs = Array.from(tablist.querySelectorAll('[role="tab"]'));

function activateTab(tab, { moveFocus = true } = {}) {
  tabs.forEach((t) => {
    const isActive = t === tab;
    t.setAttribute('aria-selected', String(isActive));
    t.tabIndex = isActive ? 0 : -1; // roving tabindex
    document.getElementById(t.getAttribute('aria-controls')).hidden = !isActive;
  });
  if (moveFocus) tab.focus();
}
```

Click handling is the easy case — no activation-mode distinction, since a click is always an explicit "activate this" gesture:

```javascript
tabs.forEach((tab) => {
  tab.addEventListener('click', () => activateTab(tab));
});
```

Keyboard handling is where automatic vs. manual activation actually diverges:

```javascript
const ACTIVATION_MODE = 'automatic'; // or 'manual'

tablist.addEventListener('keydown', (e) => {
  const currentIndex = tabs.indexOf(document.activeElement);
  if (currentIndex === -1) return;

  let newIndex = null;
  if (e.key === 'ArrowRight') newIndex = (currentIndex + 1) % tabs.length;
  else if (e.key === 'ArrowLeft') newIndex = (currentIndex - 1 + tabs.length) % tabs.length;
  else if (e.key === 'Home') newIndex = 0;
  else if (e.key === 'End') newIndex = tabs.length - 1;

  if (newIndex !== null) {
    e.preventDefault(); // stop the page from scrolling on arrow/Home/End
    const nextTab = tabs[newIndex];
    if (ACTIVATION_MODE === 'automatic') {
      activateTab(nextTab); // arrow key = immediate switch
    } else {
      tabs.forEach((t) => (t.tabIndex = t === nextTab ? 0 : -1));
      nextTab.focus(); // focus moves, but aria-selected/panel do NOT change yet
    }
    return;
  }

  if (ACTIVATION_MODE === 'manual' && (e.key === 'Enter' || e.key === ' ')) {
    e.preventDefault();
    activateTab(document.activeElement, { moveFocus: false });
  }
});
```

> **Check yourself:** In manual activation mode, after arrowing to a new tab but before pressing Enter, what does `aria-selected` say, and what is announced to a screen reader user versus a sighted keyboard user watching the visual highlight?

## Gotchas

**Every tab left at `tabindex="0"`.** This is the single most common failure — it technically "works" for mouse users and even superficially for keyboard users (Tab does eventually reach every tab), but it means the tablist consumes one Tab stop *per tab* instead of one stop for the whole widget, which is a real WCAG focus-order/efficiency problem, not a cosmetic one.

**Forgetting `e.preventDefault()` on ArrowUp/ArrowDown for vertical tabs (or Home/End generally).** The browser's default behavior for those keys is page scroll; without suppressing it, pressing End to jump to the last tab also scrolls the whole page to the bottom.

**Missing or backwards `aria-controls`/`aria-labelledby`.** If the IDs don't actually resolve to real elements (a typo, or IDs generated inconsistently for dynamic tabs), the linkage silently does nothing — screen readers fail gracefully and just don't announce the relationship, so this bug is invisible unless you specifically test with a screen reader or an axe-core-style audit.

**Automatic activation on expensive panels.** Wiring automatic activation to a tab whose panel triggers a network fetch means arrowing through tabs fires one request per tab passed through — a real, user-visible performance and backend-load problem that's easy to miss in a demo with instant, static content.

**Discarding form state by removing inactive panels from the DOM.** Toggling panels via full DOM removal/reinsertion instead of `hidden` resets any uncommitted input inside them — a classic "worked in the demo, broke in the bug report" gap.

## Follow-up Questions

**Q (High): Explain roving tabindex — why not just make every tab `tabindex="0"`, or every tab `tabindex="-1"`?**

Answer: All tabs at `tabindex="0"` means the browser's native sequential Tab-key navigation stops at every single tab, so getting from before the tablist to the content after it costs one Tab press per tab — a real, WCAG-relevant focus-order inefficiency for anything beyond two or three tabs, and it also means Tab and arrow keys do overlapping, confusing things. All tabs at `tabindex="-1"` makes the entire tablist unreachable by keyboard at all, since `-1` removes an element from sequential navigation entirely and it can only receive focus programmatically. Roving tabindex splits the difference correctly: exactly one tab (the active one) is `tabindex="0"` — the single entry point Tab lands on — and every other tab is `tabindex="-1"`, reachable only via the widget's own internal arrow-key handler, which manually calls `.focus()`. Tab now moves focus in and out of the whole widget in one stop each way; arrow keys own movement within it.

The trap: describing roving tabindex as "focus management" in the abstract without being able to state the concrete tabIndex values before and after an arrow key press — interviewers will ask you to trace it.

---

**Q (High): What's the difference between automatic and manual activation, and when is manual the right choice?**

Answer: In automatic activation, moving focus to a new tab (via arrow keys) immediately also selects it — `aria-selected` flips and the panel swaps in the same step as the focus move, so focus-state and selection-state are the same thing. In manual activation, arrow keys only move focus between tabs (updating tabindex and visual focus ring) without touching `aria-selected` or the visible panel; the user must explicitly press Enter or Space (or click) to commit the activation. Manual is the right choice whenever activating a tab has a real cost — firing a network request, mounting an expensive component, running a heavy computation — because with automatic activation, a keyboard user pressing ArrowRight three times to reach the fourth tab silently fires three throwaway activations along the way.

The trap: treating automatic activation as strictly "better UX" — for cheap content it is snappier, but calling it the universally correct default without naming the expensive-content counter-case misses exactly what the WAI-ARIA spec calls out both patterns for.

---

**Q (High): How do you decide between `hidden`, `display: none`, and removing panels from the DOM entirely?**

Answer: `hidden` (the HTML attribute) and `display: none` are functionally equivalent — both remove the element from the render tree and from the accessibility tree — but `hidden` is declarative, requires no accompanying CSS rule to work, and clearly signals intent in markup, so it's the right default. Neither approach un-mounts the subtree: any `<input>`'s value, any `textarea`'s scroll position, any component-local state inside a hidden panel persists exactly as-is and reappears when `hidden` is removed. Removing the panel from the DOM entirely (and re-creating it on activation) is the only option that actually discards that state — appropriate only when the panel's content is cheap to regenerate and there's genuinely nothing worth preserving, such as a tab that always re-renders a fresh, stateless view from a store.

The trap: choosing DOM removal "for performance" without checking whether any panel contains a form or other stateful input — the very first bug report on that implementation is a user's half-filled form vanishing when they tab away and back.

---

**Q (Medium): Why link tabs and panels with `aria-controls` and `aria-labelledby` instead of just relying on visual proximity?**

Answer: Screen reader users don't perceive visual proximity — a sighted user infers "this panel belongs to that tab" because it's directly below it, but that spatial relationship isn't exposed to assistive technology unless it's stated explicitly in the accessibility tree. `aria-controls` on the tab points to the panel's ID, telling assistive tech (and browsers that surface it) which region this tab expands/reveals; `aria-labelledby` on the panel points back to the tab's ID, so the panel is announced with the tab's own text as its accessible name (e.g., "Billing panel" rather than an unlabeled region). Without this pair, a screen reader user navigating tab-by-tab has no reliable way to know a given button controls a specific content region at all.

The trap: only wiring `aria-labelledby` (panel → tab) and skipping `aria-controls` (tab → panel), or vice versa — the relationship is genuinely bidirectional in the pattern and both attributes carry distinct information (the panel's accessible name comes from `aria-labelledby`; the "this button controls that region" relationship comes from `aria-controls`).

---

**Q (Medium): How would you support a dynamically changing set of tabs — e.g., tabs that can be added or closed at runtime, like browser tabs?**

Answer: The core mechanics (roving tabindex, arrow-key wrap, `aria-controls`/`aria-labelledby`) don't change, but the bookkeeping has to be recomputed against the live DOM rather than a fixed array captured once — `tabs` should be re-queried (or the list actively maintained) whenever a tab is added/removed, and IDs need a reliable generation scheme (e.g., a counter or the underlying data's own stable ID) rather than hardcoded strings, since two tabs can't share `tab-profile` as an ID if the same "kind" of tab can be opened twice. Closing the currently-active tab also needs an explicit fallback rule — typically activate the tab that's now in the same index position, or the previous tab if the closed one was last — otherwise focus and `aria-selected` end up pointing at a tab that no longer exists.

The trap: hardcoding tab/panel IDs as string literals in a way that only works for a fixed, known-in-advance tab set — the moment tabs become dynamic, ID collisions or stale references appear.

---

**Q (Medium): Why use real `<button>` elements for tabs instead of `<div role="tab">`?**

Answer: A `<button>` is natively focusable, natively responds to Enter/Space activation, and is exposed with sensible default semantics before any ARIA is even added — using it as the base means the only things left to layer on top are the tab-specific overrides (the role itself, `aria-selected`, the roving tabindex). A `<div role="tab">` starts with none of that: it needs `tabindex` handling from scratch even to be reachable, and Enter/Space activation has to be hand-implemented via keydown handlers, which is easy to get subtly wrong (e.g., forgetting that native buttons also fire `click` on Space *release*, not press). Starting from the right native element and only overriding what genuinely needs overriding (native buttons don't natively support roving tabindex, so that part is always custom) is the more robust default per the "use the platform" principle in ARIA guidance itself — ARIA roles override semantics but never grant behavior you didn't otherwise implement.

The trap: assuming `role="tab"` on any element automatically grants keyboard behavior — ARIA attributes are announced to assistive tech, they don't make an element behave any differently to the browser's default event/focus handling; all the actual keyboard logic here is still hand-written regardless of the base element.

---

**Q (Low): How would this change for a vertical tablist (tabs stacked on the left, like a settings sidebar)?**

Answer: Two changes: `aria-orientation="vertical"` on the `tablist` element (so assistive tech announces the correct navigation model), and swapping the "next/previous" key pair from ArrowRight/ArrowLeft to ArrowDown/ArrowUp, per the WAI-ARIA Authoring Practices' convention that arrow keys should match the tabs' visual/spatial layout. Home/End, wrapping behavior, roving tabindex, and the `aria-controls`/`aria-labelledby` linkage are all unchanged — only the orientation attribute and which two arrow keys are bound to "move" differ.

The trap: leaving ArrowLeft/ArrowRight bound for a vertically-stacked tablist "because the logic already works" — it functions, but it's disorienting for a keyboard user whose mental model of vertically-stacked items is Up/Down, and it also means you never set `aria-orientation`, which is the one attribute that tells assistive tech which pair of keys to expect in the first place.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can write the full tab/panel markup with correct `role`, `aria-selected`, `aria-controls`, `aria-labelledby` wiring from memory
- [ ] Can implement roving tabindex and explain why leaving every tab at `tabindex="0"` is wrong
- [ ] Can implement both automatic and manual activation and state the trade-off that decides between them
- [ ] Can implement Home/End and wrapping ArrowLeft/ArrowRight navigation, including `preventDefault` on scroll-triggering keys
- [ ] Can explain why `hidden` preserves panel state while DOM removal does not
- [ ] Can adapt the pattern to a vertical tablist (orientation + key pair swap)

---
*Next: Accordion Component — a closely related disclosure pattern, but multiple sections can (optionally) be open at once instead of exactly one panel being visible.*
