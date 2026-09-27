# Design a Ticket/Seat Booking UI

## Quick Reference

| Decision Point | Choice | Why |
|---|---|---|
| Concurrency model for a finite, contended resource | Server-side temporary "hold" on selection (short-lived reservation lock), with client-visible countdown, rather than optimistic client-side selection with no server involvement until final purchase | Two users can select the same seat within milliseconds of each other; only the server can arbitrate who actually gets it, and the client needs to know *immediately*, not just at final checkout time, whether its selection is still valid |
| Real-time availability sync across concurrent viewers | Push-based live updates (WebSocket/SSE) marking seats as held/sold as other users act, not polling | A seat map is exactly the kind of shared, fast-changing view where showing another user a seat as "available" when it was taken 400ms ago produces a jarring failed-selection-at-checkout experience — the same class of problem addressed by real-time delivery in the Chat and Live Comments scenarios |
| Hold expiry | Fixed-duration hold (e.g., 5–10 minutes) starting when a seat is selected, visibly counted down to the user, auto-released back to available on expiry or explicit deselection | Without an expiry, a user who selects seats and abandons the flow (closes the tab, walks away) would lock those seats unavailable indefinitely — the hold must be a lease, not a permanent claim |
| Seat map rendering at scale (large venues, tens of thousands of seats) | Virtualized/canvas-based rendering rather than one DOM node per seat for very large venues | A stadium-scale seat map can have tens of thousands of individually interactive seats; naive one-DOM-element-per-seat rendering at that scale is a real performance problem, distinct from the concurrency problem |
| Failure at final purchase (hold expired, or seat taken despite a live hold — a race at the edge) | Explicit, named failure state distinguishing "your hold expired" from "payment failed" from "seat became unavailable" — never a generic error | Each failure mode has a different correct recovery action (reselect seats vs. retry payment vs. pick different seats) — collapsing them into one generic error message leaves the user unable to act correctly on the failure |

## The Scenario

"Design the seat selection and booking flow for an event ticketing platform — think buying tickets to a concert or a flight seat map. Users see a visual map of seats, select one or more, and complete a purchase. Multiple users are looking at and selecting from the same seat inventory simultaneously, and popular events can have very high concurrent demand right when tickets go on sale. Walk me through the architecture, especially how you'd handle the concurrency."

## Clarifying Questions

- **How long should a selected-but-not-yet-purchased seat be held before it's released back to availability for others?** This single number (the hold duration) is the central trade-off of the whole design: too short frustrates legitimate users who need time to review their selection and enter payment details; too long means seats sit artificially unavailable to other buyers during high-demand on-sale moments, which is precisely the situation where availability matters most. Worth surfacing as an explicit, tunable product decision rather than an arbitrary constant.
- **What's the expected concurrency at the moment tickets go on sale — a steady trickle of bookings, or a "thundering herd" where thousands of users hit the same seat map within seconds of an announced on-sale time?** The thundering-herd case (a genuinely popular event's on-sale moment) is a materially harder problem than steady-state booking — it's the scenario where naive real-time seat-map updates broadcasting every single hold/release event to every connected client can itself become a scaling bottleneck, echoing the same over-broadcast concern raised in the Live Comments/Reactions scenario for a viral post.
- **Can a user select and hold multiple seats in one transaction, and if the group can't all be secured (e.g., someone else grabs one mid-selection), what should happen — is a partial booking acceptable, or must it be all-or-nothing?** All-or-nothing group holds are a meaningfully different transactional requirement than independent per-seat holds, and materially changes what the server-side hold/lock mechanism needs to guarantee.
- **Is seating "assigned" (specific numbered seats, as in this scenario) or "general admission" (just a quantity, no specific seat)?** General admission removes the entire per-seat-contention problem this scenario is centrally about, reducing it to a simpler inventory-count decrement — worth confirming the scenario is actually about assigned seating before designing around seat-level contention.
- **What happens if a user's hold is about to expire while they're actively in the payment step (e.g., their card is being processed) — is there any grace period or extension, or does the hold strictly expire on its timer regardless of in-progress payment?** A strict, no-exceptions expiry risks failing a legitimate in-progress purchase due to unlucky timing; some systems extend or pause the hold timer once payment processing has actually begun, which is a real UX and architecture decision worth surfacing.
- **How large is the venue (dozens of seats vs. a stadium with tens of thousands)?** Determines whether naive per-seat DOM rendering is fine as-is or needs the virtualization/canvas treatment discussed below.

## Approach & Trade-offs

**Seat selection must create a server-side hold immediately on selection, not only at final checkout — the client cannot be the arbiter of a contended, finite resource.** The instant a user clicks an available seat, the client should optimistically show it as "selected by you" and simultaneously fire a request to place a short-lived, server-enforced hold on that seat — the server is the only party that can correctly resolve two users clicking the same seat within milliseconds of each other, since only it has a single, authoritative view of the seat's current state. If the hold request is denied (someone else's hold beat this one to the server, even by a small margin), the client must immediately roll back its optimistic selection and inform the user that seat just became unavailable — waiting until final purchase to discover a selected seat was never actually securable produces a much worse experience (a user believing they'd successfully chosen seats through an entire payment flow, only to be told at the very last step that a selection made minutes ago silently was never valid) than surfacing the conflict the moment it's known.

**The hold is a time-limited lease, not a permanent claim, and this needs to be visible and honest to the user, not a silent backend timer.** A held seat reverts to available automatically once its hold expires — critically, the client must render a visible countdown (not just enforce the expiry silently server-side) so the user understands they're operating under a deadline and isn't blindsided by their selection disappearing with no warning. This is a genuine UX tension: too generous a hold duration keeps seats artificially locked away from other buyers during exactly the high-demand moments where every available seat matters most; too aggressive a duration frustrates legitimate users who need real time to review a multi-seat selection and enter payment information. There's no universally correct number — the design should treat it as an explicit, tunable parameter, and the UI must make the countdown and its consequence (seat released back to the pool) unambiguous.

**Real-time seat-map sync across concurrent viewers needs push-based updates, but at high-demand on-sale volume, needs the same over-broadcast discipline as the Live Comments scenario's viral-post case.** As other users hold or release seats, every client currently viewing that seat map needs its view updated live — a seat shown as "available" that was actually just taken 400ms ago, discovered only when the user attempts to select it, is a jarring, avoidable experience that push-based delivery (WebSocket or SSE) directly solves, the same principle underlying real-time delivery throughout this phase. At genuinely high concurrency (a popular event's on-sale moment, potentially thousands of simultaneous viewers on the same seat map), broadcasting every individual seat-state change to every connected client is the same over-broadcast risk seen in [Design a Live Comments/Reactions Feed](08-design-live-comments-reactions-feed.md)'s viral-post scenario — the mitigation is analogous: batch rapid-fire seat-state changes into periodic snapshot updates rather than one push per individual hold/release event, and/or partition broadcast by seat-map section (a user looking at the mezzanine doesn't need live updates about orchestra-section seat changes) to reduce the volume any single client actually needs to receive.

**All-or-nothing multi-seat holds require the server-side hold operation itself to be atomic across the whole requested group, not a sequence of independent per-seat holds.** If a user selects three seats together, the server must attempt to hold all three as a single transactional operation — either all three succeed (this user now holds all three) or, if even one is already held/sold by the time the request is processed, the whole group-hold attempt fails and none of the three are held for this user (rather than silently succeeding on two and failing on the third, leaving the user with a partial, likely-undesired selection). This is a meaningfully different guarantee than the independent per-seat hold model and needs to be an explicit part of the server contract if the product requires all-or-nothing group selection.

**Distinguishing failure modes precisely at the payment/finalization step is what separates a genuinely well-designed booking flow from one that merely works in the happy path.** "Your hold expired," "this seat became unavailable" (a rare race at the very edge of the hold window), and "payment was declined" are three entirely different failure causes requiring three entirely different recovery actions from the user (reselect seats and restart the hold, pick different seats, or retry/fix payment details) — collapsing them into one generic "something went wrong, please try again" error leaves the user unable to act correctly, and is a common, easily-avoided shortcoming in booking flow design.

## Solution

**Optimistic seat selection with immediate server-side hold request and rollback on denial:**

```tsx
function useSeatSelection(eventId: string) {
  const [selectedSeats, setSelectedSeats] = useState<Map<string, SeatHoldStatus>>(new Map());

  async function selectSeat(seatId: string) {
    setSelectedSeats((prev) => new Map(prev).set(seatId, 'pending')); // instant optimistic feedback

    try {
      const hold = await requestSeatHold(eventId, seatId); // server-authoritative — the only real arbiter
      setSelectedSeats((prev) => new Map(prev).set(seatId, { status: 'held', expiresAt: hold.expiresAt }));
    } catch (err) {
      setSelectedSeats((prev) => {
        const next = new Map(prev);
        next.delete(seatId); // roll back — someone else's hold won the race
        return next;
      });
      notifySeatUnavailable(seatId);
    }
  }

  function deselectSeat(seatId: string) {
    releaseSeatHold(eventId, seatId); // explicit release — don't make the user wait out the timer
    setSelectedSeats((prev) => { const next = new Map(prev); next.delete(seatId); return next; });
  }

  return { selectedSeats, selectSeat, deselectSeat };
}
```

**Visible, honest hold countdown — never a silently-enforced backend-only timer:**

```tsx
function HoldCountdown({ expiresAt, onExpire }: { expiresAt: string; onExpire: () => void }) {
  const [remainingMs, setRemainingMs] = useState(() => new Date(expiresAt).getTime() - Date.now());

  useEffect(() => {
    const interval = setInterval(() => {
      const remaining = new Date(expiresAt).getTime() - Date.now();
      setRemainingMs(remaining);
      if (remaining <= 0) { clearInterval(interval); onExpire(); }
    }, 1000);
    return () => clearInterval(interval);
  }, [expiresAt, onExpire]);

  const minutes = Math.floor(Math.max(0, remainingMs) / 60000);
  const seconds = Math.floor((Math.max(0, remainingMs) % 60000) / 1000);
  return <span aria-live="polite">Seats held for {minutes}:{String(seconds).padStart(2, '0')}</span>;
}
```

**Live seat-map sync — merging server-pushed state changes from other users into the local seat map:**

```ts
function useSeatMapSync(eventId: string, onSeatStateChange: (seatId: string, status: SeatStatus) => void) {
  useEffect(() => {
    const socket = connectToSeatMapChannel(eventId);
    socket.on('seat:held', ({ seatId }) => onSeatStateChange(seatId, 'held_by_other'));
    socket.on('seat:released', ({ seatId }) => onSeatStateChange(seatId, 'available'));
    socket.on('seat:sold', ({ seatId }) => onSeatStateChange(seatId, 'sold'));
    return () => socket.disconnect();
  }, [eventId, onSeatStateChange]);
}
```

**Distinct, named failure states at finalization — never a single generic error:**

```ts
type BookingFailure =
  | { type: 'hold_expired'; seatIds: string[] }       // recovery: restart selection
  | { type: 'seat_unavailable'; seatIds: string[] }   // recovery: pick different seats — rare edge-of-window race
  | { type: 'payment_declined'; reason: string };      // recovery: retry/fix payment details

function renderBookingFailure(failure: BookingFailure) {
  switch (failure.type) {
    case 'hold_expired':
      return `Your hold on ${failure.seatIds.length} seat(s) expired. Please reselect.`;
    case 'seat_unavailable':
      return `${failure.seatIds.length} seat(s) just became unavailable. Please choose different seats.`;
    case 'payment_declined':
      return `Payment couldn't be processed: ${failure.reason}. Please try again.`;
  }
}
```

> **Check yourself:** Without looking above, explain why the client must still roll back its optimistic selection even after successfully requesting a hold, if a later `seat:sold` or hold-expiry event arrives for that same seat — i.e., why the initial hold grant is not the last word on that seat's state.

## Data Model — Group (All-or-nothing) Holds

```ts
interface SeatHoldRequest {
  eventId: string;
  seatIds: string[];    // all seats in ONE atomic hold request when group selection must be all-or-nothing
  sessionId: string;    // ties the hold to this user's current booking session
}

interface SeatHoldResult {
  success: boolean;
  heldSeatIds: string[];     // empty if success is false — no partial holds granted
  failedSeatIds: string[];   // which of the requested seats were already unavailable, for user-facing messaging
  expiresAt?: string;
}
```

The server must implement the group hold as a single atomic operation (e.g., a database transaction acquiring locks on all requested seat rows, committing only if every one is currently free) — never as a loop issuing independent per-seat hold calls, which would risk exactly the partial-success outcome the all-or-nothing requirement exists to prevent.

## Scaling Considerations

**Virtualize or canvas-render the seat map for large venues.** A stadium-scale map with tens of thousands of individually interactive seats rendered as one DOM element each is a real performance liability — visible-region-only DOM rendering (the same windowing principle as [Virtualized List (Windowing) From Scratch](../phase-02-component-machine-coding/03-virtualized-list-windowing-from-scratch.md), applied to a 2D spatial layout rather than a linear list) or rendering the seat map on `<canvas>` entirely (trading DOM interactivity for raw rendering throughput, with hit-testing done manually against seat coordinates) are the two standard approaches at that scale; a venue with a few hundred seats has no such concern and can render every seat as a plain DOM element with no special handling.

**Batch and/or partition real-time seat-state broadcasts during high-concurrency on-sale moments.** Directly analogous to the Live Comments scenario's viral-post mitigation — aggregate rapid-fire hold/release/sold events into periodic snapshot broadcasts rather than one push per individual event, and where practical, partition the broadcast by seat-map section so a client only receives updates relevant to the section currently in view.

**Consider a queueing/waiting-room mechanism ahead of the seat map itself for extreme on-sale demand**, independent of anything covered so far — at sufficiently high simultaneous demand (far exceeding available inventory), some ticketing platforms place users in an explicit virtual queue before even granting access to the live seat map, which is as much a backend/infrastructure decision as a frontend one but worth naming as the next lever beyond seat-map-level optimization once concurrency is high enough that the seat map itself becomes the bottleneck.

## Gotchas

**Treating the client-side optimistic selection as sufficient, with no actual server-side hold until final checkout.** This is the central, most serious failure mode for this specific scenario — it allows two users to both believe they've selected the same seat, with the conflict surfacing only at final purchase (the worst possible moment to discover it), rather than the instant the selection was made.

**A hold that expires silently, with no visible countdown communicated to the user.** A seat disappearing from a user's selection with no forewarning reads as a bug (or as their selection being unfairly revoked) even when the hold's expiry is working exactly as designed — the countdown must be visible, not merely enforced.

**Implementing group (multi-seat) holds as a sequence of independent per-seat hold calls rather than one atomic operation.** Risks exactly the partial-success outcome an all-or-nothing requirement exists to prevent — a user could end up holding two of three requested seats, an outcome the product may have explicitly wanted to avoid.

**Collapsing hold-expired, seat-became-unavailable, and payment-declined into one generic error message at final purchase.** Each requires a different recovery action from the user; a generic error leaves them unable to determine what to actually do next.

**Broadcasting every individual seat hold/release/sale event, unbatched, to every connected client during a high-demand on-sale moment.** The same over-broadcast scaling failure as the Live Comments scenario's viral-post case, just triggered by ticket-sale demand instead of social engagement volume.

**Rendering a large venue's seat map as one DOM element per seat with no virtualization or canvas fallback.** A real, measurable performance problem at stadium scale, distinct from and additional to the concurrency-handling concerns that are this scenario's primary focus.

## Follow-up Questions

**Q (High): Two users click the exact same seat within 50ms of each other. Walk through exactly what happens, end to end, for both of them.**

Answer: Both clients optimistically mark the seat as "selected/pending" locally and immediately fire a hold request to the server. The server processes these two nearly-simultaneous requests using whatever concurrency-safe mechanism it uses for seat state (e.g., a database row lock, or an atomic compare-and-set on the seat's status field) — exactly one of the two requests wins, acquiring the hold; the other's request is rejected because the seat's status was no longer "available" by the time the server evaluated it. The winning client receives a successful hold response and transitions its optimistic "pending" state to a confirmed "held" state with a visible countdown. The losing client receives a rejection, immediately rolls back its optimistic selection (the seat is removed from that user's selected-seats list), and shows an explicit "this seat just became unavailable" message — critically, this should happen automatically upon the request's rejection, not require the user to notice anything is wrong on their own. Additionally, both clients (and every other client currently viewing this seat map) should shortly receive a live `seat:held` push event confirming the seat is now held by someone, so anyone else who might have been about to click it sees it become unavailable proactively rather than discovering it only if they happen to also attempt to select it.

The trap: describing only the server-side arbitration (who wins the race) without also describing the losing client's required UI rollback and messaging — a candidate who stops at "the server picks a winner" hasn't addressed what the losing user actually experiences, which is the harder and more interview-relevant half of the answer.

---

**Q (High): How would you handle the case where a user's hold is about to expire while they're in the middle of submitting payment?**

Answer: This is a genuine design decision with a real trade-off, not a case with one obviously correct answer, and a strong response names the trade-off explicitly: a strict, no-exceptions expiry (the hold releases exactly on schedule regardless of in-progress payment) is simple and fair to other buyers waiting for that seat, but risks failing a legitimate purchase due to unlucky timing (a slow payment processor response landing just after expiry) even though the user did everything right within the time given. The alternative — pausing or extending the hold timer once payment submission has actually begun (e.g., extending it by a fixed grace window the moment a "confirm purchase" action is submitted, since at that point the user has committed and further seat contention becomes less of a concern) — better serves the in-progress buyer but requires the server to track and honor this additional state transition correctly, and needs its own safeguard against indefinite extension (e.g., a payment attempt that itself hangs or never resolves must not hold the seat forever). Most real ticketing systems lean toward some form of the latter (a short grace period once payment is actively being processed) specifically because failing a near-complete purchase due to a timer technicality is a worse outcome than briefly holding a seat slightly past its nominal expiry.

The trap: treating the hold timer as untouchable and payment as something that just has to complete within it — a candidate who doesn't consider extending/pausing the timer during active payment submission is missing a real, common product/UX consideration that materially affects conversion on exactly the highest-friction step of the flow.

---

**Q (High): The interviewer says: "Tickets for a major event go on sale, and 50,000 people load the seat map within the same 10 seconds. What's your biggest concern, and how do you address it?"**

Answer: The biggest concern is the same over-broadcast scaling failure discussed in the Live Comments scenario's viral case, but potentially worse here because seat-state changes at that moment are happening at extremely high frequency (every successful hold instantly makes a seat unavailable to everyone else, and with 50,000 concurrent viewers, holds are happening continuously) — broadcasting every individual hold/release event to all 50,000 connected clients individually is both a server fan-out problem and a client-side render-churn problem. The mitigation mirrors the Live Comments approach: batch rapid seat-state changes into periodic snapshot broadcasts (e.g., every few hundred milliseconds, push "here's what changed in this window" rather than one message per event) and additionally partition broadcasts by seat-map section where practical, so a client viewing one section of a large venue isn't receiving update volume proportional to activity across the *entire* venue. A secondary, complementary mitigation independent of broadcast volume is a pre-seat-map queueing/waiting-room mechanism that limits how many users are concurrently interacting with the live seat map at all during the initial rush, reducing the peak concurrency the seat-map system itself needs to handle simultaneously.

The trap: proposing only client-side rendering optimizations (e.g., debouncing re-renders from incoming updates) without addressing that the server-side broadcast volume itself is the primary bottleneck at that concurrency — as in the Live Comments follow-up, the aggregation needs to happen before broadcast, not be left to each of 50,000 clients to independently absorb.

---

**Q (Medium): Should the seat-hold mechanism be implemented purely client-side (e.g., a `localStorage`-based lock others' clients somehow respect) or does it need genuine server-side enforcement — and why does this matter even if you trust all users to behave honestly?**

Answer: It requires genuine server-side enforcement, full stop — a client-side-only "lock" (whether in `localStorage`, a cookie, or any other browser-side mechanism) has no way to actually prevent or even reliably detect another user's browser, on another machine, from independently believing it has also secured the same seat; there is no shared state between two different users' browsers without a server in between. This isn't a matter of trusting users to behave honestly (client-side state can't enforce a rule against a *different* user's independent client at all, honest or not) — it's a fundamental architectural requirement that the single source of truth for a globally shared, contended resource must live in exactly one place all clients defer to, which can only be the server.

The trap: treating this as primarily a trust/security concern ("what if a user tampers with their own client-side lock") rather than recognizing the more basic point — even a perfectly honest, non-malicious client-side-only implementation cannot work at all, because two different users' browsers have no shared state to coordinate through without a server.

---

**Q (Medium): How would you design the seat map to remain usable and clear when many seats are simultaneously being held by other users in real time, without becoming visually noisy or confusing?**

Answer: The seat map needs a small, fixed set of clearly distinguishable visual states (available, held-by-you, held-by-someone-else, sold) applied consistently, and incoming live updates should transition a seat between these states smoothly rather than, for instance, flashing or drawing attention to every single change individually — a seat map genuinely under high contention can have many seats changing state within a short window, and treating each transition as an attention-grabbing event (an animation, a highlight flash) at that volume would produce a visually chaotic, distracting experience rather than a helpfully "live-feeling" one. A calmer default (a simple, immediate color/state change with no per-change animation once volume is high) scales better under heavy concurrent activity, while still being demonstrably live and current.

The trap: designing an eye-catching per-seat-change animation or highlight as the default without considering how it behaves under genuinely high concurrent-update volume — a treatment that looks polished in a low-activity demo can become overwhelming and counterproductive during the exact high-demand moment (a popular event's on-sale) this scenario centers on.

---

**Q (Low): Would you use the same hold/expiry mechanism for a low-demand, always-available inventory item (e.g., general admission tickets with ample supply) as for high-demand assigned seating?**

Answer: Not necessarily to the same degree — the entire hold/expiry/real-time-sync mechanism exists specifically to arbitrate contention over a *finite, uniquely-identified* resource under real demand pressure; for general admission with ample available supply, there's no meaningful per-unit contention to arbitrate (adding one more ticket to a cart doesn't "hold" any specific unit away from another buyer in the same way a specific numbered seat does), so a much simpler inventory-decrement model (reduce an available count, no per-item hold/lock needed) suffices, and building out the full seat-level hold/lock/real-time-sync machinery for that case would be unwarranted complexity. This is worth naming explicitly as an example of matching the solution's complexity to the actual shape of the underlying problem, rather than always defaulting to the more elaborate mechanism regardless of whether genuine per-unit contention exists.

The trap: applying the full seat-hold architecture uniformly regardless of whether the underlying inventory is actually contended at the individual-unit level — recognizing when the simpler model suffices (and why) is itself a relevant piece of engineering judgment this follow-up is testing for.

---

## Self-Assessment

- [ ] Can explain why seat selection needs an immediate server-side hold, not just an optimistic client-side selection resolved at checkout
- [ ] Can design the optimistic-select-then-rollback-on-denial flow and describe exactly what both the winning and losing client experience in a race
- [ ] Can explain why a hold's countdown must be visibly communicated to the user, not just silently enforced
- [ ] Can design an atomic, all-or-nothing group hold and explain why a loop of independent per-seat holds is insufficient
- [ ] Can name the three distinct booking-failure modes and why they need distinct recovery messaging rather than one generic error
- [ ] Can explain the over-broadcast scaling risk at high on-sale concurrency and connect it to the same mitigation used in the Live Comments scenario

---
*Next: Design a Polling/Voting Widget — a smaller-scoped relative of this scenario's concurrency theme: many users casting votes concurrently against shared, live-updating aggregate counts, without the added complexity of a hold/lease/expiry lifecycle, since a vote (unlike a seat) isn't a resource that needs to be temporarily reserved before being finalized.*
