# Design a Chat Application (WhatsApp Web-style)

## Quick Reference

| Decision Point | Choice | Why |
|---|---|---|
| Transport | Persistent WebSocket (or equivalent) for live messages, plain HTTP for history/pagination | Live messages need low-latency, server-initiated push; history is a bounded, request/response fetch better served by ordinary paginated HTTP |
| Sending a message | Optimistic render with a client-generated temporary ID immediately, reconciled with the server's real ID/timestamp on ack | The sender must see their own message instantly (chat feels broken if your own messages lag); reconciliation avoids either a duplicate or a flicker when the server's authoritative copy arrives |
| Message identity/dedup | Every message carries a globally unique, client- or server-assigned ID used as the render key and dedup key | Reconnection, multi-tab, and multi-device delivery can all redeliver a message the client already has — dedup by ID, not by content or array position, is what prevents duplicates |
| Reconnection | Exponential backoff with jitter, plus a resync step (fetch anything missed) on reconnect, not just "open a new socket" | A dropped connection can silently miss messages sent while offline; simply reopening the socket resumes the *live* stream but doesn't backfill the gap on its own |
| History scroll direction | Older messages load when scrolling *up*, with scroll-anchor preservation so the viewport doesn't jump | Prepending content above the current scroll position shifts everything down unless the scroll offset is explicitly corrected for the newly-inserted height |

## The Scenario

"Design the frontend architecture for a one-on-one and group chat application — think WhatsApp Web. Users send and receive messages in real time, see delivery/read receipts, see typing indicators, and can scroll up to load older message history. It needs to keep working (queuing outgoing messages) if the network drops briefly, and reconcile correctly once it reconnects. Walk me through the whole system."

## Clarifying Questions

- **Is real-time delivery required (sub-second, server-pushed), or would short-interval polling be acceptable for this product's latency expectations?** This decides between a persistent connection (WebSocket, or a fallback like long-polling/SSE for one-directional needs) versus simple polling — a chat app's whole premise generally demands the former, but it's worth confirming rather than assuming, since it changes essentially every subsequent piece of the design (connection lifecycle, reconnection logic, backoff).
- **What delivery/read-receipt granularity is expected — sent, delivered-to-server, delivered-to-recipient-device, read?** Each additional state is a real signal that has to be produced somewhere (client or server) and displayed per-message, and "read" specifically implies the recipient's client needs to report back when a message actually enters their viewport, not just when it's received — a meaningfully different plumbing requirement than delivery alone.
- **Does a message need to survive app restart / work across multiple devices logged into the same account simultaneously?** This determines whether local persistence (IndexedDB, so history is available offline and on relaunch without waiting on a full re-fetch) is in scope, and whether the dedup/ordering logic needs to handle the *same* message arriving via two different device sessions, not just via a single reconnecting client.
- **What happens to a message typed and sent while the network is down — does it need to queue locally and send automatically once connectivity returns, or is it acceptable to show an explicit "failed to send, tap to retry" state?** A genuine offline-queue-and-auto-resend design is materially more work (needs a durable local outbox, retry logic, and careful ordering-on-resume) than an explicit-failure-with-manual-retry design — worth confirming which is actually expected rather than over- or under-building.
- **Are typing indicators and read receipts expected to be reliable/durable, or are they acceptable as best-effort, ephemeral signals that can be silently dropped under bad network conditions?** This shapes the transport choice for these specifically — typing indicators, in particular, are almost always treated as fire-and-forget/lossy by design (a missed "user is typing" event has near-zero consequence), which is a materially simpler contract than the guaranteed-delivery requirement actual messages need.
- **Roughly what's the expected group size (two-person chats only, or groups up to some N), and does message ordering need to be strictly consistent across all participants' views?** Larger groups and strict cross-client ordering guarantees push toward needing a server-assigned sequence/timestamp as the authoritative order (client-side timestamps alone are not trustworthy for ordering across different users' possibly-unsynced clocks).

## Approach & Trade-offs

**Optimistic send with a client-generated temporary ID, reconciled against the server's authoritative message once acknowledged — because a chat app's core feel depends on the sender's own message appearing instantly.** The moment a user hits send, the message renders immediately in their own view (status: "sending"), using a client-generated ID (a UUID) as its key — the message is simultaneously sent to the server over the live connection (or queued if offline). When the server acknowledges receipt (assigning its own authoritative ID, timestamp, and sequence position), the client reconciles: the optimistic message is updated in place (same rendered position, now carrying the server's real ID and "sent"/"delivered" status) rather than removed-and-re-added, which would cause a visible flicker or reordering. If the send ultimately fails, the same message flips to a "failed, tap to retry" state rather than silently disappearing — the user should never wonder "did that go through," only ever see an explicit confirmed/pending/failed state.

**Message identity for dedup must be a stable ID carried on the message itself, never array position or exact content-matching, because the same message can legitimately arrive more than once.** Reconnection after a dropped socket can redeliver a message the client already has (if the resync step isn't perfectly precise about what was already received); a multi-device setup can have the same message pushed to two active sessions; even a single WebSocket, under some server implementations, might redeliver on retry logic of its own. The client's incoming-message handler needs to check "do I already have a message with this ID" before appending to the conversation's message list — a `Set`/`Map` of known message IDs per conversation, checked on every incoming message (whether from the live socket or a history page fetch), is what makes this safe regardless of how or why a duplicate delivery occurs.

**Reconnection needs exponential backoff with jitter, and — critically — an explicit resync step, not just reopening the socket and resuming the live stream.** A naive reconnect (detect `onclose`, immediately try `new WebSocket(...)`) risks a thundering-herd reconnect storm if a server-side event disconnects many clients simultaneously (a deploy, a load balancer rotation) — backoff (starting short, e.g., 1s, doubling up to a cap, e.g., 30s) with jitter (randomizing each client's exact delay) spreads reconnection attempts out. But backoff alone only gets the *socket* back — any messages sent by others while this client was disconnected were missed entirely by the live stream, which only pushes forward from "now." The reconnect handler therefore needs to also fetch "anything I might have missed" (e.g., "give me all messages in my conversations since my last-known message ID/timestamp per conversation"), merging that gap-fill result into the existing message lists via the same ID-based dedup used everywhere else — reconnecting the pipe and refilling what flowed through it while disconnected are two distinct steps, and skipping the second silently loses messages from the user's perspective.

**Read receipts require the recipient's client to observe messages actually entering the viewport, which is an `IntersectionObserver` concern layered on top of the message list, not something the server can infer alone.** "Delivered" can be established server-side (the message reached the recipient's device/socket); "read" fundamentally requires the recipient's UI to know a given message was actually visible on screen — attaching an `IntersectionObserver` to each rendered message bubble (or to a virtualization library's own visibility tracking, if one is in use) and, once a message has been visible for some minimum dwell time (to avoid marking a message "read" from a half-second flash during fast scrolling), sending a lightweight "read up to message ID X" event back to the server, batched/debounced rather than fired per-message, to avoid a flood of read-receipt events during a long scroll through history.

**Typing indicators are treated as ephemeral, lossy, debounced signals — deliberately not held to the same delivery guarantees as messages.** A "user is typing" event is sent on keystroke activity (debounced, e.g., sent at most once every second or two while actively typing, and an explicit "stopped typing" sent on a pause or blur) over the same live connection, but with no retry, no persistence, no reconciliation on reconnect — if it's dropped, the worst outcome is a typing indicator that doesn't show up or lingers slightly too long (mitigated with a client-side timeout that auto-clears a typing indicator if no follow-up "still typing" or explicit "stopped" arrives within a few seconds), which is an acceptable, low-stakes failure mode that doesn't warrant the same engineering investment as guaranteed message delivery.

## Solution

**Sending with optimistic UI and reconciliation:**

```tsx
function useSendMessage(conversationId: string) {
  const { addOrUpdateMessage } = useConversationStore(conversationId);
  const { socket, isConnected } = useSocket();

  function send(text: string) {
    const tempId = crypto.randomUUID();
    const optimisticMessage: Message = {
      id: tempId,
      conversationId,
      text,
      senderId: currentUserId,
      status: 'sending',
      createdAt: new Date().toISOString(),
    };
    addOrUpdateMessage(optimisticMessage); // renders instantly

    const payload = { tempId, conversationId, text };
    if (isConnected) {
      socket.emit('message:send', payload, (ack: MessageAck) => {
        // Reconcile in place — same list position, new authoritative fields.
        addOrUpdateMessage({ ...optimisticMessage, id: ack.serverId, status: 'sent', createdAt: ack.serverTimestamp }, tempId);
      });
    } else {
      enqueueOutboxMessage(payload); // durable local outbox, flushed on reconnect
    }
  }

  return { send };
}
```

**Dedup on every incoming message, regardless of source (live socket or history fetch):**

```ts
function addOrUpdateMessage(message: Message, replaceId?: string) {
  setMessages((prev) => {
    const idToReplace = replaceId ?? message.id;
    const existingIndex = prev.findIndex((m) => m.id === idToReplace);
    if (existingIndex !== -1) {
      const next = [...prev];
      next[existingIndex] = message; // reconcile in place, no reorder/flicker
      return next;
    }
    if (prev.some((m) => m.id === message.id)) return prev; // already have it — drop the duplicate
    return insertSortedByServerOrder(prev, message);
  });
}
```

**Reconnection with backoff, jitter, and an explicit resync:**

```ts
function useChatSocket() {
  const attemptRef = useRef(0);

  function connect() {
    const socket = openSocket();

    socket.on('close', () => {
      const attempt = attemptRef.current++;
      const delay = Math.min(1000 * 2 ** attempt, 30_000) * (0.5 + Math.random() * 0.5); // backoff + jitter
      setTimeout(connect, delay);
    });

    socket.on('open', async () => {
      attemptRef.current = 0;
      // Refill whatever was missed while disconnected — reopening the pipe alone doesn't backfill it.
      const lastKnownIds = getLastKnownMessageIdsPerConversation();
      const missed = await fetchMessagesSince(lastKnownIds);
      missed.forEach((m) => addOrUpdateMessage(m)); // same dedup path as live messages
    });
  }

  useEffect(() => { connect(); }, []);
}
```

**Read receipts via `IntersectionObserver`, batched:**

```tsx
function useReadReceipts(conversationId: string) {
  const pendingRef = useRef<Set<string>>(new Set());
  const flush = useDebouncedCallback(() => {
    if (pendingRef.current.size === 0) return;
    reportReadReceipts(conversationId, Array.from(pendingRef.current));
    pendingRef.current.clear();
  }, 800);

  const observeMessage = useCallback((el: HTMLElement, messageId: string) => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        const timer = setTimeout(() => { pendingRef.current.add(messageId); flush(); }, 500); // dwell time
        observer.disconnect();
        return () => clearTimeout(timer);
      }
    }, { threshold: 0.6 });
    observer.observe(el);
    return () => observer.disconnect();
  }, [flush]);

  return observeMessage;
}
```

> **Check yourself:** Without looking above, explain why reopening a dropped WebSocket and resuming the live stream is not, by itself, sufficient to guarantee no messages were lost — and what the reconnect handler needs to do in addition to catching up correctly.

## Data Model

```ts
interface Message {
  id: string; // server-assigned once acked; a client tempId before that
  conversationId: string;
  senderId: string;
  text: string;
  createdAt: string; // server timestamp — never trust client clocks for ordering across users
  status: 'sending' | 'sent' | 'delivered' | 'read' | 'failed';
}

interface Conversation {
  id: string;
  participantIds: string[];
  lastMessageId: string | null;
  unreadCount: number;
}
```

Ordering within a conversation is anchored to server-assigned sequence/timestamp, not client-side `Date.now()` — two users' devices can have meaningfully skewed clocks, and only the server has one consistent clock all participants' messages are ordered against.

## Scaling Considerations

**Message history needs cursor-based pagination scrolling *upward*, with scroll-anchor preservation.** Loading older messages when the user scrolls near the top prepends content above the current viewport — naively prepending shifts everything downward by the newly-inserted height, visibly yanking the viewport; the fix is measuring the height about to be added, prepending, then immediately adjusting `scrollTop` by that same delta in the same paint (or using a virtualization library with built-in "prepend without visual jump" support) so the previously-visible message stays exactly where the user's eye was.

**The active conversation's message list should be virtualized once history grows large**, for the same reasons established in the News Feed scenario — long-running conversations (years of history) accumulate far more messages than should stay mounted in the DOM simultaneously.

**Multi-tab/multi-device coordination benefits from a shared local layer (e.g., a `BroadcastChannel` or a shared IndexedDB-backed store) rather than each tab independently maintaining its own WebSocket connection to the same account.** Multiple independent sockets for the same user multiplies server-side connection load for no benefit and complicates "mark as read" semantics (which tab's read receipt is authoritative); a common pattern elects one tab as the "connection owner" and broadcasts incoming events to sibling tabs, or accepts multiple sockets but centralizes dedup/read-state in a shared local store all tabs read from.

**Presence and typing indicators should be rate-limited at the source, not just debounced on receipt** — sending a typing event on literally every keystroke, even debounced client-side before display, still means every keystroke potentially reaches the network layer; throttling the *outbound* typing event itself (not just the *displayed* result) reduces load on both the client's own network usage and the server fan-out to every other participant in a group conversation.

## Gotchas

**Removing and re-adding the optimistic message on server ack instead of reconciling it in place.** Produces a visible flicker/reorder the instant the acknowledgment arrives — the message should be updated in place (same array position, new fields), keyed so React recognizes it as the same list item rather than a removal followed by an insertion.

**Deduping by message content or array position instead of a stable ID.** Two different messages can have identical text ("ok", "😂"); array position shifts constantly as new messages arrive — only a globally stable ID per message is a valid dedup key.

**Reopening the socket on reconnect without a resync step.** The live stream only carries messages sent *after* reconnection completes — anything sent by others during the disconnected window is gone unless explicitly fetched via a "what did I miss" request keyed off the last known message per conversation.

**No backoff (or no jitter) on reconnection attempts.** A bare "retry immediately forever" loop hammers the server the instant it's back up, and a same-delay-for-everyone backoff (no jitter) still produces a synchronized thundering herd across all disconnected clients reconnecting on the same schedule.

**Trusting client-side timestamps for cross-user message ordering.** Two devices' clocks can differ by seconds or more; ordering messages from different senders by their own local `Date.now()` at send time can produce a visibly wrong (or inconsistent-between-viewers) conversation order — the server's assigned timestamp/sequence is the only trustworthy ordering signal.

**Marking a message "read" the instant it's rendered, without any dwell-time or visibility check.** A message that flashes past during fast scrolling gets marked read despite never actually being seen — an `IntersectionObserver`-based visibility check with a minimum dwell time is what makes "read" mean what it claims to mean.

## Follow-up Questions

**Q (High): Walk through, step by step, everything that needs to happen between a dropped WebSocket connection and the chat being fully caught up again — not just "reconnect the socket."**

Answer: First, detect the drop (`onclose`/`onerror`) and schedule a reconnect attempt using exponential backoff with jitter rather than an immediate or fixed-interval retry, to avoid hammering the server and to avoid a synchronized reconnect storm across many simultaneously-dropped clients. Once a new connection is successfully established, the client needs an explicit resync step: for each active conversation, ask the server for anything sent since the last message the client actually has (by ID or timestamp) — this is a separate request/response exchange, not something the live stream itself provides, since the live stream only carries messages going forward from the moment it (re)opens. The resync response is merged into existing message lists through the exact same ID-based dedup path used for ordinary live messages, so anything that happens to already be present (edge cases around exactly when the drop occurred) is safely ignored rather than duplicated. Only after this resync completes should the UI drop any "reconnecting..." indicator and consider the conversation state trustworthy again.

The trap: describing reconnection as solved once `new WebSocket(url)` successfully opens — that only restores the *forward* stream; the backward-looking gap (what was missed while disconnected) is a distinct problem that an unaware design silently fails to solve, producing a chat history with invisible gaps that only surfaces as a user complaint ("I never saw that message until I refreshed").

---

**Q (High): Two devices logged into the same account are both online. User sends a message from their phone. What has to happen for it to appear correctly (and only once) on their laptop, and for the laptop's own message list to still show correct ordering relative to messages the laptop itself might be sending around the same time?**

Answer: The server, as the single point of truth for a conversation's ordering, needs to broadcast the new message to every active connection for every participant, including other sessions/devices of the *sender's own* account, not just the other participant(s) — otherwise the laptop's copy of the conversation silently diverges from what was actually sent. On arrival at the laptop, the message goes through the same ID-based dedup/insert path as any other incoming message (which also correctly handles the case where the laptop, coincidentally, sent its *own* message around the same moment — both are inserted according to the server's assigned order, not client-arrival order). If the laptop happens to have an optimistic, not-yet-acked message of its own in flight at the same time, that's a separate, already-tracked local ID (its own tempId) and doesn't collide with the phone's message, which arrives with a distinct server ID.

The trap: assuming "sync across devices" is solved by each device polling for changes independently, or by the sending device somehow notifying its own other sessions directly (peer-to-peer) — the server is the only party positioned to guarantee one consistent order across all participants and all of one user's own devices, and every client (regardless of which device sent what) should be treated as just another subscriber to that one authoritative stream.

---

**Q (High): The interviewer says: "Assume messages can arrive out of order over the WebSocket, even without a reconnect." How does the design need to change?**

Answer: The insertion logic needs to order incoming messages by the server-assigned sequence/timestamp rather than assuming arrival order equals conversation order — instead of a naive `messages.push(newMessage)`, an incoming message is inserted at the position determined by its authoritative order relative to existing messages (a sorted-insert, or, more efficiently at scale, batching a short window of arrivals and sorting before a single re-render). This matters even without a reconnect because network-level reordering (different packets taking different routes, some transport layers not guaranteeing in-order delivery at the message-framing level above raw TCP) is a real possibility worth designing for explicitly rather than assuming "arrived over the socket" implies "arrived in conversation order."

The trap: assuming a single WebSocket connection guarantees in-order delivery of application-level messages just because the underlying TCP connection guarantees in-order *byte* delivery — TCP ordering is about bytes on one connection, not about the application's own message semantics if, for instance, retries, multiplexing, or server-side batching/fan-out introduce their own reordering above the transport layer.

---

**Q (Medium): How would you implement a durable local outbox for messages composed while offline, such that they reliably send once connectivity returns, survive a page refresh while still offline, and send in the order they were composed?**

Answer: Outgoing messages composed while disconnected are written to a durable local store (IndexedDB, not just in-memory state, specifically so they survive a page refresh that happens before connectivity returns) as an ordered queue, each entry carrying its own client-generated tempId and the timestamp it was composed at. On reconnect, the outbox is flushed in composition order — each entry sent and awaited for acknowledgment before the next is sent (or sent as a batch if the backend supports an ordered-batch send), with each successful ack removing that entry from the durable queue and reconciling it into the visible message list exactly like the live optimistic-send path. If a page refresh happens mid-offline-composition, on load the outbox is read back from IndexedDB and re-rendered as pending/"sending" messages before any reconnect attempt even begins, so the user sees their queued messages are still there, not silently lost.

The trap: keeping the offline outbox purely in memory (a `useState`/store array) — this appears to work in a quick demo (compose offline, reconnect, it sends) but loses everything the instant the tab is closed or refreshed while still offline, which is exactly the scenario "handle the network dropping" is supposed to cover robustly.

---

**Q (Medium): How would you avoid sending a flood of "read receipt" events while a user rapidly scrolls through a long stretch of message history?**

Answer: Batch and debounce the read-receipt reporting rather than firing one network event per message as it becomes visible — accumulate message IDs that have met the visibility+dwell-time bar into a local pending set, and flush that set as a single "mark read up to message ID X" (or a batched list) request on a short debounce, so a fast scroll through fifty messages produces one or two network calls once scrolling settles, not fifty. The dwell-time requirement itself (a message must stay visible for some minimum duration, not just technically intersect the viewport for a frame) is what prevents messages that merely flash past during a fast scroll from being marked read at all, which is a separate but related piece of the same fix.

The trap: solving only the "batch the network calls" half and skipping the dwell-time requirement — without dwell time, a fast scroll still marks every message technically "seen" the instant it briefly intersects the viewport, which produces technically-batched-but-still-incorrect read receipts for messages the user never actually read.

---

**Q (Low): Should typing indicators and read receipts use the same WebSocket connection as messages, or a separate channel?**

Answer: Generally the same connection is fine and simpler operationally (one connection lifecycle to manage, one reconnect/backoff strategy to reason about) — the distinction that actually matters is in how each message *type* is handled once received/sent, not which physical connection carries it: messages need guaranteed delivery, dedup, and ordering; typing indicators are explicitly lossy/ephemeral and need none of that. Multiplexing both over one socket (tagging each payload with a type) is the common, pragmatic choice; a separate connection per concern would mostly add operational complexity (two connections to open, monitor, and reconnect) without a clear corresponding benefit unless there's a specific scaling reason (e.g., wanting to shed low-priority typing-indicator traffic independently under severe load) to justify the split.

The trap: assuming "different guarantees needed" implies "different transport needed" — the guarantee level is a property of how each message type is *handled* by the application logic on both ends, not an inherent requirement of the underlying connection it travels over.

---

## Self-Assessment

- [ ] Can design the optimistic-send-then-reconcile flow and explain why in-place reconciliation (not remove-then-readd) matters
- [ ] Can explain why message dedup must key off a stable ID, never content or array position
- [ ] Can describe what a reconnect handler needs beyond reopening the socket — specifically the resync/gap-fill step — and why skipping it silently loses messages
- [ ] Can design a durable offline outbox that survives a page refresh while disconnected
- [ ] Can implement read receipts via visibility + dwell time, batched to avoid a flood of network calls
- [ ] Can explain why server-assigned ordering (not client timestamps) is required for cross-device/cross-user consistency

---
*Next: Design a Notification System (In-app + Push) — shifts from a single active conversation's live stream to broadcasting events across many independent surfaces (in-app badge, toast, push notification, browser tab title), where the new central question is de-duplicating and routing one underlying event across several simultaneous delivery channels.*
