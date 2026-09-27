# Streaming LLM Response UI: SSE vs. WebSocket

## Quick Reference

| Dimension | SSE (`EventSource` / `fetch` + `ReadableStream`) | WebSocket |
|---|---|---|
| Direction | Server → client only | Bidirectional |
| Protocol | Plain HTTP (works through existing proxies/load balancers unmodified) | Its own upgrade handshake — some infra needs explicit config |
| Built-in reconnect | `EventSource` has it natively, with `Last-Event-ID` resume | None — must be hand-rolled (see [[04-websocket-reconnection-backoff]]) |
| Cancellation | Client aborts the HTTP request (`AbortController`) | Client sends a message or closes the socket |
| Best fit here | A single request → a single streamed response, no client-to-server messages needed mid-stream | Multiple concurrent streams, or the client needs to send data *while* a response is streaming (rare for basic chat) |

## The Scenario

"We're adding an AI chat feature — the response needs to stream in token-by-token like ChatGPT, not appear all at once after a 10-second wait. The team's split: half want to use WebSockets since 'it's real-time,' the other half say that's overkill for what's fundamentally a request-response pattern. Make the call on the transport, and design the client-side handling for starting a stream, receiving it, and letting the user cancel mid-response."

## Clarifying Questions

- **Does the client ever need to send anything to the server *while* a response is actively streaming** — e.g., a genuinely bidirectional protocol where the user can interrupt with a follow-up mid-generation and have the model incorporate it live, versus the much more common pattern of "send one prompt, receive one streamed response, then the interaction is done until the next prompt"? This is the single question that resolves the team's actual disagreement — if it's request-then-stream-response with no mid-stream client input, that's a fundamentally one-directional pattern regardless of how "real-time" it feels to the user, and one-directional doesn't need a bidirectional transport.
- **Does the chat interface need multiple concurrent, independent streams** — e.g., streaming several draft responses to compare, or streaming a response while a separate tool-call/status channel also needs to push updates simultaneously — or is it always exactly one active stream per conversation turn? Multiple genuinely concurrent streams per user session is a scenario where a single persistent WebSocket connection multiplexing several logical streams can be a real architectural advantage over opening several separate SSE/fetch-stream HTTP requests.
- **What does "let the user cancel mid-response" need to actually stop** — just the client-side rendering of further tokens (cosmetic), or does it need to also stop the server (and the LLM provider) from continuing to generate and bill for tokens the user will never see? This changes whether cancellation is purely a client-side UI concern or needs an explicit signal sent to the server to propagate the cancellation upstream to the LLM API call itself.
- **Is there existing infrastructure (a load balancer, corporate proxy, CDN in front of the API) known to have trouble with either long-lived HTTP connections or WebSocket upgrade requests?** Some infrastructure (older reverse proxies, certain corporate network policies, some serverless/edge runtimes) handles one better than the other — this is a legitimate, non-hypothetical factor in the transport decision, not just a matter of taste.
- **Does the response format need anything beyond plain text tokens** — structured events (e.g., "a tool call started," "here are three citation sources," "generation finished with reason: length") interleaved with content tokens? Both SSE and WebSocket can carry structured/typed events (SSE via the `event:` field, WebSocket via a JSON envelope), so this doesn't change the transport decision much on its own, but it does shape the client-side message-handling design either way.

## Approach & Trade-offs

**The team's disagreement is really about whether "feels real-time to the user" implies "needs a bidirectional transport" — it doesn't, and that's the crux of the recommendation.** Streaming a single LLM response token-by-token is, at the protocol level, still fundamentally one request producing one long-lived, incrementally-delivered response — the client sends one prompt, the server (proxying to the LLM provider) streams tokens back as they're generated, and the interaction is complete when the stream ends. Nothing about this requires the client to send additional messages *while* the response is streaming for the common chat pattern (send prompt → receive streamed response → send next prompt only after this one is done, or after cancelling it). That's a textbook fit for SSE (or the increasingly common `fetch` + `ReadableStream` pattern, which most LLM provider APIs — OpenAI, Anthropic — already use for their own streaming responses) rather than WebSockets.

**SSE (or a `fetch`-based streaming `POST`, which most LLM chat UIs actually use in practice rather than a literal `EventSource`, since `EventSource` is GET-only and chat prompts are naturally a `POST` body) gets several things for free that a WebSocket implementation would need to hand-roll.** It runs over plain HTTP, so it passes through existing infrastructure (load balancers, corporate proxies, CDNs) that's already configured for HTTP without needing WebSocket-specific upgrade support, which matters more than it sounds for anything deployed behind infrastructure the frontend team doesn't fully control. True `EventSource` additionally provides automatic reconnection with `Last-Event-ID`-based resume built into the browser (see [[04-websocket-reconnection-backoff]] for the general reconnection/backoff problem this solves natively here) — though this specific benefit is lost if using the more common `fetch` + `ReadableStream` + `POST` pattern instead of literal `EventSource`, which is a real trade-off worth naming (see the Solution section).

**Where WebSockets would actually be the better call: genuinely concurrent, independent, bidirectional needs.** If the product needed several simultaneously-active streams multiplexed over one connection (comparing multiple draft responses live, or a persistent "presence"/typing-indicator channel that needs to coexist with response streaming), or if the client needs to send data *while* a response is generating (true mid-generation interruption/steering, not just "cancel"), a single persistent WebSocket connection genuinely earns its complexity there — those are legitimate bidirectional, multi-stream requirements that SSE's one-directional, one-stream-per-request model doesn't fit well. For the scenario as stated (one prompt, one streamed response, cancel = stop), that need doesn't exist, so the added complexity of a WebSocket (its own reconnection logic, no free HTTP-infra compatibility, a persistent connection to manage) isn't justified by a corresponding benefit.

**Cancellation needs to be designed as an explicit two-layer concern: stopping client-side rendering, and stopping server-side (and upstream LLM-provider) generation — conflating them under-delivers.** Simply stopping the client from processing further incoming chunks (closing the local reader, ignoring subsequent data) is trivial and instant from the user's perspective, but if the underlying HTTP request/stream to the server (and the server's own upstream call to the LLM provider) isn't also terminated, the server keeps generating and billing for tokens nobody will ever see — a real cost and resource-usage bug hiding behind what looks like a fully-working cancel button. `AbortController` is the mechanism that closes both gaps correctly when used properly: aborting the underlying `fetch` request both stops the client from receiving further data *and* signals the connection closure to the server, which (if the server is correctly written to detect a closed client connection) should propagate that cancellation upstream to stop the LLM provider call too.

## Solution

**Step 1 — client sends the prompt as a normal `POST`, reading the response body as a stream rather than waiting for it to complete.** This is the `fetch` + `ReadableStream` pattern most production LLM chat UIs actually use (not literal `EventSource`, since prompts are POST bodies, and `EventSource` only supports GET):

```ts
async function streamChatResponse(prompt: string, signal: AbortSignal) {
  const response = await fetch('/api/chat', {
    method: 'POST',
    body: JSON.stringify({ prompt }),
    headers: { 'Content-Type': 'application/json' },
    signal, // wired to the cancel button, see Step 3
  });
  if (!response.body) throw new Error('No response body');

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });
    // Server sends newline-delimited SSE-style events: "data: {...}\n\n"
    const events = buffer.split('\n\n');
    buffer = events.pop() ?? ''; // last, possibly-incomplete chunk stays buffered
    for (const raw of events) {
      const json = raw.replace(/^data: /, '');
      const event = JSON.parse(json);
      handleStreamEvent(event); // see Step 2
    }
  }
}
```

Parsing incrementally, buffering an incomplete trailing chunk rather than assuming each `read()` call yields exactly one complete event, is necessary because TCP/HTTP chunking has no obligation to align with the application-level event boundaries — a single `read()` can deliver a partial event, multiple events, or a mid-event split.

**Step 2 — structured event handling, not just raw text tokens, since the response likely carries more than content (start/end markers, errors, metadata):**

```ts
function handleStreamEvent(event: StreamEvent) {
  switch (event.type) {
    case 'token':
      appendToken(event.text); // batched into the render loop, see below
      break;
    case 'done':
      finalizeMessage(event.finishReason);
      break;
    case 'error':
      showStreamError(event.message);
      break;
  }
}
```

**Step 3 — cancellation wired through `AbortController`, closing both the client-side and server-side gaps described in the trade-offs:**

```tsx
function useChatStream() {
  const abortControllerRef = useRef<AbortController | null>(null);

  const send = (prompt: string) => {
    const controller = new AbortController();
    abortControllerRef.current = controller;
    streamChatResponse(prompt, controller.signal).catch((err) => {
      if (err.name !== 'AbortError') showStreamError(err.message);
      // AbortError is the expected, intentional-cancel case — not a real error
    });
  };

  const cancel = () => {
    abortControllerRef.current?.abort(); // closes the fetch, signals server via connection close
  };

  return { send, cancel };
}
```

Server-side (conceptually, in whatever backend framework): the handler needs to detect the client's connection closing (most server frameworks expose this as a request-abort/close event) and use it to abort its own upstream call to the LLM provider's streaming API — most LLM provider SDKs accept an abort signal on their own streaming calls for exactly this purpose, propagating the cancellation the last mile to where tokens are actually generated and billed.

**Step 4 — batch token rendering rather than triggering a React re-render on every single incoming token.** LLM streams can deliver tokens fast enough (especially in bursts) that naive per-token `setState` calls cause visible jank or excessive re-render overhead:

```tsx
function useTokenBuffer(onFlush: (text: string) => void) {
  const bufferRef = useRef('');
  const frameRef = useRef<number | null>(null);

  const append = (token: string) => {
    bufferRef.current += token;
    if (frameRef.current === null) {
      frameRef.current = requestAnimationFrame(() => {
        onFlush(bufferRef.current);
        bufferRef.current = '';
        frameRef.current = null;
      });
    }
  };
  return append;
}
```

Coalescing to one state update per animation frame (rather than per token) keeps rendering smooth regardless of how fast tokens arrive, without introducing a perceptible delay (a single frame is imperceptible) — the same batching principle as the resync-burst handling in [[04-websocket-reconnection-backoff]], applied here to a different source of high-frequency updates.

> **Check yourself:** If the team later adds a requirement that the user can send a follow-up message *while* the current response is still streaming (and have both stream concurrently, interleaved in the UI), does that change the transport recommendation? Why or why not — and if it does, at what specific point does the "one prompt, one stream" assumption break?

## Gotchas

**Choosing WebSockets by default because streaming "feels real-time," without identifying an actual bidirectional or multi-stream requirement that justifies it.** This is precisely the team disagreement in the prompt, and picking WebSockets here means carrying real added complexity (hand-rolled reconnection/backoff, no free HTTP-infra compatibility) for a capability (bidirectional, mid-stream client-to-server messaging) the feature doesn't use.

**Building cancellation that only stops client-side rendering, leaving the server (and the LLM provider call) still generating and billing after the user clicks cancel.** This is easy to miss because it looks completely correct from the browser — the UI stops updating instantly — while silently leaking cost and server resources on every cancelled generation, which at any real usage volume adds up.

**Assuming each `read()` call from the stream reader yields exactly one complete, parseable event**, and parsing without buffering a possibly-incomplete trailing chunk — this works in local testing (where chunk boundaries often happen to align conveniently) and breaks intermittently in production under real network chunking behavior, producing JSON parse errors that look randomly occurring rather than a straightforward buffering bug.

**Re-rendering on every single incoming token with a naive `setState`**, rather than batching to a frame or a small time window — works fine on a fast connection with a slow-generating model, but visibly janks (or drops frames) with a fast model generating tokens faster than React's render cycle can comfortably keep up with one state update per token.

**Not handling the `AbortError` case distinctly from a genuine network/server error** — if a cancelled request's resulting `AbortError` is caught and displayed as "Something went wrong," an intentional user action (clicking cancel) surfaces as if it were a failure, which is a confusing, self-inflicted UX bug distinct from any actual streaming/networking issue.

## Follow-up Questions

**Q (High): The prompt says the team is split on WebSockets vs. something simpler. Give the precise technical reason streaming a chat response doesn't require a bidirectional transport, even though the UX feels live/real-time.**

Answer: "Real-time" describes the *user's perception* of the response arriving incrementally rather than all at once — it says nothing about which direction data needs to flow. The actual data flow for a standard chat turn is: client sends one request (the prompt) to the server, and the server sends back one logical response, just delivered incrementally over time rather than as a single completed payload — this is still a one-directional, request-then-response pattern at the protocol level, just with the response's *delivery* streamed instead of buffered-then-sent-whole. A bidirectional transport (WebSocket) is justified when the client needs to send additional, independent messages to the server *during* an already-in-progress exchange, or when multiple logically-distinct streams need to be multiplexed concurrently over one connection — neither of which the basic "send prompt, receive streamed response, optionally cancel" pattern requires; cancellation itself is expressible as closing the client's side of the one-directional HTTP request (via `AbortController`), not as sending a message over an otherwise-idle reverse channel.

The trap: conflating "the UI updates incrementally/live" with "the underlying transport must support bidirectional communication" — incremental delivery of one response and bidirectional message exchange are orthogonal properties, and the former doesn't imply the latter.

---

**Q (High): Explain exactly what closing the client's `AbortController` needs to trigger server-side for cancellation to actually stop LLM token generation, not just client-side rendering. Where can this chain break?**

Answer: `controller.abort()` closes the underlying `fetch` request from the client's side, which the browser communicates by closing the underlying connection to the server — most server frameworks surface this as a request-close/abort event the handler can listen for. For cancellation to actually stop generation (not just stop the client from *displaying* further tokens), the server's request handler needs to (a) be listening for that connection-close event, and (b) use it to call the abort/cancel mechanism on its own upstream call to the LLM provider's streaming API (most LLM provider SDKs accept an `AbortSignal` or equivalent on their streaming methods specifically for this purpose) — propagating the cancellation the full distance from "browser closed the connection" to "provider stops generating and stops billing for further tokens." This chain can break at either link: if the server handler doesn't listen for the client disconnect at all, the server keeps consuming the upstream stream to completion regardless of what the client did (silent resource/cost leak, invisible from the client's perspective since its own UI looks correctly cancelled); if the server does detect the disconnect but doesn't wire that into aborting the *upstream* provider call specifically (e.g., it stops trying to write to the now-closed client connection but leaves the provider stream running server-side, perhaps just discarding the data), the provider still generates and bills for tokens even though nothing further reaches the client.

The trap: verifying cancellation purely by observing the browser UI stop updating — that only confirms the client-side half of the chain works; without checking (or at least reasoning about) server-side behavior specifically, a cancel button that looks completely correct in the browser can still be silently leaking cost on every use.

---

**Q (High): Why does the streaming response parser need to buffer a possibly-incomplete trailing chunk rather than assuming each `read()` call returns one or more complete events? What HTTP-level fact makes this necessary?**

Answer: HTTP chunked transfer encoding (or the underlying TCP segmentation beneath it) has no awareness of, or obligation to preserve, application-level message boundaries — a "chunk" at the HTTP/TCP transport level and an "event" at the application level (e.g., one `data: {...}\n\n` block) are unrelated units, and a single transport-level chunk delivered to a `read()` call can contain zero complete application events (a mid-event fragment), exactly one, several concatenated together, or several plus a trailing partial one — this is a function of network conditions, buffering behavior, and timing that the application has no control over and can't assume away. Parsing code that assumes "one `read()` = one complete, parseable JSON event" and calls `JSON.parse` directly on each `read()`'s raw output will intermittently fail whenever a chunk boundary happens to fall mid-event — which, critically, tends to work fine in local development (small payloads, low latency, chunk boundaries that happen to align) and fail unpredictably in production under real-world network chunking, making it a classic "works on my machine" bug class. The correct approach (as in Step 1) accumulates incoming data into a buffer, splits on the application-level delimiter (`\n\n` for SSE-style framing), processes every *complete* event found, and retains whatever trailing, possibly-incomplete fragment remains in the buffer for the next `read()` to complete.

The trap: testing this locally, where fast localhost connections and small test payloads often happen to deliver each event as its own clean chunk, and concluding the naive "one read = one event" parsing is correct — the actual failure mode only reliably surfaces under production network conditions with different chunking behavior, making this an easy bug to ship without a reviewer or the author ever seeing it fail in development.

---

**Q (Medium): If the product later needs to show "3 people are viewing this document" presence indicators alongside the streaming chat response, does that change the SSE-vs-WebSocket recommendation for the chat stream itself?**

Answer: Not necessarily for the chat stream in isolation — presence indicators are a separate, genuinely bidirectional-ish, persistent, low-frequency-update concern (who's currently connected, updated as people join/leave) that's a reasonable candidate for its own WebSocket connection (or its own SSE stream, since presence updates are also server-to-client-only in most implementations) independent of how any single chat response streams. The question worth asking before reaching for one shared WebSocket to carry both is whether combining them into a single multiplexed connection is actually simpler than running them as two independent connections/streams — multiplexing chat-response streaming and presence updates over one WebSocket requires an envelope/routing scheme to distinguish message types on the client (which the code already partially does via `event.type` in Step 2, so it's not a large lift), and saves one connection's worth of overhead, but it also means a chat-response stream's lifecycle (started/cancelled per message) is now living inside a connection whose other purpose (presence) is persistent and long-lived, coupling their connection-management concerns together. Unless there's a specific efficiency or architectural reason to unify them (e.g., a hard cap on concurrent connections per client in a constrained environment), keeping the chat-response stream on its own request-scoped SSE/fetch-stream and presence on its own separate, persistent WebSocket is usually the more decoupled, easier-to-reason-about design — each transport doing the job it's naturally suited for, rather than forcing one shared connection to serve two differently-shaped needs.

The trap: assuming that because *a* WebSocket is now justified for presence, the *chat stream* should be moved onto it too "since we have one now anyway" — the presence feature's bidirectional/persistent nature doesn't retroactively change what the chat-response stream itself actually needs, and conflating the two concerns' transport choices isn't automatically the more efficient design.

---

**Q (Medium): How would you test that cancellation genuinely stops server-side/upstream generation, not just client-side rendering — what would you actually check, beyond watching the UI stop updating?**

Answer: Watching the UI stop updating only verifies the client-side half of the chain, so I'd verify the server-side half independently: instrument the server handler to log (or expose via a test-only endpoint/metric) when it detects a client disconnect and when it actually calls abort on the upstream LLM provider request, then in a test, start a stream, cancel it client-side partway through, and assert both that the server-side disconnect-detection fired and that the upstream provider call was actually aborted (most provider SDKs' streaming clients expose some observable state, or the test can use a mocked/stubbed provider client and assert `abort()` was called on it) — rather than only asserting on client-visible behavior. Additionally worth checking with the real provider (not just mocked) in a staging environment: whether cancelling actually stops billed token usage, which may require checking the provider's own usage/billing API after a deliberately-cancelled request, since "we called abort on our end" doesn't guarantee the provider's infrastructure actually stops generation and billing promptly — some providers have documented behavior here, and it's worth confirming rather than assuming.

The trap: writing a test (or manual verification) that only asserts on client-observable state (the UI stops rendering new tokens after cancel) and calling cancellation "done" — that test would pass identically whether or not the server actually propagated the cancellation upstream, since the client can't observe the server's internal behavior from the browser; genuinely verifying this requires server-side instrumentation or provider-side usage confirmation, not just client-side UI assertions.

---

**Q (Low): Would Server-Sent Events' native `Last-Event-ID` reconnection feature be useful for resuming a chat response if the connection drops mid-stream? What would need to be true for it to work?**

Answer: In principle yes — if using a literal `EventSource` (rather than the more common `fetch`+`ReadableStream`+`POST` pattern used in Step 1, which forgoes this native feature since `EventSource` can't send a POST body), each streamed token/chunk could carry an `id:` field, and on an unexpected disconnect, the browser's native `EventSource` reconnection would automatically include a `Last-Event-ID` header on its reconnect request, letting a correctly-implemented server resume the stream from that point rather than restarting generation from scratch. For this to actually work for an LLM response specifically, the server would need to have buffered (or be able to reconstruct) everything generated after that ID so far, and — more fundamentally — since most production chat streaming uses POST (to send the prompt as a body) rather than GET (which `EventSource` requires), getting this native resume behavior means either restructuring the request to fit `EventSource`'s GET-only model (e.g., creating the generation server-side via an initial POST that returns a stream ID, then opening an `EventSource` GET connection to `/stream/{id}` to actually receive it) or hand-rolling the equivalent resume logic on top of `fetch`+`ReadableStream`, similar in spirit to the sequence-number-based resync design in [[04-websocket-reconnection-backoff]], since `fetch` streams don't get `Last-Event-ID` resume for free the way true `EventSource` does.

The trap: assuming the `fetch`+`ReadableStream`+`POST` pattern (the one actually used in Step 1 and by most real LLM chat UIs) gets `EventSource`'s native reconnection/resume behavior for free just because it's conceptually "SSE-style" — that native resume mechanism specifically belongs to the `EventSource` API and its GET-based reconnect protocol, not to the general idea of a text/event-stream-shaped payload; using `fetch` directly forgoes it and would need hand-rolled resume logic to get equivalent behavior.

---

## Self-Assessment

- [ ] Can explain precisely why "feels real-time" doesn't imply "needs a bidirectional transport," in terms of actual data-flow direction
- [ ] Can identify the concrete criteria (mid-stream client messages, multiple concurrent streams) that would actually justify choosing WebSockets over SSE/fetch-streaming
- [ ] Can design cancellation as a two-layer problem (client rendering + server/upstream generation) and explain how `AbortController` closes both, and where the chain can silently break
- [ ] Can explain why a stream parser must buffer incomplete trailing chunks, citing the HTTP/TCP chunking fact that makes it necessary
- [ ] Can design frame-batched token rendering and explain why per-token `setState` causes jank
- [ ] Can describe how `Last-Event-ID`-based resume works and why it's specific to true `EventSource`, not `fetch`-based streaming

---
*Next: Graceful Degradation for a Flaky Third-party API — from designing your own streaming transport to handling a dependency you don't control failing unpredictably underneath you.*
