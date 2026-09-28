# Live Region Announcements for Async Updates

## Quick Reference

| Situation | Mechanism | Why |
|---|---|---|
| Non-urgent status change (results loaded, item added) | `role="status"` (implicit `aria-live="polite"`) | Announced after the user's current activity finishes, not interrupting |
| Urgent/error condition (form submission failed, session about to expire) | `role="alert"` (implicit `aria-live="assertive"`) | Interrupts immediately — reserve for things that genuinely can't wait |
| The region's presence | Must exist in the DOM (even empty) *before* its content changes | AT only picks up on live regions it already knows about; injecting a brand-new live-region element with content already inside it is often not announced at all |
| Rapid/repeated updates | Debounce or coalesce before writing to the region | Firing on every keystroke/tick produces overlapping, unintelligible announcements |
| `aria-atomic` | `true` when the whole region should be re-read on any change, not just the changed part | Prevents fragment announcements like just a new number with no surrounding context |

## The Scenario

"We've got a page that does a lot of async stuff — a search box that shows a result count as you type, a save button that shows success/error after a network call, a background sync indicator. Product says screen reader users have no idea any of this is happening. Design and implement the live-region strategy for this page."

## Clarifying Questions

- **For each of these async updates, does the user need to hear it *immediately*, or is it fine if it's announced once their current screen-reader activity (like reading a sentence) finishes?** This is the actual decision criterion for `polite` vs. `assertive`, and it's per-update, not a single blanket choice for the page — a result count updating as someone types is not urgent; a payment failing after they hit submit arguably is.
- **How frequently does each of these values change — once per action, or continuously (a live counter, a streaming progress percentage)?** Continuously-updating content needs an explicit throttling/coalescing strategy or it produces a wall of unintelligible, overlapping announcements; a one-shot status change doesn't need that complexity.
- **Is there already a shared "toast" or "notification" component on this page, and should announcements piggyback on it or be a separate, purpose-built region?** If a toast system already exists (see [Toast / Notification Queue System](../phase-02-component-machine-coding/11-toast-notification-queue-system.md)), the right move is usually making the *existing* toast container the live region, rather than adding a second, parallel announcement mechanism that can talk over it.
- **Does the sync indicator need to announce every state transition (syncing → synced), or only failure — and would announcing every success actually be more noise than help?** Some of what "screen reader users have no idea this is happening" describes is legitimately missing information; some of it, if implemented naively, becomes *more* noise than a sighted user experiences, since a sighted user can glance at and ignore a spinner in a way a screen reader user can't glance-and-ignore an announcement.

## Approach & Trade-offs

**The single most consequential decision on this scenario is `polite` versus `assertive`, and the failure mode in both directions is real: overusing `assertive` makes a page exhausting and untrustworthy (constant interruptions train users to stop listening, the same way an over-alerting monitoring system trains engineers to ignore pages), while overusing `polite` for things that are genuinely urgent means a critical failure can go unnoticed if the user happens to have navigated away from that part of the page.** I'd apply a simple test per update: would a sighted user's task be meaningfully derailed by *not* noticing this immediately? A failed payment, yes — assertive. A background sync completing successfully, no — polite, or arguably no announcement at all if it's not actionable.

**I model live regions as a small, fixed set of purpose-scoped containers that persist in the DOM for the page's lifetime, not one created ad hoc per event.** The critical, frequently-missed mechanic: assistive tech builds its awareness of "this element is a live region" when the region is first rendered — if a component conditionally renders a brand-new `<div aria-live="polite">Success!</div>` only at the moment there's something to announce (mounting the div and its content in the same render), many AT/browser combinations miss the announcement entirely, because there was no prior "empty" live region for them to have already registered as one to watch. The reliable pattern is: render the live region container, empty, on initial page load, and only ever update its *text content* afterward — never its existence.

**Announcements for a search-as-you-type result count need explicit debouncing, decoupled from whatever debounce already governs the actual network request.** These can share a delay value but conceptually serve different purposes — the request debounce exists to avoid hammering the network; the announcement debounce exists to avoid hammering the user's ears. I'd default to updating the live region's text no more than roughly once every 500ms–1s even if results are arriving faster, and specifically make sure the *final* settled count is what gets announced, not a stale intermediate one dropped by an unlucky debounce boundary — the standard trick is always scheduling the announcement from the latest available count on a trailing-edge debounce, never firing one from a mid-flight value that's about to be superseded.

**For the background sync indicator, I'd deliberately *not* announce every state transition, and treat that as a considered accessibility decision, not an omission.** A sighted user's experience of a small spinner-then-checkmark icon is genuinely low-attention — it's peripheral, ignorable, glanced at only if something seems off. Forcing an equivalent-information screen reader experience by announcing "Syncing… Synced" on every cycle produces something *more* intrusive than the sighted experience it's supposed to match, which inverts the goal. I'd announce failures (assertive, since a failed sync a user doesn't know about can mean real data loss) and skip routine successes, treating "parity of intrusiveness," not "parity of every discrete event," as the actual target.

## Solution

A small, reusable live-region primitive, rendered once near the app root so it's always present in the DOM:

```tsx
type Politeness = 'polite' | 'assertive';

function LiveRegion({ politeness = 'polite' }: { politeness?: Politeness }) {
  const [message, setMessage] = useState('');
  return (
    <div
      aria-live={politeness}
      role={politeness === 'assertive' ? 'alert' : 'status'}
      aria-atomic="true"
      className="visually-hidden"
    >
      {message}
    </div>
  );
}
```

Rather than colocating live-region state per feature, I'd centralize announcement dispatch through a small hook/context so any part of the app can announce without needing its own region instance — this also makes debouncing a single, shared concern instead of reimplemented per feature:

```tsx
const AnnounceContext = createContext<(msg: string, politeness?: Politeness) => void>(() => {});

function AnnounceProvider({ children }: { children: React.ReactNode }) {
  const [polite, setPolite] = useState('');
  const [assertive, setAssertive] = useState('');
  const politeTimer = useRef<ReturnType<typeof setTimeout>>();

  const announce = useCallback((msg: string, politeness: Politeness = 'polite') => {
    if (politeness === 'assertive') {
      setAssertive(msg); // urgent: no debounce, fire immediately
      return;
    }
    clearTimeout(politeTimer.current);
    politeTimer.current = setTimeout(() => setPolite(msg), 500); // trailing-edge debounce
  }, []);

  return (
    <AnnounceContext.Provider value={announce}>
      {children}
      <div aria-live="polite" role="status" aria-atomic="true" className="visually-hidden">{polite}</div>
      <div aria-live="assertive" role="alert" aria-atomic="true" className="visually-hidden">{assertive}</div>
    </AnnounceContext.Provider>
  );
}

function useAnnounce() {
  return useContext(AnnounceContext);
}
```

Usage across the three cases from the scenario:

```tsx
// Search result count — polite, debounced by the provider itself
function SearchResults({ results }: { results: Item[] }) {
  const announce = useAnnounce();
  useEffect(() => {
    announce(`${results.length} result${results.length === 1 ? '' : 's'} found`);
  }, [results, announce]);
  // ...render results
}

// Save button — success is polite (nice to confirm, not urgent); failure is assertive
async function handleSave() {
  try {
    await saveItem();
    announce('Saved');
  } catch {
    announce('Save failed. Please try again.', 'assertive');
  }
}

// Background sync — deliberately silent on success, assertive only on failure
function useBackgroundSync() {
  const announce = useAnnounce();
  useEffect(() => {
    const unsubscribe = syncEngine.on('error', () =>
      announce('Sync failed — your changes may not be saved.', 'assertive')
    );
    return unsubscribe;
  }, [announce]);
}
```

Repeating the same message twice in a row (e.g., "Saved" fired again from a second identical save) won't re-announce in most AT, since the text content didn't change — a common enough case worth handling explicitly by appending a no-op distinguishing token or briefly clearing the region first if a genuine re-announcement of identical text is needed.

> **Check yourself:** Trace what a screen reader does, step by step, if the live-region `<div>` is conditionally rendered (mounted fresh) at the same moment its text content is set, rather than always present with content updated afterward — why does the difference matter mechanically, not just as a rule to follow?

## Gotchas

**Mounting the live-region element and its content in the same render.** Covered above — this is the single most common reason a "correctly" ARIA-tagged live region silently fails to announce anything, and it's invisible in code review because the JSX/ARIA attributes look entirely correct.

**Using `assertive` as the default "to be safe."** Produces a page that talks over itself and the user's own screen reader navigation constantly — the opposite of accessible, even though every individual attribute is technically valid.

**No debounce on a rapidly-changing value (a live counter, a typing-driven result count).** Produces a burst of overlapping, half-cut-off announcements that are worse than no announcement — screen reader users have reported this exact pattern as one of the more actively unpleasant accessibility failures, distinct from simple omission.

**Forgetting `aria-atomic="true"` on a region where only part of the text changes.** Without it, some AT only announces the specific text node that changed, which can produce a bare, context-free announcement (just a new number, with no surrounding "results found" framing) if the implementation updates a child node rather than the whole region's text.

**Announcing routine, non-actionable state changes at the same urgency as failures.** Background sync success, autosave ticks, and similar low-stakes routine events, if announced at all, should be low-priority and infrequent — treating every state transition as equally announcement-worthy produces alert fatigue that trains users to stop paying attention, which then also buries the announcements that actually matter.

## Follow-up Questions

**Q (High): Why doesn't a live region announce anything if it's created and populated with content in the same operation?**

Answer: Assistive tech builds its list of "elements to watch for changes" by observing the accessibility tree over time — a live region needs to already exist, in a state the AT has registered as "this is a live region I'm monitoring," before a *subsequent* mutation to its content is what triggers the announcement. If a component's first render conditionally mounts a brand-new `<div aria-live="polite">Success!</div>` only once there's something to say, the AT frequently never had the chance to register that element as a live region *before* its content appeared — from the AT's perspective, an element that already contains "Success!" the first time it's observed isn't a "change" at all, it's just static initial content. The fix is rendering the container empty and present from initial page load, and only ever mutating its text content afterward, which is a genuine, observable DOM mutation on an already-known live region.

The trap: writing ARIA-correct markup (right role, right `aria-live` value) but getting the mount timing wrong — this passes a static accessibility linter (the attributes are all valid) and only fails when actually tested with a live screen reader, which is exactly why this scenario emphasizes verification over attribute-checklist compliance.

---

**Q (High): How do you decide `polite` versus `assertive` for a given update, in general — not just for the three examples given?**

Answer: The test I'd apply: if a sighted user, mid-task, would want to be interrupted right now to notice this — not just eventually see it — it's assertive; if it's fine for them to notice it whenever they next glance at that part of the screen, it's polite. Concretely, that tends to put form-submission errors, session-expiry warnings, and anything blocking further progress in the assertive bucket, and result counts, save confirmations, and background-process status in the polite bucket. The asymmetry in cost matters too: an assertive announcement that turns out to be non-urgent is genuinely disruptive (it interrupts whatever the user's screen reader is currently doing, including mid-sentence); a polite announcement for something slightly urgent is a lesser failure (delayed, not lost) — so when genuinely unsure, I'd default to polite and treat upgrading to assertive as something to justify explicitly, not the reverse.

The trap: picking assertive by default under the reasoning "I want to make sure it's heard" — this optimizes for a single announcement's visibility at the cost of the page's overall trustworthiness, since a page that interrupts constantly gets its interruptions tuned out.

---

**Q (Medium): The design calls for a live progress percentage during a file upload ("14%… 27%… 41%…"). How would you announce that without producing a wall of noise?**

Answer: I would not announce every percentage tick — that's the continuously-changing-value case the debounce/coalesce guidance is specifically for, and even debounced to, say, once every couple of seconds, a stream of raw percentage numbers isn't especially useful information on its own. I'd announce meaningfully at a coarser grain: milestone-based ("25% uploaded," "halfway there," "almost done") rather than every tick, or — often the better answer — just start and end states ("Upload started," "Upload complete") with the fine-grained percentage remaining a purely visual affordance (a progress bar, `role="progressbar"` with `aria-valuenow` kept up to date for AT that supports querying it directly on request), since a screen reader user generally doesn't need a running numeric commentary any more than they need a running commentary of a sighted user's eye movements across a loading bar — they need to know it started, roughly how it's going if they choose to check, and definitively when it's done or if it failed.

The trap: literally translating a visual progress percentage into a 1:1 stream of live-region announcements — the sighted experience of "glancing at a number that's slowly climbing" isn't something a live region can equivalently replicate as a stream of interruptions, and attempting to is a common overcorrection.

---

**Q (Medium): How would you test that your live-region implementation actually works, beyond reading the ARIA attributes in the code?**

Answer: Attribute-level correctness (right role, right `aria-live`, region present before content changes) is verifiable by code review and static analysis, but whether an announcement actually fires, at the right time, with the right text, and without being talked over by something else, is only verifiable by running an actual screen reader (NVDA+Chrome as a baseline, per the reasoning in the [modal](01-accessible-modal-scenario.md) and [combobox](02-accessible-combobox-scenario.md) scenarios) and listening. Specifically worth testing: the initial-mount-before-content-changes ordering; rapid-fire updates actually get debounced rather than queuing up a backlog of stale announcements; an assertive announcement genuinely interrupts ongoing NVDA speech rather than queuing politely behind it; and identical repeated text (e.g., "Saved" twice in a row) either re-announces via a deliberate mechanism or is a known, accepted limitation, not an accidental silent failure.

The trap: relying on an automated accessibility scanner (axe, Lighthouse) as sufficient verification — these tools can confirm the ARIA attributes are valid and present, but cannot verify that an announcement actually fires at the correct runtime moment with correct debounce/timing behavior, which is precisely where the bugs in this scenario tend to live.

---

**Q (Low): Does the visually-hidden CSS technique used to keep the live region off-screen (but present in the accessibility tree) matter — could you just use `display: none`?**

Answer: `display: none` (and `visibility: hidden`) removes an element from the accessibility tree entirely, not just visually — a live region hidden this way is invisible to assistive tech too, defeating the purpose. The region needs to be visually hidden while remaining accessibility-tree-visible, which is what the standard "visually-hidden" utility class pattern achieves: absolute positioning, a 1px clipped size, `overflow: hidden`, and no `display`/`visibility` hiding — keeping the element genuinely present and readable by AT while invisible on screen.

The trap: reaching for `display: none` or a conditional `{show && <LiveRegion />}` render as the "clean" way to hide an always-present-but-usually-empty element — both approaches either remove it from the accessibility tree or reintroduce the mount-timing bug this scenario is built around.

---

## Self-Assessment

- [ ] Can explain exactly why a live region must exist in the DOM before its content changes, not just state it as a rule
- [ ] Can articulate the polite-vs-assertive decision test and apply it to a new example on the spot
- [ ] Can design a debounce strategy for a rapidly-changing value that still ends on the correct, final settled value
- [ ] Can explain why routine/non-actionable state changes (background sync success) are deliberately under-announced, not an oversight
- [ ] Can name the CSS requirement for "visually hidden but AT-visible" and why `display: none` doesn't satisfy it
- [ ] Can describe how to verify a live-region implementation with an actual screen reader, not just static ARIA-attribute review

---
*Next: Keyboard Trap Bug — Find and Fix — a debugging scenario: given a working-looking component with a genuine, unintentional keyboard trap, diagnose and fix it under time pressure.*
