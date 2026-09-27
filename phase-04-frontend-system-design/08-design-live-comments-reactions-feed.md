# Design a Live Comments/Reactions Feed

## Quick Reference

| Decision Point | Choice | Why |
|---|---|---|
| Concurrency model | Append-only ordered stream (comments) + per-user toggleable aggregate counters (reactions) — no OT/CRDT machinery needed | Comments are immutable-once-posted, single-author entities and reactions are simple per-user toggles, not arbitrary concurrent structural edits to shared content — a fundamentally simpler problem than the collaborative document editor scenario |
| New comment arrival while reading | A "N new comments" banner if the user has scrolled up into history; auto-append at the bottom only if already anchored to the live bottom edge | Mirrors the News Feed's banner pattern — disrupting a user's current reading position with inserted content is the same anti-pattern here as there |
| Posting your own comment | Optimistic append with a temporary ID, reconciled with the server's authoritative ID/timestamp on ack | Same pattern as chat messages — the author must see their own comment instantly; reconciliation (not remove-then-readd) avoids flicker |
| Rapid reaction toggling (like/unlike spam-tapped) | Coalesce to the *net final state* before sending, not one request per tap | A user double-tapping a like button five times in a second should produce one settled request reflecting the final state, not five racing toggle requests that can resolve out of order |
| Viral-post event volume | Server-side (and if needed client-side) batching/coalescing of rapid-fire reaction/comment events into periodic snapshots, not one push per individual event | A sufficiently popular post can generate far more raw events per second than any UI can meaningfully render one-by-one; the client should receive periodic aggregate updates, not an unthrottled firehose |

## The Scenario

"Design the comments and reactions section under a post — think a live sports-highlight video or a breaking-news article with a fast-moving comment section and reaction counts (likes, hearts, etc.) updating in real time as other users interact. New comments should appear live without requiring a refresh, reaction counts should update live, and it needs to hold up on a post that's currently going viral with thousands of simultaneous viewers. Walk me through the architecture."

## Clarifying Questions

- **Are comments flat/chronological, or threaded/nested (replies to replies)?** Threading adds real rendering complexity (recursive tree structure, as covered in [Nested Comments — Recursive Tree Rendering](../phase-02-component-machine-coding/16-nested-comments-recursive-tree.md)) layered on top of everything specific to this scenario (real-time delivery, pagination, reactions) — worth confirming which is in scope before assuming either.
- **Can a comment be edited or deleted after posting, including by a moderator rather than only its author?** If so, the client needs to handle a specific existing comment's content changing or disappearing entirely out from under a user currently reading it (including cases where a comment gets removed by moderation well after being optimistically posted and initially visible) — a different, narrower concern than the arbitrary-concurrent-editing problem in the Collaborative Document Editor scenario, but still needs explicit handling, not an assumption that once rendered, a comment is immutable for the rest of the session.
- **Are reactions a single type (a simple like count) or multiple types (like, love, laugh, etc., as in many social platforms), and can a user change their reaction (from like to love) rather than only toggling on/off?** Multiple mutually-exclusive reaction types per user is a meaningfully different state shape (a user's current reaction, if any, from a fixed set) than a plain boolean like/unlike, affecting both the optimistic-update logic and the aggregate-count data structure.
- **What's the expected peak concurrency — a normal post with occasional activity, or a genuinely viral one with thousands of simultaneous viewers generating a high sustained rate of comments/reactions per second?** This is the detail that most changes the design: a low-activity comment section can reasonably push every individual event to every client as it happens; a viral one cannot, and needs deliberate batching/throttling on the server side (and possibly client-side coalescing of rapid UI updates) to stay usable at all.
- **Should the comment count and reaction counts shown be exact/real-time-precise, or is an eventually-consistent, occasionally-approximate count acceptable (e.g., "1.2K" rather than an exact live-updating integer)?** Accepting approximate, less-frequently-updated counts for very high-volume posts is a legitimate, common trade-off that meaningfully reduces both server broadcast load and client render churn — worth surfacing as an option rather than assuming exact real-time precision is required at any volume.
- **Do older comments need to be paginated/loaded on scroll, or is the comment section expected to hold a bounded, always-fully-loaded set (e.g., only the most recent 50, with no deeper history)?** Affects whether cursor-based pagination (the same pattern used in the News Feed scenario) needs to be combined with live real-time appends, which is a slightly more involved combination than either alone.

## Approach & Trade-offs

**This is a deliberately simpler concurrency problem than the Collaborative Document Editor, and recognizing why is itself part of a strong answer.** Comments are append-only, single-author, generally-immutable-once-posted entities — there's no scenario where two users are concurrently editing the *same* comment's content the way two users might concurrently edit the same paragraph of a shared document. A new comment is simply a new, independent entity being added to an ordered list; the only "conflict" possible is ordering (whose comment appears before whose, when two arrive at nearly the same server timestamp), which is resolved trivially by a server-assigned sequence/timestamp, not by any transform or CRDT-merge logic. This distinction is worth stating explicitly — reaching for OT/CRDT machinery here would be significant, unwarranted overengineering for a problem that a much simpler ordered-append-plus-toggle-counters model fully solves.

**New comments arriving while a user is scrolled up into history should not auto-append at the visible bottom, mirroring the News Feed's "new posts" banner pattern, for the same underlying reason.** If a user has scrolled up to read earlier comments, silently inserting new ones at the live bottom edge doesn't disturb their current view directly (since it's below what they're looking at) — but it does mean returning to "the bottom" later encounters an unpredictable amount of new content, and some products additionally choose to keep the *count itself* deferred (a "15 new comments" indicator) rather than silently growing a total-count number, to avoid subtle attention-grabbing shifts elsewhere on the page (e.g., a comment-count badge changing while the user is mid-read). If the user is *already* anchored to the live bottom edge (the common case for actively watching a fast-moving comment section, akin to a chat window), auto-appending new comments as they arrive, with the view auto-scrolling to keep up, is the expected and desired behavior there — the distinction is exactly the same "am I anchored to the live edge or reading history" check used in chat UIs and streaming logs generally.

**Reaction toggling needs request coalescing, not naive one-request-per-tap, because a user can toggle far faster than a round trip resolves.** If a user taps "like" then quickly "unlike" then "like" again within a second (a real, common interaction — impatient double-tapping, or genuinely changing their mind fast), sending three separate toggle requests risks them resolving out of order (the "unlike" response arriving after the final "like" response, leaving the server-confirmed state wrong from the user's perspective) — the same class of race condition seen in the autocomplete and e-commerce filter scenarios, solved the same way: a sequence number (or simply "only the latest pending request's result is trusted, superseding any earlier one still in flight") ensures the client's displayed state converges to whatever the *most recent* user action actually intended, regardless of network resolution order. A further refinement coalesces multiple rapid toggles into a single network request reflecting only the *net* final state (if the user ends up back where they started — liked, unliked, liked again, net result "liked" — a debounced send only needs to transmit one final "like" request, not three), which both reduces request volume and sidesteps the ordering race entirely for the common rapid-tap case.

**Viral-scale event volume requires server-side batching before it ever reaches the client, not just client-side rendering optimizations.** A post with thousands of concurrent viewers reacting and commenting can generate a genuinely enormous number of discrete events per second — pushing every single one to every connected client individually would overwhelm both the server's fan-out capacity and any client's ability to meaningfully render that much incoming information (no user can process "247 individual like events" flashing by per second as separate visual updates anyway). The correct architecture has the server aggregate rapid-fire events into periodic snapshots (e.g., "here's the updated reaction count as of this 250ms window" rather than one push per individual reaction) — the client receives and renders a manageable stream of aggregate updates instead of an unthrottled firehose of individual events, and a reaction *count* updating smoothly every quarter-second reads as "live" to a human observer just as convincingly as (and considerably more sustainably than) pushing every discrete underlying event.

**Comments needing pagination *and* live append simultaneously is a slightly more involved combination than either pattern alone, and needs an explicit cursor discipline.** The initial page load fetches the most recent N comments via ordinary cursor-based pagination (same pattern as the News Feed); scrolling up loads older comments via the same cursor mechanism extended backward in time. Live-arriving new comments are a logically separate stream (append at the live/newest edge) that must be merged into the same underlying ordered list with the same ID-based dedup discipline used elsewhere in this repo's real-time scenarios (a comment fetched via a "load older" pagination request and a comment that separately arrived live must never be double-counted if, by some timing coincidence, both paths deliver the same comment).

## Solution

**Optimistic comment posting, reconciled on ack — the same pattern as chat message sending:**

```tsx
function usePostComment(postId: string) {
  const { addOrUpdateComment } = useCommentStore(postId);
  const { socket } = useSocket();

  function post(text: string) {
    const tempId = crypto.randomUUID();
    addOrUpdateComment({ id: tempId, postId, text, authorId: currentUserId, status: 'sending', createdAt: new Date().toISOString() });

    socket.emit('comment:post', { tempId, postId, text }, (ack: CommentAck) => {
      addOrUpdateComment({ id: ack.serverId, postId, text, authorId: currentUserId, status: 'posted', createdAt: ack.serverTimestamp }, tempId);
    });
  }

  return { post };
}
```

**Reaction toggling — coalesced to net final state, with a sequence guard against out-of-order resolution:**

```tsx
function useReaction(commentId: string) {
  const [liked, setLiked] = useState(false); // optimistic local state
  const seqRef = useRef(0);
  const pendingRef = useRef<boolean | null>(null);

  const commitToServer = useDebouncedCallback((desiredState: boolean) => {
    const mySeq = ++seqRef.current;
    sendReactionToggle(commentId, desiredState).then((serverState) => {
      if (mySeq !== seqRef.current) return; // a later toggle has already superseded this request
      setLiked(serverState); // reconcile with whatever the server actually confirms
    });
  }, 400); // waits for rapid toggling to settle before sending anything

  function toggle() {
    const next = !liked;
    setLiked(next); // instant visual feedback on every tap, regardless of debounce
    pendingRef.current = next;
    commitToServer(next); // only the LATEST desired state, after settling, is ever actually sent
  }

  return { liked, toggle };
}
```

**Server-side (conceptual) batching of reaction-count updates for a high-volume post** — the client only ever needs to render whatever periodic snapshot arrives:

```ts
// Conceptual server-side sketch — not client code, shown to make the client's simplicity clear:
// individual reaction events are accumulated and flushed as one aggregate update per interval,
// rather than broadcasting each one individually to every connected client.
function flushReactionAggregates(intervalMs = 250) {
  setInterval(() => {
    for (const [postId, delta] of pendingReactionDeltas) {
      broadcast(postId, { type: 'reaction_count_update', newCount: currentCounts[postId] });
    }
    pendingReactionDeltas.clear();
  }, intervalMs);
}
```

**Client-side merge discipline for comments arriving via both pagination and live append, using the same ID-based dedup principle as the Chat and News Feed scenarios:**

```ts
function addOrUpdateComment(comment: Comment, replaceId?: string) {
  setComments((prev) => {
    const idToReplace = replaceId ?? comment.id;
    const existingIndex = prev.findIndex((c) => c.id === idToReplace);
    if (existingIndex !== -1) {
      const next = [...prev];
      next[existingIndex] = comment;
      return next;
    }
    if (prev.some((c) => c.id === comment.id)) return prev; // already present via the other path — drop it
    return insertBySequence(prev, comment);
  });
}
```

> **Check yourself:** Without looking above, explain why coalescing rapid reaction toggles to their *net final state* before sending is a better default than simply debouncing each individual toggle request, and what specifically it saves versus a naive per-tap send with a sequence guard alone.

## State & Data Model

```ts
interface Comment {
  id: string;
  postId: string;
  authorId: string;
  text: string;
  createdAt: string;
  status: 'sending' | 'posted' | 'removed'; // 'removed' covers later moderation takedown
}

interface ReactionState {
  commentId: string;
  count: number; // server-aggregated, may lag slightly behind individual events on a high-volume post
  viewerReaction: 'like' | 'love' | null; // the CURRENT user's own reaction, if any
}
```

`status: 'removed'` is deliberately part of the model rather than comments simply vanishing from an array with no trace — a moderation removal happening to a comment the user is currently looking at should render as an explicit "this comment was removed" placeholder (mirroring the Error Boundary scenario's principle that an abrupt, unexplained disappearance reads as broken, even when the underlying removal is entirely correct behavior).

## Scaling Considerations

**Batch/throttle server-to-client broadcast for high-volume posts, independent of client-side rendering optimizations.** No amount of clever client-side batching compensates for a server naively pushing an individual WebSocket message per raw event on a viral post — the aggregation needs to happen before broadcast, not be left for every connected client to independently absorb and throttle on their own.

**Virtualize the comment list once a thread grows long**, for the same reasons as the News Feed and Chat scenarios — a popular post can accumulate many thousands of comments over its lifetime, and mounting all of them simultaneously has the same DOM/memory cost problem addressed elsewhere in this phase.

**Consider approximate, less-frequently-updated counts for very high-volume posts as a deliberate, acceptable trade-off.** Rounding a reaction count to "1.2K" and updating it every second or two, rather than maintaining and rendering an exact integer that changes many times per second, both reduces render churn and better matches what a human viewer can actually perceive as meaningful change — this should be framed as an explicit, volume-dependent design choice rather than an accuracy compromise forced by limitation.

**Rate-limit how often any single client re-renders from a rapid stream of incoming aggregate updates**, even after server-side batching — if the server's batch interval is faster than a comfortable UI update cadence for some reason, the client can further coalesce consecutive updates that arrive within a short window before committing a re-render, though this is typically unnecessary if the server-side batch interval is already tuned sensibly.

## Gotchas

**Reaching for OT/CRDT-style conflict resolution for what is actually a much simpler append-only-plus-toggle problem.** A candidate who over-applies the previous scenario's machinery here is demonstrating pattern-matching rather than judgment — recognizing that comments/reactions don't have the same "concurrent edits to the same mutable content" shape as a collaborative document is the actual signal this scenario is testing for.

**Auto-appending new comments at the bottom regardless of where the user is currently scrolled.** Identical failure mode to auto-prepending in the News Feed scenario — disrupts a user reading older content, even if the specific disruption (content appearing below, rather than above, current scroll position) is slightly less jarring than the feed case; a banner or an anchored-to-bottom check should still gate this.

**Sending one network request per rapid reaction tap with no coalescing.** Produces both wasted request volume and a real risk of out-of-order resolution flipping the displayed state to the wrong final value — coalescing to net-final-state-after-settling addresses both problems at once.

**Broadcasting every individual reaction/comment event to every client unthrottled on a viral post.** Works fine at low volume and becomes a genuine scaling failure (server fan-out cost, client render thrashing) at exactly the volume ("going viral") the scenario explicitly asks the design to hold up under — this is the detail most likely to separate a strong answer from a merely adequate one here.

**Letting a moderation-removed comment simply vanish from the array with no trace, mid-session, for a user currently viewing it.** An unexplained disappearance of content a user was just reading reads as a bug even when it's the system working as intended (moderation did its job) — an explicit "removed" state and placeholder is the more honest, less confusing default.

**Not deduping comments that can arrive via both the initial/pagination fetch and the live-append stream.** A comment posted right around the time a user's initial page loads (or right around a "load older comments" pagination boundary) can plausibly be delivered by both paths — without ID-based dedup, it renders twice.

## Follow-up Questions

**Q (High): Why is this scenario's concurrency problem meaningfully simpler than the Collaborative Document Editor's, and what specifically would make it *not* simpler (i.e., under what added requirement would this scenario start needing similar machinery)?**

Answer: It's simpler because comments are independent, single-author, generally-immutable-once-posted entities being appended to an ordered list — there's no case (under the stated requirements) where two users are concurrently modifying the *same* piece of content in ways that need to be merged; ordering conflicts (whose comment is "first") are resolved trivially by a server-assigned sequence, with no transform/merge logic needed at all. It would start resembling the harder problem if, for example, comments became collaboratively co-editable by multiple authors simultaneously (not just editable by their own author), or if reactions/annotations needed to attach to arbitrary, concurrently-shifting *ranges* of a comment's text (rather than to a whole comment as an atomic unit) — at that point, the same "concurrent edits to shared, mutable content" shape reappears and would warrant revisiting whether CRDT-style machinery is actually needed for that specific sub-piece, even while the rest of the system (comment ordering, reactions-as-toggles) stays exactly as simple as described here.

The trap: either dismissing the comparison entirely ("they're just unrelated scenarios") or, in the other direction, applying document-editor-grade merge machinery here reflexively — the valuable answer identifies the *specific structural property* (single-author, atomic, non-concurrently-mutated content) that makes this scenario simpler, and can name what change would remove that property.

---

**Q (High): A user taps "like," then "unlike," then "like" again, all within under a second. Walk through exactly what should be sent over the network and what could go wrong with a naive per-tap implementation.**

Answer: A naive implementation sending one request per tap fires three requests in quick succession — even with a per-request sequence guard preventing the *final displayed state* from ending up wrong (the guard ensures only the last-resolved-and-still-current request's result is trusted), this still triggers three round trips, three server-side writes, and three opportunities for network jitter to have any one of them resolve unpredictably late, which is wasted work for an interaction whose *net effect* was "no change" (ending back at liked, having started at liked). The better approach coalesces the rapid sequence into a debounced commit — the visual/local state updates instantly on every tap (a user must see immediate feedback), but the actual network request is only sent once the sequence of taps has settled (a short debounce window with no further taps), and only reflects the *final* intended state at that point — reducing three requests to one in the common rapid-toggle case, and entirely sidestepping any resolution-order race for that burst of activity since only one request was ever sent.

The trap: solving only the "final state must be correct" half (via a sequence guard) while still sending three separate requests — this is a valid fix for correctness but misses the request-volume/coalescing optimization that a stronger answer identifies as the more complete solution to genuinely rapid, back-and-forth toggling.

---

**Q (High): The interviewer says: "This post just went viral — 50,000 concurrent viewers, and the reaction count is changing dozens of times per second. What breaks first, and how do you fix it?"**

Answer: What breaks first is server fan-out cost and client render churn from broadcasting every individual reaction event to every connected client — at that volume, pushing one WebSocket message per raw reaction event to 50,000 simultaneous connections is an enormous multiplication of message volume (one event × 50,000 recipients, repeated dozens of times per second), and even a client that receives all of it has no meaningful way to render "the count changed 40 times in the last second" as 40 distinct visual updates a human could perceive anyway. The fix is server-side aggregation before broadcast: accumulate raw reaction events into a running count, and periodically (e.g., every 250ms–1s, tuned to feel "live" without being wasteful) broadcast the *current aggregate count* as a single update per interval, regardless of how many raw events actually contributed to it — clients render a steadily, smoothly updating number rather than needing to process a firehose of individual events, and the server's broadcast volume becomes bounded by the batch interval rather than by raw event rate.

The trap: proposing only client-side fixes (throttling how often the client re-renders incoming updates) without addressing that the server is what needs to stop over-broadcasting in the first place — a client-side-only fix still requires the server to have sent (and every one of 50,000 clients to have received) every individual raw event, which doesn't reduce the actual bottleneck (server fan-out and network volume); the aggregation needs to happen upstream, before broadcast, not downstream, after over-delivery has already occurred.

---

**Q (Medium): How would you handle a comment that's optimistically shown as "posted" but is later removed by a moderator while the original author (or another viewer) is still looking at it?**

Answer: The removal needs to reach the client via the same live-update channel used for new comments (a `comment:removed` event carrying the comment's ID), and on receipt, the client updates that specific comment's status to `'removed'` in the normalized store rather than deleting the array entry outright — the rendered result is an explicit placeholder ("this comment was removed") in the same position, rather than the comment silently vanishing with no explanation, which would otherwise read as a bug (a layout shift with no visible cause) to whoever's currently viewing that part of the thread. This mirrors the general principle (also seen in the Chat and News Feed scenarios) that a change to already-rendered content, whatever its cause, should be reflected as an explicit state transition on that content's existing record, not an unexplained disappearance.

The trap: implementing removal purely as "the comment stops being returned by future fetches" with no live signal to clients currently rendering it — this technically achieves "it's gone from the data," but leaves it fully visible and interactive to anyone who already has it rendered, until their next unrelated re-fetch happens to drop it, which could be an arbitrarily long and confusing gap.

---

**Q (Medium): Should the comment count shown near a post (e.g., "1,204 comments") update in real time as new comments arrive, using the exact same live channel as the comment list itself?**

Answer: It can reuse the same underlying live-update signal, but is worth treating as a distinct, smaller piece of state from the comment list itself (a lightweight counter increment on each new-comment event, versus the full comment content) — this matters because the count is often rendered in places the full comment list isn't currently mounted (a post summary card in a feed, for instance), so it should be driven by its own subscription to "new comment arrived for post X" events rather than being derived only from the length of a comment array that might not even be loaded in that particular view. For a very high-volume post, the same batching/approximation trade-off discussed for reaction counts applies equally here — updating a live count on every single new comment for a viral post has the same broadcast-volume concern as unthrottled reaction events, and periodic aggregate updates are equally appropriate for the comment count.

The trap: coupling the comment count strictly to "the length of the currently-loaded comment array" — this breaks the moment the count needs to be shown somewhere the full list isn't loaded (a feed card, a notification), and doesn't hold up under the same viral-volume broadcast concerns that motivate batching elsewhere in this design.

---

**Q (Low): How would allowing a user to change their reaction (e.g., from "like" to "love") rather than a simple on/off toggle change the client-side state model?**

Answer: `viewerReaction` becomes a single value from a fixed set (`'like' | 'love' | 'laugh' | null`) rather than a boolean, and toggling logic changes from "flip true/false" to "set to the tapped type, or clear to `null` if the same type is tapped again while already active" — the aggregate count structure on the server side also needs to track counts per reaction type rather than one flat number, and the client renders whichever breakdown the product calls for (a single dominant-type count, or a small breakdown of each type's count). The coalescing/debounce and sequence-guard principles established for the simple like/unlike case apply identically here, just operating over a small enum of possible states instead of a boolean — rapid switching between reaction types by the same user should still coalesce to the net final choice before committing a network request, for the same reasons as before.

The trap: treating multi-type reactions as requiring a fundamentally different architecture from the boolean case — the state shape changes (enum vs. boolean) but the surrounding mechanisms (optimistic update, coalescing rapid changes, reconciling with server-confirmed state) carry over directly; recognizing that continuity is the stronger answer than re-deriving the whole approach from scratch.

---

## Self-Assessment

- [ ] Can articulate why this scenario's concurrency model is simpler than the Collaborative Document Editor's, and name the specific structural property that makes it so
- [ ] Can design optimistic comment posting with ID-based reconciliation, mirroring the Chat Application's message-send pattern
- [ ] Can design reaction-toggle coalescing to net final state and explain what it saves over a naive per-tap send with only a sequence guard
- [ ] Can explain why viral-scale event volume requires server-side batching before broadcast, not just client-side rendering optimizations
- [ ] Can design explicit handling for a comment removed by moderation while currently rendered, rather than letting it silently disappear
- [ ] Can explain the "anchored to live edge vs. reading history" distinction for whether new comments should auto-append or queue behind a banner

---
*Phase 4 (scenarios 1–8) complete for this session. Next up when you're ready: Design a Resumable, Chunked File Uploader — shifts from real-time delivery/merge problems to a single large, failure-prone transfer, where the central new concerns become chunking strategy, resumability after interruption, and upload progress/retry UX.*
