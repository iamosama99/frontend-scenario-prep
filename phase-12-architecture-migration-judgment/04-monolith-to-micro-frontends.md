# Monolith Frontend to Micro-frontends — When and How

## Quick Reference

| Question | Answer | Why |
|---|---|---|
| Is the problem organizational or technical? | Micro-frontends solve *team autonomy/deploy coupling at scale*, not code quality | If it's mess in one codebase, fix modularity first |
| What's the cheapest alternative? | Modular monolith + monorepo + ownership (CODEOWNERS) + independent CI paths | ~80% of the benefit, ~20% of the cost |
| If you do split: integration approach? | Build-time packages · Module Federation (runtime) · iframes · server-side composition | Trades independence against UX consistency and perf |
| What must stay shared? | Design system, auth/session, routing contract, analytics, error handling | Otherwise users see 5 different apps |

## The Scenario

"We have a single React app maintained by ~80 engineers across 8 teams. Builds take 25 minutes, merges are a traffic jam, and one team's bug blocks everyone's release. Someone's proposing micro-frontends. Is that the right call, and if so how do we do it?"

## Clarifying Questions

- **What exactly hurts — build time, merge conflicts, release coupling, unclear ownership, or inconsistent code?** Each has cheaper fixes; micro-frontends target release/ownership coupling specifically.
- **How are the teams organized — by product domain (checkout, search, account) or by layer?** Domain-aligned teams map naturally to vertical slices; layered teams don't.
- **How coupled is the code — shared state, cross-feature imports, one giant store?** If features are tangled, splitting deployment just distributes the tangle.
- **What are the UX constraints — one seamless app, or distinct areas where page-load navigation is fine?** Seamlessness raises integration cost.
- **What's the org's platform maturity — can we run independent pipelines, versioning, observability per slice?** MFEs multiply operational surface.
- **Are there legacy/acquired apps that must be combined?** That's the strongest legitimate case: stitching together things you can't rewrite.

## Approach & Trade-offs

**Start by being skeptical.** Micro-frontends are an *organizational scaling pattern*. They buy independent deployability and team autonomy at the cost of duplication, runtime complexity, inconsistent UX, performance overhead, and harder cross-cutting concerns. Many teams adopt them for build-time or code-organization problems that a modular monolith fixes more cheaply.

**Try the cheaper ladder first.**

1. *Modular monolith with enforced boundaries:* domain folders/packages with explicit public APIs, lint rules (e.g., `eslint-plugin-boundaries`, Nx module boundaries) forbidding deep cross-imports.
2. *Monorepo tooling:* Nx/Turborepo with affected-only builds and remote caching — usually cuts the 25-minute build to a few minutes.
3. *Ownership and CI isolation:* CODEOWNERS, per-package test pipelines, trunk-based development with feature flags to decouple *release* from *deploy*.
4. *Independent deployment of a vertical slice* only if (1–3) still aren't enough.

**When micro-frontends are genuinely justified.** Many autonomous teams needing independent release cadence; different tech stacks inherited (acquisitions, legacy migration — the shell hosts old and new); a very large product surface where coordinated releases are the bottleneck. A rule of thumb: if you can't staff a platform/enablement team, you probably can't afford MFEs.

**Integration approaches and trade-offs.**

| Approach | Independence | UX/perf | Complexity |
|---|---|---|---|
| Build-time npm packages | Low (must rebuild shell to release) | Best | Low |
| Runtime via Module Federation / import maps | High | Good if shared deps are managed | Medium–High (version skew, shared singletons) |
| Server-side composition (ESI/edge includes) | High | Good for content pages | Medium |
| iframes | Very high (hard isolation) | Poor (sizing, a11y, routing, perf) | Low to start, high to live with |
| Separate apps by route (full navigation) | High | Page loads at boundaries | Low |

**Slice vertically, not horizontally.** A micro-frontend should be a full business slice (checkout, search) owned by one team end to end — not "the header" or "the buttons." Horizontal slicing yields chatty dependencies.

**Shared concerns are the hard part.** A thin platform layer owns: design system (versioned, consumed by all), authentication/session, a routing contract, event-based cross-MFE communication (custom events/pub-sub; avoid shared mutable global state), analytics/error tracking, and performance budgets. Without this, you get five apps stitched together with visible seams.

**Operational costs.** Version skew between shell and remotes, duplicate dependencies bloating bundles (React twice breaks hooks), cross-slice integration testing, debugging across boundaries, and coordinating breaking changes in the contract.

## Solution

### Recommended path

1. **Diagnose:** measure build time, PR wait time, release frequency, defect attribution by area.
2. **Do the cheap wins:** Nx/Turborepo with caching and affected builds; enforce module boundaries; feature flags + trunk-based dev; CODEOWNERS.
3. **Re-evaluate in a quarter.** If release coupling remains a top blocker, pilot **one** vertical slice as a micro-frontend.
4. **Pilot with Module Federation** for an isolated domain (e.g., Account Settings), keeping shell responsibilities small.

### Module Federation host/remote sketch

```ts
// shell/webpack.config.js
new ModuleFederationPlugin({
  name: 'shell',
  remotes: { account: 'account@https://cdn.acme.com/account/remoteEntry.js' },
  shared: {
    react: { singleton: true, requiredVersion: '^18.2.0' },
    'react-dom': { singleton: true, requiredVersion: '^18.2.0' },
    '@acme/design-system': { singleton: true },
  },
});
```

```tsx
// shell/routes.tsx
const AccountApp = lazy(() => import('account/App'));

<Route path="/account/*" element={
  <ErrorBoundary fallback={<AccountUnavailable />}>   {/* remote failure must not take down the shell */}
    <Suspense fallback={<PageSkeleton />}><AccountApp /></Suspense>
  </ErrorBoundary>
} />
```

### Cross-MFE communication — events, not shared state

```ts
// contract.ts (versioned package)
export type CartUpdated = CustomEvent<{ itemCount: number }>;
window.dispatchEvent(new CustomEvent('cart:updated', { detail: { itemCount: 3 } }));
```

### Guard rails

- Contract tests between shell and remotes; versioned event schemas
- Performance budget per remote; Lighthouse CI on composed pages
- Fallback UI + circuit breaker if a remote fails to load
- Design-system tokens/components as the single visual source

> **Check yourself:** List four cheaper alternatives to micro-frontends you'd try first, and the single organizational condition that most justifies them.

## Gotchas

- **Adopting MFEs for a technical problem.** Slow builds and messy code aren't solved by distributing deployment.
- **Horizontal slicing.** Splitting by UI layer causes chatty, tightly-coupled remotes.
- **Duplicate frameworks.** Two Reacts break hooks and double bundle size; enforce singletons.
- **Shared mutable global state.** Re-creates the monolith's coupling across a network boundary.
- **Version skew.** Shell and remote deploy independently; contract changes without compatibility windows cause runtime failures.
- **Inconsistent UX and a11y.** Without a shared design system and conventions, seams show; iframes break focus/keyboard flow.
- **Ignoring perf.** Waterfalls of remote entries delay first render; preload remotes and set budgets.
- **No failure isolation.** A failed remote must degrade gracefully, not blank the page.

## Follow-up Questions

**Q (High): Why not just use a monorepo with good boundaries?**

Answer: A modular monorepo gives shared tooling, atomic refactors, consistent dependencies, and with affected-only builds/remote caching fixes most build and merge pain, while keeping one deployable. Micro-frontends add independent *deployment*, which is only needed when release coordination across teams is the bottleneck. I'd exhaust the monorepo route before adopting runtime composition.

The trap: treating micro-frontends as the default answer to "big app."

**Q (High): How do you decide where to cut the boundaries?**

Answer: Along business domains and team ownership (Conway's law): vertical slices with their own routes, data, and UI, minimal runtime coupling, and stable contracts. Avoid cutting by UI component type. A good slice can be owned, built, tested, and released by one team with rare cross-team coordination.

The trap: "header team, footer team, content team."

**Q (Medium): How do micro-frontends communicate and share state?**

Answer: Prefer loose coupling: URL/query params, custom events or a small pub/sub with versioned schemas, props from the shell, and server-held state. Avoid a shared global store that every remote mutates. Shared auth/session via cookies/token provided by the shell.

The trap: a shared Redux store across remotes.

**Q (Medium): What's the performance cost and how do you mitigate it?**

Answer: Extra network hops for remote entries, duplicated dependencies, and runtime composition waterfalls. Mitigate with shared singleton deps, preloading remote entries, CDN caching with immutable hashed assets, route-level splitting, and per-remote bundle budgets measured in CI.

The trap: ignoring the user-facing cost while optimizing team convenience.

**Q (Low): How do you handle a remote that fails to load or ships a bug?**

Answer: Error boundary and fallback UI around each remote, timeouts/circuit-breaker on remote loading, pinned/versioned remote URLs with fast rollback, and canary rollout so a bad deploy affects a small share before full release.

The trap: no isolation, so one team's deploy takes down the whole page.

## Self-Assessment

- [ ] Can say what organizational problem micro-frontends solve (and what they don't)
- [ ] Can list the cheaper alternatives to try first
- [ ] Can compare integration approaches with trade-offs
- [ ] Can explain vertical slicing and the shared platform layer
- [ ] Can sketch a Module Federation setup with singleton shared deps
- [ ] Can describe failure isolation and rollback

---
*Next: Choosing a Rendering Strategy for a New Product — applies the CSR/SSR/SSG trade-offs from the SEO scenario to a greenfield decision.*
