# Design an Image/Video Gallery With Lazy Loading

## Quick Reference

| Decision Point | Choice | Why |
|---|---|---|
| Loading trigger | Native `loading="lazy"` for the common case; `IntersectionObserver` only when finer control is needed (custom root margin, video play/pause, priority tiers) | The browser's native lazy-loading already handles the common "don't fetch what's far off-screen" case correctly and cheaply; reach for `IntersectionObserver` only for behavior the native attribute can't express |
| Layout shift | Reserve the exact aspect-ratio box (via `width`/`height` attributes or CSS `aspect-ratio`) before the image/video loads, always | Without a reserved box, an image popping in at its natural size shoves surrounding content down — a direct, measurable CLS regression, not just a visual nicety |
| Responsive delivery | `srcset`/`sizes` (or a CDN's on-the-fly resizing) serving a device-appropriate resolution, never one fixed "master" image to every viewport/DPR | Shipping a desktop-resolution image to a phone wastes bandwidth proportional to the resolution difference, often 4–10x more data than needed |
| Above-the-fold images | Eagerly loaded (`loading="eager"`, `fetchpriority="high"`), explicitly excluded from lazy-loading | The very images most likely to affect LCP (Largest Contentful Paint) are exactly the ones lazy-loading would otherwise needlessly delay |
| Video in a scrolling grid | Poster image only until in view; playback (if autoplaying muted previews) starts on entering the viewport and pauses/unloads on leaving | Loading and buffering every video in a long gallery unconditionally wastes bandwidth and battery for content the user may never scroll to |

## The Scenario

"Design a media gallery — a scrolling grid of images and short video previews, potentially thousands of items, with a full-screen viewer when you tap into one. It needs to load fast, not jank the layout as content pops in, and not waste bandwidth loading media the user never actually scrolls to. Walk me through the loading strategy and rendering architecture."

## Clarifying Questions

- **Is this predominantly images, predominantly video, or a genuine mix — and if video, do previews autoplay (muted, inline) in the grid, or only play on explicit tap?** Autoplaying video previews in a grid is a meaningfully bigger bandwidth/battery/complexity commitment than static poster images with tap-to-play — worth confirming rather than assuming either default.
- **What's the expected gallery size per session — dozens of items, or genuinely thousands requiring virtualization on top of lazy-loading?** Lazy-loading alone (not fetching off-screen media) doesn't bound DOM node count — a gallery large enough that even the *unloaded* placeholder elements number in the thousands still needs list virtualization (windowing) layered on top of lazy asset loading, the same way the News Feed scenario needed it for post cards.
- **Does the grid need to preserve a specific layout (uniform grid cells) or does it need to reflow to each item's natural aspect ratio (a masonry/Pinterest-style layout)?** Masonry layouts are meaningfully harder to virtualize correctly (row heights aren't uniform or even knowable ahead of a measurement pass) and interact with lazy-loading differently than a fixed-aspect-ratio grid, where every cell's box is known up front regardless of what's loaded into it.
- **Is there a full-screen "lightbox" viewer for a single item, and if so, does it need to support swiping/arrow-key navigation to adjacent items with those images preloaded ahead of time?** Affects whether a preloading strategy for "the next likely item" needs to exist alongside the grid's own lazy-loading, which is a distinct concern from the grid's scroll-based loading.
- **What device/network conditions need to be designed for** — should the gallery adapt its image quality/resolution based on a slow connection or a user's OS-level data-saving preference? This decides whether `navigator.connection`/the `Save-Data` request header should influence what quality tier gets requested, versus always requesting the same responsive breakpoint regardless of network conditions.
- **Is there a requirement for images to be indexable/crawlable (SEO for a public gallery), which would push toward images being present in initial server-rendered markup rather than purely inserted after client-side `IntersectionObserver` logic runs?** A purely JS-driven lazy-load-on-scroll implementation (with no `<img>` tags present until scrolled into view) can be invisible to some crawlers or slow to be discovered — worth checking if this matters for this particular gallery's content.

## Approach & Trade-offs

**Default to native `loading="lazy"` and reach for `IntersectionObserver` only for what it can't do.** The native attribute already implements "don't fetch this image's bytes until it's within some browser-determined distance of the viewport," which covers the core ask with zero custom JS, no observer-management bugs to write, and behavior tuned by the browser vendor. Custom `IntersectionObserver` logic earns its complexity when something beyond simple fetch-deferral is needed: driving video play/pause exactly at viewport entry/exit (native lazy-loading doesn't control playback), implementing a custom root margin tuned specifically for this gallery's scroll speed/card size, or coordinating priority tiers (loading the *next* off-screen row slightly ahead of literal viewport entry, to hide network latency, versus the native attribute's more conservative default threshold). Building a custom observer-based system when the native attribute already covers the actual requirement is unnecessary complexity; the reverse mistake — reaching only for the native attribute when video-playback-gating or fine-tuned prefetch distance is actually needed — leaves real requirements unaddressed.

**Reserving layout space ahead of load is non-negotiable and needs to be correct by construction, not an afterthought fixed after a CLS complaint.** Every grid cell's box (width and height, or an aspect-ratio-derived height from a known width) must be determined *before* any image bytes arrive — from `width`/`height` attributes on the `<img>` tag (which, combined with CSS sizing, lets the browser reserve the correct aspect-ratio box even before the image loads) or an explicit CSS `aspect-ratio` on the container. Without this, each image's natural dimensions are unknown until it loads, and its arrival resizes its containing box, shifting every subsequent element on the page — a classic CLS regression, and one that gets *worse*, not better, the more successfully lazy-loading defers off-screen images (each one still triggers a shift whenever it does finally load and reveal its real size, if no space was reserved for it up front).

**Above-the-fold content is deliberately excluded from lazy-loading, not lazily loaded "a little less lazily."** The images most likely to be the page's Largest Contentful Paint element are, by definition, the ones visible without scrolling — applying `loading="lazy"` (or an `IntersectionObserver` gate) to them adds a deferral mechanism to exactly the content that most needs to start loading immediately, directly working against LCP rather than being neutral to it. The correct pattern explicitly marks the first screenful (or first N items, a number tunable per breakpoint) as `loading="eager"` with `fetchpriority="high"`, and only applies the lazy-loading treatment to everything below that threshold — this requires the rendering logic to know, at render time, roughly which items are "above the fold" (typically the first N items in the initial render, before any scrolling has occurred), not a uniform lazy-loading rule applied indiscriminately to every image in the grid.

**Video previews need their own load/play/pause lifecycle gated by visibility, layered on top of — not replacing — the grid's general lazy-loading strategy.** A `poster` image (itself subject to the same aspect-ratio-reservation and lazy-loading treatment as a static image) is what's shown by default; the actual video source is only attached/loaded, and playback only started, once an `IntersectionObserver` confirms the video element has entered the viewport by some reasonable threshold — and, symmetrically, playback is paused and (for a very long gallery) the source can be fully detached again once it scrolls back out, freeing the decoder/network resources for videos likely to be revisited less often than ones currently in view. This is meaningfully more involved than image lazy-loading because it involves an active, stateful resource (playback) rather than a one-time fetch, and needs explicit enter/exit handling rather than a fire-once loading trigger.

**Network-aware quality selection is a genuine, if often-skipped, refinement — using `navigator.connection.effectiveType`/`saveData` (where supported) to request a lower-resolution tier proactively, not just relying on responsive `srcset` breakpoints alone.** `srcset`/`sizes` already handles "give me the right resolution for this viewport size and device pixel ratio," but doesn't account for a user explicitly on a constrained/metered connection (mobile data with "Data Saver" enabled) who would prefer a lower-fidelity image even at a viewport size that would otherwise warrant a higher-resolution source. Checking `navigator.connection?.saveData` (where the API is available; it isn't universally, so this is a progressive enhancement, not a load-bearing requirement) and adjusting the requested resolution tier downward is a real, if second-order, consideration worth naming — while being clear that `srcset` responsiveness is the primary, always-needed mechanism and network-awareness is an additional refinement on top.

## Solution

**Grid cell with reserved aspect ratio and native lazy-loading, above-the-fold override:**

```tsx
function GalleryImage({ item, index, priorityCount }: {
  item: MediaItem; index: number; priorityCount: number;
}) {
  const isAboveFold = index < priorityCount; // e.g., first 8 items for a typical initial viewport

  return (
    <div className="gallery-cell" style={{ aspectRatio: `${item.width} / ${item.height}` }}>
      <img
        src={buildSrc(item, 'medium')}
        srcSet={buildSrcSet(item)}
        sizes="(max-width: 600px) 50vw, 25vw"
        width={item.width}
        height={item.height}
        loading={isAboveFold ? 'eager' : 'lazy'}
        fetchPriority={isAboveFold ? 'high' : 'auto'}
        alt={item.altText}
      />
    </div>
  );
}
```

**Video preview gated by `IntersectionObserver`, layered on the same reserved-box treatment:**

```tsx
function GalleryVideo({ item }: { item: MediaItem }) {
  const videoRef = useRef<HTMLVideoElement>(null);
  const [shouldLoadSource, setShouldLoadSource] = useState(false);

  useEffect(() => {
    const el = videoRef.current!;
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setShouldLoadSource(true); // attach source lazily, once actually in view
          el.play().catch(() => {}); // autoplay can be rejected by the browser — non-fatal
        } else {
          el.pause();
        }
      },
      { threshold: 0.5 }
    );
    observer.observe(el);
    return () => observer.disconnect();
  }, []);

  return (
    <div className="gallery-cell" style={{ aspectRatio: `${item.width} / ${item.height}` }}>
      <video ref={videoRef} poster={item.posterUrl} muted loop playsInline>
        {shouldLoadSource && <source src={item.videoUrl} type="video/mp4" />}
      </video>
    </div>
  );
}
```

**Lightbox preloading — adjacent items fetched ahead of navigation, not on demand:**

```tsx
function useLightboxPreload(items: MediaItem[], currentIndex: number) {
  useEffect(() => {
    // Preload the next and previous full-resolution images so arrow-key/swipe
    // navigation feels instant rather than triggering a visible load-in.
    [currentIndex - 1, currentIndex + 1].forEach((i) => {
      const item = items[i];
      if (!item) return;
      const preloadImg = new Image();
      preloadImg.src = buildSrc(item, 'full');
    });
  }, [items, currentIndex]);
}
```

**Network-aware quality tier selection (progressive enhancement):**

```ts
function preferredQualityTier(): 'low' | 'medium' | 'high' {
  const conn = (navigator as any).connection;
  if (conn?.saveData || conn?.effectiveType === '2g') return 'low';
  if (conn?.effectiveType === '3g') return 'medium';
  return 'high'; // default when the API is unavailable or reports a fast connection
}
```

> **Check yourself:** Without looking above, explain why reserving an aspect-ratio box before an image loads is required to prevent layout shift even for images that are lazily loaded far below the fold, not only for above-the-fold content.

## Rendering Pipeline

```
Grid mounts → first N items marked above-fold (eager, high priority, reserved boxes)
            → remaining items rendered with reserved boxes + native lazy-loading
            → item enters viewport → image fetches at responsive/network-appropriate resolution
            → (video) IntersectionObserver confirms visibility → source attaches, playback starts
            → item exits viewport → (video) playback pauses, optionally source detaches
            → user taps an item → lightbox opens → adjacent items preloaded ahead of navigation
```

For a gallery large enough to need it, this entire pipeline sits on top of list virtualization (windowing the grid itself, so off-screen cells aren't even mounted, not merely un-fetched) — lazy-loading and virtualization solve adjacent but distinct problems (don't fetch bytes I don't need yet vs. don't keep thousands of DOM nodes mounted) and a gallery at real scale needs both simultaneously.

## Scaling Considerations

**Virtualize the grid itself once item count is large, independent of lazy-loading.** Lazy-loading only defers *fetching*; every cell (loaded or not) is still a mounted DOM node with layout/paint cost unless the grid is also windowed — a gallery with thousands of items needs both lazy asset loading (bandwidth) and virtualization (DOM/memory), addressing genuinely different resource constraints.

**A CDN with on-the-fly resizing (rather than pre-generating every breakpoint at upload time) keeps the `srcset` strategy maintainable as new device/DPR combinations emerge.** Requesting `image.cdn/photo123?width=400` and letting the CDN generate and cache that specific size on first request (rather than the app needing to pre-generate and store every possible width ahead of time) scales better as breakpoints or new device classes are added later.

**Placeholder strategy (blurred low-quality image placeholder, dominant-color box, or a skeleton shimmer) should be cheap to generate and ship, not itself a source of jank.** A tiny (a few hundred bytes) blurred placeholder inlined as a base64 `data:` URI or a solid dominant-color swatch computed at upload time are both common, cheap choices; generating a full LQIP (low-quality image placeholder) requires either a build-time/upload-time processing step or an additional tiny network request per image, which is itself worth budgeting for rather than treated as free.

**Video decoding/memory needs bounding even with visibility-gated playback** — a gallery with many videos should limit how many `<video>` elements are simultaneously "active" (source attached, decoding) even within the currently-visible viewport region, since mobile browsers impose real limits on simultaneous video decoders; detaching sources for videos scrolled fully out of view (not just pausing playback) is what actually frees this resource rather than merely stopping playback while still holding a buffered source.

## Gotchas

**Applying `loading="lazy"` uniformly, including to above-the-fold hero images.** This is a subtle, easy-to-introduce LCP regression — it looks like "consistent lazy-loading everywhere" is the disciplined choice, when in fact the images most critical to load immediately are exactly the ones this blanket rule delays.

**Not reserving aspect-ratio space, relying on lazy-loading alone to "handle" layout stability.** Lazy-loading and layout-shift prevention are unrelated concerns — deferring *when* an image loads does nothing to prevent the shift that happens the moment it finally does load, if no space was reserved for it ahead of time.

**Autoplaying every video in the grid regardless of viewport visibility.** Even muted, autoplaying videos far off-screen burns bandwidth and battery, and on many mobile browsers can silently fail or throttle anyway — visibility-gated play/pause isn't just a politeness optimization, it's often required for autoplay to work reliably at all under mobile browser autoplay policies.

**Fetching a single, uniform "master" resolution image for every viewport and device pixel ratio.** Serving a 2000px-wide image to a 375px-wide mobile viewport wastes several times the necessary bandwidth per image — multiplied across a gallery of hundreds of items, this is a very real, measurable cost, not a micro-optimization.

**Building a fully custom `IntersectionObserver`-based lazy-loading system when native `loading="lazy"` would have covered the actual requirement.** Reinventing fetch-deferral logic the browser already implements adds surface area for bugs (observer leaks, incorrect root margins, cleanup on unmount) for no behavioral gain over the native attribute, when the only actual custom requirement was something native lazy-loading doesn't cover (like video play/pause).

**No preloading strategy for the lightbox's adjacent items.** Without it, every arrow-key/swipe navigation in the full-screen viewer triggers a visible load-in for the newly-shown image, which reads as sluggish precisely at the moment (full-screen, single-image focus) where load latency is most noticeable to the user.

## Follow-up Questions

**Q (High): Why does reserving an aspect-ratio box matter even for images that are lazily loaded and won't be fetched until far into a scroll session — isn't the shift only relevant for what's currently visible?**

Answer: The shift happens the moment an image *does* load and reveals its real dimensions, regardless of how far into the session that occurs — if a lazily-loaded image below the fold has no reserved box, the instant it finishes loading (having scrolled into view, or even just barely before, depending on the lazy-load threshold), its arrival still resizes its container and shifts whatever content comes after it on the page, which is exactly the CLS-causing event this technique prevents. Lazy-loading changes *when* the fetch happens; it does nothing about *what happens to the layout* at the moment the fetch resolves — those are separate mechanisms addressing separate problems, and skipping aspect-ratio reservation because "it's lazy-loaded, so it doesn't matter yet" conflates the two.

The trap: assuming deferred loading implies deferred consequences for layout stability — the shift is tied to the moment of *load completion*, not to scroll position at the time of initial render, and a lazily-loaded image without a reserved box still causes a shift the instant it loads, whether that's on initial scroll-into-view or later.

---

**Q (High): Design the criteria for deciding how many items at the top of the grid should be `loading="eager"`/high-priority versus lazily loaded. Is a fixed number (e.g., "first 8") correct across all devices and layouts?**

Answer: A fixed absolute count is a reasonable approximation but not precisely correct across breakpoints, because "how many items fit above the fold" depends on viewport size and the grid's column count at that breakpoint — a 4-column desktop layout fits more items in the first visible row(s) than a 2-column mobile layout does. A more precise approach ties the eager/priority count to the actual column count at the current breakpoint (e.g., "the first two full rows," computed as `columnsAtCurrentBreakpoint * 2`) rather than a single hardcoded number meant to cover every layout — though a reasonably generous fixed number (erring slightly high) is a defensible, simpler approximation for many real designs, as long as it's chosen with the actual layout's typical above-the-fold item count in mind rather than picked arbitrarily.

The trap: hardcoding a single "first N" constant without connecting it to the actual responsive column count — this can under-prioritize on a wide desktop layout (more items are actually above the fold than the constant assumes, so some genuinely-visible-on-load images get incorrectly lazy-loaded) or over-prioritize on mobile (eagerly loading items that aren't actually visible yet on a narrower, single/double-column layout).

---

**Q (High): How would you decide, for a video-heavy gallery, when to fully detach a video's `<source>` (not just pause it) as it scrolls out of view, versus just pausing and leaving it buffered?**

Answer: This is a resource-management trade-off between "instant resume if the user scrolls back shortly" and "free the decoder/memory/network resource for content more likely to actually be revisited." A reasonable middle ground pauses immediately on exit (cheap, instantly reversible) but only fully detaches the source after the item has been out of view for some grace period (e.g., a few seconds to tens of seconds) or once some bound on total simultaneously-buffered-but-off-screen videos is exceeded (an LRU-style cap, similar in spirit to bounding a cache) — this avoids the "immediately punish any scroll-past, even a brief one, with a full reload on scroll-back" experience while still ensuring a long scroll session doesn't accumulate unboundedly many buffered-but-invisible video resources.

The trap: choosing one extreme unconditionally — always fully detaching immediately on exit (cheap on resources, but produces a visible reload/rebuffer flicker for a user who scrolls slightly past and immediately back) or never detaching at all (instant scroll-back resume, but resource usage grows without bound across a long session) — a bounded, grace-period-based middle ground is the answer that acknowledges both failure modes and picks a deliberate trade-off between them.

---

**Q (Medium): How would `srcset`/`sizes` need to change for a masonry-style layout where each item's rendered width isn't a clean, small set of fixed breakpoints?**

Answer: `sizes` doesn't require a small fixed set of breakpoints — it can express a more granular, viewport-relative formula (e.g., `sizes="(max-width: 600px) 100vw, calc(33vw - 16px)"`) that approximates a masonry column's actual rendered width reasonably well even if it isn't perfectly precise for every possible column configuration; the browser uses whichever `srcset` candidate is closest to (and not smaller than) the size it computes from that formula at the current viewport, so a reasonably close approximation is generally sufficient — perfect precision isn't required for `srcset` to still meaningfully reduce over-fetching compared to a single fixed-resolution source. What does get harder in a masonry layout is knowing each item's rendered *height* ahead of time for layout-shift prevention, since masonry positioning is itself often computed from each item's natural aspect ratio combined with a target column width — but as long as that computation happens synchronously at layout time (from known `width`/`height` metadata already available from the item's data, not from waiting on the image to actually load), the same "reserve the box before the image arrives" principle still applies, just computed via the masonry algorithm's own column-width logic rather than a simple CSS `aspect-ratio` on a fixed grid cell.

The trap: assuming masonry layouts are incompatible with either `srcset` responsiveness or layout-shift prevention — both remain achievable, they just require slightly more careful sizing logic than a uniform fixed-grid layout, rather than being abandoned as "too hard for masonry."

---

**Q (Medium): Should the low-quality placeholder (blurred or dominant-color) be generated client-side on first load, or precomputed and stored ahead of time?**

Answer: Precomputed and stored ahead of time (typically at upload/processing time, generating a tiny blurred thumbnail or extracting a dominant color alongside the full-resolution asset, stored as part of that item's metadata) is the standard, correct approach — generating a blur/dominant-color placeholder client-side would require first downloading at least some portion of the actual image data to compute it from, which defeats the purpose of a placeholder meant to appear *before* any real image bytes have arrived. A precomputed placeholder (often small enough to inline directly as a base64 `data:` URI in the item's metadata payload, avoiding even a second network request) can render synchronously the instant the grid item's data is available, well before the real image asset begins loading.

The trap: proposing to compute the placeholder client-side "from the first few bytes of the progressive JPEG as they stream in" — this is a real technique in some specialized image-loading pipelines, but it's meaningfully more complex than precomputing at upload time, requires the image format to support progressive decoding usefully at that granularity, and isn't the default, straightforward answer this scenario is looking for.

---

**Q (Low): How would `navigator.connection.saveData` change what this design fetches, and what should happen in browsers where the Network Information API isn't available at all?**

Answer: Where available and `saveData` is true (or `effectiveType` indicates a slow connection), the design should request a lower resolution/quality tier proactively — even at a viewport size that would otherwise warrant a higher-resolution `srcset` candidate — respecting the user's explicit preference to conserve data. Where the API isn't available (it has partial/inconsistent browser support), the design simply falls back to its normal `srcset`/`sizes`-driven behavior with no network-awareness applied — this needs to be a graceful, no-op fallback (never a hard dependency the gallery breaks without), since treating an unsupported, optional API as required would make the feature's absence in some browsers a functional regression rather than the intended "nice-to-have where supported" behavior.

The trap: architecting network-awareness as a required, blocking check before any image request can proceed — if the API's absence (a real, common case) is treated as an error or blocking condition rather than "just skip this optional adjustment," the gallery becomes broken specifically in browsers that don't support a progressive-enhancement-only API, which is the opposite of how an optional refinement should be built.

---

## Self-Assessment

- [ ] Can explain when to reach for native `loading="lazy"` versus custom `IntersectionObserver` logic, and name what the native attribute can't do
- [ ] Can explain why aspect-ratio reservation and lazy-loading are solving two different problems, and why skipping reservation still causes CLS even for deferred images
- [ ] Can justify explicitly excluding above-the-fold content from lazy-loading and connect this to LCP
- [ ] Can design visibility-gated video play/pause/source-detach and articulate the resource trade-off in choosing a grace period
- [ ] Can explain why a gallery at real scale needs both lazy-loading (bandwidth) and virtualization (DOM/memory) as distinct, complementary techniques
- [ ] Can design a lightbox preloading strategy for adjacent items and explain why on-demand loading alone feels sluggish there

---
*Next: Design a Collaborative Document Editor (OT/CRDT Basics) — moves from independently-loading media into the hardest state-synchronization problem in this phase: multiple users concurrently editing the same shared document, where the central new concern is conflict resolution and consistent convergence, not just efficient rendering or data fetching.*
