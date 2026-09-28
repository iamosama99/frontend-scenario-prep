# CLS From Late-loading Images & Fonts

## Quick Reference

| Source of Shift | Mechanism | Fix |
|---|---|---|
| Images without dimensions | Browser doesn't know image's aspect ratio until it downloads, allocates 0 height, then reflows everything below it once it arrives | `width`/`height` attributes (or `aspect-ratio` in CSS) reserving space before the image loads |
| Web fonts swapping in (FOUT) | Fallback font metrics differ from the web font's, so text reflows/rewraps once the real font loads | `font-display: optional`/`swap` tuned to the case, plus a metrically-compatible fallback font (`size-adjust`, `ascent-override`, etc.) |
| Ads / embeds injected after load | Third-party script inserts content into a container with no reserved size | Reserve space with a fixed-size container/aspect-ratio box before the script runs |
| Dynamically injected banners (cookie notice, promo bar) | Content inserted above existing content pushes everything down | Reserve space up front, or position the banner so it overlays rather than pushes |
| Client-side-rendered content replacing a skeleton of different size | Skeleton's dimensions don't match the real content's eventual dimensions | Size the skeleton to match real content's layout as closely as possible |

## The Scenario

"Our Core Web Vitals report shows a Cumulative Layout Shift score well above the 'needs improvement' threshold on our article pages. Watching a session recording, I can see the page load, then the whole thing jumps down as a hero image pops in, and the body text visibly reflows a moment later too. Find every source of layout shift on this page and fix them — and explain how you'd verify each fix actually reduces CLS, not just how you'd guess at reducing it."

## Clarifying Questions

- **Are the images served with explicit dimensions in the markup (`width`/`height` attributes or CSS `aspect-ratio`), or is the layout relying on the image's natural dimensions once loaded?** This is the single highest-signal question — if no dimensions are reserved anywhere, that alone likely accounts for most of the observed shift, and is worth confirming before investigating anything more subtle like font swapping.
- **Is a custom web font in use, and if so, what's the current `font-display` value (or is it left at the default)?** `font-display: auto` (browser-decided, often behaving like `block` — text invisible until the font loads or a timeout passes) and `font-display: swap` (fallback text shown immediately, swapped once the web font loads) produce different CLS profiles — `swap` guarantees a reflow the instant the real font arrives if its metrics differ from the fallback's, which needs to be measured, not assumed away.
- **Is the layout shift consistent across users, or does it vary with connection speed?** A shift that's worse on slow connections points toward "late-arriving resource with no reserved space" (images, fonts, third-party embeds — the later something arrives, the more content has already rendered and needs to shift out of its way); a shift that's consistent regardless of speed points more toward something structural (a CSS animation running on load, a client-side layout recalculation independent of network timing).
- **Are there any third-party scripts (ads, comment widgets, social embeds, cookie-consent banners) on this page, and do any of them inject content into the DOM after initial render?** These are extremely common, often-overlooked CLS sources precisely because they're not "our" code and don't show up when reviewing just the article template — a consent banner sliding in from the top and pushing content down is one of the most common real-world CLS root causes.
- **What tool and methodology produced the CLS score being referenced — lab data (Lighthouse, a single simulated run) or field data (real user CrUX/RUM data)?** Lab data reflects one deterministic run under one simulated condition; field data aggregates real users' actual connections and devices, and a discrepancy between the two (e.g., lab score looks fine but field score is bad) usually points at something connection-speed-dependent that a fast, controlled lab environment doesn't trigger the same way.

## Approach & Trade-offs

**Treat CLS as "space wasn't reserved before content arrived" as the unifying root cause across every seemingly different case — images, fonts, third-party embeds, dynamically injected banners — rather than debugging each as an unrelated one-off.** Every distinct CLS source in the Quick Reference table reduces to the same underlying problem: the browser lays out the page with the information available at that moment, and if a later-arriving piece of content (an image's real dimensions, a web font's real character widths, a script's injected DOM, an async data fetch's result) turns out to need different space than what was reserved (often, none), something already-rendered has to move to make room. Recognizing this as one root cause, not four, is what lets a fix generalize — the fix is always "reserve the eventually-correct amount of space up front," whether that's `width`/`height` on an `<img>`, a metrically-matched fallback font, or a fixed-size ad slot container.

**Fix images first, since they're the highest-leverage, lowest-risk change, and the scenario's own description ("hero image pops in") points there directly.** An image with no `width`/`height` attributes and no CSS `aspect-ratio` is laid out by the browser with zero intrinsic height reserved (unless the surrounding CSS otherwise constrains it), because the browser genuinely doesn't know the image's dimensions until enough of the file has downloaded to read them — everything below the image then occupies the space the image will eventually need, and has to shift down once the image arrives and claims its real height. Reserving the aspect ratio up front (even without knowing the final rendered size, since aspect ratio plus a responsive width is enough to compute height) eliminates this category of shift entirely, deterministically, regardless of network speed — unlike font-related shifts, which involve trickier trade-offs discussed next.

**Font-related shift needs a trade-off decision, not just a mechanical fix, because `font-display` values trade off differently between "invisible text" and "guaranteed reflow."** `font-display: block` (or `auto`, which commonly behaves similarly) hides text for a short period while the web font loads, avoiding a visible swap-triggered reflow but introducing an invisible-text period (a different UX cost, and its own performance metric concern — it can delay meaningful content from appearing at all). `font-display: swap` shows fallback text immediately (good for perceived performance and avoiding invisible text) but guarantees a reflow the moment the real font loads if the fallback's character metrics (width, line-height) differ meaningfully from the web font's — which is almost always true unless deliberately prevented. The actual fix that avoids the trade-off rather than picking a side is choosing (or generating, via tools like Fontaine or manually tuning `size-adjust`/`ascent-override`/`descent-override`) a fallback font whose metrics are close enough to the web font's that the swap causes negligible or zero reflow — turning "avoid invisible text vs. avoid layout shift" from a forced trade-off into a problem that's actually solvable on both axes at once.

**Verification has to be measurement-based, not visual inspection, because CLS is a precise numeric score computed from actual shift distance and viewport-fraction impact — "looks better" isn't the same as "measurably better."** After each fix, I'd re-run Lighthouse (lab) to confirm the specific fix's shift no longer appears in its layout-shift trace (Lighthouse attributes each shift to the specific DOM node that moved, which tells me directly whether the image-dimension fix or font fix worked), and separately watch real-user CLS in field data (CrUX, or an RUM tool like web-vitals.js reporting to an analytics endpoint) over the following days, since lab data alone can miss connection-speed-dependent shifts that only manifest for a meaningful fraction of real users on slower connections.

## Solution

**Step 1 — reserve space for every image via explicit dimensions or CSS `aspect-ratio`, so the browser can compute layout before the image downloads:**

```html
<!-- Before: no reserved space, browser doesn't know height until image loads -->
<img src="/hero.jpg" alt="Article hero image" class="hero-image" />

<!-- After: intrinsic aspect ratio known immediately from attributes -->
<img src="/hero.jpg" alt="Article hero image" class="hero-image" width="1200" height="630" />
```

```css
/* The width/height attributes set the *intrinsic* aspect ratio;
   CSS still controls the *rendered* size responsively */
.hero-image {
  width: 100%;
  height: auto; /* height is derived from width via the intrinsic aspect ratio — no shift */
}
```

Modern browsers compute an element's aspect ratio from the `width`/`height` *attributes* (not just their pixel values as fixed sizes) and use that to reserve correctly-proportioned space even when CSS scales the image responsively — this is why both the HTML attributes and the `height: auto` CSS are needed together, not either alone.

For images whose dimensions aren't known ahead of time (e.g., user-uploaded content of varying aspect ratios), reserve space with CSS `aspect-ratio` directly, sourced from whatever metadata is available (even a stored average, or the actual per-image ratio from upload-time processing):

```css
.user-uploaded-image-container {
  aspect-ratio: 16 / 9; /* or a per-image value if known at render time */
}
```

**Step 2 — fix the font-swap reflow with a metrically-matched fallback, so `font-display: swap` doesn't visibly shift text:**

```css
@font-face {
  font-family: 'Brand Sans';
  src: url('/fonts/brand-sans.woff2') format('woff2');
  font-display: swap; /* show fallback immediately, swap when ready */
}

@font-face {
  font-family: 'Brand Sans Fallback';
  src: local('Arial');
  /* Tuned via a tool (e.g., Fontaine, Capsize) to match Brand Sans's metrics closely */
  size-adjust: 105%;
  ascent-override: 90%;
  descent-override: 22%;
}

body {
  font-family: 'Brand Sans', 'Brand Sans Fallback', sans-serif;
}
```

The `size-adjust`/`ascent-override`/`descent-override` values scale the fallback font's box metrics to approximate the real web font's — so text set in the fallback occupies nearly the same line height and character width as it will once the real font swaps in, making the swap itself produce negligible or zero visible reflow instead of a full re-layout.

**Step 3 — reserve space for third-party embeds and dynamically injected banners before their scripts run:**

```css
/* A fixed-height slot reserved before the ad script injects its content */
.ad-slot {
  min-height: 250px; /* matches the ad unit's known size */
  width: 300px;
}
```

```css
/* A cookie-consent banner overlays rather than pushing content down */
.consent-banner {
  position: fixed;
  bottom: 0;
  /* overlay, doesn't participate in document flow — nothing shifts to make room */
}
```

Where the banner's design genuinely requires pushing content (not overlaying), reserving its height in the initial server-rendered markup — rendering the banner's container (even empty, or with a skeleton) at first paint rather than injecting it after a client-side check — avoids the late-insertion shift entirely.

> **Check yourself:** Why does giving an `<img>` explicit `width`/`height` attributes still allow it to be responsive (scale with its container) without reintroducing layout shift, when naively it seems like fixed attributes should force a fixed size?

## Gotchas

**Fixing image dimensions but leaving `object-fit` unset or set incorrectly for images whose actual aspect ratio doesn't match the reserved box** — reserving the wrong aspect ratio (e.g., a stale `width`/`height` from a previous version of the image) still causes a shift once the image loads and doesn't match, and `object-fit: cover`/`contain` handles *visual* cropping/letterboxing within a correctly-sized box, but doesn't fix an incorrectly-sized box in the first place.

**Choosing `font-display: swap` and stopping there, without addressing the fallback-font metric mismatch** — `swap` alone guarantees a swap will happen (rather than hiding text and possibly never swapping visibly, as some `auto`/`block` behaviors can produce), but doesn't by itself reduce the *shift* caused by that swap; a candidate who cites `font-display: swap` as "the fix" for CLS without the metric-matching fallback step has only solved the invisible-text problem, not the layout-shift problem — they're separate concerns that happen to both involve fonts.

**Reserving space for the *first* viewport's images but missing images further down the page that load lazily as the user scrolls** — `loading="lazy"` images still need reserved dimensions; if they don't have them, scrolling down triggers the exact same shift pattern the hero image had, just delayed until the user reaches that section, which won't show up in a Lighthouse run that only measures shift during initial load unless the tool is configured to scroll/interact during the trace.

**Not accounting for CLS caused by content unrelated to images/fonts at all** — client-side data fetches that replace a skeleton/loading state with real content of a different size, a "read more" expansion, or a CSS transition/animation that moves layout-affecting properties (`margin`, `width`, `top` rather than `transform`) all contribute to CLS and are easy to miss if the investigation stops as soon as the two causes named in the prompt (images, fonts) are addressed — the prompt says "find every source," which is itself testing whether the candidate stops at the two hinted-at causes or does a genuinely complete audit.

**Measuring CLS improvement only via a single Lighthouse run immediately after the fix**, without accounting for Lighthouse's run-to-run variance (network simulation isn't perfectly deterministic) or without also checking field/RUM data over a longer window — a single before/after lab comparison can show improvement that doesn't hold up, or can miss real-user-only shifts caused by conditions the lab run doesn't simulate (slow real connections, specific real devices, ad-network variability).

## Follow-up Questions

**Q (High): Explain exactly how the CLS metric is calculated — what makes a layout shift count toward the score, and what deliberately does NOT count?**

Answer: CLS sums a series of "layout shift scores" for unexpected shifts that occur during the page's lifecycle (with some nuance around session windows, but conceptually: shifts are grouped and the largest burst is what's reported). Each individual shift's score is the product of two factors: the "impact fraction" (what fraction of the viewport was visibly affected by the shift, i.e., the union of the before- and after-shift positions of the elements that moved, relative to viewport area) and the "distance fraction" (how far, as a fraction of the viewport's largest dimension, the affected content moved). Critically, a shift only counts if it's "unexpected" — the spec explicitly excludes shifts caused by user interaction (a shift within 500ms of a click, tap, or key press is not counted, since the user caused it and it isn't a surprise to them) and shifts from CSS `transform`-based animations, since `transform` doesn't trigger layout at all — it's a compositor-only property, so animating `transform` never produces a "layout shift" in the first place, which is exactly why `transform`-based animations are the recommended way to animate position/size changes without incurring CLS penalty.

The trap: describing CLS as simply "how much stuff moves" without the impact-fraction × distance-fraction formula, or forgetting the user-interaction exclusion — a candidate who doesn't know shifts within 500ms of user input are excluded might incorrectly diagnose an accordion-expand or tab-switch interaction as a CLS problem worth "fixing," when it's already excluded from the metric by design (though it may still be worth fixing for UX reasons unrelated to the CLS score itself).

---

**Q (High): Why does using `transform: translateY()` instead of animating `top`/`margin-top` avoid contributing to CLS, given that both visually move an element the same way?**

Answer: `top` and `margin-top` are properties that affect the box model / document flow — changing them triggers the browser's layout stage (recalculating the positions of the element and everything affected by it), which is precisely the kind of "unexpected layout shift" the CLS metric is designed to detect and penalize, since sibling/descendant elements' actual layout positions are changing over time. `transform`, by contrast, is applied entirely in the compositor stage, after layout has already happened — it visually repositions the *painted* result of an element without changing anything in the layout tree, so no other element's layout position is ever affected, and there's nothing for the CLS algorithm (which specifically measures layout-tree position changes) to detect. This is also why `transform`-based animation is dramatically better for performance generally, independent of CLS — it can run on the compositor thread without triggering main-thread layout/paint work per frame, avoiding jank, whereas animating `top`/`width`/`margin` forces layout recalculation on every frame of the animation.

The trap: assuming any visual movement counts toward CLS regardless of mechanism — the metric is specifically about layout-tree shifts, not visual movement in general, and understanding that distinction (layout-affecting properties vs. compositor-only properties) is what explains why the "always animate transform/opacity, never top/left/margin/width" performance advice and the CLS-avoidance advice are actually the same underlying principle, not two coincidentally similar rules.

---

**Q (Medium): The hero image's `width`/`height` attributes are set correctly, but the image is inside a container with `display: flex` and no explicit sizing on the container itself. Could layout shift still occur, and why?**

Answer: Yes, in a subtler way — if the flex container's own size is influenced by its children (e.g., it has no explicit `height` and is sized by content, `align-items: stretch` interacting with siblings, or `flex-basis: auto` deriving from content), and other content in that same flex context loads/changes size asynchronously (a sibling's async-loaded content, a font swap affecting a text sibling's height), the container itself can resize even though the image's own reserved aspect ratio is respected — from the image's perspective nothing shifted, but from the *page's* perspective, content still moved because the shift originated from a sibling within the same flex layout context, propagating a resize to the shared container and everything positioned relative to it. This is why "did I give this specific element correct dimensions" isn't a complete CLS audit on its own — the surrounding layout context (what else can resize this container, does the container have any dependency on dynamically-sized content) also needs to be accounted for, particularly in flex/grid layouts where sizing is often intentionally interdependent across siblings.

The trap: verifying only the specific element named in the bug report (the hero image) has correct dimensions and declaring the fix complete, without checking whether that element's *layout context* (its container, its siblings sharing that container) has its own independent source of size instability that could still produce a shift even with the image itself behaving correctly.

---

**Q (Medium): A marketing team wants to A/B test hero images of genuinely different aspect ratios (some square, some widescreen) without knowing ahead of time which variant a given user will see. How would you reserve layout space without knowing the final aspect ratio until the experiment assignment resolves client-side?**

Answer: I'd reserve space using the *worst-case* (tallest, most conservative) aspect ratio among the possible variants as the default/skeleton state, sized via CSS `aspect-ratio` rather than per-image `width`/`height` attributes (since those are meant to reflect one specific known image's intrinsic ratio, not a range of possibilities) — accepting that some variants will render with slightly more empty space than strictly needed for that specific image, in exchange for guaranteeing no variant ever needs *more* space than what's reserved, which is what actually prevents shift. Alternatively, if the experiment assignment can be resolved server-side or edge-side (before the HTML is sent) rather than purely client-side, the correct aspect ratio for the assigned variant can be embedded directly in the initial markup, avoiding the need for a conservative worst-case reservation at all — a better outcome if the experiment infrastructure supports it, since it means every variant gets its own precisely-reserved space rather than sharing a padded, conservative one.

The trap: reserving space based on whichever variant happens to be tested first or most often during development, rather than deliberately accounting for the full range of possible variants — an A/B test is specifically a case where "the dimensions I saw during development" and "the dimensions a real user's assigned variant will have" can diverge, and CLS fixes that don't account for that will pass local testing while still shifting for a meaningful fraction of real users assigned a different variant.

---

**Q (Low): How would Lighthouse's CLS score for a page differ from that same page's real-user (field) CLS score in Chrome UX Report (CrUX), and why might they legitimately disagree?**

Answer: Lighthouse's CLS is a single lab measurement under one simulated network/CPU condition and one simulated session (typically just the initial load, not extended user interaction), while CrUX's field CLS aggregates real Chrome users' actual page visits across genuinely varied devices, connections, and interaction patterns, using the full "session window" methodology (which groups shifts occurring close together in time and takes the largest single window's cumulative score, rather than summing every shift across the entire page lifetime). The two can legitimately disagree in either direction: a page can lab-test well if the specific simulated conditions happen not to trigger a connection-speed-dependent shift (e.g., a font-swap shift that only manifests when the font takes long enough to load relative to first paint) that shows up for a meaningful fraction of real users on slower connections; conversely, a page can field-score better than its lab score if real users frequently interact early (triggering the user-interaction exclusion window) in ways a scripted lab run doesn't simulate. This divergence is exactly why relying on lab data alone for a production CLS investigation is risky — field data is the ground truth for what real users experience, and lab data is a useful, fast, reproducible proxy for iterating on fixes, not a substitute for confirming the fix actually worked in the field.

The trap: treating a good Lighthouse score as confirmation the CLS problem is solved without checking field data over a representative window — the scenario's original signal came from "our Core Web Vitals report," which is very likely field/CrUX-sourced, so the actual bar for "fixed" is field data improving, not just a clean local Lighthouse run.

---

## Self-Assessment

- [ ] Can state the CLS formula (impact fraction × distance fraction) and the user-interaction/transform exclusions from memory
- [ ] Can explain why `width`/`height` attributes plus `height: auto` in CSS reserve space without breaking responsiveness
- [ ] Can explain the font-swap trade-off (`font-display` values) and why metric-matched fallback fonts solve it more completely than `font-display` alone
- [ ] Can identify third-party embeds and dynamically-injected banners as CLS sources beyond the two named in the prompt
- [ ] Can explain why `transform`-based animation avoids CLS while `top`/`margin`-based animation doesn't
- [ ] Can explain the lab-vs-field CLS discrepancy and why field data is the actual source of truth

---
*Next: Responsive Grid for an Unknown Item Count — moves from preventing shift in a known layout to building a layout that must remain correct and shift-free across an item count that isn't known ahead of time.*
