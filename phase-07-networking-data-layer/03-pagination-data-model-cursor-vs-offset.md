# Pagination Data Model: Cursor vs. Offset

## Quick Reference

| Model | Mechanism | Breaks When |
|---|---|---|
| Offset/limit (`?page=3&limit=20`) | Server skips N rows, returns the next `limit` | Underlying data is inserted/deleted between page loads — rows shift, causing skips/duplicates |
| Cursor-based (`?after=<opaque-token>&limit=20`) | Server returns rows strictly after a stable reference point (typically an ID or composite sort key) | Needs a total-count or "jump to page 7" UI — cursors are inherently sequential, not random-access |
| Keyset pagination (cursor using actual sort column values, e.g. `?after_created_at=...&after_id=...`) | Same idea as cursor, but the cursor is derived from real, indexed columns rather than an opaque server-issued token | Requires the sort to be over indexed, stable columns; ties in the sort key need a tiebreaker column |

## The Scenario

"We paginate a live activity feed with `?page=N&limit=20`. QA reported that if new items get added to the feed while someone is on page 2, by the time they load page 3 they either see an item repeated that they already saw on page 2, or they skip an item entirely — it depends on whether items were added or removed. Explain why this happens, and redesign the pagination so it's correct even while the underlying data is actively changing."

## Clarifying Questions

- **Is the feed's ordering by insertion/creation time (newest-first), or something else (relevance score, a user-reorderable list)?** Cursor-based pagination assumes there's a stable, monotonic sort key to paginate against (typically a timestamp + tiebreaker ID). If the sort is by a score that can change after an item is created (e.g., "trending" ranking that recalculates), the "stable cursor" premise breaks down differently — the same item can legitimately move between pages even without insertions/deletions, which cursor pagination alone doesn't fix.
- **Does the product actually need random access ("jump to page 7") or arbitrary total-count display ("Page 3 of 48"), or is this feed consumed purely sequentially (infinite scroll / next-page-at-a-time)?** This is the central trade-off question — cursor-based pagination gives up cheap random access and exact total counts in exchange for correctness under concurrent modification; if the product genuinely needs "jump to page 7," that requirement conflicts with cursor pagination's core mechanism and needs to be resolved (approximate counts, hybrid approach) rather than assumed away.
- **How is "page 3" currently requested from the client — does the client track an explicit page number, or does it already pass some kind of continuation token it just treats as opaque?** If page number is baked into the client's own state/URL (`?page=3`) rather than derived from the previous response, migrating to cursors is also a client-side state-shape change (tracking "the cursor for the next page," not just an incrementing integer), not purely a backend/query change.
- **Are duplicates and skips equally bad here, or is one worse than the other for this product?** A social feed showing a duplicate post is mildly annoying; a duplicate in a paginated list of financial transactions being summed client-side is a correctness bug. A skipped item in an activity feed might go entirely unnoticed; a skipped item in, say, a paginated compliance audit log is a serious problem. The answer shapes how much engineering effort is justified for the fix.
- **Is the underlying store a single SQL database with an index that can support keyset pagination directly, or a system (search index, aggregated/joined view, external API) where "give me everything after this cursor" isn't a natural, cheap query?** Cursor pagination's efficiency argument depends on the sort key being backed by an actual index; if the data source can't efficiently answer "rows after X" without effectively still doing an offset-style scan internally, part of cursor pagination's benefit doesn't materialize even though the API shape changes.

## Approach & Trade-offs

**Diagnose the specific failure mode first — offset pagination's bug is a coordinate-system problem, not a data problem.** `?page=3&limit=20` means "skip 40, take 20" — it's a statement about *position in the current result set*, not about *which specific items* to return next. If an item is inserted at the front of the feed between the client fetching page 1 and page 2, everything shifts down by one; "skip 40" on page 3 now lands one item later than it would have, silently re-including whatever was previously at position 40 (a duplicate) or, on a deletion, skipping whatever shifted into that gap. The bug isn't stale data or a caching issue — it's that "offset from the start" is not a stable coordinate when the thing being offset into is itself mutating between reads.

**Cursor-based pagination fixes this by changing the question from "give me the Nth batch" to "give me everything after this specific item."** Instead of `page=3`, the client passes a cursor — typically an opaque, server-issued token that encodes the sort key value(s) of the last item it saw (e.g., `created_at` + `id` as a tiebreaker for ties on `created_at`). The next page's query becomes `WHERE (created_at, id) < (last_seen_created_at, last_seen_id) ORDER BY created_at DESC, id DESC LIMIT 20` — a query anchored to a specific, stable point in the data, not a position that shifts when the dataset does. An insertion anywhere other than exactly at that cursor point doesn't affect what "after this cursor" means; the client always gets exactly the next 20 items relative to the last one it actually saw, regardless of how many items were added or removed elsewhere in the feed.

**The trade-off: cursor pagination gives up two things offset pagination gets for free — arbitrary random access ("jump to page 7") and a cheap, exact total count.** A cursor only knows "the next N after this point" — there's no way to compute "page 7" without walking through pages 1–6 first (cursors are inherently sequential/linked-list-like, not indexed/array-like). Total counts (`COUNT(*)`) are a separate query in either model, but products using offset pagination often display "Page 3 of 48" cheaply alongside the paginated query itself; with cursor pagination that count either needs its own (potentially expensive, and itself capable of drifting from the paginated data mid-session) separate query, or the product accepts an approximate/omitted count. I'd state this explicitly: cursor pagination is the correct fix for *this* scenario's stated symptom (sequential feed consumption, correctness under concurrent modification), but it's a real regression if the product's actual requirement includes random-access page jumping.

**Where both matter — sequential correctness for infinite scroll AND occasional random access — a hybrid is more honest than picking one globally.** Some products use cursor pagination for the primary infinite-scroll/next-page flow (where correctness under mutation matters most, and where users are overwhelmingly moving sequentially) while keeping a separate, explicitly-approximate offset-based "jump to page" affordance for the rarer random-access case, accepting that a page-jump might occasionally show a minor skip/duplicate at the boundary in exchange for the feature being possible at all.

## Solution

**Step 1 — the broken offset-based query, made concrete.** Feed sorted newest-first; items are prepended as they're created:

```sql
-- page=3, limit=20 → OFFSET 40 LIMIT 20
SELECT * FROM feed_items ORDER BY created_at DESC OFFSET 40 LIMIT 20;
```

Timeline: user loads page 1 (items 1–20), then page 2 (items 21–40). Before loading page 3, 3 new items get created and prepended (now the newest 3 items in the feed). Every existing item's *offset position* shifts down by 3. "OFFSET 40" now lands 3 positions later than it did relative to what the user actually saw as items 21–40 — the query returns items that were previously at offset 37–39, three of which the user already saw on page 2. Duplicates. (Deletions produce the inverse: a gap, skipping items the user never saw.)

**Step 2 — cursor-based query, anchored to the last item actually seen, not a position:**

```sql
-- Client sends: after_created_at, after_id (from the last item on the previous page)
SELECT * FROM feed_items
WHERE (created_at, id) < ($after_created_at, $after_id)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

`created_at` alone isn't sufficient as a cursor if two items can share the same timestamp (common with second- or even millisecond-precision timestamps under load) — `id` (or any other globally unique, monotonically-assigned column) acts as a tiebreaker, making the composite `(created_at, id)` pair a genuinely unique, strictly-ordered position to anchor against.

**Step 3 — API response shape carries the next cursor explicitly, so the client never constructs or interprets it — it just echoes back what the server gave it:**

```json
{
  "items": [ /* 20 items */ ],
  "nextCursor": "eyJjcmVhdGVkX2F0IjoiMjAyNi0wOS0yNlQxMjowMDowMFoiLCJpZCI6IjQ4MjkifQ==",
  "hasMore": true
}
```

Encoding the cursor as an opaque, server-generated (typically base64-encoded JSON, or a signed token) string — rather than exposing raw `created_at`/`id` query params — keeps the client from depending on or reconstructing the cursor's internal shape, so the pagination scheme's internals (which columns compose the cursor) can change server-side without a client contract change.

**Step 4 — client-side, track the cursor from the last response, not a page number:**

```tsx
function useFeed() {
  return useInfiniteQuery({
    queryKey: ['feed'],
    queryFn: ({ pageParam }) => fetchFeed({ after: pageParam }),
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) => (lastPage.hasMore ? lastPage.nextCursor : undefined),
  });
}
```

There's no client-tracked "page 3" anymore — each request explicitly says "after this exact cursor," which is correct regardless of how many insertions/deletions happened elsewhere in the feed since the client's last fetch.

**Step 5 — for the "jump to page 7" requirement, if it exists, keep it as an explicitly separate, approximate path rather than trying to force cursors to support random access:**

```sql
-- Approximate page-jump: still offset-based, used ONLY for direct navigation,
-- clearly distinct from the cursor-based sequential feed
SELECT * FROM feed_items ORDER BY created_at DESC OFFSET :approxOffset LIMIT 20;
```

Flagging in the UI (or at least in the API contract/docs) that a direct page jump is a best-effort snapshot, not a live-consistent view the way sequential cursor-based paging is.

> **Check yourself:** If the feed's sort order is `ORDER BY created_at DESC` and two items are created within the same millisecond, what specifically breaks without the `id` tiebreaker in both the `ORDER BY` and the cursor comparison — walk through the exact query results.

## Client-side Consequences Beyond the Query

**Infinite-scroll UI code that assumed page numbers (`currentPage`, `totalPages`) needs to be reworked around cursors and `hasMore`, not page counts.** Any "you're on page 3 of 12" UI, or logic that computes "prefetch page N+1" from an incrementing integer, needs replacing with "prefetch using `nextCursor`" and "`hasMore` tells you whether there's anything left," since there may be no cheap way to know "12" at all.

**Deduplication at the client is still worth keeping as a defensive layer, not a replacement for the server-side fix.** Even with correct cursor pagination, client-side list-rendering code that merges pages into one array should dedupe by item ID before rendering — cheap insurance against edge cases (a retried request after a timeout, a backend bug, a brief migration period running both pagination schemes) without relying on it to paper over an actually-broken pagination model.

## Gotchas

**Using only a single, non-unique column as the cursor (e.g., `created_at` alone) without a tiebreaker**, silently dropping or duplicating items whenever two rows share that column's value — a subtle bug because it often doesn't show up in local testing with sparse, manually-created test data, only under real production write volume where same-timestamp collisions are common.

**Migrating the pagination API shape but leaving the underlying query as an offset under the hood** — e.g., the API now accepts an opaque cursor param but the server internally decodes it back into a page number and still runs `OFFSET`/`LIMIT` against it. This looks like a fix (the client contract changed) but doesn't actually solve the concurrent-modification correctness problem, since the underlying query is still positional, not anchored to a stable row.

**Assuming cursor pagination is a strict improvement with no trade-off, and being unable to answer why a product might still choose offset pagination.** Total-count display and random-access page-jumping are real product requirements in plenty of contexts (an admin table someone wants to jump around in, a search-results page showing "1,204 results, page 1 of 61") — presenting cursor pagination as universally correct without acknowledging what it gives up reads as not having actually weighed the trade-off.

**Forgetting that `hasMore`/"is there a next page" itself needs a cheap answer** — a naive way to compute it is fetching `limit + 1` items and checking if the extra one exists (then discarding it before returning), rather than issuing a separate `COUNT`-style query, which would reintroduce some of the cost cursor pagination was meant to avoid.

**Not handling the empty/boundary cursor case** — the very first request (no prior item, no cursor yet) and the very last page (cursor points past the last row) both need explicit handling (`initialPageParam: undefined`, `hasMore: false`), and a naive query written to always expect a non-null cursor value will error or misbehave on the first request.

## Follow-up Questions

**Q (High): Walk through, with a concrete timeline of inserts, exactly why offset pagination produces duplicates on insertion but skips on deletion — most candidates get the "it breaks" part right but not the directional mechanism.**

Answer: Offset pagination's `OFFSET N` means "skip the first N rows of the *current* full ordered result set, as it exists at query time" — it's a statement purely about position, re-evaluated fresh on every request. On insertion: say the feed is newest-first and 3 new items get prepended between page 2 and page 3 requests. Page 3's request still says `OFFSET 40`, but the *set being offset into* now has 3 new items at the front, shifting every previously-existing item's position down by 3 — so `OFFSET 40` today lands on what used to be position 37, meaning items 38, 39, 40 (which the user already saw on page 2, since page 2 was `OFFSET 20 LIMIT 20` → items 21–40) get returned again on page 3. Duplicates, specifically the last few items of the *previous* page re-appearing. On deletion: say 3 items get removed from earlier in the feed between page 2 and page 3. Every subsequent item's position shifts *up* by 3, so `OFFSET 40` on page 3 now lands 3 positions later than it would have — the query skips over the 3 items that shifted into positions 40–42, which the user never saw on any page. The direction of the error (duplicate vs. skip) is a direct consequence of whether positions shifted down (insert, ahead-of-cursor items pushed later → repeat) or up (delete, ahead-of-cursor items pulled earlier → skip) relative to a fixed offset number.

The trap: correctly stating "offset pagination breaks under concurrent modification" without being able to derive *which* direction of error comes from which kind of mutation — the mechanism (a re-evaluated positional skip against a moving target) is the actual signal of understanding, not just the memorized conclusion.

---

**Q (High): Why does the cursor need a composite key (e.g., `created_at` + `id`) rather than just the primary sort column — what specifically goes wrong with a single-column cursor?**

Answer: A single-column cursor is only a valid, unique position anchor if that column's values are themselves guaranteed unique across all rows in sort order — which a sort column like `created_at` usually isn't, especially at anything beyond very coarse (e.g., per-day) granularity, since two rows can legitimately share the same timestamp under concurrent writes or even just fast sequential inserts within the same clock tick. If the cursor is `created_at` alone and two items share the value `T`, a query like `WHERE created_at < $cursor` skips *both* same-timestamp rows entirely once either one has been seen (since `<` excludes exact matches), silently dropping the other row from the paginated result forever — the opposite failure from what cursor pagination was meant to fix, but still a correctness bug. Adding a unique tiebreaker column (typically an auto-incrementing ID, which is guaranteed unique and, combined with insertion order, usually correlates with `created_at` ordering) makes the composite `(created_at, id)` pair a genuinely unique position: the comparison becomes `WHERE (created_at, id) < ($cursor_created_at, $cursor_id)`, which correctly includes/excludes same-timestamp rows based on the tiebreaker rather than dropping ties outright.

The trap: choosing a cursor column that "usually" doesn't have duplicates and treating that as sufficient — pagination correctness needs a *guaranteed* unique, stable ordering, not a statistically-unlikely-to-collide one; production write volume reliably finds the edge case that local testing doesn't.

---

**Q (High): What does cursor pagination give up compared to offset pagination, and how would you design around the "jump to page 7" requirement if the product genuinely needs it?**

Answer: Cursor pagination gives up two things: cheap random access (a cursor only encodes "the next N after this specific point," so computing "page 7's items" without cursor pagination requires either walking sequentially through pages 1–6 first, or falling back to an offset-style query for that specific jump) and a cheap, always-in-sync total count (an exact `COUNT(*)` is a separate query either way, but products commonly display it alongside offset-paginated results as effectively free; with cursor pagination, that count query is explicitly separate and can itself drift from the cursor-paginated view mid-session, since the count reflects the moment it was queried, not the moment any given page was fetched). If a product genuinely needs page-jumping, a defensible design keeps cursor pagination for the primary sequential-consumption flow (infinite scroll, "load more," anywhere correctness under concurrent modification matters most and users are overwhelmingly moving forward sequentially) and provides a separate, explicitly-best-effort offset-based endpoint or parameter specifically for direct page navigation, documented (and ideally UI-communicated) as a snapshot-in-time view that may show minor boundary duplicates/skips if the data changed between the jump and the surrounding sequential browsing — rather than trying to force one mechanism to satisfy both access patterns with full correctness in both.

The trap: presenting cursor pagination as a strict, no-downside improvement over offset pagination — failing to name what's given up (random access, cheap exact counts) suggests not having actually weighed the trade-off, which is precisely the senior signal this kind of question is probing for.

---

**Q (Medium): The feed's sort key isn't creation time — it's a "relevance score" that gets recalculated periodically, so an item's position in the feed can change even without any insertions or deletions. Does cursor pagination still work here, and if not, what does?**

Answer: Cursor pagination's correctness guarantee depends on the sort key being *stable* — once an item has been placed after a given cursor value, it needs to stay there for the "give me everything after this cursor" query to keep returning a consistent, non-overlapping sequence across requests. A relevance score that's recalculated after the client has already fetched a page based on the *old* score breaks that assumption independently of inserts/deletes: an item could move from "not yet returned" to "already passed by the cursor" (getting skipped even though it was never actually returned) or vice versa (getting duplicated), purely because its sort key changed value between requests, which is a different failure mode than concurrent insertion/deletion but has the same symptom. The fix isn't cursor pagination alone — it needs either (a) snapshotting the sort key at the start of a pagination session (fetch all relevance scores as of session start, paginate against that frozen snapshot, and accept that the displayed ranking may be slightly stale by the time the user reaches later pages) or (b) accepting that a *live*, continuously-reranking feed fundamentally cannot offer perfectly consistent, no-skip-no-duplicate pagination without some form of snapshot/versioning, and choosing an acceptable trade-off (e.g., only re-rank at page-load boundaries, not mid-session) rather than treating "switch to cursors" as sufficient on its own.

The trap: applying "cursor pagination fixes this" as a blanket answer without checking whether the sort key itself is stable — cursor pagination fixes positional-coordinate instability from insertion/deletion, but does nothing for sort-key instability, which is a related but structurally different problem needing its own fix (snapshotting).

---

**Q (Medium): How would you paginate correctly through a dataset with duplicate items already de-duplicated for display (e.g., grouping by a shared property, or collapsing near-duplicate events)? Does that change the cursor design?**

Answer: If the server groups/collapses rows before pagination (e.g., 5 raw "user liked your post" events collapsed into one "5 people liked your post" summary row), the cursor needs to be anchored to the *post-grouping* unit's stable ordering, not the underlying raw rows' — otherwise the same instability problem resurfaces one layer up: if grouping happens dynamically per-request (e.g., grouping only events from the last hour, which shifts as time passes) rather than being a stable, queryable property, "the item at this cursor" can itself change shape between requests. The more robust design computes and persists a stable identity and sort position for the grouped/collapsed unit itself (e.g., a materialized "activity group" row with its own `created_at`/`id`, updated as new raw events fold into it) so the cursor pagination logic operates on the same stable, indexed columns discussed throughout, just one abstraction layer higher — rather than trying to derive a stable cursor from an inherently dynamic, request-time grouping computation.

The trap: assuming cursor pagination as designed for raw rows extends automatically to any derived/grouped view of those rows — grouping introduces its own stability requirement (the grouping itself must be deterministic and durable across requests, not recomputed fresh each time) that's easy to overlook if the underlying raw-row cursor design is otherwise correct.

---

**Q (Low): Would a cursor-based pagination scheme work well for a GraphQL API using Relay-style connections (`edges`/`node`/`pageInfo`)? How does that spec's approach compare to the REST cursor design above?**

Answer: Relay's connection spec is essentially a standardized shape for exactly the cursor-based design discussed above — each edge carries an opaque `cursor` alongside its `node` (the actual data), and `pageInfo` exposes `hasNextPage`/`hasPreviousPage` and `startCursor`/`endCursor`, giving clients a consistent way to request `first: 20, after: $cursor` (or `last`/`before` for backward pagination) regardless of the underlying resolver's implementation. The main difference from a hand-rolled REST cursor API is standardization (client tooling like Relay or Apollo's cache can generically handle any Relay-style connection without per-endpoint custom pagination logic) and the built-in support for *bidirectional* pagination (both `after`/`first` and `before`/`last`), which a REST design has to deliberately add if needed rather than getting for free from the spec. The underlying correctness concerns are identical — the resolver behind a Relay connection still needs a genuinely stable, unique sort key (the same composite-column tiebreaker discussion applies), and a resolver that internally implements the connection via `OFFSET`/`LIMIT` against page numbers derived from decoded cursors would have exactly the same concurrent-modification bug as the REST version, just hidden behind a spec-compliant-looking API shape.

The trap: assuming that using a "cursor-shaped" API (Relay connections, or any opaque-cursor-looking contract) automatically implies correct, stable-under-mutation pagination — the API shape and the underlying query implementation are separate concerns, and a spec-compliant-looking cursor API can still be backed by a broken offset-based resolver underneath.

---

## Self-Assessment

- [ ] Can explain, with a concrete timeline, why offset pagination produces duplicates on insertion and skips on deletion — the directional mechanism, not just "it breaks"
- [ ] Can explain why a cursor needs a composite key (sort column + unique tiebreaker), not a single column
- [ ] Can articulate what cursor pagination gives up (random access, cheap exact counts) and design around a genuine page-jump requirement
- [ ] Can identify that a dynamic/recalculated sort key breaks cursor pagination differently than concurrent insert/delete does, and know the snapshot-based fix
- [ ] Can recognize a cursor-shaped API contract backed by a still-broken offset-based query underneath

---
*Next: WebSocket Reconnection & Backoff — from paginating a feed correctly to keeping a live connection to that feed alive and self-healing under real-world network instability.*
