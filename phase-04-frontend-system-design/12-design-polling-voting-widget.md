# Design a Polling/Voting Widget

## Quick Reference

| Decision Point | Choice | Why |
|---|---|---|
| Vote state model | A vote is a single mutable per-user record (which option, if any, this user currently has selected), not an append-only log of vote events | Unlike a comment or a chat message, a vote is meant to represent one user's *current* choice — changing your vote should update, not add to, your contribution to the tally |
| Vote submission | Optimistic local update (instantly show the user's vote reflected in the tally) reconciled against server-confirmed aggregate counts, with a sequence guard against out-of-order responses | The user must see their vote register instantly; the authoritative percentages/counts come from the server and may diverge slightly (other votes arrived concurrently) — the UI must reconcile without visible flicker |
| Rapid vote-changing (tapping between options quickly) | Debounce/coalesce to the user's net final choice before committing to the server, same as reaction toggling | Identical shape to the rapid reaction-toggle problem — sending one request per intermediate tap wastes requests and risks out-of-order resolution flipping the displayed tally to the wrong state |
| Live result updates as other users vote | Push-based aggregate count updates (WebSocket/SSE), batched at high volume, not one push per individual vote | Same over-broadcast principle as the Live Comments and Ticket Booking scenarios — a popular poll can receive votes fast enough that per-vote broadcasting doesn't scale and isn't perceptible as more "live" than a periodic aggregate update anyway |
| Preventing duplicate/multiple voting | Server-side enforcement keyed on user identity (or a durable anonymous identifier), never a client-side-only check | A client-side-only "already voted" flag is trivially bypassed by a page reload, a different browser, or clearing local storage — only the server can actually enforce a one-vote-per-user constraint |

## The Scenario

"Design a polling or voting widget — think a Twitter/X-style poll embedded in a post, or a live audience-voting feature during a streamed event. Users see the options and current results, cast a vote, and results update in real time as others vote. A single poll could get a very large number of votes in a short window if it goes viral. Walk me through the design."

## Clarifying Questions

- **Can a user change their vote after casting it, or is a vote final and locked once submitted?** This fundamentally shapes the state model — a changeable vote is a single mutable per-user record that can be updated; a locked-once-cast vote is closer to an append-only, immutable event, which is a simpler model in one sense (no need to handle "moving" a vote from one option's count to another) but a stricter one in another (the UI must prevent any further interaction with the options once voted, and communicate that finality clearly).
- **Are results visible to a user before they've voted, or only revealed after they cast their own vote?** Some polls hide results until you've voted (to avoid influencing your choice) and reveal them immediately after; others show live-updating results at all times regardless of whether you've voted yet. This affects both the initial data-fetch/render logic (whether aggregate counts are even sent to a client that hasn't voted) and the reveal transition's UX.
- **Is voting restricted to authenticated users only, or can anonymous/logged-out users vote too?** Anonymous voting still needs *some* way to prevent trivial multiple-voting (a durable device/browser-scoped identifier, commonly a signed cookie), which is inherently weaker and more bypassable than authenticated-user-based enforcement (an incognito window or a cleared cookie can re-vote) — worth naming as an accepted, bounded trade-off if anonymous voting is required, rather than assuming it can be as airtight as authenticated enforcement.
- **What's the expected scale — a modest poll among a known audience, or a genuinely viral one (a poll embedded in a widely-shared post) that could receive an extremely high vote rate in a short window?** Determines whether the same broadcast-batching concerns raised in the Live Comments and Ticket Booking scenarios need to be designed for from the outset or are a later scaling concern.
- **Does the poll close at a fixed time/vote count, and if so, what happens to a vote submitted right at the boundary?** A poll with a hard close needs an explicit, server-enforced cutoff (the server, not the client's local clock, must decide whether a given vote arrives before or after closing) to avoid a race where a client's optimistic UI shows a vote as accepted that the server actually rejected for arriving just past the deadline.

## Approach & Trade-offs

**A vote should be modeled as a single mutable per-user record, not an append-only event log, if votes are changeable — this is the key structural difference from the append-only comment/reaction-count models used elsewhere in this repo.** Where a comment is a genuinely new, independent entity each time, and a "like" reaction is a simple boolean toggle, a vote in a multi-option poll needs to represent "this specific user's current choice, among N options" — changing a vote isn't adding a new event, it's replacing which option this user's single vote currently counts toward (decrementing the previously-chosen option's count and incrementing the newly-chosen one). Recognizing this distinction up front avoids designing a naive vote-events-append-forever log that would then require deriving "each user's current choice" by finding their most recent event anyway — better to model the current-choice-per-user relationship directly as the source of truth, with aggregate per-option counts derived from (or maintained incrementally alongside) that relationship.

**Vote submission follows the same optimistic-update-plus-reconciliation pattern established throughout this phase, adapted to a multi-option tally rather than a boolean.** The instant a user taps an option, the client optimistically updates the locally-rendered tally (incrementing that option's displayed count, and decrementing the user's previous choice's count if they're changing an existing vote) — giving immediate visual feedback — while a request goes to the server to actually record the authoritative vote. The server's response (or a subsequent live-update push) carries the true aggregate counts, which the client reconciles against; because other users may have voted concurrently, the server-confirmed counts may differ slightly from what the client's optimistic update alone would predict, and the UI should smoothly adopt the server-confirmed numbers rather than visibly snapping or flickering between the optimistic guess and the confirmed reality.

**Rapid vote-switching between options needs the identical coalescing treatment as rapid reaction-toggling, for the identical underlying reason.** A user tapping option A, then quickly B, then back to A within a second or two is functionally the same "rapid back-and-forth on a shared toggle-like piece of state" problem addressed in the Live Comments scenario's reaction-toggle design — sending one vote-change request per intermediate tap risks the same out-of-order-resolution race (an earlier tap's request resolving after a later one, leaving the displayed/confirmed state wrong) and wastes request volume on intermediate choices the user didn't actually settle on. The fix is the same: debounce actual network submission to the user's *settled*, final choice, while still updating the local optimistic display instantly on every tap for responsive visual feedback.

**Live result updates at scale need the same server-side broadcast-batching discipline as the Live Comments and Ticket Booking scenarios, and for the same reason — this is a recurring pattern across every "shared, contended, high-frequency-updating state broadcast to many concurrent viewers" scenario in this phase.** A poll that goes genuinely viral can receive votes at a rate far exceeding what any individual observer needs (or is able) to perceive as discrete, separate updates — the server should aggregate rapid vote submissions into periodic snapshot broadcasts of updated percentages/counts, rather than pushing an update per individual vote to every connected client. This is worth stating as a recognized, repeating pattern rather than re-deriving from scratch each time it comes up — the specific triggering condition differs (a viral social post, a ticket on-sale rush, a viral poll) but the underlying mechanism and rationale are identical throughout this phase.

**Duplicate-vote prevention must be enforced server-side, keyed on a real identity concept, never trusted to client-side state alone.** A `localStorage` flag or a disabled button state after voting is a reasonable *UX* signal (preventing an accidental double-tap, or showing "you voted" styling) but provides zero actual enforcement — any user can trivially reload the page, clear storage, open an incognito window, or use a different browser to attempt to vote again, and a client-side-only check does nothing to stop this. The server must maintain the authoritative record of which identity (authenticated user ID, or for anonymous voting, some durable server-issued/signed identifier — accepting that this is inherently weaker and more bypassable than authenticated enforcement) has already voted on a given poll, and reject/ignore a duplicate vote attempt regardless of what the client believes its own state to be.

## Solution

**Vote state as a single mutable per-user record, with optimistic local tally update:**

```ts
interface PollState {
  pollId: string;
  options: { id: string; label: string; count: number }[];
  myVote: string | null; // this user's CURRENT choice, if any — not a log of past choices
  totalVotes: number;
}

function applyOptimisticVote(state: PollState, newOptionId: string): PollState {
  const options = state.options.map((opt) => {
    if (opt.id === state.myVote) return { ...opt, count: opt.count - 1 }; // remove from previous choice
    if (opt.id === newOptionId) return { ...opt, count: opt.count + 1 };  // add to new choice
    return opt;
  });
  const totalVotes = state.myVote === null ? state.totalVotes + 1 : state.totalVotes; // only +1 if this is a FIRST vote, not a change
  return { ...state, options, myVote: newOptionId, totalVotes };
}
```

**Debounced submission of only the user's settled final choice, with a sequence guard against out-of-order server responses:**

```ts
function usePollVoting(pollId: string, initialState: PollState) {
  const [state, setState] = useState(initialState);
  const seqRef = useRef(0);

  const commitVote = useDebouncedCallback((optionId: string) => {
    const mySeq = ++seqRef.current;
    submitVote(pollId, optionId).then((serverState) => {
      if (mySeq !== seqRef.current) return; // a later vote change has already superseded this request
      setState(serverState); // adopt server-confirmed authoritative counts
    });
  }, 400);

  function vote(optionId: string) {
    if (state.myVote === optionId) return; // no-op — already this user's current choice
    setState((prev) => applyOptimisticVote(prev, optionId)); // instant feedback on every tap
    commitVote(optionId); // only the LATEST settled choice is ever actually sent
  }

  return { state, vote };
}
```

**Live aggregate updates from other users' votes, merged in without disturbing this user's own optimistic state mid-flight:**

```ts
function usePollLiveUpdates(pollId: string, setState: React.Dispatch<React.SetStateAction<PollState>>) {
  useEffect(() => {
    const socket = connectToPollChannel(pollId);
    socket.on('poll:aggregate_update', (serverCounts: { options: { id: string; count: number }[]; totalVotes: number }) => {
      setState((prev) => ({ ...prev, options: prev.options.map((o) => ({ ...o, count: serverCounts.options.find((s) => s.id === o.id)?.count ?? o.count })), totalVotes: serverCounts.totalVotes }));
    });
    return () => socket.disconnect();
  }, [pollId, setState]);
}
```

**Server-side duplicate-vote enforcement (conceptual) — the client never gets to decide this on its own:**

```ts
// Conceptual server-side sketch, not client code — shown to make explicit what the client MUST NOT be trusted to enforce alone.
async function recordVote(pollId: string, voterIdentity: string, optionId: string) {
  const existing = await getExistingVote(pollId, voterIdentity);
  if (existing) {
    await changeVote(pollId, voterIdentity, existing.optionId, optionId); // decrement old, increment new, atomically
  } else {
    await castNewVote(pollId, voterIdentity, optionId); // increment new, increment totalVotes, atomically
  }
  return getAggregateCounts(pollId);
}
```

> **Check yourself:** Without looking above, explain why `applyOptimisticVote` must check whether `state.myVote` was `null` before incrementing `totalVotes`, and describe what visible bug results if a vote *change* (not a first vote) incorrectly increments the total.

## Gotchas

**Modeling votes as an append-only event log when votes are changeable, rather than a single mutable per-user record.** Forces deriving "this user's current choice" by scanning for their most recent event every time, and makes the decrement-old/increment-new update on a vote change more error-prone than modeling the current relationship directly.

**Incrementing the total vote count on every vote submission, including when an existing voter merely changes their choice.** Produces an inflated total that doesn't match the actual number of unique voters — the total should only increment on a genuinely first-time vote from a given identity, not on every vote-change event.

**Client-side-only duplicate-vote prevention (a disabled button, a `localStorage` flag) presented as if it were real enforcement.** Trivially bypassed by a reload, a different browser, or a cleared storage — real enforcement requires the server to key on an actual identity concept and reject duplicates authoritatively.

**Sending one network request per rapid tap when a user quickly switches between options multiple times.** The identical failure mode as unThrottled reaction-toggling in the Live Comments scenario — risks out-of-order resolution and wastes request volume; needs the same debounce-to-settled-final-choice treatment.

**Broadcasting every individual vote event, unbatched, to every connected client on a viral poll.** The same recurring over-broadcast scaling failure seen in the Live Comments and Ticket Booking scenarios — aggregate before broadcast, don't push per-event at high volume.

**No server-side enforcement of the poll's close time, relying only on the client hiding the voting UI after a local clock check.** A client whose local clock is wrong, or one that simply doesn't re-check the close time before submitting, can otherwise successfully submit a vote after the poll should have closed — the server must independently verify against its own authoritative close time.

## Follow-up Questions

**Q (High): A user rapidly taps option A, then B, then back to A, all within under a second. What exactly should be sent to the server, and why?**

Answer: This is functionally the same problem as the reaction-toggle rapid-tap scenario covered under Live Comments/Reactions — the local, optimistic tally should update instantly on every single tap (immediate visual feedback is non-negotiable), but the actual network request should be debounced to fire only once the sequence of taps has settled, and should reflect only the user's *final* settled choice, not each intermediate one. In this example, since the user ends up back at option A (their original choice, if they'd already voted A before this flurry of taps, or their first-ever vote if this is their first interaction), the net effect on the server should be either zero requests sent (if debounced logic recognizes no actual state change occurred) or exactly one request reflecting "vote for A" — never three separate requests for A, then B, then A again, which would be wasted work and carries the same out-of-order-resolution race risk addressed by the sequence guard.

The trap: sending one request per tap with only a sequence guard to fix the *final displayed state* — this prevents the tally from ending up visibly wrong, but still performs three round trips and three server-side writes for an interaction whose net effect was no actual change, missing the request-coalescing half of the correct answer.

---

**Q (High): How do you prevent a poll's aggregate counts from becoming inconsistent (e.g., percentages not summing to 100%, or a total that doesn't match the sum of option counts) under concurrent voting at scale?**

Answer: Consistency has to be guaranteed at the server's data-mutation layer, not hoped for at the client's rendering layer — every vote-change or new-vote operation (incrementing one option, decrementing another, adjusting the total) must happen as a single atomic operation server-side (a database transaction, or an atomic increment/decrement pair against the same row/document), so that no concurrent pair of votes can interleave in a way that leaves counts transiently or permanently inconsistent (e.g., two simultaneous vote-change operations both reading a stale prior count before either writes, a classic lost-update race). On the client side, because the server is the single authoritative source recomputing consistent aggregate counts on every mutation, the client's job is simply to render whatever consistent snapshot the server last confirmed or broadcast — the client never needs to (and should not attempt to) independently derive or "fix up" percentages from partial/possibly-stale local state; a periodic full-aggregate broadcast (rather than incremental deltas the client would need to apply and could drift from) is a more robust choice specifically because it side-steps any risk of client-side derived-state drift accumulating over many incremental updates.

The trap: assuming client-side reconciliation logic (recomputing percentages defensively from whatever counts happen to be in local state) is an adequate substitute for server-side atomicity — no amount of client-side care can fix data that was already made inconsistent by an unsynchronized concurrent write on the server; the atomicity guarantee has to exist at the point of mutation, not be patched over afterward at render time.

---

**Q (Medium): Should results be shown to a user before they've voted, and how does that choice affect the initial data the client fetches?**

Answer: If the product hides results until the user has voted (a common poll pattern intended to avoid influencing an undecided voter's choice), the initial fetch for a not-yet-voted user should genuinely omit aggregate counts from the response entirely (sending only the option labels), not send the real counts and merely hide them in the UI — a client-side-only hide is trivially bypassed by inspecting the network response, and defeats the actual product intent (preventing influence) if a curious or technically-inclined user can see the real numbers anyway via devtools. Once the user votes, a subsequent response (the vote-submission response itself, or an immediately-following fetch) can then include the real aggregate counts for the reveal transition. This is a case where the correct backend behavior (omitting data, not merely client-side-hiding it) matters more than the frontend rendering logic, and is worth naming explicitly since it's an easy detail to get wrong by assuming a client-side conditional render is sufficient.

The trap: implementing "hide until voted" purely as a client-side conditional (fetch the real counts always, just don't render them until `myVote` is set) — this technically achieves the visible UI behavior for a typical user but leaks the actual results to anyone inspecting network traffic, undermining the actual reason a product might want results hidden pre-vote in the first place.

---

**Q (Medium): How would you handle a poll that closes at a specific time, given that a client's local clock can't be trusted to be perfectly accurate?**

Answer: The close time must be enforced server-side as the authoritative check — a vote submission arriving after the server's own clock considers the poll closed should be rejected regardless of what the submitting client's local UI believed about the poll's status (e.g., if the client's clock is running a few seconds behind and its local countdown hadn't yet reached zero when the user tapped vote). The client-side countdown/close-time display is a UX convenience for informing the user when a poll will close, not the actual enforcement mechanism — and the client should handle a late-rejected vote submission (one that was permitted locally but rejected by the server as arriving after close) with an explicit, honest message ("this poll has closed") rather than a generic error, mirroring the same principle of specific, actionable failure messaging discussed in the Ticket Booking scenario.

The trap: relying solely on the client hiding the voting UI once its local countdown reaches zero, with no server-side verification of the actual submission time against the poll's true close time — a client with a slow or fast clock, or a request that's simply in flight right at the boundary, can otherwise successfully record a vote the poll's actual close time should have excluded.

---

**Q (Low): How would this design change for a "live audience voting during a streamed event" variant, where the poll opens and closes on a very short timer (e.g., 30 seconds) and results need to be shown live as votes come in, second by second?**

Answer: The core mechanisms are unchanged (optimistic local update, server-authoritative aggregate, debounced coalescing of rapid taps, batched broadcast at scale) — what changes is mainly tuning: the broadcast batching interval likely needs to be tighter (updating more frequently, e.g., every 200–500ms rather than every few seconds) to feel appropriately "live" against a fast-moving, short-duration event, and the UI needs a highly visible, second-accurate countdown given the much shorter overall window compared to a typical social-media poll that might stay open for hours or days. The duplicate-vote and vote-change logic is identical regardless of the poll's total duration — a 30-second live poll and a 3-day social poll are the same underlying state machine, just with different time constants tuned to their respective contexts.

The trap: treating the short-duration, high-tempo live-voting variant as requiring a fundamentally different architecture — it doesn't; recognizing that the same building blocks apply with different tuning parameters (batch interval, countdown granularity) is the stronger answer than re-deriving a new design from scratch for what is, structurally, the same problem on a faster clock.

---

## Self-Assessment

- [ ] Can explain why a changeable vote should be modeled as a single mutable per-user record, not an append-only event log, and what bug results from getting this wrong
- [ ] Can design optimistic vote submission with server-reconciliation, and explain why total-vote-count increments only on a genuinely first vote
- [ ] Can design debounced/coalesced rapid vote-switching and connect it explicitly to the reaction-toggle pattern from the Live Comments scenario
- [ ] Can explain why duplicate-vote prevention must be server-side and why a client-side-only check provides no real enforcement
- [ ] Can explain why atomicity of the count-mutation operation must live server-side, not be patched over via client-side reconciliation
- [ ] Can name the over-broadcast scaling pattern shared across this scenario, Live Comments, and Ticket Booking, and describe its standard mitigation

---
*Next: Design an Instagram Stories-style Component — moves away from this phase's recent concurrency/contention thread entirely into a different category of problem: sequenced, timed autoplay media with preloading, gesture-based navigation, and per-viewer progress state, where the central new concerns become media preloading strategy and precise timer/pause coordination rather than server-arbitrated contention.*
