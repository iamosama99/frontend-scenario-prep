# Sticky Header Broken on Mobile Safari

## Quick Reference

| Symptom | Root Cause | Fix |
|---|---|---|
| Sticky header scrolls away entirely | An ancestor has `overflow: hidden`/`auto`/`scroll` — sticky is relative to the *nearest scrolling ancestor*, not the viewport | Remove/relocate the overflow, or move the sticky element outside that ancestor |
| Sticky header works on desktop Chrome, not mobile Safari | Ancestor has `transform`, `filter`, `perspective`, or `will-change` set — these create a new containing block that sticky (and fixed) resolve against | Remove the transform from the ancestor, or apply it to a wrapper that isn't in the sticky element's ancestor chain |
| Sticky header "jumps" or flickers while scrolling on iOS | Height of the sticky element changes dynamically (e.g., a dynamically-injected banner) without accounting for iOS Safari's collapsing address bar changing the visual viewport height | Use `dvh`/visual-viewport-aware sizing, and avoid layout shifts inside the sticky element during scroll |
| Sticky header covers content underneath it after sticking | No corresponding `scroll-margin-top`/`scroll-padding-top` or content padding equal to header height | Add `scroll-margin-top` on anchor targets, or top padding on the scroll container equal to the sticky header's height |
| Sticky inside a flex/grid child doesn't stick | The flex/grid item's height is being stretched or its parent doesn't have a definite height for the sticky element to stick within | Ensure the sticky element's direct parent has enough height for `top` offset to have room to apply, and isn't itself shrink-to-fit in a way that removes scrollable room |

## The Scenario

"We have a sticky header on our product listing page — `position: sticky; top: 0`. It works fine everywhere we've tested: Chrome, Firefox, desktop Safari. A user reports it doesn't stick at all on their iPhone — it just scrolls away with the rest of the page like a normal header. We can't repro it in our desktop browser's device emulation mode, sticky look fine there too. Figure out what's actually different about real mobile Safari, and fix it in a way that doesn't just work by luck."

## Clarifying Questions

- **Is there any ancestor of the sticky header that has `overflow: hidden`, `overflow: auto`, or `overflow: scroll` set, even several levels up — and does it apply differently or not at all in desktop emulation vs. a real device?** `position: sticky` is scoped to its nearest scrolling ancestor, not automatically the viewport — this is the single most common root cause and is worth checking before anything mobile-Safari-specific, since it would break identically everywhere once triggered, and "only breaks on real iPhone" more likely points to something environment-specific layered on top.
- **Does any ancestor apply a `transform`, `filter`, `perspective`, `will-change: transform`, or `contain: layout/paint` — properties that create a new containing block?** These are the classic desktop-emulation-blind-spot cause: browser dev tools' "mobile emulation" mode simulates viewport size and touch events, but doesn't change how the *rendering engine itself* computes containing blocks — a bug caused by an actual WebKit/mobile-Safari-specific containing-block quirk (or a genuine spec-compliant containing-block change from a transform) won't reproduce just by resizing a desktop browser's viewport.
- **Is desktop Safari (same rendering engine family, WebKit) also affected, or only mobile/real-device Safari specifically?** If desktop Safari also sticks correctly but mobile Safari on a real device doesn't, that narrows it toward something mobile-Safari-specific (historically, older iOS Safari versions had `position: sticky` support gaps or required `-webkit-sticky`; more recent versions have edge cases around dynamic toolbar/viewport resizing) rather than a general WebKit rendering difference.
- **What iOS/Safari version is the user on, and is this reproducible across multiple real iPhones, or just this one user's device?** Older iOS versions had known `position: sticky` bugs (including needing the `-webkit-` prefix, or sticky not working at all inside certain scroll contexts); if it's a single user on an old iOS version, the fix might be a vendor-prefixed fallback or accepting graceful degradation rather than chasing a live bug in current Safari.
- **Is the page using any scroll-hijacking or custom-scroll libraries (e.g., a "smooth scroll" polyfill, an infinite-scroll library that manipulates a container's `overflow`), and are they applied differently on mobile vs. desktop?** Custom scroll implementations frequently introduce exactly the "extra overflow ancestor" or "extra transform" causes above, and might be conditionally loaded or behave differently based on touch capability detection, which would explain a mobile-only manifestation of an otherwise generic sticky-positioning root cause.

## Approach & Trade-offs

**Start from `position: sticky`'s actual algorithm, not guesswork, because "sticky is broken" bugs are almost always one of a small, well-known set of causes, not a browser bug.** `position: sticky` behaves like `position: relative` until the element's computed `top`/`bottom`/etc. offset would be crossed by the nearest ancestor with a scrolling mechanism, at which point it behaves like `position: fixed` *relative to that ancestor* — not relative to the viewport, and not relative to `body`, unless the nearest scrolling ancestor happens to be the viewport itself. This makes the debugging process closer to "read the DOM tree upward and check two specific properties on every ancestor" than "poke at CSS values until it works" — the two properties being (1) any `overflow` value other than `visible` on an ancestor (that becomes sticky's scroll boundary), and (2) any property that creates a new containing block (`transform`, `filter`, `perspective`, `will-change: transform`, `contain: layout`) on an ancestor, since sticky positioning is computed relative to the nearest ancestor that either scrolls or establishes a containing block, whichever is found first walking up.

**Desktop emulation not reproducing the bug is itself a diagnostic clue, not a dead end — it points toward a genuine engine-level containing-block/rendering difference rather than a purely responsive-CSS issue.** Chrome DevTools' device toolbar changes the viewport dimensions and reports touch capability, but it's still Chrome's rendering engine underneath — any bug caused by how WebKit specifically resolves a containing block, or a transform-related quirk, won't show up because the code producing the layout is a different engine entirely. This means the right move when "can't repro in desktop emulation" is explicitly reaching for BrowserStack/a real device/Safari's own responsive design mode (which still uses WebKit, unlike Chrome's emulator) rather than continuing to poke at Chrome's simulated mobile view expecting a WebKit-specific bug to appear there.

**Fix at the actual source of the containing-block/overflow issue, not by reaching for `!important` or duplicating positioning logic in JS.** A common failure mode under time pressure is to "fix" a broken sticky element by re-implementing sticky behavior with a scroll event listener and manually toggling `position: fixed` — this reintroduces exactly the scroll-jank problems `position: sticky` was designed to avoid (a JS-driven positioning fix runs after the browser's own layout/paint, causing visible lag on scroll, especially on mobile hardware) and adds an entire maintenance surface (resize handling, scroll listener cleanup, edge cases sticky handles for free) to work around what's usually a one-line CSS fix once the actual containing-block/overflow ancestor is identified and corrected.

## Root Cause (For This Scenario)

Walking the header's ancestor chain reveals the actual cause: a wrapper div several levels up has `transform: translateZ(0)` applied — a common (and, on iOS specifically, historically load-bearing) trick to force GPU-accelerated rendering and avoid scroll jank on iOS Safari in older codebases. That `transform` establishes a new containing block for anything positioned inside it, including descendants using `position: sticky` — which means the sticky header is no longer sticking relative to the document/viewport, but relative to that transformed wrapper's own box. If that wrapper's box happens to be as tall as (or taller than) the whole page in this specific layout on desktop, the sticky behavior *looks* identical to viewport-relative sticking (there's enough room within the transformed ancestor for the sticky offset to ever engage) — but the transformed wrapper on the actual mobile layout (different content flow, different breakpoint) ends up shorter, or the sticky element ends up outside the visible portion of that containing block's scrollable area sooner, causing it to detach and scroll away.

This is why desktop emulation didn't repro it: the difference isn't about touch or viewport width per se, it's that the mobile layout's specific content arrangement changes how tall the transformed containing block is relative to where the sticky element sits within it — a downstream, layout-dependent consequence of the same underlying containing-block bug, not a WebKit-only rendering quirk. (Both explanations are worth knowing: sometimes this class of bug is a real WebKit-specific transform/overflow interaction, and sometimes, like here, it's the same root cause across engines but only visibly manifesting on one specific layout.)

## The Fix

**Move the `transform: translateZ(0)` off the ancestor that contains the sticky header, onto a sibling or a more specific target that doesn't sit in the sticky element's containing-block chain.**

```css
/* Before — transform on a shared ancestor breaks sticky descendants */
.page-wrapper {
  transform: translateZ(0); /* was meant for scroll perf, not layout */
}
.page-wrapper .site-header {
  position: sticky;
  top: 0;
}
```

```css
/* After — the perf hint is scoped to only the element that actually needs it */
.scroll-perf-target {
  transform: translateZ(0);
}
.site-header {
  position: sticky;
  top: 0;
  /* no longer inside an ancestor establishing a new containing block */
}
```

If the GPU-acceleration hint is still needed for scroll performance elsewhere on the page, `will-change: transform` or `translateZ(0)` should be applied narrowly — to the specific element whose paint performance is the actual concern (e.g., a long scrolling list below the header) — rather than to a broad structural wrapper that happens to also contain unrelated positioned elements.

**Additionally, verify no `overflow` ancestor is compounding the issue**, since both causes are common enough to co-occur:

```css
/* Any of these on an ancestor redefines sticky's scroll boundary */
.some-ancestor {
  overflow: hidden; /* or auto, or scroll */
}
```

If such an ancestor exists and is load-bearing for its own reasons (e.g., clipping a decorative background), the sticky element needs to either move outside that ancestor in the DOM, or the ancestor's overflow needs to be reworked (e.g., `overflow: clip` on a more narrowly scoped inner wrapper instead of the whole section).

> **Check yourself:** If the transformed ancestor's `transform` genuinely can't be removed or relocated (say, it's essential for an unrelated animation on that exact element), what's the fix for the sticky header — and why does moving the sticky element earlier/later in the DOM, rather than changing any CSS property on it, potentially solve this?

## Gotchas

**Fixing the immediate ancestor's `overflow`/`transform` without checking *every* ancestor up to the scrolling container/viewport.** Sticky positioning resolves against the *nearest* qualifying ancestor, which could be several levels removed from the sticky element itself — a candidate who checks only the direct parent and stops there can miss a further-up ancestor that's the actual cause, especially in component-based codebases where wrapper divs from shared layout components are easy to overlook.

**Not testing on an actual mobile Safari instance (real device or Safari's own responsive mode) and declaring the bug fixed based on Chrome's device emulation alone** — as established, Chrome's emulator doesn't reproduce WebKit-specific containing-block or scroll-related rendering behavior, so "it looks fixed in Chrome's mobile view" is not evidence the actual reported bug is resolved.

**Reintroducing the same class of bug later by adding an animation library, a "reveal on scroll" effect, or a parallax section that applies `transform` to a wrapper without realizing it's an ancestor of a sticky element elsewhere on the page** — this bug class tends to recur because the containing-block rule is non-obvious and easy for a different engineer, working on an unrelated feature, to trip weeks or months later.

**Forgetting `scroll-margin-top`/content padding once the header sticks correctly**, so that anchor-linked scrolling (`#section-2`) or a user's manual scroll-to-top now lands content directly underneath the sticky header, visually hidden behind it — a fix that resolves "does it stick" without also resolving "is content still readable once it does."

**Assuming `-webkit-sticky` is still required as a vendor prefix for current Safari** — this was true for older Safari versions but is no longer necessary in modern evergreen Safari; including it unconditionally is harmless but a candidate citing it as "the fix" for a modern-Safari bug is diagnosing the wrong problem (vendor-prefix history, not the actual containing-block issue at hand).

## Follow-up Questions

**Q (High): State precisely what `position: sticky` positions itself relative to, and name the two distinct categories of ancestor property that change that reference.**

Answer: A `position: sticky` element behaves like `position: relative` within normal flow until it would scroll past its specified offset (e.g., `top: 0`) within its *nearest ancestor that establishes a scrolling mechanism* — at that point it switches to behaving like `position: fixed`, but anchored to that scrolling ancestor's box, not necessarily the viewport. Two categories of ancestor property change this reference point: first, any `overflow` value other than `visible` (`hidden`, `auto`, `scroll`, and `clip` in some cases) on an ancestor makes that ancestor sticky's scroll boundary instead of the viewport/document; second, any property that establishes a new containing block on an ancestor — `transform` (any non-`none` value), `filter`, `perspective`, `will-change: transform`, or `contain: layout`/`paint` — changes the containing block the sticky element's offsets are computed against, which can produce broken or unexpected sticking even without any `overflow` involved at all. Both categories are found by walking up the ancestor chain from the sticky element to the viewport and checking each ancestor for either condition — whichever qualifying ancestor is found *first* (nearest) is the one that matters.

The trap: describing sticky as "just sticks to the top of the viewport" without qualification — that's the *common case* (no interfering ancestor exists), but it's not what the spec actually says, and a candidate who can't name the containing-block category specifically (as opposed to only the overflow category, which is more commonly known) will struggle exactly on this scenario, since the described bug is caused by the less-commonly-known category.

---

**Q (High): Why did Chrome's device emulation mode fail to reproduce a bug that's real and reproducible on an actual iPhone?**

Answer: Device emulation in Chrome DevTools changes the reported viewport dimensions, pixel density, user-agent string, and touch-event dispatch — enough to test responsive breakpoints and touch interaction logic — but the page is still being laid out and painted by Chrome's own rendering engine (Blink), not Safari's (WebKit). Any bug whose root cause is in how a specific engine computes containing blocks, resolves scroll boundaries for sticky/fixed elements, or handles engine-specific quirks (historically, various sticky-positioning edge cases have differed between Blink and WebKit) simply won't manifest under emulation, because the emulator doesn't swap out the underlying layout engine — it only changes inputs (viewport size, touch capability) to the same engine. In this specific scenario, the bug additionally depended on the mobile layout's specific content height inside a transformed containing block being different from the desktop layout's — a responsive-content difference that *could* in principle show up under emulation if the emulated viewport width altered the content flow enough, but here didn't fully reproduce until tested on the actual target engine and device.

The trap: treating "doesn't repro under Chrome mobile emulation" as evidence the bug report is inaccurate or environment-specific noise, rather than recognizing that emulation only covers a subset of what differs between a desktop browser and an actual mobile device — real-device or same-engine (Safari's own dev tools/responsive mode) testing is often required to close out engine-specific bugs credibly.

---

**Q (Medium): The `transform: translateZ(0)` on the ancestor was originally added to fix a scroll-jank problem on iOS. If it's removed to fix the sticky header, how would you verify the original jank problem doesn't come back?**

Answer: I'd reintroduce the specific performance optimization the transform was meant to provide, but scoped to only the element that actually needs GPU compositing — typically an element with expensive repeated paints during scroll (a long list with box-shadows, a section with a background gradient, an image-heavy grid) — using `will-change: transform` or `translate3d(0,0,0)`/`translateZ(0)` on that specific element rather than a shared structural ancestor. Then I'd verify the fix addresses the actual original symptom by profiling scroll performance (Safari's Web Inspector timeline, or Chrome's Performance panel as a proxy, understanding its limits per the previous answer) before and after, checking for dropped frames / long paint times during scroll specifically in the area the original optimization targeted, on a real device matching what jank was originally reported on — not assuming removing the transform from the wrong element silently also removed the benefit it was providing elsewhere.

The trap: removing the transform entirely to fix sticky without confirming what specific problem it was solving in the first place, and shipping a fix that resolves the reported sticky bug while silently reintroducing a scroll-jank regression that either goes unnoticed until user complaints resurface, or — worse — was never clearly attributed to that transform in the first place, making it hard to know it's safe to remove.

---

**Q (Medium): How would `position: sticky` interact with a parent that has `display: flex` and `align-items: stretch` (the default)? Could that cause a sticky element to silently not stick even with no overflow or transform ancestor involved?**

Answer: Yes — if the sticky element is a flex item and its flex container's cross-axis alignment stretches it to fill the container's full height (the default `align-items: stretch`), the sticky element's own box can end up as tall as its flex container, leaving no room for it to "travel" within that container as the page scrolls — sticky positioning needs the element's containing block to be taller than the sticky element itself for the offset to ever have room to take effect; if they're the same height, there's nothing for it to stick relative to, since it never has room to move before hitting its constraint. This is a distinct cause from both the overflow-ancestor and transform-containing-block causes covered above — no `overflow` or `transform` needs to be involved at all, purely a "the sticky element's positioning context isn't actually taller than the element" sizing issue, common in flex/grid layouts where `align-items: stretch` or `align-self: stretch` is left at its default.

The trap: treating "sticky broken" as always attributable to the two well-known causes (overflow ancestor, transform-created containing block) and missing a third, sizing-based cause that produces the identical symptom (the element doesn't stick) through an entirely different mechanism — a thorough debugging process checks the sticky element's own box height relative to its immediate positioning context, not just ancestor `overflow`/`transform` properties.

---

**Q (Low): If this sticky header needs to also account for iOS Safari's collapsing/expanding address bar changing the visible viewport height while scrolling, what CSS unit or technique would you reach for, and why does plain `vh` fall short?**

Answer: Plain `vh` in iOS Safari is historically defined relative to the *largest possible* viewport (the state when the address bar is collapsed/hidden), which means a `100vh` element can be taller than the actually-visible viewport when the address bar is expanded, causing content or a sticky element's available room to be miscalculated relative to what's really visible at any given scroll position. The modern fix is the `dvh` (dynamic viewport height) unit, which continuously tracks the *current* visible viewport height as the address bar collapses and expands during scroll, giving layout calculations that stay accurate through that transition rather than being pinned to one specific (largest) viewport state; `svh` (small viewport height, address bar always expanded) and `lvh` (large viewport height, always collapsed) are the two fixed bounds `dvh` interpolates between, useful when a layout specifically needs the guaranteed-worst-case or guaranteed-best-case number rather than the live-tracking one.

The trap: reaching for JavaScript (a resize/scroll listener recalculating `window.innerHeight` and setting a CSS custom property) as the only known fix — a reasonable and historically necessary workaround before `dvh` shipped broadly, but a candidate unaware `dvh` exists and solves this natively will describe meaningfully more complexity (event listeners, debouncing, a JS-to-CSS bridge) than the modern one-line CSS answer, which is worth knowing given how recently `dvh` became safe to rely on as a baseline.

---

## Self-Assessment

- [ ] Can state precisely what `position: sticky` resolves its offset against, including both the overflow-ancestor and containing-block-ancestor cases
- [ ] Can explain why Chrome's device emulation mode doesn't reproduce genuine WebKit-specific or containing-block-related bugs
- [ ] Can identify `transform`/`filter`/`perspective`/`will-change` on an ancestor as a sticky-breaking cause, not just `overflow`
- [ ] Can explain the flex/grid `align-items: stretch` sizing cause as a third, distinct failure mode
- [ ] Can propose a scoped fix (move the transform to the actual element that needs it) rather than a blanket JS reimplementation of sticky
- [ ] Knows `dvh`/`svh`/`lvh` and why plain `vh` is unreliable on iOS Safari during scroll

---
*Next: CLS From Late-loading Images & Fonts — moves from a positioning bug caused by ancestor properties to a layout-shift bug caused by content arriving after initial layout, a different category of "looks fine until real-world timing intervenes."*
