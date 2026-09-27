# Controlled vs. Uncontrolled Input Bug

## Quick Reference

| Symptom | Mechanism | Fix |
|---|---|---|
| "A component is changing an uncontrolled input to be controlled" warning | `value` prop starts as `undefined`/`null` (uncontrolled), then later becomes a defined string (controlled) | Always initialize the backing state to a defined value (`''`, not `undefined`) so `value` is never `undefined` on first render |
| Input becomes unresponsive / stuck / one keystroke behind | `value` is set from state, but `onChange` doesn't update that same state (or updates a different one) | Ensure every controlled input's `value` and `onChange` operate on the exact same state variable |
| Mixing `value` and `defaultValue` on the same input | React warns and the behavior is undefined/inconsistent — the input doesn't fully commit to either mode | Pick one mode per input, for its entire lifetime; never pass both |
| Controlled input reset unexpectedly on re-render | `value` is derived from a prop/computation that isn't stable, resetting to some default value | Ensure the source of truth for `value` isn't inadvertently recomputed/reset on unrelated re-renders |

## The Scenario

"QA reports a console warning: 'A component is changing an uncontrolled input to be controlled.' They also say that on this same form, if you type quickly, the input sometimes seems to lag a character behind what's on screen. Explain what's happening and fix it — and explain why React cares about this distinction in the first place."

## Clarifying Questions

- **What's the initial value passed to the input's `value` prop, and where does it come from** — is it `useState('')`, `useState()` (no argument, so `undefined`), or derived from a prop/API response that might not have arrived yet on first render (e.g., `value={user?.name}` before `user` has loaded)? This is almost always where the "changing an uncontrolled input to controlled" warning originates — the input starts life with `value={undefined}` and later, once real data exists, receives a defined string.
- **Is the input's `onChange` handler actually updating the same piece of state that's fed back into `value`, or could there be a naming/wiring mismatch** (e.g., `value={formData.name}` but `onChange={e => setDraftName(e.target.value)}`, updating a *different* state variable than the one driving `value`)? The "lags a character behind" symptom often comes from exactly this kind of mismatch, or from an async/debounced state update being read back into `value` before it's actually committed.
- **Does any part of the code pass both `value` and `defaultValue` to the same input, or switch which one is passed conditionally?** This is a distinct, second common cause of "component changing from uncontrolled to controlled (or vice versa)" warnings, structurally different from the `undefined`-initial-value case.
- **Is there a debounce, throttle, or async validation step sitting between the raw `onChange` event and the state update that ultimately feeds back into `value`?** If `value` reflects state that's updated *after* a debounce delay, but the actual DOM input element's native, immediate value has already changed, there's a brief window where the DOM's real value and React's `value` prop disagree — worth understanding whether this is contributing to the "lags a character" symptom, and if so, whether that's actually the debounce working as intended versus something unintentionally reading a stale value.
- **Is this specific to one input, or does the pattern of the bug recur across several inputs in the same form** (suggesting a shared form-state-management utility or pattern is the actual root cause, not one isolated typo)?

## Approach & Trade-offs

**Controlled vs. uncontrolled is a binary React expects an input to commit to for its whole lifetime.** A "controlled" input is one whose displayed value is driven entirely by React state, passed via the `value` prop, with the DOM element's actual value kept in sync with that state on every render — the browser's native input behavior is effectively overridden; typing doesn't "just work" by itself, it works because `onChange` fires, updates state, and the re-render feeds the new state back into `value`, which the browser then displays. An "uncontrolled" input, by contrast, manages its own internal value via normal DOM/browser behavior, with React only reading it out when needed (via a `ref`, or on form submission) — `defaultValue` sets the *initial* value only, and after that the DOM owns it. React needs to know, from the very first render, which mode a given `<input>` is operating in, because the internal wiring (whether React is actively forcing the DOM value on every render or leaving it alone) differs — and it determines this specifically by checking whether `value` is `undefined` (uncontrolled) or defined (controlled) on that *first* render.

**Why switching modes mid-lifetime produces the warning, and why it's not just a stylistic nag.** If `value` starts as `undefined` (React treats the input as uncontrolled, and does not take over managing its DOM value) and later becomes a defined string once some data loads, React is being asked to retroactively start controlling an input it had already decided, at mount, was uncontrolled — the practical consequence can include the input's displayed value not immediately reflecting the newly-controlled value the way an input that was controlled from the start would, since React's internal input-value-syncing logic behaves differently depending on which mode it believed it was in from the start. The warning exists because this is very rarely intentional — it's almost always a symptom of a value that's genuinely meant to be controlled from the beginning, but happens to be `undefined` momentarily due to data not having loaded yet, or a default argument being omitted from a `useState()` call.

**The "lags a character behind" symptom, and why it points at a wiring/timing issue rather than "React is slow."** A controlled input showing text one keystroke behind what was actually typed is the classic signature of `value` and `onChange` not operating on the same, synchronously-updated piece of state — either because `onChange` updates a *different* state variable than the one `value` reads (a copy-paste/refactor mistake, more common than it sounds in forms with many similarly-named fields), or because something between the keystroke and the state update introduces a delay (an unnecessary `setTimeout`, an async validation awaited before calling `setState`, or state that's derived through a chain of effects rather than updated directly in the `onChange` handler). Because a controlled input's displayed value is *only* what React's `value` prop says it is (the browser isn't allowed to just show whatever was typed, once controlled), any delay or mismatch in getting the new value back into that prop shows up immediately and visibly as lag — this is a case where a controlled input is strictly *less* forgiving of timing mistakes than an uncontrolled one, precisely because it's being actively driven rather than left to the browser's own default (instant) behavior.

**When uncontrolled is actually the better choice, and it's not a compromise.** For a form that's simple, submitted as a whole, doesn't need live validation-as-you-type, and doesn't need any other part of the UI to react to keystrokes as they happen, an uncontrolled input (read via `ref` on submit) avoids the entire class of bug described in this scenario — there's no `value`/`onChange` synchronization to get wrong, because the browser is doing what it already does well. Reaching for controlled inputs by default for every form field, even ones with no actual need for live reactivity, is a common source of exactly this bug category, for no corresponding benefit — the decision should be driven by whether something genuinely needs to observe/react to the value on every keystroke (live validation, a character counter, conditionally showing/hiding other fields), not by habit.

**Never pass both `value` and `defaultValue`.** This is React's explicit, well-documented incompatible-combination — the two props represent mutually exclusive modes, and passing both produces a dev warning and behavior that shouldn't be relied upon (in practice, `value` tends to win and `defaultValue` is ignored after mount, but this isn't a contract worth depending on). If a component needs to support "start with this initial value, but then let it become controlled later" as a genuine product requirement (relatively rare, but real — e.g., an input that's uncontrolled until the user starts editing, then becomes controlled to enforce a format), that transition needs to be modeled deliberately (a local `isEditing` flag switching which mode is active, not silently flipping `value` from `undefined` to defined based on data-loading timing).

## Solution

Reproducing the "uncontrolled to controlled" warning — `value` derived from data that isn't loaded yet:

```tsx
function ProfileForm({ userId }: { userId: string }) {
  const [user, setUser] = useState<{ name: string } | null>(null);

  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]);

  // BUG: `user?.name` is `undefined` on the first several renders (before the fetch resolves),
  // then becomes a defined string once `user` loads — flipping the input from
  // uncontrolled to controlled mid-lifetime.
  return <input value={user?.name} onChange={e => setUser(u => u && { ...u, name: e.target.value })} />;
}
```

Fix — the state always has a defined value, even before real data arrives:

```tsx
function ProfileForm({ userId }: { userId: string }) {
  const [name, setName] = useState(''); // always a defined string — controlled from the very first render

  useEffect(() => {
    fetchUser(userId).then(user => setName(user.name));
  }, [userId]);

  return <input value={name} onChange={e => setName(e.target.value)} />;
}
```

Reproducing the "lags a character behind" symptom — `onChange` updates a different state than `value` reads:

```tsx
function SearchField() {
  const [query, setQuery] = useState('');
  const [pendingQuery, setPendingQuery] = useState(''); // BUG: a second, disconnected piece of state

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setPendingQuery(e.target.value); // updates the WRONG state variable
    debounceSearch(e.target.value);
  };

  return <input value={query} onChange={handleChange} />; // `value` reads `query`, which never gets updated here
}
```

`query` never changes (nothing ever calls `setQuery`), so the input's displayed value is permanently stuck at `''` regardless of typing — a more severe version of "lags behind" (in this exact contrived example it doesn't update at all, but the same category of mismatch, with an intermediate state update somewhere in the chain, produces the "one keystroke behind" symptom described in the prompt).

Fix — `value` and `onChange` operate on the same state; debouncing happens downstream of the source of truth, not instead of updating it:

```tsx
function SearchField() {
  const [query, setQuery] = useState('');

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value); // the input's displayed value updates immediately, every keystroke
    debounceSearch(e.target.value); // the *search request* is debounced — the *display* is not
  };

  return <input value={query} onChange={handleChange} />;
}
```

The distinction that matters: the input's own displayed value must update synchronously with every keystroke (no debounce in that path at all), while a *downstream effect of* that value (firing a search request) can and should be debounced separately — conflating "debounce the side effect" with "debounce what the input displays" is what produces the lag.

> **Check yourself:** In the fixed `ProfileForm` example, if `fetchUser` fails and the `.then` callback never runs, does the input remain correctly controlled? What would happen if the initial state were `useState()` (no argument) instead of `useState('')`, and why does that one-character difference matter so much here?

## Root Cause

Both symptoms trace back to the same underlying rule: a controlled input's `value` must be a defined value from the very first render onward, and must be kept synchronized, on every render, with the exact state that its own `onChange` updates. Violating either half of that rule — an initially-undefined `value`, or a `value`/`onChange` pair pointing at different state — produces one of these two symptoms.

## Gotchas

**Using `value={someValue ?? ''}` as a band-aid without fixing the underlying state's initialization.** This does silence the warning (since `value` is now always defined), but it can mask a state initialization bug that has other consequences elsewhere (e.g., other code reading the same underlying state and getting a genuinely `undefined`/`null` value unexpectedly) — the more complete fix is making sure the state itself has a sensible default, not just papering over the symptom at the JSX call site.

**Assuming the warning only matters cosmetically, and the input "still basically works."** Depending on exactly which render the mode-flip happens on and what else is going on, the transition can produce visibly incorrect behavior (a flash of the wrong value, or the input briefly not accepting a keystroke) — treating this purely as console noise to be dismissed underestimates it.

**Fixing one occurrence in a form with many nearly-identical fields, without checking siblings for the identical mistake.** If the root cause is a state-management pattern used across a whole form (e.g., a `useState<UserData | null>(null)` object destructured into many field values, all of which are `undefined` until it loads), every field wired the same way likely has the identical bug — worth checking the whole form, not just the field the console warning happened to point at first.

**Introducing a debounce by delaying the state update that also drives `value`**, rather than debouncing only the *side effect* triggered by that state (an API call). This is the structural mistake behind the "lags behind" symptom in its more subtle forms — anywhere the input's own displayed value is made to wait on something (a debounce timer, an async validation, a round-trip to a store) before being reflected back into `value`.

## Follow-up Questions

**Q (High): Explain, mechanically, why React determines whether an input is controlled or uncontrolled specifically by checking the `value` prop's definedness on the *first* render, and why it can't just adapt if `value` starts as `undefined` and later becomes defined.**

Answer: React's input-handling logic branches based on whether `value` is present at all — if it's `undefined` on mount, React registers the input as uncontrolled and lets the DOM manage its own value natively from that point on (React doesn't attempt to sync a DOM property to a prop that doesn't exist); if `value` is defined on mount, React registers it as controlled and actively sets/syncs the DOM element's value to match the `value` prop on every render, which also means intercepting and overriding what the browser would otherwise do natively in response to typing. This decision is effectively made once, at initialization, because the two modes involve fundamentally different behavior being wired up (whether React's reconciliation actively writes to the DOM node's value property every render, or leaves it alone) — React *can* technically detect a change from `undefined` to defined on a later render (which is exactly what triggers the dev warning), but by the time that happens, the component has already been behaving as uncontrolled for however many renders preceded it, and the transition to actively-managed behavior happening mid-stream is exactly the scenario the warning exists to flag as almost certainly unintentional.

The trap: describing this as "React gets confused" in vague terms — the actual mechanism (an initialization-time branch based on prop definedness, not a continuously-re-evaluated mode) is what explains *why* this specific transition (undefined → defined) is flagged, while other transitions (one defined string to another defined string) are completely normal and expected of any working controlled input.

---

**Q (High): A junior engineer suggests fixing the "lags behind" bug by wrapping the `onChange` handler's `setState` call in `flushSync` to force it to apply synchronously. Is that the right fix? Why or why not?**

Answer: No — `flushSync` forces React to apply a given state update and re-render synchronously rather than allowing it to be batched with other updates, which addresses a *timing/batching* concern, not the actual bug in this scenario, which is that `value` and `onChange` are wired to two *different* state variables (or the update path includes debounce/async logic that delays the state actually driving `value`). Forcing a `setState` call to flush synchronously doesn't fix a mismatch between which piece of state is being read versus written — if `onChange` updates `pendingQuery` while `value` reads `query`, making `setPendingQuery` synchronous doesn't make `query` reflect the new value; `query` still never gets updated at all. `flushSync` is the correct tool for a narrower, different problem (needing a DOM measurement or focus behavior to reflect a state update immediately, before browser paint, within the same event handler) — reaching for it here treats a symptom (perceived "delay") without addressing the actual structural mismatch causing it.

The trap: pattern-matching "input feels laggy" to "must be a batching/timing problem" and reaching for a scheduling API (`flushSync`, or unnecessarily wrapping things in `useTransition`) without first confirming, via the actual code, whether the bug is really about timing at all versus a straightforward wiring mistake — the fix should be diagnosed from the actual data flow, not guessed at from the symptom's vibe.

---

**Q (High): Is it ever correct for an input to start uncontrolled and later become controlled? If so, how would you implement that transition without triggering the warning?**

Answer: Yes, in specific, deliberate cases — a documented example is an input that behaves natively (uncontrolled) while the user is freely typing, then "locks in" and becomes controlled once some condition is met (say, enforcing a specific format only after the user blurs the field, or after a certain character count). The way to implement this without triggering React's warning is to not let `value` itself silently flip from `undefined` to defined based on incidental timing — instead, explicitly and intentionally switch which *mode* the input renders in, typically by conditionally rendering either an uncontrolled version (`<input defaultValue={...} ref={inputRef} />`, no `value` prop at all) or a controlled version (`<input value={controlledValue} onChange={...} />`) based on an explicit boolean state flag, often paired with a changing `key` to force React to treat the swap as a clean mount of a genuinely different input configuration rather than attempting to reconcile one input element across the mode change. This makes the controlled/uncontrolled transition an intentional, modeled state transition rather than an accidental byproduct of data loading, which is what the warning is actually trying to catch.

The trap: proposing to just special-case around the warning (e.g., always passing *some* value, even a placeholder sentinel, and pretending the input was "always controlled") without recognizing that a genuine mode-switch use case exists and has a real, sanctioned implementation pattern (explicit conditional rendering with a key), rather than treating "always controlled from render one" as the only correct answer in every case.

---

**Q (Medium): Why does a controlled input need `onChange` at all — why can't you just pass `value` and let the user type normally?**

Answer: Because a controlled input's displayed value is driven entirely by the `value` prop and React actively enforces that on every render — without an `onChange` handler that calls `setState` with the new typed value, the state driving `value` never changes, so on the very next render (which does still occur, or even without an explicit state-triggered render, the browser's own attempt to update the field is overridden back to the unchanged `value` prop) the input snaps back to whatever `value` currently holds, effectively preventing the user from typing anything at all — this is precisely why React logs a specific warning ("You provided a `value` prop to a form field without an `onChange` handler... this will render a read-only field") when `value` is passed without `onChange`, since that combination, unless `readOnly` is explicitly intended, almost certainly indicates a missing handler rather than a deliberate read-only field.

The trap: describing the missing-`onChange` failure mode vaguely as "typing wouldn't work" without being able to explain the actual mechanism (React re-asserting the unchanged `value` prop against the DOM, effectively reverting any attempted native edit) — that mechanism is what directly explains *why* it manifests as a read-only-feeling field rather than, say, a crash or a warning-free silently-broken input.

---

**Q (Medium): How would you unit-test a form field to catch this class of bug (controlled/uncontrolled mismatch, or `value`/`onChange` wired to different state) before it reaches QA?**

Answer: A React Testing Library test that renders the form, simulates typing into the field via `userEvent.type`, and asserts that the input's displayed value (`input.value` / the rendered DOM) matches what was typed after each keystroke (or at least after the full string) would directly catch the "lags behind"/"doesn't update at all" symptom, since it exercises the actual `value`/`onChange` wiring end-to-end rather than testing either prop in isolation. Catching the "starts uncontrolled, becomes controlled" warning specifically is less naturally covered by a behavioral assertion and more reliably caught by a console-error/warning assertion in the test setup — many test configurations already fail a test if React logs a warning during it (via a mocked/spied `console.error` that fails the test on any call matching React's warning text), which is precisely the mechanism that would flag this class of bug automatically in CI without a developer needing to manually inspect the console, provided the test actually renders the component through the state transition that would trigger the flip (e.g., rendering before an async `user` fetch resolves, then resolving it within the test, rather than only rendering with already-resolved data).

The trap: proposing only a "does typing work" behavioral test — that catches the `value`/`onChange` mismatch bug but, on its own, wouldn't necessarily surface the uncontrolled-to-controlled console warning unless the test setup is specifically configured to fail on console warnings/errors and the test actually exercises the pre-data-loaded render state, not just the fully-loaded one.

---

## Self-Assessment

- [ ] Can explain precisely why React determines controlled/uncontrolled mode based on `value`'s definedness at first render, not on an ongoing basis
- [ ] Can diagnose the "lags a character behind" symptom as a `value`/`onChange` state mismatch or a delay injected into the display path, distinct from a genuine performance problem
- [ ] Can explain why debouncing a side effect (an API call) and debouncing what an input displays are two different things that shouldn't be conflated
- [ ] Can implement a deliberate, non-warning-triggering uncontrolled-to-controlled transition when that's a genuine product requirement
- [ ] Can explain the mechanism behind the "missing onChange handler" read-only-field warning
- [ ] Can describe how to catch this bug class in an automated test, including via a console-warning assertion

---
*Next: Infinite Render Loop — the next step in severity from unwanted extra renders: a render loop that never terminates, usually from a `setState` call unconditionally triggered inside the render body or an effect with a self-triggering dependency.*
