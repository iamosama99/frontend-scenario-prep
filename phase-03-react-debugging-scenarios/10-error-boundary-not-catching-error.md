# Error Boundary Not Catching an Error — Why

## Quick Reference

| Won't Be Caught | Why | What To Do Instead |
|---|---|---|
| Errors in event handlers | Error boundaries only catch errors thrown during React's own render/lifecycle work, not inside code React isn't actively calling on the stack (an `onClick` handler runs in response to a browser event, outside React's render cycle) | Wrap the handler logic in a `try/catch` directly, or funnel into a manual error-reporting call |
| Errors in async code (`setTimeout`, `fetch().then()`, `async/await` after an `await`) | Same reason — by the time the async callback runs, React is not "on the stack" catching it | `try/catch` around the async logic; surface the error into component state and render fallback UI conditionally |
| Errors during server-side rendering | Error boundaries are a client-render-tree mechanism (in the classic model); SSR errors need to be handled at the server/framework level | Framework-level SSR error handling (e.g., Next.js's own error conventions) |
| Errors thrown inside the error boundary's own fallback render | The fallback UI is still React-rendered content — an error there isn't caught by the same boundary; needs a boundary further up the tree | Keep fallback UI intentionally simple/unlikely to throw, or nest boundaries |
| Errors in the boundary component itself (in `getDerivedStateFromError`/`componentDidCatch`) | Same reasoning — a boundary can't catch its own internal errors | Keep boundary logic minimal and well-tested |

## The Scenario

"We wrapped a big chunk of the app in an Error Boundary. It successfully catches errors thrown while rendering a broken child component — great. But there's a specific bug in this feature where clicking a button throws a runtime error, and the whole app still crashes to a white screen instead of showing our nice fallback UI. Explain exactly why the Error Boundary isn't catching this one, and fix it."

## Clarifying Questions

- **Where exactly does the error get thrown — inside the button's `onClick` handler directly, inside an async callback triggered by the click (a `.then()`, an `await`ed call, a `setTimeout`), or inside the render output that results from a state update the click triggers?** This is the entire crux of the scenario: an error thrown synchronously *during render* (even if triggered indirectly by a click that calls `setState`, causing a re-render that then throws while rendering) *is* something an Error Boundary catches; an error thrown *inside the event handler function itself*, before or without going through React's render phase, is not — distinguishing exactly which of these is happening is the actual diagnostic step, not just "a click causes an error."
- **What does the browser console actually show when this happens** — an uncaught exception with a stack trace pointing at the handler function directly, or React's own "Error boundaries only catch errors in the components below them" kind of surrounding context? The former points at the event-handler case; certain framework/dev-tool setups also add their own console guidance here worth reading carefully rather than skimming past.
- **Is the "whole app crashing to a white screen" actually React unmounting the tree (the Error Boundary's own domain), or is it an entirely separate failure — an uncaught exception bringing down unrelated code, or the browser's own default handling of an uncaught error interfering with something else on the page?** If the Error Boundary genuinely never engages at all (because the error never occurs during a React-managed render), then the "crash" being observed is not React tearing down the tree via its error-handling path — it's whatever the uncaught JS exception does by default (typically just failing silently in the console, unless something else compounds it into a visibly broken page).
- **Does the button's `onClick` call a state setter, and does the actual thrown error happen synchronously in the handler itself, or during the *re-render* that the state update subsequently triggers?** These look superficially similar ("clicking the button causes the crash") but have different catchability — a `setState` call that itself throws synchronously in the handler is not caught by the boundary; but if the handler just updates state normally and the *resulting render*, given the new state, is what throws, that render-phase error *is* within the Error Boundary's jurisdiction.
- **Is this React 18 (or earlier, class-component-based Error Boundaries) or React 19+, where some behavior around uncaught errors and the introduction of Actions/Suspense-related error handling has evolved?** Worth confirming the version, since exact framework-level nuances around what surfaces where can differ, even though the fundamental "boundaries only catch render-phase errors" principle hasn't changed.

## Approach & Trade-offs

**What an Error Boundary actually is, mechanically, and why that mechanism has a specific, narrow jurisdiction.** An Error Boundary is a component (currently must be a class component, implementing `static getDerivedStateFromError` and/or `componentDidCatch`) that React specifically instruments during its own render/commit process — when React is rendering a subtree and a descendant component throws during that render (or during a lifecycle method React calls as part of that render, like `componentDidMount`/`componentDidUpdate`), React catches that throw itself, walks up to find the nearest ancestor Error Boundary, and calls its error-handling methods, allowing it to render fallback UI instead of the broken subtree. This is fundamentally a hook into *React's own call stack during rendering* — React is the one calling component render functions, so React is in a position to wrap those calls in something equivalent to a `try/catch` and intercept what's thrown. An event handler function, by contrast, is called by the *browser's* event dispatch mechanism in response to a user interaction — by the time `onClick`'s function body executes, React's render call stack for the component that defined it has long since finished; there's no React "on the stack" positioned to intercept a throw happening inside that handler, because React isn't the one calling it at that moment.

**Why the same logic applies to async code, and it's not really a separate rule.** An `await`ed call's continuation, a `.then()` callback, or a `setTimeout` callback all execute on a fresh turn of the event loop (a microtask or macrotask), entirely disconnected from whatever call stack originally scheduled them — by the time any of these run, React is definitely not "on the stack," for the same structural reason an event handler isn't. This is why "errors in event handlers" and "errors in async code" are really the same underlying rule (React can only catch what happens synchronously within a render it's actively performing) rather than two independent exceptions to memorize separately.

**The fix for both cases is the same idea: catch the error yourself, then route it into React state, which *does* trigger a render (now within the Error Boundary's jurisdiction).** Wrapping the risky logic in a `try/catch` (for synchronous handler code) or a `.catch()`/`try/catch` around `await` (for async code), and — rather than just logging the error — calling a state setter to record that an error occurred, is what bridges the gap: the *catching* of the raw exception happens manually, outside any Error Boundary's involvement, but the *consequence* (rendering different UI in response) can then be modeled as ordinary React state driving conditional rendering, which is squarely within normal React rendering and can be handled by the same component, or, if the state update itself somehow throws during the resulting re-render, genuinely caught by an ancestor Error Boundary at that point (a render-phase throw, not a handler-phase one).

**Why fallback UI needs to be deliberately simple, and why nested boundaries are a legitimate pattern rather than overkill.** Because the fallback UI rendered by `componentDidCatch`/the boundary's own render method is still React-rendered content, a bug *in the fallback itself* is a render-phase error like any other — and the boundary that just caught the original error is not in a position to catch a *new* error thrown while rendering its own fallback (that would require a boundary further up the tree, if one exists, or the fallback error becomes uncaught if not). This is a good reason to keep fallback UI intentionally minimal and well-tested (a static message and a reload button, not something that itself fetches data or does complex conditional logic) — and to nest Error Boundaries at multiple levels of granularity in a larger app (per-route, and more granular per-widget) specifically so a failure in one section's fallback doesn't take down the whole page if a broader boundary exists above it.

**This is one of several classes of errors an Error Boundary structurally cannot help with — worth being able to enumerate them, not just the specific one asked about.** Beyond event handlers and async callbacks: errors during server-side rendering (a different rendering context/lifecycle entirely, needing framework-level handling — e.g., Next.js's own `error.tsx` conventions build on top of this same underlying limitation); errors thrown by the boundary component's own error-handling methods; and (worth naming, though rarely tested directly) errors in code that runs entirely outside React altogether (a non-React script, a browser extension's injected code, a Web Worker). Understanding *why* each of these falls outside the mechanism (all trace back to "React isn't the one calling this code" or "there's no ancestor boundary available at this exact point") is what separates reciting a list from understanding the actual boundary of the boundary.

## Solution

Reproducing the bug — an error thrown directly inside a click handler, not during render:

```tsx
function DangerousButton() {
  const handleClick = () => {
    const data = getUserData(); // throws synchronously if the user isn't loaded yet
    console.log(data.profile.name); // BUG: throws a TypeError if `data` is null — inside the handler, not during render
  };

  return <button onClick={handleClick}>Show name</button>;
}

// Wrapped elsewhere:
// <ErrorBoundary fallback={<FallbackUI />}><DangerousButton /></ErrorBoundary>
```

Clicking this button throws inside `handleClick` — a function the browser's event system calls directly, not something React is calling as part of a render pass — so no ancestor `ErrorBoundary`, no matter how correctly implemented, will intercept it. The result is an uncaught exception in the console; the app doesn't necessarily "crash" in the Error-Boundary sense at all (React's tree isn't torn down by this), but nothing shows the intended fallback UI either — from a user's perspective, the button just silently fails to do anything (or produces some other broken-looking state), which can look like "the crash the Error Boundary was supposed to prevent" even though the boundary was never actually involved.

Fix — catch it manually, and route the failure into state so it becomes a normal render-phase concern:

```tsx
function SafeButton() {
  const [error, setError] = useState<Error | null>(null);

  const handleClick = () => {
    try {
      const data = getUserData();
      console.log(data.profile.name);
    } catch (err) {
      setError(err as Error); // caught manually; now it's ordinary state driving a render
    }
  };

  if (error) {
    // This is now a normal render-phase concern — if THIS throws for some reason,
    // an ancestor Error Boundary genuinely would catch it, since it's happening during render.
    return <div role="alert">Something went wrong: {error.message}</div>;
  }

  return <button onClick={handleClick}>Show name</button>;
}
```

Same idea for async code:

```tsx
function SafeAsyncButton() {
  const [error, setError] = useState<Error | null>(null);

  const handleClick = async () => {
    try {
      const data = await fetchUserData(); // an unhandled rejection here would NOT be caught by an Error Boundary
      console.log(data.profile.name);
    } catch (err) {
      setError(err as Error);
    }
  };

  if (error) return <div role="alert">Something went wrong: {error.message}</div>;
  return <button onClick={handleClick}>Show name</button>;
}
```

> **Check yourself:** In `SafeButton`, if `handleClick` instead called `setSomeOtherState(newValue)` (no `try/catch` at all) and it was the *subsequent re-render*, given `newValue`, that threw — not the handler itself — would an ancestor `ErrorBoundary` catch that? Walk through exactly why the answer differs from the version shown above, even though both start with "a click causes an error."

## Root Cause

The root cause of the confusion (not the bug itself, which is just an uncaught exception) is a mismatch between the intuitive framing ("an Error Boundary catches errors in this part of the app") and the mechanism's actual, narrower scope ("an Error Boundary catches errors React encounters while it is itself calling render/lifecycle code in this part of the tree"). Any error that occurs outside a call stack React itself initiated — event handlers, async callbacks, SSR, the boundary's own internals — falls outside that scope by construction, not due to a missing feature or a bug in the boundary implementation.

## Gotchas

**Assuming "wrap it in an Error Boundary" is a blanket safety net for a whole feature, regardless of where within that feature errors originate.** This is the exact misconception the scenario is built around — an Error Boundary is not a general `try/catch` for everything happening anywhere inside its subtree; it specifically covers render-phase errors, and every interactive feature inevitably has plenty of non-render-phase code (event handlers, effects' async logic) that needs its own explicit error handling.

**Adding a manual `try/catch` in a handler, catching the error, but only `console.error`-ing it rather than routing it into state that drives a fallback render.** This does stop the exception from being uncaught (useful for not crashing something else on the page), but it does nothing to show the user any fallback UI, silently leaving the interaction looking broken — the manual catch needs to *also* trigger a normal React re-render reflecting the failure, not just suppress the exception.

**Believing a global `window.onerror` or `unhandledrejection` listener is a substitute for either Error Boundaries or manual per-handler catching, in terms of showing fallback UI.** Those global handlers are useful for logging/reporting errors that would otherwise be entirely invisible, but they don't have any mechanism to swap in React fallback UI for a specific broken subtree — they're an observability tool, not a rendering-recovery tool, and conflating the two leaves a gap where errors are *reported* but the UI is still left in a broken state for the user.

**Forgetting that an Error Boundary's fallback render can itself throw**, and having no boundary above it to catch that — leading to a confusing "the fallback UI itself sometimes also fails to appear" secondary bug that looks unrelated to the original issue but traces back to the same "still just React render code, still needs its own coverage" principle.

**Testing Error Boundary behavior only against render-phase errors (e.g., a component that unconditionally throws in its render function) and concluding "boundaries work" without ever testing an event-handler or async error path**, missing the exact gap this scenario is about until it's discovered in production.

## Follow-up Questions

**Q (High): Trace through, precisely, why React is able to intercept an error thrown while rendering a child component, but not one thrown inside that same child's `onClick` handler.**

Answer: When React renders a component tree, it does so by calling each component's function (or class render method) itself, synchronously, as part of one call stack it controls — React can wrap those calls in the equivalent of a `try/catch` (which is, at a low level, essentially what the Error Boundary mechanism does internally), because React is the caller. An `onClick` handler is not called by React's render logic — it's registered with React's synthetic event system, which itself is invoked by the browser's native event dispatch in response to an actual click, entirely outside of any render call React is currently making. By the time the handler runs, React's render call stack for that component finished executing potentially a long time ago (the component already finished rendering and committed); there is no active render-phase call stack for React to have wrapped a `try/catch` around at the moment the handler's code executes, so nothing in React's own machinery is positioned to intercept a throw happening there.

The trap: describing this as "the error boundary only watches renders" without being able to explain *mechanistically why* that's the boundary — specifically, that it comes down to who is calling the code at the moment it throws (React, in the render case; the browser's event system, in the handler case), which is the detail that actually demonstrates understanding rather than reciting a rule.

---

**Q (High): If a click handler calls `setState`, and it's the resulting re-render (not the handler itself) that throws, is that caught by an ancestor Error Boundary? Explain why this case is different from the handler throwing directly.**

Answer: Yes, that case is caught. The handler itself doesn't throw — it just schedules a state update, which the handler function completes normally. React then performs a re-render (of that component and whatever else is affected) as a consequence of the state update, and *that* render is, once again, something React itself is calling directly, on a call stack React controls, exactly like the initial render — so if the render function throws given the new state, it's within the exact same mechanism that catches any other render-phase error, and the nearest ancestor Error Boundary catches it correctly. The distinguishing factor isn't "did a click cause this" (both scenarios are triggered by a click) — it's *where, in terms of whose call stack, the actual throw occurs*: inside the handler function's own synchronous execution (not caught) versus inside a subsequent React-initiated render triggered by that handler's side effect (caught).

The trap: answering based on the surface-level trigger ("a click caused it, so it depends on the button, not the mechanism") rather than tracing exactly which call stack the throw occurs on — this is the precise distinction that separates a shallow understanding of Error Boundaries ("catches errors near this component") from a correct one ("catches errors React itself encounters while rendering").

---

**Q (High): Design an approach for a component with several async operations (a fetch on mount, a fetch triggered by a button, a WebSocket message handler) such that any of their failures reliably show the same fallback UI an Error Boundary would show for a render-phase error — without duplicating error-handling logic three times.**

Answer: Centralize a single piece of error state (or reuse a shared custom hook, e.g., `useAsyncError` or similar) that any of the three async paths can report into via a shared setter, and render one conditional block based on that single error state — each async operation wraps its own logic in `try/catch` (or a `.catch()`), but all three funnel into calling the *same* `setError` function on failure, rather than each implementing its own separate fallback-rendering logic. This keeps the "what does the fallback look like" decision in exactly one place (the same conditional render block used for all three failure sources), while still requiring each async call site to individually catch its own errors (since, as established, none of them can rely on an Error Boundary to do this for them) — a reasonable, common pattern is a small custom hook wrapping this pattern (catch, store in shared error state, expose a `throwError`-style setter) so each async call site's `try/catch` block is a short, consistent one-liner rather than three independently-designed error-UI implementations that could drift out of sync with each other.

The trap: proposing three separate `try/catch` blocks each rendering their *own* distinct fallback UI inline at the point of failure — this technically catches every error, but produces three different, independently-maintained "error looks like this" implementations for what should conceptually be one consistent failure experience, and is more code to keep in sync than necessary.

---

**Q (Medium): Some versions of frameworks/libraries advertise "the ability to convert an async error into something an Error Boundary can catch." How would that actually work, given everything established above?**

Answer: It works by doing exactly what the manual fix in this scenario does, just packaged as a reusable utility: catching the async error where it actually occurs (in a `.catch()`/`try/catch`, since that part is unavoidable — nothing can make an Error Boundary directly intercept an async callback), and then, instead of just storing it in local state and conditionally rendering a fallback inline, deliberately *re-throwing* it during the *next render* of some component (commonly via a small hook that stores the error and, if present, throws it synchronously the next time the component renders) — because a `throw` that happens *during a component's render function* absolutely is something an ancestor Error Boundary catches, regardless of whether the *original* failure was asynchronous. This is a legitimate, sanctioned pattern (sometimes described as "the error-boundary-compatible async error" trick) — it doesn't change the fundamental rule (boundaries only catch render-phase throws); it exploits that rule by deliberately converting an async failure into a render-phase throw on purpose, on the next render, rather than trying to make the boundary reach into the original async callback (which remains impossible).

The trap: believing this represents some special new capability where Error Boundaries "can now catch async errors directly" — the underlying mechanism (render-phase-only catching) is unchanged; what's new is a convenient pattern for *converting* an async error into a render-phase one deliberately, which is a meaningfully different (and important to be able to articulate) claim than "boundaries got smarter about async."

---

**Q (Medium): Why can't the Error Boundary component itself use a `try/catch` inside its own `componentDidCatch` to protect against a bug in its own fallback UI's render logic?**

Answer: `componentDidCatch` (and `getDerivedStateFromError`) run in response to an error that already occurred in a *descendant's* render — by the time these methods run, React has already unwound the failed subtree and is about to render the boundary's own fallback output as this component's *own* render result (typically by way of `getDerivedStateFromError` updating state, which then affects what this component's own `render` method returns). Wrapping `componentDidCatch`'s own body in a `try/catch` wouldn't help, because the actual risk isn't inside `componentDidCatch`'s body itself — it's in the fallback JSX/render output that gets produced afterward, when *this* component (the boundary) is rendered again with its error state set. Since that's the boundary component's *own* render, and a component cannot catch errors thrown during its own render (an Error Boundary catches errors from its *children*, not from itself — mirroring why a `try/catch` block can't catch an exception thrown by the exact same expression that would need to be inside its own `catch` clause to be caught), an error there requires an Error Boundary positioned *above* this one in the tree to be caught at all; if none exists, it propagates as an uncaught render-phase error.

The trap: proposing to solve this by adding more error-handling code *inside* the boundary component's own methods — the issue isn't a lack of try/catch coverage inside those methods; it's a structural limitation (a component can't be its own safety net for its own render output), which requires nesting, not more defensive code in the same component.

---

## Self-Assessment

- [ ] Can explain, mechanistically, why React can intercept render-phase errors but not event-handler errors, in terms of whose call stack the throw occurs on
- [ ] Can correctly determine, for a "click triggers an error" scenario, whether the throw happens in the handler itself or in a subsequent render — and why that distinction determines catchability
- [ ] Can design a centralized error-state pattern that lets multiple independent async operations funnel into one consistent fallback UI
- [ ] Can explain the "re-throw during next render" pattern for making an async error catchable by an ancestor boundary, and why it doesn't contradict the render-phase-only rule
- [ ] Can explain why a boundary's own fallback render isn't protected by that same boundary, and why nesting (not more code inside the boundary) is the fix
- [ ] Can enumerate the full set of error categories Error Boundaries structurally cannot catch, with the underlying reason for each

---
*Phase 3 complete. Next: Phase 4 — Frontend System Design, starting with Design a News Feed (Facebook/LinkedIn-style) — the shift from "debug this broken thing" to "design this from a blank whiteboard," where the reasoning skill being tested moves from root-causing to requirements-gathering and trade-off architecture at scale.*
