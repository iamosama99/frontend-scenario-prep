# Retry With Exponential Backoff

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Exponential growth | Delay = `baseDelay * 2^attempt`, capped at a max | Retrying at a fixed interval hammers a struggling server at a constant rate; exponential backoff gives it increasing room to recover |
| Jitter | Add randomness to each computed delay | Without it, many clients that failed at the same moment (e.g., after a shared outage) retry in lockstep, creating synchronized request spikes — the "thundering herd" problem |
| Bounded retries | A max attempt count, with the *last* failure surfaced, not swallowed | Retrying forever on a permanently broken endpoint turns a fast failure into a slow, resource-leaking one |
| Retryability check | Distinguish retryable errors (5xx, network/timeout) from non-retryable ones (4xx, validation) | Retrying a 400 Bad Request five times wastes time and can't ever succeed — the input itself is wrong, not the network |

## The Scenario

"Write a `retry(fn, options)` utility that retries a failing async function with exponential backoff. It's going to wrap our API client calls — right now a flaky third-party endpoint fails intermittently and we just give up on the first error, which shows users an error toast for what's usually a transient blip."

## Clarifying Questions

- **Should every error be retried, or only some?** This is the most important question — retrying a `400 Bad Request` or a `401 Unauthorized` five times in a row is pure waste (the request will never succeed no matter how many times it's repeated) and can actively hurt (repeatedly hitting an endpoint with bad auth might trigger rate limiting or account lockout). I'd ask whether there's an existing convention for "retryable" (typically: network errors, timeouts, and 5xx/429 status codes) versus "non-retryable" (4xx other than 429), and default to that split unless told otherwise.
- **What's the maximum number of attempts, and the base/max delay?** These are the concrete tunables — without bounds, "retry forever" turns a real outage into an unbounded resource drain (open connections, queued requests) and a UI that hangs indefinitely instead of failing fast and telling the user something's wrong. I'd propose sensible defaults (e.g., 3 attempts, 300ms base, 5s cap) and confirm.
- **Does this need to support cancellation** (e.g., the user navigates away mid-retry, or a parent request is aborted)? If the retrying call is tied to a component's lifecycle, an in-flight retry sequence that keeps running after the caller no longer cares wastes network and can cause "setState after unmount" style bugs at the call site consuming the result. I'd wire in an `AbortSignal` if this integrates with `fetch`.
- **Should jitter be added, and does it matter here?** For a single client instance, jitter doesn't change correctness — but if this pattern gets copy-pasted across many client instances or many users hitting the same backend, un-jittered exponential backoff means every failed client retries at the *exact same intervals*, converting a transient blip into synchronized retry storms that look like a DDoS from the server's perspective. Worth mentioning even if the immediate use case is single-client.

## Approach & Trade-offs

The utility needs to wrap an arbitrary async function, so `fn` is a zero-argument function returning a promise (or the caller pre-binds arguments via a closure) — this keeps the retry logic generic rather than coupled to any particular API shape.

**Delay calculation — exponential, capped, with jitter**: The raw exponential formula is `delay = baseDelay * 2^attemptNumber`. Left uncapped, this grows without bound (attempt 10 would be minutes-to-hours), so I clamp it to a `maxDelay`. Then I add jitter — the two common strategies are "full jitter" (`random(0, computedDelay)`) and "equal jitter" (`computedDelay/2 + random(0, computedDelay/2)`). I default to full jitter since it's what AWS's own backoff guidance recommends as most effective at avoiding synchronized retries, while noting equal jitter as a middle ground if some backoff floor is desired (full jitter can occasionally produce a near-zero delay right after a failure, which is fine for most cases but worth flagging).

**Retryability as a pluggable predicate, not hardcoded logic**: Rather than hardcoding "retry on 5xx," I accept an optional `shouldRetry(error)` predicate defaulting to a reasonable implementation (network errors, timeouts, 429, 5xx) — this keeps the utility reusable across different API clients that might signal errors differently (thrown `Error` objects vs. rejected promises with a `.status` field vs. GraphQL error arrays), and lets call sites override for their specific error shape.

**Give up and surface the *last* error, not swallow it**: After exhausting `maxAttempts`, the function must reject with information about *why* it ultimately failed — specifically the last attempt's error, since that's the most relevant diagnostic (the error might have changed character across attempts, e.g., first attempt times out, later attempts get a definitive 503). Some implementations also attach the full history of errors for debugging; I'd include that as an `error.attempts` array if observability is a stated concern, but keep the default rejection value being just the final error to match what most callers expect (`catch (err) { ... }` where `err` is the meaningful one).

**Async delay via `setTimeout` wrapped in a promise**, not a busy-loop or synchronous sleep (which doesn't exist in JS without blocking the whole thread) — `await new Promise(resolve => setTimeout(resolve, delay))` is the standard idiom, and it's worth being explicit that this doesn't block the event loop; other work (UI updates, other async operations) continues normally during the wait.

I structured this as a loop (`for` over attempts) with a `try/catch` inside, rather than recursion, mainly for stack-trace clarity and to avoid any risk of deep recursion on a high `maxAttempts` — though for a small bounded attempt count either approach is fine; I'd mention this as a minor stylistic choice, not a load-bearing decision.

## Solution

```javascript
function defaultShouldRetry(error) {
  // Network-level failure (fetch throws TypeError on network errors, not on HTTP error codes)
  if (error instanceof TypeError) return true;
  // Explicit timeout signal from an AbortController-based caller
  if (error?.name === 'AbortError' && error?.reason === 'timeout') return true;
  // HTTP status-based retryability: 429 (rate limited) and 5xx (server errors) are retryable;
  // other 4xx (bad request, unauthorized, not found, etc.) are not — retrying won't fix them.
  const status = error?.status;
  if (typeof status === 'number') {
    return status === 429 || (status >= 500 && status < 600);
  }
  return false;
}

function delayWithJitter(attempt, baseDelay, maxDelay) {
  const exponential = Math.min(baseDelay * 2 ** attempt, maxDelay);
  return Math.random() * exponential; // "full jitter"
}

async function retry(fn, {
  maxAttempts = 3,
  baseDelay = 300,
  maxDelay = 5000,
  shouldRetry = defaultShouldRetry,
  signal,
} = {}) {
  let lastError;

  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    if (signal?.aborted) {
      throw new DOMException('Retry aborted', 'AbortError');
    }

    try {
      return await fn();
    } catch (error) {
      lastError = error;

      const isLastAttempt = attempt === maxAttempts - 1;
      if (isLastAttempt || !shouldRetry(error)) {
        throw error;
      }

      const delay = delayWithJitter(attempt, baseDelay, maxDelay);
      await new Promise((resolve, reject) => {
        const timer = setTimeout(resolve, delay);
        signal?.addEventListener('abort', () => {
          clearTimeout(timer);
          reject(new DOMException('Retry aborted', 'AbortError'));
        }, { once: true });
      });
    }
  }

  throw lastError; // unreachable given the loop above, but keeps control flow explicit
}
```

```javascript
const controller = new AbortController();

const data = await retry(
  () => fetch('/api/flaky-endpoint', { signal: controller.signal }).then((res) => {
    if (!res.ok) {
      const err = new Error(`HTTP ${res.status}`);
      err.status = res.status;
      throw err;
    }
    return res.json();
  }),
  { maxAttempts: 4, baseDelay: 250, signal: controller.signal }
);

// Later, if the user navigates away:
controller.abort();
```

> **Check yourself:** Why does the jitter calculation use `Math.random() * exponential` (full jitter) instead of just using `exponential` directly — what production failure mode does that randomness specifically prevent?

## Gotchas

**Retrying non-retryable errors.** The single most common mistake is retrying *every* error uniformly. A `400 Bad Request` caused by malformed input will fail identically on every retry — the utility should fail fast on these instead of wasting `maxAttempts * avgDelay` worth of time before eventually surfacing the same error it could have surfaced immediately. This is also a correctness issue, not just an efficiency one: for non-idempotent operations (e.g., "charge this card"), retrying a request that already partially succeeded on the server side but returned an ambiguous error can cause duplicate side effects.

**No jitter, causing synchronized retry storms.** If backoff delays are purely deterministic (`baseDelay * 2^attempt` with no randomness) and many clients fail around the same time — e.g., because the server itself just had a brief outage — they'll all retry at t+300ms, then all again at t+600ms, then all at t+1200ms, in lockstep. Each synchronized wave can be a bigger spike than the original load, potentially preventing the server from recovering at all. This is a real, documented failure mode (part of why AWS's architecture blog specifically recommends "full jitter").

**Unbounded retries, or no cap on the delay.** Without a `maxAttempts` ceiling, a permanently broken dependency turns into an infinite retry loop that never surfaces an error to the user or calling code — the UI just hangs. Without a `maxDelay` cap, `baseDelay * 2^attempt` grows so large after ~10 attempts that a single retry sequence could wait literal minutes, which is rarely the intended behavior for anything user-facing.

**Swallowing the original error on final failure.** Some naive implementations, after exhausting retries, throw a generic `new Error('Retry failed')` instead of the actual last underlying error — this destroys the diagnostic information a caller or error-tracking tool (Sentry, etc.) needs to understand *why* it ultimately failed (was it a 503? A timeout? A parse error?).

**Not accounting for cancellation during the backoff wait itself.** It's easy to add an `AbortSignal` check only *before* calling `fn()`, but forget that the utility can also be sitting in the `setTimeout` wait between attempts for seconds at a time — if the caller aborts during that window, the retry loop should stop immediately rather than waiting out the full delay and making one more doomed attempt.

## Follow-up Questions

**Q (High): Why is jitter necessary in addition to exponential backoff — what specific failure does exponential backoff alone not solve?**

Answer: Pure exponential backoff solves the "back off faster under sustained failure" problem (each retry waits longer, giving a struggling system more room to recover), but it does nothing about *synchronization* across multiple independent clients. If a shared dependency (a backend service, a third-party API) has a brief outage that affects many clients simultaneously, all of those clients compute the *exact same* sequence of delays from the *exact same* starting point (the moment of failure), so they all retry at t+300ms, t+900ms, t+2100ms, etc., in near-perfect lockstep. Each of those synchronized waves can hit the recovering server as a spike larger than normal peak traffic — potentially preventing recovery entirely, a phenomenon sometimes called a "retry storm" or "thundering herd." Adding randomness (jitter) to each computed delay spreads those retries out over time instead of clustering them, which is what actually lets the server recover under real multi-client conditions.

The trap: describing exponential backoff as sufficient on its own — it solves the *individual client's* politeness problem but not the *aggregate* synchronization problem, which only shows up under realistic multi-client load, not in a single-client test/demo.

---

**Q (High): How do you decide which errors should trigger a retry versus which should fail immediately?**

Answer: The dividing line is *idempotency and cause*: retry errors that are plausibly transient and where retrying has a real chance of succeeding — network failures (DNS issues, connection drops), timeouts, HTTP 429 (rate limited — the server is explicitly saying "try again, just not right now"), and 5xx server errors (the server itself is having a problem, unrelated to what was sent). Don't retry errors caused by the request itself being wrong — 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 422 Unprocessable Entity — because the exact same request will produce the exact same error on every retry; the fix is correcting the request, not waiting and resending it. A pluggable `shouldRetry(error)` predicate is the right API shape here because different backends signal these categories differently (HTTP status codes, GraphQL error extensions, custom error codes), so a hardcoded check doesn't generalize across API clients.

The trap: treating "retry on any caught error" as safe simplification — for non-idempotent operations in particular (payments, "create" endpoints without idempotency keys), blindly retrying a request that might have actually succeeded server-side but failed to report success back to the client can cause duplicate side effects, which is a correctness bug, not just wasted effort.

---

**Q (Medium): How would you make this retry utility respect an `AbortSignal` so an in-flight retry sequence can be cancelled, e.g., if the component using it unmounts?**

Answer: Two places need to check the signal: before starting each attempt (skip straight to throwing an `AbortError` if already aborted, rather than making one more doomed network call), and during the backoff wait itself (since that's often the majority of the total elapsed time) — the wait is a `setTimeout`-backed promise, so it needs an `abort` event listener that clears the pending timer and rejects immediately, rather than letting the full delay elapse before the cancellation takes effect. Additionally, the signal should typically be threaded through to `fn()` itself (e.g., passed to `fetch`'s own `signal` option) so the in-flight network request is also aborted, not just the retry scheduling around it — otherwise cancelling "the retry" still leaves a real HTTP request running in the background that the caller no longer cares about.

The trap: adding an abort check only at the top of the retry loop (before calling `fn`) and missing the backoff-wait window — since that's frequently where most of the wall-clock time is actually spent, an abort during that window would otherwise go unnoticed until the next attempt starts, defeating the purpose of prompt cancellation.

---

**Q (Medium): What's the difference between "full jitter" and "equal jitter," and when would you pick one over the other?**

Answer: Full jitter computes the delay as `random(0, exponentialDelay)` — the entire computed exponential value is just an upper bound, and the actual wait can be anywhere from near-zero to that bound. Equal jitter computes it as `exponentialDelay/2 + random(0, exponentialDelay/2)` — half the delay is guaranteed, and only the other half is randomized. Full jitter spreads retries out more aggressively (better at breaking up synchronized retry storms) but can occasionally produce a very short delay right after a failure, which for a genuinely overloaded/down server means some retries land almost immediately. Equal jitter guarantees a minimum backoff floor at the cost of slightly less spread. For client-facing retry logic against third-party APIs, full jitter is generally the better default (matches AWS's published guidance and most client library implementations); equal jitter is worth choosing when there's a specific reason to guarantee a minimum cooldown period regardless of randomness (e.g., a known rate-limit window).

The trap: treating jitter as a single undifferentiated concept — being able to name and compare the two common strategies (and cite that this is exactly the kind of detail AWS's backoff engineering blog post covers) signals real familiarity versus having only heard "add some randomness" secondhand.

---

**Q (Low): How would you extend this to respect a `Retry-After` header from the server, when present, instead of purely client-computed backoff?**

Answer: When a 429 or 503 response includes a `Retry-After` header (either a number of seconds or an HTTP date), that value should generally take precedence over the client's own exponential calculation — the server is telling the client exactly how long to wait, which is more accurate than a guess, and ignoring it can mean retrying too soon (getting rate-limited again) or waiting longer than necessary. Implementation-wise, `shouldRetry`/the catch block would need access to the response object (not just a generic `Error`), parse `Retry-After` if present, and use `Math.max(serverSuggestedDelay, computedBackoffFloor)` or just directly use the server value when present, falling back to the exponential-with-jitter calculation only when the header is absent.

The trap: computing backoff purely client-side even when the server has explicitly communicated a wait time — this is a case where "the utility knows best" is actually wrong; respecting explicit server guidance is both more correct and more polite to the API being called.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement `retry` with exponential backoff, jitter, a max-attempts cap, and a pluggable retryability predicate from memory
- [ ] Can explain precisely what problem jitter solves that plain exponential backoff doesn't
- [ ] Can articulate the retryable vs. non-retryable error distinction with concrete HTTP status examples
- [ ] Can wire in `AbortSignal` support correctly, including during the backoff wait itself
- [ ] Can compare full jitter vs. equal jitter and justify a default choice

---
*Next: LRU Cache — another bounded-resource problem (this time memory, not request rate), built on a different core data structure (`Map` + eviction policy instead of a promise/timer loop).*
