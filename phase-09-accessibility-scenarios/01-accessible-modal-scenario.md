# Accessible Modal — Full Scenario Walkthrough

## Quick Reference

| WCAG Success Criterion | What It Demands | Where It's Satisfied |
|---|---|---|
| 4.1.2 Name, Role, Value | AT can identify the widget, its role, and its state | `role="dialog"`, `aria-modal="true"`, `aria-labelledby` |
| 2.4.3 Focus Order | Focus moves in a sequence that preserves meaning | Focus moves into the dialog on open, restores to trigger on close |
| 2.1.2 No Keyboard Trap | User can always Tab *out* of any component | The trap must have an intentional, working exit (Escape / close button) — an *unintentional* trap is itself the violation this criterion prohibits |
| 2.4.7 Focus Visible | The focused element has a visible indicator | Never suppress `:focus` styling inside the dialog, especially on the initially-focused element |
| 1.4.3 Contrast (Minimum) | Text/UI meets contrast ratios | Applies to the dialog chrome too — close buttons, backdrop text, not just body copy |

## The Scenario

"Build an accessible modal — focus trap, Escape to close, the works. Once you're done, I'm going to close my eyes, put on a screen reader, and try to use it the way a blind user actually would. Talk me through what you're doing as you go."

This is the same underlying component as [Accessible Modal With Focus Trap](../phase-02-component-machine-coding/04-accessible-modal-focus-trap.md) in Phase 2, but the interview shape is different in a way that matters: Phase 2 tests whether you can *build* the trap correctly under time pressure. This scenario tests whether you can build it **and** narrate your reasoning **and** survive having someone actually audit it live with assistive tech watching over your shoulder — which is a distinct, and arguably harder, skill. A lot of candidates who can write a correct focus trap go silent the moment a screen reader is turned on, because they've never actually listened to what one says.

## Clarifying Questions

- **What's actually triggering this modal — a destructive action, a form, or informational content?** Same reasoning as the Phase 2 scenario: it changes whether autofocusing the primary action button is safe or actively dangerous, and whether backdrop-click-to-close is a convenience or a data-loss risk. I ask this before touching the keyboard because it changes the initial-focus decision, not as an afterthought.
- **Which screen reader/browser combination should I test with, or should I pick one and say why?** NVDA+Chrome, JAWS+Chrome, and VoiceOver+Safari are the three combinations that matter in practice, and they don't always announce identically — I'd pick NVDA+Chrome by default (most common combination in accessibility testing, free, and closely follows spec behavior) and say so explicitly, rather than silently assuming the interviewer means whatever's installed on their machine.
- **Is there an existing design system component for dialogs, or am I building the primitive from scratch?** If a `Dialog` primitive already exists (Radix, React Aria, or an in-house equivalent), the right senior answer is usually "use it, don't hand-roll a second implementation" — I'd want to know this before writing code, since building a custom trap when a vetted one exists is often the wrong call in a real codebase, even if it's what the exercise wants to see.
- **Do you want me to build this with a hand-rolled `div` + manual trap, or is `<dialog>`/`showModal()` acceptable?** Worth surfacing up front since it changes how much of the implementation is "mine" to get right versus delegated to the browser — and some interviewers specifically want to see the manual trap to verify you understand the mechanism, not just that you can call a native API.

## Approach & Trade-offs

**The technical implementation doesn't change from Phase 2** — semantics (`role="dialog"`, `aria-modal="true"`, `aria-labelledby`), focus management (capture the trigger before moving focus, move focus in on open, restore on close), the Tab/Shift+Tab boundary trap, `inert` on background content, and configurable Escape/backdrop-dismiss. What changes here is that I treat **narration and verification as first-class parts of the deliverable**, not something bolted on after the code works.

**I build in a deliberate order that produces something testable at every step, rather than writing the whole thing silently and testing once at the end.** Semantics and markup first (testable by reading the accessibility tree in devtools, before any JS exists) → focus-in/focus-out (testable by tabbing) → the trap (testable by tabbing past the boundary) → backdrop/Escape (testable directly). At each step I say out loud what I expect a screen reader to announce, *then* verify it — narrating a prediction and confirming it beats narrating what already happened, because it shows the interviewer I have a working mental model of AT behavior, not just muscle memory for the ARIA attributes.

**I narrate the difference between what's visually obvious and what's only apparent with the accessibility tree open**, because that gap is the entire point of this exercise. A modal can look completely correct — backdrop dimmed, focus visibly on the Cancel button, Tab cycling correctly — while still being broken for a screen reader user if `aria-modal` is missing (background content still gets announced) or if `aria-labelledby` points at the wrong id (the dialog gets announced as "dialog" with no name). I make a point of opening the browser's accessibility tree inspector (Chrome DevTools' Accessibility pane) alongside the visual result, specifically because that's the artifact that reveals discrepancies a purely visual review would miss.

**On the "no keyboard trap" criterion (2.1.2) specifically, I say the name of the criterion out loud and explain why it's not the same thing as the focus trap I just built.** This is a common point of confusion worth defusing proactively: 2.1.2 prohibits *unintentional*, inescapable keyboard traps — it does not prohibit intentionally scoping focus within a modal. The distinction is whether there's a reliable, discoverable way out (Escape, a close button reachable by Tab) — my trap satisfies 2.1.2 precisely *because* Escape always works, not despite trapping Tab.

## Solution

The implementation is the one from [Accessible Modal With Focus Trap](../phase-02-component-machine-coding/04-accessible-modal-focus-trap.md), ported to React + TypeScript since that's the target stack from Phase 3 onward:

```tsx
function useFocusTrap(isOpen: boolean, containerRef: React.RefObject<HTMLElement>) {
  const triggerRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (!isOpen || !containerRef.current) return;

    triggerRef.current = document.activeElement as HTMLElement; // capture BEFORE moving focus
    const container = containerRef.current;

    const getFocusable = () =>
      Array.from(
        container.querySelectorAll<HTMLElement>(
          'a[href], button:not([disabled]), textarea:not([disabled]), input:not([disabled]), select:not([disabled]), [tabindex]:not([tabindex="-1"])'
        )
      ).filter((el) => el.offsetParent !== null);

    getFocusable()[0]?.focus();

    function onKeydown(e: KeyboardEvent) {
      if (e.key === 'Escape') return; // handled by the caller's onClose, not here
      if (e.key !== 'Tab') return;

      const focusable = getFocusable();
      if (focusable.length === 0) return;
      const first = focusable[0];
      const last = focusable[focusable.length - 1];

      if (e.shiftKey && document.activeElement === first) {
        e.preventDefault();
        last.focus();
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault();
        first.focus();
      }
    }

    document.addEventListener('keydown', onKeydown);
    return () => {
      document.removeEventListener('keydown', onKeydown);
      triggerRef.current?.focus(); // restore on unmount/close
    };
  }, [isOpen, containerRef]);
}

function Modal({ isOpen, onClose, titleId, children }: ModalProps) {
  const ref = useRef<HTMLDivElement>(null);
  useFocusTrap(isOpen, ref);

  useEffect(() => {
    if (!isOpen) return;
    document.body.style.overflow = 'hidden';
    return () => { document.body.style.overflow = ''; };
  }, [isOpen]);

  useEffect(() => {
    if (!isOpen) return;
    const onKeydown = (e: KeyboardEvent) => e.key === 'Escape' && onClose();
    document.addEventListener('keydown', onKeydown);
    return () => document.removeEventListener('keydown', onKeydown);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return createPortal(
    <>
      <div className="modal-backdrop" onClick={onClose} />
      <div ref={ref} role="dialog" aria-modal="true" aria-labelledby={titleId}>
        {children}
      </div>
    </>,
    document.body
  );
}
```

The React port changes one thing structurally worth calling out: focus-trap side effects live in `useEffect`, keyed to `isOpen`, with cleanup handling both listener removal *and* focus restoration — the cleanup function is what replaces the imperative `closeModal()` function's job from the vanilla version, since React unmounting/re-rendering is the trigger, not an explicit function call.

> **Check yourself:** If `document.body.style.overflow` and the focus-restoration both live in separate `useEffect`s keyed on `isOpen`, what ordering guarantee (or lack of one) do you have about which cleanup runs first when the modal closes — and does it matter here?

## Live Screen Reader Audit — What NVDA Actually Says

This is the part that separates "I implemented the pattern" from "I understand what the pattern is for." Walking through it out loud, with NVDA+Chrome as the reference:

**On open, correctly implemented:** *"Delete your account, dialog, This can't be undone, Cancel, button."* NVDA announces the accessible name (from `aria-labelledby`) and the role (`dialog`) immediately on focus entering it, then reads the first focused element. If `aria-modal="true"` is missing, NVDA's virtual cursor can still navigate *behind* the dialog into background content that's visually obscured — this is invisible in a sighted click-through and only shows up here.

**Tabbing through, correctly implemented:** each control is announced with its role and state (*"Cancel, button"* → *"Delete, button"*), and hitting Tab on the last control wraps silently back to the first — NVDA has no special announcement for the wrap itself, which is correct; the trap should be invisible/unremarkable when it's working, not narrated by the AT.

**A broken version, missing `aria-labelledby`:** NVDA announces just *"dialog"* with no further context — a screen reader user now knows *something* modal opened but not what it's asking them to do, and has to explore the content to find out, which is exactly the disorientation the pattern exists to prevent.

**A broken version, missing `inert`/`aria-hidden` on background content:** tabbing feels correct (the trap still works, since it's implemented independently), but using NVDA's browse-mode arrow-key navigation (not Tab — the virtual cursor) can still walk into and read background page content that's sitting invisibly behind the dialog, which is confusing in a way that's easy to miss if you only test with Tab and never touch NVDA's own navigation keys.

**On close, correctly implemented:** NVDA announces whatever the restored-focus element is (*"Delete account, button"*) — confirming, audibly, that focus genuinely landed back on the trigger and not on `<body>` (which would announce nothing, or the page's outermost landmark).

## Narrating This in the Interview

The meta-skill being evaluated: **saying what you expect before you demonstrate it**, not narrating after the fact. "I expect NVDA to announce the dialog's name and role the instant focus lands inside it, because `aria-labelledby` is wired to the heading — let's check" is a categorically stronger signal than building the whole thing in silence and then saying "see, it works" at the end. The former proves a mental model; the latter proves you can follow a checklist.

The other habit worth deliberately demonstrating: **treating the interviewer's live audit as new information, not a formality to get through.** If something doesn't announce the way predicted, say so immediately and reason about why, in real time, rather than glossing over it — "that's not what I expected, let me check the accessible-name computation" is a strong recovery; quietly moving on is a tell that you don't actually know what correct looks like.

## Gotchas

**Building the whole modal silently and only "showing" it at the end.** The exercise is explicitly testing live narration — going quiet for ten minutes and presenting a finished result skips the part being evaluated, even if the code is perfect.

**Confirming the visual result and treating that as sufficient.** A modal that looks correct (backdrop dimmed, focus ring visible on the right button) can still fail 4.1.2 if the accessible name is wrong — the only way to catch this is checking the accessibility tree or an actual screen reader, not eyeballing the UI.

**Conflating the intentional focus trap with the WCAG "no keyboard trap" violation** in conversation — if asked "doesn't this violate 2.1.2?" and the answer is a confused pause rather than the escape-hatch distinction, that's a real gap, not just an unlucky question.

**Not having an opinion on which screen reader/browser to test with.** "I don't know, whatever you have" is a weaker answer than picking NVDA+Chrome (or stating a reason for a different choice) and being able to say why that combination is a reasonable default.

## Follow-up Questions

**Q (High): How does this scenario differ from what you'd be evaluated on in a pure machine-coding round for the same component?**

Answer: A machine-coding round evaluates whether the code is correct — the trap wraps at both boundaries, focus restores, semantics are right. This scenario evaluates that *plus* two additional things: whether you can predict and explain assistive-tech behavior specifically (not just DOM/keyboard behavior), and whether you communicate your reasoning as you work rather than presenting a finished artifact. A candidate can pass the machine-coding version and still struggle here if they've never actually run a screen reader and listened to what it says — which is common, because it's easy to get the ARIA attributes "correct by pattern-matching a checklist" without ever verifying what they produce audibly.

The trap: assuming this is the same question asked twice — treating it as a code-only exercise and going silent, missing the narration and live-verification components entirely.

---

**Q (High): Explain the difference between an intentional focus trap (what you built) and the "no keyboard trap" violation WCAG 2.1.2 prohibits.**

Answer: 2.1.2 prohibits keyboard traps a user *cannot escape by standard means* — historically written with things like a Flash/Java applet embed that swallowed all keyboard input with no documented way out in mind. A modal's focus trap is intentionally scoping Tab navigation to the dialog's contents, but it's not a 2.1.2 violation as long as there's a reliable, standard, discoverable way to leave: Escape closing the dialog, and/or a close button reachable via Tab. The criterion isn't "focus may never be constrained," it's "focus may never be constrained *without an exit the user can find and use*." A modal that traps Tab correctly but has no Escape handling and no visible close button *would* be a genuine 2.1.2 violation.

The trap: answering as if any focus constraint is inherently the violation being described — the distinguishing factor is escapability, not constraint itself.

---

**Q (Medium): If `aria-modal="true"` doesn't by itself prevent a screen reader's virtual cursor from reading background content, what does, and why include `aria-modal` at all?**

Answer: `aria-modal="true"` is a declarative signal telling assistive tech "treat everything outside this element as inert for the duration" — in browsers/AT combinations with full support, it does correctly prevent the virtual cursor from wandering into background content. The *actual*, universally-reliable enforcement mechanism, though, is making the background genuinely inert — via the `inert` attribute (or `aria-hidden` + `pointer-events: none` as the older two-part equivalent) — which removes it from the accessibility tree and interaction surface regardless of whether a given AT fully honors `aria-modal` on its own. I'd include both: `aria-modal="true"` as the correct semantic declaration (some AT/browser combinations rely on it, and it's the spec-correct thing to declare), and `inert` on the background as the belt-and-suspenders mechanism that doesn't depend on a specific AT's interpretation of `aria-modal`.

The trap: treating `aria-modal="true"` alone as sufficient and skipping `inert`/`aria-hidden` on the background — support for `aria-modal`'s inert-background behavior has historically been inconsistent enough that relying on it exclusively is a real risk, not a theoretical one.

---

**Q (Medium): The interviewer's screen reader announces the dialog but doesn't read the body text ("This can't be undone") automatically — is that a bug?**

Answer: No — this is expected, correct behavior, and worth being able to say confidently rather than treating it as a red flag to fix. A screen reader announces the dialog's accessible name (from `aria-labelledby`) and role on focus entry, then whatever element focus actually lands on next (the first focusable control) — it does not automatically read the dialog's entire body content the way a page-load announcement might read a heading. If the body copy is important enough that it must be heard immediately without the user having to navigate to find it, the fix is including it in the accessible description via `aria-describedby` pointing at that paragraph, which most screen readers will announce alongside the name on focus — not assuming silence means something is broken.

The trap: assuming every visible string must be auto-announced, and "fixing" this by, say, using a live region to force-announce body copy — which is unnecessary here and would be a misuse of live regions (covered in [Live Region Announcements for Async Updates](03-live-region-async-announcements.md)) for content that isn't actually dynamic.

---

**Q (Low): Would you test with only one screen reader in a real project, or is that insufficient for production sign-off?**

Answer: One combination (say, NVDA+Chrome) is a reasonable *default working setup* during development for fast iteration, but I wouldn't sign off a production-bound component on a single AT/browser pairing — JAWS, NVDA, and VoiceOver have real behavioral differences in exactly the areas this scenario touches (how `aria-modal` inertness is honored, how live regions queue, how form-associated ARIA is announced), and a component that's flawless in NVDA can have a genuine bug specific to VoiceOver+Safari or JAWS. In practice I'd test the primary combination during implementation, then run at least a spot-check pass on the other two before calling something done, prioritized by the actual user base if analytics on AT usage are available.

The trap: treating "I tested with a screen reader" as a single binary checkbox rather than acknowledging that AT behavior meaningfully diverges across implementations, and cross-AT verification is part of what separates a demo-quality accessible component from a production one.

---

## Self-Assessment

- [ ] Can build the modal's focus trap correctly from memory, matching the Phase 2 implementation
- [ ] Can predict, before testing, what NVDA announces at open, during Tab traversal, and on close
- [ ] Can explain the distinction between an intentional focus trap and the WCAG 2.1.2 "no keyboard trap" violation without hesitating
- [ ] Can name a concrete bug (missing `aria-labelledby`, missing `inert`) and describe exactly what a screen reader does differently as a result
- [ ] Can narrate a prediction before verifying it, rather than narrating only after the fact
- [ ] Can explain why `aria-modal="true"` alone isn't sufficient without `inert`/`aria-hidden` on background content

---
*Next: Accessible Combobox — Full Scenario Walkthrough — same "build it, then I audit it live" format, applied to a pattern with a genuinely trickier ARIA story: `aria-activedescendant` versus moving real DOM focus.*
