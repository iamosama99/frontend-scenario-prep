# Handling a Breaking API Change From Another Team

## Quick Reference

| Step | Mechanism | Why |
|---|---|---|
| Stop the bleeding | Roll back / feature-flag off / hotfix a compatibility shim | Restore users first, argue later |
| Isolate the blast radius | Anti-corruption layer: one adapter between API shape and UI models | Contract changes touch one file, not 200 components |
| Fix the process, not the person | Contract/versioning policy, schema diff in CI, deprecation windows | Prevents the *class* of failure |
| Negotiate the path forward | Align on timelines, versioned endpoints or additive changes, shared ownership of the contract | Cross-team trust determines the next incident |

## The Scenario

"Monday morning: the Orders page is blank for everyone. You trace it to the backend team renaming `order.total` to `order.pricing.total` and making `items` nullable — deployed Friday evening with no heads-up. Walk me through the next 48 hours."

## Clarifying Questions

- **What's the current user impact and severity — full outage, degraded feature, silent wrong data?** Silent wrong data (a missing total shown as $0) can be worse than an error.
- **Can the backend roll back or serve both shapes right now?** The fastest mitigation is often on their side; I need to know our options.
- **Did we have any warning signs — changelog, schema diff, contract tests?** Reveals whether this is communication, tooling, or ownership failure.
- **How many clients consume this API — web, mobile, partners?** Determines whether a rollback is the only safe fix (mobile can't hotfix instantly).
- **Is the change intended permanent?** If the new shape is the future, we'll adapt; if it was a mistake, they should revert.
- **Who owns the API contract and what's the existing versioning/deprecation policy?** Whether there's a process to enforce or one to create.

## Approach & Trade-offs

**Sequence matters: mitigate, then diagnose, then prevent.**

1. **Mitigate (minutes–hours).** Priority is restoring users. Options in order of preference: (a) backend reverts or ships a backward-compatible response (both `total` and `pricing.total`) — fastest and safest, especially with mobile clients; (b) we ship a frontend hotfix tolerant of both shapes; (c) feature-flag/kill-switch the affected view to a graceful fallback message. Pick the lowest-risk path that works, and don't wait for blame to be settled.

2. **Communicate.** Open an incident, state impact, owner, ETA, and mitigation status to stakeholders and in the backend team's channel. Be factual and blameless: "Friday's deploy changed the Orders response shape; client expected X." Being calm and specific preserves the relationship you need for the fix.

3. **Diagnose the real gaps.** The change broke production because several safeguards were absent: no contract tests, no schema-diff gate in CI, no deprecation policy, no staging integration check, weak error handling on our side (a missing field shouldn't blank the page).

4. **Prevent recurrence — both sides.** Technical: contract testing (consumer-driven with Pact, or OpenAPI diff checks that flag breaking changes in the backend's CI), runtime validation at the boundary, an anti-corruption layer, observability on parse failures. Process: additive-only changes by default (add `pricing.total` alongside `total`, deprecate with a window), versioned endpoints (`/v2`) for breaking changes, a changelog/notification channel, and a named contract owner.

**My side of the street.** It's tempting to blame the backend, but a frontend that crashes on a missing field has its own fragility. I'd add a defensive adapter and runtime validation, so that unexpected shapes fail loudly in monitoring and degrade gracefully in the UI instead of rendering a blank page.

**Trade-off: strict contracts vs. velocity.** Heavy governance slows teams. Aim for lightweight, automated enforcement: breaking-change detection in CI is cheap and catches most issues, whereas a manual approval board scales poorly. Pact-style consumer contracts add overhead but are valuable for widely consumed APIs.

**Trade-off: rollback vs. forward-fix.** Rolling back is fast but delays their feature; forward-fixing with a compatibility shim keeps their progress. With multiple clients, additive compatibility on the backend is usually best.

## Solution

### Immediate: tolerant adapter (hotfix)

```ts
// api/orders.adapter.ts — the only place that knows the wire format
const RawOrder = z.object({
  id: z.string(),
  total: z.number().optional(),                 // legacy
  pricing: z.object({ total: z.number() }).optional(),  // new
  items: z.array(Item).nullable().optional(),
});

export function toOrder(raw: unknown): Order {
  const parsed = RawOrder.safeParse(raw);
  if (!parsed.success) {
    reportContractViolation('orders', parsed.error);   // alert, don't crash silently
    throw new ApiContractError(parsed.error);
  }
  const r = parsed.data;
  return {
    id: r.id,
    total: r.pricing?.total ?? r.total ?? 0,   // see Gotchas: avoid masking with 0 in money
    items: r.items ?? [],
  };
}
```

### Graceful UI degradation

```tsx
<ErrorBoundary fallback={<OrdersUnavailable onRetry={refetch} />}>
  <OrdersPage />
</ErrorBoundary>
```

### Prevent: breaking-change gate in the backend's CI

```bash
# compare OpenAPI spec against main, fail on breaking changes
npx oasdiff breaking origin/main:openapi.yaml ./openapi.yaml --fail-on ERR
```

### Prevent: consumer-driven contract (Pact)

```ts
provider.addInteraction({
  uponReceiving: 'a request for an order',
  withRequest: { method: 'GET', path: '/orders/1' },
  willRespondWith: { status: 200, body: like({ id: '1', total: 100 }) },
});
```

### Blameless post-incident write-up (outline)

- Timeline, impact, detection (how long until we noticed?)
- Contributing factors on both sides
- Action items with owners: contract checks in CI, deprecation policy, adapter + validation, alert on parse errors, change-notification channel
- Agreement on additive-first changes and a deprecation window (e.g., 90 days, announced)

> **Check yourself:** What are the three mitigation options in order of preference, and what two things would you change on the *frontend* regardless of whose fault it was?

## Gotchas

- **Starting with blame.** Burns the relationship and delays the fix; stay blameless and factual.
- **Frontend hotfix only.** With mobile or other consumers, the backend must be compatible; fixing one client leaves others broken.
- **Defaulting a missing money field to 0.** Replaces a visible error with silently wrong data; for critical fields, fail loudly.
- **Fixing the incident without the process.** Same failure recurs next quarter.
- **No runtime validation.** TypeScript types won't catch a changed server shape.
- **Heavy governance proposals.** A change-approval board kills velocity; prefer automation.
- **Not measuring detection time.** If users noticed before you did, add monitoring on parse failures and error rates.

## Follow-up Questions

**Q (High): What do you do in the first hour?**

Answer: Declare an incident, assess impact, and restore service by the fastest safe route: ask the backend to roll back or add backward compatibility, or ship a tolerant frontend hotfix/kill-switch if that's quicker. Communicate status clearly. Defer root-cause and blame until users are unblocked.

The trap: debating fault or doing a clean refactor before restoring service.

**Q (High): How do you prevent breaking API changes from reaching production?**

Answer: Automated contract protection: OpenAPI/GraphQL schema diffs that fail on breaking changes in the provider's CI, consumer-driven contract tests, versioning and deprecation windows, and additive-first change practices. Plus runtime validation and monitoring on the consumer side so drift is caught immediately.

The trap: "better communication" alone, with no mechanism.

**Q (Medium): What is an anti-corruption layer and why use it here?**

Answer: An adapter that translates the backend's wire format into the frontend's domain model in exactly one place. When the API changes, only the adapter changes; components never see raw API shapes. It also is the natural place for validation and defaulting.

The trap: components reading `response.data.pricing.total` directly everywhere.

**Q (Medium): How do you handle the conversation with the other team?**

Answer: Lead with impact and data, not accusations. Align on the immediate fix, then propose shared improvements both sides benefit from (contract checks, change notifications, deprecation policy). Acknowledge our own fragility. Aim to leave with agreed owners and dates.

The trap: escalating to management as the first move.

**Q (Low): Versioned endpoints vs. additive changes — when each?**

Answer: Additive and optional-field changes need no version and should be the default. Breaking changes (removal/rename/semantic change) warrant a new version or a parallel field with a deprecation period, so clients migrate on their schedule. Versioning has a maintenance cost, so don't use it for trivial changes.

The trap: versioning everything, or never versioning.

## Self-Assessment

- [ ] Can order mitigation options and justify the first choice
- [ ] Can describe the communication during an incident
- [ ] Can name tooling that prevents breaking changes (schema diff, Pact)
- [ ] Can explain the anti-corruption layer and runtime validation
- [ ] Can distinguish blameless post-incident learning from blame
- [ ] Can state additive-first and deprecation-window practices

---
*Next: Rolling Out a Risky Refactor Without Breaking Prod — the delivery-side counterpart: how you change something large safely when you own the change.*
