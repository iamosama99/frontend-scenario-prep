# Cancelling a Long-running AI Request Cleanly

## Quick Reference

| Layer | Mechanism | Why |
|---|---|---|
| Client | One `AbortController` per request; abort on Stop, unmount, new send, route change | Releases the connection, stops UI updates, prevents interleaved streams |
| UI state | Distinct `aborted` status; keep partial output; re-enable input | Cancel is a user intent, not an error |
| Server | Detect client disconnect → cancel upstream model call, stop billing/work | Otherwise "Stop" only hides the cost |
| Long jobs | Job ID + `DELETE /jobs/:id` (or cancel endpoint) + poll/stream status | A closed connection can't cancel work that outlived the request |

## The Scenario

"Users start a long generation — say a 3,000-token report or a multi-step agent run — and want to hit Stop. Product also says: if they navigate away or send a new prompt, the old one should stop. How do you make cancellation actually work end to end?"

## Clarifying Questions

- **Is the generation tied to the HTTP request (stream open = job running), or is it a background job the request merely observes?** Request-scoped work stops when the connection closes; detached jobs need an explicit cancel API.
- **What should happen to partial output — keep, discard, save to history?** Product decision with data and UX implications (and cost: users pay/are billed for tokens already generated).
- **Can the user "continue" after stopping?** Affects whether we store partial state and context for continuation.
- **Are there side effects — tool calls, writes, emails sent during an agent run?** Cancelling mid-run can leave partial side effects; need idempotency/compensation, and the UI must tell the truth about what already happened.
- **What's the upstream provider's cancellation behavior — does closing the stream stop generation and billing?** Some providers bill for tokens generated until cancellation; some require explicit cancel.
- **Does the client need to cancel across tabs/devices?** Requires server-side job state, not just a local controller.

## Approach & Trade-offs

**Cancellation is a chain; every link must be cut.** Clicking Stop should: (1) stop the UI from updating, (2) close the network connection, (3) make the server stop reading from the model, and (4) make the model provider stop generating. A frontend that only does (1) gives the illusion of stopping while tokens (and cost) keep flowing server-side.

**Client side: AbortController.** Pass `signal` to `fetch`; `abort()` rejects the pending `read()` with `AbortError`, and tears down the connection. Tie the controller's lifetime to the request, not the component: abort when (a) user clicks Stop, (b) a new request supersedes it, (c) the component/route unmounts, (d) a timeout elapses. Keep one controller reference per active generation so a stale callback can't write into the wrong message.

**Treat abort as a first-class outcome, not an error.** `AbortError` shouldn't trigger error toasts or retry logic. Model the state explicitly (`streaming → aborted`). Keep what was generated; mark the message "Stopped"; offer Continue/Regenerate. A clean abort also needs the UI to be consistent *immediately* (optimistic): flip to `aborted` on click, don't wait for the network to confirm.

**Server side: propagate disconnects.** On connection close (`req.on('close')` / `AbortSignal` from the framework), abort the upstream provider call using its own signal; stop tool loops; flush/persist partial output if product wants it. Verify with a test: after client abort, upstream request count/tokens stop increasing. Watch intermediaries — a proxy that buffers or keeps upstream connections open can swallow the disconnect.

**Detached/long-running jobs.** If generation continues independent of the HTTP connection (queue workers, agent runs lasting minutes, resumable streams), closing the connection can't cancel it. Use `POST /runs` → `runId`; stream via `GET /runs/:id/events`; cancel via `POST /runs/:id/cancel`; reflect status (`running → cancelling → cancelled`) because cancellation is asynchronous — the worker must reach a safe point. Idempotent cancel endpoint; the UI shows "Stopping…" until confirmed.

**Side effects.** For agent runs with tool calls, "cancel" can't un-send an email. Design for cooperative cancellation at step boundaries, show the user what already completed, make tools idempotent or reversible where possible, and for destructive actions require confirmation steps that cancellation naturally precedes.

**Race conditions around cancel.** Stop vs. natural completion can race: the stream may finish just as the user clicks. The final state should be deterministic: if `complete` arrived first, ignore abort; if abort first, ignore late tokens. Guard late events with the request/run ID so tokens from a cancelled generation never append to a newer message.

**Trade-off: immediate UI feedback vs. truthfulness.** Showing "Stopped" instantly is good UX, but for detached jobs or side-effectful runs it can lie. Show "Stopping…" until the server acknowledges when correctness matters.

## Solution

### Client: lifecycle-complete cancellation

```tsx
function useGeneration() {
  const [state, setState] = useState<{ id: string | null; status: Status; text: string }>(
    { id: null, status: 'idle', text: '' });
  const ctrlRef = useRef<AbortController | null>(null);

  const cancel = useCallback((reason: 'user' | 'superseded' | 'unmount') => {
    ctrlRef.current?.abort(reason);          // reason is available as signal.reason
    ctrlRef.current = null;
  }, []);

  const start = useCallback(async (prompt: string) => {
    cancel('superseded');                    // at most one active generation
    const ctrl = new AbortController();
    ctrlRef.current = ctrl;
    const id = crypto.randomUUID();
    setState({ id, status: 'streaming', text: '' });

    try {
      for await (const ev of streamChat({ prompt }, ctrl.signal)) {
        if (ctrlRef.current !== ctrl) return;                 // stale: a newer request owns the UI
        if (ev.type === 'token') setState(s => s.id === id ? { ...s, text: s.text + ev.text } : s);
      }
      setState(s => s.id === id ? { ...s, status: 'complete' } : s);
    } catch (e) {
      if (ctrlRef.current !== ctrl && ctrl.signal.reason !== 'user') return;  // superseded/unmounted: no UI write
      const aborted = (e as Error).name === 'AbortError';
      setState(s => s.id === id ? { ...s, status: aborted ? 'aborted' : 'error' } : s);
    }
  }, [cancel]);

  useEffect(() => () => cancel('unmount'), [cancel]);
  return { ...state, start, stop: () => cancel('user') };
}
```

### Server: propagate the disconnect to the model

```ts
app.post('/api/chat', async (req, res) => {
  const upstream = new AbortController();
  res.on('close', () => { if (!res.writableEnded) upstream.abort(); });   // client went away

  const stream = await provider.stream({ messages: req.body.messages, signal: upstream.signal });
  res.setHeader('Content-Type', 'text/event-stream');
  try {
    for await (const chunk of stream) res.write(`data: ${JSON.stringify(chunk)}\n\n`);
  } catch (e) {
    if (!upstream.signal.aborted) throw e;      // aborted is expected, not an error
  } finally {
    await persistPartial(req.user.id, /* what we have */);   // if product wants it
    res.end();
  }
});
```

### Detached job variant

```ts
// start
const { runId } = await api.post('/runs', { prompt });
const events = new EventSource(`/runs/${runId}/events`);

// cancel — asynchronous, so reflect "cancelling"
setStatus('cancelling');
await api.post(`/runs/${runId}/cancel`);       // idempotent
// server pushes {type:'status', value:'cancelled'} → setStatus('cancelled')
```

### Tests

```ts
it('stop aborts the request and keeps partial text, no error toast', ...)
it('sending a new prompt aborts the previous stream; late tokens never append', ...)
it('unmount aborts the request', ...)
it('server cancels upstream when client disconnects', ...)   // integration test with fake provider
```

> **Check yourself:** Name the four links in the cancellation chain, and explain why a detached background job needs a different mechanism than `AbortController`.

## Gotchas

- **UI-only cancel.** Hiding the output while the server keeps generating wastes money and resources.
- **Surfacing `AbortError` as a failure.** Misleading toasts and retry loops.
- **Late events from a cancelled stream.** Without a request/run ID check, tokens leak into the next message.
- **Forgetting unmount/route-change cleanup.** Orphaned streams keep updating dead components or running in the background.
- **Not releasing the reader/body.** Leaks connections; abort or `reader.cancel()` and `releaseLock()`.
- **Assuming closing the socket cancels the job.** Not true for queued/detached work.
- **Proxies swallowing disconnects.** The upstream keeps running because the intermediary didn't propagate close.
- **Ignoring side effects.** Agent steps already executed can't be undone; inform the user.
- **Cancel/complete races.** Make the terminal state deterministic.
- **Re-enabling the input before abort settles** can start a second stream while the first is still draining.

## Follow-up Questions

**Q (High): What does "cancel" have to do beyond calling `abort()` on the client?**

Answer: The server must detect the disconnect and abort the upstream model/tool work; otherwise generation and billing continue invisibly. Plus: clean UI state (`aborted`, partial text preserved), no stale writes, and persisted partials if desired. For detached jobs, an explicit cancel API with an asynchronous status.

The trap: stopping at `controller.abort()` and calling it done.

**Q (High): How do you avoid late tokens from a cancelled request appearing in the next response?**

Answer: Tie every event to a request/run ID or controller identity and ignore events whose owner isn't the current one; abort the previous controller before starting a new one; scope state updates by message ID. The check-before-write pattern makes ordering irrelevant.

The trap: relying on abort timing alone — a buffered chunk can still be processed after abort.

**Q (Medium): How should the UI treat an aborted response?**

Answer: As a normal terminal state: keep the partial text, label it stopped, re-enable input, and offer Continue or Regenerate. No error styling or retry-on-failure behavior, since the user initiated it. For detached jobs show "Stopping…" until the server confirms.

The trap: wiping the message or showing an error.

**Q (Medium): What about agent runs with side effects?**

Answer: Cancellation is cooperative at step boundaries; completed tool calls aren't undone. Surface exactly what ran, design tools to be idempotent or compensatable, gate irreversible actions behind explicit confirmations, and make the run record the source of truth so the UI reflects reality after cancel.

The trap: implying Stop is an undo button.

**Q (Low): How do you test cancellation?**

Answer: Unit-test the client with a controllable fake stream and assert state transitions, no error UI, and reader release; integration-test the server with a fake provider that records whether its signal was aborted after the client disconnects; E2E test Stop while streaming and assert the network request is cancelled.

The trap: only testing the happy-path stream.

## Self-Assessment

- [ ] Can list every link in the cancellation chain
- [ ] Can write the client lifecycle with abort on stop/unmount/supersede
- [ ] Can guard against late events from cancelled requests
- [ ] Can model `aborted` as a first-class state with partial text kept
- [ ] Can explain server-side disconnect propagation and detached-job cancel APIs
- [ ] Can describe cancel/complete race handling and side-effect honesty

---
*Next: Designing UI for Agentic / Tool-use Flows — extends from one streamed answer to multi-step runs where the model acts, and the UI must show and gate what it does.*
