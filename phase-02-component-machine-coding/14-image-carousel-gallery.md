# Image Carousel / Gallery

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Slide movement | `transform: translateX(-100% * index)` on a flex track | Runs on the compositor thread, doesn't trigger layout/paint the way animating `left`/`margin-left` does — the difference between smooth and janky on low-end devices |
| Infinite loop | Clone first/last slide at each end, snap back instantly (no transition) once the clone's transition ends | Gives a seamless "endless" feel without genuinely infinite DOM nodes or index math users can perceive |
| Autoplay | `setInterval` (or rAF-driven timer), paused on hover/focus and on `visibilitychange` | An autoplaying carousel that keeps advancing in a backgrounded tab wastes cycles and jumps disorientingly when the user returns |
| Swipe gesture | Track `touchstart`/`touchmove`/`touchend`, compare distance + velocity against a threshold to commit or snap back | A raw distance threshold alone misclassifies a fast short flick as "not enough to advance" |
| Accessibility | Arrow-key nav, visible pause control, `aria-live="polite"` region announcing "Slide N of M" | Auto-rotating content with no pause control fails WCAG 2.2.2; a purely visual dot indicator conveys nothing to screen reader users |

## The Scenario

"Build an image carousel — a row of slides, one visible at a time, with next/prev arrows and dot indicators. I want it to auto-advance every few seconds, support swiping on mobile, and loop infinitely in both directions. Plain JS and CSS, no library. Let's start simple and layer features in as we go."

## Clarifying Questions

- **Does "loop infinitely" mean a true seamless wraparound (slide 5 → slide 1 feels continuous, no visible snap), or is an instant jump-cut back to slide 1 acceptable?** This changes the implementation meaningfully — modulo index arithmetic alone gives a correct but visually jarring jump at the seam (dragging past the last slide suddenly reveals slide 1 with no motion), while a seamless loop needs the clone-slide trick. I'd confirm which is expected before choosing.
- **Should autoplay pause on hover, on keyboard focus inside the carousel, and when the tab is backgrounded — or only some of those?** Each is a distinct real requirement: hover-pause is a UX courtesy so users can read a caption; focus-pause is closer to an accessibility requirement (a screen reader or keyboard user interacting with content shouldn't have it yanked away mid-interaction); tab-visibility-pause is a performance/correctness concern (no point animating what isn't shown, and resuming shouldn't instantly jump multiple slides to "catch up"). I'd implement all three, but I'd ask which are actually required in case there's a design reason to omit one.
- **Do we need real touch/swipe support, or is this desktop-primary with touch as a stretch goal?** Swipe handling done properly (distance + velocity + directional lock so a vertical page-scroll gesture isn't hijacked) is a meaningfully larger scope than next/prev buttons alone; I'd want to size the interview time against it before committing to build it in full.
- **Are images already known/fixed size, or responsive and possibly loading asynchronously?** This affects whether layout shift is a concern and whether "preload the next image" needs to handle images that haven't finished loading yet — a naive preload can itself cause jank if it's not actually ahead of when it's needed.
- **Is there a caption or any per-slide content beyond the image, and does the live region need to announce that too?** Determines whether the `aria-live` announcement is just "Slide 2 of 5" or needs to include slide-specific text, which matters for how genuinely useful it is to a screen reader user versus being a token accessibility gesture.

## Approach & Trade-offs

**Movement: `transform: translateX()` vs. `left`/`margin`.** Animating `left` or `margin-left` forces the browser to recompute layout (and often paint) on every frame, because those properties affect document flow. `transform` (and `opacity`) can be handled entirely by the compositor on their own layer, skipping layout and paint — this is the standard "GPU-accelerated" advice, and for a carousel specifically (an animation that runs on essentially every visit, often on mobile), the layout-thrash version is very visibly worse on mid-range hardware. I chose `transform: translateX(-100% * currentIndex)` on a flex row containing all slides, with `transition: transform 300ms ease` doing the actual animation, and JS only responsible for changing which index is "current" and toggling whether a transition should run at all (needed for the loop-seam snap-back).

**Infinite loop: modulo-index-jump vs. clone-slide.** Pure modulo arithmetic (`index = (index + 1) % slideCount`) is simple and correct as *state*, but naively wiring it straight to `translateX` produces a visible jump-cut at the wraparound, because going from index `N-1` to index `0` is a huge instantaneous transform change if you animate it directly (it'd visually slide backward across every slide to get there, or you disable the transition and it just cuts). The **clone-slide trick** — duplicate the first slide and append it after the last, and duplicate the last slide and prepend it before the first — lets you always animate one step in the "logical" direction: sliding from the real last slide onto the *cloned* first slide looks identical to sliding onto the real first slide, and once that transition ends, you instantly (no transition) snap the track's transform back to the real first slide's position. The user never perceives the snap because it happens on a frame with no visible motion pending. I'd build modulo-only in an interview first (as the correct, simpler baseline) and layer the clone trick on top once that's solid — this is the natural incremental order and mirrors how you'd explain the trade-off out loud.

**Autoplay: `setInterval` vs. `requestAnimationFrame`.** A carousel's autoplay is fundamentally "do a discrete thing every N seconds," not a per-frame animation — `setInterval` is the right primitive for scheduling *when* to advance; the actual slide movement is then just a CSS transition, not something driven frame-by-frame from JS. `requestAnimationFrame` would only be justified if I were manually driving the translateX value every frame instead of letting CSS transitions handle it (e.g., for a physics-based, velocity-preserving swipe release) — I'd use rAF specifically for the swipe-release-momentum piece, not for the baseline autoplay timer.

**Swipe: distance-only threshold vs. distance+velocity.** A pure "did the user drag past 30% of the container width" threshold misclassifies a fast, short flick — which is a very natural, common gesture — as "not far enough," forcing an over-explicit accompanying deliberate long-drag instead. Tracking both total distance and elapsed time (to derive velocity) and committing to advance if *either* the distance threshold or a velocity threshold is crossed matches how native mobile carousels (and OS-level swipe views) actually feel.

## Solution

Markup and base CSS:

```html
<div class="carousel" tabindex="0" aria-roledescription="carousel" aria-label="Featured images">
  <div class="carousel-track">
    <img class="slide" src="1.jpg" alt="Description of slide 1" />
    <img class="slide" src="2.jpg" alt="Description of slide 2" />
    <img class="slide" src="3.jpg" alt="Description of slide 3" />
  </div>
  <button class="prev" aria-label="Previous slide">‹</button>
  <button class="next" aria-label="Next slide">›</button>
  <button class="play-pause" aria-label="Pause automatic slideshow">⏸</button>
  <div class="dots" role="tablist"></div>
  <div class="sr-status" aria-live="polite" class="visually-hidden"></div>
</div>
```

```css
.carousel { position: relative; overflow: hidden; }
.carousel-track { display: flex; transition: transform 300ms ease; }
.slide { flex: 0 0 100%; }
.carousel-track.no-transition { transition: none; }
```

Core index state and rendering, starting modulo-only (no infinite loop yet):

```javascript
function createCarousel(root, { autoplayMs = 4000 } = {}) {
  const track = root.querySelector('.carousel-track');
  const slides = Array.from(track.children);
  const count = slides.length;
  let index = 0;

  function render() {
    track.style.transform = `translateX(-${index * 100}%)`;
  }

  function goTo(i) {
    index = (i + count) % count; // modulo handles both directions of wraparound
    render();
    announce();
  }

  function next() { goTo(index + 1); }
  function prev() { goTo(index - 1); }
```

This alone gives correct *state* wraparound but jump-cuts at the seam if you were to naively animate a `2 → 0` transform directly — worth demonstrating and naming the seam problem before fixing it.

Layering the clone-slide trick for a seamless loop:

```javascript
  // Clone first/last slide so we always animate one visual step forward/back.
  const firstClone = slides[0].cloneNode(true);
  const lastClone = slides[count - 1].cloneNode(true);
  track.prepend(lastClone);
  track.append(firstClone);
  index = 1; // real slide 0 now sits at position 1, after the prepended clone

  function render() {
    track.style.transform = `translateX(-${index * 100}%)`;
  }

  function step(delta) {
    index += delta;
    render();
  }

  track.addEventListener('transitionend', () => {
    if (index === count + 1) {
      // landed on the appended clone of slide 0 — snap back, no transition
      track.classList.add('no-transition');
      index = 1;
      render();
      track.offsetHeight; // force reflow so the class removal starts a fresh transition next time
      track.classList.remove('no-transition');
    } else if (index === 0) {
      // landed on the prepended clone of the last slide — snap forward
      track.classList.add('no-transition');
      index = count;
      render();
      track.offsetHeight;
      track.classList.remove('no-transition');
    }
  });

  function next() { step(1); }
  function prev() { step(-1); }
```

Autoplay with hover/focus/visibility pausing:

```javascript
  let timer = null;
  function startAutoplay() {
    stopAutoplay();
    timer = setInterval(next, autoplayMs);
  }
  function stopAutoplay() {
    clearInterval(timer);
    timer = null;
  }

  root.addEventListener('mouseenter', stopAutoplay);
  root.addEventListener('mouseleave', startAutoplay);
  root.addEventListener('focusin', stopAutoplay);
  root.addEventListener('focusout', startAutoplay);

  document.addEventListener('visibilitychange', () => {
    if (document.hidden) stopAutoplay();
    else startAutoplay();
  });

  root.querySelector('.play-pause').addEventListener('click', (e) => {
    if (timer) { stopAutoplay(); e.target.setAttribute('aria-label', 'Play automatic slideshow'); }
    else { startAutoplay(); e.target.setAttribute('aria-label', 'Pause automatic slideshow'); }
  });
```

Swipe handling — distance and velocity, not distance alone:

```javascript
  let startX = 0, startTime = 0, deltaX = 0;

  track.addEventListener('touchstart', (e) => {
    stopAutoplay();
    startX = e.touches[0].clientX;
    startTime = Date.now();
    track.classList.add('no-transition'); // track finger 1:1 while dragging
  });

  track.addEventListener('touchmove', (e) => {
    deltaX = e.touches[0].clientX - startX;
    track.style.transform = `translateX(calc(-${index * 100}% + ${deltaX}px))`;
  });

  track.addEventListener('touchend', () => {
    const elapsed = Date.now() - startTime;
    const velocity = Math.abs(deltaX) / elapsed; // px/ms
    const width = track.clientWidth;
    const distanceRatio = Math.abs(deltaX) / width;

    track.classList.remove('no-transition');
    const commits = distanceRatio > 0.25 || velocity > 0.5;
    if (commits) {
      step(deltaX < 0 ? 1 : -1);
    } else {
      render(); // snap back to current index, no net movement
    }
    deltaX = 0;
    startAutoplay();
  });

  function announce() {
    root.querySelector('.sr-status').textContent = `Slide ${index} of ${count}`; // index adjusted for clone offset in real code
  }

  render();
  startAutoplay();
  return { next, prev, goTo, stopAutoplay, startAutoplay };
}
```

> **Check yourself:** Why does the clone-slide snap-back have to happen inside a `transitionend` listener rather than immediately after calling `step()` — and why is `track.offsetHeight` (a forced reflow) needed between removing and re-enabling the transition class?

## Preloading

Preloading the next/previous image prevents the transition from revealing a blank or partially-loaded image mid-animation. The cheapest approach: eagerly set `loading="eager"` (or no `loading` attribute at all) only on the current, next, and previous slides, and `loading="lazy"` on the rest — so the browser only fully commits network/decode priority to images actually near the viewport, while the ones about to be shown are already requested. For a more explicit guarantee, decode the upcoming image ahead of the transition:

```javascript
function preload(src) {
  const img = new Image();
  img.src = src; // browser caches it; a later <img> with the same src paints instantly
}
```

calling `preload` for `index + 1` and `index - 1` every time `goTo`/`step` runs, so by the time the user navigates *again*, the adjacent images are already warm in the HTTP/decode cache.

## Gotchas

**Animating `left`/`margin-left` instead of `transform`.** Works correctly but forces layout recalculation on every animation frame, which is measurably janky on low-end mobile devices — this is the single most common "technically works, wrong tool" answer in this scenario.

**Snapping back on the clone without disabling the transition first.** If the index reset to the real slide happens while the transition is still active, the browser animates *backward* across every slide to get there, producing a visible, wrong-direction slide instead of an imperceptible snap. The transition must be disabled (`no-transition` class), the transform changed, a reflow forced, and only then re-enabled.

**Autoplay that doesn't pause on focus, only on hover.** A keyboard user tabbing into the carousel to interact with a slide's content (a link, a caption) can have the slide yanked out from under them mid-interaction if only `mouseenter`/`mouseleave` are wired — `focusin`/`focusout` need the same handling.

**No pause control on auto-rotating content.** WCAG 2.2.2 (Pause, Stop, Hide) requires that any auto-updating content that starts automatically and lasts more than 5 seconds have a way for the user to pause it — an autoplaying carousel with no pause button is a real, citable accessibility failure, not a nice-to-have.

**A dot indicator with no text alternative.** Dots convey "which slide" only visually; without an `aria-live` announcement or equivalent, a screen reader user navigating the carousel gets no signal that content changed at all, let alone which slide they're on now.

**Touch handling that doesn't distinguish horizontal swipe from vertical page scroll.** If `touchmove` always calls `preventDefault()`, vertical scrolling of the page becomes impossible while a finger is on the carousel — the swipe handler needs to check whether the gesture is more horizontal than vertical before committing to intercept it, usually by comparing the very first `touchmove`'s delta X vs delta Y.

## Follow-up Questions

**Q (High): Why is `transform: translateX()` preferred over animating `left` or `margin-left`, in terms of the actual rendering pipeline?**

Answer: The browser rendering pipeline is roughly style → layout → paint → composite. Properties like `left` and `margin` affect the box model, so changing them on every animation frame forces layout recalculation (and usually repaint) each frame — expensive, and the more elements/complexity on the page, the worse it scales. `transform` and `opacity` can be handled by the compositor alone, on their own layer, skipping layout and paint entirely for elements the browser has already promoted to a layer — the CPU/main-thread work per frame is dramatically lower, which is the difference between 60fps and visible jank, especially on lower-end mobile hardware where a carousel is very commonly viewed.

The trap: saying "transform is GPU-accelerated" as the entire answer without being able to name *which pipeline stages it skips* — interviewers probing this follow-up are checking for actual understanding of the rendering pipeline, not a memorized buzzword.

---

**Q (High): Walk through exactly how the clone-slide infinite loop trick avoids a visible jump at the seam.**

Answer: A real slide array of length N gets a clone of slide 0 appended after slide N-1, and a clone of slide N-1 prepended before slide 0 — so the track is now N+2 slides wide, with the "current index" state offset by 1 to account for the prepended clone. Every `next()`/`prev()` call animates exactly one slide-width in the natural direction, including the step from the real last slide onto the *cloned* first slide (which looks pixel-identical to the real first slide) — so the transition itself is always a normal, single-step animation, never a multi-slide jump. Once that transition's `transitionend` fires and the user is now looking at the clone, the code instantly (with the CSS transition disabled) resets the track's transform to point at the *real* first slide's position, which is visually identical to where the clone was sitting — so the reset is imperceptible. The critical ordering detail: transition removal, transform change, and transition re-enable must happen with an actual paint/reflow in between (`element.offsetHeight` forces this), otherwise the browser can batch the class changes and animate the "reset" too.

The trap: describing only "clone the slides" without explaining the disable-transition-then-forced-reflow-then-re-enable sequence — that sequencing is the actual mechanism that makes the snap invisible, and skipping it is the most common reason a candidate's live-coded version of this visibly stutters.

---

**Q (High): What's the WCAG requirement around auto-rotating carousels, and how does this implementation satisfy it?**

Answer: WCAG 2.2.2 (Pause, Stop, Hide, Level A) requires that for any moving, blinking, scrolling, or auto-updating content that starts automatically, lasts more than 5 seconds, and is presented in parallel with other content, the user must be able to pause, stop, or hide it. An autoplaying carousel with no pause control fails this outright. This implementation satisfies it via an explicit, always-visible pause/play button, plus (as good practice beyond the letter of the requirement) pausing automatically on hover and keyboard focus, so a user reading a slide's content doesn't have to hunt for the pause button before it changes under them.

The trap: treating "pauses on hover" as sufficient on its own — that helps mouse users but does nothing for a touch-only or keyboard-only user, who need an explicit, persistent control; hover-pause is a nice addition, not a substitute for it.

---

**Q (Medium): How would you announce slide changes to screen reader users, and why does a purely visual dot indicator fail them?**

Answer: A visually-hidden `aria-live="polite"` region (or `aria-live="off"` region updated alongside an `aria-atomic` announcement, depending on desired verbosity) gets its text content updated to something like "Slide 2 of 5" (or including the slide's caption/alt text, if meaningful) every time the active slide changes; screen readers announce `polite` live-region updates without interrupting whatever the user is currently doing. A dot indicator conveys "which slide is active" purely through visual state (a filled vs. hollow dot, or a color/size change) with no accompanying text unless each dot itself is a properly labeled, focusable control (`aria-label="Go to slide 3"`, `aria-current="true"` on the active one) — without either of those, a screen reader user gets zero signal that the carousel's content changed at all when it auto-advances.

The trap: adding `aria-live` but setting it to `assertive`, which interrupts whatever the screen reader is currently announcing — for an auto-advancing carousel that's disruptive rather than helpful; `polite` is almost always the right choice here since the content change isn't urgent.

---

**Q (Medium): The swipe handler commits to advancing if distance ratio > 25% OR velocity > some threshold. Why "or" rather than "and," and what would each alternative get wrong?**

Answer: Using "and" would require both a long drag *and* a fast flick to register, which excludes the very common fast-short-flick gesture (a quick, small swipe that users expect to register as "next," the way native OS card/story interfaces respond to flicks) — under "and," that gesture would always snap back, feeling unresponsive. Using "or" correctly captures both natural gesture styles: a slow, deliberate long drag past the threshold (even at low velocity) and a fast short flick that doesn't cover much distance but clearly signals intent through speed. Requiring only "or" one condition, rather than distance alone, is what prevents a fast tiny accidental touch-drag (like a scroll-adjacent thumb movement) from misfiring — the velocity threshold on its own filters out slow accidental drags below the distance cutoff, and the distance threshold filters out fast but tiny incidental movements below some minimum distance floor (worth adding a small minimum-distance floor alongside the velocity check in a production version, to avoid a very fast 2px jitter registering as a swipe).

The trap: implementing distance-only and calling it done — it technically "handles swipe" but feels noticeably unresponsive compared to any native swipeable UI, which interviewers who've actually built mobile-facing carousels will notice immediately.

---

**Q (Medium): How would you preload images so the transition never reveals a blank or half-loaded slide, without preloading every image up front and wasting bandwidth?**

Answer: Preload only a small, moving window around the current index — typically current, next, and previous — by either constructing an off-DOM `new Image()` with the target `src` (which triggers the browser's fetch/cache without displaying anything) ahead of the user navigating there, or by setting `loading="eager"`/no lazy-loading attribute only on those three slides' actual `<img>` elements while the rest stay `loading="lazy"`. This re-runs every time the index changes, so the "warm window" always stays centered on wherever the user currently is, rather than front-loading the entire gallery's bandwidth cost on initial page load, which matters a lot for a gallery with dozens of high-resolution images.

The trap: preloading literally everything up front "to be safe" — this defeats the purpose of lazy loading and can meaningfully hurt initial page load performance for a large gallery, trading a rare visible flash for a guaranteed slow initial load; the windowed approach gets nearly all the benefit at a fraction of the cost.

---

**Q (Low): How would `prefers-reduced-motion` change this component's behavior?**

Answer: Under `@media (prefers-reduced-motion: reduce)`, the transition duration should be reduced substantially or removed (e.g., switch to a cross-fade or an instant cut instead of a sliding transform), and — more importantly for this specific component — autoplay arguably shouldn't start automatically at all, or should start already paused, since a user who's expressed a system-level preference against motion is signaling exactly the kind of unsolicited-movement experience an auto-rotating carousel represents. This is checked via `window.matchMedia('(prefers-reduced-motion: reduce)').matches` in JS (to gate autoplay) and a corresponding media query in CSS (to shorten/remove the transition).

The trap: only handling `prefers-reduced-motion` in CSS (softening the transition) while still auto-starting the slideshow — the motion-sensitivity concern that setting exists for is as much about unsolicited automatic movement as it is about the specific animation curve.

---

**Q (Low): How would this design change for a gallery with a very large number of images (hundreds), where rendering every slide as a DOM node up front isn't reasonable?**

Answer: Rather than mounting every image as a permanent DOM node in the track, the component would render only a small window of DOM nodes around the current index (e.g., current ± 2) and re-point their `src` attributes as the user navigates, using the same index-driven `translateX` positioning logic but treating "which image is loaded into which of the few real DOM slots" as separate state from "which position is currently in view" — effectively a windowed/virtualized carousel. This adds real complexity (recycling nodes, keeping the illusion of continuous track positions while the underlying nodes get their `src` swapped out from under the user) and would only be worth building if the image count is genuinely large enough that DOM node count itself is the bottleneck, not just image bytes (which lazy-loading/preloading already handles at more modest scales).

The trap: assuming lazy-loading images alone solves this — lazy-loading defers *network fetch*, but doesn't reduce DOM node count; at large enough scale, the sheer number of `<img>` elements in the tree (and their associated layout/style computation cost) becomes its own problem independent of whether their bytes have been downloaded.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain why `transform` beats `left`/`margin` for this animation, citing the rendering pipeline stages skipped
- [ ] Can implement the clone-slide infinite loop trick, including the disable-transition/reflow/re-enable sequence
- [ ] Can wire autoplay pausing across hover, focus, and `visibilitychange`, and explain why all three are needed
- [ ] Can implement swipe detection using both distance and velocity, and explain why "or" is correct instead of "and"
- [ ] Can state the WCAG 2.2.2 requirement this component must satisfy and how the pause control satisfies it
- [ ] Can explain why a visual dot indicator alone is insufficient for screen reader users and what fixes it

---
*Next: Undo/Redo Stack for an Editor UI — from managing a moving viewport over static content to managing a history of content mutations over time.*
