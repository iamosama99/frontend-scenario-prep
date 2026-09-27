# Design a Collaborative Document Editor (OT/CRDT Basics)

## Quick Reference

| Decision Point | Choice | Why |
|---|---|---|
| Concurrency model | CRDT (e.g., Yjs/Automerge) for a new build, over hand-rolled Operational Transformation | CRDTs merge concurrent edits deterministically without a central sequencing authority, work naturally offline/local-first, and — critically — come as mature, battle-tested libraries; OT is powerful but its transform functions are notoriously hard to get correct by hand, especially for rich text |
| Local edit responsiveness | Apply the user's own edit to local state immediately (optimistic), broadcast the resulting operation/update asynchronously | A collaborative editor that waits on a round trip before showing your own keystroke is unusable — local edits must never be gated on network latency |
| Remote edit integration | Incoming operations are merged into local state via the CRDT/OT algorithm, not by re-rendering the whole document from a fetched snapshot | A full-document diff/replace on every remote keystroke is both wasteful and loses local cursor/selection/undo-stack context; merges must be incremental |
| Cursor/presence | Broadcast as a separate, ephemeral signal (not part of the document's durable operation log), with positions re-mapped through subsequent operations | A collaborator's cursor position is meaningless once other edits shift the text around it — it must be transformed alongside the document, but isn't itself part of the document's persisted content |
| Build vs. buy | Use an existing CRDT library + rich-text editor framework (e.g., Yjs + ProseMirror/Slate/Tiptap) rather than implementing OT/CRDT merge logic from scratch | This is one of the few areas of frontend engineering where "don't reinvent it" is close to a hard rule — correctly implemented conflict-free merging is a research-grade problem; the value senior engineers add is architecting the integration, not re-deriving the algorithm |

## The Scenario

"Design the client-side architecture for a collaborative rich-text document editor — think Google Docs. Multiple users can be typing in the same document simultaneously, edits from each need to show up for everyone else in near real time, and the document must converge to the same final content for everyone regardless of the order edits happened to arrive in. Walk me through how you'd architect this, including how you'd handle conflicting concurrent edits."

## Clarifying Questions

- **Is this plain text, or does it need rich formatting (bold, italic, headings, lists, embedded images)?** Rich text is a materially harder merge problem than plain text — formatting applied to a *range* of text that's simultaneously being edited by someone else (inserting/deleting characters within or around that range) needs the range itself to be tracked and adjusted consistently, not just the raw character sequence.
- **Does the editor need to function offline, with edits merging correctly once a user reconnects after an extended disconnection?** This has a large influence on the concurrency-model choice — CRDTs are specifically well-suited to this ("local-first" software), since they can merge two divergent, independently-evolved document states with no central coordinator; a from-scratch OT implementation generally assumes a more continuously-connected, centrally-sequenced model and is a harder fit for long offline periods with independent local edits.
- **Are we expected to build the conflict-resolution algorithm from scratch, or is using an established library (Yjs, Automerge, or a hosted OT-based service) acceptable/expected?** This is worth surfacing explicitly rather than assuming — in almost every real engineering context, hand-rolling correct OT transform functions or a CRDT from first principles is the wrong call given how many subtly-incorrect edge cases exist in both approaches; a senior answer should default to "use a proven library and focus the actual design effort on integration," unless the interviewer explicitly wants the underlying algorithm derived.
- **Do we need to show other users' live cursor positions and selections while they type?** Presence (cursors, selections, "who's currently viewing") is a related but distinct requirement from content merging — it needs its own ephemeral broadcast channel and its own position-remapping logic as the underlying document changes.
- **Is undo/redo required, and if so, does undo need to correctly handle undoing your own edit after someone else has since edited the same region?** Collaborative undo is a known hard sub-problem — naively undoing "my last operation" by reverting to a prior document snapshot can silently destroy another user's concurrent edit if the two touched overlapping content; a correct answer needs a position/operation-aware undo, not a snapshot-based one.
- **What's the expected concurrency level — a handful of simultaneous editors, or potentially hundreds on the same document (e.g., a shared meeting-notes doc)?** Very high concurrency raises additional questions about how presence/cursor broadcast volume is throttled and how efficiently the merge algorithm scales with editor count, beyond what a two-or-three-person scenario needs to consider.

## Approach & Trade-offs

**Why naive "last write wins" or whole-document overwrite fails, as the starting point for explaining why this needs a real algorithm at all.** If two users are editing the same document and the client simply sends the full current document content on every change, with the server (or other clients) accepting whichever arrives last, one user's edits are silently destroyed the moment two edits are in flight concurrently — there's no actual "merge," just an overwrite race. Even a naive "send only the diff/patch of what changed" (rather than the whole document) doesn't fix the underlying problem: applying two independently-computed diffs, each computed against a document state that's since changed due to the *other* diff, produces a corrupted result unless the diffs are specifically transformed/adjusted to account for each other — which is precisely the problem OT and CRDTs each solve, via different mechanisms.

**Operational Transformation (OT), at a conceptual level: operations (not document snapshots) are the unit of communication, and concurrent operations are mathematically transformed against each other so they can be applied in any order and still converge to the same result.** An edit is represented as an operation like `insert(position, text)` or `delete(position, length)` rather than a resulting document string. When two operations happen concurrently (each computed against the same starting document state, before either has seen the other), applying both naively in sequence produces a wrong result if their positions aren't adjusted for each other's effect (e.g., if operation A inserts 5 characters at position 10, and operation B — computed concurrently, unaware of A — deletes characters at position 8–12, applying B's original position range after A has already shifted everything from position 10 onward would delete the wrong characters). OT's transform function takes two concurrent operations and produces adjusted versions of each that, when applied in either order, converge to the same final document. This is powerful but has a well-earned reputation for being extremely difficult to get exactly right by hand, especially once rich formatting operations (not just plain insert/delete) are involved — a large fraction of real-world OT implementations (including, historically, Google Docs' own) required years of correctness hardening.

**CRDTs (Conflict-free Replicated Data Types), at a conceptual level: the data structure itself is designed so that merging two independently-evolved copies is commutative, associative, and idempotent by construction — order of merge never matters.** Rather than transforming operations against each other, a CRDT-based text structure typically assigns each character (or each inserted "item") a globally unique, stable identifier tied to its logical position relative to its neighbors (implementations vary — some use fractional positions, some a linked-list-of-unique-IDs structure) such that inserting new content between two existing items never requires renumbering anything else, and two clients independently inserting near the same position end up with *both* insertions present, deterministically ordered relative to each other by their IDs, with no conflict to resolve and no central server required to sequence operations. This is what makes CRDTs a natural fit for offline/local-first editing — two clients that were disconnected for hours, each accumulating their own edits, can merge their two divergent document states directly against each other on reconnect, with no operation history replay needed and no central authority having to have seen every intermediate step in order.

**Neither should be implemented from scratch for a real product — this is the load-bearing trade-off answer for this scenario.** Both OT and CRDTs have decades of published research behind them precisely because getting either exactly right (including every edge case around concurrent formatting operations, concurrent deletes of overlapping ranges, and more) is genuinely hard, with many "looks correct in a demo, diverges under a specific concurrent edge case" implementations having been built and later found broken, including by teams with significant resources. The senior-level answer is to use a mature, widely-adopted library (Yjs and Automerge are the most common CRDT choices in the JS ecosystem; ShareDB is a common OT-based choice) and focus the actual system-design effort on the *integration* — wiring the library into a rich-text editor framework (ProseMirror, Slate, or a higher-level wrapper like Tiptap, all of which have existing Yjs/Automerge bindings), the transport layer (a WebSocket provider relaying updates between clients and/or a server), and the surrounding product concerns (presence, undo, persistence) — not re-deriving the underlying algorithm.

**Presence (cursors, selections) is architecturally separate from the document's content-merging mechanism, and needs its own position-remapping treatment.** A collaborator's cursor is meaningfully "at character 42" only relative to a specific document state — the instant someone else inserts or deletes text before that position, the raw number 42 no longer points at the same logical spot, so cursor positions broadcast to other clients need to be re-mapped through every subsequent operation applied since they were last known (most CRDT libraries expose exactly this as a primitive — a "relative position" that can be resolved to an absolute index against the current document state, tracking through edits automatically, rather than the application needing to hand-roll this transformation itself). Cursor/presence data is also typically treated as ephemeral (not part of the document's durable, persisted operation history) and broadcast over a lighter-weight channel, since losing a cursor-position update on a dropped connection has near-zero consequence compared to losing an actual content edit.

## Solution

**High-level client architecture, using Yjs (a common real-world CRDT library) conceptually, paired with a rich-text editor framework:**

```tsx
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';

function useCollaborativeDocument(docId: string) {
  const ydoc = useMemo(() => new Y.Doc(), [docId]);
  const provider = useMemo(
    () => new WebsocketProvider('wss://collab.example.com', docId, ydoc),
    [docId, ydoc]
  );

  useEffect(() => () => { provider.destroy(); ydoc.destroy(); }, [provider, ydoc]);

  return { ydoc, provider }; // handed to the editor framework's Yjs binding (e.g., y-prosemirror)
}
```

The application code never manually computes transforms or merge logic — `Y.Doc` is the shared, CRDT-backed document structure; `WebsocketProvider` handles broadcasting local changes and applying remote ones; the editor framework's binding (e.g., `y-prosemirror`) is what actually wires keystrokes in the rich-text editor to updates on the shared `Y.Doc`, and reflects remote updates back into the editor's rendered content incrementally.

**Presence/cursor broadcast, conceptually — a separate, ephemeral awareness channel most CRDT providers ship alongside the document sync:**

```tsx
function useCursorPresence(provider: WebsocketProvider, currentUser: User) {
  useEffect(() => {
    provider.awareness.setLocalStateField('cursor', {
      userId: currentUser.id,
      name: currentUser.name,
      color: currentUser.color,
    });

    function handleAwarenessChange() {
      const states = Array.from(provider.awareness.getStates().values());
      renderRemoteCursors(states); // each state's cursor position is already resolved against current doc state
    }

    provider.awareness.on('change', handleAwarenessChange);
    return () => provider.awareness.off('change', handleAwarenessChange);
  }, [provider, currentUser]);
}
```

**Conceptual sketch of why a naive diff-and-apply approach corrupts under concurrency** (illustrating the problem CRDTs/OT solve, not a recommended implementation):

```
Starting document: "Hello world"

User A (sees "Hello world"): inserts "there " at position 6 → intends "Hello there world"
User B (sees "Hello world", concurrently, unaware of A's edit): deletes "world" (positions 6-11) → intends "Hello "

Naive sequential apply of raw position-based edits (no transformation):
  Apply A's insert at position 6 → "Hello there world"
  Apply B's delete of positions 6-11 (computed against the ORIGINAL "Hello world") →
    deletes "there " instead of "world" → WRONG result: "Hello world"
    (B's intended deletion target has shifted because A's insert changed what's at those positions)

This is exactly the class of bug OT's transform functions and CRDTs' identity-based
(not position-index-based) addressing both exist to prevent.
```

> **Check yourself:** Without looking above, explain in your own words why applying two concurrently-computed, position-indexed edits in sequence (with no transformation) can corrupt the document, and describe — at a conceptual level, not necessarily the exact algorithm — how a CRDT's identity-based addressing avoids this specific failure.

## Architecture Overview

```
User types → local edit applied instantly to local CRDT doc (optimistic, no network wait)
           → CRDT update broadcast via WebSocket provider to server/other clients
           → other clients' providers receive the update → merged into their local CRDT doc
             (merge is conflict-free by construction — no central sequencing needed)
           → editor framework's binding reflects the merged doc state incrementally in the UI
           → cursor/presence updates flow over a separate, ephemeral "awareness" channel,
             with positions resolved against each client's current document state
           → server persists the CRDT document's state/update log for reload/late-joining clients
```

A server is still typically present in real deployments — not as a sequencing authority the way OT usually requires, but as a relay/persistence layer (storing the document's CRDT state so a client joining later doesn't need every other client to still be online, and so the document survives all clients disconnecting).

## Gotchas

**Attempting to hand-roll OT transform functions or a CRDT merge algorithm for a real product.** This is the single most consequential miss available in this scenario — both are genuinely hard to get exactly correct (particularly under concurrent operations on overlapping ranges, and especially once rich formatting is involved), and a large amount of prior art exists specifically because many teams have tried and produced subtly-incorrect implementations that only diverge under specific concurrent-edit sequences that don't show up in casual testing.

**Re-rendering the entire document from a fetched/merged snapshot on every remote update.** Destroys local cursor position, selection, undo stack, and scroll position on every keystroke from any other collaborator — remote updates need to be applied incrementally to the existing editor state (which is what CRDT-integrated editor bindings are specifically designed to do), not treated as "refetch and replace."

**Treating cursor/presence position as a static index that never needs updating.** A collaborator's cursor position is only valid relative to the document state it was reported against — failing to re-map it through subsequent local edits produces visibly wrong cursor positions (pointing at the wrong character, or an out-of-bounds position) the moment any edit happens near it.

**Implementing undo as "revert to my document snapshot from before my last edit."** In a collaborative context, another user may have already edited the same region since — a snapshot-based undo can silently discard their concurrent work; undo needs to be operation-aware (undoing specifically *this operation's* effect, computed relative to the current state, not reverting to a stale prior snapshot).

**Broadcasting cursor/presence updates as durable, persisted operations alongside actual content edits.** Cursor positions change far more frequently than content and have no long-term meaning once a session ends — persisting them in the same durable operation log as real content edits bloats storage and conflates two very different categories of state with very different durability/replay requirements.

## Follow-up Questions

**Q (High): Why is "send the whole document on every change, last write wins" fundamentally broken for this use case, and why doesn't sending only a diff/patch (instead of the whole document) fix it on its own?**

Answer: Sending the whole document and accepting whichever version arrives last means any edit made by another user in between is simply discarded the instant a later full-document write lands — there's no merging at all, just an overwrite race, and this is obviously wrong the moment two people type concurrently. Sending only a diff/patch is a genuine improvement (it at least attempts to describe *what changed* rather than the whole resulting state) but still fails on its own: a diff computed against a document state that's since been changed by someone else's concurrent diff describes positions/ranges that may no longer mean what they meant when the diff was computed — applying it naively can affect the wrong characters or corrupt content, exactly as shown in the worked example above. The actual fix requires either transforming concurrent operations against each other (OT) or representing content with identity that doesn't depend on shifting numeric positions at all (CRDTs) — "send a diff instead of the whole doc" reduces the size of what's transmitted but doesn't address the structural correctness problem of applying concurrently-computed changes without any reconciliation step.

The trap: treating "diff instead of full document" as if it were the actual fix for concurrent-edit correctness — it's an efficiency improvement over sending the whole document, entirely orthogonal to the conflict-resolution problem, and conflating the two suggests not having identified what actually causes the corruption (stale positional assumptions), only that "sending less data seems better."

---

**Q (High): Explain, at a conceptual level, how a CRDT-based text structure allows two users to insert content at "the same position" concurrently without either operation needing to know about the other in advance.**

Answer: Rather than addressing content by a numeric index that shifts every time something is inserted or deleted before it, a CRDT text structure gives each inserted item a stable, globally unique identity anchored to its logical neighbors (implementation details vary, but conceptually: "this character comes immediately after that other specific character's unique ID," not "this character is at position 42"). When two users concurrently insert new content near the same logical position, each insertion is recorded relative to whatever neighbor it was actually typed next to in that user's local view — when the two operations are later merged (in either order, on either client, at any point), both insertions are present, and a deterministic tie-breaking rule (commonly incorporating something like each operation's unique ID or originating client ID) decides their relative order to each other, without either client having needed to coordinate with or even be aware of the other at the time of typing. Because identity, not numeric position, is what's recorded and merged, applying the same set of insertions in a different order (which is exactly what happens across different clients receiving updates at different times) still converges to the same final result.

The trap: describing this in terms of "the CRDT figures out the correct index to insert at" — the entire point is that it deliberately avoids reasoning about numeric index at merge time at all; it's addressing by stable identity/relationship-to-neighbors that sidesteps the "index shifted underneath me" problem altogether, rather than a cleverer way of computing the right index.

---

**Q (High): Why is a from-scratch, correctly-implemented CRDT or OT algorithm considered one of the few areas where "don't build it yourself" is close to a hard engineering rule, rather than just a time-saving preference?**

Answer: The correctness bar for these algorithms is unusually strict and unusually hard to verify by ordinary testing — a merge algorithm needs to converge to the *same* result regardless of the order operations are applied in, across every possible interleaving of concurrent edits, including adversarial-looking edge cases (concurrent edits to overlapping ranges, concurrent formatting changes overlapping concurrent text edits, deeply nested concurrent structural changes in a rich document). A subtly incorrect transform function or merge rule can pass every test case someone thought to write and still diverge under a specific, rare concurrent sequence that only manifests in production with real simultaneous users — and by the time it's noticed, the failure mode is often silent data corruption or divergence between users' documents, not a visible crash that's easy to trace back to its root cause. Mature libraries (Yjs, Automerge) exist specifically because this problem has already been solved, extensively tested against real-world concurrent editing patterns, and hardened over years — re-deriving it from scratch re-takes on all of that risk for no product-differentiating benefit, since the actual value a product delivers is the editing experience built on top, not a novel conflict-resolution algorithm.

The trap: treating this as "we could build it if we had more time" — the point isn't that it's merely time-consuming; it's that correctness here is exceptionally difficult to verify with confidence, and the cost of getting it subtly wrong (silent document corruption for real users) is severe enough that "use the proven library" is the right default recommendation regardless of how much engineering time is available, not just the fast path under time pressure.

---

**Q (Medium): How would you implement collaborative undo/redo such that undoing your own last edit doesn't destroy a concurrent edit someone else made to nearby content?**

Answer: Undo needs to be modeled as "reverse the specific effect of this operation," computed and applied relative to the *current* document state, not as "restore the document to a snapshot from before this operation was originally applied" — the latter would discard anything anyone else has done since, concurrent or not. Most CRDT-integrated editor setups (Yjs specifically ships an `UndoManager` for this) track a per-user undo stack of *operations*, and undoing pops the most recent own-operation and computes/applies its inverse against the current merged state, which — because the underlying document is still a CRDT with the same conflict-free merge guarantees — correctly interacts with whatever else has changed concurrently, rather than requiring the undo logic to reason about "what else happened since" by hand.

The trap: implementing undo via document snapshots ("save the doc before each edit, restore the previous snapshot on undo") — this is the natural first instinct from non-collaborative editors, where it's perfectly correct, but in a collaborative context a snapshot restore silently reverts anything anyone else did in the interim too, which is a serious, easy-to-miss correctness bug specific to the collaborative case.

---

**Q (Medium): How would presence/cursor broadcast need to be throttled differently from content-edit broadcast, and why?**

Answer: Cursor/selection position changes far more frequently than a user's actual content commits (every arrow-key press, every mouse click, every text selection drag) and has essentially zero cost if a stale intermediate position is dropped or arrives late — this makes it appropriate to throttle/debounce cursor broadcasts fairly aggressively (sending, say, at most a few updates per second per user, coalescing rapid movement into the latest position rather than every intermediate one) with no retry or guaranteed-delivery requirement, the same "ephemeral, lossy is fine" treatment given to typing indicators in the chat-application scenario. Content edits, by contrast, need to be reliably delivered and merged (this is the CRDT/OT library's core guarantee) — dropping or losing an actual content operation is a real correctness problem, not an acceptable, low-stakes gap, so it's carried over a different reliability contract than presence updates, even if both happen to travel over the same underlying WebSocket connection.

The trap: applying the same reliability/throttling treatment to both categories — either over-engineering cursor broadcast with the same guaranteed-delivery rigor content edits need (wasted effort for data that's fine to lose), or under-engineering content-edit reliability by treating it as casually as cursor updates (a real correctness risk, since a dropped content operation, unlike a dropped cursor position, actually corrupts the shared document).

---

**Q (Low): If the interviewer says "assume the product only ever needs simple, plain-text co-editing with at most two simultaneous users, and offline support isn't required" — does the CRDT-over-OT recommendation still hold, or does the calculus change?**

Answer: With those constraints relaxed, a simpler centrally-sequenced OT-style approach (or even a much lighter-weight locking/turn-based scheme, depending on exactly how "simultaneous" needs to be) becomes more defensible, since the hardest parts of both approaches — correctly handling rich formatting interactions and correctly merging genuinely divergent, long-offline document states — are explicitly out of scope. That said, the recommendation to use an existing library rather than hand-rolling the merge logic still holds regardless of scale, because even "just plain text, two users" concurrent editing has enough real edge cases (simultaneous edits to the same character range, rapid back-to-back edits) that a hand-rolled solution risks the same class of subtle correctness bugs discussed above, just with a smaller blast radius if it goes wrong. The scope reduction changes *how much* infrastructure and product complexity (presence at scale, offline merge, rich formatting) needs to be built around the core mechanism — it doesn't change the recommendation to use a proven mechanism for the core mechanism itself.

The trap: concluding that a smaller/simpler scenario justifies hand-rolling OT/CRDT logic "since it's a simpler version of the problem" — the specific correctness hazards (positional corruption under concurrent edits) exist even in the simplest two-user, plain-text case; scope reduction shrinks the *surrounding* system's complexity, not the argument for not re-deriving a conflict-resolution algorithm from scratch.

---

## Self-Assessment

- [ ] Can explain, with a concrete worked example, why naive position-indexed diff-and-apply corrupts a document under concurrent edits
- [ ] Can describe, at a conceptual level, what Operational Transformation does (transforming concurrent operations against each other) and why it's hard to implement correctly
- [ ] Can describe, at a conceptual level, how CRDTs achieve conflict-free merging via identity-based addressing rather than numeric position
- [ ] Can justify using an established library (Yjs/Automerge + an editor framework binding) over a from-scratch implementation, and articulate why this is a strong default rather than a time-saving shortcut
- [ ] Can explain why cursor/presence broadcast is architecturally separate from content merging, including its different reliability/throttling treatment
- [ ] Can explain why collaborative undo requires operation-aware reversal rather than snapshot restoration

---
*Next: Design a Live Comments/Reactions Feed — returns to a more contained real-time problem (append-only comment/reaction streams under a single post) after this scenario's hardest case (arbitrary concurrent structural edits to shared content), useful for contrasting when full CRDT/OT machinery is actually warranted versus when a much simpler ordered-append model suffices.*
