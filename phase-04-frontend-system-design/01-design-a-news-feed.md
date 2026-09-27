# Design a News Feed (Facebook/LinkedIn-style)

## Quick Reference

| Decision Point | Choice | Why |
|---|---|---|
| Pagination | Cursor-based (opaque cursor or `(timestamp, id)` tuple), not offset | The feed is a live, mutating list — items are inserted/removed between requests, so `LIMIT/OFFSET` skips or duplicates rows; a cursor anchors to a specific item regardless of what's inserted above it |
| List rendering | Virtualized, variable-height windowing (measure-on-render, not fixed row height) | Post cards vary wildly in height (text-only vs. image vs. video vs. shared-post); thousands of DOM nodes over a long session otherwise degrades scroll perf and blows up memory |
| New content arriving live | "New posts" banner the user taps, never auto-prepended | Auto-inserting above the viewport shifts what the user is currently reading — a layout-stability and trust problem, not just an aesthetic one |
| Post identity across the feed | Normalized client cache keyed by `postId`, feed is an ordered list of IDs referencing it | The same post can appear twice (a share, a re-surfaced post) — without normalization, liking one copy leaves the other stale |
| Heterogeneous content types | A renderer registry keyed by `post.type`, dispatching to a dedicated component per type | New post types (polls, live videos) get added over the product's life; a giant `if/else` in one component doesn't scale and can't be code-split |

## The Scenario

"Design the client-side architecture for a news feed — think Facebook or LinkedIn. Users scroll an infinite, mixed-content feed: text posts, image posts, video posts, shared/reposted content, maybe the occasional poll. New posts come in from people they follow, roughly ranked by relevance rather than strictly chronological. Walk me through how you'd build this on the frontend — the data flow, the rendering strategy, and how you'd keep it fast at scale."

## Clarifying Questions

- **Is the feed strictly server-ranked (client just renders what it's given, in order) or does the client have any say in ordering/filtering?** This determines whether the client can ever reorder items locally (e.g., to "smooth in" a like-count update) — if ranking is entirely server-driven and can change between pages, the client must treat page order as authoritative and never silently resort, or it desyncs from what the server considers "the feed" and reordering becomes visually jarring.
- **How is pagination expected to behave under real-time inserts — if ten new posts publish while I'm on page 3, should page 4 be affected?** This is the single most important question for this scenario: it decides between offset-based (`page=4`) and cursor-based (`before: <postId or timestamp>`) pagination, and getting it wrong (defaulting to offset because it's what junior candidates default to) produces skipped or duplicated posts the moment the feed is anything other than a static list.
- **Are posts mutable while in view — can a like count, comment count, or the post body itself change underneath a card the user is currently looking at (via a live update channel), or is a fetched page a static snapshot until the next fetch?** This decides whether the client needs any live-update transport (WebSocket/SSE/polling) at all, versus just re-fetching being sufficient — a "read-heavy, eventually-consistent-is-fine" feed is a much simpler build than one broadcasting live like/comment deltas to open clients.
- **What's the expected content-type set, and is it fixed or does the product roadmap expect new types (polls, live video, events) to be added later?** This shapes whether a hardcoded switch on post type is acceptable for an MVP or whether an extensible renderer-registry pattern is worth the upfront complexity — over-engineering extensibility for a feed that will only ever have three post types is its own mistake.
- **What are the device/scale constraints — is this expected to run acceptably on a low-end Android device on a slow connection, and roughly how many items would a typical session scroll through?** A feed that only ever needs to render 30–40 items in a session doesn't need the same virtualization investment as one where power users scroll thousands of items — over-building windowing for a short feed is wasted complexity, under-building it for a long one is a hard perf wall later.
- **Does the feed need to work for logged-out/unauthenticated users for SEO (e.g., a public profile's post history indexed by search engines)?** If so, an initial server-rendered page of content matters — an entirely client-fetched, JS-only feed is invisible to crawlers that don't execute JS, or execute it unreliably, so this affects whether the first page needs to arrive pre-rendered versus purely as a client fetch after mount.

## Approach & Trade-offs

**Cursor pagination over offset, and why offset actively corrupts a live feed rather than merely being "less efficient."** Offset pagination (`GET /feed?page=3&pageSize=20`) assumes the underlying ordered set doesn't change between requests — for a news feed, that assumption is false by design: new posts are constantly published and ranking is often server-side and can shift between requests. If ten new posts land above the ones a user has already seen, `page=3` after that insert refers to a different slice of the feed than it did a moment ago — the user gets duplicates of posts they already saw, or skips ones they never saw at all. Cursor pagination (`GET /feed?before=<opaque-cursor-from-last-item>`) anchors each request to "give me what comes after this specific item," which is stable regardless of what gets inserted above the cursor position. The trade-off is that cursors don't support jumping to an arbitrary page number (fine for a feed — no one deep-links to "page 47 of my feed") and the client must always paginate forward from where it last left off rather than requesting pages out of order.

**Variable-height virtualization, not a fixed-row-height list, because post cards are not uniform.** A naive virtualization approach (`react-window`'s `FixedSizeList`) requires knowing every row's height up front, which doesn't hold here — a text-only post might be 120px, an image post 500px, a video post with a caption and three lines of comments preview taller still. The correct approach measures each rendered card's actual height (via `ResizeObserver` or a library like `@tanstack/react-virtual` that supports dynamic measurement) and caches that measurement per item so scroll-position math stays accurate as the user scrolls past items whose heights were previously unknown. The trade-off versus fixed-height virtualization is complexity (measuring, caching, and invalidating heights, handling the "estimated height before first measurement" case for scrollbar sizing) — but a fixed-height approach here would either force awkward uniform card heights (bad for a rich content feed) or produce visibly broken scroll-jumping as it discovers actual content doesn't match assumed row height.

**New live content becomes a "N new posts" banner the user taps to reveal, not an auto-prepend.** If new posts silently insert above the viewport while a user is mid-read, everything they're looking at shifts downward — a jarring, disorienting experience, and if it happens while they're mid-tap on something, a mis-click. Instead, new items arriving (via poll or a lightweight WebSocket "new content available" signal — not necessarily the full post payload) increment a counter shown in a fixed banner ("12 new posts") above the visible feed; tapping it is what actually prepends the new items and scrolls to top. This keeps the currently-rendered list's scroll position and content completely stable until the user explicitly opts into disruption.

**A normalized client-side cache (`postsById`), with each feed being an ordered array of IDs rather than an array of full post objects, because the same post can legitimately appear more than once.** A share/repost surfaces the original post's engagement data (like count, comment count) embedded inside a wrapping "share" post; the same original post might also independently appear elsewhere in the same feed session. If each feed position holds its own denormalized copy of that post's data, liking it in one place doesn't update the other appearance — a visible consistency bug. Normalizing (a single `Map<postId, Post>` as the source of truth, with feed order stored separately as `postId[]`) means every rendered occurrence of a given post subscribes to the same underlying record, so a like/comment count update (whether from an optimistic local action or a live server push) is reflected everywhere that post is rendered, automatically.

**A renderer registry (`type → Component`) for heterogeneous post types, favoring extensibility over a single component's internal branching.** Each `FeedItem` looks up its renderer by `post.type` from a registry map rather than a component with a growing `if (post.type === 'video') {...} else if (...)` ladder. This isn't just cleaner — it means a rare type (say, polls) can be code-split (`React.lazy`) so its bundle cost is paid only when a poll actually appears in someone's feed, and a new content type can be added by registering a new entry rather than touching a shared component's branching logic (open/closed — extend by registration, don't modify the dispatcher).

## Solution

**Data fetching and pagination**, using a cursor-based infinite query:

```tsx
interface FeedPage {
  items: Post[];
  nextCursor: string | null;
}

function useFeed() {
  return useInfiniteQuery<FeedPage>({
    queryKey: ['feed'],
    queryFn: ({ pageParam }) =>
      fetchFeedPage({ before: pageParam as string | undefined }),
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    initialPageParam: undefined,
  });
}
```

**Normalized post store**, updated by both pagination and optimistic engagement actions:

```tsx
type PostsById = Record<string, Post>;

function normalizePages(pages: FeedPage[]): { order: string[]; byId: PostsById } {
  const byId: PostsById = {};
  const order: string[] = [];
  for (const page of pages) {
    for (const post of page.items) {
      byId[post.id] = post; // last write wins if the same post reappears
      order.push(post.id);
    }
  }
  return { order, byId };
}

// Optimistic like: update the single normalized record —
// every rendered occurrence of this post re-renders from the same source.
function useLikePost() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (postId: string) => likePost(postId),
    onMutate: async (postId) => {
      queryClient.setQueryData<InfiniteData<FeedPage>>(['feed'], (data) =>
        patchPostEverywhere(data, postId, (post) => ({
          ...post,
          viewerHasLiked: true,
          likeCount: post.likeCount + 1,
        }))
      );
    },
    onError: (_err, postId) => {
      // roll back the same patch on failure
      queryClient.setQueryData<InfiniteData<FeedPage>>(['feed'], (data) =>
        patchPostEverywhere(data, postId, (post) => ({
          ...post,
          viewerHasLiked: false,
          likeCount: post.likeCount - 1,
        }))
      );
    },
  });
}
```

**Virtualized list with dynamic measurement** (using `@tanstack/react-virtual`, which supports per-item measured height):

```tsx
function FeedList({ ids }: { ids: string[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: ids.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 280, // initial guess before a card is measured
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ overflow: 'auto', height: '100vh' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={ids[virtualRow.index]}
            ref={virtualizer.measureElement}
            data-index={virtualRow.index}
            style={{ position: 'absolute', top: 0, transform: `translateY(${virtualRow.start}px)`, width: '100%' }}
          >
            <FeedItem postId={ids[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Renderer registry for heterogeneous post types:**

```tsx
const POST_RENDERERS: Record<Post['type'], React.ComponentType<{ postId: string }>> = {
  text: TextPost,
  image: ImagePost,
  video: VideoPost,
  shared: SharedPost,
  poll: React.lazy(() => import('./PollPost')), // rare type — code-split
};

function FeedItem({ postId }: { postId: string }) {
  const post = usePost(postId); // reads from the normalized store
  const Renderer = POST_RENDERERS[post.type];
  return <Renderer postId={postId} />;
}
```

> **Check yourself:** Without looking above, explain precisely why an offset-based `page=N` request breaks the moment new posts are inserted at the top of a ranked feed between two requests, and why a cursor tied to the last-seen item doesn't have that failure mode.

## Data Model

```ts
interface Post {
  id: string;
  type: 'text' | 'image' | 'video' | 'shared' | 'poll';
  authorId: string;
  createdAt: string;
  content: string;
  media?: MediaAsset[];
  sharedPostId?: string; // present when type === 'shared'
  likeCount: number;
  commentCount: number;
  viewerHasLiked: boolean;
}

interface FeedPage {
  items: Post[];
  nextCursor: string | null; // opaque; server-owned encoding of rank position
}
```

The client never constructs or interprets the cursor's internal structure — it's an opaque token the server returns and the client echoes back, which keeps ranking logic (whatever it is, and however it changes) entirely server-side, with the client only responsible for "give me the next page after this token."

## Scaling Considerations

**Virtualization is what makes long sessions viable at all** — without it, a user who scrolls 500 posts in one session accumulates 500 mounted DOM subtrees (each with images, engagement buttons, possibly video elements), which degrades scroll frame rate and can crash a memory-constrained mobile browser. Virtualization keeps mounted DOM bounded to roughly "visible + overscan," regardless of total scroll depth.

**Image/video loading needs its own lazy strategy layered on top of list virtualization** — `loading="lazy"` on `<img>` plus responsive `srcset`/CDN-resized images (never shipping a desktop-resolution image to a phone), and video elements that don't begin buffering/playing until scrolled into view (an `IntersectionObserver` gating `play()`/`pause()`) and pause immediately on scrolling out, both for perceived performance and to avoid needlessly burning mobile data.

**Real-time "new content" signaling should be a lightweight notification, not a full push of content** — a WebSocket message like `{ newPostsAvailable: true, count: 12 }` (or even just a count, or a poll-based check) is enough to drive the banner; pushing full post payloads for content the user hasn't asked to see yet wastes bandwidth for posts that might never be viewed if the user never taps the banner.

**Prefetch the next page before the user hits the literal bottom of the loaded list** — triggering the next cursor fetch when the user is, say, 3–5 items from the end (rather than exactly at the end) hides fetch latency behind continued scrolling, so the "loading" spinner is rarely actually seen in practice.

## Gotchas

**Defaulting to offset pagination because it's the more familiar pattern.** This is the single most common miss in this scenario — it works fine in a demo against static seed data and only reveals its brokenness under the exact conditions (live inserts, re-ranking) that make this a "feed" rather than a plain paginated list.

**Skipping normalization and letting each feed position hold its own denormalized post copy.** Looks identical to the normalized version until a post appears twice in one session (a share, or the same post re-surfacing) — then liking one copy silently leaves the other stale, a bug that's easy to miss in testing with small seed data where duplicates rarely occur.

**Auto-prepending live updates instead of gating them behind a banner.** Technically "more real-time," but it's the kind of decision that reads as a UX miss in an interview — reordering content under a user's eyes while they're reading is a well-known anti-pattern, and naming the banner pattern unprompted is a strong signal.

**Using fixed-height virtualization for a feed with genuinely variable card heights.** Either forces an artificial uniform-height design constraint or produces visible scroll-jump/misalignment as actual measured heights diverge from the fixed assumption — dynamic measurement is the correct default here, not an optimization to add later.

**Forgetting cancellation/dedup on fast scrolling triggering multiple pagination requests in flight.** A user scrolling quickly can trigger several "fetch next page" calls before the first resolves; without request dedup (most data-fetching libraries handle this if used correctly, but it's easy to defeat with ad hoc `fetch` calls) this produces duplicate pages or out-of-order insertion.

**No accessible fallback for infinite scroll.** Scroll-triggered pagination alone leaves screen-reader and keyboard-only users with no clear way to trigger "load more" without relying on scroll gestures working correctly with their tooling — an explicit, focusable "Load more posts" button (which can be visually hidden but still present, or shown when scroll-triggering fails) is the accessible-by-default version of this pattern.

## Follow-up Questions

**Q (High): Why does cursor-based pagination require the client to always paginate strictly forward, and what does the client lose by not being able to request "page 5" directly?**

Answer: A cursor encodes "give me what comes after this specific item in this specific ranking," which only makes sense relative to a known anchor point — there's no way to derive "the state of the ranking as of page 5" without having walked through pages 1–4 first, because the ranking itself may be shifting between requests (new posts, re-ranked scores). The client loses the ability to deep-link to or restore an arbitrary page number, and loses random-access jumping — but for a news feed, neither of those is a real product requirement (no one bookmarks "page 12 of my feed"), so the trade-off costs nothing in practice while buying correctness under live mutation, which offset pagination cannot provide at all for this use case.

The trap: proposing offset pagination "for simplicity" without being asked to justify it against live inserts — a candidate who picks offset and can't be pressed into recognizing why it's wrong here is missing the actual point of the scenario, not just making a stylistic choice.

---

**Q (High): A post the user is currently viewing gets deleted (or edited) by its author while still in the visible feed. What should happen, and how would you even find out?**

Answer: Finding out requires some live-update signal — either a WebSocket/SSE event carrying `{ type: 'post_deleted', postId }` / `{ type: 'post_updated', postId, patch }`, or, in a simpler polling-based design, the next feed page's inclusion (or omission) of that post being noticed on next fetch. Once known, the fix is applied to the single normalized record (`postsById[postId]`) — a deletion either removes the card in place (with a brief "this post was removed" placeholder rather than an abrupt disappearance, which can be disorienting mid-scroll) and an edit re-renders the same card with updated content, again automatically reflected everywhere that post's ID appears because of normalization. Without normalization, this would require finding and patching every denormalized occurrence individually, which is exactly the bug class normalization exists to prevent.

The trap: assuming this requires a full feed re-fetch to resolve — re-fetching the whole feed to handle one changed item is both wasteful and reintroduces exactly the "did the ranking shift underneath me" problem cursor pagination was chosen to avoid; a targeted patch to the normalized record is the correct scope of change.

---

**Q (High): How would you prevent a fast-scrolling user from triggering the "load next page" fetch multiple times before the first one resolves?**

Answer: Guard the trigger with an explicit in-flight flag (most infinite-query libraries expose this as `isFetchingNextPage`) and only call `fetchNextPage()` when that flag is false, regardless of how many times the scroll/intersection threshold fires while a request is outstanding. If building this by hand without a library, the same idea applies manually: a `useRef` or state flag set the moment a request starts and cleared in its `finally`, checked before issuing a new one. The underlying principle is that "trigger a fetch" and "a fetch is in flight" need to be decoupled so multiple trigger events collapse into a no-op once one request is already outstanding.

The trap: solving this with a debounce on the scroll handler alone — debouncing reduces how often the trigger *checks*, but doesn't prevent two checks that both pass before either request resolves from firing two requests; the actual fix is an in-flight guard, not a timing delay.

---

**Q (Medium): Why is a normalized cache (`postsById` + arrays of IDs) preferable to each feed page just holding its own array of full post objects?**

Answer: The core reason is that the same post can appear in more than one place — as itself and as the target of a share/repost, or re-surfaced independently later in the same session — and any state that changes on that post (like count, comment count, edited content) needs a single source of truth so every rendered occurrence reflects the update consistently. With denormalized copies, updating one occurrence (say, via an optimistic like) requires either updating all occurrences by hand (error-prone, easy to miss one) or accepting visible inconsistency between them. A normalized store makes "update this post" a single, unambiguous operation regardless of how many places it's currently rendered.

The trap: describing normalization purely as a memory-savings optimization — the memory savings are real but secondary; the actual forcing function is correctness under duplicate occurrences, which is a bug class, not just an efficiency concern.

---

**Q (Medium): How would you code-split the renderer for a rare post type (e.g., polls) without introducing a loading flicker for the common case (text/image posts)?**

Answer: Only the rare type's entry in the renderer registry is wrapped in `React.lazy`, with its own `Suspense` boundary scoped tightly around just that renderer (not wrapping the whole feed list) — so a `Suspense` fallback (a lightweight skeleton matching the card's approximate shape) only ever appears the moment a poll actually needs to render, not for the rest of the feed's more common content types, which stay as ordinary eagerly-bundled components. Placing the `Suspense` boundary too high (around the entire `FeedList`) would suspend the whole visible feed whenever any single lazy item is loading, which is the flicker this approach is specifically designed to avoid.

The trap: lazy-loading every post-type renderer uniformly "for consistency" — this pessimizes the common case (text/image posts now pay a chunk-load round trip on first render) to gain nothing for types that don't need it; code-splitting should be applied selectively to genuinely rare/heavy types.

---

**Q (Medium): The interviewer asks: "What if two different feed positions show the same post, but with subtly different context (e.g., 'liked by 3 mutual friends' text that depends on *why* it was surfaced at that position)?" Does normalization still work cleanly here?**

Answer: This is the one place normalization needs a small refinement — the position-specific context ("why this appears here," a mutual-likes annotation, a "sponsored" label tied to that specific placement) is not a property of the post itself, so it shouldn't live in the normalized `Post` record. Instead, the feed's ordered list holds lightweight `FeedEntry` objects (`{ postId, surfaceReason, entryId }`) rather than bare post IDs, and rendering combines the entry's placement-specific context with the normalized post data looked up by `postId`. This keeps the actual post data (likes, comments, content) singular and consistent, while still allowing the same post to carry different contextual framing at different feed positions.

The trap: either forcing the "why it's here" context into the normalized post record (which breaks the moment the same post needs two different reasons at two positions) or abandoning normalization altogether to accommodate this — the fix is separating "the post" from "this specific placement of the post," not choosing one normalization strategy for everything.

---

**Q (Low): How would you decide whether this feed needs any server-rendered content at all versus being purely client-fetched?**

Answer: It comes down to the SEO/crawler and first-contentful-paint requirements clarified up front — if logged-out profile pages need to be indexed by search engines, or if there's a strong product requirement around fast perceived load for the first screenful of content (especially on slower connections), the first page of the feed should arrive pre-rendered (SSR or a static/ISR snapshot) with client-side hydration taking over for subsequent interaction and pagination from there. If the feed is purely an authenticated, behind-login experience with no crawler-visibility requirement, a client-fetched approach (with a skeleton-loading first paint) is simpler to build and reason about, and the SSR investment isn't buying anything the product actually needs.

The trap: assuming SSR is always "better" and should be the default answer regardless of what was established about logged-out access — the correct answer is conditional on the requirements gathered earlier in the scenario, not a fixed best practice to recite.

---

## Self-Assessment

- [ ] Can explain why offset pagination breaks specifically under live inserts/re-ranking, and why a cursor doesn't have that failure mode
- [ ] Can justify variable-height virtualization over fixed-height for a mixed-content feed, including what dynamic measurement actually requires
- [ ] Can describe the "new posts" banner pattern and why auto-prepending is the wrong default
- [ ] Can explain why a normalized `postsById` store is necessary once the same post can appear more than once in a session
- [ ] Can design a renderer-registry pattern for heterogeneous content types and explain when/how to code-split a rare type
- [ ] Can reason through what happens when a currently-visible post is edited or deleted server-side mid-session

---
*Next: Design an Autocomplete / Search-as-you-type System — moves from "render a long, mostly-passive scrolling list" to "handle a fast-firing stream of user input with cancellation, debouncing, and race conditions," a different axis of system-design skill (input-driven data flow rather than list-rendering at scale).*
