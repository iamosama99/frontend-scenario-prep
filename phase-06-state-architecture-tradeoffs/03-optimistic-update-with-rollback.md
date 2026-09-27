# Optimistic Update With Rollback

## Quick Reference

| Concern | Mechanism | Why |
|---|---|---|
| Update UI before server confirms | Write to cache immediately on mutation start | Perceived responsiveness — no spinner for a likely-to-succeed action |
| Preserve ability to undo | Snapshot previous cache state before mutating | Rollback needs to restore *exactly* what was there, not a guessed default |
| Handle server rejection | `onError` restores the snapshot | Keeps UI truthful — never leaves a false optimistic state permanently displayed |
| Reconcile with real server response | `onSettled`/`onSuccess` refetch or merge server truth | Optimistic value is a guess; server response is authoritative and may differ (e.g., server-computed fields) |
| Avoid racing updates | Request/version tokens or cache-level mutation queue | A slow first request resolving after a faster second one must not clobber newer state |

## The Scenario

"On the orders table, add a 'mark as shipped' action per row, and a 'like' button on a social feed post — both should feel instant when clicked, not wait for a network round trip before updating. Implement optimistic updates for both, and handle what happens when the server request actually fails after the UI already showed the optimistic result."

## Clarifying Questions

- **What does the server actually return on success — just a 200, or a modified/authoritative version of the resource (e.g., a `shippedAt` timestamp, an updated `likeCount` that might differ from a naive client-side increment due to concurrent likes from other users)?** If the server returns authoritative data that can differ from what the client optimistically guessed, I need a reconciliation step after success, not just "leave the optimistic value in place forever" — this is especially true for a like count, where other users liking concurrently means the client's `count + 1` guess is very likely to be wrong by the time the real response arrives.
- **Is "mark as shipped" a reversible action from the user's side (can they immediately undo it if they clicked the wrong row), or does clicking it start an irreversible process (e.g., triggers an actual shipping label/carrier pickup)?** This changes whether I need a client-side "undo" affordance in addition to server-failure rollback — an irreversible action needs a confirmation step before the optimistic update even fires, not just graceful rollback after the fact.
- **What should the error state actually communicate to the user if the mutation fails — a silent revert, a toast explaining what happened, or something more prominent given the action (shipping a wrong order) has real-world consequences?** A "like" failing silently reverting is barely noticeable and fine; an order incorrectly appearing "shipped" then silently reverting without any explanation could leave a user confused about whether the order actually shipped or not — the failure-communication bar scales with the action's stakes.
- **Can the same order be marked shipped from multiple places concurrently (e.g., two support agents both interacting with the same order), and if the mutation fails because someone else already changed its state, is that a "retry" situation or a "show me the real current state" situation?** This affects whether rollback should restore the *pre-optimistic* local snapshot or instead fetch and show the actual current server state, since those could differ if someone else's change landed in between.
- **Is there a rate limit or debounce concern for rapid repeated clicks (e.g., double-clicking "like" quickly, or a nervous double-click on "mark as shipped")?** Without protecting against this, a fast double click can fire two mutations, and the rollback/reconciliation logic needs to handle overlapping in-flight requests for the same resource correctly, not just the single-request-at-a-time case.

## Approach & Trade-offs

**The core pattern is: snapshot, apply, confirm-or-rollback — and the part most candidates get wrong under time pressure is skipping the snapshot and trying to "compute the rollback value" after the fact.** The temptation with a like button is to write `setLiked(true); setCount(c => c + 1)` optimistically, and on failure, write `setLiked(false); setCount(c => c - 1)` — this looks correct for a single isolated action, but it's fragile the moment anything else can also be changing that same count concurrently (another user's like landing via a real-time update, or a second overlapping request), because "subtract 1" assumes the only thing that changed the count since the optimistic update was this specific failed mutation, which isn't guaranteed. The more robust pattern captures the actual previous value *before* mutating (`const previous = queryClient.getQueryData(key)`) and restores that exact previous value on failure, rather than trying to algebraically undo the optimistic delta.

**React Query's mutation lifecycle (`onMutate` / `onError` / `onSettled`) maps directly onto this pattern and is worth using even if the underlying transport were hand-rolled, because it names the three moments explicitly** — `onMutate` is where the snapshot-then-optimistic-write happens (and it can return the snapshot as context, threaded automatically into `onError`), `onError` is purely "restore what `onMutate` snapshotted, nothing computed," and `onSettled` (runs on success *or* failure) is where I'd trigger a refetch to reconcile with server truth regardless of outcome — because even on success, the actual authoritative value (a real `likeCount` reflecting concurrent likes from others) might differ from the naive optimistic guess, and only a refetch (or a success payload that includes the authoritative value directly) closes that gap.

**"Mark as shipped" and "like a post" look similar but differ in stakes, and the design should reflect that rather than using one generic optimistic-mutation hook for both with no distinction.** A like's failure mode is cosmetic and reversible with zero real-world consequence — silent rollback, maybe a barely-noticeable toast, is entirely sufficient. Marking an order shipped is a business-state transition with downstream consequences (a warehouse or carrier integration might key off this) — for this one, I'd lean toward *not* firing the mutation as fire-and-forget optimistic the same way, or at minimum making the failure state loud and unmissable (a persistent, dismiss-required error notification, not an auto-fading toast) precisely because a user could otherwise walk away believing an order shipped when it didn't, or vice versa if a rollback happens silently while they've already moved on to the next task.

**Concurrent/overlapping mutations on the same resource need a way to know "is my rollback still valid, or has something newer already superseded it."** If a user rapid-double-clicks "mark as shipped," two mutations are in flight; if the first fails and its `onError` blindly restores its snapshot, it could stomp on the second mutation's optimistic update (or its own success) that landed in between — restoring stale data over newer, possibly-already-confirmed data. React Query handles a good chunk of this by default for a single query key (mutations targeting the same query are effectively serialized in terms of what the cache reflects at each point, and `onSettled`'s refetch resolves the ambiguity), but the general principle — don't let an older request's error handler clobber a newer request's result — is worth being able to state explicitly, and matters especially in a hand-rolled (non-React-Query) implementation where nothing enforces it automatically.

## Solution

**1. Like button — low-stakes, silent-rollback optimistic update:**

```tsx
function useLikePost(postId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: () => likePost(postId),

    onMutate: async () => {
      await queryClient.cancelQueries({ queryKey: ['post', postId] }); // avoid a stale in-flight refetch overwriting the optimistic write

      const previous = queryClient.getQueryData<Post>(['post', postId]);

      queryClient.setQueryData<Post>(['post', postId], (old) =>
        old ? { ...old, liked: true, likeCount: old.likeCount + 1 } : old
      );

      return { previous }; // snapshot threaded into onError
    },

    onError: (_err, _vars, context) => {
      if (context?.previous) {
        queryClient.setQueryData(['post', postId], context.previous); // restore exact prior state, not a computed undo
      }
      // low stakes — a subtle toast is enough, no blocking modal
      toast.error('Could not like this post — please try again.');
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['post', postId] }); // reconcile with authoritative likeCount either way
    },
  });
}
```

**2. Mark as shipped — higher-stakes, loud-failure optimistic update:**

```tsx
function useMarkShipped(orderId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: () => markOrderShipped(orderId),

    onMutate: async () => {
      await queryClient.cancelQueries({ queryKey: ['order', orderId] });
      const previous = queryClient.getQueryData<Order>(['order', orderId]);

      queryClient.setQueryData<Order>(['order', orderId], (old) =>
        old ? { ...old, status: 'shipped', shippedAt: new Date().toISOString() } : old
      );

      return { previous };
    },

    onError: (_err, _vars, context) => {
      if (context?.previous) {
        queryClient.setQueryData(['order', orderId], context.previous);
      }
      // higher stakes — persistent, explicit notification, not an auto-fading toast
      notifyError({
        title: 'Order was not marked shipped',
        message: 'The status shown has been reverted. Please retry, or check if this order was already updated elsewhere.',
        persistent: true,
      });
    },

    onSuccess: (serverOrder) => {
      // server is authoritative — e.g. it may set shippedAt with server clock time, not the client's optimistic guess
      queryClient.setQueryData(['order', orderId], serverOrder);
    },
  });
}
```

> **Check yourself:** In the `onMutate` for `useMarkShipped`, why snapshot and restore `previous` wholesale instead of just flipping `status` back to whatever it "probably" was before?

## Handling the Concurrent-Mutation Race

If a second "mark as shipped" click fires before the first request resolves (double-click), and the *first* request fails while the *second* succeeds:

```tsx
// React Query serializes cache effects per mutation in call order by default,
// but onSettled's invalidateQueries is the real safety net here —
// it always re-fetches actual server truth after every mutation settles,
// so even if onError's rollback from the first (failed) request runs after
// the second (succeeded) request already wrote its optimistic/success state,
// the subsequent invalidation-triggered refetch corrects the cache to match
// the real, current server state rather than trusting either client-side guess.
```

The practical mitigation that avoids relying on subtle ordering guarantees at all: disable the action's button/control while a mutation for that same resource is in flight (`mutation.isPending`), preventing the double-click race from occurring in the first place — simpler and more robust than reasoning carefully about interleaved optimistic-update ordering.

## Gotchas

**Computing the rollback value algebraically (`count - 1`) instead of snapshotting and restoring the actual previous state.** Breaks the moment anything else can concurrently affect the same piece of state — a second in-flight mutation, a real-time update from another user — because the algebraic undo assumes the only change since the optimistic write was this one failed mutation.

**Treating every optimistic update as equally low-stakes and using the same silent-toast failure UX for a "like" and a "mark as shipped."** The right failure communication scales with the real-world consequence of the user believing the wrong thing happened — a cosmetic action needs a whisper, a business-state-changing action needs a shout.

**Forgetting `cancelQueries` before writing the optimistic update.** Without it, a refetch already in flight when the mutation starts can resolve *after* the optimistic write and silently overwrite it with (now-stale) pre-mutation server data, making the optimistic update flicker or appear to have not worked.

**Not disabling the triggering control while a mutation is in flight, enabling rapid double-fires.** Even with correct snapshot/rollback logic, allowing a second identical mutation to fire before the first resolves multiplies the number of race conditions that need to be reasoned about correctly, for no user benefit.

**Treating `onSuccess`'s server response as optional to apply, leaving the client's optimistic guess in place permanently.** If the server can return authoritative data that legitimately differs from the client's guess (a `likeCount` affected by concurrent likes from others, a server-computed `shippedAt` timestamp), skipping the reconciliation step leaves the UI showing a value that's provably wrong the moment any other actor touches the same resource.

## Follow-up Questions

**Q (High): Walk through exactly what happens, step by step, if the "mark as shipped" mutation fails with a 409 Conflict because another agent already shipped the order five seconds earlier.**

Answer: `onMutate` already fired optimistically, showing "shipped" locally the instant the button was clicked. The request comes back as a 409, landing in `onError` — the snapshot restore (`previous`) would revert the local cache to what it was *before this user's own optimistic update*, which is the *original* pre-click state, not the order's actual *current* (already-shipped-by-someone-else) state — meaning a naive rollback would incorrectly show the order as "not shipped" even though it genuinely now is shipped, just not by this action. The correct handling for a conflict specifically (versus a generic network failure) is to treat it differently from a plain rollback: on a 409, I'd trigger a refetch of the real current server state (`queryClient.invalidateQueries` or a direct refetch) instead of restoring the stale pre-optimistic snapshot, and show a message reflecting the actual situation ("This order was already marked shipped by someone else") rather than a generic "action failed, please retry" — retrying the same action here would be pointless since the desired end state is already true, just not because of this user's click.

The trap: applying the same blanket rollback-to-previous-snapshot logic to every error without distinguishing "the action genuinely failed, revert" from "the action's goal-state is already true via a different path, refresh to reflect reality" — a 409/conflict response is a strong signal it's the latter.

---

**Q (High): Why cancel in-flight queries (`cancelQueries`) before writing the optimistic update in `onMutate`, specifically — what breaks if that line is omitted?**

Answer: Without cancelling, a `refetch` or background query that was already in flight *before* the mutation started (e.g., the query naturally refetching on window focus, or a previous unrelated refetch still pending) can resolve *after* the optimistic write lands, and since that in-flight request's response reflects the pre-mutation server state, its resolution overwrites the freshly-written optimistic value with stale data — producing a visible flicker (optimistic update shows, then reverts back to the old value for a moment, then the mutation's own eventual response corrects it again) that looks like a bug even though the mutation itself succeeded. Cancelling any in-flight query for that key before writing ensures nothing stale can land on top of the fresh optimistic write until the mutation's own lifecycle resolves it.

The trap: assuming the only source of a value overwrite is the mutation's own response — background refetches from other triggers (focus refetch, a polling interval, another component's `useQuery` for the same key) are just as capable of racing against an optimistic write.

---

**Q (High): The product wants the "mark as shipped" action to require confirmation before firing (since it's semi-irreversible) — does that change whether it should be optimistic at all?**

Answer: A confirmation step and optimism aren't mutually exclusive — the confirmation gates *whether the mutation fires at all* (a modal or inline confirm before calling `mutate()`), while optimism governs *what the UI shows in the gap between the mutation firing and the server responding*, and both can coexist: user confirms, click fires the mutation, the UI still updates instantly afterward rather than showing a spinner during the network round trip, because "confirmed intent" and "wait for network before showing any change" are separate concerns. Where I would reconsider optimism entirely is if the action's server-side effect is slow or has meaningful uncertainty about outcome even under normal (non-error) conditions — e.g., if "mark as shipped" actually triggers a real carrier API call that can itself take several seconds and sometimes legitimately fails for reasons outside a simple network blip (address validation failure, carrier service down) — in that case, a brief, honest loading state communicates the real uncertainty better than an optimistic update that will need to be walked back at a non-trivial rate.

The trap: treating "requires confirmation" and "should be optimistic" as the same axis — confirmation is about gating irreversible/consequential *intent*, optimism is about UI responsiveness during a *typically-fast, typically-successful* network round trip; conflating them leads to either skipping useful confirmation or skipping useful optimism unnecessarily.

---

**Q (Medium): How would this change if there were no React Query (or similar) and you were implementing optimistic updates with plain `useState` and `fetch`?**

Answer: The same three-step pattern applies, just implemented by hand: capture the previous value into a local variable before calling `setState` optimistically, then in the `catch` block of the `fetch` call, `setState` back to that captured previous value, and in a `finally` (or the success branch) reconcile with the server's actual response if it can differ from the optimistic guess. The meaningful things a library like React Query buys that are genuinely non-trivial to replicate by hand: automatic request deduplication/cancellation for in-flight queries against the same key, consistent behavior when multiple components read the same piece of "server state" (a hand-rolled version scoped to one component's local state doesn't naturally share that state with a sibling component showing the same data elsewhere on the page), and built-in retry/staleness semantics. For a single, isolated, one-off mutation with no sharing concerns, hand-rolling is entirely reasonable; the moment the same data needs consistent optimistic behavior in more than one place, a shared cache-based approach earns its complexity.

The trap: presenting the hand-rolled version as strictly worse in every case — for a genuinely simple, single-consumer optimistic update, plain `useState` plus a `try/catch` is a perfectly reasonable, lower-dependency solution, and reaching for a full query library for one component's one button is its own kind of over-engineering.

---

**Q (Medium): Should the "like" button's optimistic update be debounced or otherwise protected against a user rapidly toggling like/unlike several times in a row?**

Answer: I'd disable (or visually lock) the toggle while its own mutation is in flight rather than debouncing the click itself, because debouncing would delay the very responsiveness optimism is meant to provide — locking the control for the (typically very short) duration of one in-flight request avoids firing overlapping mutations for the same toggle while still feeling instant for the common case of a single click. If rapid toggling is a realistic pattern (someone excitedly double/triple-tapping), a short client-side lock (re-enable once the current mutation settles) handles it without needing to queue or coalesce multiple mutations — coalescing (e.g., collapsing five rapid toggles into a single final-state mutation) is a reasonable enhancement if it's observed to matter in practice, but isn't a correctness requirement for a basic implementation.

The trap: reaching for a debounce (delaying the optimistic UI update itself) when the actual risk is overlapping *mutations*, not the UI feedback — a disabled-while-pending control solves the real risk without sacrificing the instant-feedback goal optimism exists for.

---

**Q (Low): Does server-sent authoritative reconciliation (the `onSuccess` overwrite with real server data) risk a visible "flicker" if the optimistic guess and the real value differ — e.g., optimistic `likeCount: 43`, server responds with the real `likeCount: 45` because two other people liked it in the meantime?**

Answer: Yes, technically the displayed count can jump from the optimistic 43 to the authoritative 45 once the response lands, and that's the correct, honest behavior rather than something to suppress — the optimistic value was always a guess for the brief gap before real data arrives, and papering over the correction (e.g., refusing to apply the authoritative value, or animating around it to hide the jump) would leave the UI showing a value that's known to be wrong. For a rapidly-changing high-traffic count, a small, non-jarring transition (a brief count-up animation to the new value rather than an instant snap) is a reasonable polish detail, but it should still land on the real value, not stay pinned to the optimistic guess.

The trap: treating a visible correction as a bug to eliminate rather than a natural, expected consequence of optimism — the fix for a jarring correction is a smoother transition, not suppressing the correction itself.

---

## Self-Assessment

- [ ] Can state the snapshot → apply → confirm-or-rollback pattern unprompted and explain why snapshotting beats algebraic undo
- [ ] Can explain what `cancelQueries` in `onMutate` prevents, concretely
- [ ] Can distinguish a plain mutation failure (rollback to previous) from a conflict response (refetch real current state instead)
- [ ] Can articulate why failure-communication UX should scale with an action's real-world stakes, with a concrete example of each
- [ ] Can explain why disabling the triggering control during an in-flight mutation is preferable to debouncing the click

---
*Next: Undo/Redo — Architecture Decision — shifts from "recovering from a failed optimistic update" to deliberately supporting user-initiated undo of successful actions, which needs a fundamentally different state-history mechanism.*
