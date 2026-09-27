# Toast / Notification Queue System

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Decoupled store | Singleton queue + subscriber list; `toast.show(...)` callable from anywhere | Toasts are triggered from arbitrary, unrelated parts of the app — the trigger site shouldn't need a reference to the rendering component |
| Pausable timers | Store `remainingTime` + `startedAt`, not just a fire-and-forget `setTimeout` | Hovering/focusing a toast must pause its dismissal — a plain `setTimeout` can't be paused, only cancelled |
| Max-visible cap | Cap rendered toasts (e.g. 3), queue the rest FIFO | Unbounded toasts stack up and cover the whole screen if many fire in a burst |
| Dedup | Compare new toast against recently-shown ones (message + type) within a window | Prevents the same error firing 5 times from a retry loop turning into 5 stacked identical toasts |
| Accessibility | `aria-live="polite"` (or `assertive` for errors) container, no stolen focus | Screen readers must announce toasts without interrupting whatever the user is doing |
| Not for critical info | Toasts are transient and easy to miss | Anything the user *must* see/act on needs a persistent UI element, not a toast |

## The Scenario

"We need a toast notification system — the kind you see in most web apps, a little message that pops up in the corner and disappears after a few seconds. It needs to support success/error/info variants, auto-dismiss, and it should be usable from anywhere in the codebase, not just one component. Can you design and build that?"

## Clarifying Questions

- **Does "usable from anywhere in the codebase" mean an imperative API like `toast.show('Saved!')` that any module can import and call, rather than something that only works if it's rendered as a child of some specific component?** This is the core architectural question. If the answer is yes — and "from anywhere" strongly implies it — the toast system needs to be a decoupled store/singleton that any code can push into, with exactly one rendering surface (a `<ToastContainer>`) subscribed to that store, rather than toasts being owned by whichever component happens to trigger them.
- **What happens if the user hovers over or focuses a toast that's about to auto-dismiss?** Real toast systems pause the dismiss timer on hover/focus and resume on leave — I'd confirm this is expected, since it changes the timer implementation from a simple one-shot `setTimeout` to something that tracks remaining time and can be paused/resumed.
- **Is there a cap on how many toasts can be visible at once, and what happens to the overflow?** If ten toasts fire in a burst (e.g., ten failed API calls in a retry storm), should all ten stack up on screen, or should there be a visible cap (e.g., 3) with the rest queued and shown as earlier ones dismiss? I'd assume a cap is wanted, since unbounded stacking is a real, visible bug I've seen in production apps.
- **Should identical or near-identical toasts fired in quick succession be deduplicated?** A common real trigger: a retry loop or a batch operation that fails on multiple items and calls `toast.error(...)` with the same message five times in one second. Without dedup, that's five stacked identical toasts, which reads as broken rather than informative.
- **Is any toast conveying information the user absolutely must see or act on (e.g., "your session is about to expire")?** This matters because toasts are inherently transient and easy to miss — if there's a genuinely critical, actionable message in scope, I'd push back on using a toast for it at all and suggest a persistent banner or modal instead, reserving toasts for advisory/confirmatory messages.

## Approach & Trade-offs

The foundational decision is that the toast **queue/store must be decoupled from any single rendering component** — this follows directly from "usable from anywhere in the codebase." I model it as a small singleton module holding an array of active toasts plus a list of subscriber callbacks. Any code, anywhere, calls the exported `toast.show(message, options)` (or convenience wrappers `toast.success(...)`, `toast.error(...)`) to push a new toast into the store; the store notifies its subscribers; exactly one `<ToastContainer>` element, mounted once near the root of the page, subscribes to the store and re-renders whenever it changes. This is the same pub/sub shape as the earlier Pub/Sub scenario from Phase 1 — the toast store *is* a pub/sub system, specialized for one kind of event.

I explicitly rejected a design where toasts are rendered as children of whatever component triggers them (e.g., a `showToast` prop passed down through the tree) — that only works for toasts triggered from within that component's subtree, and "from anywhere" rules it out immediately. A single global store with one rendering surface is the only design that satisfies the actual requirement.

For **timers**, a naive `setTimeout(() => dismiss(id), duration)` per toast works for the simple auto-dismiss case, but breaks the moment hover-to-pause is required: `setTimeout` can be cancelled but not paused-and-resumed — once cancelled, you've lost track of how much time was left. So instead, each toast tracks `duration`, `remainingTime` (initialized to `duration`), and `startedAt` (a timestamp). On hover/focus, I clear the active timer and compute `remainingTime -= (Date.now() - startedAt)`. On leave/blur, I start a new timer for `remainingTime` and record a fresh `startedAt`. This is a small state machine per toast, not just one `setTimeout` call — that's the real complexity this feature hides.

For the **max-visible cap**, I keep two arrays conceptually: `visible` (rendered, capped at e.g. 3) and `pending` (queued, FIFO). When a toast is dismissed (by timer, by user click, or by dedup replacement) and `visible.length < cap`, the oldest `pending` toast is promoted into `visible`. This avoids the common bug where a burst of failures floods the screen with a wall of stacked toasts that's worse than useless.

For **dedup**, I compare an incoming toast's `(message, variant)` pair against toasts currently in `visible` or `pending` (or dismissed within a short recency window, e.g. last 2 seconds) and, on a match, either drop the new one entirely or bump a counter on the existing toast ("Failed to save (×3)") rather than adding a duplicate entry. I chose the counter-bump approach over silent dropping, since silently swallowing repeated errors can hide the fact that something is failing repeatedly — showing "×3" communicates frequency without spamming the screen.

For **accessibility**, the `<ToastContainer>` itself is a single `aria-live="polite"` region (or `aria-live="assertive"` for error-variant toasts, since errors are more time-sensitive) — this means new toasts are announced by screen readers automatically as they're added to the DOM, without the toast ever receiving keyboard focus or interrupting whatever the user was doing. This is a deliberate trade-off: `assertive` is more likely to be heard promptly but also more likely to rudely interrupt other speech, so I only use it for the error variant, not universally.

Finally, I'd flag as a design principle, not an afterthought: **toasts are the wrong mechanism for anything the user must see or act on.** They're transient (auto-dismiss), don't demand attention (no stolen focus, easy to miss if the user is looking elsewhere), and typically can't be recovered once dismissed. Session-expiry warnings, destructive-action confirmations, or anything requiring a response belong in a modal, a persistent banner, or an inline form error — not a toast. I'd say this proactively even if not asked, since it's a common real-world misuse.

## Solution

The store — a singleton, decoupled from any component:

```javascript
function createToastStore({ maxVisible = 3, defaultDuration = 4000 } = {}) {
  let visible = [];
  let pending = [];
  let subscribers = [];
  let nextId = 1;

  function notify() {
    subscribers.forEach((cb) => cb(visible));
  }

  function isDuplicate(message, variant) {
    return [...visible, ...pending].find(
      (t) => t.message === message && t.variant === variant
    );
  }

  function show(message, { variant = 'info', duration = defaultDuration } = {}) {
    const existing = isDuplicate(message, variant);
    if (existing) {
      existing.count += 1;
      notify();
      return existing.id;
    }

    const toast = {
      id: nextId++,
      message,
      variant,
      duration,
      remainingTime: duration,
      startedAt: null,
      timerId: null,
      count: 1,
    };

    if (visible.length < maxVisible) {
      visible.push(toast);
      startTimer(toast);
    } else {
      pending.push(toast);
    }
    notify();
    return toast.id;
  }

  function startTimer(toast) {
    toast.startedAt = Date.now();
    toast.timerId = setTimeout(() => dismiss(toast.id), toast.remainingTime);
  }

  function pause(id) {
    const toast = visible.find((t) => t.id === id);
    if (!toast || toast.timerId === null) return;
    clearTimeout(toast.timerId);
    toast.remainingTime -= Date.now() - toast.startedAt;
    toast.timerId = null;
  }

  function resume(id) {
    const toast = visible.find((t) => t.id === id);
    if (!toast || toast.timerId !== null) return;
    startTimer(toast);
  }

  function dismiss(id) {
    const index = visible.findIndex((t) => t.id === id);
    if (index === -1) return;

    clearTimeout(visible[index].timerId);
    visible.splice(index, 1);

    if (pending.length > 0) {
      const next = pending.shift();
      visible.push(next);
      startTimer(next);
    }
    notify();
  }

  function subscribe(callback) {
    subscribers.push(callback);
    return () => {
      subscribers = subscribers.filter((cb) => cb !== callback);
    };
  }

  return { show, dismiss, pause, resume, subscribe };
}

const toastStore = createToastStore();

// Convenience wrappers — the actual public API most call sites use.
const toast = {
  show: (message, options) => toastStore.show(message, options),
  success: (message, options) => toastStore.show(message, { ...options, variant: 'success' }),
  error: (message, options) =>
    toastStore.show(message, { ...options, variant: 'error', duration: options?.duration ?? 6000 }),
  info: (message, options) => toastStore.show(message, { ...options, variant: 'info' }),
};
```

The rendering surface — one instance, mounted once, subscribed to the store:

```javascript
function mountToastContainer(store) {
  const container = document.createElement('div');
  container.className = 'toast-container';
  container.setAttribute('aria-live', 'polite'); // base region; error toasts override per-node below
  container.setAttribute('aria-atomic', 'false'); // announce only what changed, not the whole region
  document.body.appendChild(container);

  store.subscribe((visibleToasts) => render(container, store, visibleToasts));
}

function render(container, store, toasts) {
  container.innerHTML = '';

  toasts.forEach((toast) => {
    const el = document.createElement('div');
    el.className = `toast toast--${toast.variant}`;
    el.setAttribute('role', toast.variant === 'error' ? 'alert' : 'status');
    // 'alert' implies assertive announcement for errors; 'status' implies polite.

    const text = toast.count > 1 ? `${toast.message} (×${toast.count})` : toast.message;
    el.textContent = text;

    el.addEventListener('mouseenter', () => store.pause(toast.id));
    el.addEventListener('mouseleave', () => store.resume(toast.id));
    el.addEventListener('focusin', () => store.pause(toast.id));
    el.addEventListener('focusout', () => store.resume(toast.id));

    const closeBtn = document.createElement('button');
    closeBtn.setAttribute('aria-label', 'Dismiss notification');
    closeBtn.textContent = '×';
    closeBtn.addEventListener('click', () => store.dismiss(toast.id));
    el.appendChild(closeBtn);

    container.appendChild(el);
  });
}
```

Usage from anywhere — this is the whole point of the decoupled design:

```javascript
// In some deeply nested, unrelated module — no prop drilling, no context needed.
async function saveDocument(doc) {
  try {
    await api.save(doc);
    toast.success('Document saved.');
  } catch (err) {
    toast.error('Failed to save document.');
  }
}
```

> **Check yourself:** Explain, without looking back, why a plain `setTimeout(() => dismiss(id), duration)` per toast is insufficient once hover-to-pause is a requirement — what specific state does the pausable version need that the naive version doesn't track?

## Gotchas

**Fire-and-forget `setTimeout` with no pause support.** If hover/focus pausing is required and the implementation only has a single un-cancellable `setTimeout`, hovering a toast can't stop it from disappearing mid-read — the fix requires tracking `remainingTime` and `startedAt` per toast, not just a timer handle.

**No max-visible cap.** A burst of near-simultaneous toasts (e.g., several failed requests in a retry loop) with no cap stacks indefinitely, potentially covering meaningful screen real estate — this needs a visible cap with FIFO queuing of the overflow.

**No dedup.** The same retry-loop scenario without dedup produces five identical stacked toasts, which reads as a bug rather than as five real, distinct events worth five separate messages.

**Toast container not a live region (or the wrong live region type).** Without `aria-live`, screen readers never announce new toasts at all — sighted-mouse-only testing won't catch this, since the toast is visually obvious but silent to assistive tech.

**`aria-live="assertive"` used for everything.** Assertive announcements interrupt whatever the screen reader is currently saying — appropriate for genuine errors, but jarring and disruptive if used for routine success confirmations too.

**Toast stealing focus.** Moving keyboard focus to a newly-appeared toast (e.g., via `.focus()` on mount) interrupts whatever the user was doing — toasts should be announced, not force focus onto themselves, unless the toast itself requires user interaction (which argues it shouldn't be a toast at all).

**Using a toast for critical, must-see information.** Session-expiry warnings, unsaved-changes warnings, or anything requiring the user's decision shouldn't be a toast — toasts auto-dismiss and are easy to miss, so critical information needs a persistent, deliberately-dismissed UI element instead.

**No cleanup of timers on unmount.** If the app is a SPA and the toast container can be torn down/remounted (e.g., during route transitions in a framework context), leftover `setTimeout` handles calling `dismiss` on a store that's been reset can throw or silently no-op in confusing ways if not guarded.

## Follow-up Questions

**Q (High): Why does "usable from anywhere in the codebase" force a specific architecture, and what would go wrong with a component-owned toast state instead?**

Answer: If toast state lived inside, say, a `<ToastProvider>` component's local state (or a React context value), only code with access to that component's state-setter (via props, context, or a hook) could trigger a toast — deeply nested or entirely unrelated modules (a non-component utility function, an API client's error interceptor, a WebSocket message handler) often have no natural access to component state at all. A decoupled singleton store with an imperative API (`toast.show(...)`) sidesteps this entirely: any module can `import` it and call it directly, with the rendering component merely subscribing to the store rather than owning it. The store becomes the single source of truth; the container is just one of potentially interchangeable renderers of it.

The trap: proposing prop-drilling or context as sufficient — those work fine for toasts triggered from within the component tree, but explicitly fail the "from anywhere, including non-component code" requirement in the prompt.

---

**Q (High): Walk through exactly what state a pausable-on-hover toast timer needs, beyond a single `setTimeout` call.**

Answer: Each toast needs: `duration` (the original full duration), `remainingTime` (how much time is left, mutated over pause/resume cycles), `startedAt` (a timestamp recorded whenever the timer is (re)started), and `timerId` (the current active `setTimeout` handle, or `null` when paused). On pause: clear the current timer via `clearTimeout(timerId)`, then compute `remainingTime -= Date.now() - startedAt` to capture exactly how much time had elapsed since the timer last started, and set `timerId = null`. On resume: start a new `setTimeout` for the current `remainingTime`, and record a fresh `startedAt`. Without tracking `remainingTime` and `startedAt` separately from a single timer handle, cancelling a timer for a pause loses all information about how much time had already elapsed, making a correct resume impossible.

The trap: implementing "pause" as just `clearTimeout` with no time bookkeeping, then "resume" as restarting a fresh full-duration timer — that silently resets the toast's remaining lifetime to full on every hover, rather than actually pausing it.

---

**Q (High): Why must the toast container be an `aria-live` region, and how do you decide between `polite` and `assertive`?**

Answer: Screen readers only automatically announce dynamically-inserted content inside a region marked as "live" — an `aria-live="polite"` region is announced after the screen reader finishes whatever it's currently saying, without interrupting, which is right for routine confirmations (save succeeded, item added to cart). `aria-live="assertive"` interrupts immediately, which is appropriate for time-sensitive errors the user needs to know about right away, but overusing it for non-urgent content is disruptive and can actually degrade the screen-reader experience by constantly interrupting. A reasonable default is a `polite` base region, with error-variant toasts using `role="alert"` (which implies assertive-like behavior) individually, rather than making the whole container assertive.

The trap: marking the entire toast container `assertive` "to be safe" — that makes even trivial confirmations interrupt the user's screen reader mid-sentence, which is a worse experience than not being accessible enough, in the other direction.

---

**Q (Medium): Why shouldn't toasts be used for critical, actionable information, and what should be used instead?**

Answer: Toasts are inherently transient (they auto-dismiss after a few seconds) and spatially easy to miss (typically a small corner element that doesn't demand visual attention or steal focus) — both properties are exactly wrong for information the user must see and potentially act on, like "your session expires in 1 minute" or "this action can't be undone." If the toast disappears before the user notices it, or the user is looking elsewhere on the page, the information is simply lost, possibly with real consequences. Critical or actionable information belongs in a persistent element that requires deliberate dismissal — a modal for anything requiring an immediate decision, or a persistent banner for standing warnings — rather than a toast's fire-and-forget model.

The trap: treating "just make the duration longer" or "don't auto-dismiss this one toast" as a sufficient fix — a long-lived or non-dismissing toast is really just a poorly-styled banner/modal at that point, and forcing it into the toast component's visual/positional conventions (small, corner-anchored, stackable) usually produces a worse UI than just using the right component for the job.

---

**Q (Medium): How would you implement deduplication, and what are the trade-offs between silently dropping a duplicate versus showing a counter?**

Answer: On each `show()` call, check the currently visible and pending toasts (and optionally a short recency window of just-dismissed ones) for a match on `(message, variant)`; if found, either discard the new call entirely (simplest) or increment a `count` field on the existing toast and re-render it with an appended "(×N)" (more informative). Silent dropping is simpler and avoids any visual noise, but it can hide the fact that an error is recurring — if a background retry keeps failing silently, showing only one static toast gives no signal that it's still happening. The counter approach communicates frequency at a glance without spamming the screen with duplicate messages, at the cost of slightly more bookkeeping (needing to reset or expire the counter appropriately).

The trap: implementing dedup keyed only on `message` and ignoring `variant` — two toasts with the same text but different severities (e.g., a benign info message that happens to share wording with an error) would incorrectly collapse into one, losing the more severe variant's distinct treatment.

---

**Q (Medium): How would you cap the number of visible toasts without just discarding overflow toasts entirely?**

Answer: Maintain two separate lists: `visible` (rendered on screen, capped at some max, e.g. 3) and `pending` (a FIFO queue of toasts waiting for a slot). When a new toast is created and `visible` is at capacity, push it onto `pending` instead of `visible`. Whenever a visible toast is dismissed — by its timer, by the user clicking its close button, or by dedup — check `pending`; if non-empty, shift the oldest pending toast into `visible` and start its timer at that point (not when it was originally queued, since starting the timer while queued-but-unseen would let it expire before the user ever saw it).

The trap: starting a pending toast's dismiss timer at creation time rather than at promotion time — that can cause a toast to auto-dismiss while still queued and never actually be shown to the user at all, silently dropping real information.

---

**Q (Low): How would you test the pause/resume timer logic without making tests slow or flaky by waiting on real multi-second timeouts?**

Answer: Use a fake-timer utility (e.g., Jest's or Vitest's fake timers, or `sinon.useFakeTimers()`) to control the passage of time deterministically — advance the mocked clock by a controlled amount, assert the toast is still visible, simulate a hover (calling `pause`), advance time further while paused and assert the toast still hasn't dismissed, simulate mouse-leave (calling `resume`), then advance exactly the remaining duration and assert dismissal fires at precisely that point. This avoids both slow real-time waits and flakiness from timing precision, since the fake timer advances deterministically rather than relying on wall-clock scheduling.

The trap: writing tests that use real `setTimeout`/`await new Promise(r => setTimeout(r, 4000))` delays — these make the test suite slow and can be flaky under CI load where actual elapsed time may not match the intended timer precisely.

---

**Q (Low): How would you support an action button inside a toast (e.g., "Undo") without violating the "toasts shouldn't be the only path to critical functionality" principle?**

Answer: An action button like "Undo" after a delete is a reasonable toast use case specifically because it's *not* the only path to the outcome — declining to click it just means the delete stands, which was already going to happen anyway; the toast is offering a time-limited convenience, not gatekeeping essential functionality. Implementation-wise, this means pausing the auto-dismiss timer while the toast has focus/hover (as already built), and on the action button's click, calling both the undo logic and `store.dismiss(id)` immediately rather than waiting for the timer. The key distinction from the "no critical info in toasts" principle is that failing to interact with the toast should degrade gracefully to a safe default, not silently lose data or leave the user stuck.

The trap: using a toast for an action that has no safe default if the toast is missed (e.g., an action that must be taken or the app ends up in a broken state) — that reintroduces the critical-information-in-a-transient-UI problem the earlier principle warns against.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can design a decoupled store/subscriber architecture that any module can call into
- [ ] Can implement a pausable-on-hover timer using `remainingTime`/`startedAt`, not a fire-and-forget `setTimeout`
- [ ] Can implement a max-visible cap with FIFO overflow queuing
- [ ] Can implement dedup with a counter-bump strategy and explain its trade-off vs. silent dropping
- [ ] Can correctly wire `aria-live`/`role="alert"`/`role="status"` and justify polite vs. assertive per variant
- [ ] Can explain, unprompted, why toasts are the wrong tool for critical/actionable information

---
*Next: Star Rating Component — a much smaller, form-control-shaped widget, but a good venue for the roving-tabindex/ARIA-widget rigor applied at a miniature scale.*
