# Accessible Modal With Focus Trap

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Semantics | `role="dialog"` + `aria-modal="true"` + `aria-labelledby` pointing at the heading | Tells assistive tech "this is a modal, everything outside it is temporarily irrelevant" |
| Focus on open | Move focus to the first focusable element (or a designated initial-focus target) | A modal that opens without moving focus leaves a keyboard/screen reader user still "inside" the page behind it |
| Focus on close | Restore focus to the element that triggered the modal | Without this, focus silently resets to `<body>`, disorienting keyboard users |
| Focus trap | Intercept Tab/Shift+Tab, cycle only within the modal's focusable elements, wrap at both ends | Without it, Tab can walk focus into page content sitting visually behind/under the modal |
| Hide the rest of the page | `aria-hidden="true"` (or `inert`) on sibling content while open | A screen reader's virtual cursor can otherwise still "read into" background content that's visually obscured |
| Escape + backdrop click | Escape always closes; backdrop click closes only if configured to | Escape is a near-universal expectation; forced backdrop-dismiss can accidentally discard destructive-action confirmations |

## The Scenario

"Build me a modal dialog component — plain JS, no framework. It needs to trap focus so Tab doesn't escape it, close on Escape, and restore focus properly when it closes. I also want it to be genuinely accessible, not just visually correct — assume this will actually be audited."

## Clarifying Questions

- **Should clicking the backdrop close the modal, or only explicit close actions (X button, Cancel, Escape)?** This isn't purely a UX preference — for a modal confirming a destructive action ("delete this permanently?"), an accidental backdrop click closing the dialog (and being interpreted as either confirm or cancel, depending on implementation) can be a real, harmful bug. I'd make this configurable per-instance rather than hardcoded either way.
- **Is there a specific element that should receive initial focus, or is "the first focusable element" always correct?** The first focusable element is a reasonable default, but for a destructive-confirmation dialog, autofocusing the primary "Confirm delete" button is actively dangerous (a user who presses Enter reflexively, expecting to dismiss a notice, triggers the destructive action instead) — I'd want the API to accept an explicit initial-focus target, defaulting to first-focusable only when none is given.
- **Can modals be nested (a confirmation dialog opened from within another modal)?** This changes the design meaningfully — a single global "the trigger to restore focus to" variable breaks the moment a second modal opens on top of a first, since closing the inner one needs to restore focus to the outer modal's context, not the original page trigger. I'd ask because it changes whether I need a stack rather than a single reference.
- **Does content behind the modal need to be visually dimmed/inert, or just accessibility-hidden?** These are related but distinct — visual dimming is CSS; making it non-interactive and non-announced to assistive tech is a separate mechanism (`inert` or `aria-hidden` plus disabling pointer events), and skipping the latter means a screen reader user can still navigate into background content that's supposed to be temporarily unreachable.
- **Should body scroll be locked while the modal is open?** Almost always yes for a true modal (scrolling the page behind an open dialog is disorienting and can scroll the dialog's trigger out of view before the dialog even closes), but worth confirming since some "modal-like" panels are intentionally designed to let background scroll continue.

## Approach & Trade-offs

**Semantics first.** `role="dialog"` (or `role="alertdialog"` for something requiring immediate acknowledgment, like a destructive confirmation) with `aria-modal="true"` tells assistive tech this is a modal context — `aria-modal="true"` is specifically what signals that content outside the dialog should be treated as inert/unreachable by the accessibility tree, which is the semantic counterpart to the focus trap being the *interactive* enforcement of the same idea. `aria-labelledby` pointing at the dialog's heading element gives it an accessible name, rather than relying on generic "dialog" announcements with no context about *which* dialog.

**Focus management is two separate moments, not one feature: focus-in and focus-out.** On open, focus must move *into* the dialog — leaving it on whatever page element had focus before (or worse, resetting to `<body>`) means a screen reader or keyboard user has no indication anything changed, and continues interacting with content that's now supposed to be inert. On close, focus must move back to the element that *triggered* the dialog's opening — not to `<body>`, not to the top of the page — because that's the only way a keyboard user's mental model ("I was on this button, I opened a dialog, I closed it") stays intact. I chose to capture the trigger element (`document.activeElement` at the moment the modal opens) explicitly, rather than relying on browser-default focus-restoration behavior, because there is no such reliable default — the browser does not automatically remember and restore focus across arbitrary DOM changes.

**The focus trap itself: intercept Tab, don't try to prevent focus from moving via other means.** The robust approach is a `keydown` listener on the dialog checking for `Tab`/`Shift+Tab`, computing the dialog's current list of focusable elements, and when focus is on the *last* focusable element and the user presses Tab (without Shift), calling `preventDefault()` and manually moving focus to the *first* focusable element — and the mirror case for Shift+Tab on the first element wrapping to the last. I considered (and rejected) the alternative of setting `tabindex="-1"` on every element outside the modal — this technically works but is invasive (mutating potentially large amounts of unrelated DOM state, and easy to forget to fully revert), versus intercepting Tab at the dialog boundary, which is scoped entirely to the dialog's own listener and doesn't touch anything outside it.

**Hiding the rest of the page from assistive tech: `aria-hidden="true"` on siblings vs. the newer `inert` attribute.** `aria-hidden="true"` on the page's other top-level containers removes them from the accessibility tree (so a screen reader's virtual cursor can't navigate into them) but does *not* by itself prevent mouse/pointer interaction or keyboard focus via Tab — you'd still need `pointer-events: none` and rely on the focus trap for the keyboard side. `inert` (now broadly supported) is the more complete, purpose-built solution: it simultaneously removes an element from the accessibility tree, makes it unfocusable, and makes it unclickable, in one attribute. I'd default to `inert` where supported, since it's a single mechanism that correctly covers all three concerns `aria-hidden` alone requires stitching together manually.

**Escape vs. backdrop click as distinct, independently-configurable dismiss mechanisms.** Escape closing the dialog is a near-universal platform convention users rely on without thinking about it, and I'd treat it as always-on. Backdrop-click-to-close is a UX nicety that's actively wrong for some dialogs (destructive confirmations, forms with unsaved input where an accidental outside click shouldn't silently discard work) — so it should be a configuration option (`closeOnBackdropClick: boolean`, defaulting to `true` for informational/non-destructive dialogs), not a hardcoded behavior baked into the component.

## Solution

Markup — the dialog, its backdrop, and the rest of the page as siblings that can be marked inert:

```html
<div id="page-content">
  <button id="open-modal-btn">Delete account</button>
  <!-- rest of the real page content -->
</div>

<div id="modal-backdrop" hidden></div>
<div
  id="modal"
  role="dialog"
  aria-modal="true"
  aria-labelledby="modal-title"
  hidden
>
  <h2 id="modal-title">Delete your account?</h2>
  <p>This can't be undone.</p>
  <button id="modal-cancel-btn">Cancel</button>
  <button id="modal-confirm-btn">Delete</button>
</div>
```

Core state and the focusable-elements query, factored out since both opening (initial focus) and the trap (Tab cycling) need it:

```javascript
const FOCUSABLE_SELECTOR =
  'a[href], button:not([disabled]), textarea:not([disabled]), input:not([disabled]), select:not([disabled]), [tabindex]:not([tabindex="-1"])';

let triggerElement = null; // who opened the modal, to restore focus to on close

function getFocusableElements(container) {
  return Array.from(container.querySelectorAll(FOCUSABLE_SELECTOR)).filter(
    (el) => el.offsetParent !== null // excludes hidden/display:none elements
  );
}
```

Opening — capture the trigger, hide the rest of the page, move focus in, lock scroll:

```javascript
const modal = document.getElementById('modal');
const backdrop = document.getElementById('modal-backdrop');
const pageContent = document.getElementById('page-content');

function openModal({ initialFocusEl } = {}) {
  triggerElement = document.activeElement; // capture BEFORE moving focus anywhere

  pageContent.inert = true; // one attribute: unfocusable, unclickable, hidden from a11y tree
  backdrop.hidden = false;
  modal.hidden = false;
  document.body.style.overflow = 'hidden'; // lock background scroll

  const focusTarget = initialFocusEl ?? getFocusableElements(modal)[0];
  focusTarget?.focus();

  document.addEventListener('keydown', onKeydown);
  if (closeOnBackdropClick) backdrop.addEventListener('click', closeModal);
}
```

Closing — reverse every side effect opening caused, and restore focus last:

```javascript
function closeModal() {
  modal.hidden = true;
  backdrop.hidden = true;
  pageContent.inert = false;
  document.body.style.overflow = '';

  document.removeEventListener('keydown', onKeydown);
  backdrop.removeEventListener('click', closeModal);

  triggerElement?.focus(); // restore focus to wherever the user actually was
  triggerElement = null;
}
```

The keydown handler — Escape closes unconditionally; Tab/Shift+Tab implement the trap:

```javascript
function onKeydown(e) {
  if (e.key === 'Escape') {
    closeModal();
    return;
  }

  if (e.key !== 'Tab') return;

  const focusable = getFocusableElements(modal);
  if (focusable.length === 0) return;

  const first = focusable[0];
  const last = focusable[focusable.length - 1];

  if (e.shiftKey && document.activeElement === first) {
    e.preventDefault();
    last.focus(); // wrap backward: Shift+Tab off the first element goes to the last
  } else if (!e.shiftKey && document.activeElement === last) {
    e.preventDefault();
    first.focus(); // wrap forward: Tab off the last element goes to the first
  }
  // Any Tab press NOT at a boundary is left alone — the browser's default
  // Tab behavior already moves focus correctly within the modal's own elements.
}
```

Wiring it up:

```javascript
const closeOnBackdropClick = true; // per-instance config, per the clarifying question above

document.getElementById('open-modal-btn').addEventListener('click', () => openModal());
document.getElementById('modal-cancel-btn').addEventListener('click', closeModal);
document.getElementById('modal-confirm-btn').addEventListener('click', () => {
  // perform the actual delete, then close
  closeModal();
});
```

> **Check yourself:** Trace what happens if `triggerElement` is captured *after* focus has already moved into the modal, instead of before — why does the order of operations in `openModal()` matter here specifically?

## Nested Modals

A single `triggerElement` variable breaks the instant a second modal opens on top of a first (a confirmation dialog launched from within another dialog) — closing the inner one would restore focus to whatever opened the *outer* one, skipping past the outer modal entirely and likely landing focus on background page content that's currently `inert`. The fix is a stack, not a single reference:

```javascript
const modalStack = []; // each entry: { modalEl, triggerElement }

function openModal(modalEl, { initialFocusEl } = {}) {
  const trigger = document.activeElement;
  modalStack.push({ modalEl, triggerElement: trigger });
  // ... same inert/focus/scroll-lock logic, scoped to modalEl ...
}

function closeTopModal() {
  const top = modalStack.pop();
  if (!top) return;
  // ... hide top.modalEl ...
  top.triggerElement?.focus(); // restores focus to the PREVIOUS modal's context, not page 1
  // Only remove page-level `inert`/scroll-lock once the stack is fully empty.
  if (modalStack.length === 0) {
    pageContent.inert = false;
    document.body.style.overflow = '';
  }
}
```

The two things this changes versus the single-modal version: (1) `Escape`/the focus trap should only ever act on the *topmost* modal — a `keydown` listener attached per-modal-instance (removed on that instance's close) naturally achieves this, since the most-recently-added listener is what's live; (2) page-level side effects (`inert`, scroll lock) should only be reverted when the *entire stack* empties, not on every individual modal close, since an outer modal is still open and still needs the page behind it inert.

## Gotchas

**Capturing the trigger element too late.** If `document.activeElement` is read *after* focus has already been programmatically moved into the modal, it captures an element inside the modal itself (or `<body>`), not the real trigger — the capture must happen as the very first step of `openModal()`, before any focus-moving side effect.

**`aria-hidden` on background content without also disabling pointer events and focusability.** `aria-hidden="true"` alone only affects the accessibility tree; it does nothing to stop a sighted mouse user from clicking through to a visually-dimmed-but-still-interactive background button, or a keyboard user from Tab-ing into it if the focus trap has a gap. `inert` is the more complete single mechanism; if using `aria-hidden` instead (e.g., for broader browser support in an older codebase), it needs to be paired with `pointer-events: none` and the focus trap doing its job correctly.

**Forgetting to remove the `keydown` listener (and other side effects) on close.** A modal that adds a document-level `keydown` listener on open but never removes it on close leaks a listener per open/close cycle, and worse, a stale listener from a supposedly-closed modal can still intercept Escape/Tab meant for whatever's now in focus.

**Autofocusing a destructive action by defaulting to "first focusable element" without exception.** For a "Delete this?" confirmation, if the Delete button happens to be the first focusable element and initial focus isn't deliberately overridden, a user pressing Enter out of habit (expecting to dismiss an informational dialog) triggers the destructive action instead of cancelling.

**Not locking body scroll, or locking it without accounting for scrollbar width changes causing layout shift.** Setting `overflow: hidden` on `<body>` removes the scrollbar, which on most desktop browsers shrinks the visible content area's width slightly, causing a small horizontal layout shift the instant the modal opens — a fully polished implementation compensates by measuring the scrollbar width and adding it as `padding-right` on `<body>` while scroll is locked.

**Single-`triggerElement` design breaking under nested modals.** Covered above — this is exactly the kind of thing that works fine in a single-modal demo and breaks the first time it's actually composed with itself, which is why interviewers frequently ask about nesting as a follow-up even if the original prompt didn't mention it.

## Follow-up Questions

**Q (High): Walk through exactly how the focus trap works — what specifically happens on Tab at the last element and Shift+Tab at the first, and why intercept at the boundary rather than on every Tab press?**

Answer: The trap listens for `keydown` and only acts when the key is `Tab`. On every such press, it checks whether the currently focused element (`document.activeElement`) is the *last* focusable element in the modal and the press is a plain Tab (no Shift) — if so, it calls `preventDefault()` (stopping the browser's default "move to the next focusable element in document order," which would otherwise leave the modal entirely) and manually focuses the *first* focusable element instead, creating the wrap-around. The mirror case: Shift+Tab while focus is on the *first* element wraps back to the *last*. Every other Tab press (anywhere in the middle of the modal's focusable elements) is deliberately left alone, because the browser's native Tab behavior already correctly moves focus to the next/previous element within the modal — intercepting only at the two boundary conditions is both sufficient and far simpler than trying to fully reimplement Tab traversal.

The trap: implementing the trap by intercepting and manually handling *every* Tab press (computing and calling `.focus()` on the next element yourself, always) — this works but is needlessly fragile (it has to perfectly replicate what the browser already does correctly for the non-boundary case, including correctly handling elements that become focusable/unfocusable dynamically) versus just letting the browser handle the interior case and only intervening exactly at the two edges.

---

**Q (High): Why must focus be restored to the *trigger* element specifically on close, rather than just letting focus go wherever it naturally lands?**

Answer: There is no natural, correct "wherever it lands" — once the modal's DOM is hidden/removed, if nothing explicitly manages focus, most browsers reset `document.activeElement` to `<body>`, which is silently and totally disorienting for a keyboard-only or screen-reader user: from their perspective, they were focused on a specific button, opened a dialog, closed it, and are now nowhere, forced to re-navigate the entire page from the top to find their place again. Explicitly capturing `document.activeElement` at the moment the modal opens (before any focus-moving side effect) and calling `.focus()` on that captured reference when the modal closes preserves the user's place in the page exactly as they left it, which is the entire point of focus management being a *first-class* feature of the component, not an afterthought.

The trap: assuming the browser "remembers" focus history automatically across arbitrary DOM visibility changes — it doesn't; this has to be implemented explicitly by capturing a reference, and forgetting to capture it *before* moving focus into the modal is the single most common implementation bug in this exact scenario.

---

**Q (High): How do nested modals break a single-`triggerElement`-variable design, and how do you fix it?**

Answer: If there's one module-level `triggerElement` variable, opening a second (inner) modal from within a first (outer) modal overwrites that variable with the inner modal's trigger — which is fine while the inner modal is open, but breaks the moment the *outer* modal eventually closes: by then, the variable holds the inner modal's trigger reference (or, if the inner modal's close logic already nulled it out, nothing at all), not the original page-level element that opened the outer modal. The fix is a stack of `{ modalElement, triggerElement }` pairs rather than a single shared variable — each modal's own trigger is captured and pushed when it opens, and closing a modal pops its own entry and restores focus to *that* entry's trigger, which correctly resolves to "the previous modal" for an inner modal's close, and "the original page element" only once the outermost modal closes. Additionally, `Escape` and the Tab-trap should only ever operate on the topmost (most recently opened) modal, and page-level side effects like `inert`/scroll-lock should only be reverted once the entire stack is empty, not after every individual close.

The trap: fixing only the focus-restoration part (switching to a stack) but forgetting that Escape/Tab handling also needs to be scoped to the topmost modal — if both modals' `keydown` listeners are simultaneously active, pressing Escape could close both, or the outer modal's trap could fight with the inner modal's trap for which one gets to intercept Tab.

---

**Q (Medium): What's the difference between `aria-hidden="true"` and the `inert` attribute for hiding background content, and when would you still reach for `aria-hidden` instead?**

Answer: `aria-hidden="true"` removes an element (and its descendants) from the accessibility tree only — a screen reader's virtual cursor can no longer navigate into it, but the element remains fully mouse-clickable and keyboard-focusable via Tab unless something else (CSS `pointer-events: none`, a working focus trap) separately prevents that. `inert` is a single attribute that simultaneously removes accessibility-tree visibility, focusability, and click/pointer interactivity — functionally, it's "this subtree doesn't exist for interaction purposes right now" in one declaration. Given `inert`'s broad current support, it's the more complete and less error-prone default. A reason to still reach for `aria-hidden` specifically: supporting a codebase/browser matrix where `inert` isn't available and a polyfill isn't in place, or a case where you deliberately want to hide something from assistive tech *without* also making it unclickable (a genuinely rare, specific need, not the common case).

The trap: using `aria-hidden` alone and believing it fully "hides" background content — it hides it from one axis (the accessibility tree) but not from mouse or keyboard interaction, which is precisely the gap `inert` was introduced to close.

---

**Q (Medium): How would you handle the case where the modal's content itself changes after opening (e.g., a form validation error appears), such that the previously-focused element is removed from the DOM?**

Answer: If the currently-focused element inside the modal is removed or hidden (e.g., replaced by an error-state re-render), focus silently falls back to `<body>`, exactly like the trigger-restoration problem but happening *within* the modal instead of at close time — and now the focus trap is also compromised, since Tab from `<body>` doesn't cycle within the modal at all. The fix is defensive: before/after any re-render of the modal's internal content, check whether `document.activeElement` is still `document.body` or otherwise outside the modal, and if so, explicitly re-focus a sensible element (the newly-appeared error message if it's meant to be announced, or back to the first focusable element as a fallback) rather than leaving focus to fall through uncontrolled.

The trap: assuming the focus trap "protects" against this — the trap only intercepts *Tab presses*, it does nothing about focus being lost due to the *focused element itself* disappearing from the DOM, which is a distinct failure mode requiring its own explicit handling.

---

**Q (Medium): Why should `closeOnBackdropClick` be configurable rather than always-on or always-off?**

Answer: Always-on backdrop-dismiss is convenient for low-stakes, informational modals (a "here's what's new" announcement) where an accidental outside click causing a close has no real cost. It's actively harmful for modals gating a destructive or high-effort action — a confirmation dialog for an irreversible delete, or a multi-field form the user has partially filled in — where an accidental click just outside the dialog silently discarding the interaction (or, worse, being misinterpreted by some implementations as an implicit "confirm") is a real, damaging UX bug. Always-off, conversely, is unnecessarily rigid for the many cases where backdrop-dismiss is exactly the expected, low-risk convenience. Making it a per-instance configuration option lets each call site make the right call for its own stakes, rather than the component author guessing one blanket answer that's wrong for some fraction of use cases.

The trap: picking one blanket default and defending it as "the accessible/correct choice" — there isn't a universally correct answer here; the correct engineering answer is recognizing it depends on the specific dialog's consequences and exposing it as configuration rather than a hardcoded opinion.

---

**Q (Low): How would you test a focus trap implementation, beyond manually tabbing through it in a browser?**

Answer: Automated approaches: simulate `Tab`/`Shift+Tab` `keydown` events via a testing library (e.g., dispatching keyboard events or using a browser-automation tool's keyboard API) and assert that `document.activeElement` after each simulated press stays within the modal's focusable-element set, specifically asserting the exact wrap-around cases (Tab on the last element lands on the first; Shift+Tab on the first lands on the last). Also worth testing: that `document.activeElement` immediately after opening is either the specified initial-focus element or the first focusable element; that it's restored to the original trigger after closing; and that background content is genuinely unfocusable while the modal is open (attempting to `.focus()` an inert/hidden background element directly and asserting it doesn't take focus). Automated accessibility scanners (axe-core, etc.) catch missing ARIA attributes but generally can't verify dynamic focus-management behavior — that requires interaction-simulating tests specifically.

The trap: relying solely on an automated accessibility linter/scanner and treating a clean report as proof the focus trap works — static a11y scanners check markup/attributes, not runtime focus-movement behavior, which is exactly what a focus trap's correctness actually depends on.

---

**Q (Low): How does this change if the modal needs to support being rendered via `<dialog>` and its native `showModal()` method instead of a hand-rolled `div`-based implementation?**

Answer: The native `<dialog>` element's `showModal()` provides some of this for free — it renders in the top layer (so it visually sits above everything without manual z-index management), automatically makes content outside it inert to interaction in modern browsers, and natively closes on Escape. It does **not** fully solve focus-trapping (some browsers historically allowed Tab to escape a native `<dialog>` under certain conditions, though this has improved) or focus restoration (you still generally need to capture the trigger and call `.focus()` on it yourself in the `close` event), so a robust implementation on top of `<dialog>` still layers a manual Tab-trap and explicit focus-restoration logic on top of the native element rather than assuming `showModal()` alone is a complete accessible-modal solution. The trade-off is less manual work for inert-background/top-layer behavior, at the cost of needing to verify current browser behavior for the parts natively provided, since "native support" for full keyboard-trap correctness has evolved unevenly across browsers.

The trap: assuming `<dialog>.showModal()` is a complete, zero-additional-code accessible modal solution — it meaningfully reduces the amount of manual work (especially around visual stacking and basic inertness), but focus-trap correctness and trigger-restoration are still generally the implementer's responsibility to verify and, in some cases, implement explicitly.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement a focus trap that correctly wraps Tab/Shift+Tab at both boundaries, intercepting only at the edges
- [ ] Can explain why the trigger element must be captured before focus moves into the modal, and why that ordering matters
- [ ] Can implement `aria-hidden`/`inert` on background siblings and explain the difference between them
- [ ] Can explain why backdrop-click-to-close should be configurable rather than a fixed default
- [ ] Can design the nested-modal case correctly (a stack, not a single trigger reference; topmost-only Escape/Tab handling; stack-emptying-gated cleanup)
- [ ] Can name at least two side effects (scroll lock, `keydown` listener) that must be explicitly reversed on close, and what leaks/breaks if they aren't

---
*Next: Accessible Tabs — Keyboard Navigation — same accessibility rigor, different interaction pattern: roving tabindex instead of a focus trap.*
