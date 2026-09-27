# Accessible Combobox / Dropdown

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Roles | `role="combobox"` on the `<input>`, `role="listbox"` on the popup, `role="option"` on each item | The ARIA 1.2 combobox pattern — three roles, one relationship |
| Virtual focus | `aria-activedescendant` on the input references the highlighted option's ID; real DOM focus never leaves the input | Lets typing keep working uninterrupted while arrow keys "move" the highlight |
| `aria-expanded` / `aria-controls` | On the input: whether the popup is open, and which element (the listbox) it controls | Standard disclosure wiring, same shape as tabs/accordion |
| Typeahead filtering | Filter the option list as the user types, update `aria-activedescendant` to the first match | Core combobox behavior — it's an editable filter, not just a styled `<select>` |
| Outside click / blur-before-click race | `mousedown` + `preventDefault()` on options, or check `relatedTarget`/use a blur timeout | `blur` fires before an option's `click` — naive close-on-blur breaks click-to-select |

## The Scenario

"Build an autocomplete/combobox — a text input that shows a filtered dropdown list of matching options as the user types. Clicking or pressing Enter on a highlighted option selects it and closes the dropdown. I want it properly accessible, and I don't want the classic bug where clicking an option doesn't work because the dropdown closes first."

## Clarifying Questions

- **Is this an editable combobox (user can type free text not in the list) or a "select-like" combobox (input just filters a fixed list, and the final value must be one of the listed options)?** This maps directly onto the `autocomplete` attribute's value in the ARIA pattern — `list` or `both` for editable/free-text, `none` for a strictly select-like variant — and it changes what happens on blur with no selection made (accept the typed text vs. revert/clear it).
- **Should the highlighted option move real DOM focus, or should focus stay in the input the whole time?** This is the crux of the whole component. The correct modern pattern (ARIA 1.2) keeps real focus in the input at all times and uses `aria-activedescendant` to communicate a "virtual" highlight to assistive tech — I'd confirm this is expected rather than an older pattern that actually moves focus into `role="option"` elements, since moving real focus breaks continued typing (you can't keep filtering by typing if focus has left the text field).
- **How many results can the list contain, and does filtering need to be debounced or is it synchronous against an in-memory array?** If matching requires a network call (server-side search), the whole interaction needs debouncing and a stale-response guard; if it's a synchronous filter over an in-memory list of a few hundred items, none of that async machinery is needed and adding it would be overengineering.
- **What should Enter do if no option is highlighted** — submit the typed text as-is, select the first visible match, or do nothing? This is an easy-to-miss edge case: a user who types a query and presses Enter immediately (before any arrow-key highlight) has an expectation, and "do nothing" silently swallowing their Enter press is a common, frustrating bug.
- **Does the dropdown need to close on outside click, on blur, on Escape, or all three — and should Escape clear the typed text or just close the popup?** Escape's two plausible behaviors (close popup only vs. close-and-revert) are both defensible; I'd default to "first Escape closes the popup and returns focus semantics to a plain text field, without clearing what's typed" since that matches most native OS autocomplete behavior.

## Approach & Trade-offs

**Virtual focus via `aria-activedescendant`, and why not just move real focus into the listbox.** The naive approach — pressing ArrowDown actually calls `.focus()` on the first `role="option"` element — breaks the interaction model immediately: once focus leaves the `<input>`, the user can no longer type to keep filtering (keystrokes now go to whatever the focused option element is, not the text field), and the browser's native text-editing affordances (cursor position, selection) are lost. The ARIA 1.2 combobox pattern solves this with "virtual focus": DOM focus never leaves the `<input>` for the entire interaction. Instead, `aria-activedescendant="option-id"` on the input tells assistive tech "the option with this ID is the one currently highlighted," and the visual highlight is applied with a CSS class, not real focus. Screen readers specifically understand and announce `aria-activedescendant` changes as if focus moved, even though, mechanically, it didn't — this is exactly the gap the attribute exists to bridge.

**Filtering strategy.** For an in-memory list, filter synchronously on every `input` event and re-render the `role="listbox"` children; there's no reason to debounce a pure client-side array filter over a reasonably-sized dataset. If matching requires a server round-trip, the interaction needs debouncing (see the debounce scenario) plus a mechanism to discard a response that arrives after a newer request has already been sent — otherwise a slow response for an earlier, now-stale query can overwrite the results of what the user is currently looking at. I'd tag each outgoing request with a monotonically increasing sequence number (or an `AbortController`) and only apply a response if it matches the most recent request sent, discarding anything older.

**The blur-vs-click race, and why naive close-on-blur breaks click-to-select.** The classic bug: implementing "close the dropdown when the input loses focus" via a `blur` listener seems reasonable, but `blur` fires on the input *before* a `click` event fires on the option the user is trying to click (the browser's event order is: `mousedown` → `blur` on the old focus target → `focus` (or nothing, given virtual focus) → `mouseup` → `click`). If the blur handler synchronously hides/removes the listbox, the subsequent `click` event on the now-hidden-or-removed option element never fires — the user's click on the item to select it appears to silently do nothing. Two real fixes: call `event.preventDefault()` inside a `mousedown` listener on each option — this prevents the input from ever actually blurring in the first place, since `preventDefault()` on `mousedown` stops the browser's default focus-change behavior, so the option's later `click` fires normally with the input still focused and the listbox still open; or, if `preventDefault`-on-`mousedown` isn't viable, delay the blur-triggered close slightly (a short `setTimeout`, or checking the blur event's `relatedTarget` to see if focus is moving to something *inside* the listbox before deciding to close) so the option's own click has a chance to run first. I'd default to the `mousedown`+`preventDefault` approach — it's simpler, has no timing assumption baked in, and is the standard fix referenced in the ARIA APG's own combobox example.

**Editable vs. select-like combobox (the `autocomplete` attribute values).** `autocomplete="list"` means the popup suggests values but the input's own free text is always accepted as the final value even if it matches nothing in the list — appropriate for something like a tag input or a city-name field where an unmatched value is still valid. `autocomplete="both"` additionally auto-completes the input's text inline with the first match as the user types (what's sometimes called "inline autocomplete," visible as browser-address-bar-style greyed-out completed text). `autocomplete="none"` is the strictly select-like variant, where the final value *must* be one of the listed options — the input behaves more like a searchable `<select>`, and blurring without an explicit selection should revert to the last valid value or clear the field, rather than accepting arbitrary typed text.

## Solution

Markup:

```html
<label for="fruit-input">Favorite fruit</label>
<input
  id="fruit-input"
  role="combobox"
  aria-expanded="false"
  aria-controls="fruit-listbox"
  aria-autocomplete="list"
  autocomplete="off"
/>
<ul id="fruit-listbox" role="listbox" hidden></ul>
```

`aria-autocomplete="list"` (a separate attribute from the HTML `autocomplete` attribute, confusingly) tells assistive tech that typing produces a filtered list of suggestions — distinct from `"both"`, which would additionally imply inline text completion.

State and rendering, built incrementally. First, filtering and rendering the listbox:

```javascript
const input = document.getElementById('fruit-input');
const listbox = document.getElementById('fruit-listbox');
const allOptions = ['Apple', 'Apricot', 'Banana', 'Blueberry', 'Cherry', 'Cranberry'];

let activeIndex = -1;
let visibleOptions = [];

function renderOptions(query) {
  visibleOptions = allOptions.filter((o) => o.toLowerCase().includes(query.toLowerCase()));
  activeIndex = -1;

  listbox.innerHTML = '';
  visibleOptions.forEach((opt, i) => {
    const li = document.createElement('li');
    li.id = `option-${i}`;
    li.role = 'option';
    li.textContent = opt;
    // preventDefault on mousedown so the input never blurs before click fires
    li.addEventListener('mousedown', (e) => e.preventDefault());
    li.addEventListener('click', () => selectOption(i));
    listbox.appendChild(li);
  });

  const isOpen = visibleOptions.length > 0;
  listbox.hidden = !isOpen;
  input.setAttribute('aria-expanded', String(isOpen));
}
```

Typing triggers filtering; `input` events don't touch `aria-activedescendant` until a highlight actually exists:

```javascript
input.addEventListener('input', () => renderOptions(input.value));
```

Keyboard handling — this is where virtual focus is actually maintained (note: no `.focus()` call ever targets an option):

```javascript
function setActive(index) {
  activeIndex = index;
  listbox.querySelectorAll('[role="option"]').forEach((el, i) => {
    el.setAttribute('aria-selected', String(i === index));
  });

  if (index === -1) {
    input.removeAttribute('aria-activedescendant');
  } else {
    // This is the ENTIRE mechanism — DOM focus never leaves `input`.
    input.setAttribute('aria-activedescendant', `option-${index}`);
    listbox.children[index].scrollIntoView({ block: 'nearest' });
  }
}

function selectOption(index) {
  input.value = visibleOptions[index];
  closeListbox();
}

function closeListbox() {
  listbox.hidden = true;
  input.setAttribute('aria-expanded', 'false');
  input.removeAttribute('aria-activedescendant');
}

input.addEventListener('keydown', (e) => {
  if (e.key === 'ArrowDown') {
    e.preventDefault();
    if (listbox.hidden) renderOptions(input.value); // reopen if closed
    setActive(Math.min(activeIndex + 1, visibleOptions.length - 1));
  } else if (e.key === 'ArrowUp') {
    e.preventDefault();
    setActive(Math.max(activeIndex - 1, 0));
  } else if (e.key === 'Home' && !listbox.hidden) {
    e.preventDefault();
    setActive(0);
  } else if (e.key === 'End' && !listbox.hidden) {
    e.preventDefault();
    setActive(visibleOptions.length - 1);
  } else if (e.key === 'Enter') {
    if (activeIndex !== -1) {
      e.preventDefault();
      selectOption(activeIndex);
    }
    // else: no highlight — let Enter behave as a normal text-field submit
  } else if (e.key === 'Escape') {
    closeListbox();
  }
});

// Outside click closes the popup; mousedown+preventDefault on options above
// means this fires only for genuinely-outside clicks, not option clicks.
document.addEventListener('click', (e) => {
  if (!e.target.closest('#fruit-input, #fruit-listbox')) closeListbox();
});
```

> **Check yourself:** Walk through the exact event order when a user clicks a visible option — `mousedown` on the option, then what, in what order — and explain precisely which step the `preventDefault()` call interrupts.

## Gotchas

**Moving real DOM focus into the listbox on ArrowDown.** This is the single most common wrong instinct — it "looks right" in a demo (the highlighted option visually looks focused) but silently breaks continued typing, since keystrokes now go to a list item, not the text input, so the user can't keep refining their filter with the arrow-highlighted list open.

**Naive close-on-blur breaking click-to-select.** Covered at length above — this is specifically the bug the scenario prompt calls out, and it's the interviewer's signal that they want to see the `mousedown`-`preventDefault` (or equivalent `relatedTarget`/timeout) fix, not just "add a blur listener."

**Enter with no option highlighted doing nothing.** If the keydown handler only handles Enter inside an `if (activeIndex !== -1)` branch and has no `else`, a user who types a full query and immediately presses Enter (a very natural first interaction, before ever touching an arrow key) gets no feedback at all — the event is silently swallowed instead of, at minimum, allowing default behavior (form submission) to proceed.

**Forgetting to clear `aria-activedescendant` when the list closes or is filtered down to nothing.** A stale `aria-activedescendant` pointing at an option ID that no longer exists in the DOM (because filtering removed it) is invalid ARIA state — assistive tech has nothing to resolve the reference to, silently breaking the announcement of "what's currently highlighted" until the next valid update.

**Stale async results overwriting fresh ones (server-backed variant).** Without a sequence-number or `AbortController`-based guard, a slow response to an earlier keystroke can arrive after a faster response to a later keystroke and overwrite the currently-displayed (correct, newer) results with stale ones — a real, common production bug in any debounced/async-filtered combobox.

## Follow-up Questions

**Q (High): Explain `aria-activedescendant` and why real DOM focus stays on the input the entire time instead of moving into the listbox.**

Answer: `aria-activedescendant`, set on the currently-focused element (the input), references the ID of another element (an option) that should be treated as "virtually" focused/active for the purposes of assistive tech announcement — the browser and screen reader announce it as though focus moved to that option, without DOM focus (`document.activeElement`) ever actually changing. This exists specifically because moving real focus into a list item would break the ability to keep typing into the input to refine the filter — once `document.activeElement` is a `<li role="option">`, keystrokes go to that list item, not the text field, so the entire "type to filter, arrow to highlight" interaction collapses. Virtual focus keeps `document.activeElement` pinned to the `<input>` for the whole interaction, and `aria-activedescendant` is purely a communication channel to assistive tech about which option is "highlighted" right now.

The trap: describing `aria-activedescendant` as "a way to focus another element" — it explicitly does NOT move focus; conflating the two is exactly the misunderstanding that leads candidates to call `.focus()` on options and break continued typing.

---

**Q (High): Walk through the blur-before-click race condition and both ways to fix it.**

Answer: When a user clicks a dropdown option, the browser fires events in this order: `mousedown` on the option → (if nothing intervenes) the input loses focus, firing `blur` on it → `mouseup` → `click` on the option. If a `blur` handler on the input synchronously hides or removes the listbox (a natural first instinct for "close dropdown when it loses focus"), the option element is gone (or hidden) by the time the browser gets to firing `click` on it — so the click that was supposed to select the option never has anything to land on, and selection silently fails. Fix 1 (preferred): attach a `mousedown` listener to each option that calls `event.preventDefault()` — this specifically suppresses the browser's default "move focus" behavior that `mousedown` would otherwise trigger, so the input never actually blurs, the listbox stays open, and the subsequent `click` fires normally on the still-present option. Fix 2: let blur happen, but don't act on it immediately — either delay the close with a short `setTimeout` so the option's `click` has a chance to fire first, or inspect the blur event's `relatedTarget` (the element about to receive focus) and skip closing if it's inside the listbox.

The trap: proposing "just don't close on blur" as the fix — that reintroduces a different, worse bug (the dropdown never closes when the user clicks elsewhere on the page or tabs away), rather than actually resolving the ordering conflict between blur and the option's click.

---

**Q (High): What should happen when the user presses Enter with no option highlighted, versus with one highlighted?**

Answer: With an option highlighted (`activeIndex !== -1`), Enter should commit that option as the selected value and close the popup, exactly like a click on it — the keyboard and mouse paths to "select this option" should converge on the same underlying `selectOption` function. With no option highlighted — a user who typed a query and pressed Enter immediately, without ever touching an arrow key — the combobox shouldn't silently swallow the keypress; the correct behavior depends on the combobox's mode: for a free-text-allowed combobox (`autocomplete="list"`/`"both"`), Enter should let default behavior proceed (submit the form, or simply accept the typed text as-is, since it's a valid value even unmatched); for a strictly select-like combobox (`autocomplete="none"`), Enter with nothing highlighted might reasonably select the first visible match instead, or do nothing if that would be surprising — either is defensible, but "do nothing and give no signal" is the one wrong answer, since the user gets no feedback that their input was ignored.

The trap: handling Enter only inside the "if activeIndex !== -1" branch with no consideration of the else case at all — this is the single most common incomplete implementation, and it's a very natural gap to have if the candidate is testing primarily via arrow-key-then-Enter flows and never tries "type and immediately hit Enter."

---

**Q (Medium): What's the difference between the `autocomplete="none"`, `"list"`, and `"both"` combobox variants, and how does each change blur/commit behavior?**

Answer: These are values of the ARIA `aria-autocomplete` attribute (distinct from, though related in spirit to, the HTML `autocomplete` attribute) describing how aggressively the combobox suggests/completes input. `"none"` means no suggestions augment typing at all beyond a plain text field (rare for what's actually called a "combobox" in practice, since the popup list itself is the whole point) — often used loosely to describe a strictly select-like combobox where the final value must match a listed option, reverting or clearing on blur otherwise. `"list"` means a popup of suggestions appears as the user types, but the input's own typed text remains a valid value in its own right even if unmatched — appropriate for free-text-with-suggestions fields like a tag input or city name. `"both"` adds inline autocompletion on top of the list — the browser fills in the remainder of the first matching option directly into the input (typically shown as selected/highlighted trailing text) as the user types, in addition to showing the popup. The variant chosen changes what should happen on blur with nothing explicitly selected: a `"list"`/`"both"` combobox accepts the typed text as-is; a strictly select-like (`"none"`-style, in the informal sense) combobox should revert to the last valid selected value or clear the field, since an unmatched value isn't a legitimate final state for that variant.

The trap: treating all three as interchangeable/cosmetic differences in how suggestions are displayed — the variant actually changes what counts as a *valid final value*, which changes blur-handling logic, not just the popup's visual behavior.

---

**Q (Medium): How would you make this combobox work against a server-side search endpoint instead of an in-memory array, without race conditions?**

Answer: Debounce the `input` handler so a fast typist doesn't fire a request per keystroke (see the debounce scenario for the mechanism), then, for each request actually sent, either use an `AbortController` to cancel the previous in-flight request when a new one starts, or tag each request with a monotonically increasing sequence number and, on response, compare it against the latest sequence number sent — only apply the response to the UI if it matches the most recent request; otherwise discard it silently. Both approaches solve the same problem: a slower response to an earlier keystroke arriving *after* a faster response to a later keystroke, which without a guard would overwrite the correct, current results with stale ones from a query the user has already moved past.

The trap: relying on debounce timing alone ("the requests are spaced out, so they'll always resolve in order") — network response times aren't guaranteed to preserve request order under real-world conditions (retries, connection pooling, server-side load variance), so debouncing reduces the *frequency* of the race but doesn't eliminate the possibility of it; an explicit ordering guard is still required for correctness.

---

**Q (Medium): Why use `<li role="option">` inside a `<ul role="listbox">` rather than `<div>`s for the popup structure?**

Answer: Functionally, ARIA roles override the implicit semantics either element type would otherwise carry, so a `<div role="option">` and an `<li role="option">` are both exposed identically to assistive tech once the role is applied — there's no accessibility difference. The reason to still prefer `<ul>`/`<li>` is that it degrades more sensibly and reads more naturally as plain HTML/CSS if roles were ever stripped or misapplied, and it matches the semantic shape of "a list of things" that a listbox conceptually is, which is a defensible authoring convention even though it isn't a hard technical requirement the way the roles themselves are.

The trap: claiming `<ul>/<li>` is *required* by the ARIA spec for a listbox — it isn't; ARIA roles are element-agnostic, and a from-scratch combobox implementation with `<div>`s throughout, correctly roled, is equally valid to a screen reader.

---

**Q (Low): How does this ARIA 1.2 "virtual focus" combobox pattern differ from the older ARIA 1.0 combobox pattern that some legacy codebases still use?**

Answer: The older (ARIA 1.0-era, now deprecated) pattern for some combobox-like widgets actually moved real DOM focus around, or relied on a differently-shaped role structure (e.g., `role="combobox"` wrapping a `role="textbox"` rather than the role living directly on the `<input>`), and had inconsistent cross-screen-reader support because the spec itself was less settled. ARIA 1.2 standardized the current pattern described throughout this file: `role="combobox"` directly on the input, `aria-activedescendant` for virtual focus, `role="listbox"`/`role="option"` for the popup, and clearer rules for `aria-expanded`/`aria-controls`. Modern implementations (including this one) should target the 1.2 pattern; legacy combobox code predating it is a common source of inconsistent screen reader behavior across NVDA/JAWS/VoiceOver that gets "fixed" simply by migrating to the current pattern.

The trap: treating this as purely a historical trivia question — in practice it's relevant because a candidate maintaining or reviewing an older accessible-combobox implementation should recognize outdated patterns (real focus movement, wrapped `textbox` roles) as a red flag worth migrating away from, not just interesting history.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can wire up `role="combobox"`, `role="listbox"`, `role="option"`, `aria-expanded`, `aria-controls` from memory
- [ ] Can explain virtual focus via `aria-activedescendant` and why real focus must stay on the input
- [ ] Can explain and fix the blur-before-click race with `mousedown` + `preventDefault`
- [ ] Can implement ArrowUp/ArrowDown/Home/End navigation that updates the virtual highlight, not real focus
- [ ] Can explain the difference between `autocomplete="none"/"list"/"both"` and how each changes blur/commit behavior
- [ ] Can describe a stale-response guard for a server-backed combobox (sequence number or `AbortController`)

---
*Next: Multi-step Form Wizard With Validation — moving from single-widget interaction patterns to multi-screen state and validation flow.*
