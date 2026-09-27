# Production Memory Leak Triage

## Quick Reference

| Triage Step | Tool/Technique | What It Tells You |
|---|---|---|
| Confirm it's real and scope it | RUM/crash data (`performance.memory` sampling, Sentry/Datadog session data, browser crash reports) | Which users/pages/duration are affected, whether it's growing or a one-time bump |
| Narrow to a feature area | Feature-flag bisection, checking what's mounted on the affected page(s) | Which code path to actually profile, out of a large app |
| Reproduce locally under matching conditions | Same route, same user flow, long-session interaction loop | A repeatable local case to attach DevTools to |
| Confirm and locate with heap snapshots | Chrome DevTools Memory panel, snapshot comparison | The specific retained object/constructor and its retainer chain |
| Decide: fix vs. mitigate now | Severity/impact vs. fix complexity | Ship a real fix, or a stopgap (feature flag off, force-refresh after N minutes) while the real fix is prepared |

## The Scenario

"We're seeing an uptick in 'page unresponsive' / tab-crash reports from users who leave a specific dashboard page open for extended periods — sometimes a full workday. It's not reproducing for you in a quick five-minute test. Walk me through how you'd actually triage this in production, from 'a handful of vague user reports' to 'here's the specific leak and the fix' — not just how you'd fix a leak you'd already located."

## Clarifying Questions

- **What do we actually know about the affected users/sessions so far** — is this from support tickets, crash reporting (e.g., a "page unresponsive" browser-level report), RUM data showing memory growth over session duration, or just anecdotal? The evidence quality determines the whole triage path: crash reports with stack traces or heap info are far more actionable than "someone said it felt slow after a while."
- **Is this specific to one page/dashboard, or does it happen anywhere in the app given enough time?** Scoped to one page narrows the search space enormously (a handful of components mounted there, vs. the entire app's codebase) — I'd want confirmation this really is page-specific and not just "the page users happen to leave open longest," which would implicate something more global (a shared provider, a websocket connection, a polling hook used app-wide) instead.
- **Does the dashboard have live/real-time data — websocket updates, polling, streaming charts — or is it mostly static after initial load?** Long-lived-open pages with continuous incoming data are exactly where accumulation bugs concentrate (a message handler appending to an ever-growing array with no eviction, a chart library retaining every historical data point, a reconnecting websocket that doesn't clean up prior connections) — a static page that only leaks from occasional user interaction has a narrower, different set of likely causes.
- **Can we get a heap snapshot or `performance.memory` reading from an actually-affected real user/session**, even informally — e.g., asking a support-flagged user to open DevTools and export a snapshot, or checking if RUM tooling already samples `performance.memory` over session lifetime? Since this doesn't reproduce quickly in a normal test session, getting *any* real data point from a genuinely long-lived affected session is far more valuable than repeatedly trying to force a five-minute repro that isn't representative of the actual failure condition (which seems to require extended duration).
- **What build is running for affected users — is this new, or has it always been present and only now being reported/noticed** (e.g., because a recent change made the dashboard more commonly left open all day, or added a new live-data feature)? A recent regression narrows to a diffable range of changes; a long-standing but only-now-reported issue means the leak might be older and unrelated to recent work, requiring a broader search.

## Approach & Trade-offs

**Triage in production is a scoping problem before it's a debugging problem — resist jumping straight to DevTools on your own five-minute session, since it won't reproduce.** The scenario explicitly states this doesn't show up in a quick test — that's information, not an obstacle to route around by trying harder to force it. I'd first establish *what's actually growing, how fast, and under what conditions*, using whatever real signal exists (RUM memory sampling if instrumented, crash reports, a support ticket's described usage pattern), before spending time trying to attach a profiler to a session that may not be long/active enough to show the effect.

**If there's no existing memory instrumentation in RUM, the immediate fix-adjacent step is adding some, rather than debugging blind.** A lightweight `performance.memory.usedJSHeapSize` sample (Chrome-only, but often enough coverage) reported periodically alongside existing analytics, or even a rough proxy (heap size at fixed intervals, logged for a sample of long sessions with user consent/existing telemetry infrastructure), turns "vague reports" into an actual growth curve — if heap size is provably climbing linearly over hours for affected sessions but flat for others, that's a strong, cheap signal before any deep investigation, and it also gives a way to *validate* a fix later without waiting for user reports to stop.

**Reproduce the *shape* of the affected usage, not just the page.** If the real failure requires "left open for a workday," I wouldn't try to literally wait eight hours — I'd simulate the accumulation by scripting or manually repeating whatever the live-data/interaction pattern is (a websocket message every few seconds, simulated at high frequency for a few minutes; a chart re-render loop run in a tight cycle) to compress hours of real usage into a reproducible few minutes, on the theory that a genuine accumulation bug's *rate* scales with the number of events/interactions, not literally with wall-clock time — so speeding up the event rate should surface the same bug faster.

**Once reproducible, the diagnostic method is the same heap-snapshot-comparison technique as any memory leak** (see [[04-memory-leak-uncleaned-subscriptions]] in Phase 3 for the underlying mechanism) — but the triage-specific judgment call is *when to stop investigating and ship a mitigation* versus *keep digging for the root cause*. A page that's crashing tabs for real users in production justifies a fast, possibly-imperfect mitigation (a feature flag disabling the live-update feature, a periodic "soft refresh" prompt after N hours, capping how much history an in-memory buffer retains) shipped quickly, in parallel with — not instead of — the deeper root-cause fix, if the full proper fix will take longer than the user impact can tolerate.

## Solution — the triage walkthrough

**Step 1 — establish the growth signal.** Add (or use existing) lightweight heap sampling to RUM:

```ts
// Sampled periodically for a subset of sessions, reported alongside existing analytics
function sampleHeap() {
  if (!(performance as any).memory) return; // Chrome-only API
  analytics.track('heap_sample', {
    usedJSHeapSize: (performance as any).memory.usedJSHeapSize,
    page: location.pathname,
    sessionDurationMs: Date.now() - sessionStartTime,
  });
}
setInterval(sampleHeap, 5 * 60 * 1000); // every 5 minutes
```

Querying this after a day or two of collection shows whether `usedJSHeapSize` on the dashboard route climbs roughly linearly with `sessionDurationMs` (leak) versus plateauing after initial load (normal, healthy behavior) — and, comparing across routes, confirms whether it's specific to the dashboard as suspected.

**Step 2 — compress the accumulation into a fast local repro.** Say the dashboard subscribes to a websocket for live metric updates. Instead of waiting hours, script rapid-fire simulated messages against a local/staging instance:

```ts
// Dev-only harness: fire 1000 simulated messages in quick succession
for (let i = 0; i < 1000; i++) {
  mockSocket.emit('metric-update', { id: i, value: Math.random(), timestamp: Date.now() });
}
```

If heap usage climbs noticeably and doesn't come back down after a forced GC, the accumulation reproduces in seconds instead of hours.

**Step 3 — heap snapshot comparison to locate the retained object.** DevTools → Memory → snapshot before the burst, run the burst, force GC, snapshot after, Comparison view. Suppose it shows thousands of retained `MetricPoint` objects:

```ts
// Found: a chart data buffer with no eviction policy
class LiveMetricChart {
  private allPoints: MetricPoint[] = []; // BUG: grows forever, one push per message, never trimmed

  onMessage(point: MetricPoint) {
    this.allPoints.push(point);
    this.render();
  }
}
```

Every incoming message grows `allPoints` with no cap — over a workday of continuous updates, this is an unbounded array holding every data point ever received, each retaining whatever it closed over/referenced.

**Step 4 — fix with a bounded buffer matching what's actually needed for display** (a chart showing "last hour" doesn't need six months of retained points in memory):

```ts
class LiveMetricChart {
  private allPoints: MetricPoint[] = [];
  private readonly maxPoints = 500; // matches the actual visible window's needs

  onMessage(point: MetricPoint) {
    this.allPoints.push(point);
    if (this.allPoints.length > this.maxPoints) {
      this.allPoints.shift(); // or splice a batch off periodically — cheaper than one-at-a-time for high-frequency streams
    }
    this.render();
  }
}
```

**Step 5 — ship a fast mitigation in parallel if the real fix needs more validation time.** E.g., a feature flag capping the live-update feature's retention immediately, or (as a last resort, communicated transparently) prompting users to refresh after N hours of continuous use, while the bounded-buffer fix goes through normal review/rollout.

**Step 6 — validate against the same signal that surfaced the problem**, not just the local repro — watch the RUM heap-sample data for the dashboard route over the following days to confirm the growth trend actually flattens for real users, since a local repro passing doesn't guarantee every accumulation path in the real, more varied usage patterns was caught.

> **Check yourself:** If the RUM heap-sampling data showed growth on *several* different pages, not just the dashboard, what would that suggest about where to look, versus a leak scoped to one page?

## Gotchas

**Trying to reproduce "leaves the page open for a workday" by literally waiting a workday.** Compressing the event rate (simulating hours of live-data traffic in minutes) is almost always valid for accumulation bugs, since the bug is driven by *event count*, not wall-clock time — insisting on real-time reproduction wastes enormous amounts of investigation time for no additional confidence.

**Treating a single successful local repro-and-fix as sufficient validation for a production issue that was only vaguely reported.** The original signal was vague (support tickets, crash reports) — closing the loop means confirming via the *same class of signal* (RUM data, reduced crash-report rate) that real-world impact actually dropped, not just that a specific hypothesized mechanism was fixed in isolation; there could be a second, independent leak contributing to the same symptom.

**Assuming `performance.memory` (or any single heap-size metric) tells the whole story.** `usedJSHeapSize` reflects JS heap only — a leak in retained DOM nodes not counted the same way, or memory pressure from something outside the JS heap (large decoded images/canvases, video buffers), can contribute to the same "page becomes unresponsive/crashes" symptom without showing up prominently in a JS-heap-only metric; a full heap snapshot's node/DOM breakdown is more complete than a single scalar.

**Fixing the specific accumulating array found in Step 3 and declaring the investigation over, without checking for siblings.** A codebase with one "push and never trim" bug in a live-data component often has the same pattern in adjacent components built by the same team around the same time — a quick audit for the same anti-pattern (unbounded array/Map with a push and no corresponding cap/eviction) elsewhere in the same feature area is cheap insurance against finding this again in three months.

**Shipping only the emergency mitigation (feature flag off, forced refresh) and treating it as done.** A mitigation that hides the symptom without fixing the underlying accumulation leaves the actual bug in the codebase, waiting to resurface the next time the mitigated feature is re-enabled or the workaround is forgotten — it needs to be tracked as a known follow-up, not quietly closed out.

## Follow-up Questions

**Q (High): You have no RUM memory instrumentation at all, and reproducing this locally in a normal five-minute session hasn't worked. What's your very first concrete action, before touching DevTools?**

Answer: Get *any* real signal about the actual growth pattern before spending time trying to force a local repro that isn't representative — this could mean adding a lightweight `performance.memory` sample to existing analytics/RUM (even a rough, Chrome-only proxy is far better than nothing) and waiting a day or two for data, or, faster, asking an affected user/support agent to open DevTools on their actual long-lived session and export a heap snapshot directly, or checking existing crash-reporting tooling for any memory-related context already being captured. The point is that "doesn't reproduce in five minutes" is itself the key piece of information — it tells me the failure condition depends on either extended duration or a usage pattern not present in a quick manual test, and guessing at a fix without confirming the actual growth curve risks fixing something that isn't the real cause, then having no way to know the real fix worked until users report improvement (or don't) weeks later.

The trap: jumping straight into DevTools and trying harder/longer to manually reproduce it — without a signal that confirms *what's* growing and under *what* conditions, extended manual attempts to reproduce are a low-probability, low-information use of time compared to instrumenting for real data first.

---

**Q (High): Explain the reasoning behind compressing hours of real usage into a fast local repro by simulating a higher event rate — why is this a valid technique, and when would it NOT be valid?**

Answer: It's valid for accumulation bugs whose root cause is "an unbounded collection grows by a fixed amount per event, with no eviction," because the resulting memory growth is a function of *event count*, not literal elapsed time — firing 1000 simulated events in 10 seconds produces materially the same retained-object growth as 1000 real events spread across a workday, since nothing about the underlying bug's mechanism depends on the gaps between events (assuming no time-based logic, like a periodic cleanup timer, is what's supposed to be preventing the leak and is being bypassed by the compression). It would NOT be valid if the actual leak mechanism is time-based rather than event-count-based — e.g., a bug that only manifests because a `setInterval`-based cleanup is supposed to run *and does*, but something about long-elapsed-time behavior (a token expiring and triggering a different, buggier reconnection code path; a date/time computation that only misbehaves after real calendar time passes) is the actual trigger, in which case compressing event *frequency* without also letting real time elapse wouldn't reproduce it, and could even mislead by reproducing a different bug that only looks similar.

The trap: applying rate-compression uncritically to every "long session" bug — it's specifically valid for count-driven accumulation, and conflating that with genuinely time-driven failure modes can lead to either a false negative (real bug requires real elapsed time, compression doesn't trigger it) or investigating the wrong mechanism entirely.

---

**Q (High): Two components on the same dashboard both hold live-data arrays with no cap. Fixing only the one that showed up as the largest retainer in your heap snapshot — is that sufficient? How would you check?**

Answer: Not necessarily sufficient on its own — a heap snapshot at one point in time shows what's *currently* the largest retained set, but a second, slower-growing unbounded collection can be present and genuinely leaking without yet being the dominant contributor at the moment the snapshot was taken, especially if the snapshot was captured after a relatively short compressed-repro burst rather than a full extended session. I'd check by re-running the same comparison methodology (before/after snapshot around a burst) with attention specifically to *all* retained object types that grew, not just the single largest one, and cross-reference against a code-level audit for the same "push/append with no cap" pattern across sibling components on the same page — since a page built with a consistent live-data pattern often repeats the same mistake across components, checking every array/Map fed by live updates on the same page directly, rather than relying solely on which one happened to dominate one snapshot's comparison, is the more complete check.

The trap: treating "biggest retainer in one snapshot" as synonymous with "the only leak" — snapshot comparison finds what grew during the specific window measured, and a smaller-magnitude but still-genuine leak elsewhere on the same page can be missed if the investigation stops at the first (or largest) hit.

---

**Q (Medium): How would you decide whether to ship an immediate mitigation (feature flag, forced refresh prompt) versus waiting for the full root-cause fix to go through normal review?**

Answer: I'd weigh actual user impact severity (tab crashes causing lost unsaved work is more urgent than "feels sluggish after hours") against how long the full fix realistically needs (a one-line bounded-buffer change might not need a separate mitigation at all if it can ship same-day; a fix requiring a broader refactor of how live data is buffered might genuinely need weeks) and how quickly a mitigation can be deployed relative to the normal fix (a feature flag flip can often go out in minutes via existing infrastructure, versus a code change needing a full release cycle). If the fix is fast and low-risk, shipping it directly is usually simpler and better than a two-step mitigate-then-fix process, which adds coordination overhead (remembering to remove the mitigation, tracking two changes instead of one). A mitigation is worth the extra process specifically when there's a real gap between "user impact is happening now" and "the properly-reviewed fix will be ready" — e.g., the fix touches a data structure that a lot of other code depends on and needs more careful review/testing than the urgency of the crash reports can wait for.

The trap: defaulting to "always ship a mitigation first" as a reflexive caution, or the opposite extreme of "never ship a stopgap, only the real fix" — the right call depends on the actual gap between fix readiness and user-visible severity, not a fixed policy either way.

---

**Q (Medium): Why might `performance.memory.usedJSHeapSize` fail to reveal a real leak, or show a leak that isn't actually a problem?**

Answer: It only reports the JS heap, so leaks in retained DOM nodes that aren't primarily referenced from JS-heap objects, or memory pressure from non-JS-heap sources (decoded `<canvas>`/`<img>` bitmap data, video decode buffers, WASM linear memory), don't show up in this number even if they're the actual cause of a tab becoming unresponsive — a JS-heap-only investigation can look "clean" while a real problem exists elsewhere. Conversely, a rising `usedJSHeapSize` over a session isn't automatically a leak — the browser's GC runs lazily and heap size legitimately fluctuates and grows somewhat before a collection cycle reclaims it, especially for an app that's genuinely doing more work as a session progresses (more data loaded, more UI state); a true leak claim needs the *forced-GC-then-still-retained* confirmation (the standard heap-snapshot-comparison technique), not just "the number went up over time" from a single unforced reading.

The trap: treating `usedJSHeapSize` as a complete or immediately-conclusive leak indicator on its own — it's a useful, cheap trend signal for triage (Step 1 above), but confirming an actual leak (versus normal heap fluctuation, or a non-JS-heap memory problem) requires the fuller snapshot-comparison methodology.

---

**Q (Low): If this dashboard is a single-page app that users never navigate away from or refresh, does that change how seriously you'd weigh even a "small" per-event leak compared to the same leak in a typically-short-lived page?**

Answer: Yes, substantially — a per-event leak that's individually tiny (a few hundred bytes retained per websocket message) is functionally harmless on a page users load, use for two minutes, and navigate away from (the whole page, leak included, gets discarded on navigation), but on a page explicitly designed to be left open indefinitely, the same per-event cost compounds without any natural reset point, over potentially unbounded real time and event count. This changes the engineering bar for "is this leak worth fixing now" — for a page with this usage pattern, essentially any unbounded-growth-with-live-data pattern deserves scrutiny and a cap, even ones that would be dismissed as negligible on a typical short-lived page, precisely because the assumption that "the page will eventually be discarded, bounding the damage" doesn't hold here.

The trap: applying a uniform "is this leak big enough to matter" threshold across the whole app, without factoring in how long a given page is actually expected to stay open — the same absolute per-event leak size warrants very different urgency depending on that context.

---

## Self-Assessment

- [ ] Can describe the triage-before-debugging sequence: confirm and scope the signal (RUM/crash data) before attempting a local repro
- [ ] Can explain why compressing event rate is valid for reproducing accumulation bugs, and identify the case (time-driven bugs) where it isn't
- [ ] Can walk through adding lightweight `performance.memory` sampling to RUM and what a growth-vs-plateau curve would indicate
- [ ] Can reason about when to ship a fast mitigation (feature flag, forced refresh) alongside, not instead of, the root-cause fix
- [ ] Can name a limitation of `usedJSHeapSize` as a leak signal and what a heap-snapshot comparison adds beyond it

---
*Next: Slow Initial Load on 3G / Low-end Device — shifts from a long-session accumulation problem to a first-load, resource-constrained-device problem, where the diagnostic tools (throttled network/CPU profiles) and fixes (code-splitting, resource prioritization) are mostly load-time rather than runtime.*
