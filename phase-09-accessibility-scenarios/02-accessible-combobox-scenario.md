# Accessible Combobox — Full Scenario Walkthrough

## Quick Reference

| Decision | Options | What I'd Pick and Why |
|---|---|---|
| How the active suggestion is tracked | Move real DOM focus into the listbox, vs. keep focus in the input and track "active" via `aria-activedescendant` | `aria-activedescendant` — real focus movement closes the mobile keyboard/loses the IME composition and breaks continued typing |
| Listbox visibility | `hidden` attribute / conditional render vs. `display: none` via class | Conditional render or `hidden` — same result, but must be paired with `aria-expanded` staying in sync |
| Announcing result count | Visual only vs. a live region announcing "N results available" | A polite live region — sighted users see the count on screen; screen reader users get nothing without an explicit announcement |
| Selecting a suggestion | Mouse click only vs. click + Enter + explicit "select" semantics | Both, and Escape must revert to the pre-search value, not just close the list |

## The Scenario

"Build a combobox — type to filter a list of suggestions, arrow keys move through them, Enter selects. Same as before: build it, narrate it, and then I'm going to test it with a screen reader and tell you what's wrong if I find something."

Same format as [Accessible Modal — Full Scenario Walkthrough](01-accessible-modal-scenario.md), applied to a pattern that trips up more candidates, because the correct-vs-intuitive implementations genuinely diverge here in a way the modal doesn't. The intuitive approach — move DOM focus down into the list as the user arrows through it — is *wrong* for a text-input-driven combobox, and most candidates who haven't specifically studied the ARIA combobox pattern reach for it instinctively. The machine-coding version of this same component lives in [Accessible Combobox / Dropdown](../phase-02-component-machine-coding/07-accessible-combobox-dropdown.md) in Phase 2 — this scenario assumes that implementation and focuses on the parts a live screen-reader audit specifically surfaces.

## Clarifying Questions

- **Is this a "select one from a constrained list" combobox (an editable select, essentially) or a free-text search-as-you-type that happens to offer suggestions?** This changes what Escape and blur-without-selecting should do — a constrained-choice combobox arguably shouldn't accept arbitrary free text as the final value, while a search box should. I'd want this settled before deciding whether an unmatched typed value is valid on submit.
- **Should the list filter as-you-type (every keystroke) or only after a debounce/explicit trigger?** Not just a performance question — it affects how often `aria-expanded`/result-count live-region updates fire, and a live region that re-announces on every keystroke of a fast typist is actively unusable, so the debounce decision and the announcement-throttling decision are linked.
- **Does arrow-down from the input, before any typing, open the list and show all options, or does the list only appear once there's a query?** A common real-world requirement (a "type or pick" combobox behaving like a searchable select) — worth confirming since it changes the empty-state keyboard behavior.
- **What should Tab do while the list is open — select the highlighted item, or just close the list and move focus to the next field like it would anywhere else?** Different comboboxes I've used in production disagree on this, and it's exactly the kind of underspecified edge case a real audit would flag if not decided deliberately.

## Approach & Trade-offs

**The core decision, and the one most candidates get wrong on first instinct: keep real DOM focus in the `<input>` at all times; track the "active" suggestion with `aria-activedescendant` pointing at the id of the currently-highlighted option, not by moving focus into the listbox.** The intuitive alternative — pressing ArrowDown moves actual focus onto the first `<li>`, ArrowDown again moves it to the next — breaks the interaction model this pattern exists to support: the user should be able to keep typing to refine the filter *while* arrowing through suggestions, and if focus has physically left the input, typing does nothing (or, worse, is interpreted by whatever now has focus). `aria-activedescendant` solves this by keeping focus in the input, where typing always works, while the input's `aria-activedescendant` attribute tells assistive tech which descendant *element* (by id, in the listbox) is the "virtually focused" one for announcement purposes — the browser/AT announces that element's content without focus ever actually leaving the input.

**`aria-expanded` on the input must be kept perfectly in sync with whether the listbox is actually visible/populated, not just toggled optimistically.** A combobox that sets `aria-expanded="true"` on every keystroke regardless of whether any results exist tells a screen reader user a list is open when it's actually empty — I'd gate it strictly on "does the listbox currently have content the user could select," and handle the zero-results case with a distinct, explicitly announced empty state rather than an ambiguously-expanded-but-empty list.

**Announcing the result count needs a dedicated live region, separate from the listbox itself**, because the listbox's own content changing doesn't get automatically announced by most AT the way a live region does — a sighted user glances at the list and sees "3 results" implicitly; a screen reader user gets nothing unless something explicitly says so. I'd add a visually-hidden `aria-live="polite"` region that updates with a short count string ("5 results available") — and specifically debounce/throttle this alongside the filtering itself, because a live region firing on every keystroke of a fast typist produces a garbled, overlapping stream of announcements that's worse than no announcement at all.

**Selecting via Escape needs to distinguish "close the list" from "clear my typed input" — these are different user intents that a naive implementation conflates.** I'd make Escape's first press close the open list and revert the input to the last *confirmed* value (not necessarily what was typed, if nothing's been selected yet) rather than clearing the field outright — clearing on Escape destroys a partially-typed query the user may have wanted to keep editing, which is a real, reported UX complaint on comboboxes that get this wrong.

## Solution

Building on the roving-highlight (not roving-focus) model, in React + TypeScript:

```tsx
function Combobox({ options, onSelect }: ComboboxProps) {
  const [query, setQuery] = useState('');
  const [isOpen, setIsOpen] = useState(false);
  const [activeIndex, setActiveIndex] = useState(-1);
  const listboxId = useId();
  const inputRef = useRef<HTMLInputElement>(null);

  const filtered = useMemo(
    () => options.filter((o) => o.label.toLowerCase().includes(query.toLowerCase())),
    [options, query]
  );

  function onKeyDown(e: React.KeyboardEvent) {
    if (e.key === 'ArrowDown') {
      e.preventDefault();
      setIsOpen(true);
      setActiveIndex((i) => Math.min(i + 1, filtered.length - 1)); // focus STAYS in the input
    } else if (e.key === 'ArrowUp') {
      e.preventDefault();
      setActiveIndex((i) => Math.max(i - 1, 0));
    } else if (e.key === 'Enter' && activeIndex >= 0) {
      e.preventDefault();
      selectOption(filtered[activeIndex]);
    } else if (e.key === 'Escape') {
      if (isOpen) { setIsOpen(false); setActiveIndex(-1); } // first Escape: close, don't clear
      else setQuery(''); // second Escape (list already closed): clear
    }
  }

  function selectOption(option: Option) {
    setQuery(option.label);
    setIsOpen(false);
    setActiveIndex(-1);
    onSelect(option);
    inputRef.current?.focus(); // no-op if already focused, but explicit — never let it drift
  }

  const activeOptionId = activeIndex >= 0 ? `${listboxId}-option-${activeIndex}` : undefined;

  return (
    <div>
      <input
        ref={inputRef}
        role="combobox"
        aria-expanded={isOpen && filtered.length > 0}
        aria-controls={listboxId}
        aria-activedescendant={activeOptionId} // the AT-visible "active" pointer — focus never leaves this input
        aria-autocomplete="list"
        value={query}
        onChange={(e) => { setQuery(e.target.value); setIsOpen(true); setActiveIndex(-1); }}
        onKeyDown={onKeyDown}
      />
      <div aria-live="polite" className="visually-hidden">
        {isOpen && `${filtered.length} result${filtered.length === 1 ? '' : 's'} available`}
      </div>
      {isOpen && filtered.length > 0 && (
        <ul id={listboxId} role="listbox">
          {filtered.map((opt, i) => (
            <li
              key={opt.id}
              id={`${listboxId}-option-${i}`}
              role="option"
              aria-selected={i === activeIndex}
              onClick={() => selectOption(opt)}
            >
              {opt.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

> **Check yourself:** Explain, specifically, what would break in the typing-while-arrowing interaction if `ArrowDown`'s handler called `.focus()` on the list item instead of just updating `activeIndex`.

## Live Screen Reader Audit — What NVDA Actually Says

**On typing, correctly implemented:** NVDA reads each typed character as usual (it's a normal text input, nothing special there), and the visually-hidden live region announces *"3 results available"* shortly after — appearing to the user as a natural pause after typing, not an interruption mid-keystroke, assuming the debounce is tuned reasonably (roughly 300–500ms is a common range that avoids both keystroke-interrupting chatter and a noticeably stale count).

**Arrowing down, correctly implemented:** NVDA announces the option's text and its position — *"Chicago, 1 of 3"* — even though DOM focus never moved. This is `aria-activedescendant` working as intended: the browser computes the accessible-focus target from the attribute and NVDA announces *that*, not whatever element literally has `document.activeElement`.

**A broken version, using real DOM focus movement instead:** the moment ArrowDown moves focus onto the `<li>`, the on-screen keyboard on a touch device dismisses (a real, reported mobile bug class), and — more relevant to a desktop screen-reader audit — continuing to type after arrowing does nothing, because the input no longer has focus to receive the keystrokes; NVDA would announce the `<li>`'s content correctly on that first arrow press, but the interaction is now broken for anyone trying to keep refining their query, which is the whole point of a combobox over a plain `<select>`.

**A broken version, missing `aria-expanded` sync:** NVDA announces the input as *"combobox, expanded"* even when zero results match and no listbox is actually rendered — telling the user there's a list to explore when there isn't one, which sends them arrowing into nothing and, without an announced empty state, gives no explanation why.

**On selecting an option via Enter, correctly implemented:** NVDA announces the input's new value change (the confirmed selection now sitting in the input) — the same behavior as if the user had typed that exact text themselves, which is precisely why writing the selected label into the input's `value` (rather than some separate "selected item" display) is the right mental model: to assistive tech, a combobox selection and manually typing the equivalent text should announce indistinguishably.

## Narrating This in the Interview

The one prediction worth stating out loud *before* writing any code, because it's the single biggest tell of whether the candidate actually understands the pattern versus is pattern-matching ARIA attribute names from memory: **"I'm keeping focus in the input the whole time and using `aria-activedescendant` to point at the highlighted option — if I moved real focus into the list instead, typing to keep filtering would stop working the moment you press an arrow key."** Saying this before building it, then demonstrating it live by typing, arrowing, and continuing to type, is a much stronger signal than building the correct version and only explaining the choice if asked.

## Gotchas

**Moving real DOM focus into the listbox on arrow-key press.** The most common mistake on this exact pattern — it looks correct in isolation (the highlighted item does get focus, does get announced) and only reveals itself as broken when the user tries to keep typing while browsing suggestions, which is core to why comboboxes exist over plain selects.

**`aria-expanded` toggling independent of whether the list actually has renderable content.** Produces a state where AT announces "expanded" over an empty or invisible list — a subtle enough bug that it's easy to ship without ever noticing in a purely visual QA pass.

**A live region that fires on every keystroke without debouncing.** Produces overlapping, garbled announcements for anyone typing at a normal pace — worse than silence, because it actively interrupts and competes with whatever else NVDA is trying to say (like echoing the typed character itself).

**Escape clearing the input instead of first closing the list.** Destroys a user's in-progress query on the first press of a key whose primary, expected job in almost every other UI is "back out one step," not "wipe everything."

**Not handling the zero-results state with an explicit, announced message.** An empty listbox that simply doesn't render leaves a screen reader user with no confirmation that their query genuinely matched nothing, versus something being broken — a visually-hidden "No results for 'xyz'" string, ideally in the same live region used for the count, closes this gap.

## Follow-up Questions

**Q (High): Why does `aria-activedescendant` require the input to keep real DOM focus, and what specifically breaks if focus moves into the listbox instead?**

Answer: `aria-activedescendant` is a mechanism for saying "focus is logically here, on this input, but treat *this other element* as the currently-active one for announcement and highlighting purposes" — it only makes sense, and only works, if the element wearing the `aria-activedescendant` attribute (the input) is the one that actually has `document.activeElement`. If real focus moves onto a list item instead, two things break: the browser stops routing keyboard events to the input (so continued typing does nothing, or is captured by whatever now has focus — often nothing useful, since list items aren't usually built to receive typed text), and on mobile, moving focus off a text input typically dismisses the on-screen keyboard, ending the interaction entirely. The entire value proposition of a combobox over a `<select>` — keep typing to refine while also being able to browse/arrow through matches — depends on focus never leaving the input.

The trap: implementing it with real focus movement because it's the more "obvious" way to make an item "become focused," and only discovering the break when actually trying to type-then-arrow-then-type-again during testing, rather than reasoning through it up front.

---

**Q (High): How would you announce that a selection was successfully made, beyond the input's value visibly changing?**

Answer: For most screen readers, the input's value changing *is* itself announced — when `aria-activedescendant`-based selection writes the chosen option's label into the controlled input's value, AT generally announces the new value the same way it would announce a user typing that text, which is sufficient confirmation for the common case. Where I'd add something explicit is a combobox where selection triggers a side effect beyond just filling the input — for example, selecting a city in a "search by location" combobox that also triggers an immediate search and updates results elsewhere on the page. In that case, I'd pair the selection with a live-region announcement describing the resulting state change ("Showing results for Chicago"), because the input-value change alone doesn't communicate that something *else* on the page also updated as a consequence.

The trap: assuming a live-region announcement is always required for selection confirmation — for the simple case, it's redundant with the (already-announced) value change, and adding one anyway produces a doubled, cluttered announcement.

---

**Q (Medium): A teammate suggests simplifying this by rendering the suggestions as a native `<select>` with a `<datalist>` for the free-text part — what would you tell them?**

Answer: `<datalist>` gets a fair amount of this "for free" from the browser — native keyboard handling, native AT announcements, zero custom ARIA to get wrong — and for a genuinely simple case (suggest-as-you-type with no custom rendering of each option, no async-loaded suggestions, no need to control exactly when the list opens) it's a legitimate, lower-maintenance choice I'd actively recommend over hand-rolling the ARIA combobox pattern. Where it falls short and a custom implementation becomes necessary: `<datalist>` options can't contain rich content (icons, secondary metadata, multi-line rows), styling the suggestion popup is not meaningfully controllable across browsers, and its keyboard/interaction behavior isn't fully consistent across browser implementations the way a hand-built `role="combobox"` pattern is when correctly implemented. I'd frame it as: default to `<datalist>` until a specific, real requirement rules it out, not the reverse.

The trap: dismissing `<datalist>` outright as "not accessible enough" or "too limited" without being able to name the *specific* requirement that rules it out — a strong answer names the actual constraint (rich option rendering, cross-browser interaction consistency) rather than a vague preference for the custom-built version.

---

**Q (Medium): How should this component behave differently, if at all, for a mobile screen reader (VoiceOver on iOS / TalkBack on Android) versus desktop NVDA?**

Answer: The core pattern (focus stays in the input, `aria-activedescendant` tracks the active option) is the same, but the interaction model for *reaching* the arrow-key behavior differs meaningfully — mobile screen readers intercept swipe gestures for their own navigation, and a physical arrow key doesn't exist, so "arrow through suggestions" has to be reachable via the mobile AT's own exploration gestures (swiping to move the VoiceOver/TalkBack cursor through the listbox's options) rather than assuming keyboard arrow-key events will ever fire on a touch device. In practice this usually means the listbox's options need to be independently reachable/swipeable as their own accessible elements (which `role="option"` elements already are, structurally), while the desktop implementation additionally layers keyboard arrow handling on top for users navigating by physical keyboard. I'd test both explicitly rather than assuming desktop-keyboard testing covers mobile screen-reader gesture navigation, since they're genuinely different interaction surfaces reaching the same underlying markup.

The trap: assuming "I tested the ARIA attributes with NVDA" transfers directly to mobile screen readers — the markup transfers, but the interaction/navigation model a user actually experiences is different enough to warrant separate verification.

---

**Q (Low): How would you extend this to support multi-select (choosing several tags from the suggestions)?**

Answer: The core combobox mechanics (focus stays in input, `aria-activedescendant` tracks highlight, arrow keys move it) stay the same; what changes is that Enter/click on an option toggles it into a separately-rendered "selected items" list (each as a removable chip/tag, typically with its own `aria-label="Remove {item}"` button) rather than replacing the input's value — the input itself usually reverts to empty or the in-progress query after each selection, ready for the next one, instead of holding the selected value the way single-select does. `aria-multiselectable="true"` goes on the listbox, and each selected option keeps `aria-selected="true"` while still visible in the list (so re-opening the list shows prior selections as checked, not just absent). The live-region announcement pattern extends naturally — announcing "added Chicago, 2 selected" on selection and "removed Chicago, 1 selected" on removal — which is important here specifically because, unlike single-select, the input value alone no longer communicates full selection state to a screen reader user.

The trap: reusing the single-select's "input value = current state" mental model unchanged — multi-select needs the input to go back to empty/query-mode after each pick, with selection state now living somewhere else (the chip list) that must be independently communicated to AT, not implicitly inferred from the input's contents.

---

## Self-Assessment

- [ ] Can state, before building, why focus must stay in the input and `aria-activedescendant` must be used instead of moving real focus
- [ ] Can predict what NVDA announces on typing, arrowing, and selecting, and explain why activedescendant-based highlighting announces correctly without focus moving
- [ ] Can implement `aria-expanded` gated strictly on actual listbox content, not just "list panel is toggled open"
- [ ] Can explain why a debounced live region is necessary for the result count and what breaks without the debounce
- [ ] Can justify Escape's two-stage behavior (close list first, clear query second) over immediate clearing
- [ ] Can describe at least one concrete difference between desktop-keyboard and mobile-screen-reader testing for this exact component

---
*Next: Live Region Announcements for Async Updates — the `aria-live` mechanics touched on here (debouncing, polite vs. assertive, why the region must exist before content changes) generalized to toasts, loading states, and async form results.*
