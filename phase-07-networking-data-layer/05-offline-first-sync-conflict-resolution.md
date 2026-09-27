# Offline-first Sync With Conflict Resolution

## Quick Reference

| Layer | Mechanism | Purpose |
|---|---|---|
| Local persistence | IndexedDB (or similar) as the source of truth the UI reads from | UI stays fully functional with zero network, reads/writes never block on connectivity |
| Outbox / mutation queue | Every offline write recorded as a durable, ordered queue entry | Nothing is lost; writes replay in order once connectivity returns |
| Sync | Queue drains against the server on reconnect, one entry at a time (or batched) | Reconciles local state with server state without the user doing anything |
| Conflict detection | Version/timestamp per record, compared on sync | Distinguishes "no conflict, just apply" from "someone else changed this too" |
| Conflict resolution | Last-write-wins, field-level merge, or user-prompted resolution | Different data needs different correctness guarantees — one strategy doesn't fit all fields |

## The Scenario

"We want this note-taking app to work fully offline — someone should be able to open the app on a plane, create notes, edit existing ones, delete a few, and have all of it show up correctly once they land and reconnect. The tricky part: they might have also edited the same note from their phone while the laptop was offline. Design the offline-first architecture, including how you detect and resolve conflicts when the same record was changed in two places."

## Clarifying Questions

- **What granularity of conflict actually matters — is a "note" a single opaque blob (title + body as one field), or does it have several independently-editable fields (title, body, tags, a pinned flag)?** This changes what "conflict" even means: if title and body are one field, any two edits to the same note while both were offline are unavoidably a whole-note conflict; if they're separate fields, editing the title on the phone and the body on the laptop while both were offline aren't actually conflicting changes at all and can both be kept — field-level granularity meaningfully reduces how often a real conflict occurs.
- **Does the product want automatic conflict resolution (silently pick a winner via some rule) or does it need to surface conflicts to the user for manual resolution** (a "your note was edited elsewhere — keep mine / keep theirs / merge" prompt)? Automatic resolution (e.g., last-write-wins) is simpler and invisible but can silently discard a user's real work with no recourse; manual resolution preserves both versions and lets the user decide, at the cost of a UX flow that has to exist and be understood.
- **Is "offline" here scoped to full network unavailability only, or does it also need to handle a flaky/partial connection** (requests sometimes succeeding, sometimes timing out, sometimes returning inconsistent results)? A pure offline/online binary is meaningfully simpler to reason about than a spectrum of degraded connectivity, and it's worth confirming the actual requirement before building for the harder, more general case.
- **How is "the same note" identified across devices while offline** — does a note created offline on the laptop get a client-generated ID upfront, or does it only get a real ID once synced to the server? This matters for a specific failure mode: if both the phone and the laptop create a *new* note offline and both happen to generate colliding IDs (or if IDs are only assigned server-side post-sync), sync logic needs a clear answer for "is this the same note or two different ones that happen to have the same working ID" — get this wrong and either two genuinely separate notes get merged into one, or object identity breaks across the sync boundary.
- **What's the expected offline duration** — minutes (a subway ride) or potentially days (an extended trip)? Longer offline windows increase the odds of an actual conflict (more time for the same note to be independently edited on two devices) and also raise questions about how much local storage the mutation queue and local dataset are allowed to grow to before something needs to be pruned or the user warned.

## Approach & Trade-offs

**The architecture has three genuinely separate pieces that need to be designed together, not bolted on independently: local-first persistence, a durable mutation queue (outbox), and a sync/reconciliation process — treating any one of them as "the offline fix" on its own leaves gaps.** Local persistence alone (caching data in IndexedDB so reads work offline) doesn't handle writes made while offline — those need somewhere durable to live until they can be sent. An outbox alone (queuing writes) doesn't handle what happens when the same record was *also* changed server-side (by another device) in the meantime — that's the conflict-resolution piece. All three need to agree on the same data model (how a record's identity and version are represented) to work together correctly.

**Every local mutation is recorded as an explicit, ordered, durable queue entry — not applied only to in-memory/local state and hoped to sync "eventually."** The UI optimistically applies each edit to local state immediately (so the app feels fully responsive offline — this is the actual point of "offline-first," not just "doesn't crash offline"), but *alongside* that, the same edit is appended to a persisted outbox queue (IndexedDB, not memory — a memory-only queue is lost on tab close/crash, defeating the "days of offline use" case). The queue entry needs to capture the operation itself (create/update/delete), the affected record's ID, the changed fields (not a full snapshot, if field-level conflict resolution is wanted), and the local timestamp/version the change was made against.

**Conflict detection needs a version marker per record that both client and server agree on — not just "did the timestamps differ," which is a weaker signal than it looks.** Wall-clock timestamps across devices can be skewed (a phone's clock a few minutes off from the laptop's) and comparing raw timestamps to decide "which edit is newer" can get this wrong in ways that are hard to debug. A monotonic version number per record (incremented by the server on every successful write, returned in every read) is a more reliable conflict signal: when a client tries to sync a locally-queued edit, it includes the version the edit was based on (the version last seen when this device last read/synced the record); if the server's current version for that record matches what the client expected, there's no conflict — apply and increment. If the server's version has moved past what the client expected, another write happened in between — a genuine conflict.

**Resolution strategy is a per-field, per-data-type decision, not a single global rule — and this is the part that most separates a thoughtful answer from a reflexive "last write wins."** Last-write-wins (LWW) is simple to implement and fine for data where losing a stale edit silently is an acceptable trade-off (e.g., a "last viewed" timestamp, a low-stakes preference toggle) — but for a note's actual content, silently discarding one device's edits because it happened to sync a few seconds later is a real, unrecoverable data-loss bug from the user's perspective, not a rare edge case, given the prompt explicitly describes the exact scenario (edited on two devices while one was offline) as expected, not exceptional. Field-level merging (if title and body are separate fields, and only one was edited on each device, both edits can be kept with no actual conflict) reduces how often true conflicts occur but doesn't eliminate them (both devices editing the *same* field). For genuine same-field conflicts on meaningful content, surfacing the conflict to the user (keep mine / keep theirs / view both and manually merge) is the only strategy that guarantees no silent data loss, at the cost of needing that UX flow to exist and of occasionally interrupting the user's flow to ask.

**Trade-off to state explicitly: automatic LWW is simpler and invisible but risks silent data loss; manual conflict resolution guarantees no data loss but adds UX surface and occasional friction.** For this scenario — a note-taking app, where "someone's actual written content getting silently discarded" is a severe, trust-destroying bug — I'd default to field-level auto-merge for genuinely independent field changes, and manual resolution (never silent LWW) specifically for same-field conflicts on content fields, while reserving LWW for clearly low-stakes metadata fields where losing a stale value is genuinely fine.

## Solution

**Step 1 — local-first read/write: the UI always reads from and writes to local storage first, synchronously from the UI's perspective, regardless of connectivity.**

```ts
async function updateNote(id: string, changes: Partial<Note>) {
  const current = await db.notes.get(id);
  const updated = { ...current, ...changes, localVersion: current.localVersion + 1 };
  await db.notes.put(updated); // local write — UI reflects this immediately, offline or not
  await outbox.enqueue({
    type: 'update',
    recordId: id,
    changes, // field-level diff, not a full snapshot
    baseVersion: current.serverVersion, // the version this edit assumes it's building on
    queuedAt: Date.now(),
  });
  syncManager.scheduleSync(); // no-op if offline; drains immediately if online
}
```

**Step 2 — the outbox is a durable, ordered queue, persisted independently of the in-memory app state:**

```ts
class Outbox {
  async enqueue(entry: QueuedMutation) {
    await db.outboxQueue.add(entry); // IndexedDB — survives tab close/crash
  }
  async drain(sync: (entry: QueuedMutation) => Promise<SyncResult>) {
    const entries = await db.outboxQueue.orderBy('queuedAt').toArray();
    for (const entry of entries) {
      const result = await sync(entry); // processed in order, one at a time
      if (result.status === 'applied') {
        await db.outboxQueue.delete(entry.id);
      } else if (result.status === 'conflict') {
        await this.handleConflict(entry, result); // see Step 4 — doesn't silently drop the entry
      }
      // network failure mid-drain: stop, leave remaining entries queued, retry later
    }
  }
}
```

Processing in order (not in parallel) matters specifically for the same-record case: two queued edits to the same note need to apply in the order they were made, not race each other.

**Step 3 — sync sends the base version alongside each mutation; the server detects conflicts by comparing it to current state:**

```ts
// server-side (conceptually)
async function applyMutation(entry: QueuedMutation) {
  const record = await db.notes.get(entry.recordId);
  if (record.version !== entry.baseVersion) {
    // someone else's write landed on this record since this client last saw it
    return { status: 'conflict', serverRecord: record, clientChanges: entry.changes };
  }
  const updated = { ...record, ...entry.changes, version: record.version + 1 };
  await db.notes.put(updated);
  return { status: 'applied', newVersion: updated.version };
}
```

**Step 4 — conflict handling: attempt field-level auto-merge first; fall back to surfacing a manual-resolution prompt only for genuine same-field collisions.**

```ts
async function handleConflict(entry: QueuedMutation, result: ConflictResult) {
  const clientFields = Object.keys(entry.changes);
  const serverChangedFields = diffFields(result.serverRecord, /* the version this entry.baseVersion pointed to */);
  const overlap = clientFields.filter((f) => serverChangedFields.includes(f));

  if (overlap.length === 0) {
    // no actual field-level collision — reapply this client's changes on top
    // of the server's current version and retry as a fresh mutation
    await outbox.enqueue({ ...entry, baseVersion: result.serverRecord.version });
    return;
  }
  // genuine same-field conflict — don't silently pick a winner
  await conflictStore.add({
    recordId: entry.recordId,
    field: overlap,
    mine: entry.changes,
    theirs: pick(result.serverRecord, overlap),
  });
  notifyUser(entry.recordId); // surfaced in the UI for manual resolution
}
```

**Step 5 — the UI surfaces unresolved conflicts explicitly, rather than the user discovering silently-overwritten content later:**

```tsx
function ConflictBanner({ conflict }: { conflict: Conflict }) {
  return (
    <Banner>
      This note was also edited on another device.
      <button onClick={() => resolveConflict(conflict.recordId, 'mine')}>Keep my version</button>
      <button onClick={() => resolveConflict(conflict.recordId, 'theirs')}>Keep their version</button>
      <button onClick={() => openMergeView(conflict)}>View both / merge manually</button>
    </Banner>
  );
}
```

> **Check yourself:** If the laptop edits a note's `body` field while offline, and the phone (already synced, online) edits the same note's `tags` field, walk through `handleConflict` above and explain why this does *not* surface a manual-resolution prompt to the user, even though both devices touched "the same note."

## Handling Client-generated IDs for Offline-created Records

A note created entirely offline has no server-assigned ID yet — it needs a client-generated ID (a UUID) upfront so the UI can reference it (open it, edit it further, link to it) before any sync has happened. On sync, the server accepts the client-generated ID as the canonical ID (simplest — requires the ID space to be effectively collision-free, which UUIDs satisfy in practice) rather than reassigning a new server-side ID and requiring the client to reconcile every local reference to the old temporary ID after the fact — the latter is a real source of bugs (any in-memory reference, any other queued mutation pointing at the old ID by the time the ID-swap response arrives) that a client-generated-ID-as-canonical approach avoids entirely.

## Gotchas

**Treating "last write wins by timestamp" as sufficient without accounting for clock skew across devices.** Two devices' local clocks can differ by minutes without either being obviously wrong to the user; comparing raw client-supplied timestamps to decide which edit is "newer" can pick the actually-older edit as the winner if the device that made it simply has a clock running fast — a monotonic server-assigned version number, not client wall-clock time, is what should decide ordering/conflict, with timestamps used only for display, not resolution logic.

**Auto-merging at the record level instead of the field level**, so any two edits to the same note while both were offline are treated as a full conflict even when they touched entirely different fields (title vs. tags) — this needlessly interrupts the user far more often than the data actually warrants, training them to blindly click through conflict prompts (undermining the entire point of surfacing conflicts only when they're real).

**Silently dropping a queued mutation on conflict rather than routing it into an explicit unresolved-conflict state** — if `handleConflict` (or its equivalent) just discards the client's change because the server's version won, that's exactly the silent-data-loss bug this whole design exists to prevent, just moved one layer deeper into the sync code instead of the naive top-level LWW version.

**Draining the outbox out of order or in parallel** — processing two queued mutations to the same record concurrently (rather than sequentially, in the order they were originally made) can produce a result that doesn't correspond to *either* device's intended final state, especially for non-commutative operations (e.g., "append to a list" applied out of order produces a different list than either device intended).

**Not persisting the outbox queue itself durably** — a queue held only in memory (a JS array, a non-persisted store) is lost if the tab closes, the browser crashes, or the OS kills a backgrounded tab before connectivity returns, silently losing every queued offline edit with no error and no recovery path, which is a worse failure than any conflict-resolution edge case.

**Client-generated IDs colliding, or being reassigned on sync without updating every local reference** — if the ID space isn't sufficiently collision-resistant (a short auto-incrementing local counter instead of a UUID) or the server reassigns IDs post-sync without the client updating all of its own local references (including other queued mutations that reference the old ID), object identity silently breaks — an edit queued against "note #3" ends up applying to the wrong record, or to nothing at all, after sync.

## Follow-up Questions

**Q (High): Why is a server-assigned monotonic version number a more reliable conflict signal than comparing client-reported timestamps? Give a concrete scenario where timestamp comparison gets the wrong answer.**

Answer: Client-reported timestamps depend on each device's local clock being accurate and synchronized, which isn't guaranteed — a phone's clock running 3 minutes fast relative to a laptop's is common and invisible to the user. Concretely: the laptop edits a note at (its own clock's) 2:00:00 while offline; the phone, already online, edits the same note at (its own clock's) 1:59:00 — but the phone's clock happens to be running 4 minutes fast, meaning the phone's edit actually happened *after* the laptop's edit in real wall-clock time, despite reporting an earlier timestamp. A pure timestamp-comparison LWW strategy would look at "2:00:00 > 1:59:00" and conclude the laptop's edit is newer and should win — the wrong conclusion, since the phone's edit was chronologically later. A server-assigned monotonic version number sidesteps this entirely: it's incremented by the server, in the actual order writes are received and applied server-side, with no dependency on any client's clock being correct — "the server's current version for this record has moved past what my queued edit expected" is a fact about actual write order, not a comparison of potentially-skewed timestamps.

The trap: treating "compare timestamps" as an obviously-correct way to determine recency — it's intuitive but relies on an assumption (synchronized clocks across all clients) that doesn't hold reliably in practice, and the failure mode (silently picking the wrong winner) is invisible until someone notices their genuinely-later edit was discarded.

---

**Q (High): Walk through what happens if the network connection flaps during outbox drain — connectivity returns, a few queue entries sync successfully, then it drops again mid-drain. What needs to be true about the drain process for this to be safe?**

Answer: The drain process (Step 2's `Outbox.drain`) needs to be resumable and idempotent at the level of individual queue entries, not an all-or-nothing transaction across the whole queue — since it processes entries one at a time and only removes each entry from the persisted queue *after* confirming the server successfully applied it (`result.status === 'applied'`), a mid-drain disconnection simply stops the loop with whatever entries remain still safely sitting in the durable queue, exactly as if they'd never started draining; nothing is lost, and nothing has been double-counted, because entries that did succeed were already removed, and the ones that didn't get a chance are untouched. The one subtlety worth naming: if a request was *sent* to the server and the server *did* apply it, but the response never reached the client (the connection dropped between the server processing the write and the response arriving), the client can't distinguish "the write failed" from "the write succeeded but I never heard back" — naively retrying that same entry on the next drain attempt risks applying it twice. This is where an idempotency key per mutation (a client-generated UUID per queue entry, sent with the request and checked/stored server-side) matters: the server can recognize "I've already applied a mutation with this exact idempotency key" and return the original success result again rather than re-applying the operation a second time, making retries of an already-applied-but-unconfirmed mutation safe.

The trap: assuming that because the queue entry only gets deleted on confirmed success, retries are automatically safe — that's true for the "request never reached the server" case, but not for the "server applied it, response got lost" case, which needs its own idempotency mechanism (a dedup key), not just queue-based retry logic.

---

**Q (High): Design the exact rule for when a conflict is "real" (needs manual resolution) versus "false" (can be auto-merged) for a note with `title`, `body`, and `tags` as separate fields. Where does this rule break down?**

Answer: The rule from Step 4: compute the set of fields the client's queued mutation changed, and the set of fields the server's version changed relative to the same base version the client's edit was built on; if these two sets don't intersect, it's a false conflict — both devices' changes can be reapplied together (client's changes on top of the server's current state) with no data loss on either side, since they touched disjoint fields. If the sets do intersect (both changed `body`, for instance), that's a genuine conflict on that specific field and needs manual resolution (or a domain-specific auto-merge, see below) — but only for the *overlapping* fields; any non-overlapping fields from either side can still be auto-merged as usual. This rule breaks down for fields that aren't atomically replaceable — a `tags` array is the clearest case: if the laptop adds tag "urgent" and the phone independently adds tag "personal" to the same note while both were offline, a naive field-level diff sees both as "changed `tags`" and flags a same-field conflict, even though the actual intended changes (add one tag each) are perfectly compatible and could be merged (union the two tag lists) without needing to ask the user anything. Getting this right requires per-field-type merge logic, not a uniform "did this field change" boolean — arrays/sets often support commutative merges (union, for lists of additions) that scalar fields like a title or a body's free text genuinely don't (there's no sensible automatic way to "merge" two different rewrites of the same paragraph).

The trap: applying one uniform "did the field change, yes/no" rule across every field type — `tags` as a set-like collection has real, safe auto-merge semantics (union) that a scalar text field doesn't; treating them identically either surfaces unnecessary conflict prompts for tags (a UX regression) or attempts an unsafe automatic merge for freeform text (a correctness regression).

---

**Q (Medium): The user chooses "keep mine" in the conflict-resolution UI. Walk through what needs to happen for that choice to actually stick, given the server already applied "theirs."**

Answer: "Keep mine" needs to be expressed as a *new* mutation, not a retroactive undo — the server's applied "theirs" write already incremented the record's version, so the client's stale queued mutation (still carrying the old `baseVersion`) can't simply be replayed as-is; it would immediately conflict again against the now-current server version. The correct flow: take the client's original intended changes (from the conflict record captured in Step 4 — `conflict.mine`), construct a *fresh* mutation with `baseVersion` set to the server's current version (the one that resulted from applying "theirs"), and enqueue and sync that as a new outbox entry — effectively "reapply my intended change on top of whatever's there now," rather than trying to force the original stale mutation through. This also needs to be presented to the user (or at least logically treated) as an intentional overwrite of "theirs" with "mine," which is exactly the same kind of write that could itself, in a three-way-editing scenario, race against yet another concurrent edit — worth at least acknowledging that conflict resolution isn't necessarily a terminal state if edits keep happening concurrently from multiple sources, though for the common two-device case this second-order race is rare enough not to need special-casing beyond "the resolution is itself just a normal versioned write."

The trap: treating conflict resolution as a client-only, purely local decision that doesn't need to go back through the same versioned-write/conflict-detection path as any other mutation — "keep mine" is still a write that could, in principle, itself encounter a new conflict if the situation is unlucky enough, and should be sent through the same mechanism as everything else rather than a special-cased "force overwrite" bypass.

---

**Q (Medium): Why does the outbox need to store field-level diffs (`{ body: 'new text' }`) rather than full record snapshots for each queued mutation? What would go wrong with full snapshots?**

Answer: A full-snapshot approach — each queued mutation storing the entire record as the client believed it should look after the edit — would make the false-conflict detection in Step 4 impossible to do correctly, since there'd be no way to tell, purely from two full snapshots, which specific fields either side actually *intended* to change versus which fields simply happened to be carried along unchanged in the snapshot; applying a full snapshot as "the new record state" would silently overwrite any field the *other* side changed, even a field this device's edit never touched, because the snapshot approach has no concept of "I didn't touch this field, don't overwrite it." Field-level diffs make the actually-changed fields explicit and are exactly what makes disjoint-field auto-merging (Step 4's no-overlap case) possible — the mutation says precisely "I changed `body` to X" and nothing about `tags` or `title`, so applying it on top of a server record that has since had `tags` changed by someone else doesn't clobber that unrelated change.

The trap: reaching for "just store the new record state" as the obvious/simpler design — it's simpler to implement but silently reintroduces record-level (rather than field-level) conflict granularity, which was specifically identified as a UX and correctness problem (unnecessary conflict prompts, or worse, silent overwrites of unrelated fields) earlier in this scenario.

---

**Q (Low): How would this design need to change if notes could be edited by multiple *different users* (not just multiple devices of the same user), e.g., a shared/collaborative note?**

Answer: The core sync/versioning/field-diff mechanism described above still applies — conflict detection via a server-assigned version and field-level diffing works the same regardless of whether the two conflicting edits came from the same person's two devices or two different people — but the conflict *resolution* UX changes meaningfully: "keep mine / keep theirs" makes sense when "theirs" is your own other device (you know what you did there and can make an informed choice), but is a much weaker UX when "theirs" is another person's edit you may not have context on, especially if that person isn't present to ask. At that point, either the product needs true collaborative editing semantics (operational transformation or CRDTs, letting both edits merge automatically at a much finer grain than field-level diffing, similar to what a collaborative document editor needs) rather than an offline-sync conflict-prompt model, or, if occasional conflicts between different users are rare and acceptable, the conflict UI needs to show enough context (who made the other edit, when, what it changed) for the user to make a genuinely informed choice rather than a blind "mine vs. theirs" pick. This is a meaningfully bigger scope increase than the single-user-multi-device version of this scenario — worth flagging explicitly as a different problem rather than assuming the same design scales to it unchanged.

The trap: assuming the offline-sync conflict-resolution design built for one user's multiple devices extends unchanged to multiple different users editing collaboratively — the mechanics of detecting a conflict are the same, but the UX and often the underlying merge strategy (field-level diffing vs. true operational/CRDT-based merging) need to be reconsidered for the multi-user case.

---

## Self-Assessment

- [ ] Can name and explain the three separate architectural pieces (local persistence, durable outbox, sync/conflict-resolution) and why each is necessary
- [ ] Can explain why a server-assigned version number is a more reliable conflict signal than comparing client timestamps, with a concrete clock-skew example
- [ ] Can design field-level (not record-level) conflict detection and explain why it reduces unnecessary conflict prompts
- [ ] Can explain why the outbox must be durably persisted (not memory-only) and processed in order
- [ ] Can reason through the "response lost after server applied the write" retry-safety problem and why it needs an idempotency key
- [ ] Can identify when a "conflict resolution" choice needs to be re-expressed as a new versioned mutation rather than a special-cased bypass

---
*Next: Streaming LLM Response UI: SSE vs. WebSocket — a different kind of live-connection design question, this time centered on incremental token-by-token delivery and cancellation rather than bidirectional sync.*
