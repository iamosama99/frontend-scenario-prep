# Undo/Redo — Architecture Decision

## Quick Reference

| Approach | Mechanism | Best For | Weakness |
|---|---|---|---|
| Command pattern (do/undo pairs) | Each action stores enough info to reverse itself | Discrete, well-defined actions (delete row, move item) | Every new action type needs its own undo logic written |
| Snapshot/patch history | Store full (or diffed) state snapshots per step | Complex, hard-to-invert state (a canvas, rich text) | Memory cost for full snapshots; still need diffing for efficiency |
| Event sourcing (replay from log) | Store the sequence of events; state is derived by replaying | Collaborative apps, audit trail needed anyway | Overkill for a simple undo button; replay cost grows with history |
| Server-side undo (compensating action) | Client asks server to reverse a prior committed mutation | Actions already committed server-side, multi-device | Requires the server to support a reverse operation; not instant |

## The Scenario

"Add undo/redo to a task board — users can delete a task, move it between columns, edit its title, or reorder it within a column, and any of these should be undoable with Cmd+Z (and redoable with Cmd+Shift+Z), including after the action has already been saved to the server. Design the architecture for this — not just 'use a library,' but how state needs to be structured to support it."

## Clarifying Questions

- **Does undo need to work across a page refresh/reload, or only within the current session/tab?** This is the single biggest fork — an in-memory undo stack (simplest, most common for this kind of feature) is fine if undo history can be lost on refresh, but "undo something from earlier even after reloading the page" requires persisting the undo history itself, which is a meaningfully bigger scope.
- **Once an action has been saved to the server (the task move actually persisted), does "undo" mean reversing it locally-and-then-syncing-the-reversal to the server, or is there a genuinely separate server-side undo/history mechanism (e.g., a real audit log the server can replay against)?** The scenario says "including after the action has already been saved" — I'd clarify whether the expectation is client-computes-the-inverse-and-sends-it (simpler, works for most CRUD-shaped actions) versus a dedicated server endpoint that reverses a specific historical mutation (needed if the action has non-trivial server-side side effects beyond a simple field update, like triggering a notification that shouldn't re-fire on undo).
- **Is this single-user, or can multiple users be editing the same board concurrently?** Multiplayer undo is a substantially harder problem — "undo my last action" needs to specifically target *my* last action, not just "the last action on the board," which might have been someone else's; a global undo stack shared across users is almost never the right model once collaboration is in scope.
- **How many steps of undo history are actually needed — a handful of recent actions, or effectively unlimited?** Affects whether an in-memory array of commands is sufficient or whether some form of capping/eviction (with a decision about what happens when undo history is truncated) is needed.
- **Do all four action types (delete, move, edit title, reorder) need to compose into a single combined undo stack (undo goes back through the true chronological sequence regardless of action type), or could each be tracked somewhat independently?** A single interleaved stack is almost certainly what "Cmd+Z" implies to a user (undo my literal last action, whatever type it was) — I'd confirm this rather than assume, since a per-type stack is a meaningfully different (and usually wrong) architecture for a general undo feature.

## Approach & Trade-offs

**The command pattern is the right fit here, not full snapshotting or event sourcing, because the four action types are discrete and each has a well-defined, cheap-to-compute inverse.** Deleting a task is undone by re-inserting it (with its original data and position); moving a task between columns is undone by moving it back; editing a title is undone by restoring the previous title; reordering is undone by restoring the previous index. Each of these is a small, explicit "do this to reverse that" operation — full-state snapshotting (storing the entire board's state before and after every action) would work but is wasteful here, since the board can be large and each individual action only ever touches a small, well-understood piece of it; snapshotting earns its cost specifically when actions are *hard to invert* (arbitrary canvas edits, rich text formatting operations where "the inverse" isn't a clean, small concept) — a task board's discrete CRUD-shaped actions aren't that.

**Every undoable action should be represented as a command object with `do()` and `undo()`, constructed at the moment the action happens — not derived after the fact by diffing.** Deriving "what changed" from a before/after snapshot comparison is strictly harder and lossier than capturing intent at the source: at the moment a user drags a task to a new column, the code already knows exactly "task X moved from column A, index 2 to column B, index 0" — encoding that directly into a command object (`{ type: 'move', taskId, from: {columnId, index}, to: {columnId, index} }`) is both simpler to write and produces a precise, cheap-to-reverse record, versus snapshotting the whole board before and after and diffing it to reconstruct the same information.

**The undo stack and the "committed to server" state need to be modeled as two related but distinct things, because undoing a server-committed action means issuing a new, real mutation (the inverse), not silently rewriting history.** This matters for correctness in a way that's easy to gloss over: if a task move was already saved to the server, and the user hits Cmd+Z, the correct behavior is to send a new "move it back" request — not to somehow retract the original request (which may already have triggered side effects, like a notification to a teammate, that can't be un-sent) — and if that inverse request fails, the undo itself needs its own failure handling (using the exact optimistic-update-with-rollback pattern from the previous scenario, applied to the *undo* action). This means the undo stack conceptually holds "how to construct the inverse mutation," and *executing* an undo is itself just another mutation going through the normal optimistic-update pipeline — undo isn't a separate state-management mechanism bolted on top, it's a way of *generating* new, ordinary actions.

**Redo is the exact mirror of undo and falls out for free if commands are modeled with both `do()` and `undo()` symmetrically** — undoing pushes the command onto a redo stack; redoing calls `do()` again and pushes it back onto the undo stack; any *new* action performed after an undo should clear the redo stack (the standard behavior every text editor and design tool follows) rather than trying to reconcile a forked history, since supporting genuine history branching is a much bigger feature nobody is asking for here.

**For a single-user board, a simple linear stack (array) of commands is sufficient; the moment collaboration enters the picture, the model breaks and needs to change to "each user has their own undo stack of their own actions," which only reverses actions that specific user performed** — because a shared global stack would let User A's Cmd+Z undo User B's unrelated action, which is almost never the intended behavior and would be actively confusing/dangerous in a shared workspace. I'd flag this explicitly as the reason the clarifying question about multi-user editing matters architecturally, not just as a minor scope note.

## Solution

**1. Command shape — captures intent directly, not derived from diffing:**

```tsx
type Command =
  | { type: 'delete'; task: Task; columnId: string; index: number }
  | { type: 'move'; taskId: string; from: { columnId: string; index: number }; to: { columnId: string; index: number } }
  | { type: 'editTitle'; taskId: string; previousTitle: string; newTitle: string }
  | { type: 'reorder'; taskId: string; columnId: string; previousIndex: number; newIndex: number };
```

**2. The undo/redo stack, and the mapping from a command to its execute/reverse mutation:**

```tsx
function useUndoRedo() {
  const undoStack = useRef<Command[]>([]);
  const redoStack = useRef<Command[]>([]);
  const { mutate: applyBoardMutation } = useBoardMutation(); // wraps the existing optimistic-update pipeline from Scenario 3

  function performAndRecord(command: Command) {
    applyBoardMutation(toMutationInput(command));
    undoStack.current.push(command);
    redoStack.current = []; // any new action invalidates the redo history — standard editor behavior
  }

  function undo() {
    const command = undoStack.current.pop();
    if (!command) return;
    applyBoardMutation(toMutationInput(invert(command))); // executes the inverse as a normal, ordinary mutation
    redoStack.current.push(command);
  }

  function redo() {
    const command = redoStack.current.pop();
    if (!command) return;
    applyBoardMutation(toMutationInput(command)); // re-executes the original — the exact same code path as the first time
    undoStack.current.push(command);
  }

  return { performAndRecord, undo, redo };
}
```

**3. Inverting a command — the core of why this pattern is cheap for this problem:**

```tsx
function invert(command: Command): Command {
  switch (command.type) {
    case 'delete':
      return { type: 'insert', task: command.task, columnId: command.columnId, index: command.index } as any; // re-insertion is the inverse of delete
    case 'move':
      return { ...command, from: command.to, to: command.from }; // swap from/to
    case 'editTitle':
      return { ...command, previousTitle: command.newTitle, newTitle: command.previousTitle }; // swap old/new
    case 'reorder':
      return { ...command, previousIndex: command.newIndex, newIndex: command.previousIndex };
  }
}
```

**4. Keyboard binding, scoped so it doesn't fire while the user is typing in an unrelated input:**

```tsx
useEffect(() => {
  function handleKeydown(e: KeyboardEvent) {
    const isModifier = e.metaKey || e.ctrlKey;
    if (!isModifier || e.key.toLowerCase() !== 'z') return;
    if (isEditableElementFocused(document.activeElement)) return; // don't hijack undo inside a text field's native undo

    e.preventDefault();
    e.shiftKey ? redo() : undo();
  }
  window.addEventListener('keydown', handleKeydown);
  return () => window.removeEventListener('keydown', handleKeydown);
}, [undo, redo]);
```

> **Check yourself:** Why does `undo()` push the *original* command onto the redo stack, rather than pushing the inverted command it just executed?

## Handling Server-committed Undo Failures

If undoing a "move" (sending the inverse move to the server) itself fails — say, the task was deleted by someone else in the meantime:

```tsx
function undo() {
  const command = undoStack.current.pop();
  if (!command) return;

  applyBoardMutation(toMutationInput(invert(command)), {
    onError: () => {
      // the undo attempt itself failed — don't silently push it to redo as if it succeeded
      notifyError('Could not undo that action — the board may have changed since.');
      // command is deliberately NOT pushed to redoStack, since redo would replay
      // a now-invalid original action on top of a board state that's diverged
    },
    onSuccess: () => {
      redoStack.current.push(command);
    },
  });
}
```

## Gotchas

**Deriving undo commands by diffing before/after snapshots instead of capturing intent at the moment of the action.** Loses precision (a diff of "task moved" can be ambiguous to reconstruct exactly, especially with reordering within a list) and is strictly more work than just recording what already happened, since the code performing the action already knows exactly what changed.

**Treating undo of an already-server-committed action as "rewriting history" instead of "issuing a new, normal inverse mutation."** The original action already happened, possibly with side effects (a notification sent, an audit-log entry written) that undo can't retroactively erase — undo should create a new fact ("moved back"), not pretend the first fact never occurred.

**A single global undo stack in a multi-user context, letting one user's Cmd+Z undo another user's action.** Feels correct in single-player testing and is actively wrong the moment a second user is editing concurrently — this needs per-user attribution on the stack from the start if collaboration is anywhere on the roadmap.

**Not clearing the redo stack when a new action is performed after an undo.** Without this, redo can reapply a stale action on top of a board state that's since diverged in an unrelated way, producing a result the user never actually had and didn't ask for — standard editors always drop redo history the moment a new distinct action is taken.

**Binding Cmd+Z globally without excluding focused text inputs.** Hijacks the browser/input's own native undo behavior while someone is editing a task title inline, which is actively worse than doing nothing — text field undo is a separate, expected mechanism that a global board-level undo shouldn't override.

**Pushing a command onto the redo stack before confirming its undo (the inverse mutation) actually succeeded.** If the inverse mutation fails, the redo stack now contains an action whose "undo" never actually completed, so a subsequent redo replays the original action on top of a board that was never actually reverted — compounding the inconsistency.

## Follow-up Questions

**Q (High): Why is the command pattern (explicit `do`/`undo` per action) the right choice here instead of storing full board-state snapshots before and after every action?**

Answer: Because the actions in scope (delete, move, edit title, reorder) are all discrete, well-understood, and each has an inverse that's cheap and unambiguous to compute directly — a snapshot approach would work but pays a real cost (storing potentially large amounts of board state per undo step, or building a diffing mechanism to avoid that) to solve a problem that direct command-based inversion already solves more cheaply and more precisely. Snapshotting earns its complexity specifically for state that's *hard to invert directly* — free-form canvas drawing, rich text with overlapping formatting ranges, anything where "what's the single clean inverse of this action" doesn't have an obvious small answer — which isn't the case for a task board's CRUD-shaped operations.

The trap: reaching for the "more general" snapshot/event-sourcing approach by default because it feels more robust, without recognizing that the actual action set here doesn't need that generality and pays real, avoidable cost (memory, diffing complexity) for it.

---

**Q (High): The user deletes task A, then moves task B, then hits undo twice. Walk through exactly what the undo stack and board state look like at each step, and what happens if the *second* undo (reversing the delete) fails on the server.**

Answer: After the two actions, `undoStack = [deleteA, moveB]` (moveB on top). First Cmd+Z pops `moveB`, executes its inverse (moves B back to where it was), pushes `moveB` onto `redoStack`; board now shows B reverted, A still deleted, `undoStack = [deleteA]`. Second Cmd+Z pops `deleteA`, attempts its inverse (re-insert A) as a real mutation to the server; if that fails (say, a conflict, or a validation error re-inserting into a column that's since changed), the `onError` path fires: the board is left showing task A still deleted (the re-insertion never actually landed), `deleteA` is *not* pushed onto the redo stack (since its undo didn't actually succeed — pushing it would let a later redo "redo the delete" that's already effectively still in place, which is confusing and wrong), and the user sees an explicit error rather than the UI silently claiming the undo worked. `undoStack` is now empty; the user's subsequent actions proceed from this actual, correct state rather than an assumed one.

The trap: assuming every undo step succeeds and glossing over the failure path — a strong answer explicitly walks through what the stacks and visible board state look like when an undo *itself* fails, not just the happy path.

---

**Q (High): If this board supports real-time collaboration (multiple users editing simultaneously via websocket-pushed updates from other users), what has to change about this undo architecture, concretely?**

Answer: The undo/redo stacks need to become per-user (scoped to actions *this* client performed), not a single shared stack — each client only tracks and can undo/redo its own history of commands, never another user's. Beyond that ownership scoping, there's a subtler correctness issue: a command's `undo()` computes its inverse based on the state *at the time the action was originally performed* (e.g., "move task from column A index 2 back to column A index 2"), but if other users' actions have landed on the board in the meantime (someone else reordered that same column), reapplying an inverse based on stale positional assumptions (a specific index) can land the task in the wrong place relative to the *current* board state. This pushes toward inverses expressed in terms more resilient to concurrent changes where possible (e.g., "move task back to column A, positioned relative to task Y" rather than a raw numeric index) or, if that's not practical, accepting that undo in a live-collaborative context is inherently best-effort and can occasionally produce a slightly different result than the user expects, which is a genuine, known trade-off in real collaborative-editing undo systems (and part of why some collaborative tools scope undo very conservatively, e.g., only undoing your own most recent action if nothing else has touched the same object since).

The trap: assuming per-user stack scoping alone fully solves multi-user undo — the deeper issue (an inverse computed against a since-changed base state) is the harder part, and a strong answer surfaces it even without fully solving it.

---

**Q (Medium): Does undo/redo interact with the optimistic-update-with-rollback pattern from the previous scenario, or are they separate mechanisms?**

Answer: They compose — executing an undo is just performing an ordinary mutation (the inverse command), which should go through the exact same optimistic-update pipeline as any other action: it updates the UI immediately, and if the server rejects it, that failure is handled with the same snapshot/rollback mechanics, just with an undo-specific consideration layered on top (as shown above: don't push onto the redo stack until the undo mutation is confirmed, and communicate an undo-specific failure message). Framing undo as "just another mutation, generated by inverting a recorded command" rather than a separate state-management system is what keeps the two mechanisms from needing duplicated logic — the undo stack decides *what* mutation to fire next; the optimistic pipeline handles *how* that mutation's UI feedback and failure handling behaves, same as always.

The trap: building a separate, bespoke state-update mechanism specifically for undo actions instead of recognizing that an undo is just a normal mutation with a computed payload, and should reuse the existing mutation/optimistic-update machinery rather than duplicating it.

---

**Q (Medium): How would you cap undo history to, say, the last 50 actions without breaking correctness?**

Answer: Cap the size of the `undoStack` array (evict the oldest entry, e.g., `shift()`, once it exceeds 50), which is safe precisely because each command is self-contained (it doesn't reference or depend on earlier commands still being present in the stack to compute its own inverse) — evicting old entries just means those actions become permanently un-undoable past that point, with no correctness risk to the remaining stack. The one thing to get right: eviction should happen on push (when a new action is recorded), not retroactively try to "compress" older history, since there's no meaningful way to merge two unrelated commands into one without changing what undo actually reverses.

The trap: worrying that capping the stack requires some kind of compaction or merging logic — because each command is independent and self-describing, capping is just a straightforward bounded-queue eviction, no different in principle from any other bounded history buffer.

---

**Q (Low): Would you ever want to debounce/coalesce a sequence of rapid `editTitle` keystrokes into a single undo step, rather than one undo step per keystroke?**

Answer: Yes — if `editTitle` commands were recorded per-keystroke (as they would be if wired directly to an `onChange`), Cmd+Z would feel broken to a user (undoing one character at a time instead of undoing "the edit" as a conceptual unit), which doesn't match how undo works in virtually every text-editing context users are used to. The fix is coalescing at the point where the command is *recorded*, not where it's applied — record a single `editTitle` command only on blur/commit of the edit (capturing the title as it was when editing started versus its final committed value), rather than one command per keystroke; if live intermediate undo *within* an in-progress edit is wanted, that's better served by the input's own native undo (which already does exactly this) rather than the board-level command stack.

The trap: recording an undo command on every state change indiscriminately (every keystroke, every intermediate drag position during a reorder) without considering what a user actually perceives as "one action" — undo granularity should match user intent, not implementation-level state-update frequency.

---

## Self-Assessment

- [ ] Can explain why the command pattern fits discrete CRUD-shaped actions better than snapshotting, and name what kind of state *would* justify snapshotting instead
- [ ] Can describe why undoing a server-committed action means issuing a new inverse mutation, not rewriting history
- [ ] Can walk through what happens to the undo/redo stacks when an undo attempt itself fails
- [ ] Can explain why a single global undo stack breaks in a multi-user context and what changes to fix it
- [ ] Can explain why redo history must be cleared on any new action after an undo
- [ ] Can reason about coalescing rapid-fire changes (keystrokes) into a single undo step at the point of recording, not application

---
*Next: Cross-tab State Sync — shifts from a single tab's history/undo problem to keeping state consistent when the same user has the same app open in multiple tabs simultaneously.*
