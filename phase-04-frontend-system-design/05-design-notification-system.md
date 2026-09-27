# Design a Notification System (In-app + Push)

## Quick Reference

| Decision Point | Choice | Why |
|---|---|---|
| Source of truth | One central notification store (client-side cache of server records), every UI surface derives from it | A bell-icon badge count, a toast, and a notification-center list are all *views* of the same underlying event stream — building each independently drifts them out of sync and risks duplicate rendering of the same event |
| In-app real-time delivery | WebSocket/SSE push updates the central store; the store drives whatever UI is currently mounted | Decouples "a new notification arrived" from "how it's currently displayed" — the same arrival can simultaneously increment a badge, spawn a toast, and prepend a list item, from one event handled once |
| Background/app-closed delivery | Browser Push API + a Service Worker, entirely separate code path from in-app delivery | Push notifications must work even when no tab is open — this requires a Service Worker registered ahead of time and a subscription the server can target independently of any live in-app connection |
| Cross-channel/cross-tab dedup | Every notification carries a stable server-assigned ID; every channel (in-app store, Service Worker, other open tabs) treats "have I already shown/recorded this ID" as the dedup gate | The same underlying event can plausibly arrive via both the live in-app socket and a push event if multiple tabs/paths are open — without ID-based dedup, the same notification can visibly double up |
| Read/unread state | Synced through the server as authoritative, broadcast to all open tabs (`BroadcastChannel` or refetch-on-focus) | Marking something read in one tab and seeing it still unread in another tab of the same session is a well-known, avoidable consistency bug |

## The Scenario

"Design the notification system for our product — in-app notifications (a bell icon with a badge count and a dropdown list), toast pop-ups for real-time events while the user is active, and browser push notifications that should arrive even if the app isn't open in any tab. All of these need to reflect the same underlying set of events without duplicating or contradicting each other. Walk me through the architecture."

## Clarifying Questions

- **What actually generates a notification — is it purely server-driven (some backend event: someone liked your post, a message arrived), or can purely client-local events also produce one (a client-side validation, an offline-sync conflict)?** This decides whether the system needs to reconcile *client-originated* and *server-originated* notifications into the same store, or whether the server is the sole source of truth for every notification's existence, which is the simpler and more common case worth confirming rather than assuming.
- **Do notifications need to work when no browser tab is open at all** — i.e., is browser push (via a Service Worker and the Push API) actually in scope, or is "in-app only, visible next time they open the app" sufficient? Push notifications require meaningfully more infrastructure (Service Worker registration, permission prompts, a subscription endpoint, VAPID/push-service integration) — worth establishing this isn't assumed into scope if it isn't actually required.
- **Should different notification categories be independently configurable — e.g., push for direct messages but not for "someone liked your post," or a do-not-disturb window?** This affects whether the store/delivery pipeline needs a per-category, per-channel preference check gating each delivery decision, versus a single on/off toggle for notifications as a whole.
- **When many similar events happen in a short window (10 people like the same post within a minute), should each produce its own separate notification, or should they collapse/group into one ("10 people liked your post")?** Ungrouped, high-frequency notifications from a popular piece of content can flood a user's notification center and badge count in a way that reads as broken rather than accurate — grouping is a real design decision, not just a nice-to-have.
- **Does read/unread state need to stay consistent across multiple simultaneously open tabs of the same account, and across multiple devices?** Determines whether a same-tab-only in-memory store is sufficient or whether cross-tab (`BroadcastChannel`/storage events) and cross-device (server-authoritative, refetched or pushed) synchronization needs to be designed in from the start.
- **What's the expected notification volume and retention — do users need to browse a long history of past notifications, or only ever see a short recent window before it's expected to be cleared/expired?** Affects whether the notification-center list needs its own pagination/virtualization treatment, similar to any other long list, or can reasonably hold everything in memory.

## Approach & Trade-offs

**One central, normalized notification store is the architectural anchor — every surface (badge, toast, dropdown list) reads from it rather than each independently listening for "new notification" events and rendering its own reaction.** The naive alternative — a badge-count component that increments on its own socket listener, a toast-spawner that separately listens for the same event and shows a popup, a notification-list component that separately fetches/appends — technically works until any of those three drift out of sync (a toast fires for an event the badge count didn't increment for, because one listener fired and the other didn't due to some timing/mounting difference). Centralizing means exactly one place receives and records each notification event (assigning/recognizing its stable ID, applying dedup, updating read/unread state), and every visual surface is a pure function of that one store's current state — a badge is `store.unreadCount`, a toast is a side effect subscribed to "new item added to the store," a dropdown list is `store.items` rendered directly. This is the same normalization principle used for the News Feed's post cache, applied to notifications instead of posts.

**In-app real-time delivery and background push delivery are two genuinely separate code paths that both feed the same store, not one mechanism serving both cases.** A live WebSocket/SSE connection only exists while a tab is open and connected — it cannot deliver anything to a user with zero open tabs, which is precisely the case browser push exists to cover. Push requires its own separate infrastructure: a Service Worker registered ahead of time (independent of whether any app tab is currently open), a push subscription (created via the browser's Push API and sent to the backend so it knows *where* to push to for this specific browser/device), and a `push` event handler inside the Service Worker that displays a native OS notification even with no tab open. When the user does open/return to a tab, the in-app store needs to reconcile with whatever arrived via push while it wasn't running — typically by fetching "notifications since my last-seen ID" on load/focus, deduped against anything already in the store by the same stable ID used everywhere else.

**Grouping/collapsing near-duplicate notifications is a deliberate aggregation step in the store, not a per-notification rendering trick.** Rather than rendering ten separate "X liked your post" list items when ten likes arrive in a short window, the store's ingestion logic recognizes a grouping key (e.g., `{ type: 'like', targetPostId }`) and, when a new notification shares that key with an existing recent, still-unread entry, updates the existing entry's aggregate count/actor list instead of appending a new one ("Alex and 9 others liked your post"). This needs a clear, deliberate rule for *when* grouping applies (recent, same type, same target, still unread) versus when it shouldn't (a comment and a like on the same post are different events and shouldn't merge into one, even though they share a target) — worked out with the same care as any other data-normalization decision, not left as an emergent side effect of whatever the rendering code happens to do.

**Read/unread state is server-authoritative, with cross-tab consistency handled explicitly, because "read" is meaningful account-wide state, not a per-tab UI flag.** Marking a notification read (opening the dropdown, clicking a specific item) should update the server immediately (or optimistically-then-confirmed, same pattern as other optimistic writes in this repo), and every other currently-open tab of the same account needs to reflect that same read state without requiring a manual refresh — achieved via a `BroadcastChannel` message to sibling tabs on the same origin (near-instant, no server round trip needed for same-browser tabs) and, for genuinely separate devices/sessions, either a live push of the read-state change over the existing socket connection or a refetch-on-focus/interval reconciliation. Skipping this produces the specific, recognizable bug of "I read this notification on my phone, it's still showing unread on my laptop" — a correctness issue, not a cosmetic one.

**Permission for browser push should be requested contextually, not immediately on page load.** Browsers show a native permission prompt for notifications that, if triggered the instant a page loads (before the user has any context for why the app wants this), is overwhelmingly likely to be dismissed or denied — once denied, most browsers make it hard-to-impossible for the app to re-prompt without the user manually changing browser settings, effectively burning the one chance permanently. The better pattern prompts at a moment tied to explicit user intent (e.g., right after the user enables "notify me" on a specific feature, or after they've taken some action signaling engagement), where the ask has clear, immediate context.

## Solution

**Central notification store, with dedup and grouping applied at ingestion:**

```ts
interface Notification {
  id: string;
  type: 'like' | 'comment' | 'message' | 'system';
  groupKey: string; // e.g., `like:${postId}` — shared by notifications eligible to merge
  actorIds: string[]; // one or more actors, for grouped notifications
  targetId: string;
  createdAt: string;
  read: boolean;
}

function ingestNotification(store: NotificationStore, incoming: Notification) {
  if (store.byId.has(incoming.id)) return; // already recorded — dedup by stable ID, regardless of channel

  const existingGroup = store.recentUnreadByGroupKey.get(incoming.groupKey);
  if (existingGroup && !existingGroup.read) {
    // Merge into the existing, still-unread notification rather than appending a new item.
    existingGroup.actorIds = [...new Set([...existingGroup.actorIds, ...incoming.actorIds])];
    existingGroup.createdAt = incoming.createdAt;
    store.byId.set(incoming.id, existingGroup); // this ID also resolves to the merged record
    return;
  }

  store.byId.set(incoming.id, incoming);
  store.order.unshift(incoming.id);
  store.recentUnreadByGroupKey.set(incoming.groupKey, incoming);
}
```

**Service Worker registration and push subscription (app shell, run once):**

```ts
async function setupPushNotifications() {
  const registration = await navigator.serviceWorker.register('/sw.js');

  // Ask for permission at a contextual moment — this function is called
  // from a user-initiated "enable notifications" action, not on page load.
  const permission = await Notification.requestPermission();
  if (permission !== 'granted') return;

  const subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey: VAPID_PUBLIC_KEY,
  });

  await sendSubscriptionToServer(subscription); // server now knows where to push for this browser
}
```

**Service Worker's push handler (runs even with no tab open):**

```js
// sw.js
self.addEventListener('push', (event) => {
  const data = event.data.json(); // { id, type, title, body, targetUrl }
  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      data: { id: data.id, targetUrl: data.targetUrl },
      tag: data.groupKey, // same-tag notifications replace each other natively, a coarse grouping fallback
    })
  );
});

self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  event.waitUntil(clients.openWindow(event.notification.data.targetUrl));
});
```

**Reconciling on app focus/load — merging whatever arrived via push while no tab was tracking the live store:**

```ts
async function reconcileOnFocus(store: NotificationStore) {
  const since = store.lastKnownId; // stable cursor, same principle as feed pagination
  const missed = await fetchNotificationsSince(since);
  missed.forEach((n) => ingestNotification(store, n)); // same dedup/group path as live + push delivery
}
```

**Cross-tab read-state sync:**

```ts
const channel = new BroadcastChannel('notifications');

function markRead(store: NotificationStore, id: string) {
  store.byId.get(id)!.read = true;
  reportReadToServer(id); // fire-and-forget-ish, but should retry on failure
  channel.postMessage({ type: 'read', id });
}

channel.onmessage = (event) => {
  if (event.data.type === 'read') {
    const n = store.byId.get(event.data.id);
    if (n) n.read = true; // reflect the other tab's action here too
  }
};
```

> **Check yourself:** Without looking above, explain why in-app (WebSocket/SSE) delivery and background push delivery need to be built as two separate mechanisms rather than one, and what specifically fails if only the in-app socket path is built.

## Delivery Channel Matrix

| Channel | Works With No Tab Open? | Mechanism | Typical Use |
|---|---|---|---|
| In-app toast | No — only while a tab is open and mounted | Store subscription → ephemeral UI, auto-dismissing | Immediate feedback for events relevant to what the user's currently doing |
| Notification-center dropdown | No (but reflects history once opened) | Store's persisted list, fetched/paginated | Browsable history of everything, read/unread state |
| Badge count | No (badge itself; can pair with OS-level Badging API for a PWA icon badge) | Derived count from the store | At-a-glance "there's something new" signal |
| Browser push | **Yes** | Service Worker + Push API, independent subscription | Re-engagement when the app isn't open at all |

## Gotchas

**Building three independent listeners (badge, toast, list) for the same underlying socket event instead of one central store all three read from.** Works in a demo, drifts in production the moment any one listener's mount timing, error handling, or reconnection state differs even slightly from the others — a single ingestion point with derived views is what keeps them provably consistent.

**No dedup by stable ID across in-app and push delivery paths.** If both a live socket message and a Service Worker push event can represent the same underlying notification (plausible if the server fires both unconditionally), and neither delivery path checks against a shared "already have this ID" gate, the same notification can visibly show up twice — once as a toast from the socket, once as a native OS notification from push, for what the user experiences as one event.

**Requesting push permission immediately on first load.** Near-guaranteed to be dismissed or denied without context, and a denial is difficult to reverse without the user manually digging into browser site settings — this is a one-shot ask that should be spent deliberately, tied to a moment of clear user intent.

**Grouping logic that merges unrelated event types sharing a target (a like and a comment on the same post) into one confusing aggregate.** Grouping needs a precise key (type + target, not target alone), or it produces technically-deduplicated but semantically-wrong summaries like "5 people interacted with your post" that obscure what actually happened.

**No cross-tab read-state sync.** A user reading and dismissing a notification in one tab while another tab of the same account still shows it unread is a specific, reportable bug — solved cheaply with `BroadcastChannel` for same-origin tabs, or server-driven sync for cross-device cases.

**Treating the notification-center list as unbounded, unpaginated state.** A long-lived, active account can accumulate thousands of historical notifications over time — the same pagination/virtualization discipline used for any other long list applies here, not an exemption just because it's "just notifications."

## Follow-up Questions

**Q (High): Why can't a WebSocket-based in-app notification system alone satisfy a "notify me even when I don't have the app open" requirement, no matter how well the reconnection logic is built?**

Answer: A WebSocket connection is inherently tied to an open, running page/tab — there is no "open connection" to push through once every tab for that user is closed, no matter how robust the reconnection/backoff logic for the case where a tab *is* open but momentarily disconnected. Browser push notifications solve a structurally different problem: the browser itself (via its underlying push service, e.g., FCM/APNs-backed infrastructure depending on platform) maintains a channel to the *browser*, independent of any specific page or tab being open, and a Service Worker (which can be woken up by the browser to handle a push event even with zero tabs open) is what actually displays the notification. These are different mechanisms addressing different lifecycle states (app open vs. app fully closed), and no amount of improving the WebSocket reconnection strategy changes which lifecycle state it's fundamentally scoped to.

The trap: proposing "just reconnect more aggressively" or "keep a background tab alive" as a substitute for real push infrastructure — browsers actively prevent pages from staying meaningfully alive in the background indefinitely (tab throttling/discarding), and there is no reliable, sanctioned way to receive events with literally no tab open other than the Push API + Service Worker mechanism built for exactly this.

---

**Q (High): The interviewer asks: "A user has the app open in two tabs and also has push notifications enabled. An event fires. Walk through everything that could show them the same notification twice, and how the design prevents it."**

Answer: Several paths could independently deliver the same underlying event: the live socket delivering it to *each* open tab independently (both tabs could show their own toast if not coordinated), and the Service Worker's push handler potentially also firing and showing a native OS notification for the same event if the server sends both a socket message and a push message unconditionally. The design prevents visible duplication at two levels: first, each tab's own store dedups by the notification's stable ID before rendering anything (so even if both tabs receive the same socket message, each tab, in isolation, shows it once — no double-render *within* a tab); second, cross-channel duplication (a toast in a tab plus a native push notification for the same event) is best avoided by having the server or client suppress the push send specifically when it can determine the user already has an active, connected tab (a common pattern: only send push for a given event if no client currently holds a live socket connection for that account, since an actively-connected client will get it live and doesn't need a native OS notification too). If push is sent regardless, the fallback is still the shared stable-ID dedup — a native OS notification the Service Worker shows can carry the same ID, and if the app later reconciles that ID into the same store already holding it from the live path, at minimum the *in-app* history doesn't show a duplicate entry, even if the user did momentarily see both a toast and a native popup for the same event.

The trap: only solving the within-tab dedup and treating cross-channel (socket vs. push) duplication as unavoidable — it's mitigated by having the sending side avoid pushing to sessions it knows are already actively connected, which is worth naming as part of a complete answer rather than assuming the client side alone must absorb every possible duplication.

---

**Q (High): How would you design notification grouping so that "10 likes in a minute" merges into one item, but a like and a comment on the same post correctly remain two separate notifications?**

Answer: The grouping key needs to encode both the event *type* and its target, not the target alone — e.g., `like:{postId}` and `comment:{postId}` are deliberately distinct group keys even though they share a `postId`, so the ingestion logic's "does an existing unread notification share this group key" check only merges genuinely-the-same-kind-of-event notifications, never conflates a like with a comment just because they touch the same post. Within one group key, incoming notifications merge into the existing unread entry (accumulating actor names/count, refreshing the timestamp) rather than each appending a new list item — the merge window is typically also bounded to "still unread" (once a user reads/dismisses the aggregated notification, a fresh like arriving afterward starts a new aggregate rather than reopening the old one).

The trap: using a coarser group key (target alone, e.g., just `postId`) for simplicity — this looks correct in a quick like-only demo and then produces confusing merged notifications ("5 people interacted with your post") the moment a comment and a like on the same post are supposed to remain independently visible and identifiable events.

---

**Q (Medium): A user marks a notification as read in one tab. Design exactly how a second open tab of the same account reflects this without a manual refresh, and how this differs for a second *device* (a phone) rather than a second tab of the same browser.**

Answer: For a second tab of the same browser/origin, `BroadcastChannel` (or, as a slightly older-browser-compatible fallback, a `storage` event on a shared `localStorage` key) is a near-instant, no-server-round-trip mechanism — one tab posts `{ type: 'read', id }`, every other same-origin tab's listener updates its own in-memory store immediately. This doesn't reach a different device at all (`BroadcastChannel` is scoped to same-browser, same-origin contexts) — a second device needs the read-state change to actually reach the server (which it should, as the authoritative record regardless of tab-sync) and then either be pushed to the other device's live connection if one is open, or picked up the next time that device's app reconciles ("fetch notifications/read-state since my last sync") on focus/reconnect/poll. The server round trip is unavoidable for cross-device sync; `BroadcastChannel` is purely a same-browser optimization to avoid needing that round trip for the common "I have two tabs open" case.

The trap: assuming `BroadcastChannel` alone is a complete cross-device sync solution — it's scoped strictly to the same browser context; conflating it with true cross-device sync (which must go through the server) misses that these are two different problems with two different mechanisms, one of which is optional-but-nice and one of which is required for correctness at all.

---

**Q (Medium): Design the moment/UX for requesting browser push permission such that it isn't wasted on an early, context-free denial.**

Answer: Defer the permission request until a moment tied to explicit, in-context user intent — e.g., a visible "Get notified when someone replies" toggle/button on a relevant feature, which the user actively engages with, at which point the actual `Notification.requestPermission()` call fires as a direct response to that action. This gives the browser's native prompt clear surrounding context (the user just asked for this) rather than appearing unprompted on first page load, where it reads as generic and is overwhelmingly likely to be dismissed or denied by users who have no idea yet what they'd be agreeing to. Some products additionally show their *own* pre-prompt UI first ("Want updates? Enable notifications") that, only if accepted, then triggers the real browser permission dialog — this adds a soft, reversible ask before spending the actual one-shot browser prompt, though it does add an extra click for users who would have said yes anyway.

The trap: firing the permission request on page load or app mount "to get it out of the way early" — this is one of the most common, well-documented anti-patterns in push-notification UX precisely because a denial is hard to undo, making the timing of the ask a high-stakes, one-shot decision rather than a minor UX preference.

---

**Q (Low): Should the notification-center dropdown list be virtualized like the News Feed's list?**

Answer: It depends on the retention/volume clarified up front — if notifications are capped to a short recent window (say, the last 50–100, with older ones simply not retrievable through this UI), a plain, non-virtualized rendered list is entirely adequate and virtualization would be unnecessary complexity for a bounded, small list. If the product supports browsing a long, effectively unbounded notification history, the same variable-height virtualization approach used for the feed applies for the same underlying reason (bounding mounted DOM regardless of how far back a user scrolls) — this is a case of applying a general principle (virtualize genuinely long lists, don't for genuinely short ones) rather than a fixed rule that notification lists specifically always need or never need virtualization.

The trap: reflexively saying "yes, virtualize it" or "no, it's just notifications, no need" without first checking which volume/retention assumption the scenario actually established — the correct answer is conditional on that detail, and stating it as unconditional either way signals not having connected the technique to the reason it's used.

---

## Self-Assessment

- [ ] Can explain why one central, normalized notification store (not three independent listeners) is the right architecture for badge/toast/list consistency
- [ ] Can explain why in-app WebSocket delivery and background Service Worker push are two separate mechanisms, and why one can't substitute for the other
- [ ] Can design ID-based dedup that prevents the same event from double-rendering across in-app and push channels
- [ ] Can design a grouping key that correctly merges same-type-same-target events while keeping different event types on the same target separate
- [ ] Can explain the cross-tab (`BroadcastChannel`) vs. cross-device (server-authoritative) distinction for read-state sync
- [ ] Can justify contextual, intent-tied timing for the push permission prompt over requesting it on page load

---
*Next: Design an Image/Video Gallery With Lazy Loading — moves from event/notification delivery to media-heavy rendering at scale, where the central concerns become lazy-loading strategy, layout-shift prevention, and progressive/responsive image delivery.*
