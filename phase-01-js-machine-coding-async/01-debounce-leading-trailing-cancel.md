# Debounce — Leading/Trailing/Cancel

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Trailing debounce (default) | Reset a timer on every call; fire once the calls stop for `delay` ms | Fires with the *last* set of args — right choice for "run after the user stops typing" |
| Leading debounce | Fire immediately on the first call in a quiet window, ignore the rest until the window closes | Right choice when you want instant feedback but still want to throttle repeats (e.g., button double-click guard) |
| `cancel()` | Clears the pending timer, drops any queued trailing call | Required in React to avoid calling `setState` after unmount, or firing a stale search after the user navigated away |
| `flush()` | Immediately invokes the pending trailing call and clears the timer | Needed when the surrounding code needs the result *now* (e.g., form submit right after a debounced field update) |

## The Scenario

"We have a search input that fires an API call on every keystroke, and it's hammering our backend. Can you write a `debounce` utility for it? I want it to support both leading and trailing edge invocation like Lodash's `debounce`, and I want to be able to cancel a pending call — we'll need that for cleanup when the component unmounts."

## Clarifying Questions

- **Does the debounced function need to preserve `this` and forward all arguments?** Yes — this is a generic utility, not tied to one call site, so it has to behave like a transparent wrapper: `this` binding and full argument forwarding both matter, or it'll break the moment someone uses it as an object method.
- **What happens if `leading` and `trailing` are both `true`?** This is the edge case interviewers plant on purpose. If a caller fires a single call, should it invoke once or twice? The correct behavior (and Lodash's): a single isolated call only fires once (leading fires it, and since there's no *additional* call to justify a trailing fire, trailing is a no-op). But if a second call comes in before the window closes, trailing fires again at the end with the latest args — so you can get two invocations total.
- **Does `cancel()` need to exist, and what should it guarantee?** Yes — the prompt already says it's needed for unmount cleanup. It needs to guarantee no queued call fires after `cancel()` is called, synchronously.
- **Is the return value of the debounced call needed?** No — a debounced function's real work happens asynchronously (after a delay), so the wrapper itself can't synchronously return the underlying function's result. This matters because it rules out designs that try to make the debounced function look synchronous.

## Approach & Trade-offs

The core idea: keep a single `timerId` in the closure. Every call to the debounced function clears any existing timer and starts a new one — that's what "resets the quiet-window clock on every call" means mechanically.

For **trailing** (the default), the invocation happens *inside* the `setTimeout` callback, using whatever `args`/`this` were captured on the *last* call before the timer fired.

For **leading**, the invocation happens synchronously on the call that starts a new window (i.e., when there was no pending timer), and then the timer that gets set is purely a "cooldown" — its callback does nothing itself, it just marks the window as closed.

Supporting *both* means tracking two pieces of state: whether we're at the start of a new window (to decide if leading should fire) and whether trailing should fire when the window's timer expires (only if at least one call happened since the window opened, otherwise a single leading-only call would double-fire).

I chose a single shared `timerId` + a `lastArgs`/`lastThis` capture rather than two independent timers, because leading and trailing aren't really two separate mechanisms — they're two decisions about *when in the same window* to invoke.

## Solution

```javascript
function debounce(fn, delay, options = {}) {
  const { leading = false, trailing = true } = options;

  let timerId = null;
  let lastArgs = null;
  let lastThis = null;
  let calledDuringWindow = false; // did a call happen after the leading fire?

  function invoke() {
    const argsToUse = lastArgs;
    const thisToUse = lastThis;
    lastArgs = lastThis = null;
    fn.apply(thisToUse, argsToUse);
  }

  function debounced(...args) {
    lastArgs = args;
    lastThis = this;

    const isNewWindow = timerId === null;

    if (isNewWindow && leading) {
      invoke(); // fire immediately, before the timer is even set
      calledDuringWindow = false;
    } else {
      calledDuringWindow = true;
    }

    clearTimeout(timerId);
    timerId = setTimeout(() => {
      // Only fire trailing if a call happened that leading didn't already handle.
      if (trailing && calledDuringWindow) {
        invoke();
      }
      timerId = null;
      calledDuringWindow = false;
    }, delay);
  }

  debounced.cancel = function () {
    clearTimeout(timerId);
    timerId = null;
    lastArgs = lastThis = null;
    calledDuringWindow = false;
  };

  debounced.flush = function () {
    if (timerId !== null && lastArgs !== null) {
      clearTimeout(timerId);
      timerId = null;
      invoke();
    }
  };

  return debounced;
}
```

```javascript
// Usage in a React component — the cancel() call is the part interviewers
// specifically check for, since forgetting it is the #1 real-world bug.
useEffect(() => {
  const debouncedSearch = debounce(runSearch, 300);
  inputEl.addEventListener('input', (e) => debouncedSearch(e.target.value));

  return () => debouncedSearch.cancel(); // no stale search fires after unmount
}, []);
```

> **Check yourself:** If you call a `debounce(fn, 300, { leading: true, trailing: true })` function three times, 50ms apart, how many times does `fn` run, and with which call's arguments?

## Gotchas

**Losing `this`.** If you write `debounced = (...args) => { ... fn(...args) }` instead of `fn.apply(thisToUse, args)`, any method-style usage (`obj.debouncedMethod()`) silently breaks because `this` inside an arrow function is lexically bound, not dynamic.

**Forgetting `cancel()` in cleanup.** In React, a debounced call from a component that has since unmounted will still try to run its callback — if that callback calls `setState`, React warns (or in older versions, leaks). The fix is always calling `.cancel()` in the effect's cleanup function.

**Leading + trailing double-fire misunderstanding.** Candidates often implement leading and trailing as two fully independent timers, which causes a lone, isolated call to fire twice instead of once. The fix is tracking whether a call happened *after* the leading fire before deciding whether trailing should also fire.

**Debounce is wrong for continuous feedback.** Using debounce on a `scroll` or `resize` handler means the handler never runs *during* the activity, only after it stops — usually the wrong tool. That's what throttle is for (next scenario).

**Debounced calls don't return values synchronously.** A common mistake is trying to `await` the return value of `debounced()` directly — it returns `undefined` since the real call happens later, asynchronously, inside a timer callback. Callers that need the result have to be redesigned around a callback/promise passed through, not a return value.

## Follow-up Questions

**Q (High): Walk through leading vs. trailing debounce with a concrete timeline. Why does UX differ between them?**

Answer: Say `delay = 300ms` and calls come in at `t=0, t=100, t=250`. With **trailing** (the default), nothing happens until 300ms after the *last* call — so the function fires once, at `t=550`, with the args from the `t=250` call. With **leading**, the function fires immediately at `t=0` with that call's args, and the calls at `t=100`/`t=250` are absorbed into the same cooldown window and produce no further trailing call (since `trailing` isn't necessarily enabled, or if it is, it fires once more at `t=550` reflecting the args from `t=250`). UX-wise: trailing is right for "wait until the user is done" (search-as-you-type); leading is right for "respond instantly to the first action but ignore rapid repeats" (a submit button that shouldn't double-fire on a double-click).

The trap: saying "leading means it fires on every call, right away" — leading only fires on the *first* call of a new window, not every call; that's the entire point of debounce (collapsing a burst into one or two calls).

---

**Q (High): Why does `cancel()` matter in a React component, and what breaks if you omit it?**

Answer: A debounced function holds a `setTimeout` handle that outlives the component's render if the component unmounts before the timer fires. If the debounced callback eventually runs `setState` (or dispatches to a store) on unmounted/stale state, you get a "can't perform a state update on an unmounted component" warning at best, and a real bug at worst — e.g., an autocomplete whose debounced fetch resolves after the user has navigated away, overwriting the new screen's state with stale data. Calling `debounced.cancel()` in the `useEffect` cleanup function guarantees the pending timer (and its captured stale closure) is discarded the moment the component tears down.

The trap: candidates who implement `cancel()` correctly but then forget to actually *call* it in cleanup — writing the utility right isn't the same as using it right.

---

**Q (High): Trace through what happens with `leading: true, trailing: true` for a single, isolated call, and for two calls close together. Why is a naive implementation prone to double-firing on the single-call case?**

Answer: For a single isolated call: leading fires immediately since it's a new window; when the cooldown timer expires, trailing should be a no-op because no *additional* call happened during the window — so the function runs exactly once. For two calls 50ms apart within a 300ms window: leading fires immediately on the first call; the second call resets the timer and marks that a call happened during the window; when the timer eventually expires, trailing fires with the second call's args — so the function runs twice total, with different arguments each time. A naive implementation that doesn't track "did a call happen since the leading fire" will incorrectly fire trailing even after a single isolated leading call, doubling every call instead of correctly collapsing bursts.

The trap: treating leading and trailing as two independent, unconditional timers instead of two conditional *decisions* about the same window.

---

**Q (Medium): How would you adapt this debounce to work with an `async` function whose result the caller actually needs?**

Answer: The debounced wrapper itself can't return a synchronous value, since the real invocation is deferred. To let a caller get the eventual result, you return a `Promise` from each call to the debounced function and resolve/reject all pending promises for the current window when the underlying function's promise settles — but you have to decide the semantics for calls that get "absorbed" into a window without being the one that actually executes: do their promises resolve with the same eventual result (fan-out), or do they never resolve because they were effectively cancelled? Most real implementations pick fan-out — every promise from calls within the same window resolves to the one underlying invocation's result — since silently-unresolved promises are a memory leak and a footgun.

The trap: assuming you can just `return fn.apply(...)` from inside the `setTimeout` — that return value goes nowhere; `setTimeout` callbacks' return values are discarded.

---

**Q (Medium): What's a memory/performance gotcha with a debounced function that closes over a large object or DOM node?**

Answer: The closure created by `debounce()` holds references to whatever `lastArgs`/`lastThis` were passed in — if those include large payloads or DOM nodes and the debounced function is long-lived (e.g., a module-level singleton rather than a per-component instance), those references stay reachable by the GC roots for the debounced function's entire lifetime, even between invocations, because the closure variables persist. This usually isn't significant for primitive search terms, but becomes real when debouncing something like a resize handler that captures a reference to a large canvas or an entire virtual DOM tree.

The trap: assuming debounce itself "cleans up" — it only clears the *timer*; it doesn't null out captured references except right before/after invocation, which this implementation does deliberately (`lastArgs = lastThis = null` after invoking) specifically to avoid holding stale references longer than necessary.

---

**Q (Low): How would you implement debounce as a class instead of a closure factory, and is there a meaningful difference?**

Answer: A class-based version stores `timerId`, `lastArgs`, etc. as instance fields instead of closure variables, and exposes `invoke()`, `cancel()`, `flush()` as methods, with the debounced call itself as a bound method (or an arrow function class field to preserve `this`). Functionally it's equivalent — the meaningful difference is discoverability/ergonomics (an instance is more inspectable in a debugger, and easier to extend with subclassing) versus the closure version's smaller footprint and more common "feels like a normal function" ergonomics that most utility libraries (Lodash, etc.) actually ship.

The trap: claiming there's a *behavioral* difference — there isn't one if both are implemented correctly; this is purely a code-organization question.

---

**Q (Low): How does a `requestAnimationFrame`-based debounce differ from a `setTimeout`-based one, and when would you prefer it?**

Answer: An rAF-based debounce schedules the deferred work via `requestAnimationFrame` instead of a fixed millisecond delay, so the callback runs right before the browser's next paint rather than after an arbitrary wall-clock delay. This is preferable for visual updates driven by rapid events (like `resize` or `scroll`-linked layout reads) because it naturally throttles to the display's actual refresh rate and avoids scheduling work when the tab is backgrounded (rAF pauses in background tabs; `setTimeout` doesn't, wasting cycles). The trade-off is that rAF-based debounce isn't meaningful for anything that isn't tied to rendering — you wouldn't rAF-debounce an API call, since there's no "next frame" relevance to a network request.

The trap: presenting rAF-debounce as a drop-in replacement for all `setTimeout` debounce use cases — it's specifically suited to visual/layout work, not general-purpose rate limiting.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement trailing-edge debounce with `cancel()` from memory
- [ ] Can explain, with a timeline, the difference between leading and trailing invocation
- [ ] Can explain why leading+trailing together doesn't double-fire on a single isolated call
- [ ] Can state why `this`/args forwarding matters and how to preserve them
- [ ] Can explain why `cancel()` is required in a React `useEffect` cleanup
- [ ] Can explain why debounce is the wrong tool for a `scroll` handler

---
*Next: Throttle — Leading/Trailing — the sibling rate-limiting technique, used when you need the handler to keep firing *during* the activity instead of only after it stops.*
