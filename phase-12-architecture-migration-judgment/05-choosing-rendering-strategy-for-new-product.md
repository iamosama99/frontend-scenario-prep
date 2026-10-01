# Choosing a Rendering Strategy for a New Product

## Quick Reference

| Product trait | Leans toward | Why |
|---|---|---|
| Public, SEO/sharing-critical, mostly static | SSG / ISR | HTML from a CDN: fastest, cheapest, most resilient |
| Public but fresh/personalized per request | SSR (+ edge/CDN caching, streaming) | Per-request HTML; pay for a runtime |
| Authenticated, interaction-heavy app | CSR (SPA) with a fast shell | No SEO need; simpler infra; client owns state |
| Mixed (marketing + app) | Hybrid per route | Choose by route, not by product |

## The Scenario

"We're starting a new product: a B2B project-management tool with a public marketing site, a docs section, and a logged-in app with real-time boards. The team is six engineers who know React. What rendering strategy do you pick, and how do you defend it?"

## Clarifying Questions

- **What's the SEO and sharing need per area?** Marketing and docs need indexable, previewable HTML; the logged-in board doesn't. This alone splits the answer by route.
- **How personalized and fresh is each page's content?** Static docs vs. per-user dashboards vs. real-time boards map to different strategies.
- **What are the performance targets and user devices?** Targets on slow mobile networks raise the value of server-rendered HTML; desktop B2B users on good connections lower it.
- **What infrastructure and ops capacity do we have?** SSR needs servers/edge runtime and observability; six engineers shouldn't run a complex platform without need.
- **Is a mobile app or public API planned?** Influences whether logic should live in a backend/BFF rather than in server-rendered components.
- **How long-lived and large will the app grow?** Choice should survive scaling the team without a rewrite.

## Approach & Trade-offs

**Rendering strategy is per route, not per product.** The most common mistake is picking one global answer. I'd map each surface to its requirements:

- **Marketing site and docs:** SSG (build-time HTML on a CDN), with ISR/on-demand revalidation if content updates frequently from a CMS. Best LCP, best SEO, near-zero server cost, high availability. Trade-off: build time grows with page count (mitigated by ISR), content edits need a rebuild/revalidation.
- **Logged-in app:** CSR with a lightweight, cacheable shell. No SEO need; content is per-user; real-time boards live on the client with WebSockets. SSR here adds server cost and hydration complexity for little gain. Trade-off: a blank/skeleton shell until JS runs, so keep the bundle lean, route-split, and show skeletons fast; consider streaming/server components only if time-to-first-meaningful-content proves poor.
- **Public pages that need freshness per request** (e.g., public shared board links, pricing by region): SSR with short CDN caching or ISR.

**Pick a framework that doesn't force the choice.** A meta-framework with per-route rendering (Next.js, Remix/React Router, Astro for content-heavy sites, SvelteKit/Nuxt in other ecosystems) lets us mix SSG, SSR, and CSR in one codebase. For a six-person React team, Next.js or Remix with the marketing/docs statically generated and the app under a client-rendered route group is a pragmatic default.

**Decision factors and costs.**

| Strategy | TTFB | LCP | Interactivity | Infra | Freshness | Complexity |
|---|---|---|---|---|---|---|
| SSG | Best | Best | Hydrate cost | CDN only | Stale until rebuild/revalidate | Low |
| ISR | Best (cached) | Best | Hydrate cost | CDN + small runtime | Bounded staleness | Medium |
| SSR | Depends on data | Good | Hydrate cost | Runtime required | Fresh | Medium–High |
| CSR | Fast shell | Slower (waterfall) | After JS | CDN only | Fresh | Low, but SEO limited |
| Streaming/RSC | Good | Good | Less JS | Runtime | Fresh | High (new mental model) |

**What I'd resist.** SSR-everything because it's "modern"; micro-optimizing hydration strategies before measuring; picking a heavy architecture for six engineers. Start simple (SSG + CSR app), instrument Core Web Vitals, and upgrade specific routes when data demands it. The framework choice keeps that door open.

**Reversibility.** Because the strategy is per route, moving a route from CSR to SSR later is a localized change if data fetching is abstracted behind a typed client and isn't tangled with component code. I'd keep data access in loader/service modules usable on server and client.

## Solution

### Route map

| Route | Strategy | Notes |
|---|---|---|
| `/`, `/pricing`, `/about` | SSG | CMS-driven, on-demand revalidate on publish |
| `/docs/**` | SSG | MDX at build; search via static index |
| `/blog/**` | ISR (`revalidate` 1h) | Large and frequently updated |
| `/share/[token]` | SSR, `s-maxage=60` | Public, fresh, per-link |
| `/app/**` | CSR | `noindex`, auth-gated; skeleton shell; WebSocket boards |

### Next.js layout sketch

```
app/
  (marketing)/page.tsx          // static
  (marketing)/pricing/page.tsx  // static
  docs/[...slug]/page.tsx       // generateStaticParams
  share/[token]/page.tsx        // dynamic = 'force-dynamic'
  (app)/app/layout.tsx          // 'use client' shell, auth guard
  (app)/app/boards/[id]/page.tsx
```

```tsx
// (app)/app/layout.tsx
'use client';
export default function AppLayout({ children }) {
  const { user, status } = useSession();
  if (status === 'loading') return <AppShellSkeleton />;     // fast paint, no blank screen
  if (!user) redirect('/login');
  return <AppShell user={user}>{children}</AppShell>;
}
```

### Decision checklist (for the interview)

1. Does this route need to be indexed or previewed? → server-generated HTML.
2. Is content the same for everyone? → SSG/ISR. Per-user? → CSR or SSR with `private` caching.
3. How fresh must it be? → revalidation interval / SSR.
4. Is interactivity the dominant cost? → minimize client JS, consider islands/RSC.
5. Can the team operate the infrastructure? → prefer CDN-only options.

### Measurement-driven escalation

Track LCP/INP/TTFB per route in RUM. If `/app` first load is too slow: reduce bundle, preload critical data, then evaluate streaming SSR for the shell — not before.

> **Check yourself:** For each of the five routes above, justify the strategy in one sentence, and say which one you'd change first if RUM showed poor LCP.

## Gotchas

- **One strategy for the whole product.** Ignores that SEO and freshness needs differ by route.
- **SSR for authenticated apps by default.** Pays server cost and hydration complexity for no SEO benefit.
- **Caching personalized SSR output publicly.** Leaks user data; use `private`/no-store or cache only anonymous variants.
- **Underestimating hydration.** Fast HTML but unresponsive page if the JS bundle is huge.
- **Locking data fetching into components.** Makes moving a route between strategies expensive.
- **Choosing by trend, not constraint.** RSC/edge/streaming adopted with no measured need adds complexity.
- **Forgetting build-time scaling.** Pure SSG with 100k pages needs ISR or on-demand generation.

## Follow-up Questions

**Q (High): How do you pick between SSR, SSG, ISR, and CSR?**

Answer: By SEO/sharing need, personalization, freshness requirement, and ops capacity — per route. Static public → SSG; large or periodically changing public content → ISR; per-request fresh public → SSR with CDN caching; authenticated and interactive → CSR. Then validate with measurements.

The trap: naming a favorite instead of reasoning from constraints.

**Q (High): Why not SSR the logged-in app too?**

Answer: There's no SEO benefit, the content is per-user so CDN caching doesn't help, and SSR adds server cost, latency dependence on data fetching, and hydration complexity. A cached shell plus client fetch is simpler and scales cheaply. If measured first-load is poor, optimize the bundle and data waterfalls first; consider streaming only if needed.

The trap: "SSR is always faster," ignoring TTFB and hydration costs.

**Q (Medium): What does hydration cost, and how do you reduce it?**

Answer: The browser must download, parse, and execute the framework and component code to attach handlers, during which the page may look interactive but isn't (INP/TBT impact). Reduce with route-level code splitting, less client JS (server components/islands for static regions), lazy hydration, and trimming dependencies.

The trap: treating SSR as eliminating JavaScript cost.

**Q (Medium): How would you future-proof the choice?**

Answer: Use a meta-framework with per-route strategy, keep data fetching in framework-agnostic modules with typed clients, avoid server-only coupling in shared UI, and instrument RUM so strategy changes are evidence-based. Then migrating a route between modes is a small change.

The trap: promising a perfect decision up front instead of keeping options open.

**Q (Low): Where do edge runtimes fit?**

Answer: Edge SSR reduces latency for globally distributed users and suits lightweight personalization and A/B routing, but limited runtime APIs, cold-start/data-locality issues (database far from the edge) can negate the gain. Use when data is also edge-accessible or cached.

The trap: assuming edge automatically means faster.

## Self-Assessment

- [ ] Can map each route of a mixed product to a rendering strategy with one-line justification
- [ ] Can state the cost/benefit of SSR beyond "SEO"
- [ ] Can explain hydration and ways to reduce it
- [ ] Can explain why per-route strategies keep options open
- [ ] Can name the data-leak risk of caching personalized HTML

---
*Next: Handling a Breaking API Change From Another Team — shifts from technical choices to cross-team engineering judgment.*
