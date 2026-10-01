# Reviewing a PR With a Subtle Race Condition

## Quick Reference

| Smell in a PR | Why it's a race | Review ask |
|---|---|---|
| `async` work started in an effect/handler with no cancellation | Responses arrive out of order or after unmount | Abort/ignore stale results; clean up in effect |
| Read-modify-write on state across an `await` | State changed during the await; write clobbers it | Functional updates / re-read after await / server-side atomicity |
| Check-then-act (`if (!exists) create`) | Two actors pass the check simultaneously | Make it atomic or idempotent (unique key, idempotency token) |
| Double-submit possible | Click/Enter twice before `pending` state renders | Disable synchronously via ref; idempotency key |
| Optimistic update + refetch | Refetch lands between mutation and response → flicker/revert | Cancel in-flight queries before optimistic write |

## The Scenario

"A teammate opens this PR: 'Add 'Save draft' with autosave.' It looks reasonable and tests pass. I want you to review it out loud — tell me what you'd flag, and how you'd say it."

```tsx
function DraftEditor({ draftId }: { draftId: string }) {
  const [text, setText] = useState('');
  const [saving, setSaving] = useState(false);

  useEffect(() => {
    fetch(`/api/drafts/${draftId}`).then(r => r.json()).then(d => setText(d.text));
  }, [draftId]);

  const save = async () => {
    setSaving(true);
    const res = await fetch(`/api/drafts/${draftId}`, {
      method: 'PUT', body: JSON.stringify({ text }),
    });
    const saved = await res.json();
    setText(saved.text);
    setSaving(false);
  };

  useEffect(() => {
    const t = setInterval(save, 5000);
    return () => clearInterval(t);
  }, []);

  return (<>
    <textarea value={text} onChange={e => setText(e.target.value)} />
    <button onClick={save} disabled={saving}>Save</button>
  </>);
}
```

## Clarifying Questions

- **What does the server do on concurrent writes — last-write-wins, versioned, merge?** Determines whether client-side ordering bugs become data loss.
- **Can a user have the draft open in two tabs/devices?** Introduces multi-writer conflicts beyond single-client races.
- **What's the PUT semantic — full replace? Is it idempotent?** Affects retry safety and the right fix.
- **What should the user see while saving — spinner, "Saved at …"?** Defines UI states to review against.
- **What's the failure behavior on save error — retry, banner, keep local changes?** The PR currently has none.
- **How long can a save take on slow networks?** If it can exceed the 5s interval, overlapping saves are routine, not rare.

## Approach & Trade-offs

**Review for interleavings, not for the happy path.** Tests pass because they exercise one sequence. Race conditions live in *other* orderings. For each `await`, I ask: "what else can happen while this is suspended?"

**Findings, in priority order:**

1. **Stale closure in the interval (real bug, likely visible).** `useEffect(..., [])` captures the first-render `save`, which closes over the initial `text = ''`. Every autosave PUTs an empty string and then `setText(saved.text)` wipes what the user typed. This is the highest severity: silent data loss. Fix: keep the latest text in a ref, or depend on `text` with a debounce rather than an interval.

2. **Response clobbers newer input (read-modify-write across an await).** Even with the closure fixed, `save` does `setText(saved.text)` after the response. If the user typed during the request, the server's older echo overwrites their newer keystrokes. Fix: don't overwrite local text with the response; only update metadata (`updatedAt`, version), or apply the response only if local text hasn't changed since the request began.

3. **Overlapping saves and out-of-order completion.** Interval + manual click can fire concurrent PUTs; if request 1 resolves after request 2, "saved" state and server state disagree. Fix: serialize saves (single in-flight, queue the latest), or send a version/ETag so the server rejects stale writes (`If-Match`, 409 handling).

4. **`saving` is async state, so double-submit is still possible.** `setSaving(true)` doesn't take effect until the next render; two fast clicks (or interval + click) both pass. Fix: guard with a ref (`inFlight.current`) checked synchronously.

5. **Load race on `draftId` change / unmount.** The initial `fetch` has no abort/ignore; switching drafts quickly lets draft A's response overwrite draft B's text — exactly the cross-draft bleed that causes a user to save the wrong content into the wrong draft. Also `setState` after unmount. Fix: AbortController in effect cleanup.

6. **No error handling → stuck `saving`.** If `fetch` rejects, `setSaving(false)` never runs; the button stays disabled forever. Fix: `try/finally`.

7. **Save during/after draft switch.** An in-flight save for draft A completing after switching to B calls `setText(saved.text)` with A's text into B's editor.

8. **Multi-tab/device conflict.** Last-write-wins silently discards the other session's edits. At least version-check and surface a conflict.

**How I'd communicate it.** Lead with severity and a concrete reproduction ("type 'hello', wait 5s → textarea clears"), not "this has a race." Group by must-fix (1, 2, 5, 6) vs. should-fix (3, 4, 8) vs. suggestions. Offer a suggested diff. Ask a question where I'm uncertain (server semantics). Ask for a test that fails on the buggy version. Be kind: the code reads reasonably, which is exactly why these bugs survive.

**Trade-off: how far to push in review.** I'd not demand a CRDT. A pragmatic bar: no data loss in single-tab use, a serialized save queue with version checks, and surfaced errors. Multi-writer reconciliation can be a tracked follow-up if the product needs it.

## Solution

### The fixed component

```tsx
function DraftEditor({ draftId }: { draftId: string }) {
  const [text, setText] = useState('');
  const [status, setStatus] = useState<'idle' | 'saving' | 'error'>('idle');
  const textRef = useRef(text);
  const versionRef = useRef<number | null>(null);
  const inFlight = useRef(false);
  const dirty = useRef(false);
  textRef.current = text;

  // Load: abort stale loads
  useEffect(() => {
    const ctrl = new AbortController();
    setText('');
    fetch(`/api/drafts/${draftId}`, { signal: ctrl.signal })
      .then(r => r.json())
      .then(d => { setText(d.text); versionRef.current = d.version; })
      .catch(e => { if (e.name !== 'AbortError') setStatus('error'); });
    return () => ctrl.abort();
  }, [draftId]);

  // Save: single in-flight, coalesce the rest, never overwrite local text
  const save = useCallback(async () => {
    if (inFlight.current) { dirty.current = true; return; }
    inFlight.current = true;
    setStatus('saving');
    try {
      do {
        dirty.current = false;
        const res = await fetch(`/api/drafts/${draftId}`, {
          method: 'PUT',
          headers: { 'If-Match': String(versionRef.current) },
          body: JSON.stringify({ text: textRef.current }),
        });
        if (res.status === 409) throw new ConflictError();
        const { version } = await res.json();
        versionRef.current = version;         // metadata only; don't touch text
      } while (dirty.current);
      setStatus('idle');
    } catch { setStatus('error'); }
    finally { inFlight.current = false; }
  }, [draftId]);

  // Debounced autosave on change instead of a closure-stale interval
  useEffect(() => {
    const t = setTimeout(save, 1500);
    return () => clearTimeout(t);
  }, [text, save]);

  return (<>
    <textarea value={text} onChange={e => setText(e.target.value)} />
    <button onClick={save} disabled={status === 'saving'}>Save</button>
    {status === 'error' && <p role="alert">Couldn't save. Your changes are kept — retry.</p>}
  </>);
}
```

### Regression tests the PR should include

```tsx
it('does not overwrite text typed while a save is in flight', ...)   // deferred PUT, type, resolve
it('does not let a slow load for draft A overwrite draft B', ...)    // deferred fetches, rerender
it('sends the latest text from autosave, not the initial empty string', ...)
it('recovers from a failed save and re-enables the button', ...)
```

> **Check yourself:** Which finding would you block the PR on and why, and how would you phrase it so the author understands the failure without reading your mind?

## Gotchas

- **Reviewing only the diff's logic, not the interleavings.** Races don't appear in single-sequence reading.
- **Accepting "tests pass."** Tests rarely control response ordering; ask for one that does.
- **Flagging everything with equal weight.** Authors tune out; rank severity, separate must-fix from nits.
- **Vague comments ("this might race").** Give a concrete repro and a suggested fix.
- **Over-engineering the fix.** Demanding CRDTs for a draft box derails the review; scope pragmatically.
- **Forgetting the server.** A client-only fix doesn't prevent conflicting writes; versioning/idempotency needs backend support.
- **Missing the stale-closure bug because the code "looks fine."** Effects with `[]` deps calling functions that read state are a top suspect.

## Follow-up Questions

**Q (High): What's the first thing you look for when reviewing async code for races?**

Answer: Every `await` or callback boundary, asking what state can change in between and whether the code reads state before and writes after (read-modify-write). Then: is there cancellation or staleness protection, can the operation overlap itself (double-submit/interval), and are results applied to the *current* context or the one that started the request?

The trap: scanning for syntax or style issues rather than reasoning about orderings.

**Q (High): How do you make a review comment about a race actually land?**

Answer: Provide a concrete reproduction ("type, wait 5s, text disappears"), state the impact (data loss), identify the mechanism (stale closure over `text`), suggest a fix or diff, and request a regression test that fails on the old code. Label severity and separate blockers from suggestions.

The trap: "possible race condition here" with no repro or direction.

**Q (Medium): Why isn't `disabled={saving}` enough to prevent double-submit?**

Answer: `saving` is React state; updates are batched and applied on the next render, so two events firing before re-render both read `false`. Interval-triggered saves bypass the button entirely. Use a synchronous guard (ref) and, for real safety, an idempotency key or version check server-side.

The trap: believing UI disabling is a correctness mechanism.

**Q (Medium): How do you handle conflicts when the same draft is edited in two tabs?**

Answer: Version/ETag with optimistic concurrency: server rejects stale writes with 409; the client surfaces a conflict (reload, or merge UI). For richer collaboration, OT/CRDT. Silent last-write-wins is acceptable only if product agrees data loss is tolerable.

The trap: ignoring multi-writer entirely or jumping to CRDTs unprompted.

**Q (Low): Could a library prevent most of this?**

Answer: Largely: React Query/SWR handle staleness, cancellation, and dedup for reads; mutation helpers serialize and handle optimistic rollback. They don't remove the need to reason about version conflicts or in-flight text vs. server echo, but they eliminate whole classes of load-race bugs.

The trap: "use a library" without understanding what bugs remain.

## Self-Assessment

- [ ] Can scan code and enumerate the interleavings at each await
- [ ] Can spot the stale-closure-in-`[]`-effect bug immediately
- [ ] Can explain why state-based `saving` doesn't prevent double-submit
- [ ] Can propose fixes: abort, serialize/coalesce, version checks, try/finally
- [ ] Can write review comments with repro, impact, fix, and severity
- [ ] Can name the regression tests to require

---
*Phase 12 complete. Next: Phase 13 — AI-integrated Frontend, starting with "Streaming an LLM Chat Response Into the UI," which applies the async, cancellation, and race reasoning from this phase to streaming interfaces.*
