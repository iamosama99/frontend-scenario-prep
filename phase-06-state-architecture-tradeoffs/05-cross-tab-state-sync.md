# Cross-tab State Sync

## Quick Reference

| Mechanism | What It Syncs | Latency | Notes |
|---|---|---|---|
| `BroadcastChannel` | Any serializable message, tab-to-tab | Near-instant | Same-origin only; not supported in very old browsers (rarely a real constraint now) |
| `storage` event (via `localStorage`) | Key/value changes | Near-instant | Fires only in *other* tabs, not the tab that wrote the value; wider browser support than BroadcastChannel |
| Server as source of truth + refetch/websocket push | Anything server-persisted | Depends on transport (poll interval, or instant via websocket) | Works across devices too, not just tabs — the more general solution |
| `SharedWorker` | Shared in-memory state, coordinated logic | Near-instant | Heavier to set up; useful when tabs need to coordinate more than just "data changed," e.g. leader election |

## The Scenario

"A user has the same app open in two browser tabs. They log out in tab 1 — tab 2 should immediately reflect that they're logged out too, not keep showing a stale authenticated view. Separately, they mark a notification as read in tab 1, and tab 2's notification badge count should update without a manual refresh. Design how state stays in sync across tabs for both cases."

## Clarifying Questions

- **Is the underlying data (auth session, notification read-state) already persisted server-side, or is any of it purely client-side/local?** If everything is server-persisted, the "real" source of truth for cross-tab consistency can just be the server, with tabs syncing to it — cross-tab sync mechanisms (BroadcastChannel, storage events) become an optimization for *faster* propagation, not the only mechanism, which changes the failure-mode analysis (a missed cross-tab message is recoverable via a normal refetch, not catastrophic).
- **For logout specifically — is "immediately" a hard requirement (must happen within the same second, since a stale-authenticated tab showing sensitive data after logout is a real problem), or is "eventually consistent within a page interaction" acceptable?** Logout has a security dimension beyond convenience — a tab that stays authenticated and keeps showing/allowing access to protected data after the user explicitly logged out elsewhere is a more serious issue than a notification badge being briefly stale, and that should shape how aggressively I push for instant (BroadcastChannel-driven) propagation versus relying on the next natural API call's 401 to catch it.
- **Does the notification read-state need to sync in the other direction too — if tab 2 marks something read, does tab 1's count also need to update?** I'd assume yes (symmetric sync, not just "tab 1 is the source of truth") unless told otherwise, since nothing in the scenario suggests an asymmetric relationship between the tabs.
- **Are there more than two tabs realistically in play (a power user with many tabs open), and does the mechanism need to scale to N tabs, or is two a reasonable proxy for testing/design purposes?** `BroadcastChannel` and `storage` events both naturally broadcast to *all* other same-origin tabs, not just a specific one, so this doesn't actually change the mechanism — but it's worth confirming there's no hidden requirement like "only sync between tabs the user explicitly grouped," which would need something more targeted.
- **What should happen to any in-progress unsaved work in tab 2 (e.g., a half-filled form) at the moment tab 1 triggers a logout?** This affects whether the cross-tab logout handler in tab 2 should attempt any "are you sure, you have unsaved changes" gate, or unconditionally force a logout/redirect — I'd lean toward unconditional for logout specifically (security should win over preserving unsaved form data), but it's worth surfacing as a deliberate trade-off rather than an accidental side effect.

## Approach & Trade-offs

**Logout and notification-read-state are both fundamentally "server state that changed, and other tabs need to know" — but they warrant different urgency and different mechanisms, which is the core design decision here.** Logout has security implications severe enough to justify an active, near-instant push mechanism (`BroadcastChannel`) specifically so a second tab doesn't sit in a stale-authenticated state showing protected content for any longer than necessary; a notification badge being a few seconds stale has essentially no consequence, so a simpler mechanism (or even just "the next time this tab makes any API call, if the server includes updated notification state or the client's own next poll picks it up" ) is perfectly adequate, and reaching for the heavier active-push mechanism there is optional polish, not a correctness requirement.

**`BroadcastChannel` is the right primary tool for both, over the `storage` event trick, because it's a purpose-built pub/sub API for exactly this problem rather than a repurposed side effect of `localStorage`.** The `storage` event approach (writing a value to `localStorage` purely to trigger the event in other tabs, often not even caring about the value's persistence) works and has historically been used specifically because `BroadcastChannel` had spottier support — but it comes with real gotchas (it doesn't fire in the tab that made the write, requires writing to actual persisted storage as a side channel even when persistence isn't otherwise needed, and needs careful cleanup of the storage key). `BroadcastChannel` is more direct: create a channel with a name, `postMessage` to it, listen with `onmessage` — no unrelated persisted storage side effect required, and support is broad enough now that it's not a meaningful constraint for a typical web app.

**Regardless of which push mechanism is used, the server must remain the actual source of truth, and the cross-tab message should be treated as a *hint to refetch/react*, not as the update itself** — because relying solely on a same-origin broadcast message as the sole mechanism for consistency has real gaps: a tab opened *after* the broadcast fires won't have received it (though it would get correct state on its own initial load from the server anyway), and if the browser or OS ever fails to deliver the message (rare, but not something to build a security boundary on), a tab could be left in a stale state with no correction mechanism. For logout specifically, defense in depth matters: even without the broadcast, the *next* authenticated API call tab 2 makes should get a 401 and trigger its own logout handling — the broadcast is what makes that happen proactively and near-instantly instead of only reactively on the next request.

**For the logout case, the broadcast payload can be minimal — just "logout happened" — since the receiving tab doesn't need any additional data to act (redirect to login, clear local auth state); for the notification-read case, I'd lean toward the broadcast being "hey, notification state changed, go refetch the count/list" rather than trying to push the full new state through the message itself,** because pushing the full state through the broadcast (e.g., "notification 123 is now read, new count is 4") duplicates logic that the normal fetch/query path already has to have anyway (computing derived counts, handling the full notification list shape) — simpler and more consistent to have the broadcast just invalidate the relevant query in the receiving tab and let the existing data-fetching layer do what it already knows how to do.

## Solution

**1. A small `BroadcastChannel` wrapper, reused for both features:**

```tsx
const authChannel = new BroadcastChannel('auth');
const notificationsChannel = new BroadcastChannel('notifications');
```

**2. Logout — broadcasting, and reacting in other tabs:**

```tsx
// Tab where logout is triggered
async function logout() {
  await api.logout(); // server-side session invalidation
  clearLocalAuthState();
  authChannel.postMessage({ type: 'logout' });
  navigate('/login');
}

// Every tab, set up once at app initialization
authChannel.onmessage = (event) => {
  if (event.data.type === 'logout') {
    clearLocalAuthState(); // e.g. clear an in-memory auth store, cached user object
    navigate('/login'); // force this tab to the login screen regardless of what it was showing
  }
};
```

**3. Defense in depth — a tab that somehow missed the broadcast still self-corrects on its next authenticated request:**

```tsx
// A shared API client interceptor
apiClient.onResponseError((error) => {
  if (error.status === 401) {
    clearLocalAuthState();
    navigate('/login'); // catches the case where this tab never received the broadcast at all
  }
});
```

**4. Notification read-state — broadcast is a lightweight "go refetch" hint, not the data itself:**

```tsx
// Tab where a notification is marked read
async function markNotificationRead(id: string) {
  await api.markNotificationRead(id); // server is source of truth
  queryClient.invalidateQueries({ queryKey: ['notifications'] }); // update this tab
  notificationsChannel.postMessage({ type: 'invalidate' }); // tell other tabs to do the same
}

// Every tab
notificationsChannel.onmessage = (event) => {
  if (event.data.type === 'invalidate') {
    queryClient.invalidateQueries({ queryKey: ['notifications'] }); // reuses the exact same fetch path as a normal update
  }
};
```

> **Check yourself:** Why does the notification broadcast intentionally avoid sending the actual updated count or notification data, and instead just trigger a refetch — what would be more fragile about sending the data directly?

## Gotchas

**Relying solely on a cross-tab broadcast for logout security, with no server-side check on subsequent requests.** If the message is somehow missed (a tab that was asleep/throttled by the browser, a bug in the listener setup), a tab could remain in an authenticated-looking state indefinitely with nothing to correct it — the 401-triggers-logout interceptor is not optional polish here, it's the actual security backstop.

**Using the `storage` event and forgetting it doesn't fire in the same tab that made the write.** Testing exclusively in a single tab (or writing test code that expects the writing tab's own listener to fire) will "work" during casual dev testing if the developer isn't specifically checking a second tab, and then silently not behave as expected for the actual cross-tab case it exists for.

**Broadcasting full data payloads (the entire updated notification list) instead of an "invalidate and refetch" signal.** Couples the broadcast message's shape to the data's shape, meaning any future change to the notification data structure needs the broadcast payload updated in lockstep too — versus a generic "go refetch key X" signal, which stays correct regardless of how the underlying data shape evolves.

**Not handling the case where `BroadcastChannel` (or the browser tab itself) is unavailable/unsupported in some environment the app needs to support** (rare today, but worth a conscious check, not an assumption) — falling back to the `storage` event or simply accepting "cross-tab sync is best-effort, the server-truth-on-next-request path is the real guarantee" for that environment.

**Forgetting to close/clean up the `BroadcastChannel` instance if it's created per-component rather than once at app scope**, leaking listeners across re-renders or remounts — a shared, module-scoped channel (as shown above) avoids this entirely, since it's created once for the app's lifetime rather than per-mount.

## Follow-up Questions

**Q (High): Why should the server remain the actual source of truth even when using an active cross-tab push mechanism like `BroadcastChannel` — isn't the whole point of the broadcast to avoid needing a refetch?**

Answer: The broadcast's job is to trigger *timely* reaction, not to *replace* the authoritative data path — treating the broadcast payload itself as the update means duplicating whatever logic already computes derived state (a notification count, a merged view of read/unread) in two places (the normal fetch path and the broadcast handler), and those two implementations can drift or handle edge cases differently. It also means a tab that missed the broadcast (opened after it fired, or missed delivery for any reason) has no path to correct itself except its own next normal fetch — which is exactly the mechanism I'd want it to fall back on anyway, so building the broadcast as "a hint to refetch" rather than "the update itself" means there's only one source of truth and one code path computing derived state, with the broadcast purely affecting *when* that path runs, not *what* it computes.

The trap: treating the broadcast as a mini data-sync protocol in its own right, duplicating fetch/derivation logic into the message handler, which doubles the surface area for bugs and inconsistency between "state updated via broadcast" and "state updated via normal fetch."

---

**Q (High): For the logout case, why is a `401`-triggers-logout interceptor necessary if `BroadcastChannel` already handles cross-tab logout — what specific scenario does it catch that the broadcast doesn't?**

Answer: It catches any case where the broadcast message doesn't reach a given tab — a tab that was backgrounded/frozen by the browser's tab-throttling behavior at the moment the message was sent and doesn't process events until it's foregrounded again, a tab opened in a separate browser process or context where `BroadcastChannel`'s same-origin guarantee still technically applies but some edge-case timing means the listener wasn't registered yet, or simply defensive engineering against any bug in the broadcast wiring itself. Since logout has real security consequences (a stale-authenticated tab could let someone continue viewing or acting on protected data after the user explicitly signed out), I wouldn't want the *only* mechanism enforcing that boundary to be a same-origin messaging API with no independent server-side backstop — the interceptor ensures that even in the worst case, the very next authenticated request that tab makes gets rejected and forces re-authentication, bounding how long a stale session can persist to "until the next API call" rather than "indefinitely, if the broadcast failed."

The trap: treating the broadcast mechanism as sufficient on its own for a security-relevant state change — client-side messaging between tabs is a UX/latency optimization, not a substitute for the server actually enforcing session invalidation on every subsequent request.

---

**Q (High): A user has three tabs open. Tab 1 logs out. Tab 2 receives the broadcast and redirects to login. Tab 3 is in the middle of submitting a large form when the broadcast arrives. What should happen in tab 3, and does your design account for it?**

Answer: For a security-sensitive action like logout, I'd still force tab 3 to react (clear auth state, redirect) rather than let it keep running as if authenticated — but the in-flight form submission's own request will likely now fail server-side (its session is invalidated), and that failure needs to be handled gracefully rather than silently swallowed: the 401 interceptor described above should surface a clear message ("You were logged out — this action wasn't completed, please log back in and retry") rather than just redirecting with no explanation, since from tab 3's user's perspective, work they were actively doing just vanished for a reason that isn't obvious unless it's tab 1 they logged out from. This is a case where the "unconditional force logout, security wins" default from the clarifying questions has a real UX cost, and calling that trade-off out explicitly (rather than pretending it doesn't exist) is the stronger answer — some products might choose to let an in-flight submission complete before forcing the redirect, accepting a brief window of technically-stale-but-about-to-be-corrected state, as a deliberate UX concession; I'd treat that as a product decision to surface, not something to silently decide either way.

The trap: only reasoning about the simple two-tab, no-in-flight-work case — a strong answer proactively considers what happens to in-progress work in a third tab and treats the security-vs-UX tension as worth surfacing explicitly rather than picking a side without acknowledging the trade-off.

---

**Q (Medium): Would `localStorage` plus the `storage` event be a reasonable alternative to `BroadcastChannel` here — what would you lose or gain?**

Answer: It would work — write a throwaway value (e.g., a timestamp) to a `localStorage` key on logout, and listen for the `storage` event in other tabs to trigger the same reaction — and it has one genuine advantage: broader historical browser support. The costs: it requires an actual (even if semantically meaningless) write to persisted storage purely as a signaling side channel, it doesn't fire in the tab that performed the write (easy to forget and a common source of "works when I test tab 2, but I assumed tab 1 also gets notified" confusion), and the payload is limited to what can be usefully round-tripped through a string value in storage, versus `BroadcastChannel`'s ability to `postMessage` structured data more naturally. For a modern web app with no unusual browser-support constraints, I'd default to `BroadcastChannel` for being purpose-built and more direct; I'd only reach for the `storage` event approach if there were a specific, confirmed need to support a browser old enough to lack `BroadcastChannel`, which is increasingly rare.

The trap: assuming `BroadcastChannel` isn't "safe" to use without checking real support data — it's been broadly supported in evergreen browsers for years now, and defaulting to the more roundabout `storage` event trick out of outdated caution adds real gotchas (no same-tab firing, storage-side-effect noise) for a support concern that usually doesn't actually apply.

---

**Q (Medium): Does this cross-tab sync design need any changes to also work across multiple *devices* (not just tabs on the same device), e.g. a user logged in on both a laptop and a phone?**

Answer: `BroadcastChannel` and the `storage` event are both strictly same-origin, same-browser, same-device mechanisms — they have no reach across devices at all, so cross-device consistency has to come from a genuinely different mechanism: a server-pushed channel (websocket/SSE) the client subscribes to, or, at minimum, relying on the same 401-triggers-logout backstop the next time the other device makes any authenticated request (which is exactly what already happens today for any session-invalidation system without active push — it's just less "immediate" than the same-device cross-tab case). If true near-instant cross-device logout propagation is a real requirement (e.g., a security-conscious product wanting "logged out everywhere, immediately, visibly"), that pushes toward a server-side push mechanism the client subscribes to per authenticated session, which is a materially bigger piece of infrastructure than anything discussed for the same-device cross-tab case — worth naming as a distinct, larger scope rather than assuming the tab-sync mechanism extends to it for free.

The trap: conflating "cross-tab" and "cross-device" as the same problem with the same solution — they look similar from a UX-requirement standpoint ("logging out anywhere should log out everywhere") but require entirely different mechanisms, and `BroadcastChannel` contributes nothing to the cross-device half of that requirement.

---

**Q (Low): If a user has 20 tabs open, does broadcasting to all of them for every notification-read event create any meaningful performance concern?**

Answer: Not meaningfully at this scale — `BroadcastChannel` messages are cheap, and 20 tabs each doing a lightweight query invalidation (which itself is deduped/coalesced by a query library if multiple invalidations fire in quick succession) is well within normal browser capability; this isn't a scenario that needs throttling or batching for a typical user's realistic tab count. Where it *could* start to matter is a pathological case (hundreds of tabs, or a broadcast firing at high frequency, like on every keystroke of some other feature) — but that's a different scale of problem than "mark one notification read," and I wouldn't add complexity (debouncing broadcasts, batching messages) to guard against a case this feature doesn't actually produce.

The trap: preemptively over-engineering broadcast-message batching/throttling for a message frequency and tab count that doesn't remotely approach a real performance concern — matching effort to actual, demonstrated need rather than a hypothetical worst case that isn't this feature's shape.

---

## Self-Assessment

- [ ] Can explain why logout and notification-read-state warrant different urgency/mechanisms despite both being "cross-tab sync" problems
- [ ] Can justify `BroadcastChannel` over the `storage`-event trick, and name the `storage` event's specific footguns (no same-tab firing, storage side-effect)
- [ ] Can explain why the broadcast should trigger a refetch/invalidation rather than carry the actual updated data
- [ ] Can explain why a 401-triggers-logout interceptor is a necessary backstop, not redundant, alongside the broadcast
- [ ] Can distinguish cross-tab sync (same device, same browser) from cross-device sync and name what mechanism the latter actually requires

---
*Next: Conflict Resolution for Concurrent Edits — moves from "the same user, multiple tabs" to "multiple different users editing the same resource concurrently," where the resolution strategy has to handle genuinely divergent, not just duplicated, intent.*
