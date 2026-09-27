# Feature-flag-driven Rollout Design

## Quick Reference

| Concern | Mechanism | Why |
|---|---|---|
| Where flag state lives | A flag service/SDK evaluated per-user (or a config object fetched once) — not scattered `localStorage` booleans | Central control, consistent targeting rules, remote toggle without a redeploy |
| Where flag *checks* go in code | As close to the decision point as reasonable, behind a named hook (`useFeatureFlag`) — never duplicated inline conditionals | One source of truth for the check; easy to grep and remove later |
| Gradual rollout | Percentage-based or attribute-based targeting (internal users → beta cohort → percentage ramp → 100%) | Bounds blast radius of a bad change to a shrinking population as confidence grows |
| Kill switch | The flag itself, flippable without a deploy | Fastest possible mitigation if something goes wrong in production |
| Flag lifecycle | Explicit removal plan — flag + old code path deleted once fully rolled out | Long-lived flags are a common, real source of codebase complexity and untested code-path combinations |

## The Scenario

"We're shipping a redesigned checkout flow. It touches a critical, high-revenue path, so leadership wants a careful, gradual rollout with the ability to instantly revert if anything looks wrong — not a big-bang deploy. Design the feature-flag strategy for this: how the flag is structured, how the rollout ramps, what telemetry backs the decision to keep ramping or roll back, and how the flag eventually gets cleaned up."

## Clarifying Questions

- **What's the actual flag-evaluation infrastructure already in place — a third-party service (LaunchDarkly, Split, GrowthBook), an in-house config system, or nothing yet?** This changes the concrete mechanics significantly (targeting rules, percentage rollout, and real-time flag updates without a redeploy are usually built into a dedicated flagging service; building this in-house from scratch is a meaningfully bigger scope than using an existing one) — I'd want to know before designing specifics, though the rollout *strategy* (targeting, ramp stages, kill-switch behavior, cleanup) is largely infrastructure-agnostic.
- **Should the flag be evaluated client-side, server-side, or both — and does checkout involve any server-side logic (pricing, payment processing) that also needs to branch on the same flag consistently with the client?** A checkout redesign is unlikely to be purely a UI change — if the new flow calls different backend endpoints or triggers different payment logic, the flag decision needs to be consistent between client and server for a given user/session (a user shouldn't see the new UI but hit old backend logic, or vice versa), which is a materially harder consistency problem than a pure client-side UI flag.
- **What telemetry already exists on the current checkout flow (conversion rate, error rate, time-to-complete, cart abandonment at each step) that the new flow's success/failure would be measured against?** Without an existing baseline, "look at the metrics and decide whether to keep ramping" isn't actionable — I'd want to confirm what's already instrumented versus what needs to be added specifically to support this rollout's decision-making, since checkout-specific step-by-step funnel metrics might not already exist at the granularity needed.
- **Is this an A/B test (deliberately comparing old vs. new with statistical rigor, potentially running both indefinitely for measurement) or a rollout (intending to fully replace the old flow once confidence is established)?** These have different endpoints — a rollout has a clear finish line (100%, then delete the flag and old code); an A/B test might deliberately hold a control group for a longer, statistically-determined period, and "when do we clean up the flag" has a different answer for each.
- **Who has the authority to pull the kill switch, and what's the actual mechanism/latency for doing so — is it a self-service toggle any on-call engineer can flip, or does it require a deploy or a request to another team?** For a rollout explicitly motivated by "ability to instantly revert," this needs to actually be instant and self-service for whoever's on point during the rollout window, not theoretically possible but practically slow.

## Approach & Trade-offs

**The flag should default to evaluating client-and-server consistently for the same user, using a stable identity key (user ID, or a stable anonymous ID for logged-out checkout), rather than being a purely client-side UI toggle — because a checkout redesign almost never stays confined to presentation, and letting the client and server disagree about which flow a given user is in is a correctness risk, not just an inconsistency.** Concretely: whichever system evaluates the flag first in a request's lifecycle (commonly the server, since it can be the authoritative source and pass the resolved value down to the client rather than each independently calling out to the flag service) should be the single point of truth for that user's session, avoiding a scenario where the client renders the new checkout UI but the server processes the submission with old-flow assumptions (or the reverse), which could produce subtly wrong pricing, order records, or payment behavior — the kind of bug that's hard to detect from a health dashboard and shows up as individual confused support tickets instead.

**The ramp should be staged by *risk-bounded population*, not by an arbitrary percentage schedule picked up front — starting with populations where a bug is cheapest to have happen and easiest to detect, and only widening once each stage clears a defined bar.** A reasonable staging: internal employees/dogfooding first (bugs are embarrassing but low-stakes, and internal users are more likely to report oddities proactively rather than silently abandoning their cart and never telling anyone), then a small percentage of real users chosen to minimize correlated risk (not literally the first 1% by signup order, which could inadvertently cluster by signup cohort/region in a way that biases the sample), then a larger percentage, then full rollout — with an explicit, pre-agreed bar for what "clears this stage" means (a conversion-rate delta within an acceptable band, an error rate not exceeding some threshold, no new categories of support tickets), decided *before* the rollout starts, not improvised in the moment when the data is already in front of people who might be motivated to see it favorably.

**Telemetry needs to be checkout-funnel-shaped, not just "is the site up" — meaning step-by-step conversion/drop-off/error tracking through the new flow specifically, segmented by flag variant, compared against the old flow's equivalent funnel for the same time period and population characteristics.** A single aggregate "conversion rate" comparison is a weak signal on its own for a multi-step flow — the new flow could have an equivalent *overall* conversion rate while actually having a much worse experience at one specific step that's being compensated for by an improvement at another, and knowing *which* step regressed (if any) is what actually lets someone decide "fix this one thing and re-ramp" versus "roll back entirely." I'd also want error-rate and latency telemetry specifically for any new backend paths introduced, since a checkout redesign's riskiest failure mode (a payment processing error, an order not being created correctly) is exactly the kind of thing that might not show up in a conversion-funnel metric at all if it fails in a way that still ultimately shows *some* completion state, just an incorrect one.

**The kill switch needs to be tested *before* it's needed, and its scope should default to "instantly reverts to the old flow for the affected population" rather than requiring a redeploy or a slow config change — and I'd deliberately verify this by using it at least once during a low-risk moment (e.g., flipping it back to 0% at the end of the internal-dogfooding stage as a matter of routine, not just in an emergency) so its actual latency and correctness are known quantities, not an assumption resting on the flagging infrastructure's marketing claims.** This matters because the worst possible time to discover a kill switch doesn't actually work as fast/cleanly as expected is during a live incident under pressure — exercising it during a deliberate, low-stakes moment earlier in the rollout builds real confidence in the mechanism the team is depending on for the higher-stakes stages.

**The flag's lifecycle needs an explicit end state decided at the start: once the rollout reaches 100% and holds there for a defined soak period with no regressions, the flag and the old code path are deleted — not left in the codebase indefinitely "just in case."** A long-lived flag that's forgotten at 100% (or, worse, left at 100% with the old code path never removed) is a common, underappreciated source of codebase complexity: every future change to checkout now has to reason about (or at least not accidentally break) a code path that's supposedly dead but is still compiled and present, and QA/testing surface area silently doubles for a decision that's already effectively been made. I'd put "delete the flag" as a tracked, owned follow-up task created at the same time the flag is created, not an afterthought discovered months later during an unrelated cleanup effort.

## Solution — the rollout design

**1. Flag evaluation — server as the source of truth, passed to the client, keyed by a stable identity:**

```ts
// Server-side, resolved once per session/request and included in the initial payload
function resolveCheckoutFlag(userId: string | null, sessionId: string): 'legacy' | 'redesign' {
  return flagService.evaluate('checkout-redesign', {
    key: userId ?? sessionId, // stable across the session even for logged-out checkout
  });
}

// Client receives the already-resolved value — never independently re-evaluates
// against a potentially different targeting outcome
function CheckoutPage({ checkoutVariant }: { checkoutVariant: 'legacy' | 'redesign' }) {
  return checkoutVariant === 'redesign' ? <CheckoutRedesign /> : <CheckoutLegacy />;
}
```

**2. Staged rollout configuration, defined up front with explicit promotion criteria:**

```
Stage 0 — internal employees only (targeting rule: email domain == company domain)
  Promote when: 3 business days with zero P1/P2 bugs filed against the new flow

Stage 1 — 5% of real users, randomly sampled by stable hashed user ID
  Promote when: funnel conversion within 1pp of legacy control, error rate not
  elevated vs. legacy, for a full 7-day window (covers weekly usage-pattern variance)

Stage 2 — 25% of real users
  Promote when: same bar as Stage 1, sustained for 5 days

Stage 3 — 100%
  Hold for 14-day soak period with the old flow's code path still present
  but unreachable, before flag/old-code removal is scheduled
```

**3. Funnel telemetry, segmented by variant, at each meaningful step:**

```ts
function trackCheckoutStep(step: 'cart_reviewed' | 'shipping_entered' | 'payment_submitted' | 'order_confirmed') {
  analytics.track('checkout_funnel_step', {
    step,
    variant: getCurrentCheckoutVariant(), // so each stage's data segments cleanly by flag value
    sessionId,
  });
}
```

**4. A deliberately exercised kill switch, and a tracked cleanup task created at flag-creation time:**

```ts
// Flipping the flag's rollout percentage to 0 is a config change in the flag
// service's dashboard/API — no deploy required. Exercised intentionally at
// the end of Stage 0 as a dry run, not only reserved for a real incident.

// Tracked alongside the flag's creation, not as an afterthought:
// TICKET-1234: "Remove `checkout-redesign` flag and legacy checkout code
// path once Stage 3 soak period completes with no regressions."
```

> **Check yourself:** Why should the server resolve the flag and hand the value to the client, rather than having the client independently call the flag service on its own — what specific inconsistency does this avoid?

## Gotchas

**Treating the flag as purely a client-side UI toggle when the redesign also changes backend behavior.** Produces a real risk of client/server disagreeing about which flow a given user is in, which for checkout specifically can mean incorrect pricing, order, or payment processing behavior — not just a visual inconsistency.

**Ramping by a fixed percentage schedule decided without a real promotion bar, then "eyeballing the dashboard" to decide whether to continue.** Without pre-agreed criteria, the decision to keep ramping (or not) becomes vulnerable to motivated reasoning in the moment, especially under pressure to ship — deciding the bar up front, before data exists, is what keeps the decision honest.

**Comparing only an aggregate conversion rate between old and new flows, missing a step-specific regression that's offset by an improvement elsewhere.** A flat overall conversion rate can hide a materially worse experience at one specific step, which is exactly the kind of thing a step-by-step funnel comparison is needed to catch.

**Never actually testing the kill switch until an emergency, discovering only then that it's slower or less complete than assumed** (e.g., it stops showing the new UI to new sessions but doesn't affect already-in-progress sessions, or takes longer to propagate than expected). Exercising it deliberately during a low-stakes stage builds real, tested confidence rather than an assumption.

**No explicit plan or owner for removing the flag and old code path once fully rolled out.** The single most common way flags become permanent, unremovable complexity — "it's been at 100% for a year, nobody wants to be the one to touch it and risk breaking something" — is exactly what an explicit cleanup task created at flag-creation time is meant to prevent.

## Follow-up Questions

**Q (High): Why should the server resolve the flag and pass the value down, rather than the client independently calling the flag service to evaluate the same flag?**

Answer: If both the client and server independently evaluate the same flag, there's a real possibility they disagree for a single user/session — not necessarily because of a bug in the flag service itself, but because of timing (the flag's rollout percentage or targeting rules changed between the two evaluations, which could happen microseconds apart during an active rollout ramp), or because the two evaluations use slightly different identity keys or context. For most features, a brief client/server disagreement about a flag is a minor cosmetic inconsistency; for checkout specifically, it could mean the UI renders one flow's assumptions while the backend processes the submission under the other flow's logic — genuinely incorrect behavior (wrong pricing display, a payment request shaped for the wrong flow), not just visual inconsistency. Resolving once, server-side, and passing that resolved value to the client as part of the initial request/response removes the possibility of disagreement entirely, at the cost of the client no longer being able to independently re-evaluate the flag mid-session (which is rarely needed anyway, and arguably undesirable for checkout — a flow shouldn't switch flag variants mid-way through a user's checkout session even if the flag's rollout percentage changes while they're actively checking out).

The trap: treating flag evaluation as a pure client-side UI concern, without considering that a feature touching both client and server behavior needs a single, consistent evaluation shared between them — independent evaluation on each side is a subtle but real correctness risk specifically for flags gating cross-cutting behavior.

---

**Q (High): Stage 1 (5% of real users) shows funnel conversion within the acceptable band, but customer support reports a small but real uptick in tickets about "my order total looked wrong for a second before checkout completed." The dashboards look fine. Do you promote to Stage 2?**

Answer: No — I'd treat this as a stop-and-investigate signal that overrides the dashboard-based promotion criteria, precisely because it's the kind of issue that's plausible to not show up cleanly in an aggregate funnel/error-rate metric (a transient, self-correcting visual glitch wouldn't necessarily register as a funnel drop-off or a hard error if the user proceeds anyway) but represents a real, concerning correctness signal for a redesigned checkout flow specifically. This is exactly why the promotion criteria shouldn't be treated as a purely mechanical "if metrics are within band, auto-promote" rule — support tickets, qualitative signals, and anything suggesting a *correctness* issue (versus a pure conversion/performance issue) for a payment-adjacent flow warrant a deliberate pause and investigation regardless of what the quantitative dashboard shows, given how much more costly a checkout correctness bug is than a checkout conversion dip.

The trap: treating the pre-agreed quantitative promotion bar as sufficient on its own and auto-promoting because "the numbers say we're within band" — the pre-agreed criteria are a floor for the decision, not a ceiling that overrides a concerning qualitative signal the dashboards weren't necessarily designed to catch.

---

**Q (High): Six months after full rollout, someone discovers the `checkout-redesign` flag and legacy code path are both still in the codebase, and nobody's sure if it's actually safe to delete the old path. How did the process break down, and how would you prevent this?**

Answer: The breakdown is almost certainly that the cleanup task, even if created, wasn't tracked with enough visibility/ownership to actually get prioritized against ongoing feature work, or it was never created as an explicit, owned ticket in the first place and relied on someone remembering informally. The prevention is process, not tooling: create the cleanup ticket at the *same moment* the flag is created (not after rollout completes, when the urgency that motivated the whole rollout has faded and it competes for priority against newer work), assign it a concrete trigger condition decided up front ("N days at 100% with no regressions" — defined before anyone has an incentive to move the goalposts), and ideally have it show up in some recurring review (a periodic "flags at 100% for over 30 days" report, if the flagging infrastructure supports it) so stale flags surface proactively rather than requiring someone to stumble onto them by accident.

The trap: assuming "we'll clean it up later" is a sufficient plan without a concrete trigger, owner, and tracking mechanism — this is one of the most common and predictable failure modes of flag-driven rollouts, and a strong answer treats prevention as a process design question decided at flag creation, not a discipline problem to solve after the fact.

---

**Q (Medium): How would you choose the specific 5% of users for Stage 1, and why does it matter how they're selected?**

Answer: I'd use a stable hash of a consistent user identifier (user ID, or a persistent anonymous ID for logged-out users) mapped into a bucket range, rather than any selection method correlated with something that could bias the sample — e.g., not literally "the first 5% of users to check out today," which could inadvertently skew toward a particular timezone/region's active hours, or a particular user segment that happens to be more active early in a rollout window. A stable hash-based assignment also has a useful property beyond unbiased sampling: the *same* user consistently lands in the same bucket across their session (and across the whole rollout, unless the percentage or targeting rules change), avoiding a jarring experience where a user sees the new flow on one visit and the old flow on the next due to re-randomization.

The trap: picking a selection method that's simple to implement but introduces sampling bias (time-of-day, request-order-based selection) without noticing, which can make Stage 1's results look better or worse than they'd be for a truly representative population — undermining the whole point of a staged, measured rollout.

---

**Q (Medium): Does this design change if the redesign is being run as a genuine, statistically-rigorous A/B test rather than a rollout intending to fully replace the old flow?**

Answer: The staging/ramp mechanics stay largely similar (still want to bound risk before scaling exposure), but the endpoint and some of the rigor around sample size and duration change meaningfully — an A/B test needs a pre-registered hypothesis and a calculated required sample size/duration to reach statistical significance for the specific metric being compared (not just "looks fine, let's promote"), and critically, needs the *control group to be deliberately maintained* at a stable size for the test's full duration rather than being ramped down to 0% as confidence grows, since a shrinking control group mid-test undermines the statistical comparison. The flag's eventual cleanup also looks different: rather than "ramp to 100% and delete the old path," the natural endpoint is "the test concludes, a decision is made based on the results, and *then* the flag ramps to whichever variant won and the losing path is deleted" — the flag exists for the test's duration as a genuine 50/50-ish (or whatever ratio the test design calls for) split, not a one-directional ramp.

The trap: applying rollout-style "ramp up as confidence grows, kill the flag once at 100%" thinking to what's actually meant to be a fixed-duration, fixed-allocation statistical comparison — conflating the two undermines the A/B test's validity by changing group sizes mid-experiment.

---

**Q (Low): Is there a case where you'd skip the internal-employees dogfooding stage entirely?**

Answer: Possibly, if the change genuinely can't be meaningfully exercised by internal employees at all (e.g., it depends on a real payment method/region/user-segment-specific behavior that internal test accounts can't authentically represent) — in that narrow case, dogfooding provides limited signal and Stage 1's small real-user percentage becomes the actual first meaningful signal. I wouldn't skip it merely to move faster for a change with genuine internal-testability, though, since dogfooding is close to free (no real user risk) and reliably catches at least some category of obvious bugs (broken layouts, crashes, obviously wrong copy) before any real user is exposed to them — skipping a free, low-risk validation stage to save a few days on a high-stakes checkout change is a poor trade even under schedule pressure.

The trap: skipping the cheap, low-risk validation stage under time pressure specifically on the highest-stakes rollout, where the cost of an avoidable bug reaching real users (even a small percentage) is much higher than the few days dogfooding would have taken.

---

## Self-Assessment

- [ ] Can explain why a checkout-flag rollout needs client/server-consistent flag evaluation, not independent client-side evaluation
- [ ] Can design a staged rollout with explicit, pre-agreed promotion criteria rather than an improvised percentage schedule
- [ ] Can explain why step-by-step funnel telemetry beats a single aggregate conversion metric for deciding whether to keep ramping
- [ ] Can articulate why the kill switch should be deliberately exercised before it's needed in an emergency
- [ ] Can describe a concrete mechanism for ensuring a flag actually gets cleaned up after full rollout, not left indefinitely
- [ ] Can distinguish a rollout (ramp-to-100%-then-delete) from an A/B test (fixed-duration, fixed-allocation) and explain why their flag lifecycles differ

---
*Next: Phase 7 — Networking & Data Layer Scenarios. Where Phase 6 focused on classifying and architecting client-side state, Phase 7 goes deeper into the data-fetching layer itself — parallelization, caching strategy, pagination models, and resilience under real network conditions.*
