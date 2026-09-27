# Janky Scroll/Animation — Find and Fix

## Quick Reference

| Jank Source | How You Spot It | Fix |
|---|---|---|
| Animating layout-triggering properties (`top`/`left`/`width`/`height`) | Performance panel shows repeated Layout (purple) + Paint (green) on every frame | Animate `transform`/`opacity` instead — compositor-only, no layout/paint |
| Scroll handler doing synchronous work every event | `scroll` fires dozens of times/sec, handler does heavy sync work each time | Throttle/rAF-batch the handler, or avoid needing one at all (CSS `position: sticky`, `IntersectionObserver`) |
| Forced synchronous layout ("layout thrashing") | Reading a layout property (`offsetTop`, `getBoundingClientRect`) right after writing a style, in a loop | Batch all reads before all writes; measure once, mutate once |
| Non-composited (or newly-promoted) layer causing full repaints | Frame times spike specifically when the animated element overlaps other content | `will-change`/a dedicated compositor layer, kept only as long as actually animating |
| Main thread blocked by unrelated JS during the animation | Frame drops correlate with a Long Task in the trace, not with the animation code itself | Break up the long task, move it off the main thread (Web Worker), or defer it until the animation finishes |

## The Scenario

"Users are complaining that scrolling through our product list feels janky — it stutters, especially on mid-range Android phones. Profile it, find the actual cause, and fix it. Don't just tell me to 'use `transform`' — show me how you'd confirm that's actually the problem here first."

## Clarifying Questions

- **Is the jank during passive scrolling itself, or during a specific animation that happens to run while/after scrolling (e.g., items animating into view, a sticky header transitioning, images fading in)?** "Scrolling is janky" can mean the browser's own scroll compositing is struggling (rare, usually a sign of heavy paint on scroll) or — far more commonly — that something *triggered by* scroll (a scroll listener, an intersection-triggered animation, layout shifting as images load) is doing expensive work on the main thread that competes with the frames scrolling needs.
- **What's actually attached to scroll — a raw `scroll` event listener doing work on every fire, `IntersectionObserver`-triggered class toggles, a library doing scroll-linked animation (parallax, reveal-on-scroll)?** Each has a different profile: a naive scroll listener fires far more often than needed and easily does synchronous layout reads inside it; observer-based approaches are usually cheaper but can still trigger expensive work per intersection; scroll-linked (parallax-style) animations that read scroll position and write styles every frame are the most jank-prone by design, since they're synchronously coupled to scroll position rather than running independently on the compositor.
- **Which specific CSS properties are being animated or changed as part of whatever's happening during scroll?** This single question usually predicts most of the answer — properties that affect layout (`width`, `height`, `top`, `margin`) or paint (`background-color`, `box-shadow`, non-composited `opacity` edge cases) force the browser to redo layout/paint on every frame, whereas `transform` and `opacity` (when eligible for compositing) can animate purely on the compositor thread, off the main thread entirely.
- **Is this reproducible in a fast desktop Chrome, or does it require actually throttling CPU to a mid-tier mobile profile to see it?** Mid-range Android devices can have 4–6x less CPU headroom than a dev machine — jank that's invisible on desktop but clear on throttled CPU points strongly at main-thread contention (JS execution eating into the ~16.6ms/frame budget) rather than something structurally guaranteed to jank on all devices.
- **Does the jank correlate with anything else happening on the page at the same time** — images lazy-loading and causing layout shift, a background poll/websocket message handler firing, React re-rendering a large list on every scroll tick (e.g., an unthrottled scroll-position state update causing a full list re-render)? Jank is often not "the animation is expensive" in isolation but "the animation is competing with unrelated main-thread work for the same frame budget."

## Approach & Trade-offs

**Profile first, using the Performance panel's frame-by-frame breakdown — jank is a main-thread-budget problem, and the trace shows exactly what's eating that budget.** Every frame has roughly 16.6ms (60fps) to complete Style → Layout → Paint → Composite; anything that keeps the main thread busy longer than that drops a frame, and the browser visibly stutters. Rather than guessing at what's slow, I'd record a Performance trace while scrolling (CPU throttled to 4x–6x to match the reported devices), then look at the frame timeline: red-flagged Long Tasks, repeated purple (Layout) or green (Paint) bars stacked one after another during scroll, and the specific call stack attributed to each — the trace literally names the function and line responsible, removing the guesswork.

**The compositor-vs-main-thread distinction is the central mental model for the fix, not just a rule to recite.** The browser can animate `transform` and `opacity` entirely on the compositor thread if the element is already on its own layer — meaning the animation runs smoothly even while the main thread is busy with something else, because it never needs Style/Layout/Paint recalculation on the main thread per frame, only a compositor-thread transform of an already-rasterized layer. Any property that affects geometry (`width`, `top`, `margin`) or requires repainting (most non-`opacity` visual properties) forces the main thread back into Layout and/or Paint on every single animation frame — at 60fps that's a layout/paint pass every ~16ms, which is where jank on anything but the simplest page comes from.

**Distinguishing "layout thrashing" (a specific anti-pattern) from ordinary layout cost is worth calling out explicitly, since it's a common and very fixable mistake.** Reading a layout-dependent property (`offsetHeight`, `getBoundingClientRect()`, `scrollTop`) immediately after writing a style in the same tick forces the browser to synchronously flush pending layout to answer that read accurately — doing this in a loop over multiple elements (read, write, read, write...) causes a full synchronous layout recalculation on every iteration instead of once, which is dramatically more expensive than the same work batched (all reads, then all writes). This is one of the highest-leverage, easiest-to-miss-in-code-review sources of scroll/animation jank.

**Fixing the symptom (throttle the scroll handler) without fixing what the handler does is a half-measure I'd call out, not silently accept.** Throttling reduces *how often* expensive work runs, which helps, but if the work itself does forced synchronous layout or unnecessary heavy computation, throttling just makes the jank less frequent rather than eliminating it — I'd want to fix what the handler actually does (batch DOM reads/writes, move to `requestAnimationFrame`, or eliminate the need for a scroll listener at all via `IntersectionObserver`/CSS) as the primary fix, with throttling as a secondary, complementary measure for handlers that must run per-scroll-position.

## Solution — the diagnostic + fix walkthrough

**Step 1 — record a throttled trace while reproducing the jank.** DevTools → Performance → CPU throttle "4x slowdown" (approximating mid-range Android) → record → scroll through the list → stop. The frame chart at the top shows red bars over dropped frames; clicking one shows the breakdown for that frame.

**Step 2 — read the breakdown for a dropped frame.** Example trace finding:

```
Dropped frame breakdown:
- Scripting:  9.8ms   (a scroll handler running on every scroll event)
- Layout:     4.1ms   (forced by reading offsetTop after a style write)
- Paint:      3.2ms
- Composite:  0.4ms
Total: 17.5ms (over the 16.6ms budget at 60fps → dropped frame)
```

The call stack under "Scripting" points directly at the offending function.

**Step 3 — find the actual code.** Suppose it's a reveal-on-scroll effect:

```ts
// BUG: fires on every scroll event (can be 20-60+ times/sec),
// and reads a layout property right after a previous iteration's write — layout thrashing
function onScroll() {
  const items = document.querySelectorAll('.product-card');
  items.forEach(item => {
    const rect = item.getBoundingClientRect(); // READ — forces layout flush if a prior write is pending
    if (rect.top < window.innerHeight) {
      item.style.opacity = '1';
      item.style.top = '0px'; // WRITE — animating a layout property
    }
  });
}
window.addEventListener('scroll', onScroll);
```

Two separate problems here: `top` is a layout-triggering property being animated (main-thread Layout+Paint every frame instead of compositor-only), and the read (`getBoundingClientRect`) interleaved with prior writes across iterations causes forced synchronous layout on each one.

**Step 4 — fix both, and remove the need for a raw scroll listener entirely using `IntersectionObserver`:**

```ts
// Batch geometry detection off the scroll event entirely — the browser tells us
// when an element enters the viewport, no per-scroll-tick polling needed.
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        entry.target.classList.add('revealed'); // toggles a class, not inline styles
        observer.unobserve(entry.target); // one-shot reveal; stop observing once shown
      }
    });
  },
  { rootMargin: '0px 0px -10% 0px' }
);
document.querySelectorAll('.product-card').forEach(item => observer.observe(item));
```

```css
.product-card {
  opacity: 0;
  transform: translateY(20px); /* compositor-friendly: no layout impact */
  transition: opacity 0.3s ease-out, transform 0.3s ease-out;
}
.product-card.revealed {
  opacity: 1;
  transform: translateY(0);
}
```

This removes the scroll listener entirely (the browser's own compositor-thread intersection tracking replaces polling on `scroll`), and the animation itself now only touches `opacity`/`transform` — both compositable, running on the compositor thread without per-frame main-thread Layout/Paint.

**Step 5 — re-trace under the same throttled profile and confirm the frame chart is clean (no red dropped-frame bars) during scroll**, and specifically confirm the "Layout" and "Scripting" phases during scroll are near-zero, not just that it "feels smoother" subjectively.

> **Check yourself:** If you had to keep a *raw* `scroll` listener for some reason (say, updating a scroll-progress indicator that needs continuous, not one-shot, values), how would you structure it to avoid both layout thrashing and excessive handler-fire frequency?

## Gotchas

**Assuming `opacity`/`transform` are always compositor-only, without checking for exceptions that force a repaint anyway.** An element with `opacity` animating but that also has effects requiring repaint on that layer (certain filter combinations, an element not actually promoted to its own layer due to overlap with other content, or `will-change` not applied where the browser's heuristics don't auto-promote it) can still end up hitting Paint per frame — the Performance panel's Layers/Paint flashing overlay is what actually confirms compositor-only animation, not just "I used `transform`."

**Throttling a scroll handler but leaving forced synchronous layout inside it.** Reduces frequency, not per-call cost — if each throttled call still interleaves reads and writes across a loop, each individual call is still expensive; profiling after "fixing" with only a throttle can still show dropped frames, just fewer of them.

**Applying `will-change` broadly or leaving it on permanently.** `will-change` tells the browser to keep a dedicated compositor layer ready in advance — useful right before an animation starts, but applied to many elements (or left on indefinitely instead of toggled off after the animation completes) consumes GPU memory for layers that mostly sit idle, which can itself cause jank on memory-constrained mobile devices — the opposite of the intended effect.

**Fixing the animation but missing that an unrelated Long Task (a large JS bundle's synchronous initialization, an unthrottled state update on scroll causing a big React re-render) is what's actually eating the frame budget.** The trace's call stack under "Scripting" during a dropped frame names the actual function — if that function has nothing to do with the animation code being reviewed, the real fix is elsewhere (e.g., a `useEffect` triggering `setState` on every raw scroll event, cascading into an expensive re-render of a large list).

**Testing exclusively on a fast desktop and concluding the fix works**, when the reported jank is specifically on mid-range Android — CPU throttling in DevTools (or, ideally, an actual mid-tier device) is necessary to reproduce and confirm the fix under the conditions where the bug is reported.

## Follow-up Questions

**Q (High): Explain precisely why animating `transform`/`opacity` can avoid the main thread entirely, while animating `top`/`left`/`width` cannot.**

Answer: The rendering pipeline is Style → Layout → Paint → Composite. `top`/`left`/`width`/`height` changes affect the geometry of the element (and potentially its siblings/ancestors, depending on layout mode), so the browser must recompute Layout to know the new positions/sizes of affected elements, then Paint the new pixels, before compositing — all three of those steps run on the main thread (Layout and Paint specifically; compositing itself runs on the compositor thread but depends on their output being ready). `transform` and `opacity`, when the element is already promoted to its own compositor layer, don't change layout geometry or require new pixels to be painted at all — a `transform: translateX(...)` is applied by the compositor thread to an already-rasterized layer's bitmap, purely as a GPU-accelerated matrix operation, and `opacity` is a blend applied at composite time. This means the animation can continue smoothly on the compositor thread even while the main thread is busy with unrelated JS — the compositor thread runs independently and isn't blocked by main-thread work the way Layout/Paint are.

The trap: saying "`transform` is faster" without being able to explain *why* in terms of which pipeline stages it skips — "faster" undersells that it's a categorically different execution path (compositor thread, independent of main-thread contention), not just a lower-cost version of the same path.

---

**Q (High): What is "layout thrashing," precisely, and write a code example that causes it, then fix it.**

Answer: Layout thrashing is repeatedly interleaving DOM reads (of layout-dependent properties like `offsetTop`, `offsetHeight`, `getBoundingClientRect()`, `scrollTop`) with DOM writes (style/attribute changes) within the same tick, across multiple elements — because a read of a layout-dependent property must reflect the current, accurate state, the browser is forced to synchronously flush any pending layout-invalidating writes before answering that read, rather than batching all pending layout work into one recalculation at the end of the tick as it normally would. Doing this once is a single forced synchronous layout; doing it in a loop over N elements (read, write, read, write, ...) causes N forced synchronous layouts instead of one.

```js
// Thrashes: read (forces flush of prior write), write, repeat per element
items.forEach(item => {
  const height = item.offsetHeight; // read
  item.style.height = height * 2 + 'px'; // write — invalidates layout for next read
});

// Fixed: batch all reads first, then all writes — one layout recalculation total
const heights = items.map(item => item.offsetHeight); // all reads
items.forEach((item, i) => { item.style.height = heights[i] * 2 + 'px'; }); // all writes
```

The trap: recognizing layout thrashing as "reading and writing the DOM a lot" without the precise mechanism (a read of a layout-dependent property forcing a synchronous flush of pending writes) — the fix (batch reads, then batch writes) follows directly from the mechanism and doesn't work by coincidence.

---

**Q (High): How would you use the Performance panel's frame timeline to determine whether a dropped frame is caused by your animation code specifically, versus unrelated main-thread work?**

Answer: Click into a dropped/red-flagged frame in the frame chart to see its stage breakdown (Scripting / Layout / Paint / Composite time) and, critically, expand the "Scripting" portion's call stack — this shows the actual function names and source locations that consumed that time within the frame. If the call stack traces back to the animation/scroll-handling code being investigated, it's confirmed as the cause; if it instead shows something unrelated (a `setInterval` callback, a websocket message handler, a large React commit from an unrelated state update), the real fix is there, not in the animation code, even though the animation is where the jank is visually observed. I'd also check whether the unrelated work happens to coincide with scroll (e.g., a poll firing every 5s that happens to land during a scroll gesture in this recording) versus being caused by scroll (e.g., a scroll-driven state update triggering that re-render) — re-recording without scrolling, to see if the same Long Task appears independently, disambiguates correlation from causation.

The trap: assuming the animation code is guilty simply because the jank is visible during the animation — the trace's call stack is the actual evidence; without it, "the animation is janky" and "something else is janky at the same time as the animation" are indistinguishable from the outside.

---

**Q (Medium): What does `will-change: transform` actually do under the hood, and why shouldn't it be applied permanently to many elements?**

Answer: It's a hint to the browser to promote the element to its own compositor layer *in advance* of the animation starting, rather than promoting it reactively at the moment the animation begins — this avoids a brief hitch/repaint that can occur at animation-start if layer promotion happens just-in-time. Each promoted layer consumes GPU memory (for its own backing store/texture) and adds overhead to compositing (more layers to composite together, more memory bandwidth) — applying it broadly, to elements not actually about to animate, or leaving it applied indefinitely after an animation completes, accumulates memory and compositing overhead for no benefit, which on memory/GPU-constrained mobile devices can itself degrade performance (or even cause the browser to hit layer limits and fall back to less efficient compositing). The correct usage pattern is to toggle it on shortly before the animation starts and remove it after the animation ends, treating it as a temporary hint, not a permanent style.

The trap: treating `will-change` as an unconditional performance boost to sprinkle liberally — it's a targeted, temporary hint with a real resource cost, not a free optimization.

---

**Q (Medium): A scroll-triggered parallax effect reads `window.scrollY` and updates an element's `transform` on every `scroll` event. Even though it's using `transform` (compositor-friendly), users still report jank on mobile. Why might that be, and how would you fix it?**

Answer: Even with a compositor-friendly property, the *handler itself* still runs on the main thread on every `scroll` event — `scroll` can fire far more frequently than the frame rate on some browsers/inputs (especially with momentum scrolling on mobile), meaning the main thread does JS work (reading `scrollY`, computing the new transform, writing the style) more often than once per frame, and if that work is non-trivial or coincides with other main-thread activity, it can still miss frames despite the animated property itself being cheap to composite. The fix is to decouple the *read/compute* from the event frequency by batching the work into `requestAnimationFrame` — read `scrollY` and schedule (at most once per animation frame) a single update, rather than updating synchronously inside the raw `scroll` handler:

```js
let ticking = false;
window.addEventListener('scroll', () => {
  if (!ticking) {
    requestAnimationFrame(() => {
      const y = window.scrollY;
      el.style.transform = `translateY(${y * 0.5}px)`;
      ticking = false;
    });
    ticking = true;
  }
});
```

This ensures at most one style write per animation frame, regardless of how many `scroll` events fired in between, aligning the main-thread work with the browser's actual paint cadence instead of the event's potentially-higher firing rate.

The trap: assuming "I used `transform`" is sufficient by itself — the property choice only helps the *paint/composite* side of the pipeline; the handler's own execution frequency and cost on the main thread is a separate axis of the same problem, and needs its own fix (rAF-batching).

---

**Q (Low): Would moving expensive per-scroll computation into a Web Worker help with this kind of jank? What can and can't move there?**

Answer: It can help for computation that doesn't need direct, synchronous DOM access — e.g., if a scroll handler does heavy non-DOM computation (complex filtering/sorting of a large dataset based on scroll position, image processing) that computation could run in a worker, off the main thread, with the result posted back and applied via a cheap main-thread style/DOM update. It doesn't help for the DOM reads/writes themselves (`getBoundingClientRect`, style mutations) — workers have no DOM access at all, so anything requiring layout geometry or direct element manipulation must still happen on the main thread; a worker only offloads the *computation* feeding into that final, still-main-thread DOM update. For most scroll/animation jank specifically (which is usually dominated by Layout/Paint from the DOM work itself, not by heavy non-DOM computation), a worker addresses a different bottleneck than the common case — worth considering only after confirming, via the trace, that the expensive "Scripting" time is genuinely computational rather than DOM-bound.

The trap: reaching for Web Workers as a generic "move it off the main thread" answer without checking whether the actual bottleneck (per the trace) is DOM-bound work that a worker structurally cannot perform.

---

## Self-Assessment

- [ ] Can explain, in terms of the Style/Layout/Paint/Composite pipeline, exactly why `transform`/`opacity` can skip the main thread and `top`/`width` cannot
- [ ] Can define layout thrashing precisely and write both a thrashing example and its batched fix
- [ ] Can read a Performance panel dropped-frame breakdown and identify whether the cause is the animation itself or unrelated main-thread work
- [ ] Can explain why `requestAnimationFrame`-batching a scroll handler matters even when the animated property is already compositor-friendly
- [ ] Can state the resource cost of `will-change` and why it should be toggled, not left on permanently

---
*Next: Bundle Size Regression After a Release — moves from runtime jank to load-time cost, diagnosing what actually grew in the JS bundle and why, using the same "profile before you guess" discipline.*
