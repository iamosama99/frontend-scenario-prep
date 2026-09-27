# Slow Initial Load on 3G / Low-end Device

## Quick Reference

| Bottleneck Layer | Symptom Under Throttling | Fix |
|---|---|---|
| Network (bytes over the wire) | Waterfall dominated by download time, not parse/execute | Code-split, compress, lazy-load below-the-fold, right-size images |
| Parse/compile (JS engine cost) | Long "Script Parsing & Compilation" and "Evaluate Script" blocks even after download finishes | Reduce shipped JS volume, defer non-critical scripts, split large bundles |
| Main-thread execution (CPU-bound) | Long Tasks during hydration/initial render, worse on throttled CPU than throttled network alone | Reduce hydration cost (streaming/selective hydration, fewer components hydrating eagerly) |
| Waterfall/dependency chains | Requests fire sequentially instead of in parallel; each round-trip costs full 3G RTT (~400ms+) | Flatten request chains, preconnect/preload, parallelize independent fetches |
| Render-blocking critical path | Nothing paints until a chain of CSS/JS/font resolves | Inline critical CSS, defer non-critical JS, avoid `@import` chains |

## The Scenario

"We're seeing a huge drop-off in signups from users in regions where 3G and older Android devices are still common — way worse than our own testing ever shows, since everyone on the team has a fast phone and good Wi-Fi. Figure out what's actually slow for those users and what you'd do about it."

## Clarifying Questions

- **Do we have field data (RUM/CrUX) segmented by connection type or region, or only our own internal dashboards which are presumably skewed toward good conditions?** CrUX and most RUM tools can segment by effective connection type (`4g`/`3g`/`2g` via the Network Information API) or geography — I'd want the actual field distribution for the affected segment before assuming "3G" means literally 3G-classified connections versus just generally higher-latency/lower-bandwidth mobile networks in that region.
- **Is "slow" here about time-to-interactive/first meaningful content, or specifically about the page failing to load at all (timeouts, users abandoning before anything renders)?** These call for different diagnostic priorities — a page that eventually loads but slowly is a classic waterfall/bundle-size problem; a page that frequently fails to load at all under real 3G conditions might be hitting request timeouts, a resource that's simply too large for a realistic 3G budget (a multi-MB unsplit bundle), or connection instability that a simple "make it faster" fix doesn't address (needs retry/resilience, not just speed).
- **What device profile is actually representative of the affected users — specifically CPU, since "low-end Android" varies hugely** (a $100 device can have 1/10th the single-core performance of a mid-range one)? Network throttling alone (fast connection, slow CPU) versus real combined throttling (slow connection AND slow CPU) produce very different bottleneck profiles — a bundle that parses fine on a throttled-network-only test can still be dominated by parse/compile/execute time on genuinely weak hardware, which is a different problem with a different fix than pure network optimization.
- **Is the app doing client-side rendering with a JS-heavy initial bundle, or is there already server-side rendering / static generation for the initial paint?** If everything currently depends on JS downloading, parsing, and executing before *anything* useful appears, that's a fundamentally different (and usually much worse, on 3G+low-end-CPU) starting point than an app that already server-renders meaningful content and only needs JS for interactivity — the fix set is very different (SSR/streaming vs. optimizing an already-CSR-appropriate bundle).
- **Are there third-party scripts (analytics, ads, chat widgets, A/B testing tools) loaded on this page, and are they render-blocking?** Third-party scripts are extremely common, easy-to-overlook contributors to exactly this profile — they're often added without performance review, can block rendering, and their cost scales the same way (worse on slow network + weak CPU) as first-party code, but get missed because they're "not our code."

## Approach & Trade-offs

**Combined network + CPU throttling is the only representative test condition — network-only throttling alone systematically understates the problem.** A dev machine with a fast multi-core CPU, even with network artificially slowed to "Slow 3G" in DevTools, still parses and executes JS far faster than a genuine low-end Android device's CPU would — meaning a network-only throttled test can look "acceptably slow" while the real combined condition (slow network *and* a CPU that takes 5-10x longer to parse/compile/execute the same JS) is dramatically worse. I'd always test with both DevTools network throttling ("Slow 3G"/"Fast 3G") and CPU throttling (4x-6x slowdown, or ideally an actual representative low-end device) applied together, since that's the only way to see the compounding effect that field users actually experience.

**Separate the four cost categories — bytes, parse/compile, execute, and round-trip latency — since 3G and low-end CPU stress different ones disproportionately.** A slow *network* mostly punishes total bytes transferred and the number of sequential round trips (each one costing a full 3G RTT, often 300-500ms+, which dominates waterfall time far more than it would on a fast connection where RTT is negligible). A slow *CPU* mostly punishes how much JS has to be parsed, compiled, and executed/hydrated, independent of how fast it arrived — a perfectly-sized, well-compressed bundle can still take multiple seconds to parse and execute on weak hardware. Treating "slow load" as one undifferentiated problem and reaching for one category of fix (compress images, say) while the actual bottleneck is JS execution time wastes effort on the wrong lever.

**For a CSR-heavy app, moving meaningful initial content to server-rendered HTML has outsized impact on exactly this profile, more than most individual optimizations.** If the current architecture requires the full JS bundle to download, parse, execute, and run a data fetch before anything user-visible appears, every one of those steps is individually punished by 3G+low-end-CPU, and they're on the critical path sequentially. Server-rendering (or statically generating) the above-the-fold content removes the *entire chain* from blocking first paint — the browser can show real content the moment the HTML response arrives, with JS/hydration happening in parallel/after, rather than as a blocking prerequisite. This is usually a bigger architectural lever than incremental bundle trimming, though it's also a bigger lift — worth stating as a trade-off explicitly (a redesign of the rendering strategy vs. a series of smaller optimizations) rather than assuming it's always the first move.

**Third-party scripts deserve the same scrutiny as first-party code, since they're invisible in a "we tested and it's fine" internal check** if internal testing happens to run with ad-blockers, on a fast connection where the third party's own CDN latency is negligible, or in a region where that third party's infrastructure happens to be fast. I'd specifically check the waterfall for every third-party origin and its blocking behavior under the throttled profile — a chat widget or A/B testing SDK that's synchronously render-blocking can single-handedly dominate the affected users' experience while being effectively invisible in a well-resourced team's daily testing.

## Solution — the diagnostic + fix walkthrough

**Step 1 — set up a representative test condition.** DevTools → Network: "Slow 3G" (or a custom profile matching field RTT/bandwidth data if available) + Performance panel CPU throttle: 6x slowdown. Record a trace loading the page cold (disable cache).

**Step 2 — read the waterfall and trace for the dominant cost category.** Example finding:

```
Waterfall (Slow 3G):
- HTML document:         0 – 900ms   (large TTFB + slow transfer of a bloated HTML response)
- main.js (1.8MB):       900ms – 4200ms  (download alone, before any execution)
- Script evaluation:     4200ms – 7100ms  (parse/compile/execute on 6x-throttled CPU)
- Data fetch (waterfall, not parallel): 7100ms – 8400ms
- First meaningful paint: ~8400ms
```

This immediately shows two distinct problems worth separate fixes: (1) a 1.8MB main bundle dominating download time on 3G, and (2) the data fetch only starting *after* the bundle finishes executing — a sequential chain instead of a parallel one.

**Step 3 — fix the bundle size problem with route/feature-based code-splitting** (using the diagnostic approach from [[03-bundle-size-regression-after-release]] to find what's in that 1.8MB — commonly, everything for every route bundled together):

```ts
// Before: every route's code in one bundle, all shipped for the initial load
import Dashboard from './Dashboard';
import Settings from './Settings';
import Reports from './Reports';

// After: only the initial route's code ships up front; others load on demand
const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));
const Reports = lazy(() => import('./Reports'));
```

**Step 4 — fix the sequential fetch-after-hydration chain by making the data fetch independent of JS bundle execution**, e.g. via a server-rendered initial payload embedded directly in the HTML (no client round-trip needed for first paint) or, at minimum, firing the fetch in parallel with the bundle download rather than after it:

```html
<!-- Data needed for first paint is embedded in the initial HTML response,
     available the instant the document parses — no waiting on JS to fetch it -->
<script>window.__INITIAL_DATA__ = {"user": {...}, "widgets": [...]};</script>
```

```ts
// Client hydrates from the embedded data instead of re-fetching
const initialData = window.__INITIAL_DATA__;
```

**Step 5 — audit and defer/re-evaluate third-party scripts** found in the same waterfall:

```html
<!-- Before: render-blocking, loaded synchronously in <head> -->
<script src="https://widget.example.com/chat.js"></script>

<!-- After: deferred, doesn't block parsing or the critical rendering path -->
<script src="https://widget.example.com/chat.js" defer></script>
```

**Step 6 — re-measure under the same combined throttle profile, and specifically check the new breakdown's dominant category** — confirming, for instance, that the fix moved the bottleneck from "waiting on a 1.8MB bundle" to something smaller and check whether a *new* bottleneck (e.g., now CPU-bound on hydration cost for the smaller-but-still-substantial remaining bundle) has taken its place, since fixing the largest bottleneck often just reveals the next one.

**Step 7 — validate against real field data (RUM segmented by connection type/region), not just the lab trace**, since a lab improvement under a fixed throttle profile doesn't guarantee the same magnitude of improvement across the actual heterogeneous field distribution of devices/networks in the affected region.

> **Check yourself:** Given the four-category breakdown (bytes / parse-compile / execute / round-trip-latency), which category does splitting a large bundle into smaller chunks primarily help, and which category does it leave completely unaddressed?

## Gotchas

**Testing only with network throttling and not CPU throttling, and concluding the page "loads fine on 3G."** This is close to the single most common mistake in this exact scenario — a fast dev CPU parsing/executing a large bundle in 200ms hides a bottleneck that takes 2 seconds on real low-end hardware, since network throttling alone doesn't touch parse/compile/execute cost at all.

**Optimizing image weight aggressively while missing that the JS bundle itself is the dominant cost.** Image optimization is often the first instinct (it's visible, easy to point at), but on a JS-heavy CSR app, the bundle's download+parse+execute chain frequently dwarfs image cost, especially if images are already reasonably lazy-loaded/responsive — the waterfall/trace breakdown, not intuition, should determine where effort goes.

**Code-splitting by route but leaving a large, shared "vendor" chunk unaddressed**, so every route still pays for a bloated shared bundle regardless of which specific route-level chunk got smaller — route-splitting helps the "not-needed-yet" code, but doesn't help the genuinely-shared code, which needs its own scrutiny (the bundle-analyzer techniques from the previous scenario apply directly here).

**Missing that a redirect chain, a DNS lookup for a new third-party origin, or a lack of `preconnect` adds full extra round trips — each disproportionately expensive on high-latency 3G.** On a fast connection, an extra DNS lookup or redirect costs a barely-noticeable few tens of milliseconds; on 3G with 300-500ms+ RTT, each additional round trip in the chain (DNS → TCP → TLS → request → response, repeated for every new origin encountered) can add whole seconds, cumulatively — a `<link rel="preconnect">` for known third-party origins, or eliminating an unnecessary redirect, has outsized impact specifically under high-latency conditions.

**Treating "first paint" as the finish line and not checking time-to-interactive/INP on the same throttled profile.** A page that paints something quickly (e.g., server-rendered shell) but where the JS bundle then takes several more seconds to parse/execute/hydrate on weak CPU can look "fast" by a paint-only metric while still being unresponsive to input for a long stretch afterward — the full picture needs both paint timing and interactivity timing under the same combined throttle.

## Follow-up Questions

**Q (High): Explain precisely why combined network + CPU throttling is necessary to reproduce this class of bug, and what specifically gets missed by network-only throttling.**

Answer: DevTools' network throttling slows down data transfer (bandwidth/latency simulation) but runs entirely on the host machine's real CPU — so parse, compile, and execute/hydrate time for whatever JS eventually arrives is measured at the dev machine's (typically fast, multi-core, desktop-class) actual speed, not the field device's. Real low-end Android devices can take 5-10x longer (or more) to parse and execute the same JS payload compared to a modern dev machine, independent of how the bytes arrived — this means a bug where the bottleneck is genuinely CPU-bound (a large bundle's parse/compile cost, an expensive hydration pass, heavy synchronous JS on the critical path) is invisible under network-only throttling, since the CPU side of the equation is never actually slowed down to match. Combined throttling (network AND CPU, via DevTools' CPU throttle slider or an actual representative device) is the only way to see both halves of the real field bottleneck simultaneously, and specifically to see how they *compound* — a slow network delaying a large bundle's arrival, followed by a slow CPU taking much longer than the dev machine would to then parse and execute it.

The trap: describing the fix as just "test with Slow 3G in DevTools" — network throttling alone is necessary but not sufficient, and omitting CPU throttling specifically hides exactly the class of bottleneck (parse/compile/execute cost) that's often dominant on genuinely low-end hardware.

---

**Q (High): A trace shows the JS bundle downloads reasonably quickly even on Slow 3G (it's well-compressed and not huge), but "Script Evaluation" takes several seconds on 6x CPU throttle. What does this indicate, and what's the fix — is it the same as fixing a bundle-size problem?**

Answer: This indicates the bottleneck has shifted from network transfer to JS engine cost (parsing, compiling, and executing/running the script, including any synchronous initialization or hydration work it performs) — the bundle isn't too large to *download*, but it's expensive to *run* on weak hardware, which is a distinct problem from download size even though both are sometimes casually lumped together as "the bundle is too big." The fix overlaps partially with bundle-size fixes (less code shipped generally means less to parse/compile too) but isn't identical — specifically relevant here are reducing the amount of JS that needs to execute *synchronously before the page is usable* (deferring non-critical initialization, lazy-instantiating features not needed immediately, reducing eager hydration scope for a framework that supports partial/selective hydration), since parse/compile cost scales with bytes shipped but *execute* cost (especially hydration walking a large tree and attaching all its event handlers) scales with the amount of actual runtime work being done, which byte-count reduction alone doesn't fully address if the remaining code still does a lot of synchronous work per byte.

The trap: assuming any "shrink the bundle" fix (code-splitting, tree-shaking, minification) automatically also fixes execution-time cost — it helps (less to parse/compile), but a bundle that's small in bytes and still does expensive synchronous work at startup (e.g., initializing a large in-memory data structure, hydrating a huge component tree eagerly) can still be slow to *execute* on weak CPU independent of its download size.

---

**Q (High): The team wants to fix this primarily by adding server-side rendering. Walk through the actual mechanism by which SSR helps this specific scenario, and one thing SSR does NOT fix on its own.**

Answer: SSR moves the generation of the initial HTML (including meaningful, visible content) to the server, so the browser can begin painting real content the moment the HTML response arrives and is parsed — removing the current CSR-only chain's dependency on "download JS bundle → parse/compile it → execute it → fetch data → render" being fully complete before *anything* user-visible appears. This directly and substantially helps first-paint-related metrics (First Contentful Paint, and often LCP if the LCP element is part of the server-rendered content) precisely because it removes several entire steps from the critical path for visible content, each of which was individually punished by 3G latency and low-end CPU. What SSR does NOT fix on its own is the JS bundle still needing to download, parse, execute, and hydrate for the page to become *interactive* — a server-rendered page with a large, slow-to-hydrate client bundle can show content quickly (good FCP/LCP) while remaining unresponsive to input for a long stretch afterward (poor INP/TTI) if hydration cost isn't separately addressed — SSR alone can even make this gap more visible, since users now see content that *looks* ready to interact with well before it actually is.

The trap: treating SSR as a complete fix for "slow load" broadly — it specifically addresses the paint/content-visible side of the metric set, and without also addressing hydration/bundle execution cost, it can create a worse-feeling failure mode (visible-but-unresponsive) than a purely CSR page where at least the blank state doesn't invite premature interaction.

---

**Q (Medium): Why does a single extra DNS lookup or redirect matter so much more on 3G than on a fast connection — quantify the reasoning, not just "latency is higher."**

Answer: Every new step in a connection chain (DNS resolution, TCP handshake, TLS negotiation, an HTTP redirect requiring a fresh request) costs at least one full network round-trip, and round-trip time (RTT) on 3G is commonly 300-500ms or more, versus perhaps 20-50ms on a fast broadband/Wi-Fi connection — meaning the *same* structural inefficiency (say, three sequential round trips: DNS lookup for a new third-party origin, then TCP+TLS handshake, then the actual request) costs roughly 60-150ms total on a fast connection (barely perceptible) but 900ms-1.5s+ on 3G (very perceptible, and potentially a meaningful fraction of an already-tight load budget). This is why request-chain flattening, `preconnect`/`dns-prefetch` hints, and eliminating unnecessary redirects have disproportionate value specifically for high-latency conditions — the fix's absolute time savings scale with the connection's RTT, so the same code change that saves an imperceptible amount on a fast connection can save a very meaningful amount for exactly the affected users in this scenario.

The trap: reasoning about network chain inefficiencies in the abstract ("fewer round trips is better") without quantifying that the *value* of removing one is proportional to RTT — this is precisely why a fix that looked like a negligible micro-optimization in the team's own fast-network testing can be one of the highest-leverage fixes for the actually-affected 3G segment.

---

**Q (Medium): How would you prioritize between (a) further shrinking the JS bundle and (b) embedding initial data directly in the HTML to avoid a client-side fetch waterfall, if you could only ship one this sprint?**

Answer: I'd prioritize based on which one the actual waterfall/trace shows as the larger contributor for the affected condition, not a general rule — the diagnostic from Step 2 in this scenario names the specific breakdown (e.g., "4200ms parse/execute" vs. "1300ms for a sequential post-hydration fetch"), and the larger number under the representative combined throttle is where effort should go first, since fixing a smaller contributor first leaves most of the actual user-facing delay unaddressed while consuming the same sprint's capacity. All else being equal, embedding initial data to eliminate a *sequential* round trip tends to have an outsized, RTT-multiplied benefit specifically on high-latency connections (per the DNS/redirect reasoning above) even for a relatively modest amount of data, whereas bundle-shrinking benefits scale more with bandwidth-constrained conditions and CPU parse/execute cost — but the actual trace data for this specific page, not a general heuristic, should make the final call.

The trap: picking a fix based on which one is more familiar/comfortable to implement (bundle shrinking is a well-worn playbook) rather than which one the measured breakdown shows as the actual larger contributor to this specific page's slow load.

---

**Q (Low): The affected region has a reasonably fast average connection speed according to general reports, but your field data still shows this problem concentrated there. What might explain the mismatch, and how would you dig further?**

Answer: Aggregate/average regional connection-speed reports can mask a bimodal or long-tail distribution — a region might have good average bandwidth from urban fast-connection users while a meaningful fraction of actual users on the affected page are on genuinely constrained connections (rural coverage, congested mobile networks at peak hours, older 3G-only devices still in wide use) that don't move the regional average much but still represent a large enough absolute user count to show up clearly in page-specific field data. I'd dig further by segmenting the RUM data more finely than "region" alone — by effective connection type (via the Network Information API, where available), by device model/CPU class if capturable, and by time-of-day (network congestion patterns), to find the actual sub-segment driving the signal rather than treating "region" as a proxy for "connection quality," which it often isn't a precise one for.

The trap: taking a regional average network-speed statistic at face value as evidence against a network-related hypothesis — averages hide exactly the tail-end distribution that field performance problems for a global product are usually concentrated in.

---

## Self-Assessment

- [ ] Can explain why combined network + CPU throttling (not network alone) is necessary to reproduce this class of bug, with the specific mechanism
- [ ] Can distinguish bytes-over-the-wire cost from parse/compile/execute cost and give a fix that addresses each specifically
- [ ] Can explain the mechanism by which SSR improves first-paint metrics and name what it does NOT fix (hydration/interactivity cost)
- [ ] Can quantify why request-chain length (DNS/redirects/round trips) matters disproportionately more under high-RTT conditions
- [ ] Can prioritize between competing fixes (bundle size vs. request waterfall vs. third-party scripts) based on trace evidence rather than habit

---
*Next: Redundant Network Requests on a Page — from "requests are slow because of network conditions" to "requests shouldn't be happening at all," auditing duplicate/unnecessary fetches that waste bandwidth and time regardless of connection quality.*
