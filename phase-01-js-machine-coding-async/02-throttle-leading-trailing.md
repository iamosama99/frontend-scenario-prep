# Throttle — Leading/Trailing

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Leading throttle | Invoke immediately, then ignore calls for the rest of the interval | Guarantees the *first* event in a burst is handled instantly (scroll position at the moment scrolling starts) |
| Trailing throttle | After the interval ends, if a call was suppressed during it, fire once more with the latest args | Without it, the *final* state of a burst (e.g., where the user stopped scrolling) is silently dropped |
| Timestamp-based (no timer) | Compare `Date.now()` to `lastCallTime`; only invoke if `>= interval` has passed | Simple, but leading-only — loses the trailing call by construction |
| `cancel()` | Clears any pending trailing timer | Same unmount-safety need as debounce |

## The Scenario

"We have a scroll handler that updates a 'reading progress' bar, and right now it runs on every single `scroll` event — that's hundreds of calls per second on a fast trackpad fling. Write a `throttle` function so it runs at most once every, say, 100ms, but make sure the progress bar still ends up accurate when the user stops scrolling — like Lodash's `throttle` with leading and trailing options."

## Clarifying Questions

- **Is `leading` the default `true`?** Yes, matching Lodash's default — the first event in a burst should be handled immediately, since a scroll handler that only ever fires on a trailing edge would feel laggy at the start of every scroll gesture.
- **Why does "make sure the progress bar ends up accurate" matter specifically?** Because it's telling me `trailing` must be enabled — if the last scroll event in a burst lands inside the throttle window and gets suppressed, the progress bar freezes at a stale value until the *next* scroll event happens to trigger an update, which might be never if the user has stopped scrolling. That's the concrete bug trailing-edge invocation prevents.
- **Can `leading` and `trailing` both be `false`?** That's a degenerate case (the function never runs) — worth flagging but not worth defending against unless asked; Lodash throws/no-ops in that combination.
- **Should this be built on `setTimeout` or on `requestAnimationFrame`?** For a scroll-driven visual update specifically, rAF-based throttling is arguably the *better* real answer (see the Low follow-up below), but the prompt asked for Lodash-style `throttle`, which is `setTimeout`/timestamp based — I'll implement that, and mention rAF as the follow-up refinement.

## Approach & Trade-offs

Throttle's core difference from debounce: debounce resets its clock on every call (so a continuous burst *never* fires until it stops); throttle runs on a *fixed cadence* regardless of how many calls arrive, guaranteeing regular execution during a sustained burst.

The simplest correct mental model is: track `lastInvokeTime`. On each call, if `now - lastInvokeTime >= interval`, invoke immediately (this covers the leading-edge case, and also every subsequent invocation once the interval has elapsed). If less time has passed, don't invoke — but if `trailing` is enabled, schedule a `setTimeout` for the *remaining* time in the interval so that if no further "fresh" call resets it, the last received args still get their turn to fire.

I chose timestamp comparison over "just use a boolean cooldown flag" because a pure boolean can't tell you *how much* of the interval remains, which trailing-edge scheduling needs to compute the correct remaining delay rather than a full fresh `interval`.

## Solution

```javascript
function throttle(fn, interval, options = {}) {
  const { leading = true, trailing = true } = options;

  let lastInvokeTime = 0;
  let timerId = null;
  let lastArgs = null;
  let lastThis = null;

  function invoke(time) {
    lastInvokeTime = time;
    const argsToUse = lastArgs;
    const thisToUse = lastThis;
    lastArgs = lastThis = null;
    fn.apply(thisToUse, argsToUse);
  }

  function throttled(...args) {
    const now = Date.now();

    // First call ever, with leading disabled: treat as if the window just started,
    // so we don't immediately invoke, but future calls should still throttle correctly.
    if (lastInvokeTime === 0 && !leading) {
      lastInvokeTime = now;
    }

    const remaining = interval - (now - lastInvokeTime);
    lastArgs = args;
    lastThis = this;

    if (remaining <= 0) {
      // Interval has elapsed — invoke now, and cancel any stale trailing timer.
      clearTimeout(timerId);
      timerId = null;
      invoke(now);
    } else if (trailing && timerId === null) {
      // Schedule exactly the remaining time so the last call in the burst still fires.
      timerId = setTimeout(() => {
        timerId = null;
        invoke(Date.now());
      }, remaining);
    }
  }

  throttled.cancel = function () {
    clearTimeout(timerId);
    timerId = null;
    lastInvokeTime = 0;
    lastArgs = lastThis = null;
  };

  return throttled;
}
```

```javascript
// Usage
const updateProgressBar = throttle(() => {
  const pct = (window.scrollY / (document.body.scrollHeight - window.innerHeight)) * 100;
  progressBarEl.style.width = `${pct}%`;
}, 100);

window.addEventListener('scroll', updateProgressBar);
```

> **Check yourself:** With `interval = 100`, calls arrive at `t=0, 30, 60, 90, 120, 400`. Using leading+trailing throttle, at which timestamps does `fn` actually run?

## Debounce vs. Throttle — When to Use Each

| | Debounce | Throttle |
|---|---|---|
| Fires during a continuous burst? | No — only after it stops (or once at the very start with leading) | Yes — at a regular cadence throughout |
| Right for | Search-as-you-type, form autosave, resize-end layout recalculation | Scroll position tracking, mousemove-driven drag, rate-limiting a button click handler |
| Guarantees a response to the *last* event? | Yes, by construction (trailing is the default) | Only if `trailing: true` is enabled |
| Guarantees a response to the *first* event? | Only if `leading: true` | Yes by default (`leading: true` is Lodash's default) |

## Gotchas

**Losing the final state without `trailing`.** A leading-only throttle on a scroll handler updates at the start of scrolling and then at fixed intervals, but if the user stops scrolling mid-interval, the handler never runs again for that final position — the UI is left showing a stale value. This is the exact bug the scenario's "make sure it ends up accurate" requirement is testing for.

**Timestamp-only implementations can't do trailing.** A throttle built purely on `Date.now()` comparisons (no `setTimeout` at all) is leading-only by construction — there's no mechanism to "wake up later" and fire a trailing call, since nothing schedules future work. Recognizing this trade-off — and that it's sometimes an acceptable simplification — is part of a strong answer.

**Clock skew / drift with `Date.now()`.** For high-precision needs, `performance.now()` is monotonic and unaffected by system clock adjustments (NTP sync, manual clock changes), whereas `Date.now()` can jump. For a 100ms UI throttle this rarely matters, but it's worth naming as the more correct primitive.

**Forgetting `cancel()`.** Exactly the same unmount-safety issue as debounce — a pending trailing `setTimeout` can fire after a component using the throttled callback has unmounted.

**Confusing throttle's job with rate-limiting a network call.** Throttle controls how often a *function* executes; it says nothing about server-side rate limits, concurrent request caps, or retry behavior — those need separate handling (see the retry-with-backoff and concurrency-limited queue scenarios later in this phase).

## Follow-up Questions

**Q (High): Explain leading vs. trailing throttle with a timeline. Why would a scroll progress bar specifically need trailing enabled?**

Answer: With `interval = 100ms` and leading+trailing both on, calls at `t=0, 30, 60, 90, 120` produce invocations at `t=0` (leading, immediate), then at `t=100` (the trailing call scheduled for the remainder of the first window, using the most recent args available at that point — from the `t=90` call), then at `t=120` a fresh call arrives after the interval has fully elapsed since the last invoke, so it invokes immediately too. If a burst then goes quiet, the very last received call's args are guaranteed to fire once trailing's remaining-time timer expires — that's specifically what keeps a scroll progress bar accurate at rest, since without it, whatever position the user stopped at might fall inside a throttle window and never get its own render.

The trap: describing throttle as "runs every N ms no matter what" without acknowledging that a quiet period after the last call still needs the trailing mechanism to deliver that last event's data — throttle isn't purely time-driven, it's driven by "the interval since the last real invocation."

---

**Q (High): Why would you choose throttle over debounce for a scroll listener, and what visibly breaks if you use debounce instead?**

Answer: Debounce resets on every call, so during a continuous, uninterrupted scroll (no natural pauses), a purely trailing debounce never fires at all until scrolling fully stops — the progress bar would stay frozen the entire time the user is actively scrolling and only jump to the correct value once they lift their finger/stop the trackpad. Throttle guarantees regular updates *during* the activity, which is what continuous visual feedback requires. Debounce is correct when you only care about the *end state* of a burst (e.g., "run the search once the user stops typing"); throttle is correct when you need ongoing feedback proportional to an ongoing action.

The trap: treating debounce and throttle as interchangeable "rate limiters" — they solve different problems (respond-once-after vs. respond-regularly-during), and picking the wrong one produces a UI that's either laggy (throttle where debounce was right) or frozen during activity (debounce where throttle was right).

---

**Q (High): Implement throttle using only timestamp comparison, no `setTimeout`. What capability do you lose?**

Answer:
```javascript
function throttleLeadingOnly(fn, interval) {
  let lastTime = 0;
  return function (...args) {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn.apply(this, args);
    }
  };
}
```
This fires immediately on the first call and on every subsequent call that arrives at least `interval` ms after the last invocation — but any call that arrives *before* the interval has elapsed is simply dropped with no follow-up, because there's no scheduled work to "catch" it later. The lost capability is exactly the trailing-edge guarantee: the final event of a burst that happens to land inside the throttle window is discarded permanently, which is the bug the earlier "reading progress bar" scenario specifically needs to avoid.

The trap: presenting this as a complete throttle implementation without flagging that it's leading-only — many candidates write exactly this and stop, missing that the original ask explicitly required trailing behavior.

---

**Q (Medium): How does requestAnimationFrame-based throttling differ from setTimeout-based throttling, and when is it preferable for scroll-linked work?**

Answer: An rAF-based throttle schedules the deferred work via `requestAnimationFrame` rather than a fixed millisecond delay, so it naturally runs at most once per rendered frame (typically ~16.7ms at 60Hz, but adapting automatically to the display's actual refresh rate, including 120Hz displays) and it automatically pauses when the tab is backgrounded, since rAF callbacks don't fire for backgrounded tabs. For scroll-linked visual updates specifically (like the reading progress bar), this is more correct than an arbitrary `100ms` `setTimeout` interval, because it ties the update rate to the browser's actual paint cadence instead of a guessed constant, avoiding both wasted work (updating faster than the screen can show it) and visible stutter (updating slower than paint cadence in a way that's out of sync with it).

The trap: assuming rAF-throttle is a strict improvement for every use case — for something like a rate-limited "save draft" API call, rAF is meaningless (there's no "next frame" relevance to a network request); it's specifically suited to layout/paint-adjacent work.

---

**Q (Medium): How do you cancel a pending trailing call, and why is it needed for the same reason as debounce's `cancel()`?**

Answer: The trailing branch schedules a `setTimeout` holding a closure over the latest `args`/`this`; if the component or listener that owns the throttled function is torn down before that timer fires, the callback still runs against a stale/unmounted context unless something clears it. `cancel()` needs to `clearTimeout` the pending trailing timer and reset `lastInvokeTime`/`lastArgs` so no stale invocation slips through — the underlying risk (a `setState`-after-unmount, or an event handler firing against a DOM node that's been removed) is identical to debounce's cancellation requirement, since both utilities are, at the implementation level, "closures holding a pending timer."

The trap: forgetting that `cancel()` must also reset `lastInvokeTime` — otherwise a fresh call right after `cancel()` might incorrectly compute `remaining <= 0` based on stale timing state and behave unpredictably depending on implementation details.

---

**Q (Low): How would you implement a throttle whose interval adapts dynamically — for example, tightening under low load and relaxing under high load?**

Answer: Instead of a fixed `interval` constant, read the interval from a mutable reference (a variable, or a function call) at the moment each throttle decision is made, so the same throttled function can respond to changing conditions — e.g., measuring recent frame times via `PerformanceObserver` and widening the interval when frames are being dropped, or tightening it when the system is idle. The throttle's internal logic (`remaining = interval - (now - lastInvokeTime)`) doesn't change; only the source of `interval` becomes dynamic rather than a closed-over constant. The main design question this raises is whether changing the interval mid-window should affect an *already scheduled* trailing timer — most implementations leave an in-flight timer alone and only apply the new interval to the *next* throttle decision, to avoid inconsistent behavior mid-burst.

The trap: overcomplicating the answer by trying to reschedule in-flight timers on every interval change — the simpler and more defensible design applies the new interval prospectively, not retroactively.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement leading+trailing throttle using timestamp comparison plus a single `setTimeout` for the remainder
- [ ] Can explain why a timestamp-only throttle can't support trailing invocation
- [ ] Can state, from memory, the practical difference between debounce and throttle and give one correct use case for each
- [ ] Can explain why a scroll progress bar specifically needs trailing enabled
- [ ] Can explain when rAF-based throttling is preferable to `setTimeout`-based throttling

---
*Next: Deep Clone — Circular References & Special Types — a different closures-and-edge-cases machine coding staple, this time about recursively walking a data structure instead of rate-limiting a function.*
