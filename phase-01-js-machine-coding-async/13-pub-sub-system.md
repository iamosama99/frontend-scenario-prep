# Pub/Sub System

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Topic-based routing | `Map<topic, Set<handler>>` — subscribers register interest in a named topic, not a specific emitter instance | Decouples publishers from subscribers entirely — neither needs a reference to the other, only to the shared bus and a topic name |
| Unsubscribe via returned handle | `subscribe()` returns a function that removes exactly that registration | Avoids needing subscribers to keep track of "which handler did I register with which topic" themselves — the cleanup capability is handed back at registration time |
| Synchronous vs. queued delivery | Decide explicitly whether `publish` calls handlers synchronously or defers them (microtask/macrotask) | Synchronous delivery means a slow/throwing handler blocks the publisher and other subscribers; async delivery isolates them but changes ordering guarantees |
| Isolate handler failures | Wrap each handler invocation in its own `try/catch` | One subscriber's bug (a thrown error) must not prevent other subscribers on the same topic from receiving the same event |

## The Scenario

"Build a simple pub/sub (publish/subscribe) system — `subscribe(topic, handler)`, `publish(topic, payload)`, `unsubscribe`. This is going to be the backbone for decoupling a few unrelated widgets on a dashboard — e.g., a filter panel publishes 'filters:changed' and several independent chart widgets subscribe to it without knowing about each other."

## Clarifying Questions

- **How is this different from a plain `EventEmitter` (`on`/`off`/`emit`), and does that difference matter here?** They're extremely similar in mechanism — the real distinction is more about intended usage pattern than implementation: an `EventEmitter` is usually a property of one specific object emitting events *about itself* (a specific socket, a specific stream), while pub/sub is usually a shared, topic-addressed bus that many unrelated publishers and subscribers all talk through without holding a reference to each other. I'd confirm which flavor is actually wanted — the dashboard example described (decoupled widgets, shared bus) is the classic pub/sub use case, not a per-instance emitter.
- **Should handlers run synchronously (blocking the publisher until all handlers finish) or asynchronously (queued via microtask, publisher continues immediately)?** This has real behavioral consequences: synchronous delivery means `publish()` doesn't return until every handler has run, and a slow handler delays everything after it, including other handlers on the same event. I'd propose synchronous-by-default (matching Node's `EventEmitter` and most simple pub/sub libraries) since it's simpler to reason about, but flag async delivery as worth considering if handlers can be slow or if publish-time ordering shouldn't be coupled to subscriber execution time.
- **Should a publish with no active subscribers on that topic be a no-op, or is it worth warning/logging?** For a dashboard where widgets mount/unmount dynamically, publishing to a topic with zero current subscribers (e.g., a chart widget hasn't mounted yet, or already unmounted) is completely normal and shouldn't be treated as an error — I'd default to silent no-op, possibly with an optional debug-mode logging hook.
- **Do subscribers need wildcard/pattern topic matching** (e.g., subscribing to `'filters:*'` to catch `'filters:changed'` and `'filters:reset'`), or is exact topic-string matching sufficient? I'd default to exact matching for simplicity and ask if wildcard support is actually needed, since it adds real complexity (pattern compilation, matching cost per publish) for a feature that may not be used.

## Approach & Trade-offs

The core data structure is a `Map` from topic name to a `Set` of handler functions — a `Set`, not an array, specifically because it gives O(1) `add`/`delete`/`has` and, more importantly, naturally prevents the exact same function reference from being registered twice under careless double-subscription (arrays would silently allow duplicate registrations, calling the same handler multiple times per publish).

**Unsubscribe via a returned handle, not a separate `unsubscribe(topic, handler)` call**: I chose to have `subscribe()` return a small `() => void` function that, when called, removes that exact registration — this is a slightly nicer API than requiring the caller to hold onto both the topic string and the original handler reference to unsubscribe later (which is error-prone if the handler was an inline arrow function that the caller doesn't have a stable reference to anymore). Both approaches are valid and seen in real libraries (Node's `EventEmitter` uses the `off(event, handler)` style, RxJS-influenced libraries tend to use the returned-disposer style) — I'd mention this as a deliberate API choice, not the only correct one.

**Synchronous delivery, each handler wrapped in its own `try/catch`**: I default to calling handlers synchronously and in registration order, matching the mental model most engineers already have from `EventEmitter`/DOM events. The critical detail is isolating failures — if handler A throws, handlers B and C for the same topic must still run; a naive `for (const handler of handlers) handler(payload)` loop without a `try/catch` inside it would have handler A's exception propagate out of the entire `publish()` call, silently preventing B and C from ever running, and likely crashing whatever code called `publish()` in the first place (a filter panel shouldn't crash because one chart widget's handler has a bug).

**Snapshotting the handler set before iterating**: if a handler, when invoked, synchronously calls `unsubscribe()` (its own, or another handler's) or subscribes a *new* handler to the same topic being published — both realistic scenarios (a "run once" pattern, or a handler that reacts to an event by registering a follow-up listener) — mutating the `Set` being iterated *during* that same iteration has implementation-defined-feeling but actually spec-defined (for `Set`, newly added elements *are* visited if added during iteration; removed-but-not-yet-visited elements are skipped) behavior that's easy to get subtly wrong. I iterate over a copy (`[...handlers]`) taken at the start of `publish()`, so the current publish's handler list is stable regardless of what handlers do to the subscriber list mid-flight — new subscriptions apply starting from the *next* publish, not retroactively to the one in progress.

**Optional `once` support** as a thin wrapper around `subscribe`+auto-unsubscribe, rather than a parallel code path — `once(topic, handler)` subscribes a wrapper function that calls the real handler and then immediately invokes its own returned unsubscribe function, keeping the "once" logic from duplicating the core subscribe/publish machinery.

## Solution

```javascript
class PubSub {
  #topics = new Map(); // topic -> Set<handler>

  subscribe(topic, handler) {
    if (!this.#topics.has(topic)) {
      this.#topics.set(topic, new Set());
    }
    this.#topics.get(topic).add(handler);

    // Return a disposer so callers don't need to retain (topic, handler) themselves.
    return () => {
      const handlers = this.#topics.get(topic);
      if (!handlers) return;
      handlers.delete(handler);
      if (handlers.size === 0) this.#topics.delete(topic); // avoid leaking empty Sets
    };
  }

  once(topic, handler) {
    const unsubscribe = this.subscribe(topic, (payload) => {
      unsubscribe();
      handler(payload);
    });
    return unsubscribe;
  }

  publish(topic, payload) {
    const handlers = this.#topics.get(topic);
    if (!handlers) return;

    // Snapshot before iterating: handlers may subscribe/unsubscribe during this
    // publish, and those changes should apply to future publishes, not this one.
    for (const handler of [...handlers]) {
      try {
        handler(payload);
      } catch (err) {
        // One subscriber's failure must not stop delivery to the others.
        console.error(`PubSub handler for "${topic}" threw:`, err);
      }
    }
  }

  clear(topic) {
    if (topic) {
      this.#topics.delete(topic);
    } else {
      this.#topics.clear();
    }
  }
}
```

```javascript
const bus = new PubSub();

const unsubscribeChart1 = bus.subscribe('filters:changed', (filters) => {
  console.log('chart1 redrawing with', filters);
});
bus.subscribe('filters:changed', (filters) => {
  console.log('chart2 redrawing with', filters);
});

bus.publish('filters:changed', { category: 'shoes' });
// logs both chart1 and chart2 lines

unsubscribeChart1();
bus.publish('filters:changed', { category: 'bags' });
// only chart2 logs now

bus.once('filters:reset', () => console.log('reset handled — fires only once'));
bus.publish('filters:reset', null);
bus.publish('filters:reset', null); // no log the second time
```

> **Check yourself:** If a handler subscribed to `'filters:changed'` calls `bus.subscribe('filters:changed', newHandler)` from *inside* itself, during an in-progress `publish('filters:changed', ...)` call, should `newHandler` be invoked as part of that same publish? Why does this implementation say no, and what would need to change to say yes?

## Gotchas

**Mutating the handler collection while iterating it during `publish`.** If a handler synchronously unsubscribes itself or another handler on the same topic mid-publish, iterating the live `Set` directly (instead of a snapshot) can skip handlers that haven't run yet, or — depending on engine/iteration-protocol specifics — behave inconsistently. Snapshotting the handler list (`[...handlers]`) at the start of each `publish` call sidesteps this entirely by making the current publish operate over a fixed list, regardless of concurrent mutation.

**Letting one handler's thrown error abort delivery to the rest.** Without a `try/catch` around each individual handler invocation, one buggy subscriber breaks every other subscriber on that topic for that publish — and since `publish()`'s caller (the filter panel, in this scenario) has no reason to expect that publishing an event could throw based on unrelated subscriber code, this can crash or corrupt state in a completely unrelated part of the app.

**Using an array instead of a `Set` for handlers, allowing accidental duplicate subscriptions.** If a component subscribes on every render (a classic React anti-pattern when subscription isn't properly scoped to mount/unmount via `useEffect`), an array-backed handler list accumulates duplicate registrations of the same handler, and each `publish` then calls that handler multiple times — a `Set` at least prevents the *exact same function reference* from being double-registered, though it doesn't fully solve "subscribed in the wrong lifecycle" (see below).

**Not unsubscribing on component unmount, causing memory leaks and "ghost" updates.** This is the pub/sub analogue of the "memory leak from uncleaned subscriptions" React debugging scenario — a widget that subscribes to a topic on mount must call its returned unsubscribe function on unmount; forgetting this keeps a reference to a handler closing over the (now-unmounted) component's state/props alive indefinitely, both leaking memory and, if the handler calls `setState` on an unmounted component, could produce warnings or bugs depending on the framework.

**Silently swallowing subscriber errors without any visibility.** Isolating errors (so one handler's failure doesn't break others) is correct, but doing so with an empty `catch {}` block, with no logging at all, trades one bug (crash) for another (silent failure that's much harder to debug later) — errors should be caught *and* reported (logged, sent to error tracking) even while being contained.

## Follow-up Questions

**Q (High): How is pub/sub different from a plain `EventEmitter`, and when would you reach for one over the other?**

Answer: Mechanically, they're nearly identical — both maintain a mapping from event/topic name to a list of handlers and invoke them on emission. The distinction is architectural intent: an `EventEmitter` is typically a property *of* a specific object, emitting events *about that object's own lifecycle* (a specific WebSocket's `'message'` event, a specific stream's `'data'` event) — consumers hold a reference to that particular instance. Pub/sub is typically a shared, application-wide (or module-wide) bus that arbitrary, mutually-unaware publishers and subscribers route messages through by topic name alone, without either side holding a reference to the other — which is exactly the decoupling the dashboard scenario wants (the filter panel doesn't import or reference the chart widgets, and vice versa). In practice, many pub/sub implementations *are* built as a thin wrapper around an `EventEmitter`-shaped core, so the "difference" is often more about how the same underlying mechanism is deployed and named than about fundamentally different code.

The trap: treating these as fundamentally different data structures requiring different algorithms — the interview-worthy nuance is that the difference is architectural/usage-pattern, not implementation, and being able to articulate *when* the decoupled-bus framing is the right mental model (many-to-many, mutually unaware participants) versus a per-instance emitter (one object, many listeners to *that object's* events) is the actual signal being tested.

---

**Q (High): Why is it important to isolate each subscriber's handler execution in its own `try/catch` during `publish`, and what happens if you don't?**

Answer: Pub/sub's entire value proposition is decoupling — the publisher shouldn't need to know or care what subscribers exist, and subscribers shouldn't be able to interfere with each other. If one handler throws and that exception isn't caught locally, it propagates up through the `publish()` call itself, which means: (1) every subsequent handler in that same publish's iteration never runs, so, in the dashboard example, chart2 silently never gets the filter update because chart1's handler happened to have a bug — a failure with no relationship to chart2 breaks chart2 anyway; and (2) the exception surfaces in the *publisher's* call stack (the filter panel's code), which has no way to meaningfully handle an error that's actually about some unrelated subscriber's internal logic. Wrapping each handler invocation individually contains failures to exactly the handler that caused them, preserving the decoupling guarantee pub/sub is supposed to provide.

The trap: adding a single `try/catch` around the *entire* `for` loop in `publish` instead of around each individual handler call inside the loop — this still doesn't run later handlers after an earlier one throws (the `catch` only stops the exception from escaping `publish` itself, but the loop has already been aborted by the throw), so it looks like a fix but doesn't actually solve the "every subscriber should still get delivered to" requirement.

---

**Q (Medium): Should `publish` deliver events synchronously or asynchronously, and what breaks if you pick the wrong one for a given use case?**

Answer: Synchronous delivery (calling handlers immediately, in-line, before `publish()` returns) is simpler to reason about and gives deterministic ordering relative to the calling code — but it means `publish()`'s caller is blocked until every handler finishes, so a slow handler (one doing expensive synchronous work, or one that itself triggers further synchronous publishes) directly slows down the publisher and delays every other handler queued after it in the same publish. Asynchronous delivery (deferring each handler call via `queueMicrotask` or `setTimeout`) decouples the publisher's return time from subscriber execution time and prevents one slow handler from blocking others, but changes ordering guarantees — code immediately after a `publish()` call now runs *before* any subscriber has been notified, which can surprise callers who assumed synchronous "fire and every subscriber has definitely already reacted" semantics (e.g., code that publishes an event and then immediately reads some state a subscriber was expected to have just updated).

The trap: picking one universally without acknowledging the trade-off is context-dependent — a UI event bus reacting to user clicks typically wants synchronous delivery (immediate, predictable feedback), while a system distributing potentially-slow or many-subscriber notifications (e.g., logging/analytics fan-out) often benefits from async delivery to avoid blocking the critical path; presenting either choice as objectively correct without naming what it costs is the weaker answer.

---

**Q (Medium): How would you add wildcard/namespaced topic support (e.g., subscribing to `'filters:*'` to receive both `'filters:changed'` and `'filters:reset'`)?**

Answer: Exact-match topic lookup (`Map.get(topic)`) no longer suffices once patterns are involved — `publish('filters:changed', ...)` now needs to find every *matching* subscription, not just an exact key hit. A straightforward approach: store wildcard subscriptions separately (or all subscriptions in a structure like an array of `{ pattern, handler }`), and on publish, in addition to the exact-match `Set`, iterate wildcard subscriptions and test each pattern against the published topic (e.g., converting a simple glob like `'filters:*'` into a regex, or doing a segment-by-segment match if topics are colon/dot-delimited namespaces). This adds real per-publish cost (pattern matching instead of an O(1) map lookup) proportional to the number of wildcard subscriptions, which is worth naming as a trade-off — a system with thousands of publishes per second and many wildcard subscribers would need a more efficient matching structure (e.g., a trie keyed by topic segment) rather than testing every pattern linearly.

The trap: assuming wildcard matching is "free" to add on top of the exact-match `Map` — it fundamentally changes the lookup from O(1) hash access to some form of pattern search, and glossing over that cost (especially in a system-design-adjacent follow-up) misses a real scalability consideration.

---

**Q (Low): How does a pub/sub bus like this relate to the Observer design pattern, and to how frameworks like Redux or RxJS structure event flow?**

Answer: Pub/sub is a direct, practical implementation of the Observer pattern — "subjects" (topics) that "observers" (subscribers) register interest in, with the subject notifying all current observers on a relevant change, without either side needing a direct reference to the other's concrete type. Redux's store is conceptually a single-topic pub/sub system in disguise: `store.subscribe(listener)` is exactly `subscribe`, and `dispatch(action)` triggers listener notification (though Redux's actual "topic" is just "the store changed at all" — a single implicit topic — with `listener`s expected to read `getState()` themselves and diff for relevant changes, rather than Redux delivering typed payloads per named topic). RxJS's `Observable`/`Subject` types are a considerably more powerful generalization of the same idea — a `Subject` is literally a multicast pub/sub primitive at its core, with the rest of RxJS's operator ecosystem (`map`, `filter`, `debounceTime`, `mergeMap`, etc.) built on top of that same subscribe/notify foundation to support composeable transformation of event streams, something a bare pub/sub bus doesn't provide out of the box.

The trap: describing pub/sub, Observer, Redux, and RxJS as unrelated technologies rather than recognizing them as points on the same conceptual spectrum (bare notify-on-change → typed/topic-addressed → composable stream transformation) — being able to place this scenario's implementation within that spectrum is a strong signal of broader architectural fluency, not just having memorized one specific pattern.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement `subscribe`/`publish`/`unsubscribe`/`once` with correct handler isolation from memory
- [ ] Can explain why the handler collection must be snapshotted before iterating in `publish`
- [ ] Can articulate the architectural (not implementation) difference between pub/sub and a plain `EventEmitter`
- [ ] Can explain the synchronous-vs-asynchronous delivery trade-off with a concrete example of what breaks under each choice
- [ ] Can connect this pattern to the Observer design pattern and name at least one real framework that implements a version of it

---
*Next: Array Method Polyfills: map/filter/reduce/flat — shifts from building coordination primitives (queues, buses) to re-implementing the standard library methods those primitives are usually built out of, testing fundamentals at an even lower level.*
