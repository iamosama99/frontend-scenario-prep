# OTP / PIN Input

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Internal value model | Array of N single-char strings, joined on complete | Each `<input>` only ever holds 0 or 1 characters — the "real" value is the array, not any one box |
| Input type | `type="text"` + `inputmode="numeric"` + `pattern="[0-9]*"`, filtered in JS | `type="number"` has broken caret/selection behavior and lets `e`, `+`, `-`, `.` through — numeric-only has to be enforced at the JS layer regardless |
| Auto-advance / retreat | `input` handler moves focus forward on valid entry; `keydown` Backspace-on-empty moves focus back | Matches the physical mental model of "typing across boxes" without the user ever touching Tab |
| Paste handling | Single `paste` listener (usually on the first box), split the string, distribute across boxes, clamp to length | Users paste OTPs from a messages app constantly — per-box paste handling that only fills one box is the #1 broken-feel bug |
| Accessibility | `aria-label="Digit N of M"` per box, or a single hidden input with `autocomplete="one-time-code"` for SMS autofill | A visually segmented input is a UX pattern, not an accessibility requirement — screen reader and autofill needs can conflict with it |

## The Scenario

"Build a 6-digit OTP input component — six separate boxes, one digit each. Typing a digit should auto-advance to the next box, and when the user's done, I want a single callback with the full code. It should also handle pasting a code copied from a text message. Keep it plain JS, no framework — assume this needs to live inside a design system that doesn't have React yet."

## Clarifying Questions

- **Is the "value" six independent characters, or one 6-digit number/string?** This decides the data model up front. Treating it as six independent DOM inputs whose values happen to get joined is simpler and matches what's visually happening; treating it as "one value split across six views" invites over-engineering a two-way binding layer that plain JS doesn't need. I'd model it as an array of per-box characters internally, joined into a string only when reporting "complete" to the caller.
- **Should boxes accept only digits, or should the component be reusable for alphanumeric codes too?** OTPs are usually numeric, but some 2FA codes are alphanumeric. This affects both the `inputmode`/`pattern` hints and the JS-level regex filter. I'd default to digits-only per the prompt but keep the filter as a configurable regex rather than hardcoding `[0-9]`.
- **What happens on paste if the pasted string is shorter or longer than the box count, or has whitespace/dashes in it?** Real-world OTP texts often look like "Your code is: 482 991" with a space in the middle, or get copied with a trailing newline. Silently failing to fill anything is worse than being lenient — I'd strip non-matching characters first, then fill as many boxes as there are matching characters, truncating extra and leaving remaining boxes empty if it's short.
- **Does Backspace on an already-empty box need to move focus back, or only clear the current box?** This is the detail that separates a component that "feels right" from one that's technically functional — real OTP inputs (bank apps, 2FA prompts) retreat focus on a second Backspace once the current box is already empty, matching how you'd expect deleting to work across a single logical field.
- **Do we need `autocomplete="one-time-code"` for SMS autofill on mobile, and does that conflict with the six-separate-boxes visual design?** iOS/Android surface an SMS autofill suggestion above the keyboard keyed on `autocomplete="one-time-code"` on an input — but that attribute is most reliable on a *single* input that accepts the whole code, not distributed across six. This is worth flagging as a real design tension rather than silently picking one.

## Approach & Trade-offs

The central decision is what the "real" input is. Two viable designs:

**Design A — six real `<input>` elements, each holding one character**, wired together with JS to fake single-field behavior (auto-advance, backspace-retreat, shared paste handling). This is what's visually expected and what the prompt describes literally.

**Design B — one real, often visually-hidden input** (holding the whole code, with `autocomplete="one-time-code"` for SMS autofill) with six *decorative* boxes rendered from its value, and all keystrokes actually routed to the single hidden input. This is what production-grade OTP components (e.g., how many payment/banking flows implement it) often do, specifically because it makes SMS autofill and screen reader behavior trivial — there's only one real form control.

I'd build Design A live in an interview, since it's what's asked for and demonstrates the harder DOM-choreography skills the prompt is testing (focus management, paste splitting, per-box a11y), but I'd explicitly name Design B as the production-grade alternative and the reason a design system might prefer it: with six real inputs, autofill either doesn't fire at all or fires into the wrong box first, and screen readers announce six separate ambiguous "edit text" fields unless every one is carefully labeled.

Within Design A, three more decisions:

- **Numeric enforcement**: use `type="number"` on each box? No — number inputs suppress leading zeros oddly, let `e`/`+`/`-`/`.` through since those are valid in a `<input type=number>` per spec, and have inconsistent `selectionStart`/`selectionEnd` support (needed for "select the box's content on focus" and cursor logic) across browsers. `type="text"` with `inputmode="numeric"` (numeric keyboard on mobile, no visual difference on desktop) plus a JS regex filter on every keystroke is the reliable combination.
- **Advancing logic**: trigger on the `input` event (fires after the value actually changes, including on IME composition edge cases) rather than `keydown` (fires before the character lands, and doesn't reflect what was actually typed if the filter rejects it).
- **Paste distribution**: rather than wiring a `paste` handler to every box, attach one listener (event delegation on the container) so pasting into *any* box distributes the full string starting from that box's position — matching what users expect when they paste into the middle of a partially-filled code.

## Solution

Markup first — plain, semantic, no framework:

```html
<div class="otp-input" role="group" aria-label="Enter 6-digit verification code">
  <input class="otp-box" type="text" inputmode="numeric" pattern="[0-9]*"
         maxlength="1" aria-label="Digit 1 of 6" autocomplete="one-time-code" />
  <input class="otp-box" type="text" inputmode="numeric" pattern="[0-9]*"
         maxlength="1" aria-label="Digit 2 of 6" />
  <!-- ...boxes 3–6, same shape, aria-label incrementing -->
</div>
```

`autocomplete="one-time-code"` is left on the first box only — that's the one Safari/Chrome's SMS heuristics latch onto to offer the autofill suggestion; putting it on all six doesn't help and can confuse autofill about which box the suggestion should land in.

Core state and setup:

```javascript
function createOtpInput(container, { length = 6, onComplete } = {}) {
  const boxes = Array.from(container.querySelectorAll('.otp-box'));
  const digits = new Array(length).fill('');

  boxes.forEach((box, i) => {
    box.addEventListener('input', (e) => handleInput(e, i));
    box.addEventListener('keydown', (e) => handleKeydown(e, i));
    box.addEventListener('paste', (e) => handlePaste(e, i));
    box.addEventListener('focus', () => box.select()); // click a filled box, re-type over it
  });

  function checkComplete() {
    if (digits.every((d) => d !== '')) {
      onComplete?.(digits.join(''));
    }
  }
```

Filtering + auto-advance, the part that most directly answers "why not `type=number`":

```javascript
  function handleInput(e, i) {
    const raw = e.target.value;
    const clean = raw.replace(/[^0-9]/g, '').slice(-1); // last valid char typed
    e.target.value = clean;
    digits[i] = clean;

    if (clean && i < boxes.length - 1) {
      boxes[i + 1].focus();
    }
    checkComplete();
  }
```

Backspace-retreat and arrow-key navigation:

```javascript
  function handleKeydown(e, i) {
    if (e.key === 'Backspace' && !e.target.value && i > 0) {
      // current box already empty — retreat and clear the previous box too,
      // matching the mental model of deleting across one logical field
      digits[i - 1] = '';
      boxes[i - 1].value = '';
      boxes[i - 1].focus();
      e.preventDefault();
    } else if (e.key === 'ArrowLeft' && i > 0) {
      boxes[i - 1].focus();
    } else if (e.key === 'ArrowRight' && i < boxes.length - 1) {
      boxes[i + 1].focus();
    }
  }
```

Paste handling — the part that breaks in most first attempts:

```javascript
  function handlePaste(e, startIndex) {
    e.preventDefault();
    const pasted = (e.clipboardData || window.clipboardData).getData('text');
    const clean = pasted.replace(/\s+/g, '').replace(/[^0-9]/g, '');

    let boxIndex = startIndex;
    for (const char of clean) {
      if (boxIndex >= boxes.length) break; // longer than remaining boxes — truncate
      digits[boxIndex] = char;
      boxes[boxIndex].value = char;
      boxIndex++;
    }
    // focus the next empty box, or the last box if the paste filled everything
    boxes[Math.min(boxIndex, boxes.length - 1)].focus();
    checkComplete();
  }

  return { getValue: () => digits.join(''), reset: () => { /* clear all boxes + digits */ } };
}
```

> **Check yourself:** If a user pastes `"482 991"` (with a space) into the third box of a 6-box input, which boxes end up filled, and why does the space have to be stripped *before* indexing, not just skipped during the loop?

## Why Not `type="number"` — In Detail

This is worth a dedicated section because it's a near-guaranteed follow-up. `<input type="number">` looks tempting for a digit-only box, but breaks in ways that specifically matter for this component: it accepts `e`, `E`, `+`, `-`, `.` as valid interim characters per the HTML spec (they're part of valid floating-point/exponential syntax even though the field will reject the final value), so a naive filter that only runs `onchange` lets them through visibly before rejection. It also has inconsistent, often absent support for `.selectionStart`/`.select()` across browsers — needed for "select the existing character so a new keystroke overwrites it" (the `focus` handler above) — and some browsers render spinner arrows that visually clutter a 1-character box. `maxlength` is also ignored on `type="number"` in most browsers, so nothing natively stops someone from typing a 5-digit number into one box. `type="text"` sidesteps all of this and puts number-only enforcement fully under the component's control, at the cost of needing that JS filter — which the component needs anyway, to handle paste and IME edge cases.

## Gotchas

**Relying on `keydown` to detect the typed character.** `keydown` fires before the input's value updates, so `e.target.value` inside a `keydown` handler still reflects the *previous* state — auto-advance logic built on `keydown` either fires one keystroke late or requires reading `e.key` and re-deriving what the value will become, which is fragile against IME composition and mobile autocomplete/autocorrect insertion.

**Building auto-advance without handling the "typed a digit into an already-filled box" case.** If a user clicks back into box 2 (already containing a digit) and types a new digit, `maxlength="1"` combined with the raw browser value can produce a two-character string momentarily; the `.slice(-1)` in the filter (using the *last* typed character, not the first) is what makes overwrite-on-retype behave correctly instead of silently rejecting the new keystroke.

**Backspace-retreat clearing the wrong box.** A common bug: Backspace on an empty box clears and focuses the previous box, but the *next* keystroke lands as if that box were still selected without actually re-focusing it in some browsers unless `.focus()` is called synchronously in the same handler (calling it inside a `setTimeout` introduces a visible flicker and can lose the keystroke race on fast typers).

**Paste handled per-box instead of via delegation, only filling the box pasted into.** This is the single most common "looks done, isn't" bug — it satisfies a manual click-through test (typing digit by digit still works) but immediately breaks the moment someone pastes a full code, which is the primary real-world entry method for OTPs.

**Treating `aria-label="Digit N of 6"` as sufficient without testing with an actual screen reader.** Six separately-labeled inputs are technically accessible, but a screen reader user still has to navigate box-by-box and mentally reassemble the code — genuinely worse than one input they can paste into and have announced as a whole. This is why production components increasingly reach for Design B.

## Follow-up Questions

**Q (High): Why is `type="text"` with JS-level filtering preferred over `type="number"` for single-digit boxes, given that `type="number"` seems like the more "correct" semantic choice?**

Answer: `type="number"` accepts interim characters (`e`, `+`, `-`, `.`) that are valid partial input for a number per the HTML spec but meaningless for a single OTP digit, and browsers vary in whether/when they reject them. It also has unreliable `selectionStart`/`select()` support and often ignores `maxlength`, both of which this component depends on (select-on-focus for overwrite behavior, maxlength as a backstop against multi-character paste-into-one-box). `type="text"` plus `inputmode="numeric"` gives the numeric mobile keyboard (the actual UX win people are reaching for with `type="number"`) while keeping full, consistent control over validation in JS, which the component needs regardless for paste-splitting and IME handling.

The trap: candidates who reach for `type="number"` because "the value is digits" are pattern-matching on data type rather than on actual input behavior — a senior candidate has hit the caret/selection quirks in production and knows this from experience, not from first principles.

---

**Q (High): Walk through what has to happen when a user pastes a full 6-digit code into the third box of a partially-typed 6-box input.**

Answer: The `paste` event fires on box 3; the default paste behavior (which would just insert text at the cursor in that one box) must be prevented via `e.preventDefault()`, since native paste doesn't know about the other five boxes. The clipboard text is read via `e.clipboardData.getData('text')`, stripped of whitespace and non-digit characters, then distributed character-by-character starting at box index 2 (0-indexed), overwriting boxes 3 through however many characters fit, ignoring any earlier boxes 1–2 that already had values (or optionally overwriting from box 0 if the paste is meant to always replace the whole code — a judgment call worth stating explicitly). Focus moves to the next empty box, or the last box if the paste filled through the end. The completion callback fires only if all boxes end up filled.

The trap: handling paste only on the first box, or only appending from the box's own index without considering it should completely take over the fill sequence — either produces a paste that "sort of" works for the happy path (paste into box 1 with an exact-length code) but breaks for the equally common case of pasting mid-sequence or with extra characters.

---

**Q (High): How would you make this component's SMS/autofill behavior work correctly on mobile, and what's the fundamental tension with the six-visible-boxes design?**

Answer: Mobile SMS autofill (`autocomplete="one-time-code"`) works most reliably when it's attached to a single input that will receive the entire code as one autofill action — the OS reads the incoming SMS, matches a heuristic (recent message containing a code-like number near the word "code"), and offers to fill *one* field. With six separate single-character inputs, the OS has no natural target for a six-character autofill; behavior varies by platform and can silently not offer the suggestion at all, or fill only the first box. The most robust production pattern is Design B: a single real (visually hidden or zero-width) input carrying `autocomplete="one-time-code"` and `maxlength="6"` that receives the actual autofill and keystrokes, with the six visible boxes purely reflecting slices of that one input's value for display. This trades some implementation complexity (redirecting all real input to a proxy field, and manually painting the boxes) for materially better autofill and screen-reader behavior.

The trap: not knowing this design exists at all and insisting the six-separate-real-inputs version is "how OTP inputs are built" — it's how many are built, and it's a reasonable answer to a live-coding prompt, but a senior candidate should know it's not what mature production components use precisely because of this constraint.

---

**Q (Medium): How would you handle IME composition (e.g., a user with a non-Latin input method) typing into these boxes?**

Answer: During IME composition, `input` events can fire mid-composition with intermediate, non-final characters, and `compositionstart`/`compositionupdate`/`compositionend` events bracket the actual composition session. For a digits-only field this is mostly moot in practice (composed characters aren't digits, so the regex filter strips them anyway), but the auto-advance-on-input logic should ideally wait for `compositionend` before treating a value as final, to avoid advancing focus mid-composition on a value that's about to change. In an interview, flagging this without necessarily building a full composition-aware handler for a digits-only field is the right depth — over-engineering IME support for a 0–9 field would be solving a problem the input's own filter already handles.

The trap: either ignoring IME entirely with no acknowledgment (misses an accessibility/internationalization consideration some interviewers specifically probe for) or over-building a full composition state machine for a field that, once digit-filtered, doesn't actually need one.

---

**Q (Medium): The component currently calls `onComplete` every time `digits.every(d => d !== '')` becomes true. What happens if the user then edits a middle box after completion — should `onComplete` fire again, and does the current implementation handle that?**

Answer: As written, yes — any subsequent `input` event that leaves all boxes filled re-triggers `checkComplete()`, so editing box 3 after the code was already "complete" fires `onComplete` again with the new (possibly different) 6-digit string. Whether that's desired depends on the caller: if `onComplete` triggers an automatic form submission, firing again on every edit could cause a second premature submit attempt with a code the user was still correcting. A more careful design would debounce the completion callback slightly, or explicitly diff against the previously-reported value and only fire when it actually changes, or expose a separate explicit "submit" action rather than auto-firing on fill.

The trap: assuming "fires once when complete" is the actual current behavior without tracing through what happens on a post-completion edit — a candidate who hasn't thought this through will describe behavior the code doesn't actually have.

---

**Q (Medium): How would `maxlength="1"` alone, without the JS filter, fail to fully solve numeric-only input?**

Answer: `maxlength` constrains *character count*, not character *content* — it stops a fourth character from being typed but does nothing to stop a letter, symbol, or emoji from being that one character. It's a useful backstop against a paste inserting many characters into a single box's native value, but the actual "digits only" constraint has to be enforced by the JS regex filter on every `input` event; the two mechanisms solve different problems and both are needed.

The trap: treating `maxlength="1"` as if it implies "one digit" — it only implies "one character," full stop.

---

**Q (Low): How would you support a "6-digit code where the user can also see/toggle a masked vs. unmasked view," similar to a password field?**

Answer: Each box's actual character stays in the `digits` array regardless of display; a masked view swaps each box's rendered value to `•` (via a separate display-only overlay, or by toggling the input's own value between the real digit and a mask character while keeping the real value tracked in the JS state array, not in the DOM's `.value`, since the DOM can only hold one or the other at a time). Toggling "show" swaps all boxes back to their real values from the `digits` array in one pass. This is a reasonable extension but not something to over-build unprompted — it's worth mentioning as a natural next feature rather than implementing speculatively.

The trap: trying to derive the real value back out of a masked box's DOM value (`•`) — once masked, the DOM node itself no longer holds the real character, so the source of truth must already live in the JS array, not be reconstructed from the visible input.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can build the six-box auto-advance/retreat interaction from memory, including the empty-box Backspace edge case
- [ ] Can explain precisely why `type="number"` is the wrong choice here, citing at least two concrete failure modes
- [ ] Can implement full paste-splitting that correctly handles a paste shorter/longer than the box count and containing whitespace
- [ ] Can state the accessibility trade-off between six labeled boxes and a single hidden input with `autocomplete="one-time-code"`
- [ ] Can explain the SMS autofill implication of `autocomplete="one-time-code"` and why it favors a single real input

---
*Next: Image Carousel / Gallery — another compact, highly-interactive widget, this time built around a sliding viewport instead of discrete input boxes.*
