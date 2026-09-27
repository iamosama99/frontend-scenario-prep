# Redundant Network Requests on a Page

## Quick Reference

| Redundancy Source | How You Spot It | Fix |
|---|---|---|
| Same GET fired multiple times (no cache/dedup) | Network panel shows identical URL+method repeated in one page load | In-flight request dedup + response cache (React Query/SWR, or a manual cache map) |
| Multiple components independently fetching the same resource | Waterfall shows N identical requests fired from N different mount points | Lift the fetch to a shared parent/cache layer; one source of truth per resource |
| `useEffect` re-fetching on every render due to a missing/unstable dependency | Requests keep firing well past initial load, correlating with re-renders | Fix the dependency array; memoize objects/functions passed as deps |
| Over-fetching — requesting more than the view needs | One request returns a huge payload when only a few fields are rendered | GraphQL field selection, sparse fieldsets, a narrower REST endpoint |
| Polling that doesn't back off or dedupe with a manual refresh | Poll interval fires alongside a user-triggered refetch for the same data | Coordinate poll + manual refetch through one request-management layer |

## The Scenario

"Someone looked at the Network tab on our dashboard page and counted the same `/api/user/profile` endpoint being called four times on a single page load. Nothing about the page looks broken to a user, but that's clearly wasteful. Find every redundant request on this page, explain why each one is happening, and fix them without changing what the user sees."

## Clarifying Questions

- **Are the four calls truly identical (same URL, method, and params) or do they differ in some argument that isn't obvious from a glance** (a different query param, a cache-busting timestamp, different auth headers)? Genuinely identical requests are a pure dedup problem; near-identical-but-subtly-different ones might indicate several *different* features each independently needing "the profile," which is more of an architectural/data-layer question than a simple caching fix.
- **Are the four calls near-simultaneous (fired within the same tick/render, suggesting independent components each triggering their own fetch on mount) or spread out over the page's lifecycle (suggesting re-fetches triggered by re-renders, route changes, or polling)?** Near-simultaneous duplicate calls point at multiple independent components each owning their own fetch logic for the same resource — the fix is architectural (shared cache/query layer). Spread-out re-fetches point more at an effect dependency bug or an intentional-but-uncoordinated polling/refetch pattern.
- **Is there already a data-fetching library in use (React Query, SWR, Apollo, RTK Query) or is fetching done with raw `fetch`/`axios` calls scattered through components?** If a caching data-fetching library is already present, four duplicate calls to the same key is almost certainly a bug (a missing/inconsistent cache key, or bypassing the library with a raw fetch somewhere) rather than an architectural gap — the fix is narrow. If there's no shared data layer at all, this is one symptom of a broader missing-abstraction problem, and the fix is closer to "introduce one" than "patch four call sites."
- **Does fixing this need to preserve independent loading/error states per component, or can all four consumers safely share one in-flight request and its result?** If the four call sites have genuinely different loading/error-handling needs (one shows a skeleton, another silently retries), a shared cache still works but needs to preserve each consumer's own subscription to loading state, not just merge everything into one shared boolean — worth confirming before assuming a single dedup fix drops in cleanly.
- **Is this specifically about count (four calls that could be one) or does the endpoint's response also matter — e.g., is the same large payload being over-fetched even in the "correct," deduplicated single-call version?** Deduping four identical calls to one doesn't help if that one remaining call still returns 10x more data than any consumer actually uses — worth treating as a related but separate axis of "redundant" (redundant calls vs. redundant bytes within each call).

## Approach & Trade-offs

**Diagnose the specific cause per duplicate before applying a single generic fix — "four calls" can come from at least three structurally different bugs with different remedies.** (1) Multiple independent components each call the same endpoint on their own mount, with no shared cache — the fix is a shared request/cache layer so the second, third, and fourth callers get the first caller's in-flight promise or cached result instead of issuing a new request. (2) A single component's effect has an unstable dependency (a new object/array/function literal recreated every render) causing it to re-run and re-fetch on every render — the fix is dependency-array hygiene, not caching. (3) The same data is fetched by design in multiple, seemingly-unrelated features that don't know about each other (e.g., a header avatar and a settings panel both independently need "the user profile") — the fix is recognizing these as the same logical resource and routing both through one shared data-access point, which is as much an architectural/team-coordination fix as a code one.

**A dedicated data-fetching library (React Query/SWR-style) solves the *general* class of problem, not just this one page — worth advocating for as the durable fix rather than four bespoke patches.** These libraries key requests by a cache key (typically the URL + params), automatically deduplicate concurrent requests for the same key (a second caller within the same "stale" window gets the first request's in-flight promise, not a new network call), and cache results so subsequent mounts requesting the same key reuse cached data instead of re-fetching. Introducing this as the standard data-fetching pattern (rather than raw `fetch` scattered through components) prevents this exact bug class from recurring on other pages, which matters more than deduplicating these four specific call sites by hand.

**Where introducing a full library isn't the right call for this task's scope** (e.g., a small, narrowly-defined bug fix rather than an app-wide data-layer migration), a minimal manual in-flight request cache achieves the same dedup property for the specific endpoint in question, as a targeted fix — trading generality for a smaller, more reviewable change. I'd state this as an explicit trade-off: the manual cache fixes this endpoint; the library fixes this *class* of bug everywhere, at the cost of a larger, riskier change to land.

**Fixing duplicate requests must not change user-visible behavior — each consumer's loading/error state needs to keep working correctly even when sharing one underlying request.** A naive fix that makes only the *first* caller show a loading spinner while later callers silently get stale/undefined data during the shared request's flight would be a regression dressed as an optimization; the fix needs each consumer to still individually subscribe to the shared request's state (loading/success/error) even though only one network call is actually in flight.

## Solution — the diagnostic + fix walkthrough

**Step 1 — confirm what's actually different (or not) between the four calls.** Network panel, filter to `/api/user/profile`, inspect each request's initiator (DevTools' "Initiator" column/stack trace) and exact params/headers.

```
Call 1: initiator → <Header /> mount,        params: {} 
Call 2: initiator → <ProfileWidget /> mount, params: {}
Call 3: initiator → <SettingsPanel /> mount, params: {}
Call 4: initiator → <Header /> re-render,    params: {}  (fired again ~2s later)
```

Three components independently fetching the identical resource on mount (calls 1-3), plus one re-fetch from a component that fired again later (call 4) — two distinct bugs.

**Step 2 — find the re-fetch bug (call 4) in the code.**

```tsx
// BUG: `options` is a new object literal every render, so this effect
// re-runs on every render, re-fetching every time
function Header({ userId }: { userId: string }) {
  const [profile, setProfile] = useState(null);
  const options = { userId, includeAvatar: true }; // new reference every render

  useEffect(() => {
    fetchProfile(options).then(setProfile);
  }, [options]); // unstable dependency — "changes" every render

  return <div>{profile?.name}</div>;
}
```

Fix — stabilize the dependency:

```tsx
function Header({ userId }: { userId: string }) {
  const [profile, setProfile] = useState(null);

  useEffect(() => {
    fetchProfile({ userId, includeAvatar: true }).then(setProfile);
  }, [userId]); // only re-fetch when userId actually changes

  return <div>{profile?.name}</div>;
}
```

**Step 3 — fix the independent-duplicate-fetch bug (calls 1-3) by introducing a shared, deduplicated data layer.** Using React Query as the shared abstraction:

```tsx
function useProfile(userId: string) {
  return useQuery({
    queryKey: ['profile', userId],
    queryFn: () => fetchProfile({ userId, includeAvatar: true }),
    staleTime: 60_000, // treat data as fresh for 60s — no refetch within that window
  });
}

// Each component calls the same hook independently...
function Header({ userId }: { userId: string }) {
  const { data: profile } = useProfile(userId);
  return <div>{profile?.name}</div>;
}
function ProfileWidget({ userId }: { userId: string }) {
  const { data: profile, isLoading } = useProfile(userId);
  return isLoading ? <Skeleton /> : <ProfileCard profile={profile} />;
}
function SettingsPanel({ userId }: { userId: string }) {
  const { data: profile } = useProfile(userId);
  return <SettingsForm initialName={profile?.name} />;
}
```

...but because all three share the same `queryKey`, React Query deduplicates: if all three mount within the same tick, only *one* network request fires — the other two subscribe to that same in-flight request's result — and each component still independently reads its own `isLoading`/`data` from the shared cache entry, preserving correct per-component loading states without a second network call.

**Step 4 — check for the over-fetching axis separately.** Even one deduplicated call might return more than any consumer uses — if `fetchProfile` returns a large payload (order history, full settings blob) when only `name` and `avatarUrl` are ever rendered on this page, narrow the endpoint/query to just those fields.

**Step 5 — re-verify in the Network panel**, confirming exactly one `/api/user/profile` request per distinct `userId` for the page's full lifecycle (not per component), and confirm each of the three consumers still renders correctly (loading skeletons still appear/disappear as expected, since this was a UX-preservation requirement).

> **Check yourself:** If `SettingsPanel` needed *fresher* data than `Header`/`ProfileWidget` (say, it must reflect a just-saved change instantly, bypassing the 60-second `staleTime`), how would you express that without breaking the shared caching benefit for the other two consumers?

## Gotchas

**Fixing the effect-dependency bug (call 4) and declaring the investigation done, missing that calls 1-3 are a structurally different bug.** "Four duplicate calls" reads as one problem but is frequently two or more distinct root causes that happen to produce the same symptom — a full fix requires diagnosing each initiator separately, not applying one dependency-array fix and assuming the count will drop to one.

**Introducing a shared cache/dedup layer but with an unstable or inconsistent cache key across call sites**, e.g. one component calling `useProfile(userId)` and another accidentally passing `useProfile({ userId })` (an object, not the primitive) — if the key isn't normalized consistently, the "same" logical resource ends up under different cache entries, silently defeating the dedup the library was introduced to provide.

**Deduplicating requests but not accounting for genuinely different staleness needs across consumers**, forcing either an overly long `staleTime` (some consumers see data staler than they need) or overly short/zero (defeats the dedup benefit for the others) — a shared cache needs per-consumer override mechanisms (most libraries support per-call staleness/refetch options against the same key) rather than one blanket setting for every use of a shared resource.

**Assuming request count is the only relevant "redundant" axis and ignoring payload size.** Reducing four calls to one is a real win, but if that one remaining call still returns a needlessly large response (unused fields, full nested objects where only an ID is needed), a meaningful amount of the original waste is still present — worth explicitly checking, not just counting requests before/after.

**A poll interval and a manually-triggered refetch (e.g., a "refresh" button) both hitting the same endpoint independently, occasionally overlapping.** This looks like the same class of bug as the mount-time duplicates but has a different fix — coordinating both through the same request-management layer (so a manual refetch resets/skips the next poll tick rather than firing alongside it) rather than deduplicating concurrent identical requests alone, which doesn't address the *scheduling* overlap between two different triggers.

## Follow-up Questions

**Q (High): Explain precisely how a library like React Query deduplicates concurrent requests for the same resource — what makes two calls "the same" from its perspective, and what happens to the second caller?**

Answer: The library identifies a request by its cache key (in React Query, the `queryKey` array — typically something like `['profile', userId]`), not by anything about which component or call site issued it — any number of components calling `useQuery` with a structurally-equal key are, from the library's perspective, all subscribers to the *same* logical query. When a query with a given key is already in-flight (a request was dispatched and hasn't resolved yet), a second call with the same key doesn't dispatch a new network request at all — it registers as an additional subscriber to the already-in-flight promise, and when that promise resolves, all subscribers (both the original caller and every deduped one) receive the same result and update their own component state/re-render accordingly. Once resolved, the result is cached under that key for a configurable `staleTime`; further calls with the same key within that window return the cached data synchronously without any network request, in-flight or otherwise, until the data is considered stale again.

The trap: describing this as "it caches responses" without the concurrent in-flight-request-sharing mechanism specifically — the four-simultaneous-calls scenario in this exact prompt is about *concurrent* dedup (no cache entry exists yet for any of them), which is a distinct mechanism from post-resolution caching, and conflating the two misses why near-simultaneous mount-time duplicates get deduped at all.

---

**Q (High): Two components call the same data-fetching hook with what the developer believes is "the same" resource, but React Query still fires two separate network requests. What's the most likely reason, and how would you confirm it?**

Answer: The most likely reason is that the two call sites' `queryKey`s aren't actually structurally equal, even if they're conceptually referring to the same resource — a very common version of this is one call site passing a primitive (`['profile', userId]`) and another passing an object or a differently-ordered/shaped key (`['profile', { id: userId }]`, or `['profile', userId, 'v2']` with an extra segment added by one call site but not the other), which the library's key-equality check (typically a deep/structural comparison) treats as genuinely different queries, each maintaining its own independent cache entry and in-flight state. I'd confirm by using the library's devtools (React Query has a dedicated DevTools panel) to inspect the actual registered query keys for both call sites side by side — a key mismatch is immediately visible there, more reliably than reading through both call sites' source and assuming they match.

The trap: assuming a shared cache/query library is being deliberately bypassed (a raw `fetch` snuck in somewhere) when a much more common and easy-to-miss cause is a subtly inconsistent key shape across call sites that defeats the library's own dedup mechanism from the inside — checking key equality directly (via devtools) is faster and more conclusive than guessing.

---

**Q (High): A "redundant" request is deduplicated down to one, but that one request now needs to satisfy the loading-UX needs of three different consumers with different requirements (one wants a skeleton, one wants silent background refresh with no visible loading state, one needs to know specifically if this was a cache hit vs. a fresh fetch). Is a single shared boolean `isLoading` sufficient? How would you handle this?**

Answer: A single shared `isLoading` boolean is not sufficient for divergent per-consumer UX needs — most data-fetching libraries expose a richer status set precisely for this reason (React Query, for example, distinguishes `isLoading` — no data yet, genuinely first fetch — from `isFetching` — a request is in flight, but possibly with existing cached data still being shown — and `isPlaceholderData`/`dataUpdatedAt` for more granular "is this stale/fresh" checks). The consumer wanting a skeleton would key off `isLoading` (no data at all yet), the one wanting silent background refresh would ignore `isFetching` and just render existing `data` even while a refetch is in flight, and the one needing fresh-vs-cached awareness would check `isFetching` alongside whether `data` was already present before this fetch started. This works because the underlying shared request is still just one network call — but each consumer derives its own UI treatment from the richer state object the library exposes, rather than the library flattening everything to one boolean that can't satisfy all three needs simultaneously.

The trap: treating "shared request" as requiring "shared, identical loading UI" — the network-level deduplication and the per-consumer presentation logic are separate concerns, and a good data-fetching library's API is specifically designed to let one underlying request serve several different UI treatments without each consumer needing its own separate network call to get its own UI logic right.

---

**Q (Medium): Write the effect-dependency bug from Step 2 in a way that's harder to spot than an inline object literal — e.g., involving a function passed as a prop instead — and explain why it's still the same underlying bug.**

Answer:

```tsx
// Parent re-creates `onProfileLoaded` on every render (no useCallback)
function Dashboard({ userId }: { userId: string }) {
  const onProfileLoaded = (profile) => { console.log('loaded', profile); };
  return <Header userId={userId} onProfileLoaded={onProfileLoaded} />;
}

function Header({ userId, onProfileLoaded }: Props) {
  useEffect(() => {
    fetchProfile({ userId }).then(onProfileLoaded);
  }, [userId, onProfileLoaded]); // onProfileLoaded is a new reference every parent render
  // ...
}
```

This is the identical underlying bug (an unstable reference in the dependency array causing the effect to re-run more often than intended) but harder to spot because the unstable value crosses a component boundary — reviewing `Header` in isolation, the dependency array looks reasonable (`userId` and the callback it uses), and the actual instability is only visible by also checking how `onProfileLoaded` is defined in the *parent*. The fix is either wrapping `onProfileLoaded` in `useCallback` in the parent (fixing it at the source) or, if `Header`'s effect doesn't actually need to re-run when the callback identity changes (it just needs to *call* whatever the current callback is), reading it via a ref inside the effect instead of putting it in the dependency array.

The trap: reviewing only the component with the bug's visible symptom (`Header`'s effect) and missing that the actual instability originates from a parent component's render — this is exactly the kind of cross-component dependency-array bug that's easy to miss without either tracing prop origins or using a lint rule (`react-hooks/exhaustive-deps`) that at least flags the dependency is present, even if it doesn't diagnose *why* it's unstable.

---

**Q (Medium): Should polling and a user-triggered "refresh" button hitting the same endpoint be considered "redundant requests" in the same sense as the four duplicate mount-time calls in this scenario? Why or why not, and how would you fix overlap between them?**

Answer: They're a related but distinct category — the four mount-time duplicates in this scenario are genuinely unnecessary (the same data, requested needlessly many times, for no additional information gained), whereas polling and a manual refresh are each individually *intentional* (polling for freshness on a cadence, a manual refresh for on-demand freshness) — the actual waste only occurs in the *overlap* case, where both happen to fire close together and one of them becomes redundant relative to the other in that specific moment. The fix isn't "deduplicate concurrent identical requests" in the generic caching sense (which handles truly simultaneous calls) so much as coordinating the two triggers through one request-scheduling layer — e.g., a manual refresh resets the poll timer (so the next scheduled poll doesn't fire immediately after a just-completed manual one) and, if a poll happens to already be in-flight when the user clicks refresh, the manual trigger reuses that in-flight request rather than firing a second one, which most data-fetching libraries' `refetch()` methods handle correctly by default if both go through the same shared query key.

The trap: treating this as identical to the mount-time-duplicates fix and assuming a generic cache/dedup layer automatically solves *all* forms of "too many requests" — concurrent-identical-request dedup and scheduling-overlap coordination are related but separate mechanisms, and a fix aimed only at the former can still leave poll/manual-refresh overlap unaddressed if the two aren't wired through the same underlying query/trigger coordination.

---

**Q (Low): If this page's four duplicate calls were each going to a different microservice behind an API gateway rather than the same monolith endpoint, would client-side deduplication alone be a complete fix, or would you also look at the backend?**

Answer: Client-side deduplication would still eliminate the *redundant* client-issued calls (the actual bug being fixed here), but if the underlying data genuinely needs to be assembled from several backend calls per logical "page load need" regardless of client-side request count, that's a separate, backend-facing concern — worth considering whether an API gateway/BFF (backend-for-frontend) layer that aggregates what the client currently has to request as multiple separate calls into one composed response would reduce total round trips further, independent of client-side caching. This matters more when the redundant calls are hitting genuinely different downstream services (each with its own latency/cost) rather than one shared cache being defeated — client-side fixes address "don't ask twice for the same thing," while a BFF/aggregation layer addresses "don't make the client orchestrate multiple round trips for one logical view's data needs" — different problems that can both be present in a more complex, multi-service backend than this scenario's single monolith endpoint implies.

The trap: assuming client-side deduplication is a universal fix regardless of backend topology — it fully solves genuine duplication of *identical* requests, but doesn't address a legitimately-necessary fan-out to multiple distinct backend services, which needs a server-side aggregation answer instead.

---

## Self-Assessment

- [ ] Can distinguish the three structurally different causes of "duplicate requests" (independent-component fetch, unstable effect dependency, uncoordinated polling/refresh) and give the correct fix for each
- [ ] Can explain precisely how a data-fetching library deduplicates concurrent in-flight requests by cache key, not just "it caches results"
- [ ] Can diagnose a cache-key mismatch as the cause of dedup silently not working, and know to check it via devtools rather than by re-reading source
- [ ] Can design a shared-request fix that still preserves each consumer's independent loading/error UX
- [ ] Can identify request-count reduction and payload-size reduction as separate, both-worth-checking axes of "redundant"

---
*Next: Long Tasks Blocking the Main Thread — from wasted network round trips to wasted main-thread time, diagnosing and breaking up the specific JS execution patterns that make a page feel frozen during otherwise-normal interaction.*
