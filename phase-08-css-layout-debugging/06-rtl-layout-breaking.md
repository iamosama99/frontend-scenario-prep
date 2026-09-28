# RTL Layout Breaking

## Quick Reference

| Physical (LTR-assuming) | Logical (direction-aware) | Why |
|---|---|---|
| `margin-left` / `margin-right` | `margin-inline-start` / `margin-inline-end` | "Start"/"end" flip automatically with `dir`; "left"/"right" never do |
| `padding-left` / `padding-right` | `padding-inline-start` / `padding-inline-end` | Same — inline axis follows text direction |
| `left: 0` / `right: 0` (positioning) | `inset-inline-start: 0` / `inset-inline-end: 0` | Absolute/fixed positioning offsets also need direction awareness |
| `text-align: left` | `text-align: start` | `start` means "beginning of the inline direction" — left in LTR, right in RTL |
| `border-left` / `border-right` | `border-inline-start` / `border-inline-end` | Border shorthand also has logical equivalents |
| `float: left` / `float: right` | Rarely has a clean logical equivalent — often needs flex/grid restructuring or `[dir="rtl"]` override | Float direction has no built-in logical property; this is a case requiring explicit handling |
| `width` / `height` | `inline-size` / `block-size` | For vertical writing modes specifically, not just RTL — width/height assume horizontal text |

## The Scenario

"We're launching in Arabic and Hebrew markets, both right-to-left languages. QA just filed a bug: on the RTL version of the app, the navigation icons overlap the text, a modal's close button is in the wrong corner, and one component's content visibly runs off the edge of the screen. The English version looks fine. We're setting `dir="rtl"` on the `<html>` element for these locales already. Explain why setting `dir="rtl"` alone didn't fix this, and fix the underlying issue so this doesn't require a page-by-page, bug-by-bug chase every time a new component ships."

## Clarifying Questions

- **How widespread is the use of physical CSS properties (`left`/`right`, `margin-left`/`margin-right`, `float: left/right`, `text-align: left/right`) across the codebase — is this a handful of components, or the majority of the existing layout CSS?** This determines whether the fix is a targeted patch of the specific broken components QA found, or a genuinely codebase-wide migration to logical properties, which is a much larger scoping and prioritization conversation.
- **Is there an existing design system or shared component library, and does it already use logical properties, or would this be introduced for the first time as part of this fix?** If shared components are the source of the bug (e.g., a shared `Modal` component with a hardcoded `right: 16px` close button), fixing them once in the design system fixes every consumer at once — a much higher-leverage fix than patching each page that happens to use a broken shared component.
- **Are there icons or images that are inherently directional (e.g., a "back" arrow, a chevron indicating "next") that need to be mirrored in RTL, versus ones that shouldn't be (a logo, a photo, a play button)?** Not all visual mirroring is automatic or even desirable — some icons need `transform: scaleX(-1)` (or a directionally-aware icon set) under `[dir="rtl"]`, and getting this list right (mirror these, don't touch those) is a real design/product decision, not purely a CSS mechanics question.
- **Does `dir="rtl"` need to be scoped per-locale-only, or could a single page ever need to mix LTR and RTL content (e.g., an Arabic UI displaying a user-entered English name, or a code snippet)?** Real RTL apps very often need this — the `dir` attribute (and the `bdi`/`bdo` elements, and `unicode-bidi` CSS) can be scoped to a specific inline span so embedded LTR content (names, emails, numbers, code) doesn't get bidi-reordered incorrectly within an otherwise-RTL paragraph.
- **What testing exists today for RTL, if any** — is there a way to toggle `dir` in local development, visual regression coverage for the RTL variant, or does RTL correctness currently rely entirely on manual QA passes after each release, as this scenario's bug report implies? Understanding the current gap in tooling/coverage is relevant to the "doesn't require a bug-by-bug chase every time" part of the ask — a durable fix includes closing that testing gap, not just fixing today's specific bugs.

## Approach & Trade-offs

**`dir="rtl"` on `<html>` genuinely does flip a meaningful set of browser default behaviors — text direction, default text alignment, the direction native form controls and default block/inline flow resolve in — but it does nothing to reinterpret *author-written* physical CSS properties, which is the actual gap causing this bug.** The browser's own defaults (how text wraps, which edge `text-align: start`/`end` — not `left`/`right` — resolve to, how `<input>` fields align their cursor and placeholder) do respond to `dir`. But `margin-left: 16px`, `padding-right: 8px`, `left: 0`, and `float: right` are physical properties — they mean exactly what they say, "left" and "right" as fixed compass directions on screen, regardless of `dir`. A developer who wrote `right: 16px` to position a modal's close button in the top-right corner (which, in an LTR mental model, is "the corner near where text ends, where dismissal controls conventionally sit") gets exactly what they asked for — the button 16px from the *physical* right edge — in both LTR and RTL, even though the *logical* intent ("place this near the end of the reading direction") would mean the opposite physical corner in RTL. This is why `dir="rtl"` alone "didn't fix it": it was never going to, because it doesn't touch anything written explicitly in physical terms, and the codebase is full of exactly that.

**Migrate to CSS logical properties as the structural fix, not a directional override stylesheet, because logical properties encode "start of reading direction" and "end of reading direction" as the actual semantic intent — which is direction-agnostic by construction, rather than direction-specific and then manually flipped.** The alternative approach — keep writing `margin-left`/`right` as before, and add a parallel `[dir="rtl"] { .component { margin-left: 0; margin-right: 16px; } }` override for every physical property used — technically works, but doubles the CSS surface for every affected component and requires remembering, forever, for every future component, to write and maintain that mirrored override; it's the CSS equivalent of duplicating logic instead of parameterizing it. Logical properties (`margin-inline-start`, `padding-inline-end`, `inset-inline-start`, `text-align: start`) resolve automatically based on the computed `direction` (and `writing-mode`) at the point they're used — the same single declaration works correctly in both LTR and RTL without any override rule existing at all, because "start" and "end" were never LTR-specific in the first place; they're relative to the flow direction, which is exactly the abstraction actually needed here.

**Directional icons are a real exception that logical properties don't solve, and need explicit handling — this is where "just migrate to logical properties" isn't a complete answer.** An arrow icon pointing left to mean "go back" needs to visually point right in RTL, because the *meaning* ("back," i.e., toward the start of a conceptual sequence) is direction-relative even though the icon asset itself is a fixed image/SVG with no inherent direction-awareness. This can't be solved by a logical CSS property, because there's no "logical arrow direction" primitive — it requires either maintaining direction-aware icon variants, or applying `transform: scaleX(-1)` conditionally under `[dir="rtl"]` to specific icons identified as directionally meaningful (explicitly excluding icons that shouldn't mirror, like a company logo or a play/pause icon, which are compass-neutral or governed by a different convention entirely).

**Fix at the design-system/shared-component layer first, since that's the highest-leverage point of intervention and directly serves the "doesn't require a page-by-page chase" part of the ask.** If the modal's close-button positioning bug lives in a shared `Modal` component used across dozens of pages, fixing that one component's CSS (physical → logical properties) fixes every page using it simultaneously, versus finding and patching each individual page's usage after QA reports it — which is reactive, doesn't scale, and is exactly the "bug-by-bug chase" pattern the prompt asks to eliminate. This reframes the task from "find and fix the three bugs QA reported" to "find the shared component(s) those three bugs trace back to, fix those, then run a systematic audit (grep for physical properties, visual regression testing with `dir="rtl"` toggled) across the rest of the codebase for the same pattern before it surfaces as more individually-reported bugs."

## Root Cause (For This Scenario)

Tracing the three reported bugs back to their source: the navigation icon/text overlap comes from a shared `NavItem` component using `margin-left: 8px` on the icon (intended to create a gap between icon and text when the icon precedes the text in LTR reading order) — in RTL, text and icon order visually reverses via `dir`'s effect on flex/inline layout defaults, but the hardcoded `margin-left` still adds space on the physical left, which is now the *trailing* side of the icon post-reversal, producing zero gap on the actual leading side and the overlap QA observed. The modal close button's wrong-corner placement comes from `position: absolute; right: 16px; top: 16px` on the shared `Modal` component — a straightforward physical-property case, identical to the mechanism described above. The component "running off the edge" traces to a `float: left` on an image within a text block, which (per the Quick Reference table) has no automatic logical equivalent and simply keeps floating to the same physical side regardless of `dir`, while surrounding text reflows around it assuming RTL conventions — producing a visibly broken interaction between a physically-anchored float and logically-reflowing text.

## The Fix

**Step 1 — replace physical properties with logical properties in the shared components identified as root causes:**

```css
/* NavItem — before */
.nav-icon {
  margin-left: 8px;
}

/* NavItem — after */
.nav-icon {
  margin-inline-start: 8px; /* "start" = left in LTR, right in RTL — always the correct leading gap */
}
```

```css
/* Modal close button — before */
.modal-close {
  position: absolute;
  right: 16px;
  top: 16px;
}

/* Modal close button — after */
.modal-close {
  position: absolute;
  inset-inline-end: 16px; /* "end" = right in LTR, left in RTL */
  top: 16px; /* block-axis (vertical) positioning is unaffected by text direction, stays physical */
}
```

**Step 2 — for the float-based image, restructure using flex or grid rather than looking for a logical `float` equivalent (since none exists cleanly):**

```css
/* Before — float has no direction-aware equivalent, breaks in RTL */
.article-image {
  float: left;
  margin-right: 16px;
}

/* After — flex row naturally respects direction via flow, no explicit left/right needed */
.article-body {
  display: flex;
  gap: 16px;
}
.article-image {
  flex-shrink: 0;
}
```

Restructuring away from `float` entirely (toward flex/grid, which inherently participate in logical/directional flow) sidesteps the missing-logical-property gap rather than trying to patch around it with a `[dir="rtl"]` override on `float` itself (which does work — `float: right` under `[dir="rtl"]` is valid CSS — but reintroduces the exact "maintain a mirrored override forever" maintenance burden the broader fix is trying to eliminate).

**Step 3 — handle directional icons explicitly, since logical properties don't cover icon mirroring:**

```css
[dir="rtl"] .icon-back {
  transform: scaleX(-1); /* "back" arrow needs to visually point the other way in RTL */
}

/* Explicitly NOT mirrored — direction-neutral icons */
.icon-logo,
.icon-play {
  /* no [dir="rtl"] override — these are correct in both directions as-is */
}
```

**Step 4 — establish an audit/prevention mechanism so this doesn't require a reactive, bug-by-bug chase going forward:**

```bash
# A lint rule or CI grep step flagging physical properties in new/changed CSS
grep -rnE '(margin|padding|border)-(left|right)\s*:|(?<!inset-)(left|right)\s*:|float\s*:\s*(left|right)' src/
```

In practice, this is better enforced via a stylelint rule (e.g., `stylelint-use-logical-properties-and-values`) integrated into CI, flagging any new physical directional property as a lint error rather than relying on a manually-run grep or, worse, waiting for the next RTL QA pass to catch it — turning this from a recurring bug category into a build-time-caught category.

> **Check yourself:** Why does `top`/`bottom` positioning (the modal close button's `top: 16px`) NOT need to become a logical property here, while `left`/`right` does? What's the general rule for which physical axis is direction-sensitive and which isn't?

## Gotchas

**Assuming `dir="rtl"` handles everything because it visibly does flip *some* things correctly** (default text alignment, native form control behavior, the icon-before-text vs. text-before-icon default reversal in inline/flex contexts without explicit physical overrides) — this partial correctness is exactly what makes the bug non-obvious: the page "mostly looks RTL" which can mask that specific hardcoded physical properties are silently still LTR-anchored underneath.

**Migrating `margin`/`padding` to logical properties but missing `float`, `text-align: left/right` (vs. `start`/`end`), `background-position`, or `box-shadow` offsets**, all of which have direction implications but don't all have equally clean 1:1 logical replacements — a thorough migration audit needs to go beyond the most commonly-known logical properties (`margin-inline-*`, `padding-inline-*`) to catch the less obvious ones.

**Mirroring an icon that shouldn't be mirrored** (a company logo, a numeral, a QR code, a media play/pause icon — none of which have inherent left/right semantic meaning) as a side effect of a blanket "mirror everything in RTL" pass rather than a deliberately curated list — over-correction is a real, common mistake in RTL migrations, not just under-correction.

**Not scoping `dir` for embedded opposite-direction content** — an Arabic UI (RTL) that displays a user's English name, an email address, or a code snippet needs that specific content to render LTR even within an RTL page (numbers and Latin-script text don't reverse correctly if the browser's bidi algorithm treats the surrounding RTL context as applying to them too) — this typically needs `dir="ltr"` (or the semantic `<bdi>` element) scoped specifically to that embedded content, which is easy to miss entirely if RTL testing only ever uses all-Arabic/Hebrew test content and never mixed-direction real-world content.

**Treating this as a one-time fix rather than fixing the process gap** — without a lint rule or automated check, the exact same category of bug (a new component shipped with `margin-left` instead of `margin-inline-start`) will recur the next time any engineer unfamiliar with the RTL requirement writes new CSS, which is precisely the "bug-by-bug chase" the prompt explicitly asks to avoid perpetuating.

## Follow-up Questions

**Q (High): Explain precisely why `dir="rtl"` changes some layout behavior automatically but does nothing for `margin-left`/`right: 16px` written explicitly in CSS.**

Answer: `dir` (and the corresponding CSS `direction` property, which `dir` sets) changes the browser's resolution of *logical* concepts — what "start" and "end" of the inline axis mean, which side `text-align: start` resolves to, the default reading/flow order the browser uses for inline content, native form control text alignment and cursor behavior, and how logical CSS properties (`margin-inline-start`, `inset-inline-end`, etc.) map onto physical sides. It does not, and by design cannot, reinterpret physical properties — `margin-left` means "the left margin" unconditionally, in the same sense that changing a page's language doesn't change what the English word "left" means if a developer typed it as a literal value; `left` and `right` in CSS are compass directions, fixed regardless of any direction/language context, precisely because CSS needs an unambiguous way to say "the actual physical left side" when that's genuinely what's needed (which does still come up — e.g., a decorative element that should always appear on the physical left side of the screen regardless of text direction, though this is rarer than developers usually assume). The entire reason logical properties exist as a separate, newer addition to CSS is to give authors a way to write direction-*relative* declarations instead of direction-*absolute* ones, when direction-relative is what's actually meant — which, for the vast majority of everyday spacing/positioning/alignment CSS, it is.

The trap: describing `dir="rtl"` as "mirroring the page" or "flipping everything," which overstates what it actually does — a candidate with this mental model will be surprised when specific hardcoded values don't flip, rather than correctly predicting in advance (as this question asks) exactly which category of CSS is and isn't affected.

---

**Q (High): Why does a straightforward, blanket `[dir="rtl"] { * { transform: scaleX(-1); } }` (mirror the entire page) not work as a shortcut to full RTL support, even though it would visually flip left/right positioning correctly?**

Answer: A blanket page-level mirror would flip *everything* indiscriminately, including content that must never be mirrored — text itself would render backwards/mirrored (individual glyphs flipped, not just their position, which is nonsensical for actual Arabic/Hebrew script, which has its own correct glyph shapes and needs to be read normally, not as a mirror image of LTR text), images and photos would be mirrored in ways that are often factually wrong (a photo of a real street sign, a person's face, a screenshot of an English-language interface embedded as an image), and icons that shouldn't mirror (logos, media controls) would be mirrored along with ones that should. A `transform: scaleX(-1)` on an ancestor also visually mirrors it as a compositor-level trick, not a semantic layout change — it doesn't correct the underlying accessibility tree, tab order, or genuinely correct text/number shaping, so it produces a visually-plausible-looking but semantically/functionally broken result (a screen reader, or a sighted keyboard user tabbing through, would experience an interface that doesn't actually behave correctly, just looks flipped). Real RTL support requires the browser (and the app's CSS/markup) to genuinely understand direction — which is what `dir`, logical properties, and the bidi algorithm are for — not a purely visual illusion applied after the fact.

The trap: proposing a global mirror transform as a "clever" universal fix without considering that mirroring is a purely visual/compositor operation that doesn't distinguish between content that's genuinely mirror-symmetric in meaning (a directional back arrow) and content that would become incorrect or nonsensical if visually flipped (text glyphs, photos, logos) — a good answer needs to identify that RTL support and "mirror image" are fundamentally different problems that happen to overlap for some UI elements but not others.

---

**Q (Medium): A `<input type="text">` with placeholder text works correctly (placeholder aligns to the right, cursor starts from the right) in RTL without any explicit CSS from the team. Why does this work automatically when `margin-left` on a sibling `<button>` doesn't?**

Answer: Native form controls' internal rendering (text alignment, cursor start position, placeholder alignment) is implemented by the browser itself and is explicitly direction-aware as part of the browser's own default UA stylesheet and native widget behavior — the browser looks at the computed `direction` (set via `dir` or the CSS `direction` property) and adjusts these internal behaviors accordingly, because text input direction is considered a first-class, universally-needed behavior the platform handles for every app, not something each website should have to reimplement. `margin-left` on an arbitrary sibling button, by contrast, is author-written CSS with no special browser-level direction-awareness attached to it — the browser has no way to know that a specific `margin-left` declaration was "meant" to represent a logical "leading gap" versus a genuinely intentional, always-physical-left gap; physical properties are taken completely literally, which is why the responsibility for direction-awareness shifts entirely to the author choosing logical properties instead, for anything the browser's own native-widget behavior doesn't already handle automatically.

The trap: concluding from "some things just work automatically in RTL" that CSS generally reacts to `dir`, and being surprised when author-written spacing/positioning doesn't — the automatic cases are specifically the ones the browser's own native widgets and default stylesheet implement with direction-awareness built in; everything an author writes in plain physical CSS is exempt from that automatic behavior by definition.

---

**Q (Medium): How would you approach testing for RTL regressions going forward, beyond a stylelint rule catching new physical properties at write-time?**

Answer: Beyond the write-time lint rule (which catches new violations but doesn't verify existing components or the actual visual result), I'd add RTL variants to visual regression testing — most visual regression tools (Percy, Chromatic, Playwright's screenshot testing) support rendering the same component/page twice, once with `dir="ltr"` and once with `dir="rtl"`, and diffing each against its own separate baseline; this catches both new physical-property regressions and structural RTL bugs a lint rule can't (like the `float`-based image case, which is syntactically valid CSS with no lint-flaggable physical property misuse, but is still structurally wrong in RTL). I'd also want targeted RTL testing in whatever component development environment exists (Storybook with an RTL toggle addon, if Storybook is in use) so individual component authors can visually verify RTL correctness during development, before a change ever reaches a full page or QA — catching the bug at the point of authorship rather than downstream. Finally, I'd specifically include mixed-direction content (an English name/email embedded in an Arabic UI) in at least some test fixtures, since, per the earlier gotcha, pure single-language RTL test content can pass while still hiding a bidi-embedding bug that only appears with genuinely mixed content.

The trap: treating a stylelint rule alone as sufficient RTL regression coverage — it's necessary but not sufficient, since it only catches one specific category of bug (physical properties in authored CSS) and has no visibility into structural issues (float-based layouts, JS-driven inline styles, third-party component RTL behavior) or the actual rendered visual result, which only a real render-and-compare (visual regression) step can verify.

---

**Q (Low): Would `writing-mode` (e.g., vertical text for some East Asian typesetting styles) interact with this same logical-properties fix, or is that an entirely separate concern from RTL?**

Answer: They're related but distinct axes of the same underlying "logical vs. physical" concept — `direction` (LTR/RTL) governs the *inline* axis's direction, while `writing-mode` governs which physical axis (horizontal or vertical) *is* the inline/block axis in the first place. Logical properties (`margin-inline-start`, `inline-size`, `block-size`) are actually designed to be correct under *both* axes simultaneously — `inline-size` means "size along whichever axis is currently the inline axis," which is `width` in a horizontal writing mode but `height` in a vertical one, and `margin-inline-start` correctly resolves to the right physical margin regardless of both the writing mode and the text direction combined. This means the same logical-properties migration done for RTL support already provides correctness for vertical writing modes too, essentially for free — a codebase using `width`/`height` and `left`/`right` physically would need a completely separate, likely more invasive fix to support vertical writing modes (since `width` and `height` don't swap meaning the way logical `inline-size`/`block-size` do), whereas one already migrated to logical properties for RTL is most of the way there already.

The trap: treating RTL support and vertical-writing-mode support as unrelated problems needing entirely separate solutions — the logical-properties system was specifically designed to abstract over both simultaneously, and a candidate aware of this can point out that the RTL fix has broader payoff than just the two languages named in the scenario, which is a stronger, more complete answer than treating this as an RTL-only concern.

---

## Self-Assessment

- [ ] Can explain precisely why `dir="rtl"` doesn't reinterpret physical CSS properties, with the correct mental model (logical vs. physical, not "mirroring")
- [ ] Can name the logical-property equivalent for at least five common physical properties (`margin-left/right`, `padding-left/right`, `left/right` positioning, `text-align: left/right`, `border-left/right`)
- [ ] Can explain why `float` has no clean logical equivalent and needs restructuring, not a direct swap
- [ ] Can identify which icons should vs. shouldn't be mirrored under RTL and articulate the distinction
- [ ] Can explain why a blanket page-mirror transform is not a valid shortcut for genuine RTL support
- [ ] Can propose both a write-time (lint) and render-time (visual regression) mechanism to prevent regression, not just a one-time fix

---
*Next: Nested Scroll Container Overflow Trap — moves from a directional-axis bug to a scroll-axis one, where overflow, sticky positioning, and touch/wheel event propagation interact across nested scroll containers in ways that are easy to get wrong in similar "looks fine until a specific real interaction" ways.*
