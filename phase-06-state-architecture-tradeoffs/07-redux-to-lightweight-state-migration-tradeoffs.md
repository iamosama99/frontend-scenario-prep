# Redux → Lightweight State — Migration Trade-offs

## Quick Reference

| Question | Why It Matters |
|---|---|
| What's actually in the Redux store today — server data, URL-shaped data, or genuine client UI state? | Most "migrate off Redux" wins come from realizing a large fraction of the store is server state that a query library should own, not from replacing Redux's UI-state role with a different UI-state library |
| Is this a big-bang rewrite or an incremental migration? | Incremental (module-by-module, with both systems coexisting temporarily) is almost always the right answer for anything beyond a small app — a full rewrite carries high risk for uncertain payoff |
| What's the actual pain being solved — boilerplate, bundle size, mental model, or a genuine architectural problem? | The right target (Zustand, Jotai, plain Context, or just deleting slices in favor of a query library) depends entirely on which pain is real |
| Can the migration be done slice-by-slice with a compatibility shim? | Determines whether teams can ship other work concurrently with the migration, or need a dedicated freeze |

## The Scenario

"This codebase has a large, years-old Redux store — action types, reducers, thunks, the works — for everything from the current user's session to cached API responses to which modal is open. The team wants to move to 'something lighter,' citing boilerplate fatigue. You're asked to lead this migration. Before writing any code, how do you approach deciding what changes, what the target architecture is, and how to roll it out without freezing feature work for months?"

## Clarifying Questions

- **What, concretely, is driving "boilerplate fatigue" — is it the action-type/reducer/dispatch ceremony for straightforward CRUD-shaped state, the amount of code needed to handle async request lifecycles (loading/error/success actions per API call), or something else entirely?** These have different fixes: if it's async-lifecycle boilerplate, the actual answer is likely "adopt a server-state/query library for API data," which would shrink the Redux store dramatically without needing to replace Redux's role for genuine UI state at all; if it's boilerplate for simple UI toggles, a lighter client-state library is the more direct fix. I wouldn't assume "replace Redux entirely" is the right scope before understanding which pain is actually driving the request.
- **How much of the current store is genuinely server-data-shaped (fetched from an API, cached, potentially stale) versus genuinely client-only UI state (modal open/closed, form draft, selected filters)?** This is close to the single most important question — in most real, years-old Redux codebases, a large fraction of "state management pain" turns out to be hand-rolled server-state caching that a purpose-built library (React Query, SWR, RTK Query) solves far better than any general state-management library, lightweight or not, and this often turns out to be a bigger win than the client-state library choice itself.
- **Is there appetite for a full rewrite, or does this need to be incremental — coexisting with the current Redux store for some transition period while feature work continues?** For "years-old" and presumably non-trivial in size, I'd push hard against a big-bang rewrite regardless of team preference, since the risk (regressions across a large surface area, a long feature freeze) is usually not justified by the payoff (less boilerplate) — but I'd still ask, since a small-enough store or an unusually strong business case could change that calculus.
- **Are there existing consumers of the Redux store outside of React components** — middleware doing cross-cutting things (analytics dispatch listeners, persisted-state sync, devtools-driven debugging workflows the team relies on) — that a lighter alternative would need to replicate or intentionally drop? Redux's middleware ecosystem is a real, sometimes-underestimated part of "what Redux is doing here" beyond just holding state, and losing it silently is a common migration regret.
- **What's the team's collective familiarity with the candidate target(s)** — has anyone used Zustand/Jotai/plain Context in production before, or would this be the team's first real usage? A migration to unfamiliar tooling carries its own risk and ramp-up cost distinct from the migration's mechanical difficulty, and is worth weighing honestly rather than assuming the "lighter" option is a free win just because it has less code.

## Approach & Trade-offs

**The first, most important move is auditing what's actually in the store and re-bucketing it using the same four-category classification from Scenario 1 (server / URL / session-UI / form state) — because the most common outcome of this audit in a real years-old codebase is discovering that a large share of "Redux pain" is actually misplaced server state, and the highest-leverage fix isn't picking a different general-purpose client-state library at all, it's introducing a dedicated query/cache library and deleting the corresponding Redux slices entirely.** This reframes "replace Redux" from "swap library A for library B, same shape" into "shrink the problem Redux needs to solve in the first place" — a store that goes from 40 slices (half of them hand-rolled API-response caching with manual loading/error/success actions) down to 8 slices of genuine client UI state is a fundamentally easier, lower-risk migration than trying to replicate all 40 slices' behavior in a new client-state library, boilerplate and all.

**For whatever's left after that reclassification — genuine client-only UI state — the choice of target (Zustand, Jotai, plain Context + `useReducer`, or even just more local component state) should be driven by the actual shape of what remains, not by "whichever is trendiest."** If what's left is mostly independent, loosely-related pieces of UI state (a scattered handful of "is this modal open," "which tab is selected" flags) that don't need cross-cutting subscription/selector optimization, plain Context or even fully local state per feature might be sufficient with no new dependency at all. If there's meaningfully complex, frequently-updated, widely-subscribed client state (a real-time collaborative UI's ephemeral state, a complex multi-step wizard's cross-step state) where Context's re-render-everything-on-any-change behavior would be a real performance problem, a library with fine-grained subscriptions (Zustand, Jotai) earns its keep. Naming this trade-off explicitly — rather than picking a library first and fitting the problem to it — is the actual senior signal here.

**The migration itself should be incremental, feature-by-feature or module-by-module, with both systems coexisting for a defined transition period — not a big-bang rewrite — for a codebase described as large and years-old.** The practical mechanism: introduce the new state layer for *new* features immediately (no reason to keep adding to a store the team has decided to move away from), and migrate *existing* Redux-backed features to the new approach one at a time, prioritized by which slices are easiest to extract cleanly (few cross-slice dependencies, clear boundaries) or which are causing the most active pain (frequent bug reports, the most boilerplate-heavy to touch) — not necessarily starting with the biggest or most central slice first, since that maximizes risk on the very first migration step before the team has built confidence with the new pattern on lower-stakes ground.

**A key mechanical risk during the coexistence period is state that's read or written from *both* systems during the transition — e.g., a feature migrated to the new store still needs to read something that another, not-yet-migrated feature still keeps in Redux.** I'd handle this with a clear, explicit boundary rather than ad hoc cross-reads: either a thin adapter/selector layer that both old and new code read through (so migrating the underlying source later doesn't require touching every consumer), or accepting a temporary "this one piece of data is intentionally duplicated/synced across both stores during the transition" bridge for the specific handful of genuinely cross-cutting pieces of state (most commonly: authenticated user/session info), with a tracked, deliberate plan for which system is authoritative during the overlap and when the bridge gets removed.

**Rollout risk should be managed the same way any large refactor's risk is managed — feature flags gating the new-vs-old code path per migrated feature, enabling a quick revert to the old Redux-backed implementation if a migrated feature regresses, without needing to revert the whole migration effort.** This is the same feature-flag-driven-rollout thinking covered in the next scenario, applied specifically here: each migrated slice/feature ships behind a flag, gets validated in production at low risk (a small percentage rollout, or internal-users-first), and the flag is removed (old code deleted) only once the new path has proven itself, rather than either migrating everything at once or leaving both code paths live indefinitely with no plan to retire the old one.

## Solution — a concrete migration plan

**1. Audit and reclassify every slice:**

```
Store audit (illustrative):
- `usersSlice`, `ordersSlice`, `productsSlice` → server state (API-cached data)
  → target: delete, replace with React Query hooks
- `authSlice` → mostly server state (session/user), but with a few genuine
  client-only derived flags (`isSessionExpiringSoon`) mixed in
  → target: split — session data via query/auth-context, derived flags
     recomputed locally where needed, not stored
- `uiSlice` (modal open/closed, active tab, sidebar collapsed) → genuine
  client UI state
  → target: Zustand store (or per-feature local state, if usage is scoped)
- `filtersSlice` → actually URL-shaped state (per Scenario 1)
  → target: delete, move to router search params
- `notificationsSlice` → server state + a genuine live-update layer
  → target: query cache + the cross-tab/real-time patterns from
     Scenarios 2 and 5
```

**2. New features go straight to the target architecture, immediately — no more additions to the old store:**

```tsx
// New feature, written directly against the target pattern, never touches Redux
const useSidebarStore = create<{ collapsed: boolean; toggle: () => void }>((set) => ({
  collapsed: false,
  toggle: () => set((s) => ({ collapsed: !s.collapsed })),
}));
```

**3. A thin compatibility bridge for state genuinely needed by both not-yet-migrated and already-migrated code during the transition:**

```tsx
// Session/user info: authoritative source during transition, exposed identically to both old and new consumers
function useCurrentUser() {
  // could internally read from the old Redux store OR the new source —
  // consumers on either side of the migration don't need to know which,
  // so migrating the underlying source later doesn't require touching every call site
  return useSelector(selectCurrentUser); // today; swapped internally when ready
}
```

**4. Feature-flagged rollout per migrated slice:**

```tsx
function OrdersPage() {
  const useNewOrdersState = useFeatureFlag('migrate-orders-state');
  return useNewOrdersState ? <OrdersPageV2 /> : <OrdersPageLegacy />;
  // V2 uses React Query + local/URL state; Legacy still reads ordersSlice
  // Both can coexist and ship concurrently; flip the flag per rollout stage
}
```

> **Check yourself:** Why prioritize migrating server-state-shaped slices to a query library *before* deciding on a client-state library replacement for the rest — what does doing it in that order avoid?

## Rollout Plan

**Phase 1 — audit and classify** every existing slice (as above), producing a migration backlog ordered by a mix of "easiest to extract cleanly" and "currently causing the most pain," not by slice size or centrality.

**Phase 2 — introduce the query library for server-state slices first**, since this is typically the highest-value, most mechanically well-trodden part of the migration (React Query's patterns for this are well-established) and immediately shrinks both the Redux store and the amount of hand-rolled async boilerplate the team was complaining about, independent of whatever client-state library decision comes next.

**Phase 3 — introduce the client-state target for genuine UI state**, migrating feature-by-feature behind flags, starting with lower-risk/higher-pain slices to build team confidence and establish patterns before tackling anything central (auth, cross-cutting UI chrome).

**Phase 4 — retire flags and delete old code** once each migrated feature has proven stable in production for a deliberate soak period, rather than leaving both implementations live indefinitely "just in case."

## Gotchas

**Treating this as "pick a new client-state library" without first auditing what fraction of the pain is actually misplaced server state.** Migrating 40 slices' worth of hand-rolled API caching logic into a different general-purpose state library, boilerplate and all, is a much larger and lower-value effort than recognizing 25 of those slices should be deleted in favor of a query library that solves caching/staleness/loading-state far better than any general state library would.

**A big-bang rewrite of a large, years-old store.** High risk of regressions across a broad, poorly-remembered surface area, usually requiring either a long feature freeze (expensive) or extremely thorough test coverage the team may not actually have for all of this state's edge cases (the more likely reality for old, organically-grown code).

**Choosing the new library before understanding what remains after the server-state extraction.** Picking Zustand/Jotai/Context first and then fitting whatever's left into it, rather than letting the actual shape of the remaining genuine UI state inform the choice, risks either over-provisioning (a fine-grained-subscription library for a handful of independent boolean flags that plain Context would have handled fine) or under-provisioning (Context for something with real fan-out performance needs).

**No plan for retiring old code once a feature is migrated.** Feature flags that never get cleaned up, with both old and new implementations of the same feature living indefinitely, produce exactly the kind of long-term complexity and confusion this migration was supposed to reduce.

**Losing Redux middleware-driven cross-cutting behavior silently during migration** — analytics dispatched off specific actions, state persistence, time-travel debugging the team actually relies on day-to-day — without an explicit replacement plan for each, discovered only after the fact when someone notices analytics events stopped firing.

## Follow-up Questions

**Q (High): Why should extracting server-state slices to a query library happen before finalizing the choice of client-state library for what remains — what specifically goes wrong if the order is reversed?**

Answer: If a client-state library is chosen and the *entire* existing store (server-shaped and UI-shaped state alike) is migrated into it before recognizing which parts are actually server data, the team ends up reimplementing cache invalidation, staleness, request deduplication, and loading/error lifecycle management by hand inside the new library — the exact same boilerplate-heavy problem that was presumably part of the original complaint about Redux, just rebuilt in a different, "lighter" library that was never designed to solve that problem well. Extracting server state first, into a purpose-built query library, removes that entire category of complexity from consideration when picking the client-state tool — the remaining decision (what handles genuine UI state) becomes much simpler and the resulting client-state code stays genuinely lightweight, because it's only handling what it's actually good at.

The trap: conflating "migrate off Redux" with "migrate everything currently in Redux into one new library" — the stronger move is recognizing that different categories of state deserve different tools, which was already the lesson from Scenario 1, just applied here at the scale of an existing large store rather than a greenfield design.

---

**Q (High): The team wants to know how long this migration will take before committing to it. How do you answer that, given it's explicitly incremental?**

Answer: I'd avoid giving a single end-to-end estimate for "the whole migration" as if it's one project with one deadline, and instead frame it as an ongoing capacity allocation with milestones: an initial audit (bounded, estimable — probably days, not weeks, for cataloging and classifying existing slices), then a series of individually-estimable, independently-shippable migration units (one slice/feature at a time, each with its own small estimate), proceeding at a pace the team can sustain alongside regular feature work rather than as a dedicated blocking initiative. I'd give a rough estimate for the *highest-value first phase* (extracting the biggest server-state slices, since that's usually the most mechanically similar across many slices once a pattern is established) and be explicit that the "genuine UI state" phase's pace depends on how much true complexity is discovered per feature, which is inherently less certain until each feature is actually looked at closely — a wide range with checkpoints to re-estimate, rather than a single confident number up front, is the honest answer for an incremental migration of unknown-in-detail legacy code.

The trap: giving a single confident total-time estimate for a large, incremental migration of code that hasn't been fully audited yet — this is a classic setup for a blown estimate, since the actual complexity of legacy slices (hidden cross-slice dependencies, subtle middleware behavior) is usually underestimated until each one is examined individually.

---

**Q (High): A slice being migrated turns out to be read by a Redux middleware that fires an analytics event on every dispatched action matching a pattern (e.g., every `*_SUCCESS` action). How do you handle this during migration, and what's the risk if it's missed?**

Answer: Before migrating a slice, I'd specifically audit for any middleware (or other cross-cutting Redux-specific behavior — persistence, devtools integration, saga/thunk side effects) that depends on actions from that slice being dispatched, since this is exactly the kind of dependency that's easy to lose silently when a slice is deleted in favor of a query library or a different state approach that doesn't dispatch the same actions. The concrete fix is making sure the *behavior* the middleware provided (in this case, an analytics event on a successful data fetch) gets an explicit equivalent in the new code — e.g., a query library's `onSuccess` callback firing the same analytics call directly — rather than assuming "delete the slice, the app still functions, must be fine," since a missing analytics event isn't something that shows up as a broken feature in QA; it shows up weeks later as a gap in a dashboard nobody's specifically watching for it, which makes it a genuinely dangerous silent regression to introduce.

The trap: verifying only that the migrated feature's *visible* behavior still works (the UI renders the same data correctly) and missing invisible side effects (analytics, logging, persistence) that were riding along on the old dispatch-based mechanism with no direct UI signal if they silently stop firing.

---

**Q (Medium): If the team has zero production experience with Zustand or Jotai, does that change your recommendation, even if one of them is technically the better architectural fit for the remaining UI state?**

Answer: It's a real factor, not a rounding error — a migration's risk isn't purely about the target architecture's technical merits, it also includes the team's ability to use it correctly and debug it confidently under pressure, and a first production use of an unfamiliar library, happening simultaneously with a large migration effort, compounds two sources of risk (the migration itself, plus the learning curve) at once. I'd weigh this against the specific gap between "the well-fitting choice" and "a choice the team already knows" — if plain Context/`useReducer` (something the team likely already has some familiarity with, being built into React) is genuinely sufficient for the actual remaining UI-state complexity after server-state extraction, defaulting to it and avoiding a new dependency and learning curve entirely is a reasonable, even preferable, call; if the remaining complexity genuinely needs fine-grained subscriptions that only a dedicated library provides well, I'd still recommend it, but explicitly budget time for the team to ramp up (a small non-critical feature as a first trial) before betting the core migration on unfamiliar tooling.

The trap: treating "which library is the objectively best architectural fit" as the only axis that matters — team familiarity and the compounding risk of learning a new tool during a high-stakes migration is a legitimate, often decisive, engineering trade-off, not a soft/political consideration to set aside.

---

**Q (Medium): How would you decide which Redux slice to migrate *first*, given the plan is incremental?**

Answer: I'd weight two factors: how cleanly the slice can be extracted (few or no cross-slice dependencies, a clear boundary — a slice that reads/writes only its own data with minimal coupling to other slices' state is far lower-risk to migrate than one deeply entangled with several others), and how much real pain it's currently causing (a slice that's the subject of frequent bug reports or that developers specifically complain about touching) — prioritizing something that's both easy to extract *and* meaningfully painful gives the migration effort visible, credible early wins that build team buy-in for continuing, versus starting with either the largest/most central slice (highest risk, slowest to show results) or an easy-but-low-impact slice (low risk, but doesn't demonstrate the migration is worth the ongoing investment to skeptical stakeholders).

The trap: starting with the biggest, most complained-about slice specifically because it's the most complained about, without weighing that it's also very likely the most deeply entangled and highest-risk to extract first, before the team has built any confidence or established patterns with the new approach on lower-stakes ground.

---

**Q (Low): Does this migration plan change if the app is small enough that the entire Redux store fits comfortably in a person's head and could genuinely be understood and rewritten in a few days?**

Answer: Yes — for a genuinely small store, the incremental/feature-flagged migration machinery described here is disproportionate overhead relative to the actual risk, and a more direct rewrite (understand the whole thing, replace it in one focused effort, verify against existing tests/manual QA) is likely faster and simpler than building out compatibility bridges and flag-gated coexistence for a store where "big-bang" risk is genuinely low because the surface area is genuinely small. The underlying principle (audit and reclassify by state category, extract server state to a query library first) still applies regardless of size — what changes is only the *rollout mechanism* (incremental-with-flags vs. a direct swap), which should scale with the actual size and risk of what's being migrated, not be applied as a fixed process regardless of context.

The trap: applying the same heavyweight incremental-migration machinery reflexively regardless of actual codebase size and risk — the scenario as posed describes a large, years-old store specifically because that's what justifies the incremental approach; a smaller store legitimately warrants a lighter-weight migration process.

---

## Self-Assessment

- [ ] Can explain why auditing and reclassifying existing state (server/URL/UI/form) is the first move, before picking a new library
- [ ] Can articulate why extracting server-state slices to a query library usually delivers more value than swapping client-state libraries
- [ ] Can justify an incremental, flag-gated migration over a big-bang rewrite for a large legacy store, and identify when a direct rewrite would be reasonable instead
- [ ] Can name a concrete risk of losing Redux middleware-driven side effects (analytics, persistence) during migration and how to guard against it
- [ ] Can reason about prioritizing which slice to migrate first using both extraction difficulty and current pain as factors

---
*Next: Feature-flag-driven Rollout Design — the flag-gated migration mechanism used here to de-risk each migrated slice becomes the main subject, generalized to rollout design for any risky change, not just a state-management migration.*
