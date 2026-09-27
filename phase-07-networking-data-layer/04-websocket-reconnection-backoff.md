# WebSocket Reconnection & Backoff

## Quick Reference

| Concern | Mechanism | Why It Matters |
|---|---|---|
| Detecting disconnection | `onclose`/`onerror` handlers, plus app-level heartbeat/ping-pong for silent drops | TCP/proxy-level drops don't always fire a clean close event promptly |
| Reconnect timing | Exponential backoff with jitter, capped at a max interval | Prevents thundering-herd reconnect storms against a recovering server |
| Missed-message recovery | Resume from a last-seen sequence number/cursor, or full resync, on reconnect | A reconnected socket is a *new* connection — anything sent while disconnected is gone unless explicitly recovered |
| Duplicate/out-of-order handling | Sequence numbers or idempotency keys on messages | Resync and normal delivery can race, delivering the same event twice or out of order |
| User-visible state | Explicit connection-status state machine (connected/reconnecting/offline), not silent retries | Users need to know if what they're looking at might be stale |

## The Scenario

"We have a live dashboard that gets updates over a WebSocket — stock prices, whatever. When someone's wifi drops for even a few seconds, the socket dies, and the dashboard just... stops updating. No error, no indication anything's wrong, it just silently goes stale. When the connection comes back, sometimes it reconnects, sometimes it doesn't, and we've had reports of data that looks 'jumpy' — like it skipped a few updates then caught up all at once. Design the reconnection logic properly, from detecting the drop through recovering missed data."

## Clarifying Questions

- **When the connection drops, does the browser/OS fire a clean `close` event promptly, or can the socket sit in a zombie state where `readyState` still reads `OPEN` but no data is actually flowing?** This determines whether `onclose` alone is a sufficient detection mechanism or whether an application-level heartbeat (periodic ping, expecting a pong within some timeout) is also needed — a mid-network failure (wifi drops, a NAT/proxy silently kills an idle connection) often doesn't produce a clean TCP-level close on the client side for a while, if ever, leaving the browser socket API unaware anything is wrong.
- **What does "sometimes it reconnects, sometimes it doesn't" mean concretely — are reconnect attempts happening but failing, or does the client's reconnect logic itself sometimes fail to trigger at all?** This is the first thing to isolate: a genuinely inconsistent reconnect *attempt* rate points at a bug in the reconnect-triggering logic itself (e.g., only wired to `onclose` but the actual failures are silent/zombie disconnects that never fire it), whereas consistent attempts that sometimes succeed and sometimes don't is closer to a legitimate "server/network still recovering" case that backoff and retry limits are meant to handle gracefully.
- **Is missed data during the disconnect window acceptable to lose (as long as the client resyncs to current state on reconnect), or does every update need to be delivered, even ones that happened while offline?** For a live stock price dashboard, typically only the *latest* value matters — a full resync to current state on reconnect is sufficient, and individually-missed intermediate updates are fine to drop. For something like a chat app or an audit trail, every message matters and a gap is a correctness bug, requiring the server to support resuming from a last-seen point rather than just "give me current state."
- **Is there a REST/HTTP fallback for the same data**, so the dashboard could keep showing reasonably fresh data via polling while the WebSocket is reconnecting, or is the WebSocket the only path to this data? Determines whether "silently goes stale with no indication" should be fixed purely with UI messaging ("reconnecting...") or whether a temporary fallback data path is worth building so the dashboard degrades to polling rather than fully freezing during a reconnect window.
- **Roughly how many clients connect to this WebSocket endpoint, and does the server have its own capacity/rate-limiting concerns on reconnection storms** (e.g., a deploy or brief outage that drops every connected client simultaneously)? Shapes how aggressive backoff and jitter need to be — a small internal tool's reconnect logic can be fairly naive; a consumer-facing dashboard with thousands of concurrently-connected clients reconnecting in lockstep after a server restart can itself cause a second outage if backoff isn't designed with that thundering-herd case in mind.

## Approach & Trade-offs

**Split the problem into four genuinely separate concerns, because conflating them is how "reconnection logic" scenarios end up half-solved: (1) detecting the drop, (2) deciding when to retry, (3) recovering what was missed, (4) representing connection state to the user.** A common shallow answer only addresses (2) — wraps the WebSocket constructor in a retry loop — while leaving (1) unreliable (zombie connections never trigger it), (3) unaddressed (reconnecting gets you a *new* socket with no memory of what you missed), and (4) invisible to the user (the dashboard silently goes stale with no signal, which is explicitly called out as the actual complaint in the prompt).

**Detection needs to handle both clean and silent failures — `onclose` catches the former, a heartbeat catches the latter.** `WebSocket.onclose` fires reliably when the server explicitly closes the connection or the browser detects a clear-cut network failure, but network paths through NATs, corporate proxies, or degraded wifi can leave a socket in a state where the browser believes it's still open while no data is actually flowing in either direction — no `close` event fires until some underlying OS-level timeout eventually kicks in, which can be minutes. An application-level heartbeat (client sends a ping on an interval, expects a pong within some shorter timeout; or the server pushes a periodic heartbeat message the client expects to see regularly) makes silent failures detectable on the app's own timeline instead of waiting on OS/network-layer timeouts that are out of the app's control.

**Reconnection timing needs exponential backoff with jitter, not a fixed retry interval — and the reason is as much about protecting the server as the client.** A fixed short retry interval (e.g., "always retry every 1 second") is fine for one client but catastrophic at scale: if a server restart or brief network partition drops thousands of concurrently-connected clients simultaneously, all of them retrying on the same fixed interval creates a synchronized reconnect storm that can prevent the server from ever stabilizing (each wave of reconnects adds load right as it's trying to recover, potentially causing it to fail again, causing another synchronized wave). Exponential backoff (each failed attempt waits longer than the last, up to a cap) spreads reconnect attempts out over time; adding jitter (randomizing each client's exact wait within a range) additionally desynchronizes what would otherwise still be near-simultaneous waves even under pure exponential backoff, since all clients failed at roughly the same moment and would otherwise retry at roughly the same computed intervals.

**Recovering missed data is a protocol design question, not just a client-side concern — the server has to support whatever recovery strategy is chosen.** A reconnected WebSocket is a *brand new connection* with no memory of the old one; the server needs to either (a) always push full current state on connect, making individual missed updates irrelevant (fine when only latest-value matters, as in the stock-price case), or (b) support the client saying "I last saw sequence number N, send me everything since," which requires the server to buffer/persist recent messages long enough to replay them and requires every message to carry a sequence number or timestamp the client can track and report back. Choosing between these is the same latest-value-vs-every-event trade-off surfaced in the clarifying questions, and it constrains the server's own design, not just the client's reconnect loop.

**"Jumpy" data — several updates arriving in a burst after reconnect — is very likely the *correct* behavior of a good resync (server sends everything missed, all at once) colliding with UI code that assumed one update at a time.** Rather than treating the burst itself as the bug, I'd check whether the UI renders each queued/replayed update as a separate transition (causing visible jank) versus collapsing to the final state, which is usually what's actually wanted for a live-value display — this reframes "jumpy" from a networking bug into a rendering-of-a-burst UI decision.

## Solution

**Step 1 — connection state as an explicit, typed state machine, not an implicit boolean.** This directly addresses "no error, no indication anything's wrong":

```ts
type ConnectionState =
  | { status: 'connecting' }
  | { status: 'connected' }
  | { status: 'reconnecting'; attempt: number; nextRetryAt: number }
  | { status: 'offline' }; // exhausted retries or explicitly offline
```

The dashboard reads this state to render a visible "Reconnecting… (attempt 3)" banner instead of silently freezing — closing the actual complaint in the prompt before anything about the reconnect mechanics itself.

**Step 2 — the reconnect manager: exponential backoff with jitter, driven by both `onclose` and a heartbeat timeout.**

```ts
class ReconnectingSocket {
  private ws: WebSocket | null = null;
  private attempt = 0;
  private heartbeatTimer: ReturnType<typeof setTimeout> | null = null;
  private lastSeenSeq: number | null = null;
  private state: ConnectionState = { status: 'connecting' };

  private readonly baseDelayMs = 500;
  private readonly maxDelayMs = 30_000;
  private readonly heartbeatIntervalMs = 15_000;
  private readonly heartbeatTimeoutMs = 5_000;

  connect() {
    this.ws = new WebSocket(this.url);
    this.ws.onopen = () => {
      this.attempt = 0; // reset backoff on a genuinely successful connection
      this.setState({ status: 'connected' });
      this.resync();
      this.startHeartbeat();
    };
    this.ws.onmessage = (evt) => this.handleMessage(evt);
    this.ws.onclose = () => this.handleDisconnect();
    this.ws.onerror = () => this.ws?.close(); // funnel errors through one path
  }

  private handleDisconnect() {
    this.stopHeartbeat();
    this.attempt += 1;
    const delay = Math.min(this.baseDelayMs * 2 ** this.attempt, this.maxDelayMs);
    const jitter = delay * (0.5 + Math.random() * 0.5); // 50%–100% of computed delay
    this.setState({ status: 'reconnecting', attempt: this.attempt, nextRetryAt: Date.now() + jitter });
    setTimeout(() => this.connect(), jitter);
  }

  private startHeartbeat() {
    this.heartbeatTimer = setInterval(() => {
      const timeout = setTimeout(() => this.ws?.close(), this.heartbeatTimeoutMs);
      this.ws?.send(JSON.stringify({ type: 'ping' }));
      this.pendingPongTimeout = timeout; // cleared in handleMessage on 'pong'
    }, this.heartbeatIntervalMs);
  }
  // ...stopHeartbeat, handleMessage (clears pong timeout on 'pong', otherwise
  // dispatches app messages and tracks lastSeenSeq), setState (notifies subscribers)
}
```

The heartbeat closes the zombie-connection gap: if a `pong` doesn't arrive within `heartbeatTimeoutMs` of a `ping`, the client force-closes the socket itself, funneling into the same `onclose` → backoff path rather than waiting on the OS to eventually notice.

**Step 3 — resync on every successful (re)connection, using the last-seen sequence number if the protocol supports incremental replay:**

```ts
private resync() {
  this.ws?.send(JSON.stringify({
    type: 'resync',
    // omit lastSeenSeq entirely on first-ever connect → server sends full state
    lastSeenSeq: this.lastSeenSeq ?? undefined,
  }));
}
```

Server-side (conceptually): if `lastSeenSeq` is present and still within the server's replay buffer window, send everything since that sequence number; if it's absent or too old (buffer expired — client was offline too long), send a full current-state snapshot instead, and the client should treat a snapshot message as authoritative, replacing rather than merging its local state.

**Step 4 — handle the resync/live-message race explicitly.** Between requesting a resync and its response arriving, new live messages can still arrive on the same socket — the client needs to buffer and de-duplicate by sequence number rather than assuming resync data and live data arrive in a clean, non-overlapping order:

```ts
private handleMessage(evt: MessageEvent) {
  const msg = JSON.parse(evt.data);
  if (msg.type === 'pong') { clearTimeout(this.pendingPongTimeout); return; }
  if (msg.type === 'snapshot') { this.applyFullState(msg.state); this.lastSeenSeq = msg.seq; return; }
  if (msg.seq !== undefined) {
    if (this.lastSeenSeq !== null && msg.seq <= this.lastSeenSeq) return; // duplicate, drop
    this.lastSeenSeq = msg.seq;
  }
  this.applyUpdate(msg); // dispatched to app state, batched for rendering — see below
}
```

**Step 5 — batch a burst of resync-delivered updates into a single render, addressing the "jumpy" complaint directly.** Rather than dispatching each replayed update as its own state transition (causing visible flicker through several intermediate values), collect updates arriving within a short window and apply the *final* state in one update for anything the UI only needs the latest value of:

```ts
private pendingUpdates: Update[] = [];
private flushTimer: ReturnType<typeof setTimeout> | null = null;

private applyUpdate(update: Update) {
  this.pendingUpdates.push(update);
  if (!this.flushTimer) {
    this.flushTimer = setTimeout(() => {
      const latestPerKey = collapseToLatestPerKey(this.pendingUpdates); // e.g. latest price per symbol
      this.notifySubscribers(latestPerKey);
      this.pendingUpdates = [];
      this.flushTimer = null;
    }, 50); // small window — imperceptible delay, absorbs resync bursts
  }
}
```

> **Check yourself:** If the server's replay buffer only holds the last 5 minutes of messages and a client was disconnected for 8 minutes, what does `resync` need to do differently, and how does the client know which case it's in from the server's response?

## Gotchas

**Wiring reconnect logic only to `onclose`/`onerror` and never adding a heartbeat**, leaving zombie/silent disconnects (the actual described symptom — "wifi drops... just stops updating, no error") completely undetected until some unrelated event eventually forces a close, if ever. This is the single most common incomplete answer to this scenario, since `onclose`-triggered reconnection alone looks complete in a demo where disconnects are always clean (closing a tab, killing a local dev server) but not against the real-world silent-failure case the prompt is actually describing.

**Resetting the backoff counter on every *attempt* rather than only on a genuinely successful, stable connection** — if `attempt` resets the moment a new `WebSocket` is constructed (rather than on `onopen`, or better, after staying connected for some minimum duration), a server that accepts the TCP/WebSocket handshake but then immediately drops the connection again (a common failure mode during a rolling deploy or an overloaded server) causes the client to reconnect at the fastest possible rate indefinitely, defeating the entire purpose of backoff.

**No jitter, only exponential backoff** — all clients that disconnected at the same moment (a server restart affecting everyone connected) compute the same backoff delays and retry in near-synchronized waves indefinitely, since pure exponential backoff without randomization doesn't desynchronize clients that failed together, just spaces out *when* they retry as a group.

**No maximum retry limit or escalation to a genuinely "offline" state** — infinite reconnection attempts, even with a capped backoff interval, means a client whose network is permanently gone (not coming back — a legitimately offline user) keeps silently hammering retries forever rather than surfacing a clear "you're offline" state and, ideally, backing off retry frequency further or stopping until some external signal (the browser's `online` event, a manual retry action) suggests trying again is worthwhile.

**Treating "reconnected" as "resynced," without an explicit resync step and without protecting against the resync/live-message race** — connecting a fresh socket successfully doesn't by itself recover anything that was missed; skipping the explicit resync round trip (or getting the sequence-number de-duplication wrong, so replayed and live messages double-apply) is exactly the class of bug that produces the "jumpy... skipped a few updates then caught up all at once, sometimes with visible duplication" symptom.

## Follow-up Questions

**Q (High): Why doesn't `WebSocket.onclose` alone reliably detect a dropped connection, and what specifically does a heartbeat add that `onclose` can't provide?**

Answer: `onclose` fires when the browser's networking stack determines the connection has ended — either an explicit close frame from the server, or the underlying OS/TCP layer concluding the connection is dead, which for many real-world failure modes (wifi dropping, a NAT or proxy silently discarding an idle connection's state, a server crashing without sending a close frame) can take anywhere from tens of seconds to several minutes, or in some cases may not fire promptly at all — the browser's socket object can report `readyState === WebSocket.OPEN` while no data is actually flowing in either direction, because from the browser's perspective, nothing has explicitly told it the connection ended. An application-level heartbeat closes this gap by putting detection under the app's own control and timeline: the client sends a ping on a short interval (e.g., every 15 seconds) and starts a short timeout (e.g., 5 seconds) waiting for a pong; if no pong arrives in that window, the app concludes the connection is dead *right now*, on its own schedule, and force-closes the socket itself — triggering the same reconnect path `onclose` would have, but on a timeline the app controls rather than one dependent on OS/network-layer timeouts that can be arbitrarily long or, in some failure modes, may never trigger.

The trap: describing `onclose` as "handling disconnection" without qualifying that it only reliably covers *clean* disconnection — the scenario's stated symptom (silent staleness, no error) is specifically describing the case `onclose` alone doesn't catch, so an answer that stops at "listen for `onclose` and reconnect" hasn't actually addressed the reported bug.

---

**Q (High): Explain exactly why jitter is necessary in addition to exponential backoff — what failure does pure exponential backoff (no randomization) still allow?**

Answer: Exponential backoff alone controls *how much* each individual client's retry interval grows after repeated failures, but it does nothing to desynchronize clients that all started failing at the same moment — if a server restarts and drops 10,000 concurrently-connected clients simultaneously, every one of them computes the identical sequence of backoff delays (attempt 1 → 1s, attempt 2 → 2s, attempt 3 → 4s, ...) and, having failed at essentially the same instant, retries in near-perfect synchronized waves: all 10,000 hit the server again at ~1s, then (for whichever subset failed again) all retry together at ~2s, and so on. Each wave is a load spike precisely timed to arrive right as the server may still be recovering from the *previous* wave, which can itself cause the next round of failures, perpetuating the storm rather than letting the server stabilize. Jitter — randomizing each client's actual wait within a range around the computed exponential delay (e.g., 50%–100% of the nominal backoff value, chosen independently per client) — spreads what would otherwise be a synchronized wave into a smoother, desynchronized distribution of retry attempts over time, meaningfully reducing peak concurrent load on the recovering server even though the *average* backoff behavior per client is similar.

The trap: treating "exponential backoff" and "avoiding a thundering herd" as the same thing — exponential backoff alone solves the *individual* client's retry-rate problem; the *collective*, many-clients-failing-together problem specifically requires randomization to desynchronize, which is a distinct mechanism with a distinct purpose.

---

**Q (High): The dashboard reconnects successfully and resyncs, but the user reports seeing a value flicker backward — an old value briefly shown after a newer one — before settling on the correct current value. What's the likely cause, and how does sequence-number tracking prevent it?**

Answer: This is very likely the resync/live-message race described in Step 4 — while a resync request is in flight (client asked the server "send me what I missed since sequence N"), the same socket can still receive newly-arriving live messages ahead of the resync response, particularly if the resync response involves the server doing some work (querying a buffer, assembling a snapshot) that takes measurably longer than a live message's normal delivery latency. If the client applies messages in the raw order they arrive on the wire, without checking each one's sequence number against what it's already applied, a live message that arrived first (representing genuinely newer data) can get overwritten moments later by a resync-response message representing older, already-superseded data arriving after it — a visible backward flicker. Tracking `lastSeenSeq` and comparing every incoming message's sequence number against it before applying (as in Step 4's `handleMessage`) prevents this unconditionally: any message whose sequence number is less than or equal to what's already been applied is a stale/duplicate and gets dropped, regardless of the order it happened to arrive on the wire, so the client's displayed state is always monotonically advancing by sequence number even if the underlying message *delivery* order isn't strictly sequential.

The trap: assuming that because resync and live delivery share the same socket, they're implicitly ordered correctly — a single socket does guarantee in-order *delivery* of frames as sent, but doesn't guarantee that the server *generates and sends* a resync response before any subsequently-live-generated message, especially if resync involves any asynchronous work server-side; the client needs its own ordering guarantee (sequence numbers) rather than relying on wire order.

---

**Q (Medium): How would you decide when to give up retrying and show the user a genuinely "offline" state rather than continuing to reconnect indefinitely?**

Answer: A reasonable design caps the backoff delay at some maximum interval (so retries don't become arbitrarily infrequent) but doesn't necessarily need to cap the *number* of attempts outright, since a user's network genuinely could come back at any point and silently giving up would be worse than an infrequent background retry; instead, I'd distinguish the *displayed* state from the retry loop's internal behavior — after some number of consecutive failed attempts (or some elapsed time in the `reconnecting` state), transition the user-visible state to `offline` (a clear, honest "you're offline" indicator rather than an indefinitely-spinning "reconnecting…") while the retry loop can keep trying in the background at its capped max interval, so reconnection still happens automatically and promptly once the network genuinely returns, without misleadingly implying to the user that reconnection is imminent the entire time it's actually been failing for minutes. Additionally, listening for the browser's `online`/`offline` events (`window.addEventListener('online', ...)`) lets the client react immediately when the OS reports connectivity restored, rather than waiting for the next scheduled backoff attempt, which can otherwise add up to the full max backoff interval of extra, avoidable delay.

The trap: conflating "stop trying" with "tell the user we're offline" as if they have to happen at the same time — the better design decouples them: keep a background retry loop alive (bounded by the backoff cap, not abandoned) while being honest with the user sooner about the current state, rather than either giving up silently or displaying misleading "reconnecting" optimism indefinitely.

---

**Q (Medium): Would Server-Sent Events (SSE) be a simpler alternative to WebSockets for this dashboard, given it's one-directional (server → client updates only)? What reconnection behavior would change?**

Answer: For a purely one-directional, server-to-client update stream (which this stock-dashboard scenario is, based on the prompt — no mention of the client needing to send anything beyond an implied subscribe), SSE is a reasonable, often simpler alternative: it runs over plain HTTP (simpler to proxy/load-balance through infrastructure that may need special configuration for WebSocket upgrades), and critically, the browser's built-in `EventSource` API has automatic reconnection with backoff already implemented natively — no hand-rolled reconnect-manager class required for the basic case. It also supports the `Last-Event-ID` header natively as part of the reconnection protocol: on reconnect, the browser automatically includes the ID of the last event it received, and a correctly-implemented server can use that to resume the stream from that point — a browser-native analog to the manual `lastSeenSeq`-based resync built by hand in this scenario's WebSocket solution. The trade-off is SSE's one-directional nature (no client-to-server messages over the same connection — a heartbeat *ping* from client to server isn't directly possible the way it is over a WebSocket, though a client can still detect staleness via a client-side "haven't received an expected periodic message in N seconds" timeout without a true ping/pong round trip) and generally fewer framing/binary-data options than WebSockets provide. If the dashboard ever needs to send data back over the same channel (user-initiated actions, not just receiving updates), SSE stops being sufficient and WebSockets (or SSE plus a separate REST call for the write path) become necessary.

The trap: assuming WebSockets are simply "the more powerful, always-correct choice" without considering that a chunk of this scenario's hand-rolled reconnection/resume complexity (backoff, `Last-Event-ID`-style resume) is native, built-in behavior with SSE for the common one-directional case — proposing WebSockets by default without checking directionality requirements misses a simpler option that solves much of the stated problem for free.

---

**Q (Low): If this dashboard needed to work correctly across multiple browser tabs open to the same page, would you want each tab maintaining its own independent WebSocket connection and reconnect logic, or something shared?**

Answer: Independent per-tab connections work correctly but are wasteful and don't scale well if a user commonly has many tabs open to the same dashboard — each tab independently connects, reconnects, and heartbeats, multiplying server-side connection count and duplicate bandwidth for identical data with no benefit, since every tab is displaying the same live data to the same user. A shared-connection design — using a `SharedWorker` (a single WebSocket connection owned by one worker instance, shared across all same-origin tabs, which relay messages to/from it via `postMessage`) or electing one tab as a "leader" via the Web Locks API or a `BroadcastChannel`-coordinated election, with other tabs subscribing to that leader's connection and data via `BroadcastChannel` — reduces this to one actual WebSocket connection (and one reconnect/backoff/heartbeat loop) per browser instance regardless of open-tab count, at the cost of meaningfully more implementation complexity (worker lifecycle, leader election and hand-off when the leader tab closes, message relay plumbing) than each tab independently owning its own connection. I'd only reach for this if tab-multiplication was a measured, real problem (server-side connection count/cost, or actual user reports of degraded behavior with many tabs open) rather than building it preemptively, given the added complexity.

The trap: assuming shared-connection architecture is obviously "more correct" regardless of scale — for most products, independent per-tab connections are simpler, more robust to reason about (no leader-election edge cases), and genuinely fine; the shared approach is a scale-driven optimization, not a default best practice.

---

## Self-Assessment

- [ ] Can explain precisely why `onclose` alone misses zombie/silent disconnects and what a heartbeat adds
- [ ] Can derive why jitter is necessary in addition to exponential backoff (desynchronizing a thundering herd, not just controlling one client's retry rate)
- [ ] Can design a resume/resync protocol using sequence numbers and explain why a reconnected socket has no memory of missed data on its own
- [ ] Can diagnose and fix a resync/live-message race that causes stale data to overwrite fresher data
- [ ] Can represent connection state as an explicit, user-visible state machine rather than a silent retry loop
- [ ] Can compare WebSockets and SSE for a one-directional update stream and name what SSE gives natively (backoff, `Last-Event-ID` resume)

---
*Next: Offline-first Sync With Conflict Resolution — from keeping a live connection alive and resuming cleanly, to the harder problem of letting a client keep working productively while fully disconnected, then reconciling changes made on both sides when it reconnects.*
