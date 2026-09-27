# Design a Resumable, Chunked File Uploader

## Quick Reference

| Decision Point | Choice | Why |
|---|---|---|
| Transfer unit | Fixed-size chunks (e.g., 5–10MB) uploaded independently, not one streamed request for the whole file | A single multi-GB request that fails at 95% means restarting from zero; independent chunks mean only the failed chunk is retried |
| Resumability across page reload/browser close | Server-side upload session with a chunk-completion manifest, checked against a durable client-side record of the file (via a content hash / fingerprint) on return | An in-memory-only record of "which chunks are done" is lost on refresh; the client must be able to ask the server "what do you already have for this exact file" and resume from there, not just from its own memory |
| Identifying "this is the same file" after interruption | Content-based fingerprint (partial hash of the file, e.g., first+last N bytes + size, or a full hash for smaller files) rather than filename/size alone | Filename and size alone falsely match different files with the same name/size, and falsely mismatch the same file re-selected with a trivially different name |
| Concurrency | Multiple chunks in flight in parallel (bounded pool, same pattern as [Concurrency-limited Task Queue](../phase-01-js-machine-coding-async/12-concurrency-limited-task-queue-promise-pool.md)), not strictly sequential | Sequential chunk upload leaves bandwidth on the table on a fast connection; a bounded pool saturates available bandwidth without overwhelming the browser or server |
| Failure handling per chunk | Per-chunk retry with exponential backoff ([Retry With Exponential Backoff](../phase-01-js-machine-coding-async/10-retry-with-exponential-backoff.md)), independent of other in-flight chunks | One chunk hitting a transient network blip shouldn't abort the whole upload or block unrelated chunks that are succeeding |
| Large-file hashing cost | Hash incrementally / in a Web Worker, or hash only a sparse sample for very large files, rather than blocking the main thread on a full synchronous hash | Hashing a multi-GB file synchronously on the main thread would freeze the UI for a potentially long, user-visible stretch |

## The Scenario

"Design a file upload feature that needs to handle large files reliably — think a video upload to a platform like YouTube, or a large dataset upload to a cloud storage product. The connection may drop mid-upload, the user might close the tab and come back later, and it needs to resume from where it left off rather than starting over. Walk me through how you'd design this, client-side and the parts of the server contract you'd need."

## Clarifying Questions

- **What's the realistic file size range and expected network conditions?** A design for occasional 50MB uploads on reliable office wifi looks different from one for multi-gigabyte video files routinely uploaded from unreliable mobile connections — the latter is where chunking, aggressive resumability, and careful retry/backoff tuning stop being nice-to-haves and become the entire point of the design.
- **Does resumability need to survive a full page reload / browser restart / days-later return, or only a transient network drop within the same page session?** This is the single biggest fork in the design: surviving only an in-session network blip can be handled with pure client-side in-memory retry state; surviving a reload, tab close, or the user returning tomorrow to finish an upload requires a durable, server-queryable record of upload progress that the client can reconcile against on return, which is a meaningfully larger scope.
- **Is there a server-side API already decided (e.g., a provider like a resumable-upload-capable cloud storage API), or is the chunk/session protocol itself part of what's being designed here?** Some real-world implementations sit entirely on top of a provider's existing resumable-upload protocol (which handles session creation, chunk offsets, and completion server-side); if that's the case, the interesting client-side design is orchestration, hashing, and UX, not protocol design — worth clarifying which layer the question is actually probing.
- **Can multiple chunks be uploaded in parallel, or does the (hypothetical) server require strictly sequential, ordered chunk delivery?** Some resumable-upload server implementations require chunks to arrive in order (each request specifies a byte offset that must match server-side expected state); others accept out-of-order chunks identified independently by index. This materially changes whether the client can parallelize for throughput or must serialize.
- **What happens if the user selects a different file for upload while a previous upload for a different file is still resumable/paused?** Determines whether the design needs to support multiple concurrent/paused upload sessions being tracked simultaneously, versus a simpler single-upload-at-a-time assumption.
- **Does the UI need per-chunk progress granularity, or is an overall percentage sufficient?** Affects whether progress state needs to be tracked and aggregated per-chunk (needed for accurate resumed-upload percentage, since resuming at 60% needs to reflect that immediately, not restart the progress bar from zero) or can be a single coarse value.

## Approach & Trade-offs

**Chunking the file client-side is the foundational decision, and the chunk size itself is a real trade-off, not an arbitrary constant.** Using the [Blob.slice()](https://developer.mozilla.org/en-US/docs/Web/API/Blob/slice) API, the file is divided into fixed-size pieces (commonly 5–10MB) before any network activity begins. Smaller chunks mean more requests (more HTTP overhead, more round trips) but a smaller unit of retry-on-failure and finer-grained resumability (losing progress mid-chunk on a drop loses at most one chunk's worth of work); larger chunks mean fewer requests and less overhead but a larger unit of loss on failure and coarser resumability. The right default depends on expected network reliability — flakier expected conditions (mobile networks) favor smaller chunks; more reliable conditions (stable broadband) can use larger ones to reduce overhead. This is the same fundamental "unit of retry" trade-off underlying the chunk/checkpoint size chosen in [Retry With Exponential Backoff](../phase-01-js-machine-coding-async/10-retry-with-exponential-backoff.md), just applied to bytes-of-a-file rather than a whole request.

**Surviving a page reload or tab close requires a content-based file fingerprint, not reliance on the browser tab's memory.** If the client's only record of "which chunks are already uploaded" lives in a React state variable or a closure, it's gone the instant the tab closes — resuming after a reload requires the client to be able to re-identify "this is the same file I was uploading before" purely from re-selecting the file (or drag-dropping it again), and then ask the server what it already has for that identity. A fingerprint derived from the file's actual content (not just its name and size, which can coincidentally match different files, or mismatch the same file trivially renamed) is what makes this possible — practically, this is often a hash of the full file for smaller files, or a cheaper partial hash (first N bytes + last N bytes + total size) for very large files where a full hash would itself be a slow, blocking operation worth avoiding.

**Hashing a large file must not block the main thread, which pushes the fingerprinting step into a Web Worker (or an incremental, chunked hash computation yielding back to the event loop between pieces).** A naive `crypto.subtle.digest` call on a multi-gigabyte `File` object, awaited synchronously on the main thread's perspective, can take a genuinely long time — during which the UI thread is otherwise free (the crypto API itself is async/non-blocking at the Promise level), but a poorly structured surrounding implementation (e.g., manually chunking and hashing in a tight synchronous loop without yielding) can still produce jank. The safer, more scalable pattern computes the fingerprint in a Web Worker, keeping the main thread free to render upload-preparation UI (or, for very large files, sampling only a sparse portion of the file for the fingerprint rather than the entire content, accepting a theoretically higher but practically negligible collision rate in exchange for speed).

**Chunks should upload with bounded parallelism, using the same concurrency-limited pool pattern as the dedicated Promise Pool scenario.** Uploading all chunks with unlimited concurrency risks overwhelming the browser's per-origin connection limit and the server; uploading strictly one at a time leaves available bandwidth unused on a fast connection. A bounded pool (e.g., 3–4 concurrent chunk uploads) is the standard middle ground, directly reusing the mechanism built in [Concurrency-limited Task Queue (Promise Pool)](../phase-01-js-machine-coding-async/12-concurrency-limited-task-queue-promise-pool.md) — each chunk upload is a task submitted to the pool, and the pool's existing retry-per-task logic composes naturally with per-chunk exponential backoff.

**Each chunk needs independent retry with exponential backoff, isolated from the state of other in-flight or already-succeeded chunks.** A transient failure on one chunk (a dropped connection, a 5xx from the server) shouldn't abort chunks that are concurrently succeeding, and shouldn't restart the whole upload — it should retry just that chunk, with backoff, up to some bounded attempt count, while unrelated chunks proceed unaffected. This mirrors the retry scenario's core lesson almost exactly, scoped down to a single chunk as the unit of failure rather than a single request to an otherwise-monolithic endpoint.

**Resuming needs a server round trip to reconcile "what does the server already have," not blind trust in the client's last-known local state.** On returning to an interrupted upload (whether from a reload or a fresh session entirely), the client shouldn't assume its last-recorded progress is still accurate — the server is the source of truth for what was actually durably received (a chunk the client believes succeeded might not have been fully persisted server-side, e.g., if the connection dropped after the client's local state was optimistically updated but before the server's ack was received). The correct resume flow: recompute the file's fingerprint, ask the server "for this file's identity, which chunks do you already have," and only upload whichever chunks the server confirms are still missing — treating the server's chunk-manifest as authoritative, with the client's own prior local progress record as, at best, an optimization hint for showing a progress bar instantly rather than a source of truth for what to actually (re-)send.

## Solution

**Fingerprinting the file (main-thread-safe partial hash for large files, offloadable to a worker):**

```ts
async function computeFileFingerprint(file: File): Promise<string> {
  const SAMPLE_SIZE = 2 * 1024 * 1024; // 2MB from each end — cheap, good-enough uniqueness for resumability
  const headBlob = file.slice(0, SAMPLE_SIZE);
  const tailBlob = file.slice(Math.max(0, file.size - SAMPLE_SIZE), file.size);

  const [headBuf, tailBuf] = await Promise.all([
    headBlob.arrayBuffer(),
    tailBlob.arrayBuffer(),
  ]);

  const combined = new Uint8Array(headBuf.byteLength + tailBuf.byteLength + 8);
  combined.set(new Uint8Array(headBuf), 0);
  combined.set(new Uint8Array(tailBuf), headBuf.byteLength);
  new DataView(combined.buffer).setFloat64(combined.length - 8, file.size);

  const digest = await crypto.subtle.digest('SHA-256', combined);
  return Array.from(new Uint8Array(digest)).map((b) => b.toString(16).padStart(2, '0')).join('');
}
```

**Chunking and orchestrating the upload with bounded concurrency, per-chunk retry, and resume-aware skipping:**

```ts
interface ChunkUploadState {
  index: number;
  start: number;
  end: number;
  status: 'pending' | 'uploading' | 'done' | 'failed';
}

async function uploadFile(file: File, chunkSizeBytes = 8 * 1024 * 1024, maxConcurrent = 4) {
  const fingerprint = await computeFileFingerprint(file);
  const { uploadId, completedChunkIndexes } = await startOrResumeSession(fingerprint, file.size, file.name);

  const totalChunks = Math.ceil(file.size / chunkSizeBytes);
  const chunks: ChunkUploadState[] = Array.from({ length: totalChunks }, (_, i) => ({
    index: i,
    start: i * chunkSizeBytes,
    end: Math.min((i + 1) * chunkSizeBytes, file.size),
    status: completedChunkIndexes.has(i) ? 'done' : 'pending', // server's manifest is authoritative
  }));

  await runWithConcurrencyLimit(
    chunks.filter((c) => c.status !== 'done'),
    maxConcurrent,
    (chunk) => uploadChunkWithRetry(uploadId, file, chunk),
  );

  return finalizeUpload(uploadId);
}

async function uploadChunkWithRetry(uploadId: string, file: File, chunk: ChunkUploadState, maxAttempts = 5) {
  const blob = file.slice(chunk.start, chunk.end);
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    try {
      await uploadChunk(uploadId, chunk.index, blob);
      chunk.status = 'done';
      return;
    } catch (err) {
      if (attempt === maxAttempts - 1) { chunk.status = 'failed'; throw err; }
      await sleep(Math.min(2 ** attempt * 500, 15_000) + Math.random() * 300); // capped exponential backoff + jitter
    }
  }
}
```

**Resume reconciliation on session start — the server's chunk manifest, not client memory, decides what still needs sending:**

```ts
async function startOrResumeSession(fingerprint: string, size: number, name: string) {
  const res = await fetch('/api/uploads/session', {
    method: 'POST',
    body: JSON.stringify({ fingerprint, size, name }),
  });
  const { uploadId, completedChunkIndexes } = await res.json();
  // The server either finds an existing in-progress session matching this fingerprint (a real resume)
  // or creates a fresh one (completedChunkIndexes comes back empty) — the client doesn't need to know which.
  return { uploadId, completedChunkIndexes: new Set<number>(completedChunkIndexes) };
}
```

> **Check yourself:** Without looking above, explain why the client should treat the server's returned `completedChunkIndexes` as authoritative even when its own local progress record (e.g., in `localStorage`) disagrees, and construct a concrete scenario where trusting local state instead would silently corrupt the uploaded file.

## Server Contract & Data Model

```ts
interface UploadSession {
  uploadId: string;
  fingerprint: string;      // ties this session to a specific file's content
  totalSize: number;
  chunkSize: number;
  completedChunks: Set<number>; // authoritative — durably persisted server-side, not inferred from client claims
  createdAt: string;
  status: 'in_progress' | 'finalizing' | 'complete' | 'expired';
}
```

The server needs to durably persist each chunk's bytes (or a running assembly of them) as it's received, and mark that chunk index complete only after the bytes are actually safely stored — not on merely receiving the request, since a crash between receiving and persisting must NOT be recorded as a completed chunk, or a resume will incorrectly skip re-sending data that was never actually durably saved. Sessions should have an expiry (e.g., 24–48 hours of inactivity) after which partial uploads are garbage-collected, both to bound storage cost and because indefinitely resumable half-uploaded files are rarely a real product requirement.

## Solution — Progress & Pause/Resume UX

```tsx
function useResumableUpload(file: File | null) {
  const [progress, setProgress] = useState(0); // 0–100, aggregated across chunks
  const [status, setStatus] = useState<'idle' | 'uploading' | 'paused' | 'done' | 'error'>('idle');
  const controllerRef = useRef<AbortController | null>(null);

  function start() {
    if (!file) return;
    controllerRef.current = new AbortController();
    setStatus('uploading');
    uploadFile(file, /* ...pass controllerRef.current.signal through to per-chunk fetches... */)
      .then(() => setStatus('done'))
      .catch((err) => { if (err.name !== 'AbortError') setStatus('error'); });
  }

  function pause() {
    controllerRef.current?.abort(); // in-flight chunk requests abort; ALREADY-DONE chunks remain recorded server-side
    setStatus('paused');
  }

  function resume() { start(); } // re-entering startOrResumeSession naturally skips whatever's already done

  return { progress, status, start, pause, resume };
}
```

Pausing is implemented as aborting in-flight requests, not as discarding any progress — because completed-chunk state lives server-side, resuming is simply re-running the same orchestration function, which immediately skips every chunk the server already confirms, so the progress bar picks back up at its prior percentage rather than restarting.

## Gotchas

**Treating the client's local progress record as authoritative on resume, rather than reconciling against the server's manifest.** If a chunk upload's request succeeded server-side but the client's success handler never ran (e.g., the tab closed between the server persisting the chunk and the client processing the response), the client's local state incorrectly believes that chunk is still pending — re-uploading it is wasteful but harmless; the dangerous direction is the reverse (client believes a chunk succeeded when the server never durably received it), which is why the server, never client-side state, must be the source of truth queried on every resume.

**Using filename + size alone as the "is this the same file" resume key.** Trivially defeated by a user re-selecting a differently-named copy of the same file, or falsely matched by two coincidentally-same-sized different files — a content-based fingerprint is the only reliable identity check.

**Blocking the main thread computing a full cryptographic hash of a multi-gigabyte file synchronously.** Produces a real, user-visible freeze before the upload even begins — offload to a Worker or use a cheaper partial-sample fingerprint for very large files.

**Marking a chunk "complete" server-side on request receipt rather than after durable persistence.** A crash in the narrow window between "received the bytes" and "wrote them to durable storage" that still records the chunk as complete produces a resumed upload that's missing data at finalization time — completion must be recorded only after the write is actually durable.

**No session expiry, leaving abandoned partial uploads accumulating indefinitely server-side.** A meaningful fraction of started uploads are abandoned entirely (tab closed and never returned to) — without a TTL and cleanup job, this is unbounded storage growth for data that will never be finalized.

**Assuming chunks can always be uploaded in parallel without checking the actual server protocol.** Some resumable-upload implementations (including some real third-party ones) require strictly sequential, offset-verified chunk delivery — parallelizing against such a server produces protocol errors, not a faster upload; this must be confirmed against the actual contract, not assumed.

## Follow-up Questions

**Q (High): Walk through exactly what happens, step by step, when a user's laptop goes to sleep mid-upload and they resume three hours later on a different wifi network.**

Answer: On waking and returning to the tab (or re-opening it, if it was closed), the client re-selects or re-references the same `File` object (if the tab/page state survived sleep) or the user re-selects the file (if the page was reloaded) — either way, the client recomputes the file's content fingerprint, which is identical regardless of elapsed time or network change since it's derived purely from file content. It then calls the session-start/resume endpoint with that fingerprint; the server looks up any existing in-progress session matching it, finds the one from three hours ago (assuming it hasn't expired), and returns its `completedChunkIndexes`. The client constructs its chunk list, marks every server-confirmed-complete index as `done` immediately (so the progress bar reflects the prior 60%, say, instantly rather than restarting from zero), and begins uploading only the remaining pending chunks through the same bounded-concurrency pool — entirely unaffected by the network change, since each chunk upload is just an independent HTTP request against whatever network path is currently available.

The trap: describing this as needing any special "reconnect" or "resync" logic distinct from a normal fresh upload start — the elegant property of this design is that resuming and starting fresh are literally the same code path (`startOrResumeSession` followed by uploading whatever's pending); there's no special-cased "resume mode," which is itself worth stating explicitly as a design strength.

---

**Q (High): Why does chunk completion need to be recorded only after durable persistence server-side, and what's a concrete failure mode if that ordering is wrong?**

Answer: If the server marks a chunk index complete immediately upon receiving the request body (before the bytes are actually written to durable storage — disk, object storage, wherever the assembly ultimately lives), a crash or process restart occurring in the gap between "marked complete" and "actually persisted" leaves the manifest claiming a chunk is done when its data doesn't actually exist anywhere durable. On a later resume, the client asks for the manifest, is told that chunk is complete, and correctly skips re-uploading it — but at finalization time, assembling the full file from "completed" chunks encounters a gap where that chunk's data simply isn't there, either failing finalization outright or, worse, silently producing a corrupted/incomplete final file if the assembly step doesn't validate chunk presence carefully. The fix is ordering the write-then-mark-complete operations correctly (or making them transactional/atomic) so "complete" in the manifest is only ever true when the bytes are genuinely durably present.

The trap: treating this as a minor implementation detail rather than a correctness-critical ordering constraint — a candidate who doesn't flag this specific ordering hazard hasn't fully thought through what "the server's manifest is authoritative" actually requires to be true in practice.

---

**Q (High): How would you choose the chunk size, and would you use the same chunk size for a user on a fast fiber connection as for one on a spotty mobile connection?**

Answer: Chunk size is fundamentally a trade-off between per-request overhead (favoring larger chunks — fewer requests, less HTTP/TLS handshake and header overhead as a proportion of data transferred) and the cost of retrying a failed unit (favoring smaller chunks — a failure loses less progress, and resuming has finer granularity). A reasonable static default (e.g., 5–10MB) works acceptably across most conditions, but a more sophisticated design can adapt chunk size to observed conditions — e.g., starting with a moderate default and shrinking it if repeated chunk failures/retries are observed (signaling a poor connection where smaller units of retry pay off), or growing it if chunks are consistently succeeding quickly (signaling headroom to reduce request-count overhead). This is directly analogous to adaptive strategies seen in [Retry With Exponential Backoff](../phase-01-js-machine-coding-async/10-retry-with-exponential-backoff.md) — reacting to observed failure/success patterns rather than committing to one static assumption regardless of conditions.

The trap: proposing a single fixed chunk size with no acknowledgment that it's a genuine trade-off with a "it depends on conditions" answer, or over-engineering fully adaptive per-connection chunk sizing as the default answer without first establishing that a reasonable static default is a perfectly legitimate starting point for most implementations.

---

**Q (Medium): The interviewer asks: "What if the same user opens the upload page in two different browser tabs and selects the same file in both — what happens?"**

Answer: Both tabs compute the identical content fingerprint for the same file and both call the session-start endpoint with it — the server should recognize this as the same logical upload (matching fingerprint) and return the *same* `uploadId` and manifest to both tabs, rather than creating two independent competing sessions for what is, from the server's perspective, one upload target. Both tabs then proceed to upload from the same shared pending-chunk pool — which does mean some wasted duplicate work is possible if both tabs happen to pick up the same not-yet-done chunk index before either's request completes (a race the server can resolve simply by having a chunk-completion write be idempotent — re-receiving and re-persisting a chunk index that's already complete is harmless, just wasted bandwidth), but no correctness issue results, and the upload correctly completes once all chunks are done regardless of which tab happened to upload which.

The trap: assuming this scenario requires explicit cross-tab coordination (e.g., BroadcastChannel-based locking to prevent both tabs from uploading) — while that's a valid optimization to avoid wasted duplicate bandwidth, it's not required for *correctness*, since idempotent chunk persistence on the server already prevents any actual corruption or double-counting; a candidate should recognize the difference between an efficiency nicety and a correctness requirement here.

---

**Q (Medium): How would you show upload progress that's accurate immediately on a resumed upload, rather than starting the progress bar back at 0%?**

Answer: Progress should be computed as `(completed chunks' total bytes) / (total file size)`, and critically, the "completed chunks" figure must be seeded from the server's returned manifest *before* any new chunk uploads begin — so a resumed upload that's server-confirmed 60% complete renders the progress bar at 60% immediately upon the resume handshake completing, not at 0% climbing back up. This requires the progress-tracking state to be initialized from the manifest response, not solely incremented as new chunk-upload promises resolve during the current session.

The trap: initializing progress state to 0 and only incrementing it as new chunk uploads complete *in the current session* — this produces a confusing, regressive-looking UX where a 60%-done upload appears to restart from zero and race back up to where it already was, even though no actual re-upload of already-done chunks occurs.

---

**Q (Low): Would you use the same design (client-side chunking, custom session/manifest protocol) if uploading directly to a cloud storage provider that already offers a resumable upload API (e.g., a signed resumable session URL)?**

Answer: Not the full custom protocol — if the storage layer already provides resumable-upload semantics (session creation, chunked PUT with byte-range headers, built-in server-side manifest tracking), the client-side design should be a thinner orchestration layer on top of that existing protocol rather than reinventing chunk-manifest tracking from scratch: the client still needs to chunk the file, handle retry/backoff per chunk, track progress, and handle pause/resume UX, but the "does the server already have this chunk" reconciliation is handled by the provider's existing resumable-upload session mechanics rather than a bespoke `uploadId`/`completedChunkIndexes` contract built in-house. Recognizing when to build a custom protocol (no existing provider-level primitive available) versus orchestrate on top of one already provided is itself a relevant judgment call.

The trap: designing a fully custom server-side manifest protocol as if greenfield, without first asking whether the actual storage target already provides resumable-upload primitives that should be used directly — reinventing an existing, well-tested protocol is real, avoidable engineering cost.

---

## Self-Assessment

- [ ] Can explain why chunking is the foundational decision and articulate the chunk-size trade-off (overhead vs. retry granularity) in one breath
- [ ] Can design a content-based file fingerprint and explain why filename+size is insufficient for resume identity
- [ ] Can explain why the server's chunk manifest, not client-local state, must be authoritative on resume — and construct the corruption scenario that results from getting this backwards
- [ ] Can design bounded-concurrency chunk upload with independent per-chunk retry/backoff, connecting it back to the Promise Pool and Retry scenarios
- [ ] Can explain why chunk-completion must be recorded only after durable persistence, with a concrete failure mode if the ordering is wrong
- [ ] Can design progress/pause/resume UX that reflects resumed progress immediately rather than restarting the bar at zero

---
*Next: Design a Configurable Dashboard With Widgets — shifts from a single-purpose transfer problem to a composition problem: an extensible grid of independently data-fetching, user-arrangeable widgets, where the central new concerns become layout persistence, per-widget data isolation, and render performance as widget count grows.*
