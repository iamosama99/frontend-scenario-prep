# Graceful Degradation for a Flaky Third-party API

## Quick Reference

| Failure Mode | Defense | Why |
|---|---|---|
| Total outage (all requests fail) | Fallback UI / cached last-known-good data, not a blocked page | The rest of the app shouldn't die because one dependency did |
| Slow responses (works, but very late) | Timeout + fallback, decoupled from the rest of the page's load | An unbounded wait on one dependency blocks perceived load for everything else on the page |
| Intermittent failures (works sometimes) | Retry with backoff, capped attempts, only for idempotent requests | Distinguishes "transient blip, worth retrying" from "genuinely broken, stop wasting time" |
| Cascading failure (this API's failure breaks unrelated features) | Circuit breaker — stop calling a consistently-failing dependency for a cooldown window | Protects the rest of the app from wasting resources on calls very likely to fail anyway |
| Silent degraded data (wrong, not absent) | Explicit "unavailable" state, never a silently stale/wrong value presented as current | Wrong data with no signal is worse than visibly absent data |

## The Scenario

"We show real-time shipping estimates on the product page, sourced from a third-party logistics API. That API is unreliable — sometimes it's slow (8+ seconds), sometimes it returns errors, and once a month it goes down entirely for 20-30 minutes. Right now when it's slow or down, the whole product page hangs waiting for it, or shows a broken-looking blank space where the estimate should be. Fix this so the product page always loads and functions normally, with shipping estimates degrading gracefully instead of taking the whole page down with them."

## Clarifying Questions

- **Is the shipping-estimate call currently blocking the page's initial render/load, or does it happen after the rest of the page has already rendered?** If it's part of a server-side render or a blocking data-fetch that gates the whole page, that's an architectural coupling problem (an unrelated, optional feature is on the critical path for everything) that needs fixing regardless of the third-party API's reliability; if it's already fetched independently post-render, the fix is more narrowly scoped to that one component's failure/loading handling.
- **What's an acceptable "good enough" fallback when the real-time estimate isn't available** — a generic static estimate ("Usually ships in 3-5 business days"), a cached previous successful response (even if a few hours stale), or simply omitting the estimate section entirely? This is a product decision as much as an engineering one, and changes what the fallback UI needs to be able to source from (does the app need to start persisting the last successful response per product, or is a hardcoded generic message sufficient).
- **Does a failure here have any correctness/business consequence beyond a degraded page** (e.g., is the shipping estimate also used to compute a price, a delivery-date guarantee shown at checkout, or eligibility for a promotion), or is it purely informational? If it's purely informational, graceful degradation is straightforward (show a fallback, move on); if downstream logic depends on this value's correctness, a fallback needs to distinguish "informational display" from "input for a decision" and handle the latter more conservatively (e.g., don't guarantee a delivery date the app doesn't actually know).
- **Is "once a month, down for 20-30 minutes" a known, monitored pattern (their status page, a known maintenance window) or a genuinely unpredictable outage?** Doesn't change the client-side resilience design much, but it's worth asking since a known pattern might justify a coordinated response (proactively switching to fallback mode during a known maintenance window rather than waiting to detect the failure reactively).
- **Is this the only place in the app calling this third-party API, or are there other features depending on it too** — and if the latter, is there a shared client/service layer already, or does each feature call it independently? Determines whether the resilience patterns (timeout, retry, circuit breaker) need to be built once in a shared layer (correct answer if there's any reuse) or are being designed for a single, isolated call site.

## Approach & Trade-offs

**Structure the fix around four independent resilience mechanisms, because "graceful degradation" isn't one technique — it's a combination that each address a different failure mode this exact API exhibits (slow, erroring, and totally down are three different problems).** A timeout addresses slowness specifically (bound the wait, regardless of whether the request would have eventually succeeded). Retry-with-backoff addresses intermittent failure specifically (a transient blip is often worth one or two quick retries; a persistent failure isn't worth retrying indefinitely). A circuit breaker addresses sustained outage specifically (stop even attempting calls to a dependency that's currently, demonstrably down, rather than repeatedly timing out or erroring on every single page load during a 20-30 minute outage). A fallback UI addresses the user-facing consequence of any of the above kicking in (something reasonable to show regardless of which failure mode is currently active). Treating any single one of these as "the fix" leaves the others' specific failure mode unaddressed.

**The shipping estimate must never be allowed to block the rest of the page — this is the single highest-leverage architectural fix, independent of anything about the third-party API's own behavior.** An optional, supplementary piece of information (shipping estimate) being able to hang or fail the entire product page (price, images, add-to-cart, description — everything) is a coupling bug regardless of how reliable the third-party API is; even a perfectly reliable API taking 300ms shouldn't be allowed to gate the rest of a page that doesn't structurally depend on it. The fix here connects to the general data-fetching pattern from [[01-data-fetching-layer-for-related-resources]]: this is an independent resource, fetched in parallel with (not sequentially before) everything else the page needs, and rendered progressively — the page renders fully functional without it, with the shipping-estimate section rendering its own loading/fallback/success state independently as that one request resolves, fails, or times out.

**Timeouts need to be tuned to the actual UX requirement, not left at whatever default the HTTP client happens to use (often none, or a very long one).** "Slow sometimes (8+ seconds)" is itself the bug from the user's perspective — nobody should wait 8 seconds for a shipping estimate on a product page. Setting an aggressive, product-appropriate timeout (e.g., 2-3 seconds) and falling back immediately past that point trades "always show the real number, however long it takes" for "usually show the real number quickly, sometimes show a reasonable fallback instead" — the right trade-off for a supplementary, non-critical piece of information, and a different one than I'd make for something the user is actively blocked on (e.g., a payment confirmation).

**Retries need to be scoped carefully — not every failure is worth retrying, and retrying the wrong kind of request is actively harmful.** A network blip or a `5xx` from the third-party API is often transient and worth one or two quick retries (ideally with a short backoff, not immediate, to avoid hammering an already-struggling service). A `4xx` (bad request, invalid product ID) is not transient — retrying it wastes time and won't succeed on attempt two any more than attempt one. And requests need to be idempotent to retry safely at all — a read-only "get shipping estimate" call is safe to retry; if this same resilience pattern were being reused for a non-idempotent write (rare for this specific feature, but worth stating as a general principle), blind retries could cause duplicate side effects.

**A circuit breaker is what actually addresses the stated "goes down entirely for 20-30 minutes" pattern efficiently — without one, every single page load during that window pays the full timeout-then-fallback cost redundantly.** Without a circuit breaker, if the API is down for 25 minutes, every product-page view during that window independently discovers the outage the slow way (waits out the timeout, or retries and waits out multiple timeouts, before falling back) — correct in outcome, but wasteful, and it means every single user during the outage experiences at least the full timeout delay before seeing the fallback, even though the app "knows" (from the previous request 30 seconds ago) that this dependency is currently down. A circuit breaker tracks recent failure rate for calls to this dependency and, once it crosses a threshold, "opens" — subsequent calls skip attempting the network request entirely and go straight to the fallback, for a cooldown period, after which it allows a trial request through to check if the dependency has recovered ("half-open" state) before fully closing again. This turns "every user pays the timeout cost during an outage" into "the first few users pay it, then everyone else gets the fallback instantly until it recovers."

## Solution

**Step 1 — decouple the shipping-estimate fetch from the rest of the page entirely; it's rendered as its own independently-loading section, never gating page render.**

```tsx
function ProductPage({ productId }: { productId: string }) {
  return (
    <Layout>
      <ProductImages productId={productId} />
      <ProductDetails productId={productId} />
      <AddToCartButton productId={productId} />
      <ShippingEstimate productId={productId} /> {/* owns its own loading/error/fallback state */}
    </Layout>
  );
}
```

**Step 2 — the call itself, wrapped with a timeout and limited, backed-off retries for idempotent failures:**

```ts
async function fetchShippingEstimate(productId: string, opts: { timeoutMs: number; maxRetries: number }) {
  for (let attempt = 0; attempt <= opts.maxRetries; attempt++) {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), opts.timeoutMs);
    try {
      const res = await fetch(`/api/shipping-estimate/${productId}`, { signal: controller.signal });
      clearTimeout(timeout);
      if (res.status >= 500) throw new RetriableError(`Upstream ${res.status}`);
      if (!res.ok) throw new NonRetriableError(`Upstream ${res.status}`); // e.g. 4xx — don't retry
      return await res.json();
    } catch (err) {
      clearTimeout(timeout);
      const isLastAttempt = attempt === opts.maxRetries;
      if (err instanceof NonRetriableError || isLastAttempt) throw err;
      await sleep(200 * 2 ** attempt); // short backoff between retries
    }
  }
}
```

**Step 3 — a circuit breaker wrapping calls to this dependency, addressing the sustained-outage case efficiently:**

```ts
class CircuitBreaker {
  private failureCount = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';
  private openedAt = 0;
  private readonly failureThreshold = 5;
  private readonly cooldownMs = 30_000;

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.openedAt < this.cooldownMs) {
        throw new CircuitOpenError(); // skip the network call entirely — known-down
      }
      this.state = 'half-open'; // cooldown elapsed, allow one trial request through
    }
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  private onSuccess() {
    this.failureCount = 0;
    this.state = 'closed';
  }
  private onFailure() {
    this.failureCount += 1;
    if (this.failureCount >= this.failureThreshold || this.state === 'half-open') {
      this.state = 'open';
      this.openedAt = Date.now();
    }
  }
}

const shippingEstimateBreaker = new CircuitBreaker(); // module-level, shared across all calls
```

While the breaker is `open`, calls fail immediately with `CircuitOpenError` — no network request attempted, no timeout paid — which is what turns a 25-minute outage from "every page load pays the full timeout+retry cost" into "the first handful of failures open the breaker, then every subsequent page load gets the fallback instantly for the rest of the outage."

**Step 4 — the component composes all of this into an explicit state machine, always rendering *something* reasonable:**

```tsx
function ShippingEstimate({ productId }: { productId: string }) {
  const [state, setState] = useState<'loading' | 'success' | 'fallback'>('loading');
  const [estimate, setEstimate] = useState<Estimate | null>(null);

  useEffect(() => {
    let cancelled = false;
    shippingEstimateBreaker
      .call(() => fetchShippingEstimate(productId, { timeoutMs: 2500, maxRetries: 1 }))
      .then((data) => {
        if (!cancelled) { setEstimate(data); setState('success'); }
      })
      .catch(() => {
        if (!cancelled) setState('fallback'); // covers timeout, retry exhaustion, AND open-circuit
      });
    return () => { cancelled = true; };
  }, [productId]);

  if (state === 'loading') return <ShippingEstimateSkeleton />;
  if (state === 'success') return <RealEstimate estimate={estimate!} />;
  return <GenericEstimate />; // "Usually ships in 3-5 business days" — never a blank space, never an error page
}
```

Every failure path — slow-then-timed-out, errored-then-retries-exhausted, or circuit-already-open — converges on the same `fallback` UI state; the rest of the page never knows or cares which one occurred.

> **Check yourself:** During the circuit breaker's `half-open` trial request, if that single trial also fails, what should happen — and why does the breaker specifically go back to `open` (not stay `half-open` and keep retrying) rather than trying a few more times before giving up again?

## Gotchas

**Building retry logic without excluding non-idempotent or clearly-non-transient failures (4xx errors)**, wasting time retrying a request that will deterministically fail again, and in the worst case (if this pattern gets reused for a write, not this specific read-only feature), risking duplicate side effects from retrying a non-idempotent call.

**Setting a circuit breaker's failure threshold too low relative to normal, expected transient failure rates**, causing it to trip (and start serving fallback data) during ordinary, brief blips rather than genuine sustained outages — a circuit breaker tuned too aggressively degrades UX during periods the API would have actually been fine to keep calling.

**No cooldown/half-open recovery check — once open, staying open until a deploy or manual intervention.** A circuit breaker without a mechanism to periodically test whether the dependency has recovered means the app keeps serving fallback data indefinitely even after the third-party API is back up, which defeats a large part of the point (getting back to real data as soon as it's genuinely available again).

**Treating "the fetch succeeded" as sufficient, without validating the response shape** — a third-party API that's degraded rather than fully down can sometimes return a `200` with an unexpected, malformed, or partial payload; code that only checks `res.ok` and doesn't defensively validate the response body before treating it as a trustworthy estimate can end up displaying garbage data as if it were a real, confident shipping estimate — worse than a clearly-labeled fallback.

**Letting the shipping-estimate section's error state leak into a broken visual (a raw error message, a broken layout, an infinite spinner) instead of a designed fallback UI** — "graceful" degradation specifically means the fallback state looks intentional and reasonable, not like something broke; a technically-working timeout/retry/circuit-breaker system that still renders an ugly, alarming failure state hasn't actually delivered the UX goal stated in the prompt.

**Applying the same aggressive timeout/fallback treatment to a value the app's own logic depends on downstream** (per the clarifying question about correctness consequences) — if this estimate secretly also determined delivery-date guarantees shown at checkout, silently falling back to a generic "3-5 days" without that fallback being reflected consistently everywhere the value is used could create a mismatch between what was shown on the product page and what's promised at checkout.

## Follow-up Questions

**Q (High): Explain precisely what problem a circuit breaker solves that a timeout + retry combination, on its own, doesn't — using the "down for 20-30 minutes" detail from the scenario specifically.**

Answer: Timeout and retry are both *per-request* mechanisms — they bound how long any single request is allowed to take and give any single request a couple of extra chances to succeed, but they have no memory across requests; every new page load, with no awareness that the exact same dependency failed identically 10 seconds ago for the previous visitor, independently pays the full timeout-then-retry-then-fallback cost from scratch. During a genuine 20-30 minute outage, that means every single user who loads the product page during that window individually experiences the full multi-second timeout (plus however many retry attempts and their backoff delays) before finally seeing the fallback — correct in outcome, but a materially worse experience than necessary, and needlessly wasteful of client and server resources (every single page load fires a doomed request/retries against a dependency that's currently, demonstrably down). A circuit breaker adds cross-request memory: after enough recent failures cross a threshold, it "opens" and every subsequent call *skips the network request entirely* for a cooldown period, going straight to the fallback with no delay — turning "every user during the outage pays the full timeout cost" into "roughly the first handful of requests pay it (enough to trip the breaker), then everyone else for the rest of the outage gets the fallback instantly."

The trap: describing a circuit breaker as "another layer of retry protection" or conflating its purpose with timeout/retry — its distinguishing property is that it prevents *future* requests from even attempting a call once a dependency's current unhealthiness has been established, which per-request mechanisms structurally can't do since they have no memory of prior requests' outcomes.

---

**Q (High): Why should the shipping-estimate fetch be architecturally decoupled from the rest of the product page's data-loading, independent of anything about the third-party API's reliability? What would still be wrong even if that API had 100% uptime and always responded instantly?**

Answer: Even a perfectly reliable, instant third-party API shouldn't be allowed to gate the rest of the page's render if the shipping estimate is genuinely optional/supplementary information — coupling an unrelated feature's data-fetch to the critical render path of core page content (price, images, add-to-cart) is a resilience anti-pattern independent of that specific dependency's actual reliability, because it means *any* future degradation of that one feature (a slow deploy on the third-party's end, a network partition specific to that one endpoint, anything) automatically becomes a degradation of the entire page, for a piece of functionality that didn't need to be on the critical path in the first place. This is the same principle as the parallel-vs-sequential data-fetching design in [[01-data-fetching-layer-for-related-resources]]: an optional, independently-renderable resource should be fetched and rendered on its own track, not folded into a shared "the page isn't ready until everything is ready" gate — the architectural decoupling is valuable on its own merits (isolating blast radius, enabling independent loading states), and it's what makes the resilience patterns (timeout, retry, circuit breaker) actually *sufficient* rather than just mitigating a coupling problem that's still there underneath.

The trap: presenting the fix as entirely about handling the third-party API's specific failure modes (timeout/retry/circuit breaker) without naming the separate, more fundamental architectural issue — an unreliable dependency being on the critical path is a coupling bug that a hypothetically perfectly-reliable dependency would still exhibit differently (e.g., the page being needlessly slower than it needs to be, since it's now bottlenecked by a feature that didn't need to block anything).

---

**Q (High): The circuit breaker's `half-open` trial request also fails. Walk through exactly what the breaker should do next, and explain why immediately reopening (rather than trying a few more times before giving up) is the correct behavior.**

Answer: When the cooldown period elapses, the breaker transitions to `half-open` and allows exactly one (or a small, deliberately limited number of) trial request(s) through as a live health check of the dependency, without yet fully trusting it recovered. If that trial fails, the breaker should immediately transition back to `open` and restart its cooldown timer, rather than staying in a more lenient "let's try a few more times" state — the reasoning is that a `half-open` trial failing is fresh, current evidence that the dependency is still unhealthy, and giving it several more immediate chances would reintroduce exactly the problem the breaker exists to prevent (repeatedly hammering a dependency that's still down, paying the cost of those attempts on behalf of whichever user's request happened to trigger them). Going back to `open` and waiting out another full cooldown period before trying again treats "still failing" with the same skepticism as the original trip, and only fully returns to `closed` (normal operation, no gating) once a trial request actually succeeds — at which point the failure count resets and calls flow through normally until/unless the failure threshold is crossed again in the future.

The trap: designing `half-open` to allow a burst of several retries before deciding whether to fully close or reopen — this reintroduces the exact "hammer a struggling dependency" problem the breaker is meant to prevent, defeating the purpose of gating recovery checks behind a single (or minimal) trial request in the first place.

---

**Q (Medium): The fallback shown when the real estimate is unavailable is a generic, static message ("Usually ships in 3-5 business days"). What are the risks of this specific choice, and when would a cached last-known-good response be a better fallback?**

Answer: A generic static fallback risks being meaningfully wrong for products where the real answer diverges a lot from the generic default — a product that's actually backordered, or ships from a specific warehouse with genuinely different (often longer) lead times, would show a falsely reassuring "3-5 business days" during a degradation window, which is worse than showing nothing in cases where accuracy materially affects the customer's purchase decision. A cached last-known-good response (the last successful real estimate fetched for this specific product, persisted with a timestamp) is a better fallback when per-product variance is significant, since it's product-specific and was genuinely accurate as of some recent point, rather than a one-size-fits-all guess — the trade-off is needing to build and maintain a cache layer (with its own staleness considerations, similar to the discussion in [[02-cache-invalidation-strategy]]) and clearly signaling to the user that the shown estimate might be slightly stale (e.g., "Estimated as of [time]" rather than presenting it with the same confidence as a fresh real-time value) so a genuinely outdated cached estimate isn't presented as equally authoritative as a live one.

The trap: treating "some fallback is better than a blank space" as sufficient without considering whether the specific fallback chosen could be actively misleading for at least some products — graceful degradation should still aim for the least-wrong available option, not just the easiest-to-implement one, especially if this value has any influence on a purchase decision.

---

**Q (Medium): How would you monitor this system in production to know whether the circuit breaker, timeouts, and retries are actually working as intended, versus silently masking a problem that should be getting more attention?**

Answer: I'd instrument each layer distinctly rather than only tracking the end-user-visible outcome (real estimate shown vs. fallback shown): metrics for raw request success/failure/timeout rate to the third-party API (the ground truth of how the dependency itself is behaving, independent of any resilience logic wrapping it), circuit breaker state transitions (how often and for how long it opens, since frequent opening is itself a signal worth alerting on even though the user experience stays smooth throughout), and retry outcome breakdown (what fraction of failures succeed on retry vs. exhaust all retries) — together these distinguish "the dependency is having brief, retry-recoverable blips" from "the dependency is having a sustained outage the breaker is correctly absorbing" from "the breaker itself might be misconfigured" (e.g., opening too aggressively on normal transient noise, per the earlier gotcha). Critically, graceful degradation succeeding at its job (users see a smooth fallback, no visible breakage) can make an underlying, worsening problem invisible to anyone just watching user-facing error rates or support tickets — the whole point of graceful degradation is that users don't notice, which means the team needs to be watching the *masked* signal (raw dependency health, breaker state) directly, not inferring dependency health from an absence of user complaints.

The trap: only monitoring user-facing success/failure (does the page load, are there error reports) and treating a quiet dashboard as "everything's fine" — a well-built graceful-degradation system can make a severely degraded or entirely down third-party dependency invisible from the user-facing side precisely because it's working correctly, which means dependency health needs its own, separate, honest monitoring layer that isn't filtered through the fallback logic.

---

**Q (Low): If this shipping-estimate API call happened server-side (e.g., during SSR, to include the estimate in the initial HTML) rather than client-side, would the resilience strategy need to change?**

Answer: The same core principles apply (timeout, retry, circuit breaker, fallback), but server-side introduces a sharper consequence for getting the decoupling wrong: if the shipping-estimate fetch blocks the SSR response itself (the server won't send *any* HTML until it resolves), a slow or hung third-party call doesn't just delay one component's data — it delays Time to First Byte for the entire page, for every concurrent request being server-rendered, which can also tie up server-side request-handling capacity/concurrency slots for longer than necessary under load, a resource-exhaustion risk that a purely client-side hung fetch doesn't create for the server itself. This argues even more strongly for excluding this call from the blocking SSR data-fetch path — either fetching it client-side after the initial page render (accepting a brief loading state for just that section, exactly as designed above) or, if it needs to be server-rendered for SEO/no-flash-of-missing-content reasons, giving the *server-side* fetch an even more aggressive timeout than the client-side version would use, specifically because a hung server-side request has a blast radius (concurrent request capacity) that a hung client-side request doesn't.

The trap: assuming the resilience patterns (timeout/retry/circuit breaker/fallback) are architecture-agnostic and porting them unchanged from client to server without considering the additional server-specific consequence (TTFB for the whole page, and server concurrency/capacity impact under load) that a hung dependency call has when it's blocking a server response rather than a client-side component render.

---

## Self-Assessment

- [ ] Can name and distinguish the four resilience mechanisms (timeout, retry, circuit breaker, fallback UI) and which specific failure mode each addresses
- [ ] Can explain precisely what a circuit breaker adds beyond timeout + retry, using the cross-request-memory argument
- [ ] Can explain why decoupling an optional feature's data-fetch from the critical render path matters independent of that dependency's actual reliability
- [ ] Can walk through circuit breaker state transitions (closed → open → half-open) and justify why a failed half-open trial reopens rather than allowing more immediate retries
- [ ] Can evaluate fallback-UI choices (generic static vs. cached last-known-good) against the risk of the fallback itself being misleading
- [ ] Can identify that graceful degradation succeeding can mask a worsening dependency problem, and knows to monitor raw dependency health separately from user-facing success rate

---
*Next: GraphQL Client-side N+1 / Over-fetching — closes out Phase 7 by shifting from resilience against an external dependency's failures to correctness and efficiency in how the client itself shapes its data requests.*
