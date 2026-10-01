# Choosing SSE vs. WebSocket for a Real-time Feature

## Quick Reference

| Need | Pick | Why |
|---|---|---|
| Server → client only (LLM tokens, notifications, live scores, progress) | SSE | Plain HTTP, auto-reconnect + `Last-Event-ID`, works through proxies/HTTP2, simpler infra |
| Bidirectional, low-latency, high-frequency (collab editing, games, presence, chat typing) | WebSocket | Full duplex over one persistent connection |
| Client → server occasional + server → client stream | POST for requests + SSE for stream | Keep each direction simple and cacheable/observable |
| Corporate proxies / HTTP-only constraints | SSE or long-polling fallback | WebSockets sometimes blocked/buffered |

## The Scenario

"We're building three features: (1) an AI assistant that streams answers, (2) live order-status notifications, and (3) a collaborative whiteboard with cursors. A teammate says 'just use WebSockets for all of it.' Do you agree?"

## Clarifying Questions

- **For each feature, which direction does data flow and how often?** One-way server push vs. frequent two-way messaging is the main discriminator.
- **What latency and message-rate requirements?** Cursor positions at 30–60 Hz demand a different transport posture than a notification every few minutes.
- **What's the infrastructure — load balancers, CDNs, corporate proxies, serverless?** Long-lived connections have real infra implications (connection limits, idle timeouts, sticky sessions).
- **How many concurrent connections at peak?** Persistent connections cost memory/file descriptors per user.
- **Need authentication beyond cookies — custom headers?** Affects `EventSource` viability and WebSocket auth handling.
- **Reliability needs — must we guarantee delivery/ordering, resume after disconnect?** Determines how much protocol you must build on top.
- **Binary data needed?** WebSocket supports binary natively; SSE is text only.

## Approach & Trade-offs

**Don't pick one hammer.** The right answer is per-feature, driven by data-flow direction and operational cost. WebSockets are more powerful, but power costs complexity — you pay it only where you need it.

**SSE (Server-Sent Events).**
- *Pros:* one-way server→client over normal HTTP; works with existing auth cookies/middleware/observability; HTTP/2 multiplexing avoids the 6-connection-per-host limit of HTTP/1.1; built-in framing, reconnection, and `Last-Event-ID` resume; trivially load-balanced like any HTTP request; text-based and easy to debug in DevTools/curl.
- *Cons:* unidirectional; text only (binary needs base64); native `EventSource` lacks custom headers and POST (use fetch-based parsing); on HTTP/1.1 limited concurrent connections per origin; some proxies buffer streaming responses.

**WebSocket.**
- *Pros:* full duplex, low overhead per message after the handshake, binary support, great for high-frequency bidirectional traffic.
- *Cons:* you own more of the protocol — heartbeats/ping-pong, reconnection with backoff, resume/replay, message framing/versioning, backpressure; stateful connections complicate scaling (sticky sessions or pub/sub fan-out like Redis); not covered by normal HTTP middleware/caching; some proxies/firewalls interfere; auth handled at handshake only (can't set custom headers from browser `WebSocket`, so tokens go in the query string or subprotocol or a first message — each has trade-offs); harder to observe and test.

**Mapping the three features.**
1. *AI streaming answer:* the client sends one request and receives a long response → **POST + streamed response (SSE format)**. Simple, cancelable by aborting the request, no persistent connection to manage.
2. *Order-status notifications:* server→client, infrequent, must survive reconnects → **SSE** (resume with `Last-Event-ID`), or push notifications if the app may be closed.
3. *Collaborative whiteboard with cursors:* high-frequency, bidirectional, low latency → **WebSocket** (or WebRTC data channels for P2P).

**Alternatives worth naming.** Long-polling as a universal fallback; WebTransport (HTTP/3, streams + datagrams) as an emerging option but limited support; managed services (Pusher, Ably, Supabase Realtime) when you don't want to run connection infrastructure.

**The scaling reality.** Both involve long-lived connections. The difference is that WebSocket state often pins users to a server and needs a pub/sub backplane; SSE is also long-lived, but each stream is a plain HTTP response that proxies/CDNs/HTTP2 handle with existing tooling. Consider connection idle timeouts (heartbeat comments every 15–30s on SSE; ping frames on WS).

**Trade-off summary:** choose the simplest transport that meets the data-flow requirement. Each unnecessary WebSocket is operational surface area you maintain forever.

## Solution

### Decision flow

```
Is data only server → client?
  ├─ yes → SSE (or POST + streaming response if it's request-scoped)
  └─ no  → Is client → server frequent/low-latency?
            ├─ no  → HTTP POST for client→server + SSE for server→client
            └─ yes → WebSocket (add heartbeat, reconnect, resume)
```

### SSE with resume (server-side sketch)

```ts
app.get('/events', auth, (req, res) => {
  res.set({ 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache',
            Connection: 'keep-alive', 'X-Accel-Buffering': 'no' });
  const lastId = Number(req.get('Last-Event-ID') ?? 0);
  replaySince(req.user.id, lastId).forEach(e => res.write(`id: ${e.id}\ndata: ${JSON.stringify(e)}\n\n`));
  const unsub = bus.subscribe(req.user.id, e => res.write(`id: ${e.id}\ndata: ${JSON.stringify(e)}\n\n`));
  const hb = setInterval(() => res.write(': ping\n\n'), 20_000);   // keeps proxies from idling out
  req.on('close', () => { unsub(); clearInterval(hb); });
});
```

### Client: EventSource when cookie auth suffices

```ts
const es = new EventSource('/events', { withCredentials: true });
es.onmessage = e => dispatch(JSON.parse(e.data));
es.onerror = () => { /* browser auto-reconnects and sends Last-Event-ID */ };
```

### WebSocket client with the things you must build yourself

```ts
class ReconnectingSocket {
  private attempt = 0; private ws!: WebSocket; private hb?: number;
  connect() {
    this.ws = new WebSocket(`wss://x.com/ws?ticket=${ticket()}`);   // short-lived ticket, not long-lived JWT
    this.ws.onopen = () => { this.attempt = 0; this.startHeartbeat(); this.resync(); };
    this.ws.onclose = () => this.scheduleReconnect();
    this.ws.onmessage = e => this.handle(JSON.parse(e.data));
  }
  private scheduleReconnect() {
    const delay = Math.min(30_000, 2 ** this.attempt++ * 500) * (0.5 + Math.random() / 2); // backoff + jitter
    setTimeout(() => this.connect(), delay);
  }
  // heartbeat, resync(lastSeq), backpressure, message versioning ...
}
```

### Feature mapping table

| Feature | Transport | Reason |
|---|---|---|
| AI answer | POST + streamed SSE-format response | Request-scoped, one-way, abortable |
| Order notifications | SSE (+ web push if closed) | One-way, infrequent, resumable |
| Whiteboard cursors/edits | WebSocket | Bidirectional, high-frequency |

> **Check yourself:** Give two reasons SSE is operationally simpler than WebSocket and one scenario where those advantages don't matter because WebSocket is required.

## Gotchas

- **WebSockets by default.** Choosing the heavier tool for one-way push adds reconnect/heartbeat/backplane work for no benefit.
- **`EventSource` with token auth.** No custom headers; either use cookies, a fetch-based SSE client, or short-lived signed URLs.
- **Putting long-lived JWTs in WebSocket query strings.** They land in logs; use short-lived tickets.
- **No heartbeat.** Idle timeouts on load balancers (often 60s) silently kill connections.
- **Reconnect storms.** After a deploy, all clients reconnect at once; add jittered backoff.
- **No resume/replay.** Reconnecting without catching up on missed events means silently stale UI.
- **HTTP/1.1 connection limits.** Six SSE streams per origin starve other requests; use HTTP/2 or one multiplexed stream.
- **Proxy buffering.** Streaming responses held until completion; disable buffering and set `no-cache`.
- **Ignoring tab lifecycle.** Backgrounded tabs throttle timers and may be disconnected; reconcile on `visibilitychange`.

## Follow-up Questions

**Q (High): When would you choose WebSocket over SSE?**

Answer: When the client needs to send frequent, low-latency messages as part of the real-time interaction — collaborative editing, multiplayer, presence/cursors, trading — or when binary payloads matter. If client→server traffic is occasional, regular HTTP requests plus SSE for push is simpler and sufficient.

The trap: "WebSocket is faster" without tying it to bidirectional/high-frequency need.

**Q (High): What do you have to build yourself with WebSockets that SSE gives you?**

Answer: Reconnection with backoff and jitter, resume/replay of missed messages, heartbeats/liveness detection, message framing and versioning, auth strategy for the handshake, and backpressure handling. SSE provides framing, auto-reconnect, and `Last-Event-ID` natively.

The trap: ignoring the reliability layer and assuming a socket is "just a pipe."

**Q (Medium): How do you scale each?**

Answer: Both hold long-lived connections, so capacity is memory/FD per connection. SSE streams are ordinary HTTP responses and work with standard load balancers and HTTP/2; WebSocket connections are stateful and need a pub/sub backplane (Redis/NATS) so any server can deliver to any user, and consideration of sticky routing and graceful drain on deploys. In both cases tune idle timeouts and use jittered reconnect.

The trap: claiming WebSockets don't scale or that SSE is stateless.

**Q (Medium): How would you authenticate each?**

Answer: SSE: cookies work seamlessly with `EventSource`; for bearer tokens use a fetch-based client or a short-lived signed URL. WebSocket: browsers can't set headers on the handshake, so use cookies (with origin checks to prevent CSWSH), a short-lived one-time ticket obtained via authenticated HTTP, or a first-message auth with a timeout. Avoid long-lived tokens in URLs.

The trap: putting the JWT in the query string without considering log leakage.

**Q (Low): Where does long-polling or WebTransport fit?**

Answer: Long-polling is a compatibility fallback for hostile networks, with higher overhead. WebTransport (HTTP/3) offers multiple streams and unreliable datagrams — attractive for games/media — but has limited browser/server support today, so adopt only with a clear need and fallback.

The trap: dismissing alternatives or adopting them as default.

## Self-Assessment

- [ ] Can state the data-flow criterion that separates SSE from WebSocket
- [ ] Can list what WebSocket forces you to build (reconnect, heartbeat, resume, auth)
- [ ] Can map three different features to transports with reasons
- [ ] Can describe heartbeats, `Last-Event-ID`, and jittered backoff
- [ ] Can explain auth options and pitfalls for each
- [ ] Can mention proxy buffering and HTTP/1.1 connection-limit gotchas

---
*Next: Rendering Streaming Markdown/Code Safely — back to the streamed text itself: how to turn partial, untrusted model output into safe, stable UI.*
