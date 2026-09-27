# Key Prop Misuse — State Bleeding Between List Items

## Quick Reference

| Cause | Mechanism | Fix |
|---|---|---|
| Index as key, list reorders/filters/deletes | React matches old and new elements at the same *position*, not the same *item* — internal state (uncontrolled inputs, `useState`) gets reattached to whatever item now sits at that index | Key by a stable, unique, item-intrinsic id — never array index, for any list that can reorder, filter, or have items removed from the middle |
| No key at all | React falls back to index-based matching implicitly, with a dev-mode console warning | Same fix — stable id |
| Key that isn't actually unique/stable | Two siblings sharing a key, or a key derived from something that changes | React's reconciliation behavior becomes undefined/buggy in ways similar to index-keying; pick something guaranteed unique and immutable for the item's lifetime |
| Key changed unnecessarily (e.g., `key={Math.random()}`) | Forces React to always treat it as a brand-new element — unmounts and remounts every render | Only use a changing key deliberately, when you *want* to force a full remount (e.g., resetting a form's internal state on a specific transition) |

## The Scenario

"This is a list of editable rows — each has its own text input holding a local draft value, plus a 'saved' checkbox. If you type into row 3's input, then delete row 1 from the list, row 3's text mysteriously appears in what is now the second row, and the checkbox states get scrambled too. Nothing about the actual data array looks wrong when you log it. Find the bug."

## Clarifying Questions

- **How is the list currently being rendered — what's used as the `key` prop on each row, if anything?** This is the first and most likely place to look; the specific symptom described (an input's *local* state appearing to jump to a different row's array position after a deletion) is close to a textbook signature of index-based (or missing) keys.
- **Is the input's value itself stored in the parent's data array (a controlled input, driven by `data[i].text`), or is it local, uncontrolled state living inside each row component (e.g., `useState` initialized from a prop, or an uncontrolled `<input defaultValue>`)?** This matters because the bug manifests differently and is diagnosed differently depending on where the "state that leaks" actually lives — if it's controlled entirely from the parent's array, the *displayed* value can't independently drift from the array's contents the way described; the scenario's symptom (values appearing in the wrong row after the data changed correctly) points strongly toward local/uncontrolled row state that isn't correctly tied to a stable identity.
- **Does the underlying data array have a genuinely unique, stable identifier per item** (a database id, a UUID) **or only implicit positional identity (a plain array of strings, or objects without an id field)?** If there's truly no stable id in the data as it exists today, part of the fix is ensuring one exists (even if the API needs to be asked to provide it, or one is generated client-side at creation time and persisted) — you can't key by something that doesn't exist.
- **Does the bug also occur on simple appends (adding a new row at the end) or only on deletions/reorders from the middle?** Appending at the end usually doesn't disturb any existing item's index, so an index-keyed list can appear to "work fine" for a long time under append-only usage and only reveal the bug once a mid-list deletion, insertion, or sort happens — worth confirming this matches, since it explains why an existing index-keyed list might have shipped without anyone noticing until now.
- **Is this reproducible consistently, or does it seem to depend on exactly which row is edited before which row is deleted?** Confirms whether the symptom precisely tracks "state that was tied to index N now shows up wherever index N points after the array shifted" — if so, that's about as direct a confirmation of the index-key hypothesis as you can get without reading the code.

## Approach & Trade-offs

**What `key` actually does — it's an identity hint for reconciliation, not just a React-required prop to silence a warning.** When React re-renders a list, it needs to decide, for each element in the new list, whether it corresponds to an *existing* DOM node/component instance (update it in place, preserving its internal state) or is a genuinely *new* item (mount a fresh instance). `key` is the signal React uses to make that decision — elements with the same `key` across two renders are treated as "the same conceptual item," and React reuses/updates the existing instance (including its internal state) rather than remounting; elements whose key doesn't appear in the previous render are mounted fresh; keys that disappear between renders have their instances unmounted (and any internal state discarded, correctly, since the item is genuinely gone).

**Exactly why index-as-key breaks under a mid-list deletion.** Before deleting the item originally at index 0, the list is rendered with keys `0, 1, 2, ...` — each key/position pairing is stable because nothing has moved yet, so it can *look* like index keys work fine. The moment the item at index 0 is removed, everything after it shifts down by one position — what was rendered at key `1` (the second row, holding whatever local state that row instance had accumulated) is now, after the array shift, being asked to render the data that used to belong to index `2`, but React still sees "key `1`" being requested at that position and — because it's matching by key, and key `1` existed last render too — reuses the *existing component instance* that previously represented the old index-1 row, now handed *new* props corresponding to what's logically a different row's data. The component instance (and all its internal `useState`/uncontrolled-input state) doesn't reset, because as far as React's reconciliation is concerned, key `1` is the same item it was last render — it just quietly received different data. That's the exact mechanism behind "row 3's typed text appears to jump into the new second row" — it's not that data got corrupted; it's that a *stateful component instance* got reattached to a different logical item's data while keeping its own leftover internal state.

**Why this doesn't happen (or happens far less severely) with fully controlled inputs even under index-keying.** If a row's input value comes entirely from `data[i].text` on every render — no internal component state at all — then even if React reuses the "wrong" instance due to index-keying, that instance is handed the *correct* prop (`data[i].text` for whatever item is now genuinely at index `i`) and renders correctly, because there's no leftover internal state to bleed. This is worth noting explicitly: index-keying is *always* structurally fragile and worth avoiding, but it's *most visibly, dangerously* broken specifically when combined with uncontrolled/local component state — which is exactly the combination in this scenario (a local draft value plus a local checkbox state).

**The fix — key by something intrinsic and stable to the item, never derived from position.** A database id, a UUID generated once at creation time and never regenerated, or any other value guaranteed unique-and-stable for that item's entire lifetime in the list. With a stable id as the key, deleting item A means React sees that A's key is now absent — correctly unmounting exactly that instance and discarding exactly its state — while every other item's key is unchanged between renders, so React correctly reuses each of *those* instances (and their internal state) as the *same* item, position notwithstanding. This is precisely the guarantee the scenario needs: a row's own internal draft state stays attached to *that row*, regardless of what index it now occupies after a deletion elsewhere in the list.

**When index-as-key is actually fine.** For a list that is genuinely static (never reordered, filtered, or has items inserted/removed from anywhere but the end, and items have no per-item internal state at all — pure, stateless, controlled-by-parent-data rendering), index keys are harmless, because none of the failure conditions above ever occur. It's worth being able to name this rather than treating "never use index as key" as an unexamined rule — the actual rule is "index-as-key is unsafe specifically when list order/membership can change *and* items carry state that could bleed," which is the overwhelming majority of real interactive list UIs, but not literally all lists.

## Solution

Reproducing the bug — index-keyed rows, each with local uncontrolled state:

```tsx
type Row = { id: string; label: string };

function EditableList({ rows, onDelete }: { rows: Row[]; onDelete: (id: string) => void }) {
  return (
    <ul>
      {rows.map((row, index) => (
        <EditableRow key={index} row={row} onDelete={() => onDelete(row.id)} /> // BUG: key is positional
      ))}
    </ul>
  );
}

function EditableRow({ row, onDelete }: { row: Row; onDelete: () => void }) {
  const [draft, setDraft] = useState(row.label); // local state — this is what bleeds
  const [saved, setSaved] = useState(false);

  return (
    <li>
      <input value={draft} onChange={e => setDraft(e.target.value)} />
      <label>
        <input type="checkbox" checked={saved} onChange={e => setSaved(e.target.checked)} /> Saved
      </label>
      <button onClick={onDelete}>Delete</button>
    </li>
  );
}
```

Type into row index 2's input (draft becomes "hello"), then delete the row at index 0. The array shifts; what's now rendered at index 1 is the data that used to be at index 2 — but React, matching on key `1` (which existed both before and after, since it's purely positional), reuses the *component instance* that used to be at index 1, complete with whatever `draft`/`saved` state *that* instance had accumulated — while the instance that used to hold "hello" (previously at key `2`, now nonexistent since the list shrank by one) is discarded along with its state. The net visible effect: the "hello" text and any checked box appear to have jumped to a different row's *data*, when what actually happened is a stateful component instance got silently reattached to different data.

Fix — key by the item's stable, intrinsic id:

```tsx
function EditableList({ rows, onDelete }: { rows: Row[]; onDelete: (id: string) => void }) {
  return (
    <ul>
      {rows.map(row => (
        <EditableRow key={row.id} row={row} onDelete={() => onDelete(row.id)} /> // stable, intrinsic identity
      ))}
    </ul>
  );
}
```

`EditableRow`'s implementation doesn't need to change at all — the fix is entirely in the `key`. Now, deleting the row with id `"r1"` means React sees exactly one key (`"r1"`) disappear between renders; every other row's key is unchanged, so every other row's component instance — and its local `draft`/`saved` state — is correctly preserved as belonging to *that specific row*, regardless of what index it now occupies.

> **Check yourself:** If `EditableRow`'s `draft` state were instead derived as `const [draft, setDraft] = useState(row.label)` but the `row` object passed in could have its `label` field updated *externally* (e.g., a bulk "reset all labels" action from elsewhere in the app) while keyed correctly by `row.id`, would the row's displayed `draft` value update to reflect the new `label`? Why or why not — and what does this reveal about `useState`'s initializer argument?

## Gotchas

**"It works fine when I test append-only or edit-without-delete."** Index-keying failures are specifically triggered by insertions/deletions/reorders that aren't at the very end of the list — a list that's only ever appended to can mask this bug indefinitely during casual testing, right up until the first mid-list deletion or sort in production.

**Using `key={row.id}` correctly, but generating `row.id` freshly on every render** (e.g., `id: Math.random()` computed inline during render, or a `crypto.randomUUID()` call placed in the render path rather than at true item-creation time) — this produces a *new* key every render regardless of whether the item is "the same" conceptually, forcing a full remount every single render, the opposite failure mode (unnecessary state loss on every render, rather than incorrect state reuse).

**Assuming any non-index key is automatically safe.** A key that happens to not be unique (two different items sharing a key due to a data bug, or a composite key built from fields that aren't actually distinct per item) reintroduces the same class of problem — React's guarantees about correct reconciliation depend on keys within a sibling list actually being unique, not merely "not literally the array index."

**Treating this purely as a "React quirk" rather than as an instance of a general identity problem.** The same underlying issue — treating positional/incidental identity as if it were the item's real identity — recurs in other guises (as covered in the nested-comments scenario, and again here); recognizing it as one recurring pattern, not a list of unrelated one-off React gotchas, is the deeper signal an interviewer is often listening for.

**Fixing the `key` but leaving a `useState(row.label)` initializer that silently ignores subsequent prop changes.** `useState`'s initial-value argument is only used on the component's *first* mount for a given key — if `row.label` changes later via a prop update to an already-mounted (same-key) instance, `draft` will not automatically follow it, because `useState` doesn't re-run its initializer on subsequent renders. This is a legitimate, separate design question (should local edits ever be silently overwritten by external changes?) worth surfacing rather than assuming the key fix alone makes everything about state-and-props interaction correct.

## Follow-up Questions

**Q (High): Walk through, step by step, exactly what React's reconciler does differently between the index-keyed and id-keyed versions when the first item in a three-item list is deleted.**

Answer: Before the deletion, both versions render three elements with keys `[0, 1, 2]` (index-keyed) or `["r0", "r1", "r2"]` (id-keyed) respectively — structurally equivalent at this point. After deleting the first item, the index-keyed version's next render produces keys `[0, 1]` (the two remaining items, at their new positions) — React compares this against the previous `[0, 1, 2]` and sees keys `0` and `1` present in both, key `2` no longer present: it reuses the component instances previously at keys `0` and `1` (with their existing internal state intact) and feeds them the *new* props corresponding to whatever data is now at those positions (which, after a shift, is different data than they previously represented), then unmounts whatever was at key `2`. The id-keyed version's next render produces keys `["r1", "r2"]` (the two remaining items, correctly using their own unchanging ids) — React compares against previous `["r0", "r1", "r2"]` and sees `"r0"` is gone (unmounts exactly that instance, discarding exactly its state, which is correct since that item genuinely no longer exists) while `"r1"` and `"r2"` are both still present (reuses those two instances, with their internal state correctly still attached to the same underlying items, now simply at different array positions).

The trap: describing the difference only at the level of "index keys are bad, id keys are good" without being able to narrate the actual reconciliation steps — specifically naming which instances get reused-with-new-props versus unmounted-and-remounted in each case is what distinguishes understanding the mechanism from having memorized the rule.

---

**Q (High): If every field in this list were fully controlled — the input's value coming directly from `rows[i].label` with no internal `useState` in the row component at all — would index-as-key still cause a visible bug on deletion? Why or why not?**

Answer: The *reconciliation behavior* is identical either way — React still reuses the "wrong" component instance for a given index positionally, exactly as described above — but the *visible consequence* differs, because a fully controlled row has no internal state of its own to bleed. Each render, the reused instance is simply handed `rows[i].label` as a prop and renders it directly; since there's no local state holding a stale value, the displayed content is always correct for whatever the current props say, regardless of which underlying instance is technically rendering it. The bug becomes purely internal/theoretical in this case (React did "the wrong thing" reconciliation-wise, reusing an instance that isn't conceptually the same item, but nothing observable is wrong as a result) — right up until that component gains *any* internal state (a focus state, an animation-in-progress flag, a `useRef` tied to a DOM measurement) at which point the same latent fragility becomes visible again. This is why "always key by stable id, even for currently-stateless list items" remains the safer default — the moment someone innocently adds local state to a list item component later, an index-keyed list silently becomes buggy without that addition looking suspicious in review.

The trap: concluding that fully-controlled rows make index-as-key "safe" in an unqualified sense — it's safe *today*, for *this* component's current implementation, but it's a latent trap for whoever adds local state to that row component later without realizing the list above it uses index keys.

---

**Q (High): A teammate proposes fixing this by using `key={JSON.stringify(row)}` instead of an id, reasoning "this way the key definitely reflects the row's actual content." What's wrong with this approach?**

Answer: Several problems. First, it's expensive — serializing every row's entire contents on every render, for every row, purely to produce a key, is unnecessary work compared to reading an already-existing id field (or negligible-cost stable UUID). Second, and more importantly, it ties identity to *content* rather than to the item's actual identity — if a row's `label` is *edited* (exactly the scenario described in this prompt), `JSON.stringify(row)` produces a *different* string, which means React sees a "new" key for what is conceptually the *same* item being edited, unmounting and remounting it and destroying exactly the local state (the in-progress draft, the checkbox) editing the item was supposed to preserve — reintroducing state loss, just via a different mechanism than the original bug, and in fact directly defeating the interactive-editing use case the row component exists for. A `key` needs to track *identity*, which is conceptually orthogonal to *content* — the same conceptual item can have its content change over time while remaining "the same item," and the key needs to reflect that persistence, not the current snapshot of its fields.

The trap: accepting "the key reflects the actual data" as inherently safer without working through what happens when that data — specifically the fields being actively edited via local state — changes; a content-derived key is arguably worse than an index-derived one for a list with editable fields, since it guarantees a remount on every edit rather than only on reordering.

---

**Q (Medium): The data array genuinely has no unique identifier per item — it's an array of plain strings, e.g., `["Alice", "Bob", "Carol"]`, and duplicates are possible. How would you key this list correctly?**

Answer: Since neither array index nor the string content itself (given possible duplicates) is a safe key on its own, the item needs an identity assigned independently of its content — most commonly, transforming the raw array into an array of `{ id, value }` pairs once, at the point the data enters application state (assigning a stable id via a counter or `crypto.randomUUID()` at creation time, stored alongside the value, never regenerated on subsequent renders), and keying off that assigned `id` from then on. If the array is only ever read, never mutated in place (e.g., freshly re-fetched from an API on every load with no client-side identity), and duplicates and reordering are both genuinely expected, this is a sign the data model itself is under-specified for the UI's needs — the right fix is asking whichever system is the source of truth for this data to provide a real identifier, since inventing one client-side on every fetch (rather than once, and persisted) just reproduces the original problem one layer removed.

The trap: suggesting `key={value + index}` as a compromise — this "solves" the immediate duplicate-string problem but reintroduces every index-based failure mode the moment the list reorders or an item is removed from the middle, since the index portion of the composite key still shifts.

---

**Q (Medium): Does this same class of bug apply to non-list scenarios — for instance, conditionally rendering one of two different form components in the same position depending on some state?**

Answer: Yes, in a related form — if two structurally different components (or the same component representing conceptually different "things") render in the same JSX position across a state change without a distinguishing `key`, React may reconcile them as "the same" element type in the same position and attempt to update in place rather than unmount/remount, which can preserve stale internal state across what should be a clean transition (e.g., switching a form between "add new item" mode and "edit existing item" mode, reusing the same `<Form>` component type for both — without a key that changes between the two modes, an in-progress draft from "add" mode could persist into "edit" mode's initial render). The fix is the same principle applied outside of a `.map()`: give the two conceptually distinct renders different `key` values (even though they're not siblings in an array) so React treats a transition between them as an unmount-then-mount rather than an update-in-place, deliberately resetting internal state exactly when that reset is the correct behavior.

The trap: assuming `key` is exclusively a "list rendering" concept because that's the context it's almost always introduced in — its actual purpose (identity across reconciliation) applies to any place React decides whether to reuse or replace a component instance, list or not.

---

## Self-Assessment

- [ ] Can narrate exactly what React's reconciler does, step by step, differently between index-keyed and id-keyed lists on a mid-list deletion
- [ ] Can explain why index-keying is comparatively harmless for fully controlled, stateless list items but a latent trap the moment local state is added
- [ ] Can explain why a content-derived key (e.g., `JSON.stringify(item)`) is also wrong, and specifically how it fails differently than index-based keys
- [ ] Can propose a correct keying strategy when the underlying data has no natural unique identifier
- [ ] Can generalize the identity-vs-position distinction to non-list reconciliation scenarios (conditional rendering of structurally similar components)

---
*Next: Controlled vs. Uncontrolled Input Bug — a closely related failure mode involving the same "who owns this input's state" question, this time manifesting as a console warning and erratic typing behavior rather than state bleeding across siblings.*
