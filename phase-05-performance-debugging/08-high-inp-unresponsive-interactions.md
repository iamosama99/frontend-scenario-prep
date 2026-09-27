# High INP / Unresponsive Interactions

## Quick Reference

| INP Phase | What's Being Measured | Common Cause |
|---|---|---|
| Input delay | Time from the physical interaction to the event handler starting to run | Main thread busy with an unrelated task when the input occurs |
| Processing time | Time the event handler(s) themselves take to run | Expensive synchronous work directly in the handler (state computation, DOM reads/writes) |
| Presentation delay | Time from handler completion to the next frame being painted | Large re-render triggered by the update, or layout/paint cost from the resulting DOM changes |
| INP metric itself | The *worst* (or high-percentile) interaction latency across the whole page's session, not an average | One bad interaction (an occasional expensive filter/sort) can dominate the score even if most are fast |

## The Scenario

"Our Core Web Vitals report shows INP at 340ms — 'needs improvement.' Product hasn't heard specific complaints, but the number is what it is and it affects our search ranking. Explain what INP actually measures, find what's driving ours up, and fix it."

## Clarifying Questions

- **Is the 340ms figure a field aggregate (p75 INP from CrUX/RUM, across all interactions across all users) or from a single lab measurement/synthetic test?** INP is fundamentally a field metric — it's defined as the worst (technically, a high-percentile, effectively near-worst for typical traffic volumes) interaction latency observed *within a single user's session*, then aggregated as p75 across users; a lab tool can simulate specific interactions but can't reproduce "the worst interaction across a real session" the same way, so I'd want to know whether 340ms represents genuine field data before treating it as the actual target to chase in a lab environment.
- **Does the RUM tooling report which specific interaction (element, event type) is driving the score, or only the aggregate number?** Modern web-vitals libraries and some RUM tools can attribute INP to a specific `target` element and event type, not just the number — if that attribution exists, it points directly at what to profile; without it, the investigation starts broader (which interactions are least likely to be fast, based on what's known about the app) rather than with a specific known culprit.
- **Is the interaction distribution spread evenly across many different interactive elements, or concentrated on a small number of specific ones** (e.g., one particular filter dropdown, one "add to cart" button)? INP being driven by one specific bad interaction (common) has a narrow, targeted fix; INP being moderately bad *everywhere* points at something systemic (e.g., a large global re-render pattern, an app-wide performance issue affecting every interaction similarly) with a broader, more architectural fix.
- **What does "the interaction" typically look like from a code perspective — a simple click handler, a form input triggering validation on every keystroke, a button triggering a state update that re-renders a large portion of the page?** This shapes which of the three INP phases (input delay / processing time / presentation delay) is likely dominant, and therefore where to look first in a trace.
- **Since product "hasn't heard specific complaints" despite a middling INP score — is it possible the affected interactions are ones users don't consciously notice as "laggy" (e.g., a background-ish toggle) versus core interactions users would definitely complain about if slow (checkout button, primary navigation)?** This matters for prioritization: a 340ms INP driven by a rarely-used, non-critical interaction is a lower-priority fix than the same number driven by a core, high-frequency interaction that happens not to have generated a support ticket yet.

## Approach & Trade-offs

**INP replaced First Input Delay (FID) specifically because it measures the full interaction cost across a session, not just the first one — this distinction should shape where I look.** FID only measured the delay before the *very first* interaction's handler started running, which systematically missed problems that only show up on later interactions (e.g., a page that's fine on first click but degrades after some data has loaded and state has grown, or an interaction whose cost scales with something that accumulates during the session). INP is computed from *all* interactions during a session and reports a high percentile of their latencies — meaning a page can have a perfectly fine "average" interaction speed while INP is dragged up by a smaller number of genuinely slow interactions, which is exactly why aggregate impressions ("nothing feels wrong to us") can coexist with a bad INP score: the team's own casual usage may simply not be hitting the specific slow interaction(s) often enough to notice, while the field p75 aggregates across enough real user sessions that it does surface.

**Break INP into its three phases (input delay, processing time, presentation delay) before proposing a fix, since — exactly like the LCP breakdown in this phase's first scenario — each phase has a structurally different cause and fix.** Input delay being large means the main thread was busy with *something else* (an unrelated long task, a scheduled timer, a large ongoing render) at the moment the user physically interacted — the fix is reducing what's competing for the main thread at typical interaction moments, not the interaction's own handler. Processing time being large means the handler itself is doing too much synchronous work — the fix is inside the handler (this overlaps directly with the Long Task diagnosis technique from the previous scenario in this phase). Presentation delay being large means the handler finished promptly but the resulting visual update (a large re-render, expensive layout/paint from DOM changes) takes a while to actually reach the screen — the fix is in what the update triggers downstream, not the handler's own execution.

**Because INP is a percentile-based, worst-case-leaning field metric, chasing "average interaction speed" is the wrong target — the fix needs to specifically address the tail.** Making 95% of interactions marginally faster does very little to a p75/p98-style metric if the remaining slow interactions stay exactly as slow — improving the actual worst-behaved interactions (even if they're a small fraction of total interaction volume) moves the metric far more than broad, shallow improvements everywhere. This should directly shape prioritization: find and fix the specific worst offenders identified via RUM attribution (if available) rather than performing a general "make everything a bit snappier" sweep.

**React's concurrent features (`useTransition`, `startTransition`) exist specifically for the presentation-delay and some processing-time cases, and are worth reaching for over manual chunking where applicable**, since they let the framework's own scheduler interrupt non-urgent rendering work in favor of the next urgent update (like the user's next keystroke), directly targeting the "the browser is busy finishing a low-priority update when a new interaction comes in" failure mode that manual `setTimeout`-based chunking (from the previous scenario) handles more crudely for arbitrary JS but that React's scheduler can handle natively for state-driven re-renders.

## Solution — the diagnostic + fix walkthrough

**Step 1 — get INP attribution from field data, if available.** Using the `web-vitals` library's attribution build in production RUM:

```ts
import { onINP } from 'web-vitals/attribution';

onINP((metric) => {
  analytics.track('inp', {
    value: metric.value,
    target: metric.attribution.interactionTarget, // e.g., "button.apply-filters"
    eventType: metric.attribution.interactionType,  // "click", "keydown", etc.
    inputDelay: metric.attribution.inputDelay,
    processingDuration: metric.attribution.processingDuration,
    presentationDelay: metric.attribution.presentationDelay,
  });
});
```

Aggregating this across sessions might reveal: `button.apply-filters` accounts for a disproportionate share of the worst INP samples, with `processingDuration` dominating the breakdown.

**Step 2 — reproduce locally and record a Performance trace clicking that specific element**, ideally under a representative CPU throttle. Confirm the phase breakdown matches the field attribution (processing time dominant, in this example).

**Step 3 — find the expensive synchronous work in the handler.**

```tsx
// BUG: filtering, sorting, and re-rendering a large list synchronously
// inside the click handler, all before the next frame can paint
function FilterPanel({ allProducts }: { allProducts: Product[] }) {
  const [filters, setFilters] = useState(defaultFilters);
  const [results, setResults] = useState(allProducts);

  function handleApplyFilters(newFilters: Filters) {
    setFilters(newFilters);
    const filtered = allProducts
      .filter(p => matchesFilters(p, newFilters))
      .sort(compareByRelevance); // expensive, synchronous, blocks the click's processing time
    setResults(filtered); // triggers a large re-render of the results list
  }

  return <button onClick={() => handleApplyFilters(currentFilters)}>Apply Filters</button>;
}
```

**Step 4 — apply `useTransition` to mark the expensive state update as non-urgent**, letting React deprioritize the resulting re-render relative to any more urgent interaction that might come in, and letting the click handler itself return quickly rather than blocking on the full synchronous computation:

```tsx
function FilterPanel({ allProducts }: { allProducts: Product[] }) {
  const [filters, setFilters] = useState(defaultFilters);
  const [results, setResults] = useState(allProducts);
  const [isPending, startTransition] = useTransition();

  function handleApplyFilters(newFilters: Filters) {
    setFilters(newFilters); // urgent — reflect the selected filter immediately
    startTransition(() => {
      // Marked non-urgent: React can interrupt/deprioritize this work
      // in favor of a more urgent update if one comes in before it finishes
      const filtered = allProducts
        .filter(p => matchesFilters(p, newFilters))
        .sort(compareByRelevance);
      setResults(filtered);
    });
  }

  return (
    <>
      <button onClick={() => handleApplyFilters(currentFilters)}>Apply Filters</button>
      {isPending && <Spinner />}
    </>
  );
}
```

**Step 5 — if the filter/sort computation itself is large enough that even non-urgent main-thread time is a concern, combine with chunking or a Web Worker** (per the previous scenario's techniques) rather than relying on `startTransition` alone — `startTransition` changes *priority*, not total cost; a sufficiently expensive computation still consumes real main-thread time whenever it does run.

**Step 6 — re-measure both in a local trace (phase breakdown improved, particularly processing time no longer blocking the click's immediate handler) and, after deploying, in field RUM data** — confirming the field p75 INP number actually moved, and specifically re-checking attribution to see whether `button.apply-filters` is still the top contributor or whether a *different* interaction has now become the new worst offender (fixing the biggest contributor often reveals the next one, same as with Long Tasks generally).

> **Check yourself:** In Step 4, why does `setFilters(newFilters)` stay outside `startTransition` while the filtering/sorting/`setResults` moves inside it — what would go wrong (from a UX standpoint) if `setFilters` were also wrapped in the transition?

## Gotchas

**Treating INP as something you can fully validate in a lab/synthetic test alone.** INP field aggregation reflects the *worst* interaction across real, varied user sessions and devices — a lab test of one specific interaction, however careful, only samples one scenario; genuine confirmation that a fix worked requires watching the field metric over time after deploying, not just a clean local trace.

**Optimizing the average or most-common interaction while ignoring RUM attribution pointing at a specific worse offender.** Given INP's percentile-based nature, broad "make everything snappier" effort that doesn't specifically target the worst-behaving interaction(s) can leave the actual metric largely unmoved, even if the overall app subjectively feels a bit faster.

**Wrapping *all* state updates in a click handler inside `startTransition`, including ones that need to feel immediate** (like visually reflecting which filter option is now selected) — this can make the UI feel unresponsive in a different way (the selected-state highlight itself lagging), since `startTransition` explicitly deprioritizes the wrapped work; only the genuinely expensive, deferrable part of the update belongs inside it.

**Assuming `startTransition` reduces the total amount of work the computation does.** It changes scheduling priority (letting React interrupt/restart the transition's work if something more urgent arrives), not the computation's actual cost — a `startTransition`-wrapped filter/sort over a very large dataset can still itself become a Long Task if it's expensive enough; for genuinely large computations, this needs to be combined with chunking or worker-offloading, not treated as a substitute for them.

**Missing that presentation delay (the third INP phase) can dominate even when the handler itself is fast and cheap.** A handler that finishes in 5ms but triggers a state update causing an expensive re-render/layout further downstream (a large list re-rendering, a layout-triggering style recalculation across many elements) can still produce a high INP overall — profiling only the handler function's own execution time and missing the downstream render/paint cost it triggers is an incomplete diagnosis.

## Follow-up Questions

**Q (High): Explain the three phases INP is broken into (input delay, processing time, presentation delay) and, for a given 340ms INP sample, how you'd determine which phase is responsible without guessing.**

Answer: Input delay is the time from the user's physical interaction (the click/keypress/tap itself) to the browser actually starting to run the corresponding event handler — a large value here means the main thread was occupied with something *unrelated* to this interaction at the moment it occurred (a scheduled timer callback, an ongoing unrelated render, another long task), so the interaction's own handler had to wait in the queue before it could even begin. Processing time is the duration of the event handler(s) themselves actually executing — a large value here means the handler is doing too much synchronous work directly. Presentation delay is the time after the handler(s) finish until the browser actually paints the resulting visual update to the screen — a large value here means the handler returned promptly, but whatever it triggered (a state update cascading into an expensive re-render, layout recalculation, paint) is what's slow. To determine which phase dominates for a real sample, I'd use the `web-vitals` library's attribution build (as in Step 1 of this scenario), which reports all three phase durations for the actual worst interaction contributing to the field INP score — this gives the breakdown directly from real data rather than requiring it to be inferred or guessed from a local trace alone.

The trap: describing INP as one undifferentiated "interaction speed" number and proposing a fix (usually "reduce the handler's work") without confirming which phase is actually large — a large input delay, for instance, isn't fixed by making the handler itself faster at all, since the handler wasn't even running yet during that time; the fix there is reducing what's competing for the main thread around typical interaction moments.

---

**Q (High): Why does INP use a high percentile of interaction latencies within a session (not an average, and not just the first interaction like the metric it replaced, FID) — what failure mode does this specifically catch that an average or first-interaction-only metric would miss?**

Answer: An average across all interactions in a session can be pulled down (made to look fine) by a large number of trivially fast interactions (simple hovers, cheap toggles) even if a session includes one or two genuinely bad interactions — averaging masks exactly the kind of occasional-but-severe slowness that materially affects real user experience, since users remember and are frustrated by *individual* bad interactions, not a session-wide average. A percentile-based measure specifically surfaces "how bad does it get for this user, at the higher end of what they experienced," which is a better proxy for the frustration a real slow interaction causes than an average would be. FID's "first interaction only" scope missed anything that degrades *after* the first interaction — a page that's fast on initial click but slows down later in a session (as more state/data accumulates, more listeners/components mount, memory pressure builds) would score well on FID despite a real, session-length degradation that INP, evaluating across the whole session, is specifically designed to catch.

The trap: assuming any single-number "how fast is this interaction" metric that isn't a percentile-of-worst-in-session inherently captures the same thing INP does — averaging and first-interaction-only sampling both specifically hide the tail-end, later-session degradation that INP was introduced to surface.

---

**Q (High): `startTransition` wraps an expensive state update, and DevTools no longer shows a Long Task for the associated click. Does this mean the underlying computation got cheaper, and could INP still be bad for this interaction?**

Answer: No — `startTransition` doesn't reduce the total cost of the wrapped computation at all; it changes its *scheduling priority*, letting React's concurrent renderer interrupt (and, if state changes again before the transition finishes, restart/discard) that work in favor of a more urgent update, and it lets React yield to the browser between chunks of the transition's rendering work rather than blocking synchronously start-to-finish. This means the same total amount of main-thread work still has to happen eventually — it's just less likely to *block* the very next urgent interaction, since it's now interruptible and lower-priority rather than one monolithic synchronous block. Whether INP for this interaction actually improves depends on which phase was originally dominant: if the original problem was "this expensive re-render was itself the direct synchronous continuation of the click's handler, so it counted as processing time," `startTransition` can genuinely improve INP by letting the click handler itself finish quickly and moving the expensive part to a deprioritized, interruptible phase; but if the computation itself is large enough to still take a long time even at lower priority — and no *other*, more urgent interaction happens to preempt it — the total wall-clock time to actually see the result can be similar, and if this transition work is what's blocking the *next* frame's presentation (someone waiting to see the result, not a competing interaction), presentation delay for this specific update could still be high, just no longer counted the same way for a *different*, later interaction that got to jump the queue.

The trap: treating "no Long Task flag" as equivalent to "the underlying work is now fast" — `startTransition` is a scheduling/priority tool, and its main benefit is protecting *other, later* interactions from being blocked by this one's cost, not reducing this update's own total cost or guaranteeing this specific update's own presentation is now fast.

---

**Q (Medium): RUM shows INP driven by a `keydown` handler on a search input that runs a synchronous filter over a large list on every keystroke. What are two independent ways to reduce this interaction's INP, and how are they different?**

Answer: (1) Debounce/throttle the actual filtering logic so it doesn't run synchronously on every single `keydown` — e.g., updating the input's displayed value immediately (so typing itself feels responsive) but deferring the expensive filter operation until typing pauses briefly (a short debounce), which reduces how *often* the expensive work runs, directly reducing the number of slow interactions contributing to the INP distribution. (2) For whichever filter operations do still run, apply `startTransition` (or chunk/move to a worker if large enough) so that even when it runs, it doesn't block the very next keystroke's own input delay — reducing the *severity* of each individual slow interaction, independent of how often it happens. These are complementary and address different aspects: debouncing reduces the *frequency* of expensive work (fewer bad samples entering the INP distribution at all), while transition/chunking reduces the *severity* of whichever expensive work does run (each individual bad sample becomes less bad) — a search input under heavy, fast typing benefits from both, since debouncing alone doesn't help the keystroke that does trigger the search after the pause, and priority/chunking alone doesn't reduce how many keystrokes needlessly trigger full synchronous filtering in the first place.

The trap: picking only one of the two and assuming it fully resolves the interaction — debouncing without also addressing the eventual filter's own cost still leaves one expensive, potentially INP-dominating keystroke per debounce window; transition/chunking without debouncing still runs the (now-deprioritized but not eliminated) expensive work on every keystroke, needlessly, when most of those intermediate keystrokes' results are immediately superseded anyway.

---

**Q (Medium): Product says INP is bad but no one has complained. Is that grounds for deprioritizing the fix? How would you reason about it, given the ranking-signal aspect mentioned in the scenario?**

Answer: I wouldn't treat "no complaints" as strong evidence the problem doesn't matter, for two separate reasons: first, users experiencing a laggy interaction very often don't file a complaint about it specifically — they might just quietly bounce, or attribute the sluggishness to their own device/network rather than reporting it as a bug, meaning absence of complaints is weak evidence of absence of impact, especially for a metric that (per the earlier discussion) specifically surfaces tail-end bad experiences that might affect a meaningful minority of sessions without ever generating a support ticket. Second, the scenario explicitly states this affects search ranking (Core Web Vitals are a documented, if modest, ranking factor) — meaning there's a concrete, measurable business cost independent of whether any user has personally complained, similar to how a team might fix a security vulnerability that's never been exploited yet purely because the exposure itself is the risk. I'd prioritize based on the RUM-attributed interaction's actual usage frequency/criticality (a rarely-hit, non-critical interaction driving the score is lower urgency than a core, frequently-used one) rather than on complaint volume, which this scenario's premise already tells us is an unreliable signal here.

The trap: using "no one has complained" as if it were equivalent to "no real user impact exists" — for both UX and product-metric reasons (search ranking), a quantified field metric showing a real problem shouldn't be discounted just because it hasn't yet generated qualitative feedback.

---

**Q (Low): If INP improved substantially in field data after a fix, but Lighthouse's lab score for the same page didn't show a corresponding INP-related improvement, would that be surprising?**

Answer: Not particularly surprising — Lighthouse, as a lab tool, simulates a fixed script of interactions (or, for INP specifically, has more limited/newer support compared to its long-standing LCP/CLS measurement, since INP requires simulating varied real interactions rather than just page-load timing) under a single, fixed device/network profile, whereas the field INP score aggregates across a percentile of genuinely varied real user interactions, devices, and usage patterns. A fix targeted at a specific real-world worst-offender interaction (identified via field RUM attribution, as in this scenario) might not be well-represented by whatever fixed interaction sequence Lighthouse happens to simulate in its lab run, especially if Lighthouse's simulated interaction doesn't specifically exercise the code path that was fixed. This is the same field-vs-lab distinction discussed for LCP earlier in this phase — the two measurement approaches answer related but non-identical questions, and a gap between them, rather than being alarming on its own, is informative about how representative (or not) the lab tool's specific simulated scenario is of the real-world worst-case the fix targeted.

The trap: treating Lighthouse's lab score as the authoritative validation of an INP fix — given INP's field-metric, worst-case, real-interaction-dependent nature, the field data itself (not a lab proxy) is the actual source of truth for whether the fix worked as intended.

---

## Self-Assessment

- [ ] Can define INP's three phases (input delay, processing time, presentation delay) and, for a given sample, explain how to determine which one dominates without guessing
- [ ] Can explain why INP uses a high percentile across a whole session rather than an average or first-interaction-only measurement, and what failure mode that specifically catches
- [ ] Can explain precisely what `startTransition` does and does not change (scheduling priority, not total computation cost)
- [ ] Can propose two independent, complementary fixes (reducing frequency vs. reducing per-call severity) for an expensive handler triggered on every keystroke
- [ ] Can reason about why a field-measured metric with a business impact (ranking) shouldn't be deprioritized just because of an absence of user complaints

---
*This closes Phase 5 — Performance Debugging Scenarios. Phase 6, State Management & Architecture Trade-offs, shifts from "diagnose and fix a performance symptom" to "design where state should live and how it should flow" — a different kind of judgment, less about profiling tools and more about long-term maintainability trade-offs.*
