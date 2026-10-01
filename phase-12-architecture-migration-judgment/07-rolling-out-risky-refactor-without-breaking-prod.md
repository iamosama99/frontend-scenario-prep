# Rolling Out a Risky Refactor Without Breaking Prod

## Quick Reference

| Technique | Mechanism | Why |
|---|---|---|
| Decouple deploy from release | Feature flag the new path; deploy dark | Rollback = flip a flag, not redeploy |
| Progressive exposure | Internal → 1% → 10% → 50% → 100% with guardrail metrics | Limit blast radius; catch what tests missed |
| Run old and new in parallel | Shadow/dual-run, compare outputs | Validate equivalence on real traffic without user impact |
| Small, reversible steps | Branch by abstraction; many small PRs behind the flag | Reviewable, bisectable, always shippable |

## The Scenario

"You're replacing the checkout form's state management — a tangle of component state and a custom event bus — with a state machine plus React Query. Checkout drives most of our revenue. You can't have a bad day. How do you roll this out?"

## Clarifying Questions

- **What's the risk profile — revenue impact per minute of breakage, regulatory/payment concerns?** Sets how conservative the rollout is.
- **What does existing test coverage look like, especially for the checkout journey?** If thin, characterization tests come first.
- **What observability exists — error rates, funnel conversion, RUM, session replay?** Guardrail metrics are how you detect regression at 1%.
- **Do we have a feature-flag/experimentation system with percentage and cohort targeting?** Determines the rollout mechanics.
- **Is the refactor purely behavior-preserving, or does it intentionally change UX?** Mixing behavior changes into a refactor makes regressions undiagnosable; I'd separate them.
- **Can both implementations coexist in the codebase for a while?** Needed for flagged rollout; costs duplication.
- **Business calendar constraints?** No rollout during peak sales events or code freezes.

## Approach & Trade-offs

**The core idea: make the change reversible at every step and observable at every stage.** A refactor is "risky" mostly because it's *irreversible and unobserved*. I turn it into a series of small, reversible, measured changes.

**1. Build the safety net first.** Characterization tests/E2E on the existing behavior (happy path, validation errors, payment failure, address edge cases, back-button). These tests define "behavior-preserving" and must pass against both old and new implementations.

**2. Branch by abstraction.** Introduce an interface/hook (`useCheckoutState`) that both the old and new implementations satisfy. Consumers depend on the interface. Now the new implementation can be built incrementally in the same codebase and merged to main continuously — no long-lived branch, no mega-merge.

**3. Ship dark behind a flag.** Merge the new implementation with the flag off. Deploy is decoupled from release. Production exercises the build, but users run the old path. Trunk-based and small PRs keep integration cheap.

**4. Verify equivalence before exposing users.** Options: *shadow mode* — run the new state machine alongside the old one on real sessions and compare derived outputs (totals, validation results, next step), logging mismatches without affecting the user. This finds divergences with zero user risk. Cost: extra compute, complexity; only viable for pure/derivable logic, not side effects (never double-submit payments in shadow).

**5. Progressive rollout with guardrails.** Employees/internal → 1% → 5% → 25% → 50% → 100%, pausing at each stage long enough to get statistical signal. Define *before* the rollout: guardrail metrics (checkout error rate, JS error rate, conversion rate, payment success rate, p95 latency) and automatic or explicit rollback thresholds. Compare cohort vs. control (it is effectively an A/B test of correctness).

**6. Rollback plan, tested.** Kill switch is a flag flip, ideally server-evaluated so it takes effect without a deploy or cache issues. Rehearse it. Data compatibility matters: if the new code writes state in a new format (localStorage, drafts), the old code must still read it, or you need a migration/dual-write.

**7. Clean up.** After full rollout and a soak period, delete the flag and the old implementation. Leaving both is accumulating debt and a latent source of bugs.

**Trade-offs.** Flags and dual implementations add temporary complexity and test-matrix size; they can rot. Percentage rollouts of a checkout may take weeks; stakeholders may push for speed. Shadow-mode comparison costs engineering effort. I'd accept all of these for a revenue-critical path, and scale the rigor down for low-risk changes — rigor should match blast radius.

**Sticky cohorts.** Users must see a consistent variant within a session (bucket by user/session ID), otherwise a user flips between implementations mid-checkout.

## Solution

### Branch by abstraction

```ts
// checkout/useCheckoutState.ts — the seam
export interface CheckoutState {
  step: Step; values: FormValues; errors: Errors;
  submit(): Promise<void>; next(): void; back(): void;
}

export function useCheckoutState(): CheckoutState {
  const useNew = useFlag('checkout-state-machine');   // sticky per user
  return useNew ? useCheckoutMachine() : useLegacyCheckoutState();
}
```

### Shadow comparison (no user impact, no side effects)

```ts
function useShadowCompare(legacy: CheckoutState) {
  const machine = useCheckoutMachine({ dryRun: true });     // no network side effects
  useEffect(() => {
    const a = project(legacy), b = project(machine);
    if (!deepEqual(a, b)) {
      telemetry.warn('checkout.shadow.mismatch', { step: a.step, diff: diff(a, b) });
    }
  }, [legacy, machine]);
}
```

### Rollout plan with guardrails

| Stage | Audience | Min soak | Advance if |
|---|---|---|---|
| 0 | Internal staff | 2 days | No new errors, QA sign-off |
| 1 | 1% | 3 days | Error rate ≤ baseline +0.1%, payment success within 0.5% |
| 2 | 10% | 3 days | Conversion non-inferior (CI excludes −1%) |
| 3 | 50% | 1 week | Same + p95 latency flat |
| 4 | 100% | 2 weeks soak | Then delete flag & legacy code |

Rollback trigger: any guardrail breach → flip flag off (no deploy), open incident, analyze with session replay.

### Observability checklist

- Error tracking tagged with variant (`checkout_impl: legacy|machine`)
- Funnel metrics by variant (step drop-off, completion rate)
- Alert on guardrail breach; dashboard comparing cohorts
- Log shadow mismatches, triage until zero

### Rehearsed rollback

```ts
// Run in staging and a canary: toggle flag mid-session — does in-flight checkout survive?
// New code persists draft in v2 format: legacy reader must tolerate it (dual-write v1+v2 during rollout).
```

> **Check yourself:** Why is the percentage rollout also a correctness experiment, and what must you define before starting it?

## Gotchas

- **Big-bang merge of a long-lived branch.** Conflicts, unreviewable diff, all-or-nothing release.
- **Flag without a rollback test.** A kill switch that wasn't exercised may not work under stress.
- **Non-sticky bucketing.** Users flipping implementations mid-flow causes bizarre bugs.
- **Mixing behavior changes into the refactor.** When metrics move you can't tell why.
- **No pre-defined success/rollback criteria.** Leads to argument and hesitation at the moment of decision.
- **Shadow mode with side effects.** A "dry-run" that double-charges is a disaster; keep it pure.
- **Data format drift.** New code writes data old code can't read; rollback then corrupts state.
- **Leaving flags forever.** Dead branches rot; schedule cleanup as part of the plan.
- **Insufficient sample size.** At 1% you may lack power to detect a 1% conversion drop; rely on error/technical metrics early and conversion at larger stages.

## Follow-up Questions

**Q (High): What's the difference between deploy and release, and why does it matter?**

Answer: Deploy puts code in production; release exposes it to users. Decoupling via feature flags lets you merge small changes continuously, test in production dark, roll out gradually, and roll back instantly without redeploying. It converts a high-stakes event into a controlled, reversible process.

The trap: treating a successful deploy as done.

**Q (High): How do you know the rollout is safe at each stage?**

Answer: Pre-defined guardrail metrics compared against a control cohort — error rates, payment success, conversion, latency — with explicit thresholds and soak times, plus tagged observability per variant. Technical health metrics gate early stages (low sample), business metrics gate later ones. Breach → flip the flag.

The trap: "we'll watch the dashboards" with no thresholds.

**Q (Medium): How do you refactor without a long-lived branch?**

Answer: Branch by abstraction: introduce a seam both implementations satisfy, land the new one incrementally behind a flag in small PRs to main. The codebase stays always-shippable and integration pain is spread out.

The trap: a six-week refactor branch and a Friday merge.

**Q (Medium): What can't be tested via shadow mode?**

Answer: Anything with side effects (payments, emails, writes) and behavior depending on real user timing. Shadow suits pure derivations (computed totals, validation, next-step decisions). For side effects, rely on tests, staged rollout, and idempotency keys.

The trap: suggesting dual-running the payment call.

**Q (Low): When is this level of rigor overkill?**

Answer: For low-traffic, easily-reversible, non-critical areas, a normal PR with tests and a quick follow-up is fine. Scale ceremony to blast radius and reversibility; the aim is risk-proportional engineering, not process for its own sake.

The trap: applying revenue-path ceremony to every change, or none to any.

## Self-Assessment

- [ ] Can explain deploy vs. release and why flags matter
- [ ] Can describe branch by abstraction with a concrete seam
- [ ] Can design staged rollout with guardrails and rollback triggers
- [ ] Can explain shadow mode and its limits
- [ ] Can list data-compatibility and sticky-bucketing pitfalls
- [ ] Can say when to clean up flags and when the process is overkill

---
*Next: Reviewing a PR With a Subtle Race Condition — closes the phase at the code-review level: spotting what automated checks miss.*
