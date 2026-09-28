# Accessible Drag-and-Drop Alternative for Keyboard Users

## Quick Reference

| Approach | Mechanism | Verdict |
|---|---|---|
| `aria-grabbed`/`aria-dropeffect` | The original WAI-ARIA drag-and-drop attributes | **Deprecated** in ARIA 1.1 — don't reach for these in new code |
| "Move up" / "Move down" buttons per item | Explicit, always-visible controls that reorder by one step | Simplest to build and announce correctly; visually adds clutter to every row |
| Keyboard "grab mode" (select item, arrow keys move it, second select drops it) | Mirrors the mouse-drag mental model most closely, announced via a live region | Best UX parity with pointer drag-and-drop; more state to manage correctly |
| Cut/paste-style reorder (select item, select target position, confirm) | Two-step, no modal "grabbed" state to track | A reasonable middle ground; less discoverable without a visible affordance explaining it |

## The Scenario

"We shipped a drag-and-drop sortable list — reorder your dashboard widgets, reorder a priority queue, whatever the product is. Accessibility flagged it: there's no way to reorder anything without a mouse. Design a keyboard-accessible alternative that doesn't just bolt on an afterthought — it needs to actually be usable, not a checkbox-compliance version nobody would choose to use."

This is a direct extension of [Drag-and-Drop Sortable List](../phase-02-component-machine-coding/10-drag-and-drop-sortable-list.md) from Phase 2, and exactly the kind of finding a real audit surfaces — see finding #2 in [Auditing an Existing App for Accessibility](06-auditing-an-app-for-accessibility.md) for the same class of issue on a different component.

## Clarifying Questions

- **Is pointer-based drag-and-drop staying as-is, with a keyboard alternative added alongside it, or is this a chance to replace the whole interaction model with something that serves both input methods equally well?** These are genuinely different scopes — adding a parallel keyboard path to existing mouse-drag code versus redesigning the reordering interaction from scratch to be input-method-agnostic from the start (which several accessible list libraries, like `react-beautiful-dnd`'s successors, now do by default).
- **How many items are typically in the list, and how far might something need to move — adjacent swaps, or across a long list?** A "move up/move down" per-step model gets genuinely tedious for moving an item from position 2 to position 40 in a fifty-item list — worth knowing whether that's a realistic use case before committing to a purely step-based interaction.
- **Does the list have a visual/semantic grouping (sections, categories) items can move between, or is it always a single flat list?** Cross-group reordering (moving a widget from one dashboard section to another) is a meaningfully harder announcement and interaction problem than reordering within one flat list, and changes the scope of what "drop target" even means.
- **Is there an existing convention elsewhere in the product for this pattern already, or is this the first one?** If a sibling team already solved this for a different reorderable list, consistency with their solution is usually worth more than a locally "better" but divergent pattern — I'd check before designing from scratch.

## Approach & Trade-offs

**I'd reject `aria-grabbed`/`aria-dropeffect` immediately and say so explicitly if anyone suggests them** — they were part of the original WAI-ARIA drag-and-drop model, are deprecated as of ARIA 1.1, have inconsistent-to-nonexistent modern AT support, and current guidance (including the ARIA Authoring Practices Guide) doesn't recommend them for new implementations. Their replacement isn't a newer pair of ARIA attributes — it's designing an explicit, button-and-live-region-driven reordering interaction that doesn't rely on ARIA trying to describe a live drag gesture at all.

**I'd choose the "keyboard grab mode" pattern (select an item, arrow keys move it one position at a time while "grabbed," a second selection or Enter confirms/drops it) over simple move-up/move-down buttons, specifically because it preserves the mental model sighted mouse users already have, rather than offering keyboard users a visibly different, second-class interaction.** The trade-off I'd state plainly: this is more implementation complexity (a "grabbed" state to track, more keyboard handling, more announcement logic) than a pair of move-up/move-down buttons per row, which is genuinely simpler to build correctly. I'd still pick grab-mode as the default for a list where reordering is a primary, frequent interaction (a dashboard widget layout users tune often) — for a list where reordering is rare and secondary, the simpler move-up/move-down buttons are a perfectly legitimate, lower-effort choice, and I'd say so rather than treating one pattern as universally correct.

**Every state transition in the grab-mode interaction needs an explicit live-region announcement, because none of it has any equivalent visual affordance a screen reader user could otherwise infer** — a sighted user watching a mouse-drag sees the item visually follow the cursor and sees other items shift to make room, continuously, for free. None of that visual feedback exists for a screen reader user; every step needs to be said out loud: entering grab mode ("Widget 'Revenue Chart' grabbed, currently position 2 of 6. Use arrow keys to move, Enter to drop, Escape to cancel."), each move ("Moved to position 3 of 6"), and the final drop ("Widget 'Revenue Chart' dropped at position 3 of 6"). This is directly the [Live Region Announcements](03-live-region-async-announcements.md) discipline applied to a case where, unlike a search result count, every single announcement is actionable and expected — so the "don't over-announce routine events" caution from that scenario doesn't apply here the same way; a user actively performing a multi-step reorder wants a running account of exactly what's happening.

**I'd keep the underlying reorder function (the actual array-splice logic) identical for both the pointer-drag path and the keyboard path**, rather than building two independent reordering implementations — the interaction *triggers* differ completely (a `dragend` event's drop position vs. an Enter-key confirmation at a keyboard-tracked position), but "given a from-index and a to-index, produce the reordered array" is the same pure function either way, and keeping it shared means a bug fix or behavior change (like a "can't move item past a locked/pinned item" rule) only needs to be implemented once.

## Solution

Shared reorder logic, used by both interaction paths:

```tsx
function reorder<T>(list: T[], fromIndex: number, toIndex: number): T[] {
  const result = [...list];
  const [moved] = result.splice(fromIndex, 1);
  result.splice(toIndex, 0, moved);
  return result;
}
```

The keyboard grab-mode interaction, with a dedicated live region for reorder-specific announcements (kept separate from a page-wide general-purpose live region, since this one needs immediate, uncoalesced announcements for every step — the opposite of the debounced case in [Live Region Announcements](03-live-region-async-announcements.md)):

```tsx
function SortableList({ items, onReorder }: { items: Item[]; onReorder: (next: Item[]) => void }) {
  const [grabbedIndex, setGrabbedIndex] = useState<number | null>(null);
  const [announcement, setAnnouncement] = useState('');
  const itemRefs = useRef<(HTMLLIElement | null)[]>([]);

  function onKeyDown(e: React.KeyboardEvent, index: number) {
    if (grabbedIndex === null) {
      if (e.key === 'Enter' || e.key === ' ') {
        e.preventDefault();
        setGrabbedIndex(index);
        setAnnouncement(
          `${items[index].label} grabbed, position ${index + 1} of ${items.length}. ` +
          `Use arrow keys to move, Enter to drop, Escape to cancel.`
        );
      }
      return;
    }

    // Grabbed: arrow keys move, Enter drops, Escape cancels
    if (e.key === 'ArrowUp' && grabbedIndex > 0) {
      e.preventDefault();
      const to = grabbedIndex - 1;
      onReorder(reorder(items, grabbedIndex, to));
      setGrabbedIndex(to);
      setAnnouncement(`Moved to position ${to + 1} of ${items.length}`);
      itemRefs.current[to]?.focus(); // focus follows the item as it moves
    } else if (e.key === 'ArrowDown' && grabbedIndex < items.length - 1) {
      e.preventDefault();
      const to = grabbedIndex + 1;
      onReorder(reorder(items, grabbedIndex, to));
      setGrabbedIndex(to);
      setAnnouncement(`Moved to position ${to + 1} of ${items.length}`);
      itemRefs.current[to]?.focus();
    } else if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      setAnnouncement(`${items[grabbedIndex].label} dropped at position ${grabbedIndex + 1} of ${items.length}`);
      setGrabbedIndex(null);
    } else if (e.key === 'Escape') {
      e.preventDefault();
      // NOTE: a real implementation tracks the original index to revert to on cancel —
      // omitted here for brevity, but Escape must restore the pre-grab order, not just exit grab mode
      setAnnouncement('Reorder cancelled');
      setGrabbedIndex(null);
    }
  }

  return (
    <>
      <ul role="listbox" aria-label="Reorderable widget list">
        {items.map((item, i) => (
          <li
            key={item.id}
            ref={(el) => (itemRefs.current[i] = el)}
            role="option"
            tabIndex={0}
            aria-selected={grabbedIndex === i}
            aria-roledescription={grabbedIndex === i ? 'grabbed, reorderable item' : 'reorderable item'}
            onKeyDown={(e) => onKeyDown(e, i)}
          >
            {item.label}
          </li>
        ))}
      </ul>
      <div aria-live="assertive" role="alert" className="visually-hidden">{announcement}</div>
    </>
  );
}
```

`assertive` here is a deliberate choice, not the default-to-polite guidance from the live-region scenario — while actively performing a multi-step keyboard reorder, the user needs each move confirmed immediately and in sequence, and a `polite` region risks queuing/delaying rapid successive arrow-key moves in a way that desyncs the announcement from the actual keypress that caused it.

> **Check yourself:** Trace through what happens, step by step, if `itemRefs.current[to]?.focus()` were omitted after a move — where does focus end up, and why does that specifically break the "grabbed" mental model even if the reordering itself still works correctly?

## Gotchas

**Reaching for `aria-grabbed`/`aria-dropeffect`.** Deprecated, inconsistent modern support — a candidate citing these as "the accessible drag-and-drop attributes" is working from outdated information, which is itself worth flagging rather than silently going along with if a teammate suggests it.

**Building the keyboard alternative as a visually-hidden, functionally-separate "screen reader only" version rather than a real, usable-by-anyone interaction.** A common but weak pattern: a hidden set of move-up/move-down buttons that only sighted keyboard users would ever discover by accident, existing purely to make an automated scanner pass, without any real design thought toward whether a sighted keyboard-only user (not just a screen reader user) would ever find or want to use it.

**Not moving focus to follow the item as it's reordered.** If focus stays on whatever DOM position it started at while the *item* moves past it, the grabbed mental model breaks immediately — the user pressed ArrowDown expecting to still be "on" the item they grabbed, and instead their focus/announcement context silently detaches from it.

**Escape not reverting the reorder.** If Escape merely exits grab mode without restoring the pre-grab position, a user experimenting with "let me see if this fits" and backing out ends up with an unintended, silent reorder they didn't confirm — Escape needs to mean "cancel," not just "stop being grabbed at wherever it currently is."

**Using `polite` for the move-by-move announcements.** Produces desynced or dropped announcements during rapid arrow-key presses, which is actively worse than a slight interruption here, since the entire point of the announcement is confirming each discrete action the user just took, in order.

## Follow-up Questions

**Q (High): Why are `aria-grabbed` and `aria-dropeffect` no longer the recommended approach, and what replaced them?**

Answer: They were part of the original WAI-ARIA 1.0 drag-and-drop model, attempting to describe a live pointer-drag gesture declaratively to assistive tech — in practice, AT support for meaningfully interpreting them was always inconsistent, and they were formally deprecated in ARIA 1.1. Nothing replaced them as a like-for-like ARIA-attribute swap — the current recommended approach (per the ARIA Authoring Practices Guide and common practice in accessible component libraries) is not trying to make AT understand a drag gesture at all, but instead offering a discrete, keyboard-operable alternative interaction (grab-mode-and-arrow-keys, or move-up/move-down controls) with explicit live-region announcements for each step — essentially, redesigning the interaction to not depend on continuous drag semantics for non-pointer users, rather than trying to accessibility-describe the drag gesture itself.

The trap: citing `aria-grabbed`/`aria-dropeffect` as "the" accessible drag-and-drop attributes in an interview — this is dated information that a senior candidate should know has been superseded, and confidently citing deprecated guidance is a worse signal than not knowing the specific attribute names at all.

---

**Q (High): Walk through exactly what a screen reader announces during a full grab-move-drop cycle with your implementation, and identify the single most important announcement in that sequence.**

Answer: On Enter to grab: "[Widget name] grabbed, position 2 of 6. Use arrow keys to move, Enter to drop, Escape to cancel" — this is arguably the single most important announcement in the whole interaction, because without it a screen reader user has no way to discover the interaction model exists at all; visually, sighted users infer "I can drag this" from cursor affordances and a grab handle icon, and this announcement is the only equivalent onboarding a non-sighted user gets. On each ArrowDown: "Moved to position 3 of 6" — confirming the specific new position, not just "moved," since the number is what lets the user track progress toward wherever they're trying to place it. On Enter to drop: "[Widget name] dropped at position 3 of 6" — confirming completion and exiting grab mode. If forced to rank, the initial grab announcement matters most, because every subsequent announcement is meaningless to a user who never discovered the interaction pattern in the first place — a perfectly-announced move sequence a user never triggers because they didn't know Enter would grab the item is a design failure regardless of how well the rest is implemented.

The trap: focusing implementation effort primarily on the move/drop announcements (the "interesting" state-tracking part) while treating the initial grab announcement as a minor detail — it's actually the highest-leverage single announcement in the sequence, since it's the discoverability mechanism for the entire feature.

---

**Q (Medium): How would you handle reordering across groups (moving a widget from "Favorites" to "All Widgets") with this keyboard model?**

Answer: The core grab-move-drop mechanic extends, but "move" needs a way to cross a group boundary, not just move within a flat list — I'd add explicit boundary announcements ("Reached end of Favorites — press ArrowDown again to move into All Widgets" or a dedicated key, depending on how discoverable that needs to be) rather than silently blocking ArrowDown at a group's end or silently crossing into the next group without any announcement of the fact that a group boundary was just crossed, either of which would be confusing. The reorder function itself would need to become group-aware (moving an item to a new group means changing both its position and, semantically, some group-membership field, not just an index within one flat array), but the interaction pattern and live-region discipline stay the same — just with an added announcement specifically for the group-crossing event, since that's a materially different, higher-stakes action (moving something into a different logical category) than a same-group position swap.

The trap: treating cross-group movement as "the same thing, just with more items to iterate through" — the group boundary itself is significant information a screen reader user needs explicitly announced, not something that can be silently absorbed into the existing position-count announcements.

---

**Q (Medium): A teammate suggests using `react-beautiful-dnd` (or a similar drag-and-drop library) instead of hand-rolling this — does that solve the accessibility problem for free?**

Answer: Partially, and it depends heavily on which library and version — some modern accessible-drag-and-drop libraries (and `react-beautiful-dnd`'s more actively maintained alternatives, like `dnd-kit` with its accessibility-focused sensors, or Atlassian's `pragmatic-drag-and-drop`) do ship a built-in keyboard-alternative interaction and live-region announcements out of the box, in which case adopting the library is a legitimate, often better-tested way to get this rather than hand-rolling it — the same "use the vetted primitive if one exists" reasoning from the [Accessible Modal](01-accessible-modal-scenario.md) scenario applies here too. It's not automatic, though — a drag-and-drop library that only implements pointer/touch dragging without a documented keyboard alternative doesn't solve this by adoption alone, and I'd specifically check the library's own accessibility documentation and, ideally, actually test its keyboard path with a screen reader before assuming it's covered, rather than assuming any popular library handles this correctly by default.

The trap: assuming any well-known drag-and-drop library inherently "has accessibility built in" without verifying the specific library's actual keyboard/screen-reader support — library popularity and accessibility completeness aren't the same axis, and this exact gap (no keyboard alternative) is a commonly cited criticism of several popular drag-and-drop libraries.

---

**Q (Low): Does this same grab-mode pattern make sense for a two-dimensional layout (a drag-and-drop dashboard grid, not a single-column list), or does it need to change?**

Answer: The core mechanic (select/grab, directional movement, confirm/drop, live-region announcement per step) extends conceptually, but "arrow keys move by one position" becomes ambiguous in two dimensions in a way it isn't in a single ordered list — ArrowUp/ArrowDown/ArrowLeft/ArrowRight each need a well-defined meaning against the grid's actual layout (move to the next open cell in that direction, potentially swapping with or displacing whatever's there), and the announcement needs to communicate a coordinate or row/column position rather than a single linear index ("moved to row 2, column 3" instead of "position 3 of 6"). This is meaningfully more design work than the one-dimensional case — collision/displacement behavior when moving into an occupied cell needs its own explicit rule and its own announcement, which doesn't have an equivalent in a simple linear reorder — so I'd treat a 2D grid version as a related but distinctly harder scenario, not a trivial extension of the list version.

The trap: assuming the one-dimensional grab-mode pattern trivially generalizes to two dimensions — the directional-movement semantics and collision handling are genuinely new problems a linear list never has to solve, not just "the same thing with two more arrow keys wired up."

---

## Self-Assessment

- [ ] Can explain why `aria-grabbed`/`aria-dropeffect` are deprecated and what replaced them conceptually
- [ ] Can implement the grab-move-drop keyboard interaction with correct focus-following behavior
- [ ] Can write the exact announcement text for grab, move, and drop, and explain why the initial grab announcement is the highest-leverage one
- [ ] Can justify using `assertive` here despite the general polite-by-default guidance from the live-region scenario
- [ ] Can explain why Escape must revert the reorder, not just exit grab mode
- [ ] Can evaluate whether an existing drag-and-drop library actually solves this, rather than assuming any popular library does

---
*This closes Phase 9 — Accessibility Scenarios. Next: Phase 10 — Security Scenarios, starting with Stored XSS Found in Production.*
