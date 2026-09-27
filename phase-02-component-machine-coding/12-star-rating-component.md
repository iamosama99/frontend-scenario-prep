# Star Rating Component

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| ARIA model | `role="radiogroup"` of `role="radio"` stars, or `role="slider"` | Radiogroup matches "pick one of N discrete values" semantics; slider matches a continuous range better if half-steps are exposed as a single value |
| Hover vs. committed state | Separate `hoverValue` (transient, visual) from `value` (actual rating) | Moving the mouse away must revert the visual preview without touching the real, committed rating |
| Keyboard support | Arrow Left/Right (or Up/Down) change value, mirroring native radiogroup behavior | A mouse-only star rating is a hard accessibility failure for a form control |
| Half-star increments | Double the internal value scale (0–10 instead of 0–5); render partial fill via `clip-path`/gradient | Detecting "half a star" from raw mouse X position is fragile and inconsistent across icon sizes; scale doubling is the robust approach |
| Read-only mode | A separate render path with no interactive elements, ARIA reflects value only | Common variant (e.g., average rating on a product card) that must not be keyboard-focusable or announced as an editable control |
| Touch handling | Tap commits the value directly — no hover-then-click flow | Touch devices have no hover state, so the interaction model can't depend on it |

## The Scenario

"Build a star rating component — five stars, click to set a rating, and it should show a hover preview as the user moves across the stars. We'll also want a read-only version for displaying an average rating on a product listing. Bonus if you can support half-star ratings."

## Clarifying Questions

- **Should this be modeled as a group of discrete radio-like choices (1 through 5), or as a continuous slider that happens to render as stars?** This decides the ARIA pattern: `role="radiogroup"` of five `role="radio"` stars matches "choose exactly one of 5 discrete values" cleanly, while `role="slider"` with `aria-valuenow`/`aria-valuemin`/`aria-valuemax` fits better if the value can be any number in a range (which becomes relevant the moment half-stars are in scope) since a radiogroup model with half-star support would need 10 individual radio options, which is awkward to represent and announce.
- **Does keyboard interaction need to fully match a native form control, or is click-only acceptable for a first pass?** A mouse/touch-only star rating is a real, common accessibility failure — I'd want to confirm this is in scope for interaction, not just visual polish, since it changes the markup and event handling from the start.
- **For half-star ratings, how is the half-star actually set — is it a separate gesture (e.g., clicking the left half of a star), or does it just need to be *displayable* (e.g., showing a 3.5 average) without necessarily being *settable* interactively?** These are very different problems: displaying a fractional value is just a rendering detail, while letting a user *set* a half-star value means the interactive hit-testing has to distinguish left-half vs. right-half of each star icon.
- **Is the read-only/display mode ever focusable or interactive at all — e.g., does hovering it show a tooltip with the exact number?** This determines whether the read-only version needs any JS event handling at all, or whether it's genuinely static markup (which is simpler and correctly non-interactive for assistive tech).
- **What should happen on touch devices, where there's no hover state to preview the rating before committing?** I'd confirm the expectation is "tap directly sets the value" rather than trying to simulate a hover-preview on touch, since touch doesn't have a natural equivalent for a preview-before-commit gesture.

## Approach & Trade-offs

The first real decision is the **ARIA modeling** — radiogroup-of-radios versus slider. Radiogroup fits the mental model of "5 discrete, mutually exclusive choices" very naturally when the rating is whole-star only, and gets a lot of expected keyboard behavior "for free" conceptually (Arrow keys move selection among a group of radios, matching native `<input type="radio">` groups). The complication is half-stars: modeling 10 discrete values (0.5 increments from 0.5 to 5) as 10 individual `role="radio"` elements is technically possible but semantically strained — a "half star" isn't really a separate, individually-labeled choice the way "3 stars" is. For that reason, once half-star support is required, I'd lean toward `role="slider"` with `aria-valuemin="0"`, `aria-valuemax="5"`, `aria-valuestep="0.5"`, and `aria-valuenow`/`aria-valuetext` reflecting the current value (`aria-valuetext="3.5 out of 5 stars"` is what actually gets announced, since a screen reader announcing a bare "3.5" without units is confusing). I'd state this trade-off explicitly rather than silently picking one — whole-star-only favors radiogroup, half-star support favors slider — and for this build I'd implement the slider model since half-stars are an explicit "bonus" ask.

The second decision is separating **hover-preview state from committed value state**, which is easy to get subtly wrong if a candidate stores only one `value` variable and overwrites it directly on `mouseover`. The correct model keeps `value` (the actual, committed rating — what would be submitted with a form, or persisted) completely separate from `hoverValue` (purely visual, reset to `null` on `mouseleave`, and only ever used for rendering the preview fill, never for reading "what is the rating"). Rendering logic then displays `hoverValue ?? value` — hover state visually overrides the display only while active, and vanishing it on mouse-leave cleanly reverts to the true committed value with no extra bookkeeping.

For **keyboard support**, I mirror native radiogroup behavior: Arrow Right/Up increases the value, Arrow Left/Down decreases it, Home jumps to the minimum, End to the maximum — all operating on the *committed* value directly (keyboard interaction has no natural "preview" phase the way hover does, so Arrow keys commit immediately, which also matches how a native `<input type="range">` behaves).

For **half-star rendering**, I explicitly reject trying to infer "half star intent" from raw mouse pixel position within a single star icon in a naive way (e.g., "if mouseX is in the left 50% of the icon's bounding box, treat it as a half star") without first deciding the value scale — the robust approach is doubling the internal scale: instead of reasoning about "2.5 stars" as a fractional float sprinkled through the code, the internal value is an integer 0–10 (`5` displayed stars × 2 half-steps each), and only the *rendering* layer divides by 2 for display and uses CSS (`clip-path: inset(0 50% 0 0)` on a full star, or a `linear-gradient` background matched to the fill percentage) to paint a partial star. Mouse position still matters for *detecting* which half of a given star icon the pointer is over, but the value itself is always a clean integer internally, avoiding floating-point comparison headaches (`value === 3.5` style checks) throughout the rest of the component.

For the **read-only display mode**, I don't reuse the interactive component with event handlers merely disabled — I render a genuinely separate, simpler path: no `role="slider"`/`role="radiogroup"`, no `tabindex`, no event listeners at all, just the visual stars (full/partial/empty) plus a plain text equivalent (e.g., a visually-hidden `<span>` reading "4.2 out of 5 stars") for assistive tech, since a read-only rating is informational content, not a form control, and shouldn't be announced or focusable as one.

For **touch**, the interaction model can't rely on hover-then-click, since touch devices don't fire `mouseover` before a tap in any useful way (some fire a synthetic hover on first tap, which is exactly the kind of inconsistent behavior not to design around). So on touch, a single `tap` directly sets the committed value at the tapped star/half-star position — there's no preview phase at all, matching how e.g. a native `<input type="range">` behaves on touch (you drag/tap directly to a value, you don't preview it first).

## Solution

State shape, and the rendering formula that reads clean because hover/committed state are separate:

```javascript
const state = {
  value: 0,        // committed rating, 0–10 internal scale (integer, half-steps)
  hoverValue: null, // transient preview, same scale, null when not hovering
  max: 5,           // displayed stars
  readOnly: false,
};

function getDisplayValue(state) {
  return state.hoverValue ?? state.value; // hover overrides display only while active
}
```

Markup — slider model, chosen because half-star support is in scope:

```html
<div
  class="star-rating"
  role="slider"
  tabindex="0"
  aria-valuemin="0"
  aria-valuemax="5"
  aria-valuenow="3.5"
  aria-valuetext="3.5 out of 5 stars"
  aria-label="Rating"
>
  <!-- Individual star icons are purely visual (aria-hidden); the group
       itself is the single slider control announced to assistive tech. -->
  <span class="star" data-index="0" aria-hidden="true"></span>
  <span class="star" data-index="1" aria-hidden="true"></span>
  <span class="star" data-index="2" aria-hidden="true"></span>
  <span class="star" data-index="3" aria-hidden="true"></span>
  <span class="star" data-index="4" aria-hidden="true"></span>
</div>
```

Computing fill and half-star detection from pointer position:

```javascript
function getValueFromPointer(container, clientX, max) {
  const stars = [...container.querySelectorAll('.star')];
  const containerRect = container.getBoundingClientRect();

  for (const star of stars) {
    const rect = star.getBoundingClientRect();
    if (clientX >= rect.left && clientX <= rect.right) {
      const index = Number(star.dataset.index);
      const isLeftHalf = clientX - rect.left < rect.width / 2;
      // Internal scale is 0-10: each star index contributes 2 half-steps.
      return index * 2 + (isLeftHalf ? 1 : 2);
    }
  }

  // Pointer is past the last star or before the first — clamp.
  return clientX < containerRect.left ? 0 : max * 2;
}

function renderStars(container, displayValueOutOf10, max) {
  const stars = [...container.querySelectorAll('.star')];
  stars.forEach((star, index) => {
    const starValueOutOf10 = (index + 1) * 2;
    const fillPercent = Math.max(
      0,
      Math.min(100, (displayValueOutOf10 - index * 2) * 50)
    );
    // 0% = empty, 50% = half, 100% = full — driven purely by CSS custom property.
    star.style.setProperty('--fill', `${fillPercent}%`);
  });
}
```

```css
.star {
  /* Empty star as the base background; filled star painted via gradient
     clipped to --fill, avoiding any JS-side "half star icon" asset. */
  background: linear-gradient(
    90deg,
    var(--star-filled-color) var(--fill, 0%),
    var(--star-empty-color) var(--fill, 0%)
  );
  -webkit-mask: var(--star-shape-mask);
  mask: var(--star-shape-mask);
}
```

Wiring hover preview (mouse), commit on click, and keyboard support:

```javascript
function initStarRating(container, { max = 5, onChange } = {}) {
  let value = 0;
  let hoverValue = null;

  function update() {
    renderStars(container, hoverValue ?? value, max);
    container.setAttribute('aria-valuenow', (value / 2).toString());
    container.setAttribute('aria-valuetext', `${value / 2} out of ${max} stars`);
  }

  container.addEventListener('mousemove', (event) => {
    hoverValue = getValueFromPointer(container, event.clientX, max);
    update();
  });

  container.addEventListener('mouseleave', () => {
    hoverValue = null; // revert preview, committed value is untouched
    update();
  });

  container.addEventListener('click', (event) => {
    value = getValueFromPointer(container, event.clientX, max);
    onChange?.(value / 2);
    update();
  });

  // Touch: tap commits directly, no preview phase — there is no hover to speak of.
  container.addEventListener('touchend', (event) => {
    const touch = event.changedTouches[0];
    value = getValueFromPointer(container, touch.clientX, max);
    onChange?.(value / 2);
    update();
  });

  container.addEventListener('keydown', (event) => {
    const step = event.shiftKey ? 1 : 1; // 1 internal unit = one half-star step
    if (event.key === 'ArrowRight' || event.key === 'ArrowUp') {
      value = Math.min(max * 2, value + step);
    } else if (event.key === 'ArrowLeft' || event.key === 'ArrowDown') {
      value = Math.max(0, value - step);
    } else if (event.key === 'Home') {
      value = 0;
    } else if (event.key === 'End') {
      value = max * 2;
    } else {
      return; // don't preventDefault for keys we don't handle
    }
    event.preventDefault();
    onChange?.(value / 2);
    update();
  });

  update();
}
```

Read-only mode — a genuinely separate, non-interactive render path:

```javascript
function renderReadOnlyRating(container, value, max = 5) {
  container.innerHTML = '';
  container.removeAttribute('role');
  container.removeAttribute('tabindex');
  container.removeAttribute('aria-valuenow');

  const visual = document.createElement('div');
  visual.className = 'star-rating star-rating--readonly';
  visual.setAttribute('aria-hidden', 'true'); // visual stars are decorative here
  renderStars(visual, value * 2, max);

  const textEquivalent = document.createElement('span');
  textEquivalent.className = 'visually-hidden'; // standard sr-only utility class
  textEquivalent.textContent = `${value} out of ${max} stars`;

  container.append(visual, textEquivalent);
}
```

> **Check yourself:** Explain why `hoverValue` and `value` must be two separate variables rather than one — what specific bug does merging them into a single variable produce, and when would it show up?

## Gotchas

**Merging hover and committed state into one variable.** If `mousemove` writes directly to the same `value` used for the actual rating, moving the mouse away without clicking silently changes the committed rating to whatever was last hovered — a serious functional bug, not just a visual one.

**Radiogroup model that doesn't scale to half-stars.** Committing early to `role="radio"` × 5 and then bolting on half-star support forces either 10 awkwardly-labeled radio options or abandoning the ARIA pattern midway — this is why the ARIA model choice should be made with half-star support in mind from the start, if it's in scope.

**`aria-valuenow` without `aria-valuetext`.** A screen reader announcing a bare `aria-valuenow="3.5"` on a slider gives no unit context ("3.5 what?") — `aria-valuetext` should carry the full human-readable string ("3.5 out of 5 stars").

**Detecting half-stars from raw pixel math without a clean internal scale.** Trying to carry fractional values (`2.5`, `3.5`) through comparisons and state updates directly invites floating-point equality bugs; doubling the scale to integers (0–10) internally and only dividing by 2 at the rendering/output boundary avoids this entirely.

**Read-only mode reusing the interactive component with handlers merely disabled.** Leaving `role="slider"` and `tabindex="0"` on a "read-only" rating means it's still announced and focusable as an editable control that happens to do nothing when interacted with — confusing and technically incorrect; read-only should be a structurally different, non-interactive render.

**Assuming a touch `mouseover`-equivalent exists.** Building the interaction model around "hover reveals the preview, tap commits" doesn't work on touch, since there's no reliable pre-tap hover signal — touch needs its own direct-commit-on-tap path, not a hover state that never fires.

**Keyboard step size wrong relative to displayed granularity.** If keyboard Arrow presses move the value by a whole star while the pointer interaction supports half-star granularity, keyboard users get strictly less precision than mouse users — the step size (internal scale units) should match across both input modes unless there's a deliberate reason to differ (e.g., Shift+Arrow for fine steps).

## Follow-up Questions

**Q (High): Why keep `hoverValue` and `value` as two separate pieces of state instead of one?**

Answer: `value` represents the actual, committed rating — what would be submitted with a form or persisted to a backend — while `hoverValue` is a purely transient, visual-only preview that should have zero effect on the real rating unless the user actually commits (clicks, or on touch, taps). If they're merged into a single variable, moving the mouse across the stars during a hover preview directly overwrites the "real" rating value, so simply moving the mouse away without ever clicking silently changes what's stored as the user's rating — a functional bug that's easy to miss in manual testing (since visually, hovering and clicking look similar) but very visible once state is actually submitted or persisted. The fix is rendering `hoverValue ?? value` for display purposes only, while every place that reads "what is the current rating" (submission, persistence, `onChange` callbacks) reads `value` exclusively.

The trap: describing this only as "cleaner code" — it's not a style preference, it's the difference between correct and incorrect behavior; a merged-state version has an actual bug, not just messier code.

---

**Q (High): Walk through the trade-off between `role="radiogroup"`/`role="radio"` and `role="slider"` for this component.**

Answer: `role="radiogroup"` of `role="radio"` stars models the rating as N discrete, mutually exclusive, individually meaningful choices — this fits a whole-star-only rating well (5 clearly distinct options, arrow-key navigation between them mirrors native radio-group behavior exactly) and gives screen reader users a very familiar interaction pattern ("radio button 3 of 5, selected"). It breaks down once half-star increments are required, since representing 10 half-star options as 10 individually-labeled radios is semantically awkward — a "half star" isn't a separately meaningful, nameable choice the way "3 stars" is. `role="slider"` instead models the rating as a single continuous(-ish) value with `aria-valuemin`/`aria-valuemax`/`aria-valuestep`/`aria-valuenow`/`aria-valuetext`, which naturally accommodates any granularity (whole star, half star, even finer) without needing a discrete ARIA node per increment, at the cost of losing the very familiar "pick one of these N labeled things" radiogroup interaction model for simpler, whole-star-only cases.

The trap: picking one pattern dogmatically without connecting the choice back to whether half-star support is actually in scope — the "right" answer depends on that requirement, and a senior candidate should say so explicitly rather than asserting one is universally correct.

---

**Q (High): How do you avoid floating-point value bugs (e.g., `value === 3.5` failing due to float imprecision) when supporting half-star increments?**

Answer: Represent the value internally on a doubled integer scale — 0 to `max * 2` (so 0–10 for a 5-star rating) — where every increment is a whole integer half-step, and only divide by 2 at the boundaries where the value needs to be human-readable or externally reported (`aria-valuetext`, the `onChange` callback's emitted value, form submission). This sidesteps floating-point comparison entirely for all internal logic (clamping, keyboard step increments, hover/commit comparisons), since integer arithmetic and equality are exact, and confines the division-by-2 (the only place a non-integer appears) to a single, easily-reviewed conversion point.

The trap: allowing fractional values (`2.5`, `3.5`) to flow through comparisons and state transitions throughout the component — even though JS floats can represent values like `0.5` exactly in this particular case, building the habit of comparing floats directly for equality is a latent bug magnet the moment the increment granularity or arithmetic changes (e.g., adding quarter-star support later).

---

**Q (Medium): Why does the read-only display variant need to be a structurally different render path rather than the interactive component with handlers disabled?**

Answer: A "disabled" interactive control (still `role="slider"`, still `tabindex="0"`, event handlers just no-op) is still announced to assistive technology as a focusable, nominally-interactive form control, and still receives keyboard focus during Tab navigation — that's misleading for something that's purely informational (e.g., an average rating shown on a product card), since a screen reader user would tab to it expecting to be able to interact with it, only to find nothing happens. The read-only path should have no `role`, no `tabindex`, and no event listeners at all — it's decorative visual content (marked `aria-hidden="true"` on the star icons themselves) paired with a plain, visually-hidden text equivalent conveying the value ("4.2 out of 5 stars") for assistive tech to read as ordinary content, not as a control.

The trap: reaching for `aria-disabled="true"` on the interactive markup as the fix — that communicates "this control exists but is currently unavailable," which is a different (and wrong) semantic from "this isn't a control at all, it's just a number being displayed."

---

**Q (Medium): How do you handle the interaction on touch, given there's no hover-preview phase?**

Answer: The interaction model for touch drops the preview phase entirely — a `touchend` (or `pointerup` in a unified Pointer Events implementation) handler computes the tapped position directly and commits it as the new value immediately, the same computation the desktop `click` handler uses, just without a preceding `mousemove`-driven preview. This mirrors how native range-like controls behave on touch generally (you don't get a "preview" before dragging a native slider on mobile either — direct manipulation commits immediately), and it avoids the fragile, inconsistent behavior of trying to synthesize a hover state from touch events, since mobile browsers' synthetic hover/mouse-event emulation on tap is notoriously inconsistent across devices and easy to get subtly wrong.

The trap: trying to simulate hover-then-tap-to-confirm on touch (e.g., first tap previews, second tap commits) — this adds an extra, non-obvious step to what should be a single, direct interaction, and doesn't match user expectations built from every other touch-based rating control they've used.

---

**Q (Medium): What does `aria-valuetext` do differently from `aria-valuenow`, and why is it necessary here?**

Answer: `aria-valuenow` communicates the raw numeric value of a range-like widget (e.g., `3.5`) for programmatic/API purposes, but a screen reader announcing just the bare number gives no unit or context — the user hears "3.5" with no indication of what that number represents. `aria-valuetext` overrides what's actually *announced* to the user with a fully human-readable string (e.g., "3.5 out of 5 stars"), while `aria-valuenow` remains available for anything reading the value programmatically. Whenever a slider-modeled control's raw numeric value isn't self-explanatory out of context, `aria-valuetext` should be set alongside `aria-valuenow`, not instead of it.

The trap: setting only `aria-valuenow` and assuming that's sufficient because "the number is right there" — sighted users see the visual star fill for context, but a screen reader user hears only what's explicitly announced, and a bare number without units is meaningfully less informative.

---

**Q (Low): How would you extend this component to support a "clear rating" interaction (resetting to 0/unrated)?**

Answer: A common pattern is making a second click on the currently-committed value (e.g., clicking the 3rd star when the rating is already exactly 3) reset the value to 0/unrated, rather than being a no-op — this requires comparing the newly-computed click value against the current committed `value` before applying it, and setting to 0 instead if they match exactly. For keyboard, this is usually a dedicated key (e.g., Backspace or Delete) rather than overloading Arrow keys, since Arrow keys already have clear increment/decrement semantics that shouldn't also carry a "jump to zero" meaning.

The trap: implementing "click to clear" as "clicking anywhere on an already-rated component clears it" rather than specifically "clicking the exact currently-selected value clears it" — the former makes it impossible to simply re-click the same rating to confirm it without accidentally clearing it.

---

**Q (Low): If this component needs to support a form submission (e.g., part of a review form), how does it integrate with native form semantics?**

Answer: Since a custom `role="slider"`/`role="radiogroup"` element isn't a native form control, it doesn't automatically participate in form submission (`FormData`) the way an `<input>` does — the usual fix is pairing it with a hidden native input (`<input type="hidden" name="rating">`) whose `value` is kept in sync with the component's committed `value` on every change, so the surrounding `<form>` picks it up on submit without needing custom JS to read the component's state separately. Alternatively, if using Web Components / form-associated custom elements, `ElementInternals`' `setFormValue` API achieves the same result more natively.

The trap: assuming a custom ARIA widget "just works" inside a `<form>` because it visually looks like part of the form — without an explicit hidden input or form-associated custom element wiring, its value is invisible to native form submission entirely.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can justify the ARIA model choice (radiogroup vs. slider) based on whether half-stars are in scope
- [ ] Can implement hover-preview state fully separated from committed value state
- [ ] Can implement Arrow/Home/End keyboard support matching native range-control conventions
- [ ] Can implement half-star support via a doubled integer scale, avoiding float-based value comparisons
- [ ] Can build a structurally separate, non-interactive read-only render path with a text equivalent for assistive tech
- [ ] Can explain why touch interaction must commit directly on tap rather than relying on a hover-then-click flow

---
*Next: OTP / PIN Input — another compact form-control widget, this time built from N separate character boxes that have to behave like one logical input.*
