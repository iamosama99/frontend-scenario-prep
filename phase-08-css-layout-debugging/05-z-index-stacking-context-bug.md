# Z-index / Stacking Context Bug

## Quick Reference

| Symptom | Root Cause | Fix |
|---|---|---|
| Element with `z-index: 9999` still renders behind something with a lower z-index | Both elements are in *different* stacking contexts, and z-index only compares siblings within the same context | Raise the z-index of the correct ancestor stacking context, not the descendant, or restructure so both compete in the same context |
| Modal/dropdown appears behind page content despite high z-index | An ancestor of the modal has `transform`, `opacity < 1`, `filter`, or similar, silently creating a new stacking context that traps the modal's z-index inside it | Remove the trapping property from the ancestor, or portal the modal out of that ancestor's subtree |
| Increasing z-index further has no effect ("z-index war") | The elements are already correctly ordered within their local contexts; the actual comparison happening is between two *ancestor* contexts elsewhere in the tree | Find and fix the actual competing ancestor stacking contexts instead of continuing to raise descendant z-index values |
| A `position: fixed` element gets clipped/scrolled with the page instead of staying fixed to viewport | An ancestor has `transform`/`filter`/`will-change` set, which redefines the containing block for `fixed` descendants, not just their stacking order | Same fix as sticky's containing-block issue in [[02-sticky-header-broken-mobile-safari]] — remove or relocate the property |
| Two sibling elements with the same z-index render in an unexpected order | Default stacking falls back to DOM order among equal z-index siblings in the same context | Give one an explicit higher z-index, or reorder the DOM if paint order should match visual/interaction order |

## The Scenario

"We have a dropdown menu component. It has `z-index: 1000` and is supposed to appear above everything else on the page. On most pages it works fine. On one specific page, it renders *behind* a card component that only has `z-index: 2`. Bumping the dropdown's z-index to 99999 doesn't fix it. Figure out what's actually going on — not just a z-index number that happens to work — and fix it in a way that won't silently break again the next time someone adds an unrelated `transform` somewhere on the page."

## Clarifying Questions

- **Does the card with `z-index: 2` (or any of its ancestors) have a `transform`, `opacity` less than 1, `filter`, `perspective`, `will-change`, `mix-blend-mode`, `contain: layout` or similar set anywhere between it and a shared ancestor with the dropdown?** This is the single most likely cause and worth checking first — any of these properties, even set for a completely unrelated purpose (an entrance animation, a hover effect, a performance hint), creates a new stacking context, which changes what `z-index` values are even being compared against each other.
- **Is the dropdown rendered in-place in the DOM (as a child of whatever triggers it) or rendered via a portal (e.g., React `createPortal`) to a location like `document.body`?** If it's in-place, it inherits whatever stacking context its ancestors establish, which is exactly the kind of scenario where an unrelated ancestor's `transform` can trap it; if it's portaled, the bug is more likely something else entirely (a competing high z-index on the portal target itself, or the portal target being placed before an element with its own high z-index sibling).
- **Does "one specific page" have something structurally different from the pages where the dropdown works correctly** — a different layout wrapper, an A/B test variant, an animated hero section, a carousel library, or any third-party component that might apply its own CSS to a shared ancestor? Since the bug is page-specific rather than universal, the cause is very likely something present on that one page's ancestor chain and absent elsewhere, rather than anything inherent to the dropdown component itself.
- **What does browser DevTools' "Layers" panel (Chrome) or 3D view show for this page** — is there a visibly separate compositing layer around the card or one of its ancestors? This is a direct, authoritative way to confirm a stacking-context-creating property is in play, rather than manually re-deriving it by reading CSS, and is worth asking whether it's already been checked before doing manual investigation.
- **Is z-index being used anywhere in this codebase without any documented/agreed-upon scale (e.g., an arbitrary "1000", "2", "99999" mix, as implied by the numbers in the prompt), or is there a project-wide z-index token system already in place that this bug is violating?** If there's no existing system, the long-term fix (beyond this one bug) may need to include establishing one — an unbounded, ad hoc z-index scale is exactly what produces recurring "z-index wars" like the one implied by "bumping to 99999 doesn't help."

## Approach & Trade-offs

**Diagnose by finding the nearest shared stacking-context ancestor of the two competing elements, not by comparing their z-index values directly — because z-index comparisons only happen between elements that are stacking-context siblings, and the two elements here almost certainly aren't.** The core misconception the "bump z-index to 99999, still doesn't work" symptom reveals is treating z-index as a single global, page-wide ordering — it isn't. Each stacking context is its own self-contained ordering universe: elements are painted back-to-front within a context according to a well-defined order (negative z-index, then normal-flow non-positioned content, then positioned content with `z-index: auto`, then positive z-index in ascending order, roughly), but a *whole stacking context*, once created, is painted as a single unit at whatever position its own z-index (or default stacking order) places it within *its parent* context. This means an element with `z-index: 1000` deep inside stacking-context A can never visually appear above an element with `z-index: 2` in a sibling stacking-context B if context A itself is painted before context B — no z-index value inside A, however large, escapes A's own position in the painting order of A's parent. The fix has to happen at the level where A and B (or their respective ancestor contexts) are actually being compared, which requires walking up the DOM tree from both elements to find where a new stacking context was introduced, not adjusting the leaf elements' own z-index indefinitely.

**Use browser DevTools directly rather than re-deriving the stacking order by reading CSS in your head, because stacking-context-creating properties are numerous, easy to apply unintentionally, and easy to miss on a quick read.** Modern Chrome DevTools has a dedicated "Layers" panel (and an z-index/stacking-context indicator directly in the Elements panel on relevant nodes in recent versions) that shows exactly which elements establish their own stacking context and why — this turns "guess which ancestor has a `transform`" into "look at the tool's answer directly," which is both faster and more reliable than manually re-deriving the full list of context-creating CSS properties (there are more than a dozen: `position` + non-auto `z-index`, `opacity < 1`, `transform`, `filter`, `backdrop-filter`, `perspective`, `clip-path`, `mask`, `mix-blend-mode` other than `normal`, `isolation: isolate`, `will-change` naming any of the above, `contain: layout/paint/strict/content`, and flex/grid items with non-`auto` `z-index`, among others) live, from memory, under time pressure.

**Fix at the actual source (the property creating the unwanted stacking context) rather than the symptom (the dropdown's own z-index) — otherwise the fix is fragile and the bug class recurs.** Continuing to raise the dropdown's own z-index, or worse, wrapping the dropdown in its own `position: fixed; z-index: 999999` "just in case" without understanding why that's needed, treats the symptom without addressing why an unrelated ancestor's styling was able to trap it in the first place — the next engineer who adds a `transform` to some other ancestor for an unrelated animation can reintroduce the exact same bug, because the actual constraint (dropdowns must not be nested inside anything that creates a stacking context) was never made explicit or structurally enforced. The more durable fix, especially for an interactive overlay component like a dropdown, tooltip, or modal, is to render it via a portal to a stable, top-level DOM location (`document.body` or a dedicated `#overlay-root`) specifically so its stacking context is never subject to whatever arbitrary CSS gets applied to components elsewhere in the tree — this is a structural fix, not a numeric one, and is why professional component libraries (Radix, Headless UI, MUI) portal their overlay components by default rather than relying on z-index alone.

## Root Cause (For This Scenario)

Investigating the specific page reveals that the card component sits inside a carousel wrapper that applies `transform: translateX(-33%)` to position the currently-active slide (a completely ordinary, unrelated carousel implementation detail) — and that `transform` on the carousel wrapper establishes a new stacking context. Because the dropdown is rendered in-place as a descendant of a *different* ancestor that does not create its own competing stacking context — but that ancestor happens to be painted, at the root of the page, *before* the carousel wrapper in DOM/stacking order — the carousel's entire stacking context (including the card and its low `z-index: 2`) gets painted as a unit *after* the dropdown's context, regardless of the dropdown's much higher `z-index: 1000`. The dropdown's `z-index: 1000` is real and does its job correctly — but only within the stacking context it's actually a member of; it has no ability to reach into or reorder relative to a sibling stacking context established by the carousel elsewhere in the tree, which is the entire reason bumping it to 99999 doesn't change anything.

## The Fix

**Option A — portal the dropdown to a stable top-level DOM node, removing it from the page's regular layout/stacking hierarchy entirely:**

```tsx
import { createPortal } from 'react-dom';

function Dropdown({ children, isOpen }: { children: React.ReactNode; isOpen: boolean }) {
  if (!isOpen) return null;
  return createPortal(
    <div className="dropdown-menu" style={{ position: 'fixed', zIndex: 1000 }}>
      {children}
    </div>,
    document.getElementById('overlay-root')! // a dedicated node, sibling to the app root, not nested in it
  );
}
```

```html
<body>
  <div id="app-root"><!-- carousel, cards, everything else lives here --></div>
  <div id="overlay-root"><!-- dropdowns, modals, tooltips render here, unaffected by any app-root ancestor's transform/opacity/filter --></div>
</body>
```

With `#overlay-root` a sibling of `#app-root` at the top of the DOM (not nested inside it), no CSS applied anywhere within `#app-root` — including a carousel's `transform`, a hero section's `opacity` animation, or any future component's `filter` — can ever create a stacking context that traps the dropdown, because the dropdown is no longer a descendant of anything in `#app-root` at all.

**Option B — if portaling isn't feasible (e.g., a design constraint requiring the dropdown to remain in-flow for some layout reason), remove or relocate the stacking-context-creating property on the carousel wrapper specifically:**

```css
/* Before — transform on the wrapper traps everything inside its stacking context */
.carousel-track {
  transform: translateX(-33%);
}
```

```css
/* After — apply the transform to an inner element instead, so the outer
   .carousel-track itself doesn't establish a stacking context */
.carousel-track {
  /* no transform here */
}
.carousel-track__inner {
  transform: translateX(-33%);
}
```

This works only if the card (and anything else needing to compete in the page's outer stacking context) sits outside `.carousel-track__inner`, which may or may not be structurally possible depending on the carousel's actual markup — Option A is more robust precisely because it doesn't depend on that structural constraint holding.

> **Check yourself:** If Option A is used, does the dropdown still need `z-index: 1000`, or could it safely be a much lower number like `z-index: 1`? Reason about what it's now actually competing against inside `#overlay-root`.

## Gotchas

**Treating this as "just bump the number higher" and stopping once *some* number happens to work by luck** — a z-index value that "wins" today, without understanding why, provides no guarantee it keeps winning after any future change to any ancestor's CSS anywhere in the tree; this is precisely the failure mode the scenario's prompt is testing for by explicitly stating that 99999 didn't help and asking for a fix that "won't silently break again."

**Not checking `opacity` as a stacking-context-creating property** — it's less immediately obvious than `transform`, since `opacity` is usually thought of purely as a visual/transparency property, but any `opacity` value less than 1 also establishes a new stacking context, and is an extremely common thing to find on hover-state transitions, fade-in animations, or disabled-state styling on an ancestor, making it a frequent, easy-to-miss cause.

**Assuming stacking contexts are only created by explicitly positioned elements (`position: relative/absolute/fixed` + `z-index`)** — many of the more modern context-creating properties (`filter`, `will-change`, `contain`, `mix-blend-mode`, `isolation: isolate`) apply regardless of an element's `position` value at all, which surprises engineers whose mental model of stacking contexts predates these additions to the CSS spec.

**Fixing the immediate reported case but not searching for other overlay components in the codebase with the same in-place-rendering pattern**, which are all one unrelated ancestor `transform`/`opacity`/`filter` away from reproducing the identical bug on a different page — since the actual root issue is structural (overlay components aren't isolated from arbitrary ancestor styling), a single-instance fix without addressing the pattern leaves the same bug class latent elsewhere.

**Using `isolation: isolate` as a "fix" without understanding it creates a new stacking context itself, potentially trapping something else** — `isolation: isolate` is sometimes reached for specifically to *intentionally* scope a stacking context (e.g., ensuring a component's internal z-index values never leak out to compete with unrelated page elements), which is a legitimate, different use case from this bug — but applying it reflexively without understanding its effect can just relocate the same trapping problem rather than resolve it.

## Follow-up Questions

**Q (High): Name at least six distinct CSS properties/values that create a new stacking context, beyond `position` + `z-index`, and explain why this list matters for debugging z-index issues specifically.**

Answer: Beyond `position: relative/absolute/fixed/sticky` combined with a non-`auto` `z-index`, stacking contexts are also created by: any `opacity` value less than 1; any non-`none` `transform`; any non-`none` `filter`; any non-`none` `backdrop-filter`; any non-`none` `perspective`; any non-`none` `clip-path`; any non-`none` `mask` (or mask-related properties); `mix-blend-mode` set to anything other than `normal`; `isolation: isolate`; `will-change` naming any property that itself would create a stacking context (e.g., `will-change: transform`); `contain` set to `layout`, `paint`, `strict`, or `content`; and flex or grid items with a `z-index` other than `auto` (context creation here depends on being a flex/grid item specifically, not just having z-index in normal flow). This list matters because it means a stacking context can be created by CSS that has nothing to do with layering or z-index at all — an engineer adding a hover `opacity` transition, a GPU-acceleration `transform: translateZ(0)` hint, or a `will-change` performance optimization can unknowingly create a new stacking context on an ancestor, silently changing how every descendant's z-index behaves relative to the rest of the page, without touching the word "z-index" anywhere in their change.

The trap: naming only `position` + `z-index` and `opacity`, the two most commonly known causes, and treating the list as exhaustive — a candidate who can't name `transform`/`filter`/`will-change`/`contain` specifically will struggle to diagnose exactly this scenario's bug (caused by a carousel's `transform`) from first principles, and will fall back to trial-and-error rather than targeted investigation.

---

**Q (High): Two elements are DOM siblings, both `position: relative`, neither has an explicit `z-index`. Which one paints on top, and why?**

Answer: Neither `position: relative` alone (without an explicit non-`auto` `z-index`) creates a new stacking context, so both siblings remain within the same (parent) stacking context, participating in that context's normal painting order — for elements at the same effective stacking level (here, both are positioned but with `z-index: auto`, which is treated as if the element doesn't participate in explicit z-index ordering and instead paints in normal flow order relative to other `z-index: auto` positioned elements), the one that comes *later in DOM order* paints on top of the earlier one. This is the fallback rule worth knowing explicitly: absent any z-index-based ordering distinguishing them, source order is the tiebreaker, which is also why swapping two sibling elements' order in the DOM (without touching any CSS) can visibly change which one appears "in front" if they ever overlap.

The trap: assuming `position: relative` by itself changes stacking order or "brings an element forward" — it doesn't, on its own; `position: relative` only takes an element out of being purely a normal-flow box (allowing `top`/`left`/etc. offsets) and makes it *eligible* to have an explicit z-index meaningfully affect its stacking, but without an explicit z-index value, its position in the paint order relative to same-context siblings remains governed by DOM order, identical to as if it weren't positioned at all.

---

**Q (Medium): The team decides to portal all overlay components (dropdowns, modals, tooltips, toasts) to a shared `#overlay-root`. Now two of them — a modal and a toast notification — need a specific relative order (the toast should always appear above an open modal). How do you manage that without reintroducing an ad hoc z-index numbering war inside the overlay root itself?**

Answer: I'd establish a small, explicit, documented z-index scale specifically for the fixed, known set of overlay component *types* that can render inside `#overlay-root` — e.g., named constants or design tokens (`Z_MODAL = 100`, `Z_TOAST = 200`, `Z_TOOLTIP = 300`) rather than ad hoc per-instance numbers chosen at the point of use — since the overlay root is now a small, closed, well-understood set of competing contexts (unlike the rest of the app, where arbitrary future components might appear), a documented scale here is tractable to maintain and reason about, unlike a page-wide z-index free-for-all. Each overlay component references its designated constant rather than a locally invented number, which means adding a new overlay type in the future requires deliberately choosing where it fits in the existing documented scale, rather than someone guessing a number that happens to "feel high enough."

The trap: solving this by having engineers each pick increasingly large ad hoc numbers when a new ordering requirement comes up (which is exactly the "z-index war" anti-pattern the portal-based restructuring was meant to eliminate) — the value of consolidating overlays into one shared root is largely wasted if the z-index values *within* that root aren't also deliberately managed via a small, documented, closed scale.

---

**Q (Medium): Chrome's Layers panel shows the dropdown's stacking context and the card's stacking context as two separate, non-nested boxes rather than one nested inside the other. Does that change the diagnosis or the fix?**

Answer: It refines rather than changes it — "non-nested, both direct or indirect children of a shared ancestor context" versus "one nested inside the other" both reduce to the same underlying principle (each element's applicable z-index is compared only against siblings within its *own* stacking context, and whole contexts are compared against each other only within *their* shared parent context), but the specific ancestor where the comparison actually happens differs. If they're sibling contexts under a shared parent, the fix needs to happen at that shared parent's level — reordering the DOM position of the two context-creating ancestors, or adjusting the z-index of the context-creating ancestors themselves (not the leaf dropdown/card elements) — rather than assuming one is nested inside the other's context and looking for a single "trapping" ancestor to fix. Confirming the actual shape (nested vs. sibling contexts) via the Layers panel before proposing a fix avoids applying the wrong category of fix — e.g., trying to "escape" a context that the dropdown was never actually nested inside in the first place.

The trap: assuming all z-index bugs follow the "descendant trapped inside an ancestor's unexpectedly-created context" shape (as in this scenario) without verifying via DevTools whether that's actually what's happening — sibling-context-ordering bugs exist too and require a structurally different fix (reordering/adjusting the competing ancestors directly, not extracting a descendant from one of them).

---

**Q (Low): Does `z-index` do anything at all on an element with `position: static` (the default)?**

Answer: No — `z-index` only has an effect on elements that are "positioned" (`position` set to `relative`, `absolute`, `fixed`, or `sticky`) or that are flex/grid items (where z-index applies even without an explicit `position` value, per the flexbox/grid specs specifically carving out that exception). On a `position: static` element outside a flex/grid context, `z-index` is simply ignored — the element remains in normal document flow with no explicit stacking order of its own, participating in painting purely by DOM order relative to other normal-flow content. This is a common source of "I set z-index and nothing happened" confusion distinct from the stacking-context-trapping bug this scenario is about — sometimes the fix genuinely is just adding `position: relative` (with no offset values needed) to make the z-index declaration take effect at all, before any stacking-context reasoning becomes relevant.

The trap: immediately assuming a stacking-context-trapping bug (this scenario's actual cause) when the real issue for a *different* bug report could be the much more basic "z-index doesn't apply to static elements" — worth ruling out first, cheaply, before escalating to a full stacking-context investigation with DevTools.

---

## Self-Assessment

- [ ] Can explain why z-index only compares elements within the same stacking context, not globally across the page
- [ ] Can name at least six properties/values (beyond position+z-index) that create a new stacking context
- [ ] Can use (or describe using) DevTools' Layers panel to identify actual stacking contexts rather than guessing from reading CSS
- [ ] Can explain why portaling overlay components is a structural fix, not just a workaround, and why it prevents recurrence
- [ ] Can explain the DOM-order tiebreaker for equal/auto z-index siblings
- [ ] Can distinguish "z-index ignored because position is static" from "z-index correct but trapped in a sibling stacking context" as two different root causes with the same surface symptom

---
*Next: RTL Layout Breaking — moves from a rendering/depth axis (stacking) to a directional one, where physical CSS properties (`left`, `margin-right`) that seemed perfectly fine in LTR quietly assume a direction that isn't universal.*
