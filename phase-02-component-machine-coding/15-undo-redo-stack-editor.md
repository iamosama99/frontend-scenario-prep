# Undo/Redo Stack for an Editor UI

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Command pattern | Each action stores `do()`/`undo()` (or an inverse op) | Memory-efficient — stores deltas, not full state; scales to large documents |
| Snapshot/memento | Each undo point stores a full (or diffed) copy of state | Trivially correct to implement, but memory cost grows with state size × history depth |
| Two-stack model | `undoStack` (past) + `redoStack` (undone-but-recoverable future) | Standard shape for linear undo/redo; makes both operations O(1) push/pop |
| New action after undo | Push to `undoStack`, **clear `redoStack`** | The redone-able "future" is invalidated the instant a new divergent action happens — keeping stale redo entries lets you redo into a timeline that no longer matches state |
| Batching rapid changes | Group by timeout / explicit commit boundary (blur, pause-in-typing) | One undo step per keystroke makes Ctrl+Z useless — nobody wants to undo one character at a time |
| Bounding history | Cap stack size, evict oldest entries (like a ring buffer) | Unbounded undo history is an unbounded memory leak in a long editing session |

## The Scenario

"We're building a simple rich-text/diagram editor and need undo/redo — standard Ctrl+Z / Ctrl+Shift+Z. Users will be typing continuously, moving shapes, changing styles, all sorts of actions. I want undo to feel right — not one step per keystroke — and I want to understand the trade-offs in how you'd actually store the history, not just get a working demo."

## Clarifying Questions

- **How large and how complex is the state being tracked — a single text blob, or a document with many independent, structured objects (shapes, styles, nested elements)?** This is the single biggest driver of command-pattern vs. snapshot choice. A small, flat state (a form, a short text field) makes snapshotting cheap and simple; a large structured document (a canvas with hundreds of shapes) makes full snapshots expensive per undo step, favoring commands that store only what changed.
- **Do all actions need to be undoable, or are some (e.g., changing a zoom level, a UI-only toggle) explicitly excluded?** Not everything a user does is meant to be part of the "content" history — conflating UI/view state with document content in the same undo stack produces confusing behavior (undoing a text edit that also, incidentally, jumps the viewport because a zoom change got captured in the same stack).
- **Is there a maximum history depth, or is memory effectively unbounded for a long session?** A user who edits for hours without ever hitting Ctrl+Z accumulates history forever unless it's explicitly bounded — worth deciding a cap (and what happens at the cap: silently drop the oldest, or something else) rather than discovering the problem in production.
- **How should rapid, continuous input (typing a sentence, dragging a shape) be batched into undo steps?** This is the UX-defining question of the whole feature — if I don't ask this and just wire "record a snapshot on every state mutation," undo becomes technically correct but practically useless, requiring dozens of Ctrl+Z presses to undo one sentence.
- **Is this single-user, or could the same document be edited by multiple people/tabs concurrently (collaborative editing)?** A naive linear undo/redo stack assumes there's one linear timeline of changes — that assumption breaks the moment two actors can mutate the same state concurrently, which changes the entire architecture (this is worth flagging explicitly even if collaborative editing is out of scope for this exercise).

## Approach & Trade-offs

**Command pattern vs. snapshot/memento — the central decision.**

The **command pattern** records each user action as an object capturing enough information to both `redo()` (re-apply the action) and `undo()` (reverse it) — either as two closures, or as a single object with a `do`/`undo` pair, or by storing the *inverse* operation (e.g., "insert 'x' at position 5" undoes via "delete 1 character at position 5"). This is memory-efficient because each history entry is proportional to the size of the *change*, not the size of the whole document — undoing a single-character edit in a 50-page document costs the same, tiny amount of memory whether the document is 50 pages or 5. The cost is implementation complexity: every distinct action type needs its own carefully-written, correctly-invertible undo logic, and getting an inverse subtly wrong (e.g., an undo that doesn't restore exact prior formatting) is a real, easy-to-miss bug class.

The **snapshot/memento** approach instead stores a full (or structurally-shared/diffed) copy of the entire state at each undo-able point. Undo is just "restore the previous snapshot"; redo is "restore the next one." This is far simpler to implement correctly — there's no per-action-type inverse logic to get right, since a snapshot doesn't care *how* the state changed, only *what* it now is. The cost is memory: naively storing N full copies of a large document is O(N × document size), which becomes real for a large or long-lived document, though structural sharing (persistent data structures, or diffing consecutive snapshots and storing only the delta) mitigates this considerably.

**Which one for this scenario?** Given "editor UI" with likely-substantial document state (shapes, styles, structured content, not just a short string), I'd lean command pattern for anything that mutates a large shared document, and reserve snapshotting for small, self-contained pieces of state where correctness-by-construction matters more than the memory overhead (e.g., a form with a handful of fields). I'd say this out loud as the trade-off rather than picking one silently — this is exactly the kind of judgment call that separates "wrote an undo stack" from "understands why you'd choose one design over the other for a given system."

**The two-stack model.** Regardless of command vs. snapshot, the standard structure is two stacks: `undoStack` holds everything that's happened and can be undone (most recent on top); `redoStack` holds everything that's been undone and can be redeemed by redo (most recently-undone on top). `undo()` pops from `undoStack`, applies the inverse, and pushes the original entry onto `redoStack`. `redo()` pops from `redoStack`, re-applies it, and pushes it back onto `undoStack`. The rule that trips people up: the instant a *new* action is performed after one or more undos, `redoStack` must be cleared — the "future" it represented (redoing forward through actions that were undone) no longer corresponds to reality, because the user has now diverged onto a different timeline. Keeping stale redo entries around and letting a later redo re-apply them on top of a now-different state is a correctness bug, not a style choice.

**Batching.** The naive version — push a new undo entry on every single state mutation — technically satisfies "undo/redo exists" but fails the actual UX requirement: undoing one keystroke at a time is not what any real user wants from Ctrl+Z. The fix is defining a **commit boundary**: a point at which an in-progress burst of related changes (continuous typing, a drag operation) gets collapsed into a single undo-able unit. Two common mechanisms, often combined: (1) a debounce-style timer that keeps *extending* the current "open" undo entry as long as new changes keep arriving within some window (e.g., 500ms–1s of typing pause), and (2) explicit boundaries tied to real interaction semantics — a text field's `blur` event, the `mouseup` that ends a drag, or pressing Enter/a punctuation mark that naturally ends a "thought." I'd implement the timer-based approach as the general mechanism and layer explicit boundaries (blur, drag-end) on top since they're cheap wins with an existing DOM event to hook.

**Bounding history size.** A production editor session can run for hours; without a cap, `undoStack` (especially in snapshot form) grows unboundedly. Treating it as a fixed-capacity ring buffer — once at capacity, pushing a new entry evicts the oldest — bounds memory at the cost of eventually being unable to undo arbitrarily far back, which is an acceptable, standard trade-off (this connects directly to the LRU cache scenario's "bounded resource" theme, just applied to a stack instead of a cache).

## Solution

Command-pattern skeleton — each history entry is a `{ do, undo }` pair:

```javascript
class UndoRedoManager {
  #undoStack = [];
  #redoStack = [];
  #maxHistory;

  constructor({ maxHistory = 100 } = {}) {
    this.#maxHistory = maxHistory;
  }

  // `command` = { do: () => void, undo: () => void }
  // `do` has typically already been called by the caller before pushing —
  // pushing records the ability to reverse it, it doesn't perform it.
  push(command) {
    this.#undoStack.push(command);
    if (this.#undoStack.length > this.#maxHistory) {
      this.#undoStack.shift(); // evict oldest — bounds memory, ring-buffer style
    }
    this.#redoStack.length = 0; // critical: a new action invalidates the old "future"
  }

  undo() {
    const command = this.#undoStack.pop();
    if (!command) return false;
    command.undo();
    this.#redoStack.push(command);
    return true;
  }

  redo() {
    const command = this.#redoStack.pop();
    if (!command) return false;
    command.do();
    this.#undoStack.push(command);
    return true;
  }

  canUndo() { return this.#undoStack.length > 0; }
  canRedo() { return this.#redoStack.length > 0; }
}
```

A concrete command for a text-editing action (insert):

```javascript
function makeInsertCommand(editorModel, position, text) {
  return {
    do: () => editorModel.insertAt(position, text),
    undo: () => editorModel.deleteRange(position, position + text.length),
  };
}

// usage
const history = new UndoRedoManager({ maxHistory: 200 });

function handleUserInsert(position, text) {
  const command = makeInsertCommand(editorModel, position, text);
  command.do();       // perform the action
  history.push(command); // record it — this also clears any pending redo
}
```

Batching continuous typing into a single undo step, using a commit-boundary timer:

```javascript
class BatchedTextInsert {
  #position;
  #text = '';
  #editorModel;

  constructor(editorModel, startPosition) {
    this.#editorModel = editorModel;
    this.#position = startPosition;
  }

  append(char) {
    this.#editorModel.insertAt(this.#position + this.#text.length, char);
    this.#text += char;
  }

  toCommand() {
    const { insertedAt, insertedText } = { insertedAt: this.#position, insertedText: this.#text };
    return {
      do: () => this.#editorModel.insertAt(insertedAt, insertedText),
      undo: () => this.#editorModel.deleteRange(insertedAt, insertedAt + insertedText.length),
    };
  }
}

function createTypingRecorder(history, editorModel, { idleMs = 600 } = {}) {
  let batch = null;
  let idleTimer = null;

  function commit() {
    if (batch) {
      history.push(batch.toCommand());
      batch = null;
    }
    clearTimeout(idleTimer);
    idleTimer = null;
  }

  function onCharTyped(char, position) {
    if (!batch) {
      batch = new BatchedTextInsert(editorModel, position);
    }
    batch.append(char);

    clearTimeout(idleTimer);
    idleTimer = setTimeout(commit, idleMs); // pause in typing = commit boundary
  }

  // explicit boundaries beyond the idle timer
  editorEl.addEventListener('blur', commit);
  editorEl.addEventListener('keydown', (e) => {
    if (e.key === 'Enter' || e.key === '.' || e.key === ' ') commit(); // natural "thought" boundaries
  });

  return { onCharTyped, commit };
}
```

> **Check yourself:** Why does `push()` clear `#redoStack` unconditionally, even if the new action is "the same kind" of action as whatever was undone? Walk through a concrete sequence — type "hello", undo twice, type "hey" — and describe exactly what's in each stack at every step.

## Bounding History Without Losing Correctness

Evicting the oldest undo entry once at capacity (the `shift()` above) is simple but has a real consequence worth stating explicitly: once an entry is evicted, it's *gone* — there's no way to undo back past that point, ever, for the rest of the session. This is a deliberate, accepted trade-off (unbounded memory growth is worse), but it should be a conscious choice, not an accident. An alternative that avoids losing granularity forever is periodically **coalescing** old history into a single "checkpoint" snapshot once entries age past some threshold (keep fine-grained commands for recent history, collapse older history into coarser snapshots) — meaningfully more complex, and usually not worth building unless the product genuinely needs very deep undo history (most editors don't; a few hundred steps is generous).

## Gotchas

**Forgetting to clear `redoStack` on a new action.** This is the single most common correctness bug in this scenario. Sequence: type "A", type "B", undo (back to "A", "B" sits in redoStack), then type "C" instead of redoing. If `redoStack` isn't cleared, a later `redo()` re-applies "B" on top of a document that now has "C" instead of "A" — producing state that never actually existed in the user's real edit history. The redo stack must be invalidated the moment a fresh action diverges from the undone path.

**One undo step per keystroke.** Technically correct, practically unusable — this is what happens when "record a snapshot/command on every mutation" is implemented literally without an explicit batching/commit-boundary layer on top.

**Storing a reference to mutable state instead of a value/inverse operation, in the snapshot approach.** If a snapshot stores a *reference* to a mutable object (e.g., `snapshot = currentState`) rather than a deep copy, later mutations to `currentState` silently corrupt what was supposed to be a frozen historical snapshot — undo then "restores" a state that's already been mutated into looking like the present. Snapshots need genuine copies (deep clone, or structural sharing via a persistent/immutable data structure), never live references.

**Undoing a command whose closure captured stale references.** In the command pattern, if `undo()` closes over a specific DOM node or object reference that's since been replaced/removed (e.g., the shape was deleted and a *new* shape object created in its place with the same visual position), naively re-inserting via the stale reference can silently fail or attach to a detached node. Commands generally need to operate through stable IDs/lookups into current state, not direct object references captured at creation time.

**No cap on history size.** A long editing session with no history limit means memory grows for as long as the user keeps working — for a snapshot-based design in particular, this can become a real, user-visible memory problem in browser tabs left open for hours.

**Conflating "undoable content changes" with "UI/view state changes" in the same stack.** Pushing a zoom-level change or a panel-toggle onto the same undo stack as text edits means a content-focused undo unexpectedly also changes the viewport, confusing the user about what "undo" actually reverses.

## Follow-up Questions

**Q (High): Walk through why pushing a new action must clear the redo stack, with a concrete state trace.**

Answer: Say the document starts empty. User types "hello" (one command, call it C1) → `undoStack: [C1]`, `redoStack: []`. User undoes → document is empty again, `undoStack: []`, `redoStack: [C1]`. Now, instead of redoing, the user types "hey" (command C2, applied to the current, empty document) → `undoStack: [C2]`. If `redoStack` weren't cleared here, it would still contain `[C1]` ("hello"), and a subsequent `redo()` would re-apply C1's `do()` — inserting "hello" — on top of a document that now already contains "hey", producing "heyhello" or some equally nonsensical result, a state that never existed in the actual sequence of things the user did. Clearing `redoStack` on every new push ensures redo can only ever replay a timeline that's still consistent with the current one.

The trap: describing the two-stack model correctly but failing to mention the clear-on-push rule unprompted — this is precisely the detail interviewers use to separate "read about undo/redo once" from "actually implemented and debugged one."

---

**Q (High): Compare the command pattern and the snapshot/memento approach in terms of memory and implementation risk. When would you choose each?**

Answer: Command pattern stores, per history entry, only the information needed to apply and reverse one discrete action — memory cost scales with the size of the *change*, independent of total document size, which is why it's the right choice for a large or complex document (a canvas editor with many shapes, a large text document) where full-state snapshots would be expensive per step. The implementation risk is that every action type needs its own correctly-written inverse, and subtle bugs (an undo that doesn't perfectly restore prior formatting, or that fails on an edge case the forward action doesn't hit) are easy to introduce and easy to miss in testing, since each command type is a separate surface area. Snapshot/memento stores a full (or diffed) copy of state at each point — implementation risk is much lower, since there's no per-action inverse logic at all, just "restore this blob" — but memory cost scales with document size × history depth, which becomes a real problem for large state unless mitigated with structural sharing or diffing. I'd choose command pattern for a document-heavy editor with many distinct mutation types and meaningful document size, and snapshot for smaller, simpler, or more homogeneous state where correctness-by-construction outweighs the memory cost.

The trap: presenting one as strictly "better" rather than naming the actual axis of trade-off (memory efficiency vs. implementation correctness risk) — a senior answer frames this as a genuine trade-off decided by the shape of the state being tracked, not a universal best practice.

---

**Q (High): How would you batch rapid, continuous changes (like typing) into a single undo step, and what determines where one batch ends and the next begins?**

Answer: Rather than committing a new undo-stack entry on every keystroke, changes accumulate into an "open" batch (e.g., a growing string being inserted, or a shape's position updating as it's dragged), and that batch only gets pushed onto the undo stack as a single command when a **commit boundary** is reached. Two complementary mechanisms typically define that boundary: an idle timer that keeps extending as long as new changes keep arriving within a short window (so a pause in typing — around 500ms to a second — signals "this burst is done"), and explicit interaction-driven boundaries that map to real semantic breaks (a text field losing focus, a drag gesture's `mouseup`/`touchend`, or punctuation like Enter or a period that naturally ends a thought). Choosing the exact boundary is a UX judgment call, not a purely technical one — too short a timer re-fragments what should be one undo step; too long makes undo feel unresponsive because a lot of unrelated typing gets glued together.

The trap: implementing only the idle-timer mechanism and missing the explicit boundaries — without a blur/mouseup handler forcing a commit, a batch can be left "open" indefinitely if the idle timer never fires (e.g., the user switches away from the tab mid-typing-burst before the timer elapses), and the very next unrelated action could get silently merged into that stale open batch.

---

**Q (Medium): How does a naive linear undo/redo stack break down in a collaborative, multi-user editing scenario, and how do real systems (like Google Docs or CRDTs-based editors) address it?**

Answer: A single linear undo/redo stack implicitly assumes there's exactly one actor producing a single, well-ordered sequence of changes to undo through. The moment a second user can concurrently mutate the same document, "undo my last action" becomes ambiguous the instant another user's changes are interleaved with yours — undoing your last local action might need to reverse a change that a remote edit has since built on top of (e.g., you inserted a paragraph, someone else formatted text inside it, then you undo the insertion — does the formatting change go with it, get orphaned, or block the undo?). Real collaborative editors generally solve this with either per-user local undo stacks that store enough context to correctly rebase an undo against intervening remote operations (often built on operational transformation or CRDT primitives that make operations commutative/reversible in a way that's well-defined even when reordered), or by scoping undo to only reverse operations that are provably still "on top" of the document in a consistent way, sometimes falling back to disabling undo for an action once it's been built upon by another user. This is meaningfully deeper than the single-user case and belongs in a dedicated system-design discussion, not a from-scratch implementation in this kind of exercise — but a senior candidate should recognize and name the problem rather than assume the same two-stack model just works unmodified.

The trap: either not recognizing the problem exists at all (assuming undo "just works" the same way with multiple collaborators) or trying to over-solve it live in the interview — the right depth here is naming the failure mode and the general class of solution (OT/CRDT-aware undo) without attempting to design the full mechanism on the spot.

---

**Q (Medium): How would you bound the undo history's memory footprint for a long editing session, and what does the user lose as a result?**

Answer: Treat the undo stack as a fixed-capacity structure — a ring buffer, effectively — where pushing a new entry once at capacity evicts the oldest entry rather than growing unboundedly. The direct cost to the user: once an entry has been evicted, there is no way to undo back past that point for the rest of the session, no matter how many times Ctrl+Z is pressed — the history genuinely only goes back N steps. This is an accepted, standard trade-off in virtually every real editor (most have a finite, if generous, undo depth) rather than a compromise unique to this implementation; the alternative of truly unbounded history risks unconstrained memory growth in a long-lived session, which is the worse failure mode of the two.

The trap: proposing an unbounded stack "since memory is cheap" without acknowledging that a long-running session (hours of continuous editing, common in real editors) can still accumulate enough history to become a real, measurable memory problem, especially for a snapshot-based design where each entry can be large.

---

**Q (Medium): If two different parts of the UI (say, a text editor pane and a separate properties panel) both need undo/redo, would you use one shared history or two independent ones?**

Answer: This depends on whether the two panes represent genuinely independent pieces of state or two views onto the same underlying document. If the properties panel is editing attributes of objects that live in the same document the text editor is mutating (e.g., a diagram editor where the panel edits a selected shape's color), a single shared undo stack is usually correct — the user's mental model of "undo my last action" spans the whole document, regardless of which UI surface performed it, and a shared stack keeps a coherent global timeline. If instead the two are truly independent domains (e.g., undo history for document content vs. undo history for an unrelated settings panel that doesn't affect the document), separate stacks better match user expectations, since undoing a settings change while focused on text editing would otherwise be surprising. The general principle: undo scope should match the user's mental model of "what one undo-able unit of work is," not the UI's component boundaries.

The trap: defaulting to "one stack per component" purely because that maps cleanly onto the code's module boundaries — undo scope is a product/UX decision about what users perceive as one coherent timeline, not a decision that should be driven by component architecture.

---

**Q (Low): How would you persist undo history across a page reload (e.g., the user accidentally closes the tab mid-edit)?**

Answer: The command pattern's closures don't serialize (a `do`/`undo` pair captured as JS functions can't be written to `localStorage` or sent over the network as-is), so persisting history typically requires either switching to a serializable representation for storage — plain data objects describing each operation (`{ type: 'insert', position, text }`) that get *re-hydrated* into executable commands on reload by looking up the operation type in a registry of known command constructors — or falling back to periodic full-document snapshots (simpler to serialize, but loses fine-grained undo granularity across the reload boundary, typically collapsing everything before the reload into a single "restored" baseline state with no further undo past it).

The trap: assuming you can just `JSON.stringify` the command objects directly — functions aren't serializable to JSON at all, so anything closure-based needs an explicit data/command-type separation before persistence is possible.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain the command-pattern vs. snapshot trade-off and justify a choice for a given state shape
- [ ] Can implement the two-stack undo/redo model, including the redo-stack-clear-on-new-action rule, from memory
- [ ] Can trace a concrete undo/redo/new-action sequence and state exactly what's in each stack at every step
- [ ] Can design a commit-boundary/batching mechanism for continuous input like typing or dragging
- [ ] Can explain why an unbounded history is a real problem and how a ring-buffer cap addresses it
- [ ] Can name, at a high level, why collaborative editing breaks a naive linear undo stack

---
*Next: Nested Comments — Recursive Tree Rendering — a data-shape-and-recursion problem, closing out Phase 2 before Phase 3 shifts into React-specific debugging.*
