# Cache Invalidation Strategy

## Quick Reference

| Strategy | Mechanism | Best For |
|---|---|---|
| Time-based staleness (`staleTime`/TTL) | Data is trusted for N seconds, then refetched on next access | Data that changes on a predictable-ish cadence and where slight staleness is tolerable |
| Explicit invalidation on mutation | A write (POST/PATCH/DELETE) invalidates specific cache keys it affects | Data mutated by this client — the client knows exactly what it just changed |
| Tag/dependency-based invalidation | Cache entries are tagged; invalidating a tag invalidates every entry with it | Many cache entries can be affected by one mutation (e.g., "any list containing this item") |
| Server-driven invalidation (ETags/cache headers, push) | Server tells the client data changed, via conditional requests or a push channel | Data mutated by *other* clients/users, where polling/staleTime alone would be too slow or wasteful |
| Manual/optimistic invalidation | Client immediately updates cache to the new expected value without waiting for a refetch | User-initiated actions where instant feedback matters more than guaranteed server-confirmed accuracy |

## The Scenario

"Our app caches API responses on the client to avoid re-fetching on every navigation. Users are now reporting that after they edit something — say, rename a project — the old name still shows up in other parts of the app: the sidebar list, a breadcrumb, a recently-viewed widget. The edit itself works, the server has the new name, but stale cached data is showing everywhere except the screen they just edited. Design a cache invalidation strategy that fixes this without just disabling caching."

## Clarifying Questions

- **How many distinct places in the app cache a representation of "this project"** — is it one canonical cache entry read everywhere, or does each screen (sidebar, breadcrumb, recently-viewed) independently cache its own copy (e.g., a list endpoint response that happens to embed a denormalized project name)? This is the crux of the bug: if every consumer reads through one cache entry keyed by project ID, invalidating that one key fixes everywhere at once; if five endpoints each embed their own copy of the project's name in five differently-shaped responses, invalidation has to know about all five, which is a fundamentally harder, list-of-affected-keys problem.
- **Is the rename happening from this client, or could another user/tab rename the same project while this client has it cached?** Same-client invalidation is solvable entirely client-side (this client knows exactly what it just mutated). Cross-client staleness (another user renamed it, or the same user in a different tab) requires either polling, a reasonably short `staleTime`, or a push mechanism (WebSocket/SSE invalidation event) — a materially different problem with a materially different fix.
- **What's the acceptable staleness window if perfect, instant, everywhere-consistent invalidation isn't achievable** — is "next time this screen is visited/refocused, it's correct" good enough, or does it need to update instantly across all currently-open, currently-rendered instances (e.g., the sidebar visible right now, without a refresh)? This determines whether cache invalidation (marking data stale for the next read) is sufficient, or whether active cache *update* (pushing the new value into every currently-mounted consumer) is required.
- **Does the app use a data-fetching/cache library already (React Query, SWR, Apollo), or is caching hand-rolled** (a module-level object, `localStorage`, manually-managed `useState` lifted to context)? Determines whether the fix is "use the library's existing invalidation primitives correctly" (usually the right framing, and often the actual bug is *not* using them) versus "design an invalidation scheme from scratch."
- **Is "the old name still shows up" a rendering bug (stale cache never gets read again correctly, even after a full page reload) or a staleness-window bug (it's correct after a refresh/re-navigation, just not the instant after the edit)?** If a hard reload also still shows the old name, that's not a cache invalidation problem at all — it suggests the write itself isn't actually persisting, or a *different* cache layer (CDN, service worker, HTTP cache) is involved, which needs ruling out before assuming this is client-side application-cache invalidation.

## Approach & Trade-offs

**First separate "cache invalidation" from "cache update" — they solve overlapping but distinct problems, and conflating them is the most common shallow answer here.** Invalidation marks cached data as no-longer-trustworthy so the *next* read triggers a fresh fetch; it does nothing for data already rendered on screen in a currently-mounted component that won't re-fetch on its own (e.g., a sidebar list that fetched once on mount and holds its own local `useState` copy — invalidating a shared cache entry it never re-reads from doesn't help it). Update pushes the new, known-correct value directly into every place currently holding stale data, without waiting for anyone to trigger a re-fetch. For "the old name still shows up in the sidebar right now, before I've navigated anywhere," update is required, not just invalidation.

**Since the mutation happens on this client, the client already knows the new value — use optimistic cache update, not just invalidation-then-refetch.** The weakest fix (invalidate the sidebar's cache entry after the rename succeeds, then wait for whatever eventually re-triggers a fetch) leaves a window where the sidebar is invalidated-but-not-yet-refetched, so it can still show stale data until something causes it to re-render. The strongest fix writes the new name directly into every cache entry that includes this project's name, synchronously with the mutation response — no refetch required, no stale window, and it works even for entries a naive invalidation approach wouldn't know to target if it's inferring "what needs invalidating" from a URL pattern rather than data identity.

**The actual design problem is knowing *which* cache entries need updating when one entity changes — this is where a shared, normalized cache pays off over per-endpoint caching.** If the sidebar list, breadcrumb, and recently-viewed widget each cache their own independently-fetched, denormalized response (each embedding a copy of `project.name`), the invalidation/update logic has to explicitly know about all three response shapes and update each one's copy — brittle, and guaranteed to miss a fourth place the next screen introduces. A normalized cache (Apollo's normalized store, or React Query used with query-key structures keyed by entity ID plus a query-invalidation strategy keyed on entity type) keeps exactly one canonical copy of "project #123" and has every consumer — sidebar, breadcrumb, recently-viewed — read *through* that one entry rather than holding their own copy, so updating it once updates every consumer simultaneously, without needing to enumerate them.

**Trade-off: normalized caching isn't free — it requires consistent entity identification and adds a layer of indirection that's harder to reason about than "this endpoint's response, cached as-is."** For a small number of screens or infrequently-mutated data, per-endpoint invalidation lists (explicitly invalidating known-affected query keys on mutation) is simpler to write and reason about, at the cost of needing manual upkeep every time a new screen adds a new place this entity's data appears. I'd frame this as: normalize when an entity is genuinely denormalized across many independent views (this scenario's exact symptom); keep it simple/explicit when an entity only ever appears in one or two places.

## Solution

**Step 1 — the buggy version: mutate, but only invalidate the screen the user is on.**

```tsx
function useRenameProject() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (vars: { id: string; name: string }) => renameProject(vars),
    onSuccess: (_, vars) => {
      // BUG: only invalidates this one query — sidebar, breadcrumb, and
      // recently-viewed each have their OWN cache entries that also
      // embed this project's name, and none of them get invalidated
      queryClient.invalidateQueries({ queryKey: ['project', vars.id] });
    },
  });
}
```

**Step 2 — the invalidation-list fix: explicitly enumerate every affected query key.** Works, but requires this list to be kept in sync by hand every time a new screen caches project data:

```tsx
onSuccess: (_, vars) => {
  queryClient.invalidateQueries({ queryKey: ['project', vars.id] });
  queryClient.invalidateQueries({ queryKey: ['sidebar-projects'] });
  queryClient.invalidateQueries({ queryKey: ['breadcrumbs', vars.id] });
  queryClient.invalidateQueries({ queryKey: ['recently-viewed'] });
},
```

This also only *invalidates* (marks stale for next read) — anything currently mounted and rendered won't visually update until it happens to re-render/re-fetch.

**Step 3 — the update fix: since the mutation's response already contains the new name, write it directly into every cache entry that embeds it, synchronously, with no dependency on a refetch happening at all.**

```tsx
function useRenameProject() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (vars: { id: string; name: string }) => renameProject(vars),
    onSuccess: (updatedProject, vars) => {
      // Update the canonical entry directly.
      queryClient.setQueryData(['project', vars.id], updatedProject);

      // Update every list/derived cache entry that embeds a copy of
      // this project's name, using each cache's own updater function
      // rather than waiting for a refetch.
      queryClient.setQueryData(['sidebar-projects'], (old?: Project[]) =>
        old?.map((p) => (p.id === vars.id ? { ...p, name: vars.name } : p))
      );
      queryClient.setQueryData(['recently-viewed'], (old?: Project[]) =>
        old?.map((p) => (p.id === vars.id ? { ...p, name: vars.name } : p))
      );
    },
  });
}
```

Every consumer subscribed to any of these query keys — including a sidebar list currently mounted and rendered — re-renders immediately with the new name, with zero network round trips and zero stale window.

**Step 4 — where the entity appears in places too numerous or too dynamic to enumerate safely (Step 3's per-key list still doesn't scale to an arbitrary number of screens), normalize instead — every consumer reads through one canonical entity cache, so there's nothing to enumerate:**

```tsx
// Consumers select from a normalized entity store rather than
// holding their own denormalized copy.
function useProjectName(id: string) {
  return useEntityStore((state) => state.projects[id]?.name);
}

function renameProject(id: string, name: string) {
  return api.patch(`/projects/${id}`, { name }).then((updated) => {
    useEntityStore.getState().upsertProject(updated);
    // every useProjectName(id) subscriber anywhere in the tree
    // re-renders with the new name — nothing to enumerate
  });
}
```

> **Check yourself:** If the recently-viewed widget's list came from a *different* backend service than the sidebar's project list (so the "project" shape isn't identical between them — one has `name`, the other has `title`), would normalized caching still work as cleanly? What would need to be true first?

## Cross-client Staleness — When Someone Else Made the Change

Everything above assumes this client made the edit. If another user (or another tab) renamed the project, this client has no mutation response to update its cache from — it needs to *learn* the data changed:

- **Short `staleTime` + refetch-on-focus/refetch-on-reconnect** — cheap, works with existing polling-style infrastructure, but bounded by the staleTime window (data can be stale for up to that window even after a genuine change) and only refreshes when the user does something (refocus, navigate) that triggers a re-check.
- **Server push (WebSocket/SSE) carrying invalidation events, not just raw data** — the server emits `{ type: 'invalidate', entity: 'project', id: '123' }` on any write, and every connected client invalidates/refetches that specific cache entry the moment it's told to, closing the staleness window to roughly the push-channel's latency rather than a polling interval. More infrastructure to build and keep connected, but the only approach that gets genuinely near-real-time cross-client consistency.
- **Server push carrying the new *value*, not just an invalidation signal** — skips the refetch round trip entirely (same principle as the optimistic-update fix above, but driven by a push event instead of this client's own mutation response) at the cost of a larger, more failure-prone payload over the push channel and a need to reconcile push-delivered updates with whatever's already in the local cache (ordering, partial updates).

## Gotchas

**Invalidating a query key but the consumer that shows the stale data isn't actually subscribed to that key** — e.g., the sidebar rendered a list once via a component-local `useState` populated from an initial fetch, never re-reading from the shared cache at all. Invalidation only helps things that *read from* the cache being invalidated; a component holding its own disconnected copy needs to be refactored to read from the shared cache first, or it needs its own explicit update path.

**Treating invalidation as sufficient when the actual requirement is instant visual update.** Invalidation without an accompanying refetch (or without the consumer being currently subscribed and re-rendering) can leave stale data on screen indefinitely if nothing happens to trigger a re-read — "it'll be correct next time it fetches" is not the same guarantee as "it's correct right now."

**Enumerating affected query keys by hand (Step 2) and missing one** — this is exactly how the bug in the scenario prompt happened in the first place (three separate places, one presumably had invalidation added when it was built, the others didn't). An enumerated invalidation list is a maintenance liability that silently drifts out of sync as new screens are added — worth calling out as the reason a normalized cache or update-by-entity-ID pattern is the more durable answer, even though it's more work upfront.

**Optimistically updating the cache with the mutation's *variables* (what was sent) rather than its *response* (what the server actually persisted)** — if the server applies additional transformation (trimming, capitalization, a slug regenerated from the name), updating the cache from the optimistic input rather than waiting for/using the actual server response can leave the client showing something subtly different from what's actually stored, invisible until the next real fetch corrects it.

**Cross-client staleness handled purely by shortening `staleTime` to something aggressive (e.g., 0)**, effectively disabling caching to sidestep the invalidation design problem — this defeats the entire reason caching existed (avoiding redundant refetches) and re-introduces the waterfall/redundant-request problems that caching was meant to solve, without actually closing the staleness window to zero (there's still a window between another client's write and this client's next fetch, however short the interval).

## Follow-up Questions

**Q (High): What's the concrete difference between "cache invalidation" and "cache update," and why does this scenario specifically need the latter, not just the former?**

Answer: Invalidation marks a cache entry as stale/untrustworthy so that the *next* read of it triggers a fresh network fetch — it's a promise about future reads, not an immediate change to what's currently rendered. Update writes a new value directly into the cache entry (and, transitively, into every component currently subscribed to and rendering that entry) synchronously, with no network round trip and no dependency on a future read happening. This scenario's specific symptom — "the old name still shows up... right after I edited it" — is about data currently on screen in already-mounted components (the sidebar, breadcrumb, recently-viewed widget), which invalidation alone doesn't touch; those components won't magically re-render just because their cache entry was marked stale, they'd need to actually re-fetch and re-render, which might not happen until an unrelated re-render or navigation. Since the mutating client already has the authoritative new value (from the mutation's own response), directly updating every affected cache entry closes that gap immediately, with update being strictly better than invalidate-and-wait whenever the new value is already known.

The trap: reaching immediately for `invalidateQueries` (the more commonly-known API) as the complete fix, without recognizing that invalidation's "stale, refetch next time" semantics don't guarantee an immediate visual correction for already-rendered UI — which is exactly the bug being reported.

---

**Q (High): Design the invalidation strategy for the case where the rename happens on a *different* client (another tab, or another user) — walk through what changes and why polling/staleTime alone might not be good enough.**

Answer: The mutating-client's own optimistic-update approach doesn't apply at all here, because this client never made the mutation and has no response to update its cache from — it has to *learn* the data changed from an external signal. The weakest option is a short `staleTime` combined with refetch-on-window-focus/refetch-on-reconnect, which is simple and needs no new infrastructure, but bounds staleness to however long the window is and only actually refreshes on a triggering event (refocus, reconnect, manual navigation) — a user staring at an already-focused tab won't see the update until something re-triggers a fetch, however short staleTime is set to. For genuinely near-real-time cross-client consistency, the server needs to actively push an invalidation (or update) signal over a persistent channel (WebSocket/SSE) the moment the write happens — every connected client, including ones sitting idle on an already-open tab, receives the signal and invalidates/refetches (or applies the pushed value) immediately, closing the staleness window to roughly the push channel's latency instead of a polling/staleTime interval. The trade-off is real infrastructure — a push channel, server-side fan-out to all connected clients interested in that entity, and reconciliation logic for out-of-order or missed push events (e.g., a client that was briefly disconnected needs a way to catch up, not just rely on future pushes).

The trap: assuming aggressive polling (very short `staleTime`, frequent refetch intervals) is an acceptable substitute for push-based invalidation — it trades staleness for load (many redundant "nothing changed" refetches) rather than actually solving cross-client near-real-time consistency, and still has a floor on staleness equal to the poll interval.

---

**Q (High): Why is normalized caching (one canonical entry per entity, read through by every consumer) considered more durable than enumerating affected query keys on every mutation? What's the cost of normalizing?**

Answer: Enumerating affected keys (Step 2 in the Solution) requires the developer writing the mutation to know, at mutation-write time, every current place in the app that caches a copy of this entity's data — a list that's correct today but silently goes stale the moment a new screen is added that also caches this entity, unless that screen's author remembers to go add their new query key to every relevant mutation's invalidation/update list elsewhere in the codebase, which is exactly the kind of cross-cutting manual bookkeeping that reliably drifts out of sync in a real codebase (this scenario's bug is a textbook example of that drift). A normalized cache inverts the dependency: every consumer reads through one canonical entry per entity ID rather than holding its own denormalized copy, so updating that one entry updates every current and future consumer automatically — a new screen added later that reads the same entity ID gets correct, live data for free, with zero changes needed to any existing mutation code. The cost is real: normalization requires consistent entity identification (every response needs to be decomposable into entities with stable IDs), adds an indirection layer that's less immediately readable than "this endpoint's response, as returned, cached as-is," and doesn't help with data that's genuinely computed/aggregated per-view rather than a denormalized copy of one entity (e.g., a "top 5 most active projects this week" list isn't a normalization candidate in the same way a single project's name is).

The trap: treating normalization as an unconditional best practice to reach for regardless of scale — for an entity that only ever appears in one or two places, explicit per-mutation invalidation is simpler to read, simpler to debug, and not actually at meaningful risk of the drift problem that justifies normalization for a widely-denormalized entity like this scenario's project name.

---

**Q (Medium): The mutation optimistically updates the cache using the values the user typed, before the server confirms the request succeeded. The request then fails. What needs to happen, and what's the general pattern for this?**

Answer: This is optimistic *mutation*, not just optimistic *update-after-success* — it requires capturing the cache's prior state before applying the optimistic change, so that a failure can roll back to exactly that prior state rather than leaving the cache in an inconsistent, silently-wrong condition. The general pattern (React Query's `onMutate`/`onError`/`onSettled` lifecycle is a direct implementation of it): in `onMutate`, snapshot the current cache value for the affected key(s), then apply the optimistic change immediately; in `onError`, restore the snapshotted value, undoing the optimistic change; in `onSettled` (runs on either success or failure), invalidate/refetch the affected keys to reconcile with whatever the server's authoritative state actually is, catching any subtle mismatch between the optimistic guess and reality (e.g., server-side transformation) even on the success path. Skipping the rollback step (only handling the happy path) means a failed mutation leaves the UI showing a change that never actually persisted, with no visual indication anything went wrong beyond whatever error toast fired separately — a correctness bug on top of a UX one.

The trap: implementing the optimistic update but not the rollback, treating "the request usually succeeds" as good enough — the scenario explicitly becomes a data-integrity bug (UI shows unpersisted state as if it were persisted) the moment a request fails, which is exactly the case optimistic UI needs to handle correctly, not just the happy path.

---

**Q (Medium): How would ETags or `If-None-Match`/conditional requests factor into a cache invalidation strategy, and how is this different from the application-level cache invalidation discussed above?**

Answer: ETags operate at the HTTP caching layer, below the application's own in-memory cache (React Query's cache, a normalized store, etc.) — they let the client ask the server "has this resource changed since the version I have, identified by this ETag?" via a conditional request, and the server responds with a cheap `304 Not Modified` (no body) if unchanged, or the full new representation with a new ETag if it has. This is complementary to, not a replacement for, application-level invalidation: ETags reduce the *cost* of a refetch that turns out to be unnecessary (a `304` is far cheaper than re-transferring and re-parsing a full payload), but the application still needs to decide *when* to issue that conditional request in the first place — the same staleTime/refetch-on-focus/push-driven triggers discussed above still apply; ETags don't tell the client anything changed unless the client asks. Where ETags shine is exactly the cross-client staleness case: a client can afford to poll (conditionally) far more aggressively than it otherwise would, because most of those polls cost a cheap `304` rather than a full payload transfer, softening the trade-off between staleness and network cost.

The trap: presenting ETags as an invalidation *mechanism* (something that tells the client when to refetch) rather than an *optimization* on refetches the application has already decided to make — the decision of when to check is still the application's cache strategy; ETags just make checking cheap.

---

**Q (Low): If this were a server-rendered page (not a client-side SPA with an in-memory cache), would "cache invalidation" mean something different? What layers would be involved?**

Answer: Yes — server-rendering introduces additional cache layers between the data source and the browser that a pure client-side SPA cache doesn't have to consider: a CDN or reverse-proxy cache in front of the server-rendered HTML (which needs its own invalidation, typically via a cache-purge API call or a short TTL plus cache-tag-based purging keyed to the entity that changed), the server's own data-layer cache if it fetches from a backend service and caches that response before rendering HTML (same invalidation problem, one layer removed from the browser), and the browser's own HTTP cache for the rendered document and its assets (governed by `Cache-Control` headers, which need to be set appropriately — e.g., `no-cache` or a short max-age for pages showing mutable data like this project's name, versus long-lived immutable caching for hashed static assets). A rename on this architecture would need to purge/invalidate at every one of these layers that could be holding a stale copy — missing the CDN layer, for instance, would mean even a hard browser refresh still serves a stale cached HTML response from the edge, which is a materially different failure mode than anything discussed above (all of which assumed a single client-side application cache).

The trap: assuming "cache invalidation" only ever refers to the client-side application cache discussed throughout this scenario — in an SSR/CDN architecture, that's one layer among several, and each layer needs its own invalidation trigger; fixing the application cache correctly while a CDN in front of it still serves a stale cached page leaves the user-visible bug completely unresolved.

---

## Self-Assessment

- [ ] Can explain the difference between cache invalidation and cache update, and identify which this scenario's symptom actually requires
- [ ] Can design an optimistic-update-with-rollback flow using a snapshot/apply/rollback/reconcile pattern
- [ ] Can explain why cross-client staleness needs a different mechanism (push or polling) than same-client mutation does (direct cache write)
- [ ] Can articulate the trade-off between per-mutation invalidation lists and normalized entity caching
- [ ] Can place ETags/conditional requests correctly as a refetch-cost optimization, not an invalidation-trigger mechanism
- [ ] Can name the additional cache layers (CDN, server data cache, browser HTTP cache) relevant in an SSR architecture beyond the client-side application cache

---
*Next: Pagination Data Model: Cursor vs. Offset — once cached data can be trusted to stay fresh, the next question is how to paginate through it correctly when the underlying dataset is itself changing.*
