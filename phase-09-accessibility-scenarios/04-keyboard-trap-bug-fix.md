# Keyboard Trap Bug — Find and Fix

## Quick Reference

| Symptom | Likely Cause | Check First |
|---|---|---|
| Tab cycles forever inside a widget, no way out | A `keydown` listener intercepts Tab unconditionally, with no matching "am I actually at a boundary" or "is this even supposed to trap" condition | Does the trap check *which* element currently has focus before calling `preventDefault()`? |
| Focus silently vanishes / lands on `<body>` | Something in the widget's render tree unmounted or hid the focused element without moving focus first | Did the focused element get removed from the DOM (conditional render, `hidden`) without an explicit re-focus? |
| Third-party embed (map, video, rich-text editor) swallows Tab | The embed's own internal focus management captures keyboard events before they reach your app's listeners | Is the trap listener attached where it can actually intercept events from inside the embed, or does the embed stop propagation first? |
| Trap works, but Escape/close button don't release it | The exit handler and the trap's `keydown` listener are racing, or the exit handler isn't wired to the same open/close state the trap reads | Does closing actually flip the same state variable the trap's effect is keyed on? |

## The Scenario

"QA filed a bug: 'Once you open the settings panel and start tabbing through it, you can never leave — Tab and Shift+Tab just keep cycling through the panel's fields forever, even after clicking Close.' Here's the component. Find the bug and fix it — and tell me whether this is actually a WCAG violation and which criterion."

This is deliberately the mirror image of the [Accessible Modal](01-accessible-modal-scenario.md) scenario: there, the goal is building an *intentional* trap correctly. Here, an *unintentional* trap already exists in working-looking code, and the job is diagnosis under time pressure — which is a meaningfully different skill from writing the pattern from scratch.

## Clarifying Questions

- **Does this happen on every browser, or is it browser-specific?** A trap that only reproduces in one browser points toward something environment-specific (a Tab-handling quirk, a focus-order difference) rather than a logic bug in the trap code itself, which changes where I'd look first.
- **Does clicking Close visually close the panel, or does it stay open with the trap still active?** This distinguishes two very different bug classes: a rendering/state bug (Close doesn't actually close anything) versus a listener-cleanup bug (the panel closes but the `keydown` listener that traps Tab is never removed, so it keeps intercepting events against a now-hidden or nonexistent panel).
- **Is there any third-party embed inside the panel (a date picker widget, a rich-text editor, an iframe)?** Third-party components sometimes run their own internal focus management that can interact badly with a hand-rolled trap wrapping them — worth ruling in or out before assuming the bug is entirely in first-party code.
- **When did this start — is this a regression, or has the trap never actually released?** A regression points at a recent diff being the fastest path to the bug; "never worked" reframes this as a review of the original implementation rather than a bisect.

## Approach & Trade-offs

**I'd start by reproducing it myself and narrating exactly what I observe, rather than reading the code first and guessing.** Specifically: open the panel, Tab through every field once noting the order, then click Close and try to Tab again — is focus still visually inside the panel's DOM? Does `document.activeElement` (checked live in devtools) point somewhere inside the (possibly still-mounted, just visually hidden) panel? This single check usually immediately separates the two most common root causes: either the panel's Close button doesn't actually unmount/hide the trap's listener, or focus is being forced back into the panel by a stale effect that still thinks it's open.

**I'd treat "walk through the code that manages the trap's lifecycle" as the actual debugging target, not the Tab-handling logic itself** — a Tab/Shift+Tab boundary-wrapping implementation that's internally correct can still produce an unreleasable trap if the *mechanism that's supposed to remove it* is broken, which is a lifecycle bug, not a keyboard-logic bug. This maps directly onto the review checklist from the [Accessible Modal](01-accessible-modal-scenario.md) scenario's implementation: the `keydown` listener must be added when the trap activates and removed when it deactivates, and those two operations must be triggered by the *same* piece of state.

**On the WCAG framing specifically, I'd name 2.1.2 (No Keyboard Trap) explicitly and confirm this genuinely qualifies** — unlike the modal scenario, where the trap is *intentional* and satisfies 2.1.2 precisely because Escape provides a reliable exit, this bug report describes exactly the failure mode 2.1.2 exists to catch: keyboard focus becomes stuck with no standard way out. I'd also flag it as a likely **Level A** violation (the base conformance level, not AA or AAA) — meaning this isn't a nice-to-have polish item, it's a baseline accessibility blocker that should be treated with corresponding urgency, on par with a broken login page for a portion of the user base.

## Solution — Finding It

The buggy component, as it might arrive in a real bug report:

```tsx
function SettingsPanel({ isOpen, onClose }: { isOpen: boolean; onClose: () => void }) {
  const panelRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!isOpen) return;

    function onKeydown(e: KeyboardEvent) {
      if (e.key !== 'Tab' || !panelRef.current) return;

      const focusable = panelRef.current.querySelectorAll<HTMLElement>(
        'input, button, select, textarea, [tabindex]'
      );
      const first = focusable[0];
      const last = focusable[focusable.length - 1];

      // BUG: no boundary check — this fires and calls preventDefault()
      // on every single Tab press, not just at the first/last element.
      e.preventDefault();
      if (e.shiftKey) {
        last.focus();
      } else {
        first.focus();
      }
    }

    document.addEventListener('keydown', onKeydown);
    // BUG: no cleanup function returned — this listener is NEVER removed,
    // including after the panel closes.
  }, [isOpen]);

  return (
    <div ref={panelRef} hidden={!isOpen} role="dialog" aria-label="Settings">
      <input placeholder="Display name" />
      <input placeholder="Email" />
      <button onClick={onClose}>Close</button>
    </div>
  );
}
```

**Bug #1 — the boundary check is missing entirely.** Every Tab press — not just one at the last focusable element — calls `preventDefault()` and forces focus to `first`/`last`. This alone means the user can never Tab *through* the panel's fields in order at all; every press just bounces between the very first and very last element. This is already a keyboard trap even while the panel is legitimately open and the trap is "supposed" to exist — it's not correctly implementing "stay within the panel," it's implementing "you may only ever focus these two specific elements."

**Bug #2 — the `useEffect` never returns a cleanup function, so the `keydown` listener is added on every `isOpen` transition to `true` and never removed.** This is the one that makes the bug survive clicking Close: even once `isOpen` becomes `false` and the panel is visually hidden (`hidden={!isOpen}`), the listener added while it was open is still live on `document`, still finding `panelRef.current` (still mounted, just hidden — `hidden` doesn't unmount), and still hijacking every Tab press on the entire page, indefinitely, because nothing ever called `removeEventListener`.

Either bug alone would be a real problem; together, they compound: the trap is *both* internally broken (Bug #1, wrong even while legitimately active) *and* permanently un-removed (Bug #2, active forever). QA's report — "can never leave, even after Close" — is specifically Bug #2's signature; Bug #1 would have been reported differently ("can't tab through the fields properly") if it had shipped alone.

## The Fix

```tsx
useEffect(() => {
  if (!isOpen) return;

  function onKeydown(e: KeyboardEvent) {
    if (e.key !== 'Tab' || !panelRef.current) return;

    const focusable = Array.from(
      panelRef.current.querySelectorAll<HTMLElement>('input, button, select, textarea, [tabindex]')
    );
    if (focusable.length === 0) return;
    const first = focusable[0];
    const last = focusable[focusable.length - 1];

    // FIX: only intervene at the actual boundaries.
    if (e.shiftKey && document.activeElement === first) {
      e.preventDefault();
      last.focus();
    } else if (!e.shiftKey && document.activeElement === last) {
      e.preventDefault();
      first.focus();
    }
    // any other Tab press is left alone — native browser behavior handles it correctly
  }

  document.addEventListener('keydown', onKeydown);
  return () => document.removeEventListener('keydown', onKeydown); // FIX: cleanup on every effect re-run and unmount
}, [isOpen]);
```

Both fixes together: the boundary check restores normal Tab traversal within the panel (this is the actual "trap" doing its intended, narrow job — only intervening exactly at the two edges), and the cleanup function guarantees the listener's lifetime is scoped exactly to `isOpen === true`, so closing the panel via any path (the Close button, Escape if wired up, a parent unmounting it) correctly and immediately restores normal page-wide Tab behavior.

> **Check yourself:** The fixed version still has one more subtlety worth reasoning through: what happens if `isOpen` flips `true → false → true` rapidly (e.g., a double-click on something that toggles it)? Walk through whether the effect's cleanup-then-re-run sequence could ever leave two listeners attached simultaneously.

## Gotchas

**Fixating on the Tab-handling logic and missing the missing cleanup function.** The boundary-check bug is the more "interesting," more visible logic bug, and it's tempting to declare victory after fixing it — but Bug #2 alone is sufficient to reproduce exactly the bug report as filed ("even after Close"), and a candidate who fixes only Bug #1 has fixed a real but different problem than the one described.

**Assuming `hidden` (or `display: none`) on a container means its effects/listeners are automatically cleaned up.** It doesn't — `hidden` is a purely visual/rendering change; any `useEffect` and its side effects (including `document`-level listeners) keep running exactly as before unless explicitly torn down in response to the same state.

**Testing the fix only by tabbing through the panel while open, and not explicitly re-testing "close the panel, then try tabbing on the rest of the page."** The original bug report's most damning symptom is specifically about *post-close* behavior — a fix verified only against in-panel tabbing could pass that check while Bug #2 is still present.

**Not connecting this to the WCAG criterion by name when asked, or misidentifying which one.** This is squarely 2.1.2 (No Keyboard Trap), not 2.1.1 (Keyboard) — 2.1.1 is about all functionality being operable via keyboard *at all*; 2.1.2 is specifically about not getting keyboard focus stuck somewhere with no way out. Conflating the two, or not being able to cite either, is a gap worth being honest about rather than guessing under pressure.

## Follow-up Questions

**Q (High): Of the two bugs, which one alone would still be sufficient to trigger the exact symptom QA reported — "you can never leave, even after clicking Close" — and why?**

Answer: The missing cleanup function (Bug #2) alone is sufficient and is specifically what produces the *post-close* persistence QA describes — even with a perfectly correct boundary-checking trap, if the `keydown` listener is never removed when the panel closes, it stays live on `document` indefinitely and keeps intercepting Tab presses everywhere on the page, panel open or not. The missing boundary check (Bug #1) alone, without the cleanup bug, would produce a different, narrower symptom: the trap would be broken *while the panel is legitimately open* (bouncing between only the first and last field instead of cycling through all of them), but closing the panel and removing the listener would correctly restore normal behavior afterward — QA would have filed a different bug ("tabbing inside settings is broken," not "I can never leave").

The trap: treating both bugs as equally responsible for the reported symptom, or fixing the more visually obvious one (the boundary logic) and declaring the ticket resolved without separately verifying the post-close case, which is what the cleanup bug specifically affects.

---

**Q (High): Which WCAG success criterion does this violate, and how would you distinguish it, in conversation, from the intentional focus trap used correctly in the Accessible Modal scenario?**

Answer: This is 2.1.2, No Keyboard Trap, Level A. The distinguishing factor between this and the modal's intentional trap isn't "does focus get constrained" — both constrain focus similarly while active — it's whether there's a reliable, standard way out. The modal's trap satisfies 2.1.2 because Escape (and a reachable close control) always works to exit it; this bug violates 2.1.2 because the trap never releases at all, through any means, including the explicit Close button the user did click — the panel visually disappears, but the underlying keyboard-event capture keeps running, so there is no actual exit despite one being presented to the user.

The trap: describing this only as "a bug in the Tab handling" without naming the specific WCAG criterion it violates and its conformance level — a senior-level answer connects the observed bug to the exact accessibility standard it fails, since that's often what determines how urgently it needs to be triaged in a real backlog.

---

**Q (Medium): How would you have caught this before it reached QA — what in code review or automated testing would have flagged it?**

Answer: Code review: any `useEffect` that calls `addEventListener` without a corresponding cleanup function returned is a red flag worth a reviewer comment on sight, independent of what the listener does — this is a common enough class of bug (not specific to accessibility) that "does this effect clean up after itself" is a reasonable default review checklist item. For the boundary-check bug specifically, a code reviewer familiar with the focus-trap pattern (as covered in the [Accessible Modal](01-accessible-modal-scenario.md) scenario) would recognize the missing `document.activeElement === first/last` condition as deviating from the known-correct shape of this pattern. Automated: a targeted test simulating Tab presses and asserting `document.activeElement` stays within the panel while open, then simulating Close and asserting a further Tab press moves focus normally outside the panel, would catch both bugs directly — this is exactly the kind of interaction-simulating test suggested as a testing strategy in the modal scenario, and its absence here is arguably the deeper process gap, not just the code bug itself.

The trap: answering only "better code review" without naming the automated-test gap — a recurring, reproducible pattern bug like this is exactly the kind of thing that should have a regression test guarding it going forward, not just a one-time manual fix.

---

**Q (Medium): Suppose the panel contains a third-party rich-text editor embed, and the trap works correctly for the panel's own fields but Tab still occasionally escapes when focus is inside the embed. What would you investigate?**

Answer: I'd check whether the embed renders inside an `<iframe>` — if so, the parent document's `keydown` listener never sees events that originate inside the iframe at all (they're dispatched against the iframe's own separate document), so a trap implemented purely as a parent-document listener structurally cannot intercept Tab presses happening while focus is inside iframe content; the fix generally requires either the iframe's own content cooperating (posting a message when focus reaches its boundary) or restructuring to avoid framing the editor if that's not under your control. If it's not an iframe but a same-document embed with its own internal focus/keyboard handling (common in rich-text editors, which often manage complex internal Tab behavior like indenting), I'd check whether the embed calls `stopPropagation()` on its own `keydown` handling before the event reaches the document-level listener — if so, the trap's listener never fires for Tab presses that originate and get handled entirely within the embed's own DOM subtree, and the fix is usually coordinating with the embed's own focus-boundary APIs if it exposes any, rather than fighting its internal event handling from outside.

The trap: assuming a single document-level `keydown` listener is architecturally sufficient for any focus-trap scenario — third-party embeds, and especially iframes, are a real, common exception where the naive pattern silently doesn't apply, and recognizing that boundary is itself the signal being tested here.

---

**Q (Low): If you fixed the cleanup bug by using `AbortController` instead of manually calling `removeEventListener`, would that change anything about correctness here?**

Answer: Functionally equivalent for this case — `document.addEventListener('keydown', onKeydown, { signal })` paired with `controller.abort()` in the effect's cleanup achieves the same listener removal as calling `removeEventListener` directly, and some codebases prefer it as a slightly more modern idiom, particularly when a single cleanup needs to tear down multiple listeners at once (one `abort()` call instead of several `removeEventListener` calls). It doesn't change the underlying fix — the bug was the *absence* of any cleanup mechanism, not which specific API is used to implement one — so this is a stylistic/idiomatic choice, not a correctness difference, worth mentioning if asked but not something to over-index on as "the real fix."

The trap: treating this as a meaningfully different or superior fix rather than recognizing it's the same fix expressed with a different API — the substance of the bug (missing cleanup) and the substance of the fix (add cleanup) are unchanged either way.

---

## Self-Assessment

- [ ] Can identify both bugs (missing boundary check, missing effect cleanup) by reading the buggy code without hints
- [ ] Can explain why the missing cleanup function specifically produces the "even after Close" symptom from the bug report
- [ ] Can name WCAG 2.1.2 unprompted and articulate why this violates it while the modal's intentional trap does not
- [ ] Can describe what automated test would have caught this class of bug before it shipped
- [ ] Can reason through why a document-level `keydown` listener doesn't reach focus events happening inside an iframe
- [ ] Can write the corrected `useEffect` from memory, including the cleanup function

---
*Next: Designer Pushback on Color Contrast — How You Handle It — a behavioral scenario: the code is fine, the disagreement is with a stakeholder, and the skill being tested is negotiation grounded in the actual standard, not just knowing the standard.*
