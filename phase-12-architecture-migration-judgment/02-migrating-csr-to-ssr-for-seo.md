# Migrating a CSR App to SSR for SEO

## Quick Reference

| Decision | Mechanism | Why it's the right call |
|---|---|---|
| Verify SEO is actually the problem | Check Search Console, rendered-HTML (URL Inspection), indexing coverage | Google renders JS; the issue might be crawl budget, metadata, or speed, not CSR itself |
| Render only what needs it | SSR/SSG/ISR for public, indexable, shareable routes; keep CSR for authed app areas | SEO value is concentrated in public pages; SSR everything is costly |
| Move incrementally | Adopt a framework (Next/Remix/Nuxt) route by route behind the same domain | Avoids a big-bang rewrite; measurable per route |
| Plan for hydration & server realities | Isomorphic-safe code, data fetching on server, caching/CDN, streaming | SSR changes deployment, performance, and failure modes |

## The Scenario

"Our marketing and product-catalog pages are in a React SPA served as an empty `<div id="root">`. SEO says organic traffic is flat and competitors outrank us; link previews on Slack/Twitter show nothing. They want SSR. How do you approach this?"

## Clarifying Questions

- **What evidence says CSR is the cause?** Googlebot does render JavaScript, but later and with budget limits. I'd check URL Inspection "rendered HTML," indexed-page count, and crawl stats before assuming. Link previews failing is a separate, definitely-CSR issue (social crawlers don't run JS).
- **Which pages matter for SEO — a small set (landing, category, product, blog) or the whole app?** Scopes the work; logged-in dashboards need no SSR.
- **How dynamic is the content — per-user, per-request, frequently changing prices/stock, or mostly static?** Chooses between SSG, ISR, and true per-request SSR.
- **What's the current stack and infra — static hosting on a CDN, a Node platform, serverless/edge?** SSR requires a runtime; this is a real operational change.
- **What are the performance and Core Web Vitals baselines?** SSR can improve LCP but can hurt TTFB and INP if done naively.
- **Team capacity and timeline, and is there a hard deadline from marketing?** Determines whether we start with a cheaper stopgap (prerendering).

## Approach & Trade-offs

**First, diagnose; second, choose the cheapest sufficient fix.** The spectrum, cheapest to most invasive:

1. *Fix metadata and crawlability within CSR:* unique titles/meta/canonical, structured data, sitemap, clean URLs, internal links as real `<a href>`. Often delivers more than expected.
2. *Prerender/dynamic rendering for bots:* serve pre-rendered HTML snapshots (prerender service/static prerender at build). Fast to ship, but caveats: staleness, cloaking concerns if content differs, and Google no longer recommends dynamic rendering as a long-term solution.
3. *SSG/ISR for mostly-static pages:* build-time or incrementally regenerated HTML served from a CDN. Best performance/cost for catalog/blog pages.
4. *Full SSR (per-request):* for content that must be fresh per request or personalized. Highest operational cost.

I'd advocate **hybrid by route** in a framework like Next.js/Remix: SSG/ISR for catalog and marketing, SSR where freshness demands it, CSR for authenticated app shells.

**What SSR buys and costs.** Buys: meaningful first HTML for crawlers and social previews, often better LCP/First Contentful Paint on slow devices. Costs: a server to run, scale, and monitor; TTFB now depends on data fetching; hydration cost (the page is visible but not interactive until JS runs — INP/TBT can regress); code must run in two environments (no `window` at module scope); more complex caching; harder debugging.

**Incremental path.** Put the new framework behind the same domain and route migrate page types: reverse-proxy `/products/*` to the SSR app while `/app/*` stays on the SPA. Measure each tranche (indexation, impressions, CWV) before expanding. This also bounds risk and lets SEO verify gains.

**Data fetching and shared code.** Move data fetching to the server per route (loaders/`getServerSideProps`/server components), keep a typed API client usable in both environments, and ensure auth cookies are forwarded. Cache aggressively at the CDN with `stale-while-revalidate`.

**SEO details that are easy to get wrong.** Correct HTTP status codes (real 404/301, not soft-404 with a 200), canonical URLs, `hreflang`, structured data, no content differing between bot and user, consistent URL structure during migration with 301s to prevent rankings loss.

## Solution

### Rendering strategy per route

| Route | Content | Strategy |
|---|---|---|
| `/` , `/pricing`, `/blog/*` | Static, changes on publish | SSG with on-demand revalidation |
| `/products/[slug]` | Thousands of pages, price/stock changes hourly | ISR (`revalidate: 300`) + on-demand purge on update |
| `/search?q=` | Dynamic | SSR with short CDN cache, or CSR + noindex |
| `/app/*` (authed) | Per-user | CSR; `noindex` |

### Product page (Next.js App Router)

```tsx
// app/products/[slug]/page.tsx
export const revalidate = 300;

export async function generateMetadata({ params }): Promise<Metadata> {
  const p = await getProduct(params.slug);
  if (!p) return {};
  return {
    title: `${p.name} | Acme`,
    description: p.summary,
    alternates: { canonical: `https://acme.com/products/${p.slug}` },
    openGraph: { title: p.name, images: [p.image] },
  };
}

export default async function ProductPage({ params }) {
  const product = await getProduct(params.slug);
  if (!product) notFound();                       // real 404 status
  return (
    <>
      <script type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(productJsonLd(product)) }} />
      <ProductDetails product={product} />
      <AddToCart sku={product.sku} />             {/* client component, hydrates */}
    </>
  );
}
```

### Incremental routing via reverse proxy

```nginx
location /products/ { proxy_pass http://ssr-app; }
location /blog/     { proxy_pass http://ssr-app; }
location /          { root /var/www/spa; try_files $uri /index.html; }
```

### Making code isomorphic-safe

```ts
// ❌ crashes on the server
const width = window.innerWidth;
// ✅ guard or move into an effect / client-only component
useEffect(() => setWidth(window.innerWidth), []);
```

### Measurement plan

- Before/after: indexed pages, impressions/clicks (Search Console), CWV (field data), TTFB, crawl stats.
- Verify rendered HTML via URL Inspection and `curl` with a bot user-agent.
- Roll out by tranche, with rollback by flipping the proxy rule.

> **Check yourself:** For a product catalog with hourly price changes and 50k SKUs, which strategy would you pick and why not full per-request SSR?

## Gotchas

- **Assuming SSR is the fix without evidence.** Google renders JS; the real culprit may be metadata, thin content, or slow pages.
- **Hydration mismatches.** Server and client render different output (dates, random IDs, `window` checks) → warnings, flicker, or broken interactivity.
- **Soft 404s.** Returning 200 with a "not found" component hurts indexing; return real status codes.
- **Hurting INP.** SSR delivers pixels fast but a big hydration bundle blocks interactivity; use selective/streaming hydration and server components where available.
- **Per-request SSR without caching.** TTFB and cost explode under load; cache at the CDN.
- **Breaking URLs during migration.** Lost redirects = lost rankings.
- **Cloaking.** Serving bots different content than users risks penalties.
- **Leaking user data in SSR cache.** Caching personalized HTML at the CDN shows one user's data to another.

## Follow-up Questions

**Q (High): Doesn't Googlebot already execute JavaScript? Why would SSR help?**

Answer: Googlebot does render JS, but in a second-wave queue with delays and resource limits, so indexing of CSR content can be slow or incomplete; other crawlers (Bing is better now, but social/preview bots and many others) often don't execute JS. SSR/SSG gives complete HTML on the first response, improving indexing reliability and link previews, and often LCP. But it's not guaranteed to move rankings by itself — content quality, links, and speed still matter.

The trap: saying "Google can't read JS" (outdated) or "SSR guarantees better ranking."

**Q (High): SSR vs. SSG vs. ISR — how do you choose?**

Answer: By freshness and personalization. Static-per-publish content → SSG. Large catalogs with tolerable staleness → ISR (stale-while-revalidate regeneration, on-demand purge). Per-request or personalized freshness → SSR, ideally with short CDN caching. Authenticated app areas → CSR. Mix per route.

The trap: picking one strategy for the whole app.

**Q (Medium): What are the downsides of SSR?**

Answer: Operational complexity (a server runtime to scale and monitor), TTFB tied to data fetching, hydration cost impacting INP, dual-environment code constraints, trickier caching/security (user-data leakage), and harder local/prod parity. Mitigate with CDN caching, streaming, server components, and performance budgets.

The trap: presenting SSR as strictly better than CSR.

**Q (Medium): How would you migrate without a big-bang?**

Answer: Route-level strangling through a reverse proxy/CDN rules: public SEO routes served by the SSR app, authenticated routes stay in the SPA. Share design system and API client; migrate page types in tranches with metrics gates and instant rollback by routing config.

The trap: proposing a full rewrite of the app into Next for SEO of a handful of pages.

**Q (Low): How do you verify SEO outcomes after launch?**

Answer: Compare rendered HTML (URL Inspection, `curl` as Googlebot), monitor indexed page counts, impressions/CTR, crawl stats and CWV field data, check for duplicate/soft-404 reports, and validate structured data. Expect lag of weeks; judge by trend, not a day.

The trap: declaring success on launch day.

## Self-Assessment

- [ ] Can list the cheaper alternatives to full SSR and when they suffice
- [ ] Can choose SSG/ISR/SSR/CSR per route with reasoning
- [ ] Can explain hydration cost and mismatch risks
- [ ] Can describe an incremental route-level migration with rollback
- [ ] Can list SEO pitfalls (soft 404, canonical, redirects, cloaking)
- [ ] Can say how to measure whether it worked

---
*Next: Introducing TypeScript to a Large Untyped JS Codebase — another incremental migration, this time of the type system rather than rendering or framework.*
