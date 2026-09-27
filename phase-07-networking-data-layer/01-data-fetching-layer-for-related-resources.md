# Data-fetching Layer for N Related Resources (Waterfalls → Parallelization)

## Quick Reference

| Pattern | Mechanism | When It's Right |
|---|---|---|
| Sequential waterfall (the bug) | Each fetch's `useEffect`/`await` starts only after the previous one resolves | Never intentionally — only when resource B's params depend on resource A's *response* |
| Parallel fan-out | Fire all independent requests together (`Promise.all`, sibling `useQuery`s), render as each resolves | Default for independent resources — orders of magnitude faster on high-latency connections |
| Parallel with partial dependency | Fire the independent ones immediately; the dependent one fires once its input is ready, still in parallel with the others | Realistic mixed graphs — e.g., user profile and site config are independent, but "user's orders" needs the user ID first |
| Server-driven aggregation (BFF/GraphQL) | One round trip; server composes the response server-side | When round-trip latency dominates and the resources are always needed together |

## The Scenario

"This page needs four things to render: the current user, their permissions, a list of projects, and site-wide feature flags. Right now it takes almost three seconds to show anything useful, and when I open the Network tab I see the four requests going out one after another, like a staircase. Fix the data-fetching so this loads as fast as it possibly can, and walk me through how you'd structure the data layer so the next person who adds a fifth dependency doesn't reintroduce the same problem."

## Clarifying Questions

- **Are any of these four resources genuinely dependent on another's response** (e.g., does fetching "projects" require the user ID that only comes back from the "current user" call), or are all four independently fetchable given information already available on page load (a route param, an auth token)? This is the single question that determines whether full parallelization is even possible — a real dependency can't be parallelized away, but a *perceived* one (e.g., "projects" was written to take a `user` object as an argument purely out of convenience, when it only actually needs `userId` which is already known from the route) often can.
- **Does the UI need all four before it can render anything, or can each piece render independently as it arrives?** If the page can show the shell + feature-flag-gated UI immediately and stream in the user's projects a moment later, the target isn't "make four requests faster," it's "stop blocking the whole page on the slowest of the four" — a fairly different fix (progressive rendering per-resource) than pure request parallelization.
- **Is this fetch waterfall coming from `useEffect` chains inside components, or from route-level data loading (getServerSideProps/loader functions), or a mix?** Component-level waterfalls are usually an accidental architecture problem (each component fetches on mount, and a child component's effect depends on a parent's fetched data being present as a prop); route-level waterfalls are more often a genuine sequencing decision that needs re-examining. The fix location differs.
- **What's the actual network condition being optimized for** — is three seconds mostly latency-bound (each request ~700ms regardless of payload, meaning round-trip count matters most) or bandwidth-bound (payloads are large)? Parallelizing four sequential 700ms round trips down to one round trip's worth of wall-clock time is a latency fix; it does nothing for a slow connection struggling with payload size, which needs a different fix (smaller payloads, compression).
- **Is a shared data-fetching library (React Query, SWR, Apollo) already part of the stack, or is this raw `fetch`/`axios` in `useEffect`?** Changes whether the fix is "restructure existing waterfall effects to fire in parallel" or "introduce a library that makes parallel-by-default the path of least resistance for the next feature," which is the more durable fix given the prompt explicitly asks about preventing recurrence.

## Approach & Trade-offs

**Start by building the actual dependency graph between the four resources — not assumed, verified from the code.** A waterfall is only sometimes a real dependency chain; more often it's an accidental one, where a component's `useEffect` fires a fetch only after a parent has passed down data from *its* fetch, even though the child's fetch doesn't structurally need anything from the parent's response — just something (like a route param) the parent happened to have handy. I'd trace each of the four fetches back to what data they actually consume as input versus what they're waiting on incidentally.

**Independent resources should fire as one parallel fan-out, not four separate sequential effects.** With raw `fetch`/`async-await`, the classic mistake is:

```ts
// BUG: sequential waterfall — each await blocks the next fetch from starting
async function loadPageData(userId: string) {
  const user = await fetchUser(userId);
  const permissions = await fetchPermissions(userId);
  const projects = await fetchProjects(userId);
  const flags = await fetchFeatureFlags();
  return { user, permissions, projects, flags };
}
```

Every `await` here blocks the *next line*, not just the assignment — since none of `permissions`, `projects`, or `flags` actually needs `user`'s response body (they all only need `userId`, which is already known), this is four sequential round trips where one round trip's worth of wall-clock time would do.

**The trade-off in parallelizing isn't free — it changes error handling and partial-failure semantics, which needs to be an explicit decision, not an accident.** `Promise.all` rejects as soon as *any* one of the four rejects, discarding the other three even if they'd have succeeded — appropriate when the page genuinely can't render without all four, but wrong if, say, feature flags failing shouldn't block showing the user's projects. `Promise.allSettled` gets every result regardless of individual failure, at the cost of needing to check each result's `status` explicitly rather than being able to destructure a happy-path array. I'd pick per prompt: since the scenario says "this page needs four things to render," `Promise.all`-style fail-together semantics are defensible, but I'd confirm with the interviewer whether any of the four have sane fallbacks (e.g., feature flags defaulting to "off" on failure) that would argue for `allSettled` on that one instead.

**For the durable fix — preventing the next added dependency from silently becoming a fifth waterfall step — the shape of the data layer matters more than fixing these four calls by hand.** A raw-`fetch`-in-`useEffect` pattern makes sequential the path of least resistance (a developer adding a fifth fetch naturally reaches for "another `useEffect`, maybe depending on data from the one above it for convenience"). A shared data-fetching abstraction (React Query, SWR) where each resource is its own independent hook keyed by its own inputs makes *parallel* the path of least resistance instead — sibling `useQuery` calls fire independently by default; a developer has to go out of their way to force sequencing (e.g., via `enabled: false` gated on another query's result), which is a visible, deliberate choice rather than an accidental default.

## Solution

**Step 1 — with a shared data-fetching library, express each resource as its own independent hook:**

```tsx
function useCurrentUser() {
  return useQuery({ queryKey: ['user'], queryFn: fetchUser });
}
function usePermissions(userId: string | undefined) {
  return useQuery({
    queryKey: ['permissions', userId],
    queryFn: () => fetchPermissions(userId!),
    enabled: !!userId, // only fires once userId is known
  });
}
function useProjects(userId: string | undefined) {
  return useQuery({
    queryKey: ['projects', userId],
    queryFn: () => fetchProjects(userId!),
    enabled: !!userId,
  });
}
function useFeatureFlags() {
  return useQuery({ queryKey: ['flags'], queryFn: fetchFeatureFlags });
}
```

**Step 2 — compose them in the page component; React Query fires all `enabled`, non-dependent queries in parallel automatically:**

```tsx
function DashboardPage() {
  const { data: user } = useCurrentUser();
  const { data: permissions } = usePermissions(user?.id);
  const { data: projects } = useProjects(user?.id);
  const { data: flags } = useFeatureFlags();

  // user and flags fire immediately, in parallel, on mount.
  // permissions and projects fire the moment user.id is available —
  // still in parallel with each other, and as early as the true
  // dependency allows (not gated behind the whole user fetch resolving
  // in a `.then()` chain, just behind the one field they actually need).

  return (
    <Dashboard user={user} permissions={permissions} projects={projects} flags={flags} />
  );
}
```

This isn't a full waterfall-to-parallel win — `permissions`/`projects` still genuinely wait on `user.id` — but it's the *minimum* necessary sequencing rather than the accidental four-step staircase from the original code: two round trips deep at most (user → {permissions, projects} in parallel), with `flags` never in that chain at all.

**Step 3 — for progressive rendering, don't gate the whole page behind the slowest resource.** Each consumer reads its own query's loading state independently, so `flags`-gated UI can render the instant `flags` resolves without waiting on `projects`:

```tsx
function Dashboard({ user, permissions, projects, flags }: Props) {
  return (
    <Layout>
      <FeatureGatedBanner flags={flags} /> {/* renders as soon as flags resolves */}
      <UserHeader user={user} />
      {projects ? <ProjectList projects={projects} /> : <ProjectListSkeleton />}
    </Layout>
  );
}
```

**Step 4 — if this were raw `fetch` without a library, the equivalent minimum-necessary-sequencing version:**

```ts
async function loadPageData(userIdKnownUpfront: string) {
  const [user, flags] = await Promise.all([
    fetchUser(userIdKnownUpfront),
    fetchFeatureFlags(),
  ]);
  const [permissions, projects] = await Promise.all([
    fetchPermissions(user.id),
    fetchProjects(user.id),
  ]);
  return { user, permissions, projects, flags };
}
```

Two sequential stages instead of four, each stage internally parallel — the true dependency depth of this data, not an accidental one imposed by write order.

> **Check yourself:** If `fetchPermissions` and `fetchProjects` both only need a `userId` string (not the full `user` object), could you start them *before* the full `fetchUser` call resolves? What would that require knowing about where `userId` comes from?

## Data Model — Representing the Dependency Graph Explicitly

For pages with more than a handful of resources, tracking "what depends on what" implicitly through effect order gets unreadable fast. Making the dependency graph an explicit, inspectable structure (rather than inferred from where each `useQuery` call happens to sit in the component tree) is what actually prevents the next added resource from silently becoming a sixth waterfall step:

```ts
// Each resource declares its own inputs. A resource with no
// dependency on another resource's *output* always fires immediately.
const resourceGraph = {
  user: { deps: [], fetch: fetchUser },
  flags: { deps: [], fetch: fetchFeatureFlags },
  permissions: { deps: ['user'], fetch: (ctx) => fetchPermissions(ctx.user.id) },
  projects: { deps: ['user'], fetch: (ctx) => fetchProjects(ctx.user.id) },
};
```

This doesn't need a bespoke resolver to be valuable — the value is in making a reviewer able to answer "does adding resource #5 introduce a new sequential stage?" by reading one declaration (`deps: []` vs. `deps: ['user']`) instead of tracing render order across several components' effects.

## Gotchas

**Fixing the four calls in this scenario but leaving the underlying pattern (raw `fetch` in ad hoc `useEffect`s) in place**, so the fix is correct today and silently regresses the moment someone adds a fifth resource the "normal" way for this codebase. The prompt explicitly asks for this — treating it as "make these four fast" instead of "make sequential-by-accident structurally hard to do" is the most common way to under-deliver on this scenario.

**Reaching for `Promise.all` without considering that it fails all-or-nothing**, so one flaky, non-critical resource (feature flags) failing takes down user/permissions/projects rendering too, when a sane default (flags = off) would have let the rest of the page render fine. Not asking whether partial failure should be tolerated is a real design gap, not a pedantic point.

**Confusing "starts later because it depends on prior data" with "starts later because it was written after in the code"** — the second is the actual bug (accidental sequencing); mistaking real dependencies for accidental ones and trying to parallelize them anyway produces broken requests (calling `fetchProjects(undefined)` before `user.id` exists), while mistaking accidental ones for real dependencies leaves performance on the table.

**Parallelizing the network calls but not the loading UI** — if the page still renders one big spinner gated on `Promise.all` resolving (even though it's now one round trip deep instead of four), the wall-clock win exists but the perceived-performance win (something useful visible sooner) is left unclaimed; per-resource loading states matter as much as the request-level fix.

## Follow-up Questions

**Q (High): Walk through exactly why `await` inside a single `async` function creates a waterfall, at the mechanism level — what is actually blocking what?**

Answer: `await` doesn't block the thread (JS is still single-threaded and non-blocking under the hood — other work like rendering and event handling can proceed), but it does block the *next line of the same async function* from executing until the awaited promise settles. So in `const user = await fetchUser(); const permissions = await fetchPermissions();`, the call to `fetchPermissions()` — meaning the point where the network request for it is actually dispatched — doesn't happen until the `fetchUser()` promise has resolved, even though `fetchPermissions` doesn't read anything from `user`. The fix is to call both functions first (which dispatches both requests immediately, synchronously, before either `await` suspends execution) and only then `await` the resulting promises: `const userPromise = fetchUser(); const permissionsPromise = fetchPermissions(); const [user, permissions] = await Promise.all([userPromise, permissionsPromise]);` — the requests fire back-to-back on the same tick, and both are in flight concurrently while the function awaits both.

The trap: saying "await is blocking" without the more precise "blocks the next line of *this* function, not the runtime" — and missing that the actual bug is about *when the request is dispatched* (call time), not when the response is awaited; a request already in flight from an earlier `await`ed call would continue regardless, the waterfall is created by delaying the *call*, not the read.

---

**Q (High): The interviewer says: "Now project's owner field needs to resolve to a full user object, and that requires a second round trip to `/users/{id}` after `/projects` returns." Is this now an unavoidable waterfall, or can it still be improved?**

Answer: This is a genuine dependency (the second call's input — the owner's ID — only exists after the first call's response), so within a pure REST client-side data-fetching model, this specific pair can't be flattened into one parallel round trip — the two stages are real. What *can* still be improved: (1) this two-stage chain doesn't need to block the rest of the page — projects' owner-enriched view can render progressively while other independent resources (user, flags) are already parallel and unaffected; (2) if this "fetch a list, then fetch related entities by ID from the list" pattern recurs across the app, it's a strong signal for either a server-side aggregation endpoint that returns projects pre-joined with owner data (eliminating the second round trip entirely) or, if using GraphQL, a single query that expresses the join declaratively and lets the server resolve it in one round trip; (3) if there are many projects each needing an owner lookup, batch the second stage (`fetchUsers(idsFromAllProjects)`) rather than N individual owner fetches, which is a different, N+1-shaped problem layered on top of the two-stage dependency.

The trap: accepting "it's a real dependency, so nothing more to do" — a genuine two-stage dependency is still often collapsible at the server/API-design layer, and even when it isn't, it shouldn't be allowed to block unrelated parts of the page from rendering, which is a separate, still-solvable axis.

---

**Q (Medium): How would `Promise.all` vs. `Promise.allSettled` change what you show the user if the `permissions` fetch fails but `user`, `projects`, and `flags` all succeed?**

Answer: With `Promise.all`, the entire combined promise rejects the instant `permissions` rejects — even though `user`, `projects`, and `flags` had already resolved successfully (or would have), none of their results are usable from that call site, because `Promise.all`'s resolved value is only produced on full success; the typical result is the whole page falls back to a generic error state despite three-quarters of its data being genuinely available. With `Promise.allSettled`, all four settle independently and the caller gets an array of `{status, value | reason}` for each — the page can render `user`, `projects`, and `flags` normally and show a scoped, local error/fallback specifically where `permissions` would have rendered (e.g., defaulting to a maximally-restrictive permission set, or a small inline "couldn't load permissions, retry" affordance) rather than failing the whole page for one resource's failure.

The trap: treating this as purely a syntax choice — the real distinction is a product/UX decision (does one resource's failure justify failing the whole view, or should failures be scoped to just that resource's UI) that should be made deliberately per resource, not defaulted to whichever API happens to be reached for first.

---

**Q (Medium): If you introduced React Query (or SWR) purely to solve this waterfall, what do you get "for free" beyond parallelization that a hand-rolled `Promise.all` version wouldn't give you?**

Answer: Beyond firing independent queries in parallel by default, a library like React Query gives request deduplication across components (if another part of the page also needs `user`, it reuses the same in-flight request/cache entry rather than firing again), caching with configurable staleness (revisiting the page within the stale window skips the network entirely), built-in retry/backoff on failure, background refetching (e.g., on window refocus) to keep data fresh without a manual polling loop, and per-query loading/error state that composes naturally with progressive rendering — all of which a hand-rolled `Promise.all` orchestration would need to be built and maintained separately, and would tend to get inconsistently reimplemented every time a new page needs similar multi-resource fetching.

The trap: describing the library purely as "does parallel fetching" — parallelization is something `Promise.all` already gives you for free without a dependency; the actual case for the library is the surrounding concerns (dedup, cache, retry, refetch-on-focus, consistent loading-state ergonomics) that a hand-rolled version would otherwise have to reinvent per page.

---

**Q (Low): Would moving this data-fetching to the server (SSR, a loader function, or a BFF endpoint that aggregates all four calls into one) be a better fix than client-side parallelization? What would you need to know to decide?**

Answer: It depends on where the latency actually is and who's making the calls. If the four backend services are each fast and the bottleneck is purely the client making four separate round trips over a high-latency connection (mobile, distant CDN edge), a server-side aggregation layer (BFF) that fans out to the four services *from the server* — where inter-service latency is typically far lower than client-to-server latency — and returns one composed response can turn four client round trips into one, which client-side `Promise.all` parallelization can't achieve (it still requires four separate client-to-server round trips, just concurrent ones instead of sequential). The trade-off is added backend complexity (an aggregation layer to build and own) and a coupling point (that layer now needs updating whenever the page's data needs change) versus the client-only fix, which needed no backend changes at all. I'd want to know: is client-to-server latency the dominant cost (favors BFF), is there already a BFF/gateway layer to extend cheaply, and is this data-shape specific to one page or reused enough elsewhere to justify a shared aggregation endpoint.

The trap: assuming client-side parallelization and server-side aggregation are mutually exclusive alternatives rather than complementary — even with a BFF, the BFF's own internal fan-out to its four upstream services should itself be parallelized, so the same underlying principle (don't sequence independent calls) applies at whichever layer the fan-out actually happens.

---

## Self-Assessment

- [ ] Can explain precisely why `await`-per-line creates a waterfall — in terms of when the request is dispatched, not when it's awaited
- [ ] Can distinguish a genuine data dependency between resources from an accidental one caused by write order or convenience prop-passing
- [ ] Can articulate the `Promise.all` vs. `allSettled` trade-off in terms of partial-failure UX, not just API syntax
- [ ] Can design a data-fetching layer (via hooks or an explicit dependency graph) that makes parallel-by-default the path of least resistance for future additions
- [ ] Can explain what a data-fetching library adds beyond parallelization (dedup, cache, retry, refetch-on-focus)

---
*Next: Cache Invalidation Strategy — once resources are fetched efficiently, the next question is how long to trust what's cached and exactly when to throw it away.*
