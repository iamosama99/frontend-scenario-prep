# Multi-step Form Wizard With Validation

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| State shape | One flat form-state object for the whole wizard, not per-step slices | A single source of truth avoids sync bugs when a field on step 1 affects validation on step 3, or the user navigates back and forth |
| Validation timing | Validate the current step when leaving it (gates "Next"); validate everything again at final submit | Catches mistakes early without the "I filled out five screens before it told me step 1 was wrong" failure |
| Preserving state on back/forward | Steps read from and write to the shared state object; steps are never re-mounted/reset when revisited | Re-mounting a step component from scratch wipes whatever the user already entered there |
| Step indicator | Reflects both "current step" and "which steps are already valid," independently | Users need to see progress *and* know which earlier steps still need fixing |
| Async validation (e.g. username availability) | Debounce the check, tag requests with a sequence number/AbortController, ignore stale responses | An in-flight check for an old value must not overwrite the UI after a newer check has already resolved |
| Persistence across refresh | `sessionStorage` (or a URL step param) for step index + non-sensitive field values; never persist passwords/secrets | Refresh-survives-progress is a real UX win, but persisting secrets to storage is a real security problem |

## The Scenario

"We need a multi-step signup wizard — three or four screens, collecting things like account info, profile details, and preferences, with a review step at the end. Users need to be able to go back and forward without losing what they've entered. Some fields need async validation, like checking if a username is taken. Build the state management and validation flow — the visual design doesn't matter much, focus on getting the data flow right."

## Clarifying Questions

- **Is validation per-step (gates the Next button) or only at final submit?** This is the central UX decision. I'd default to per-step validation — check the current step's fields when the user tries to advance, block "Next" and show errors if invalid — because the alternative (only validating at the very end) means a user can fill out four screens and only then discover step 1 had a typo, which is a materially worse experience and a common source of drop-off in real signup flows. I'd confirm because there's a legitimate counter-case: if steps are highly interdependent or optional/skippable, strict per-step gating can feel punitive.
- **Should the wizard state survive a page refresh?** If yes, that requires a persistence layer (`sessionStorage` is the natural default over `localStorage`, since wizard progress is normally session-scoped, not something you'd want lingering across browser restarts weeks later) and a decision about exactly what's safe to persist — this rules out storing raw password fields even temporarily.
- **Can the user jump directly to a step (e.g., clicking step 3 in the indicator) or only move one step at a time via Next/Back?** This affects whether the step indicator's steps are clickable and whether jumping ahead should be blocked if earlier steps aren't valid yet — I'd default to allowing backward jumps freely (since revisiting completed steps is harmless) but blocking forward jumps past the first invalid/incomplete step.
- **For the async username-availability check — what should the UI do while the check is in flight, and what if the user keeps typing?** This needs an explicit answer: showing a stale "available" checkmark while a newer keystroke's check is still pending is a real, embarrassing bug class (user sees "available", submits, backend rejects because the check was for an older, now-superseded value). Debouncing alone isn't sufient — I'd also confirm whether "Next" should be blocked entirely while an async check is pending, versus allowed with a warning.
- **Does going back to an earlier step and changing a field need to invalidate anything already entered on later steps** (e.g., changing "country" on step 1 might invalidate a "state/province" selection made on step 2)? If cross-step dependencies exist, later steps can't just be validated once and forgotten — they need to be re-validated (or at least flagged as potentially stale) whenever an upstream field they depend on changes.

## Approach & Trade-offs

**State shape: one flat object, not per-step slices.** I'd model the entire wizard's data as a single object — `{ email, password, username, displayName, plan, ... }` — rather than `{ step1: {...}, step2: {...}, step3: {...} }`. The flat shape avoids an entire category of sync bugs: if a field conceptually belongs to "step 2" but a validation rule on "step 3" needs to read it (the cross-step-dependency case above), a per-step-sliced state forces either reaching across slices (which defeats the point of slicing) or duplicating the value into both slices (which immediately raises "which copy is the source of truth when they disagree" — exactly the bug class a single object avoids by construction). A flat object also makes "what does final submit send to the backend" trivial — it's just the object, no merging step required. The one place per-step organization still matters is which *fields* each step's UI renders and validates, which is a UI-layer concern (a map of `step -> field names` used to know what to validate on "Next"), not a reason to fragment the state itself.

**Per-step validation gating "Next," plus a full re-validation at final submit.** Validating the current step's fields when the user attempts to leave it (clicks Next, or attempts to jump ahead via the step indicator) catches mistakes at the point they were made, when the context is freshest for the user to fix them — this is both better UX and reduces backend load from doomed submissions. But per-step validation alone isn't sufficient at submit time, for two reasons: cross-step dependent fields might have become invalid due to a later change to an earlier field (the country/state example) without the dependent step being re-visited to re-trigger its own validation; and a user could, depending on how forward-navigation is gated, reach the final step through a path that skipped a step's validation (e.g., browser back/forward button navigation, or a bug in the step-jump guard). A full validation pass immediately before actually submitting is cheap insurance against both, and it's the same validation functions being re-run, not new logic.

**Never re-mounting steps on navigation.** The wizard's step components should be more like "views over a shared state object" than independent, self-contained forms with their own local state — a step reads its fields' current values from the shared state object on render and writes back to it on change, but it does not own an internal copy of "what the user has typed so far" that gets thrown away and recreated each time the step is shown. If steps are naively swapped by conditionally mounting/unmounting a fresh component per step index, any step-local state (an uncommitted keystroke, a focus position, a collapsed/expanded UI sub-section within that step) resets to its initial value every time the user navigates back to it — the shared-state-object approach sidesteps this because there's nothing step-local to lose in the first place; the "recreated" step component just re-reads the same shared values it left behind.

**Step indicator reflecting two independent pieces of information.** "Which step is currently active" and "which steps are already valid/complete" are orthogonal — a user could be sitting on step 3 while step 1 is valid, step 2 is valid, and step 4/5 are simply not yet visited (neither valid nor invalid, just unknown). I'd track step validity as its own piece of state (a `Map<stepIndex, boolean | null>`, where `null` means "not yet validated/visited") separate from `currentStepIndex`, and render the indicator from both: current step gets a distinct visual treatment from valid-and-complete steps, which get a distinct treatment from not-yet-visited steps, which in turn should look different from a step the user visited and left in an *invalid* state (a real, useful fourth state — "you've been here, and it's currently wrong").

**Async validation without blocking the UI or racing a stale check.** The same race-condition shape as the combobox scenario applies directly here: as the user types a username, debounce the availability check (don't fire one request per keystroke), and guard against a slow response to an old value overwriting a fast response to a newer one — tag each outgoing check with a sequence number (or use `AbortController` to cancel the previous in-flight request outright) and only apply a response if it corresponds to the latest request sent. Separately, I wouldn't block all UI interaction while a check is pending — the rest of the form should stay usable — but I would block *advancing past this specific step* until the pending check resolves, showing a clear "checking availability…" state on the field itself rather than silently letting the user proceed with an unconfirmed username.

**What survives a refresh, and what explicitly must not.** Persisting `currentStepIndex` plus the shared form-state object to `sessionStorage` (scoped to the tab/session, cleared when the tab closes, unlike `localStorage` which would persist indefinitely) lets a user recover from an accidental refresh without losing everything — a meaningful real-world UX win for any form long enough to be called a "wizard." But this persistence layer needs an explicit exclusion list: raw password fields (and anything else genuinely sensitive — SSNs, payment details) should never be written to `sessionStorage` even transiently, since browser storage isn't encrypted at rest and is readable by any script that can run in that origin (including, in the worst case, an XSS payload) — the password field should be excluded from the persisted snapshot and simply left for the user to re-type if they refresh mid-flow, which is an acceptable, standard trade-off.

## Solution

The shared state object plus per-step field ownership and validators, built incrementally:

```javascript
const wizardState = {
  data: { email: '', username: '', password: '', displayName: '', plan: 'free' },
  currentStep: 0,
  stepValidity: new Map(), // stepIndex -> true | false | null (null = not yet visited)
};

const steps = [
  {
    name: 'account',
    fields: ['email', 'username', 'password'],
    validate: (data) => {
      const errors = {};
      if (!/\S+@\S+\.\S+/.test(data.email)) errors.email = 'Enter a valid email';
      if (data.password.length < 8) errors.password = 'Password must be at least 8 characters';
      // username's own availability check is async — handled separately, below
      return errors;
    },
  },
  {
    name: 'profile',
    fields: ['displayName'],
    validate: (data) => (data.displayName.trim() ? {} : { displayName: 'Display name is required' }),
  },
  {
    name: 'preferences',
    fields: ['plan'],
    validate: () => ({}), // always valid — has a default
  },
];
```

Navigation: validating the current step before allowing "Next," and re-validating everything at submit:

```javascript
function validateStep(index) {
  const errors = steps[index].validate(wizardState.data);
  const isValid = Object.keys(errors).length === 0;
  wizardState.stepValidity.set(index, isValid);
  return { isValid, errors };
}

function goNext() {
  const { isValid, errors } = validateStep(wizardState.currentStep);
  if (!isValid) {
    renderErrors(wizardState.currentStep, errors); // block advancing, show why
    return;
  }
  if (wizardState.currentStep < steps.length - 1) {
    wizardState.currentStep++;
    persist();
    render();
  }
}

function goBack() {
  // No re-validation gate going backward — revisiting a completed step is always allowed.
  if (wizardState.currentStep > 0) {
    wizardState.currentStep--;
    render();
  }
}

function goToStep(targetIndex) {
  // Allow free jumps backward; block forward jumps past the first not-yet-valid step.
  const firstInvalid = steps.findIndex((_, i) => wizardState.stepValidity.get(i) !== true);
  if (targetIndex <= wizardState.currentStep || targetIndex <= firstInvalid) {
    wizardState.currentStep = targetIndex;
    render();
  }
}

function submit() {
  // Full re-validation immediately before submit — catches cross-step drift
  // and any step reached without going through goNext()'s gate.
  const allErrors = steps.map((_, i) => validateStep(i));
  const firstFailing = allErrors.findIndex((r) => !r.isValid);
  if (firstFailing !== -1) {
    wizardState.currentStep = firstFailing;
    renderErrors(firstFailing, allErrors[firstFailing].errors);
    render();
    return;
  }
  sendToBackend(wizardState.data);
}
```

Async username-availability check with a stale-response guard:

```javascript
let usernameCheckSeq = 0;

const debouncedCheckUsername = debounce(async (username) => {
  const thisCheckSeq = ++usernameCheckSeq;
  setFieldStatus('username', 'checking');

  const isAvailable = await checkUsernameAvailability(username); // API call

  if (thisCheckSeq !== usernameCheckSeq) return; // a newer check superseded this one — discard
  setFieldStatus('username', isAvailable ? 'available' : 'taken');
}, 400);

usernameInput.addEventListener('input', (e) => {
  wizardState.data.username = e.target.value;
  setFieldStatus('username', 'pending'); // immediate feedback that a check is queued
  debouncedCheckUsername(e.target.value);
});
```

Persistence, explicitly excluding the password:

```javascript
function persist() {
  const { password, ...safeData } = wizardState.data; // never persist secrets
  sessionStorage.setItem('wizard', JSON.stringify({
    data: safeData,
    currentStep: wizardState.currentStep,
  }));
}

function restore() {
  const saved = sessionStorage.getItem('wizard');
  if (!saved) return;
  const { data, currentStep } = JSON.parse(saved);
  Object.assign(wizardState.data, data); // password stays at its initial '' value
  wizardState.currentStep = currentStep;
}
```

> **Check yourself:** If a user is on step 3, goes back to step 1, changes their email, and never revisits step 1's "Next" button again before eventually clicking final Submit — does the email get re-validated? Trace through which function call makes that happen.

## Gotchas

**Per-step-sliced state instead of one flat object.** This looks tidy at first (`step1State`, `step2State`, ...) but breaks the moment any validation or UI logic needs to read a field that conceptually "belongs" to a different step — candidates either duplicate the field across slices (creating a two-sources-of-truth bug) or add ad hoc cross-slice reaching that defeats the point of slicing in the first place.

**Only validating at final submit.** Technically simpler to build, but it's the exact failure mode the scenario is designed to probe: a user who fills out four screens only to be told "step 1 is wrong" at the very end is a materially worse experience, and it's the kind of shortcut an interviewer expects a senior candidate to flag as a trade-off even if time constraints mean it's not fully implemented in the room.

**Re-mounting a step component fresh on every navigation.** If a step's local UI is torn down and rebuilt from scratch every time it's revisited (rather than persistently reading from shared state), any state that lives only in that component instance — not yet committed to the shared object, a mid-edit keystroke, scroll position — is silently lost on back/forward navigation.

**Stale async validation overwriting a newer check's result.** Without a sequence-number or `AbortController` guard, a slow "is this username taken" response for an earlier keystroke can resolve after a faster response for a later keystroke, showing the user a stale (and wrong) availability status right before they submit.

**Persisting the password (or other secrets) to `sessionStorage`.** A naive "just persist the whole state object" implementation captures the password field along with everything else — even temporarily, this writes a plaintext secret to browser storage that isn't encrypted at rest and is readable by any script running in that origin, a real security regression, not just a style nitpick.

**Not re-validating cross-step-dependent fields when an upstream field changes.** If step 2's "state/province" was validated once when step 1's "country" was, say, "USA," and the user later goes back and changes country to "Canada" without ever re-visiting step 2, a naive implementation that only invalidates on explicit re-visit leaves step 2 marked "valid" against data that no longer makes sense — the final pre-submit full re-validation pass is the safety net for exactly this, but it's worth calling out explicitly since it's easy to design a wizard where that safety net is the *only* thing catching it.

## Follow-up Questions

**Q (High): Why use a single flat state object for the whole wizard instead of state scoped to each step?**

Answer: A flat object makes the wizard's data have exactly one source of truth regardless of which step is currently rendered — any step's validation logic, or the final submit payload, can read any field directly with no merging or cross-slice reaching required. Per-step-scoped state (`step1State`, `step2State`, ...) works fine as long as every field's relevance is strictly contained within its own step, but breaks down the moment a validation rule or a later step's rendering needs to read a field that "belongs" to an earlier step — the two available fixes (duplicate the field into both slices, or add ad hoc cross-slice access) either introduce a two-copies-of-the-truth bug or defeat the purpose of slicing in the first place. A flat object also trivializes building the final submit payload — it's already shaped as exactly what needs to be sent, no assembly step required.

The trap: presenting per-step slices as a reasonable, purely organizational choice with no real downside — the downside is concrete and specific (cross-step dependent validation, easy divergence between duplicated copies of a field), not just a vague "it's less clean."

---

**Q (High): Should validation happen per-step (gating Next) or only at final submit — and what does a senior candidate say about the trade-off, not just pick one?**

Answer: Per-step validation (checking the current step's fields when the user tries to advance, blocking Next and surfacing errors if invalid) gives the user feedback at the point the mistake was made, when it's cheapest and least frustrating to fix, and avoids wasted effort on later steps built on top of invalid earlier data. Submit-only validation is simpler to implement (one validation pass, one place) but risks a materially worse experience — a user who completes every screen only to be told the very first field was wrong has to backtrack through everything they just did. The trade-off runs the other way too, though: strict per-step gating can feel punitive for wizards with optional or loosely-coupled steps, or where a field's validity genuinely can't be determined until a later step provides context — a senior answer names both directions rather than treating per-step validation as an unconditionally correct default, and typically lands on "validate per-step to gate Next, but always re-run full validation immediately before actual submission" as the balance that covers both the UX and the cross-step-drift correctness concern.

The trap: picking one approach without naming why the other is sometimes preferable — interviewers are specifically listening for the trade-off narration, not just a confident single answer.

---

**Q (High): How do you prevent a stale async validation response (e.g., username availability) from overwriting the result of a newer check?**

Answer: Tag every outgoing check with a monotonically increasing sequence number (or use `AbortController` to actually cancel the previous in-flight request when a new one starts) captured at the moment the request is sent; when a response comes back, compare its associated sequence number against the current/latest one — only apply the response to the UI if it matches the most recent request issued, discarding it silently otherwise. Debouncing the input (so a fast typist doesn't fire one check per keystroke) reduces how often this race actually manifests, but doesn't eliminate it on its own, since network response times aren't guaranteed to preserve request order — a request sent later can still resolve before one sent earlier under real-world conditions.

The trap: treating debounce alone as sufficient — debounce controls how often requests are *sent*, not the order in which their *responses* arrive; those are two separate problems, and only the second one is what actually causes the stale-overwrite bug.

---

**Q (Medium): How should the step/progress indicator represent a step's state, and how many distinct states does it actually need?**

Answer: At minimum four, and they're genuinely independent: not-yet-visited (never validated, unknown status), currently active (the step being shown right now — orthogonal to validity), valid/complete (visited and passed its own validation), and invalid (visited, but currently failing its own validation — distinct from "not yet visited" because the user has already engaged with it and it's currently wrong). Conflating "not yet visited" with "invalid" is a common mistake — a step nobody has touched yet isn't wrong, it's simply unknown, and showing it with the same visual treatment as a step the user actively got wrong is misleading. The current-step highlight is independent of all of the above — a user can be actively on a step that's also, at that moment, in an invalid state (they just triggered a validation error and haven't fixed it yet).

The trap: collapsing this down to a simpler two-state model (done vs. not-done) — that loses the ability to distinguish "haven't gotten there yet" from "went there and it's currently broken," which is exactly the distinction a real step indicator needs to communicate.

---

**Q (Medium): What should and shouldn't be persisted to survive a page refresh, and why `sessionStorage` over `localStorage`?**

Answer: Persist the current step index and the shared form-data object, minus any genuinely sensitive fields — passwords, payment details, SSNs, anything similar — which should be explicitly excluded from the persisted snapshot even though the rest of the object is captured wholesale; a refreshed user simply re-types the excluded fields, which is an acceptable, standard trade-off against not writing plaintext secrets into browser storage. `sessionStorage` is the right default over `localStorage` because wizard progress is normally session-scoped in the user's mental model — closing the tab (or browser) and coming back days later to find a half-finished signup form still populated is more surprising than helpful, whereas surviving an accidental mid-session refresh or back-button tap is the actual, valuable use case `sessionStorage`'s lifetime matches.

The trap: persisting the entire state object indiscriminately, including the password field, "because it's simpler" — this is a real security issue (browser storage is unencrypted and readable by any script in that origin, including an XSS payload), not just a minor code-quality nitpick, and a candidate who doesn't proactively exclude it is missing something interviewers specifically probe for in this scenario.

---

**Q (Medium): If a user goes back and changes an earlier field that a later step's validation depends on, how do you make sure that later step gets re-validated?**

Answer: Two complementary mechanisms: first, if the dependency is explicit and known (e.g., "state/province" options depend on "country"), the dependent step's validator can be re-run reactively whenever its upstream field changes, not only when the user explicitly navigates to and leaves that step — this requires the validation-triggering logic to watch for changes to specific shared-state fields, not just "the user clicked Next on this particular step." Second, as a safety net regardless of whether every dependency was explicitly wired up, running a full re-validation pass over every step immediately before final submit catches anything the reactive wiring missed, since it re-runs every step's validator fresh against the current, possibly-changed state rather than trusting a validity flag set earlier in the session.

The trap: relying solely on "the user will naturally revisit step 2 after changing step 1" — there's no guarantee of that; a user can go back, make a change, and jump straight to Submit from a later step without ever revisiting the step whose validity that change might have affected, which is exactly the gap the final pre-submit re-validation pass exists to close.

---

**Q (Low): How would you handle a wizard where a step's very presence depends on an earlier answer (e.g., a "business account" checkbox on step 1 adds an extra "company details" step)?**

Answer: Model `steps` as a function of the current state rather than a fixed array — compute the effective step list (and therefore total step count, and the step indicator's rendering) from `wizardState.data` each time it's needed, so toggling the upstream field immediately changes which steps exist going forward. Navigation logic (`goNext`/`goBack`/`goToStep`) then operates against this dynamically-computed list rather than a static one, and per-step validity tracking (the `Map<stepIndex, ...>`) needs to be keyed by something stable across a changing step count — a step's own identifier (`'companyDetails'`) rather than a raw numeric index that would silently point at a different step after the list changes shape.

The trap: keying step validity or navigation purely by numeric array index in a wizard whose step list can change shape — inserting or removing a step shifts every subsequent index, silently misattributing previously-recorded validity/progress to the wrong step.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can justify a single flat state object over per-step state slices with a concrete cross-step-dependency example
- [ ] Can implement per-step validation gating "Next" plus a full re-validation pass immediately before submit
- [ ] Can explain why steps must read/write shared state rather than being re-mounted with their own local state
- [ ] Can design a step indicator with at least four independent states (not-visited / current / valid / invalid)
- [ ] Can implement a stale-response guard (sequence number or `AbortController`) for an async field-level check
- [ ] Can state what belongs in persisted wizard state and explicitly name what must be excluded, and why

---
*Next: Data Table — Sort, Filter, Paginate — from managing state across steps to managing state across simultaneous derived views (sort + filter + page) of one dataset.*
