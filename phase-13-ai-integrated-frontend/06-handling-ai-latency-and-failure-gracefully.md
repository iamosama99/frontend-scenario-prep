# Handling AI Latency and Failure Gracefully in the UI

## Quick Reference

| Problem | Mechanism | Why |
|---|---|---|
| Slow time-to-first-token | Immediate acknowledgment, skeleton/typing indicator, staged status text, streaming | Perceived latency matters as much as actual; silence feels broken |
| Transient failures (5xx, network, overload) | Bounded retry with backoff + jitter, only when safe; surface "Retry" | Recover without duplicating work or hammering a struggling backend |
| Rate limits / quota | Honor `Retry-After`, explain, queue or disable with countdown | Clear, actionable feedback beats a generic error |
| Wrong/low-quality output | Honest framing, regenerate/edit/feedback, citations, no hard-fail UX | AI failure isn't only exceptions — it's also confidently wrong answers |

## The Scenario

"Our AI feature has p50 time-to-first-token of 1.5s but p95 of 12s, a couple of percent of requests fail, and we sometimes get rate-limited at peak. Users are rage-clicking and complaining that it 'just hangs.' How do you design the UI to handle latency and failure gracefully?"

## Clarifying Questions

- **Where is the latency — queueing, model time-to-first-token, total generation time, or retrieval/tool calls before generation?** Different phases deserve different feedback (and different fixes).
- **What failure types occur, and which are retryable?** Network drop, 429 rate limit, 503 overload, timeout, content-policy refusal, malformed output — each needs distinct handling and copy.
- **Are requests idempotent / is there a cost to duplicating a generation?** Auto-retry on a non-idempotent or expensive/side-effecting request can double-charge or double-act.
- **Can we stream, and can we resume?** Streaming turns a long total time into a short perceived wait; resume avoids restarting.
- **What are the fallback options — a smaller/faster model, cached answer, non-AI path?** Graceful degradation needs something to degrade to.
- **What are the business/UX expectations — hard timeout, SLAs?** Defines when to give up.

## Approach & Trade-offs

**Design for the whole distribution, not the median.** p50 is fine; the pain is at p95 and in failures. The UI should have defined behavior at each time threshold, not a spinner that spins until it doesn't.

**Latency: manage perception honestly.**

1. *Acknowledge instantly (<100ms)*: show the user's message and an assistant placeholder immediately (optimistic UI), disable duplicate submits, move focus appropriately.
2. *Stream* whenever possible; first token ends the wait. Time-to-first-token is the metric to optimize and display against.
3. *Progress that conveys real stages* when work precedes generation ("Searching documents…", "Reading 3 sources…") — truthful and event-driven, never fake progress bars.
4. *Escalating feedback by elapsed time:* 0–2s typing indicator; 2–8s "Still working on it…"; >8–15s offer "This is taking longer than usual — keep waiting or cancel", with Stop always available. Hard-timeout at a product-defined limit and convert to a recoverable error.
5. *Prevent rage-clicking:* disable/replace the Send action while pending, and make repeat-clicks idempotent (client-generated request IDs deduped server-side).

**Failure: classify, then respond.**

| Failure | UI behavior | Retry? |
|---|---|---|
| Network offline/drop pre-response | "You're offline" banner; queue send; auto-retry on reconnect | Yes (safe: nothing generated) |
| Drop mid-stream | Keep partial text, mark incomplete, offer Continue/Retry | User-initiated |
| 429 / quota | Explain, show countdown from `Retry-After`; optionally queue | After delay |
| 5xx/overload | Short automatic retry with backoff + jitter (max 2–3), then manual Retry | Auto within limits |
| Timeout | Offer Retry / fallback model | User-initiated |
| Content-policy refusal / 4xx | Explain plainly, no retry button that can't work; suggest rewording | No |
| Malformed/empty output | Treat as failure; offer regenerate; log | User-initiated |

**Retries need judgment.** Retry only idempotent or not-yet-started requests automatically; use exponential backoff with jitter to avoid synchronized thundering herds; cap attempts and total time; and make the retry visible ("Retrying… attempt 2/3") so the user isn't staring at silence. For expensive generations, prefer user-initiated retry after a failure rather than silent repeats.

**Preserve user work.** Never lose the user's prompt or draft on failure: keep it in the input or the message with an inline retry. Keep partial outputs. Persist drafts across reloads.

**Graceful degradation.** Options: fall back to a faster/smaller model after a timeout (and disclose it), serve cached/previous answers, or degrade to non-AI functionality (search results without the summary). Circuit-breaker style: if the AI service is failing broadly, stop sending new requests for a cool-down and show a status message rather than letting every user wait through a timeout.

**Failure includes "wrong".** Models can be confidently incorrect. Design for it: show sources/citations, label AI-generated content, provide regenerate, edit, and thumbs up/down feedback, avoid auto-applying consequential changes, and make verification easy. This is a UX responsibility, not only a backend one.

**Observability.** Instrument time-to-first-token, total time, error rate by type, retry rates, abandonment/cancel rate, and rage-click events. These tell you whether the UI changes actually helped, and which phase to optimize.

**Accessibility.** Announce status changes (polite live region for "Still working…", assertive for errors requiring action), don't rely on color or animation alone, respect reduced motion for typing indicators, and keep error actions keyboard reachable.

**Trade-offs.** More states = more UI to build and test; aggressive auto-retry improves success rate but risks duplicate cost and masks outages; falling back to a weaker model preserves availability at the cost of quality (and trust, if undisclosed). Choose based on request cost and user stakes.

## Solution

### Request state machine

```ts
type GenState =
  | { s: 'idle' }
  | { s: 'pending'; startedAt: number; reqId: string; stage?: string }      // before first token
  | { s: 'streaming'; text: string }
  | { s: 'done'; text: string }
  | { s: 'error'; kind: 'network' | 'rate_limited' | 'overloaded' | 'timeout' | 'refused' | 'empty';
      partial?: string; retryAfter?: number; canRetry: boolean }
  | { s: 'aborted'; partial: string };
```

### Elapsed-time feedback

```tsx
function PendingIndicator({ startedAt, stage, onCancel }: Props) {
  const elapsed = useElapsed(startedAt);            // ticks each second
  const message =
    elapsed < 2 ? null :
    elapsed < 8 ? (stage ?? 'Working on it…') :
    'This is taking longer than usual.';
  return (
    <div role="status" aria-live="polite">
      <TypingDots aria-hidden />
      {message && <p>{message}</p>}
      {elapsed >= 8 && <button onClick={onCancel}>Cancel</button>}
    </div>
  );
}
```

### Retry with backoff + jitter, only where safe

```ts
async function withRetry<T>(fn: (signal: AbortSignal) => Promise<T>, signal: AbortSignal) {
  const max = 3;
  for (let attempt = 0; ; attempt++) {
    try { return await fn(signal); }
    catch (e) {
      const err = classify(e);
      if (!err.retryable || attempt >= max - 1 || signal.aborted) throw err;
      const base = err.retryAfterMs ?? Math.min(8000, 500 * 2 ** attempt);
      const delay = base * (0.5 + Math.random() / 2);          // jitter
      onRetrying(attempt + 1, max);                            // visible to the user
      await sleep(delay, signal);
    }
  }
}
```

### Idempotency against double-submit and retries

```ts
await fetch('/api/chat', {
  method: 'POST',
  headers: { 'Idempotency-Key': reqId },     // server dedupes; retries don't double-generate/charge
  body: JSON.stringify(payload),
  signal,
});
```

### Error copy mapped to action

```tsx
const ERROR_UI: Record<ErrorKind, (e: GenError) => ReactNode> = {
  rate_limited: e => <>You've hit the limit. Try again in <Countdown to={e.retryAfter} />.</>,
  overloaded:   () => <>The assistant is busy right now. <Retry /></>,
  timeout:      () => <>That took too long. <Retry /> <UseFasterModel /></>,
  network:      () => <>Connection lost. Your message is saved — <Retry /></>,
  refused:      () => <>I can't help with that request. Try rephrasing it.</>,   // no Retry
  empty:        () => <>No response came back. <Retry /></>,
};
```

### Degradation and circuit breaker

```ts
if (breaker.isOpen('ai')) return <AiUnavailable fallback={<PlainSearchResults q={q} />} />;
```

### Metrics to track

TTFT p50/p95, total time, error rate by kind, auto-retry success rate, cancel/abandon after N seconds, rage-click rate, fallback usage.

> **Check yourself:** For a 429, a mid-stream disconnect, and a content-policy refusal, state what the UI does, whether it auto-retries, and why.

## Gotchas

- **Infinite spinner.** No time-based escalation, no cancel, no timeout → users assume it's broken.
- **Fake progress bars.** Dishonest and erode trust; show real stages or none.
- **Auto-retrying non-idempotent or costly requests.** Duplicates work, charges, side effects.
- **Retry without jitter/backoff.** Synchronized retries amplify an outage.
- **Generic "Something went wrong."** Not actionable; distinguish error kinds.
- **A Retry button on errors that can't succeed** (policy refusals) — frustrating.
- **Discarding user input or partial output on failure.** Preserve both.
- **Undisclosed model fallback.** Quality change without user awareness harms trust.
- **Ignoring "wrong but successful" outputs.** Provide citations, regenerate, and feedback.
- **Silent live regions.** Screen reader users get no indication that anything is happening or failed.
- **Every client timing out separately during an outage.** Use a circuit breaker/status banner to fail fast.

## Follow-up Questions

**Q (High): How do you improve perceived latency when the model is slow?**

Answer: Acknowledge instantly with optimistic UI, stream so the first token ends the wait, show truthful stage-based progress when pre-generation work occurs, escalate messaging with elapsed time, and keep Stop/Cancel available. Measure time-to-first-token and optimize that, since total generation time matters less once streaming starts.

The trap: only talking about a spinner, or proposing fake progress.

**Q (High): When should the client retry automatically, and when not?**

Answer: Automatically only for failures that are transient and safe — network errors before any generation, 503/overload — using capped exponential backoff with jitter and visible status, ideally with idempotency keys. Don't auto-retry policy refusals/4xx, mid-stream failures that would duplicate output, or expensive/side-effecting requests; offer a user-initiated retry or continue instead. Honor `Retry-After` for 429.

The trap: "just wrap it in retry(3)."

**Q (Medium): How do you handle a mid-stream failure?**

Answer: Keep the partial text, mark the message incomplete with a clear indicator, and offer Continue (if the server can resume from the last event/ID) or Retry/Regenerate. Don't discard what the user already read, and don't silently restart in a way that changes the text under them.

The trap: showing a generic error and clearing the message.

**Q (Medium): How do you degrade gracefully when the AI service is down or slow?**

Answer: Use a circuit breaker to stop piling requests onto a failing service, show a clear status, and fall back: a faster model (disclosed), cached results, or the non-AI experience (e.g., plain search). Preserve the user's input so they can retry later.

The trap: letting every request wait for a full timeout, and showing a blank panel.

**Q (Low): How do you address outputs that are confidently wrong?**

Answer: Treat it as a UX failure mode: label AI content, show sources/citations, make regenerate/edit/feedback easy, avoid auto-applying consequential outputs, and surface uncertainty where the system has signal. Verification affordances are part of graceful failure.

The trap: considering only HTTP errors as failures.

## Self-Assessment

- [ ] Can describe time-based escalating feedback and why TTFT matters
- [ ] Can classify failure types and give UI + retry policy for each
- [ ] Can implement backoff with jitter, `Retry-After`, and idempotency keys
- [ ] Can explain preserving user input and partial output
- [ ] Can describe circuit breaker/fallback strategies with disclosure
- [ ] Can name the metrics that show whether the UX improved

---
*Phase 13 complete — all 122 scenarios across 13 phases are done.*
