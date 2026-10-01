# Streaming an LLM Chat Response Into the UI

## Quick Reference

| Concern | Mechanism | Why |
|---|---|---|
| Transport | `fetch` + `ReadableStream` (or SSE) consuming chunked response | Tokens arrive incrementally; show them as they land |
| Parsing | Decode bytes with a streaming `TextDecoder`; buffer partial lines; parse SSE/NDJSON frames | Chunks don't align with message boundaries |
| Rendering | Append tokens to state; batch updates per animation frame | Per-token `setState` thrashes React and the main thread |
| Lifecycle | AbortController, status state machine (idle → streaming → done/error/aborted) | Stop button, unmount safety, race-free |

## The Scenario

"We're adding a chat assistant. The backend streams the model's reply. I want the text to appear token by token like ChatGPT, with a Stop button. Walk me through how you'd build the client side."

## Clarifying Questions

- **What wire format does the stream use — SSE (`text/event-stream`), NDJSON, or raw text chunks?** Decides the parser; SSE has framing (`data:` lines, blank-line delimiters) and a native `EventSource` that only supports GET and no custom headers.
- **Do we need POST with auth headers and a JSON body?** Almost certainly (conversation history, bearer token), which rules out `EventSource` and points to `fetch` streaming or a library like `@microsoft/fetch-event-source`.
- **Is it plain text, or markdown/code/tool-call events?** Determines the rendering pipeline and event types (covered in later scenarios).
- **What are the failure modes to handle — mid-stream disconnect, rate limit, content filter, timeout?** Mid-stream failures leave a partial message: keep it and offer retry/continue.
- **Is the conversation persisted server-side, and do we need to resume a dropped stream?** Affects whether partial output is saved and how reconnect works.
- **Scale of UI — long conversations, many concurrent streams?** Affects render performance and virtualization.

## Approach & Trade-offs

**Transport choice.** Native `EventSource` is simple but GET-only with no custom headers — a poor fit for authenticated chat POSTs. `fetch` with `response.body.getReader()` handles POST, headers, and abort via `AbortController`, at the cost of writing the framing parser yourself. WebSockets give bidirectional comms but are overkill for request→stream-response and complicate infra (see the SSE-vs-WebSocket scenario). My default: **POST via fetch, SSE-formatted stream, parsed manually**.

**Parsing correctly.** Network chunks split arbitrarily: a chunk may end mid-UTF-8 character or mid-JSON-line. Use `TextDecoder` with `{ stream: true }` so multibyte characters split across chunks decode correctly, accumulate into a buffer, split on the frame delimiter, and keep the incomplete tail for the next chunk. Naively doing `JSON.parse(chunk)` works in demos and fails in production.

**State model.** Treat a message as `{ id, role, content, status }` where status ∈ `streaming | complete | error | aborted`. Model the request lifecycle as a small state machine rather than booleans, so impossible combinations (`isLoading && isError`) can't exist. Keep the partial text when an error occurs — don't blank the answer the user was reading.

**Render performance.** A fast model can emit dozens of tokens per second. Calling `setState` per token re-renders the message list each time. Mitigations: buffer tokens in a ref and flush at most once per animation frame (`requestAnimationFrame`) or every ~30–50ms; memoize completed messages so only the streaming one re-renders; render the streaming message as its own component. Trade-off: slight latency (a frame) for far smoother UI.

**Scrolling.** Auto-scroll to bottom only if the user is already at/near the bottom; if they scrolled up to read, don't yank them down (show a "jump to latest" button). Respect `overflow-anchor` behavior and avoid layout thrash.

**Cancellation.** Stop button calls `abort()`; the reader rejects with `AbortError`, which isn't a failure — set status `aborted`, keep the partial text. Also abort on unmount and when starting a new request to prevent two streams writing into one message.

**Accessibility.** Don't make a screen reader announce every token. Use `aria-live="polite"` on a container updated in coarse chunks (e.g., sentence-level) or announce only "Response complete"; `aria-busy` while streaming. Keep Stop reachable by keyboard.

## Solution

### Stream parser (SSE over fetch)

```ts
export async function* streamChat(
  body: ChatRequest,
  signal: AbortSignal,
): AsyncGenerator<StreamEvent> {
  const res = await fetch('/api/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token()}` },
    body: JSON.stringify(body),
    signal,
  });
  if (!res.ok || !res.body) throw new HttpError(res.status);

  const reader = res.body.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      buffer += decoder.decode(value, { stream: true });   // handles split UTF-8

      let idx: number;
      while ((idx = buffer.indexOf('\n\n')) !== -1) {      // SSE frame delimiter
        const frame = buffer.slice(0, idx);
        buffer = buffer.slice(idx + 2);
        const data = frame.split('\n')
          .filter(l => l.startsWith('data:'))
          .map(l => l.slice(5).trimStart()).join('\n');
        if (data === '[DONE]') return;
        if (data) yield JSON.parse(data) as StreamEvent;
      }
    }
  } finally {
    reader.releaseLock();
  }
}
```

### Hook with batched updates and lifecycle

```tsx
function useChat() {
  const [messages, setMessages] = useState<Message[]>([]);
  const ctrlRef = useRef<AbortController | null>(null);

  const send = useCallback(async (text: string) => {
    ctrlRef.current?.abort();                              // never two streams at once
    const ctrl = new AbortController();
    ctrlRef.current = ctrl;

    const id = crypto.randomUUID();
    setMessages(m => [...m, userMsg(text), { id, role: 'assistant', content: '', status: 'streaming' }]);

    let pending = '';
    let raf = 0;
    const flush = () => {
      raf = 0;
      const chunk = pending; pending = '';
      setMessages(m => m.map(x => x.id === id ? { ...x, content: x.content + chunk } : x));
    };

    try {
      for await (const ev of streamChat({ messages: history(messages, text) }, ctrl.signal)) {
        if (ev.type === 'token') {
          pending += ev.text;
          if (!raf) raf = requestAnimationFrame(flush);   // at most once per frame
        }
      }
      cancelAnimationFrame(raf); flush();
      setStatus(id, 'complete');
    } catch (e) {
      cancelAnimationFrame(raf); flush();                  // keep partial text
      setStatus(id, (e as Error).name === 'AbortError' ? 'aborted' : 'error');
    }
  }, [messages]);

  const stop = () => ctrlRef.current?.abort();
  useEffect(() => () => ctrlRef.current?.abort(), []);     // unmount cleanup
  return { messages, send, stop };
}
```

### Streaming message component

```tsx
const Message = memo(function Message({ m }: { m: Message }) {
  return (
    <article aria-busy={m.status === 'streaming'}>
      <Markdown>{m.content}</Markdown>
      {m.status === 'streaming' && <span className="caret" aria-hidden />}
      {m.status === 'error' && <RetryBar />}
    </article>
  );
});
```

### Smart auto-scroll

```ts
const nearBottom = el.scrollHeight - el.scrollTop - el.clientHeight < 80;
if (nearBottom) el.scrollTop = el.scrollHeight;
```

> **Check yourself:** Why `TextDecoder` with `stream: true`, and why is a ref + rAF better than `setState` per token?

## Gotchas

- **`JSON.parse` on raw chunks.** Chunks split mid-frame; buffer until a delimiter.
- **Decoding without `{ stream: true }`.** Corrupts multibyte characters (emoji, CJK) split across chunks.
- **`EventSource` for authenticated POST.** It can't send headers or a body.
- **Per-token `setState`.** Re-renders and long tasks tank INP on long answers.
- **Treating abort as an error.** User-initiated Stop shouldn't show an error banner.
- **Dropping partial text on failure.** Users lose the content they already read; keep it and offer retry/continue.
- **Two concurrent streams.** Sending again before the first finishes interleaves tokens; abort the previous.
- **Aggressive auto-scroll.** Fights the user trying to read earlier text.
- **Screen reader spam.** Announcing every token makes the UI unusable with AT.
- **Proxy/CDN buffering.** Some proxies buffer responses and defeat streaming; ensure `Cache-Control: no-cache`, disable buffering (`X-Accel-Buffering: no`).

## Follow-up Questions

**Q (High): Why not use `EventSource`?**

Answer: It only supports GET, can't set headers (no `Authorization`), and can't send a request body, which chat needs. It also auto-reconnects in ways that may resend or duplicate a generation. `fetch` streaming gives control over method, headers, abort, and error handling; libraries wrap the SSE parsing.

The trap: knowing only `EventSource` and missing the auth/body limitation.

**Q (High): How do you handle partial chunks and unicode?**

Answer: Keep a persistent `TextDecoder` and decode with `{ stream: true }` so incomplete multibyte sequences carry over; accumulate text into a buffer, split on the protocol delimiter, process complete frames only, and keep the remainder for the next read.

The trap: assuming one chunk equals one message.

**Q (High): How do you keep the UI smooth when tokens arrive very fast?**

Answer: Decouple arrival from rendering: accumulate in a ref and flush once per animation frame (or ~50ms), memoize finished messages, isolate the streaming message in its own component, and avoid re-parsing markdown for the whole conversation each token. Measure INP/long tasks.

The trap: "React batches updates, so it's fine."

**Q (Medium): How do you implement Stop and handle it correctly?**

Answer: One `AbortController` per request; Stop calls `abort()`, which rejects the pending `read()` with `AbortError`; treat it as a distinct `aborted` state, keep the partial text, release the reader, and ensure the server also cancels generation (connection close should stop upstream billing/work).

The trap: only hiding the spinner client-side while the server keeps generating.

**Q (Medium): What if the connection drops mid-stream?**

Answer: Keep the partial content, mark the message errored/incomplete, and offer "Continue"/"Retry." If the server supports resumable streams (event IDs/`Last-Event-ID` or a generation ID you can re-attach to), resume from the last event; otherwise regenerate. Don't silently retry a non-idempotent generation.

The trap: discarding the message or auto-retrying and duplicating output.

**Q (Low): How do you make streaming accessible?**

Answer: Mark the container `aria-busy` during streaming, avoid per-token live announcements, announce completion (or coarse chunks) via a polite live region, keep focus stable, and provide a keyboard-operable Stop.

The trap: ignoring accessibility entirely or marking the whole transcript `aria-live`.

## Self-Assessment

- [ ] Can explain fetch streaming vs. `EventSource` and pick one with reasons
- [ ] Can write a buffered SSE parser with `TextDecoder` streaming
- [ ] Can batch token updates per frame and justify it
- [ ] Can model message status as a state machine incl. aborted
- [ ] Can implement Stop with AbortController and describe server-side cancellation
- [ ] Can handle mid-stream failure while preserving partial text

---
*Next: Choosing SSE vs. WebSocket for a Real-time Feature — generalizes the transport choice made here.*
