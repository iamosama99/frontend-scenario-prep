# Design an Instagram Stories-style Component

## Quick Reference

| Decision Point | Choice | Why |
|---|---|---|
| Progress/advance model | A single active timer per story item, paused on user hold/interaction, auto-advancing to the next item (or next author) on completion or explicit tap | Mirrors how the feature actually behaves — a visible per-item progress bar that's the authoritative driver of "when do we advance," not a UI decoration separate from the actual advance logic |
| Media preloading | Preload the next 1–2 items' media ahead of the currently-viewing one, not the entire story set up front, and not only the current item with no lookahead | Preloading everything up front wastes bandwidth on content the user may never reach (especially across many authors' story sets); preloading nothing ahead of time produces a visible load flash on every single advance — a short lookahead window is the standard middle ground |
| Pause behavior | Press-and-hold pauses the timer and (for video) the media; releasing resumes both from exactly where they left off | The core interaction users expect from this pattern — an accidental full skip or restart-from-zero on release would break the fundamental "hold to pause and read/watch" affordance |
| Navigating between authors' story sets vs. items within one author's set | Two distinct navigation levels — tap left/right (or swipe) moves between items within the current author's set; reaching the end advances to the next author's set entirely, from its first item | These are meaningfully different transitions (progress bar layout, preloading scope) and conflating them as "just one long flat sequence" loses the per-author grouping and progress-bar-per-author-set structure users expect |
| Seen/unseen tracking | Persisted per-user, per-story-item view state, synced to the server so "unseen" indicators (e.g., a colored ring around an author's avatar) are consistent across sessions/devices | A purely client-local seen/unseen record (e.g., only in memory or `localStorage`) doesn't survive a different device or a cleared browser, producing an inconsistent, occasionally wrong "have I seen this" indicator |

## The Scenario

"Design the 'Stories' feature — think Instagram or Snapchat Stories: a horizontal row of circular avatars, tapping one opens a full-screen sequence of that person's photos/videos that auto-advance with a visible per-item progress bar, tapping left/right or the edges of the screen navigates between items, and reaching the end moves to the next person's stories. Walk me through the architecture — the playback/timer logic, preloading, and the navigation model."

## Clarifying Questions

- **Can story items be a mix of images and videos, and if so, does a video's own natural duration drive the auto-advance timer, or does every item (image or video) advance on a fixed, uniform duration regardless of media type?** This changes the timer model meaningfully — a uniform fixed duration (e.g., every item, image or video, gets exactly 5 seconds) is a simpler, single timer mechanism; deriving the advance timing from a video's actual playback duration (advancing only once the video finishes, however long that is) requires the progress-bar/timer logic to be driven by media playback events rather than a fixed `setTimeout`.
- **What interactions need to be supported for navigation — tap zones (left third of screen = previous, right two-thirds = next), swipe gestures, keyboard (for a web/desktop version), or all of the above?** A web-first design (this repo's scenarios use React + TypeScript, implying a web context) still commonly needs tap-zone and swipe support for touch devices alongside keyboard arrow-key support for desktop/accessibility — worth confirming all the expected input modalities before designing the gesture/tap-zone handling.
- **How many authors' story sets, and roughly how many items per author, are we designing for — a small following list, or a scale where preloading strategy meaningfully affects data usage (e.g., mobile users on limited data plans)?** At larger scale, preloading strategy (how far ahead to fetch media) becomes a real, deliberate trade-off between perceived smoothness and unnecessary data/bandwidth consumption, rather than something that can default to "just preload everything."
- **Does viewing a story need to notify the author (e.g., a "seen by" list), and if so, does that need to be real-time or is an eventually-consistent view-count sufcient?** Affects whether view-tracking needs immediate server notification per item viewed or can be batched/sent less urgently.
- **Should the progress bar reflect exact elapsed time precisely (matters if, say, a user could scrub/seek within an item), or is a purely forward-only, non-seekable progress indicator sufficient?** Most story implementations are forward-only with no scrubbing, which meaningfully simplifies the progress bar to a one-directional animation rather than a full interactive seek control — worth confirming this simpler assumption holds before designing anything more elaborate.

## Approach & Trade-offs

**The progress bar must be the authoritative driver of advancement, not a decorative animation running alongside separately-implemented timer logic.** A common design mistake is implementing "advance to next item after 5 seconds" as an independent `setTimeout` and, separately, a CSS or JS-driven progress bar animation intended to visually match that same 5 seconds — these two independently-implemented mechanisms are trivially prone to drifting out of sync (a pause/resume that correctly pauses the `setTimeout` but not the CSS animation, for instance, or vice versa), and any drift is immediately, jarringly visible to the user (the bar reaching full width before or after the actual advance fires). The correct model treats the progress state as a single source of truth (a current elapsed-time or fraction-complete value) that drives both the visual bar's width and the advance decision, so pausing, resuming, or otherwise interacting with it can never leave the two out of sync, because there's only ever one thing to pause or resume.

**Media preloading needs a bounded lookahead window, not "everything" or "nothing," and this is a genuine bandwidth-vs-smoothness trade-off worth naming explicitly.** Preloading only the currently-viewing item with no advance fetch produces a visible loading flash on every single advance to the next item (a broken-feeling experience for a feature whose whole premise is fast, continuous consumption); preloading an entire author's full story set (or worse, every followed author's entire set) the moment the feature is opened wastes substantial bandwidth on content a user may abandon well before reaching (a very common behavior pattern — many viewers don't watch every single story from every author before moving on or closing the feature). The standard middle ground preloads the next one or two items ahead of whatever is currently showing, advancing that lookahead window as the user progresses — enough to eliminate the visible load flash on a normal-paced advance, without committing bandwidth to content that may never be reached.

**Pausing on press-and-hold must genuinely pause both the timer/progress state and any actively-playing media, and resume both from the exact same point — not restart, and not merely pause visually while the timer silently keeps counting underneath.** This is the core interaction affordance of the entire feature (users expect to be able to hold to pause and read a caption or watch more carefully), and a broken version of it — where releasing a hold causes an unexpected skip-ahead (because the underlying timer kept running invisibly while paused visually) or a restart-from-zero (because pausing a video's playback also, incorrectly, reset it) — undermines a genuinely fundamental, expected behavior of the pattern. This reinforces the single-source-of-truth point above: if the progress/timer state is one authoritative value driving both the bar and the advance decision, "pause" simply means "stop advancing that value," and "resume" means "continue advancing it from wherever it was" — there's no separate thing to accidentally desynchronize.

**Navigating within one author's items versus advancing to the next author's entire set are two distinct levels of the navigation model, not one flat sequence.** Tapping/swiping within a currently-open author's story set moves between that author's individual items, updating that author's own multi-segment progress bar (typically rendered as N discrete segments across the top, one per item, showing which have been viewed/are current/are upcoming); reaching the last item and advancing further (or swiping past the end) transitions to an entirely new author's set — a materially bigger transition involving a fresh progress-bar-segment-count (matching the new author's item count), a fresh preload window, and generally a distinct visual transition (a horizontal slide to the next avatar's stories, commonly). Modeling these as two distinct levels (an "author sequence" containing a "current author's item sequence") rather than flattening everything into one long list keeps the progress-bar-per-author-set structure and the preloading scope correctly bounded to "the current author's nearby items," rather than needing to reason about position within a single enormous cross-author flat list.

**Seen/unseen state needs to be a server-synced, per-user record, not merely a client-local flag, because the unseen indicator (commonly a colored ring around an avatar) needs to be consistent regardless of which device or session the user is currently on.** Marking an item "viewed" once its progress bar completes (or once some minimum viewing threshold is met, if the product wants to avoid counting an extremely brief accidental open as a full "view") should fire an update to the server recording that view for this user against that specific story item — and the avatar-ring unseen-indicator shown in the story row should be driven by the server-confirmed view record (fetched/synced on load), not purely local state that would reset or diverge the moment the user opens the app on a different device or after clearing local storage.

## Solution

**A single authoritative progress/timer hook driving both the visual bar and the advance decision:**

```tsx
function useStoryProgress(durationMs: number, onComplete: () => void) {
  const [elapsedMs, setElapsedMs] = useState(0);
  const [isPaused, setIsPaused] = useState(false);
  const rafRef = useRef<number>();
  const lastFrameTimeRef = useRef<number>();

  useEffect(() => {
    function tick(now: number) {
      if (!isPaused) {
        const delta = now - (lastFrameTimeRef.current ?? now);
        setElapsedMs((prev) => {
          const next = prev + delta;
          if (next >= durationMs) { onComplete(); return durationMs; }
          return next;
        });
      }
      lastFrameTimeRef.current = now;
      rafRef.current = requestAnimationFrame(tick);
    }
    rafRef.current = requestAnimationFrame(tick);
    return () => { if (rafRef.current) cancelAnimationFrame(rafRef.current); };
  }, [isPaused, durationMs, onComplete]);

  return {
    progressFraction: Math.min(1, elapsedMs / durationMs), // drives the bar's width — SAME value that drives advancement
    pause: () => setIsPaused(true),
    resume: () => setIsPaused(true) && setIsPaused(false), // resumes from current elapsedMs — no reset
  };
}
```

Driving the progress bar's rendered width directly from `progressFraction` — the exact same value `onComplete` fires from — is what guarantees the bar and the advance decision can never drift apart; there is only one number being tracked, not two independently-animated approximations of the same intended duration.

**Two-level navigation model — author sequence containing an item sequence:**

```ts
interface StoryItem { id: string; mediaUrl: string; mediaType: 'image' | 'video'; durationMs: number; }
interface AuthorStorySet { authorId: string; items: StoryItem[]; }

function useStoriesNavigation(authorSets: AuthorStorySet[]) {
  const [authorIndex, setAuthorIndex] = useState(0);
  const [itemIndex, setItemIndex] = useState(0);

  function nextItem() {
    const currentSet = authorSets[authorIndex];
    if (itemIndex + 1 < currentSet.items.length) {
      setItemIndex(itemIndex + 1); // advance within the current author's items
    } else if (authorIndex + 1 < authorSets.length) {
      setAuthorIndex(authorIndex + 1); // exhausted this author — move to the NEXT author's set entirely
      setItemIndex(0);
    } else {
      closeStoriesViewer(); // exhausted every author — nothing left to advance to
    }
  }

  function prevItem() {
    if (itemIndex > 0) setItemIndex(itemIndex - 1);
    else if (authorIndex > 0) {
      const prevSet = authorSets[authorIndex - 1];
      setAuthorIndex(authorIndex - 1);
      setItemIndex(prevSet.items.length - 1); // enter the previous author's set at ITS last item
    }
  }

  return { authorIndex, itemIndex, nextItem, prevItem };
}
```

**Bounded-lookahead preloading, advancing its window alongside navigation:**

```ts
function usePreloadLookahead(currentItems: StoryItem[], currentIndex: number, lookahead = 2) {
  useEffect(() => {
    for (let i = currentIndex + 1; i <= currentIndex + lookahead && i < currentItems.length; i++) {
      const item = currentItems[i];
      if (item.mediaType === 'image') {
        const img = new Image();
        img.src = item.mediaUrl; // browser caches it — instant when actually navigated to
      }
      // video preloading commonly uses a hidden <video preload="auto"> element per lookahead item instead
    }
  }, [currentIndex, currentItems, lookahead]);
}
```

**Seen/unseen tracking, synced server-side rather than kept only in local state:**

```ts
function useMarkStoryViewed(authorId: string, itemId: string, progressFraction: number) {
  const hasMarkedRef = useRef(false);
  useEffect(() => {
    if (progressFraction >= 0.7 && !hasMarkedRef.current) { // a meaningful-viewing threshold, not a bare instant-open
      hasMarkedRef.current = true;
      recordStoryView(authorId, itemId); // persisted server-side — drives the unseen-ring indicator across all sessions/devices
    }
  }, [progressFraction, authorId, itemId]);
}
```

> **Check yourself:** Without looking above, explain why the progress bar's rendered width and the auto-advance decision must be derived from the exact same underlying value, and describe the specific user-visible bug that results from implementing them as two independently-animated approximations of the same duration instead.

## Gotchas

**Implementing the progress bar's animation and the advance timer as two separate mechanisms rather than one authoritative value driving both.** The single most common structural mistake for this scenario — any drift between them (especially around pause/resume) is immediately, visibly wrong to the user.

**Pausing the visual progress bar on hold without actually pausing the underlying timer/media, or vice versa.** Produces exactly the "unexpected skip on release" or "restart from zero" bugs that break the core, expected hold-to-pause affordance.

**Preloading either everything up front or nothing ahead of time, with no bounded lookahead.** The former wastes bandwidth on content that may never be reached; the latter produces a visible load flash on every advance — a small bounded lookahead window is the standard, deliberate middle ground.

**Flattening the author-sequence-of-item-sequences into one long flat list of items across all authors.** Loses the natural per-author progress-bar-segment grouping and complicates preloading scope (which should stay bounded to "near the current author's current position," not "somewhere in one enormous cross-author list").

**Keeping seen/unseen state only in client-local memory or storage.** Produces an inconsistent unseen-indicator ring that doesn't carry over across devices or a cleared browser — this needs to be a server-synced per-user record, the same durability requirement seen in other cross-session state throughout this repo.

**Marking a story "viewed" the instant it's opened, with no minimum-viewing threshold.** A story opened and immediately swiped past within a fraction of a second arguably wasn't meaningfully "seen" — some minimum elapsed-fraction threshold before recording a view is a reasonable, common product choice worth surfacing rather than assuming "opened" and "viewed" are identical.

## Follow-up Questions

**Q (High): Walk through exactly what happens, mechanically, when a user presses and holds on a currently-playing video story item, then releases after 3 seconds.**

Answer: On press, the single authoritative progress hook's `pause()` is called, which stops the `requestAnimationFrame`-driven elapsed-time accumulation (the `isPaused` flag short-circuits further elapsed-time updates in the tick function) — simultaneously, the currently-playing `<video>` element's own playback is paused via its native `pause()` method, driven by the same press event handler, so both the timer and the actual video freeze together at the same instant. While held, neither advances — the progress bar's rendered width stays fixed at whatever fraction it had reached, and the video frame stays static. On release, `resume()` clears the paused flag, and the `requestAnimationFrame` loop resumes accumulating elapsed time starting from exactly where it left off (not reset to zero, since `elapsedMs` itself was never touched during the pause, only its accumulation was halted) — and the video element's `play()` is called, resuming playback from its own currently-paused timestamp (native HTML video elements preserve their playback position on pause by default, requiring no special handling to "remember" where they were). The net effect: after a 3-second hold-and-release, the story item has effectively gained 3 extra seconds of total display time compared to had it not been paused, with no skip, no restart, and no desync between the bar and the actual media.

The trap: describing only the timer-pausing half or only the video-pausing half — a complete answer must address both mechanisms being paused and resumed together, driven by the same interaction event, since a design that pauses one but not the other produces exactly the desync bugs this scenario's Gotchas section calls out.

---

**Q (High): How would you determine the right preload lookahead window size, and what would you actually measure to validate the choice?**

Answer: The lookahead window size is a direct trade-off between two measurable, opposing costs — too small a window (e.g., only preloading the immediately-next item) risks a visible load stall on faster-than-typical advance sequences (a user quickly tapping through several items in succession, faster than each item's media has time to fully preload one-at-a-time), while too large a window wastes bandwidth on content a meaningful fraction of users never reach (measurable via analytics on how many items into an author's story set the median viewer actually gets before navigating away or closing the viewer). A reasonable approach starts with a modest default (2 items ahead, as sketched above) and validates/tunes it against real usage data — specifically, the rate of visible load-stall events actually experienced by users (a client-side signal: "the next item's media wasn't yet loaded when navigation to it occurred") traded off against measured wasted-preload bytes (media preloaded but never actually viewed, derivable from comparing preload-triggered fetches against actual view-completion events). This turns what could be an arbitrary constant into a metric-driven tuning decision.

The trap: picking an arbitrary lookahead number with no stated basis, or reflexively maximizing it ("preload as much as possible for maximum smoothness") without acknowledging the real bandwidth cost that scales with over-preloading — a strong answer explicitly names both sides of the trade-off and what would be measured to tune it, rather than asserting one extreme as obviously correct.

---

**Q (High): A video story item's natural duration (say, 15 seconds) differs from a fixed uniform duration used for image items (say, 5 seconds). How does the progress/timer model need to change to support both in the same sequence?**

Answer: The single authoritative progress value from the earlier solution generalizes cleanly — instead of a fixed constant `durationMs` passed uniformly to every item, each item (image or video) supplies its own `durationMs` (a fixed product-chosen value for images, and the video element's actual `duration` — read from its metadata once loaded — for videos), and the same progress hook logic applies unchanged regardless of which kind of item is currently active. The one added wrinkle for video specifically is that the progress value driving the bar should, for a smoother and more accurate result, be derived from the video element's own `currentTime`/`duration` (updated via its `timeupdate` event) rather than purely from a `requestAnimationFrame` loop tracking wall-clock elapsed time — this keeps the bar accurately reflecting actual video playback progress even if the video briefly buffers/stalls (wall-clock elapsed time would otherwise keep advancing the bar even during a moment the video itself isn't actually progressing, causing the same kind of desync the design otherwise works hard to avoid). Image items, having no independent playback clock of their own, are well served by the simpler `requestAnimationFrame`-driven elapsed-time approach already described.

The trap: assuming a single uniform timer mechanism (one fixed duration, one `requestAnimationFrame` loop) can drive both media types identically without modification — a video item's actual playback (which can stall on buffering, unlike a simple wall-clock timer) needs its progress driven by the video's own playback events specifically to avoid the bar advancing ahead of a video that's actually still buffering.

---

**Q (Medium): How would you support keyboard navigation (left/right arrow keys) for a desktop/web version of this feature, and does it change anything about the progress/pause model?**

Answer: Keyboard arrow keys map directly onto the same `nextItem`/`prevItem` navigation functions already driving tap-zone and swipe navigation — no new navigation logic is needed, just an additional input handler (a `keydown` listener while the stories viewer is open) invoking the same functions. The one addition worth considering for parity with touch's press-and-hold-to-pause gesture is a keyboard-equivalent pause affordance (commonly the spacebar, matching the widely-understood "space to pause/play media" convention) wired to the same `pause`/`resume` functions already used for touch-hold — reusing the identical single-source-of-truth progress hook, since pause/resume behavior shouldn't differ based on which input method triggered it.

The trap: building a parallel, separate keyboard-specific navigation or pause implementation rather than wiring keyboard events into the exact same navigation and pause/resume functions already used for touch — this is a straightforward case of one interaction intent (advance, go back, pause) having multiple possible input triggers (tap zone, swipe, key press), which should converge on the same underlying handlers rather than duplicating logic per input modality.

---

**Q (Medium): Should a story item's "seen" state be recorded the moment its progress bar completes, or could recording it too early or too late cause a problem?**

Answer: Recording view state strictly on full completion (progress fraction reaching 1.0) risks under-counting genuine views for users who navigate away just slightly before an item's timer would have naturally completed (a very common pattern — swiping to the next item a bit early once a viewer feels they've seen enough) — treating "didn't wait for the literal last millisecond" as "never viewed" undercounts real engagement and can leave the unseen-indicator ring incorrectly still showing for content the user meaningfully did view. A minimum-viewing-threshold approach (marking viewed once some meaningful fraction, e.g., 70%, of the item's duration has elapsed, as sketched in the solution above) better matches what most users and products would consider "having seen" an item, without requiring exact full completion. Recording too early (e.g., the instant an item opens, with no threshold at all) has the opposite problem — an accidental brief open immediately swiped past registers as a full view, which may overcount in ways that make view-count-based features (like a "seen by" list) misleadingly inclusive.

The trap: picking either extreme (only-on-full-completion, or the-instant-it-opens) without recognizing both have real, distinct downsides — a thoughtful middle-ground threshold, explicitly justified, is the stronger answer than either extreme asserted without qualification.

---

**Q (Low): How would you handle a story item that fails to load (a broken image URL, or a video that fails to buffer) in the middle of a viewing session?**

Answer: The failure should be treated as an explicit, distinct state for that item (not silently skipped and not left frozen indefinitely on a broken/loading placeholder) — a reasonable default shows a brief, clear "couldn't load this" indicator for that specific item and then auto-advances to the next item after a short pause, rather than either halting the entire viewing sequence indefinitely or silently skipping with no indication anything went wrong (which could otherwise look like a shorter-than-expected story set for no apparent reason). This mirrors the general principle (seen elsewhere in this phase, e.g., the moderation-removed comment case in Live Comments) that a piece of content becoming unavailable mid-session should be surfaced as an explicit state, not an unexplained gap.

The trap: either blocking the entire sequence waiting indefinitely for a failed item to somehow succeed, or silently skipping it with zero user-facing indication — both leave the user confused about what happened, versus a brief, explicit failure indicator that still keeps the overall viewing flow moving forward.

---

## Self-Assessment

- [ ] Can explain why the progress bar and the auto-advance decision must be driven by one authoritative value, and describe the desync bug that results from splitting them into two mechanisms
- [ ] Can design pause/resume that stops and restarts both the timer and any playing media from the same point, with no skip or reset
- [ ] Can design the two-level author-sequence/item-sequence navigation model and explain why flattening it loses the per-author progress structure
- [ ] Can justify a bounded preload lookahead window over preloading everything or nothing, and describe what metrics would validate its size
- [ ] Can explain why seen/unseen state must be server-synced rather than client-local only
- [ ] Can adapt the timer model to a mixed image/video sequence, including why video progress should be driven by playback events rather than wall-clock time alone

---
*Next: Design a Schema-driven Form Builder — moves into a metadata/configuration-driven rendering problem: a form's fields, validation, and layout are described by data (a schema) rather than hardcoded JSX, where the central new concerns become schema-to-component mapping, dynamic validation, and conditional field logic driven entirely by configuration.*
