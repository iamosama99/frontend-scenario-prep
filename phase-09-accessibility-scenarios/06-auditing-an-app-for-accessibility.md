# Auditing an Existing App for Accessibility

## Quick Reference

| Pass | Tool/Method | Catches | Misses |
|---|---|---|---|
| Automated scan | axe DevTools / Lighthouse | Missing alt text, invalid ARIA, contrast failures, missing form labels — roughly 30-40% of real issues | Focus order, keyboard traps, live-region behavior, whether an interaction actually makes sense to a screen reader user |
| Keyboard-only pass | Unplug the mouse, Tab through every flow | Unreachable controls, missing focus indicators, keyboard traps, illogical tab order | Whether the *content* being reached is announced meaningfully |
| Screen reader pass | NVDA+Chrome (or VoiceOver+Safari) through the same flows | Missing/wrong accessible names, unannounced state changes, confusing reading order | Nothing structural — this is the closest to real user experience, but it's slow and requires deliberate practice to do well |
| Zoom/reflow pass | Browser zoom to 200%, check for clipped/overlapping content | Fixed-width layouts that break, horizontal scroll at 400% (1.4.10) | Not itself a screen-reader or keyboard issue — a separate axis |

## The Scenario

"We're about to sign a big enterprise customer who's requiring a VPAT and WCAG 2.1 AA conformance. You've got two hours to audit the checkout flow — cart, shipping form, payment, confirmation — before we scope the real remediation work. What's your process, and what would you actually produce at the end of two hours?"

## Clarifying Questions

- **Is the goal a defensible, structured artifact (something that could inform an actual VPAT or be handed to legal), or an internal engineering punch list to start fixing things immediately?** These have different shapes — a VPAT-adjacent artifact needs to be organized by WCAG success criterion with pass/fail/partial per criterion; an engineering punch list is better organized by page/component and severity. I'd ask which is actually needed before picking a format, since producing the wrong one wastes a meaningful fraction of the two hours.
- **Is there an existing accessibility baseline (a prior audit, known issues already tracked) or is this a from-scratch first pass?** If prior audit results exist, two hours is much better spent verifying whether known issues are still present and scoping *new* ones than re-discovering everything from zero.
- **Should I audit with my own screen reader/keyboard skill level, or is there someone on the team (or a contracted accessibility specialist) who'll do a deeper follow-up pass regardless of what I find?** This changes how much I invest in an exhaustive screen-reader pass myself versus prioritizing breadth (catching the most severe, most probable issues quickly) and flagging "needs specialist review" for anything genuinely ambiguous, rather than trying to be the final word on nuanced AT behavior in two hours.
- **What assistive tech does the actual target customer's compliance requirement specify, if any?** A VPAT is sometimes scoped against particular AT/browser combinations by the requesting customer — worth checking rather than assuming a generic "NVDA+Chrome" baseline is what's actually being asked for.

## Approach & Trade-offs

**Two hours across four flows is roughly 30 minutes per flow, and I'd spend that budget on a fixed sequence of passes per flow rather than one long, meandering exploration** — automated scan first (fast, catches the "free" issues in minutes so I don't spend manual time rediscovering them), then keyboard-only, then a screen reader pass, then a quick zoom/reflow check. Running them in this order specifically matters: automated tools catch a meaningful chunk of issues in seconds, which means the more expensive, slower manual passes (keyboard, screen reader) can be spent exclusively on what automation structurally can't see, rather than re-verifying things a scanner already flagged.

**I'd deliberately not aim for exhaustive AT-behavior verification with every screen reader — I'd aim for defensible severity-ranked coverage across the whole flow, which is a different and more valuable output under this time constraint.** A perfect, exhaustive audit of the cart page alone that never reaches payment is a worse use of two hours than a somewhat-less-deep pass across all four flows, because a checkout flow's accessibility is only as good as its worst step — a blocking issue on the payment page (where the actual transaction happens) matters more than polish on the cart page, and I wouldn't know payment has a blocker at all if I spent the whole budget perfecting cart.

**I'd triage every finding by a severity × reach framework, not just list issues in the order I found them** — severity being "does this block the user from completing the task at all, or just make it harder/less pleasant," reach being "does this affect every user of this AT, or only an edge case." A missing form label on the credit card field that makes it unidentifiable by a screen reader is high severity (blocks task completion) and high reach (affects every screen reader user attempting checkout) — that's the top of the list. A slightly suboptimal but still announced and usable heading structure is low severity — that's real, worth fixing, but shouldn't compete for attention with a blocker in a two-hour triage output.

**I'd write down what I did NOT have time to check, as explicitly as what I did check** — under a hard time constraint, an audit that silently omits, say, "I didn't have time to test with VoiceOver, only NVDA" or "I didn't test with a screen magnifier" produces false confidence if that gap isn't stated. A defensible audit artifact names its own scope boundaries, especially one that's going to inform legal/compliance decisions (a VPAT) where an unstated gap is a real liability, not just an inconvenience.

## The Process, Step by Step

**Minutes 0–15: automated scan across all four pages, batched.** Run axe DevTools (or Lighthouse's accessibility category) against cart, shipping form, payment, and confirmation, and dump every flagged issue into a running list without yet trying to fix or deeply investigate any of them individually — the goal here is coverage and speed, not depth. This alone typically surfaces missing/duplicate `alt` text, unlabeled form inputs, insufficient contrast, missing landmark regions, and invalid ARIA usage (an attribute on an element it's not valid for) — mechanical, structurally-detectable issues that would otherwise eat manual-testing time to rediscover.

**Minutes 15–75 (~15 min/flow): keyboard-only pass through each flow, end to end, narrating what breaks.** Physically avoid the mouse. For each flow: can every interactive element be reached via Tab, in an order that matches the visual/logical flow? Is there always a visible focus indicator? Does any component (a custom dropdown, a date picker, a "quantity" stepper) trap focus unintentionally, or fail to be reachable at all? Specifically for a checkout flow, this pass usually surfaces the highest-severity findings fastest — a payment field that's genuinely unreachable by keyboard is an immediate, unambiguous blocker, distinct from the more nuanced "is this announced well" questions a screen reader pass answers.

**Minutes 75–105 (~10 min/flow, more targeted): a screen reader pass on the specific components the keyboard pass flagged as risky, not a from-scratch re-walk of everything.** Given the time budget, I would not re-traverse every field of every flow with NVDA from zero — I'd use the keyboard pass's findings to target: any custom component (not a native `<input>`/`<button>`), anything with a dynamic state change (an error appearing after validation, a shipping-cost total updating after selecting a method), and the final "order confirmed" state, since a checkout flow's success confirmation not being announced is a surprisingly common, high-impact miss.

**Minutes 105–120: write up findings, severity-ranked, with an explicit "not covered in this pass" section.** Given the remaining time, I'd prioritize getting the top 5–10 findings clearly documented (specific element, specific WCAG criterion, specific reproduction step, suggested fix direction) over a longer but shallower list — a triage document a team can actually act on beats an exhaustive dump that takes longer to read than to have just found the issues myself.

> **Check yourself:** If you only had one hour instead of two, which of these four passes would you cut or compress first, and why — and how would you justify that trade-off if asked?

## What the Two-Hour Output Actually Looks Like

A findings table, not prose — severity-ranked, one row per issue:

| # | Severity | Flow / Component | WCAG Criterion | Finding | Suggested Fix |
|---|---|---|---|---|---|
| 1 | Blocker | Payment / card number field | 1.3.1, 4.1.2 | Input has no associated `<label>` or `aria-label` — screen reader announces only "edit text," no indication it's the card number field | Add `<label for>` or `aria-label="Card number"` |
| 2 | Blocker | Shipping form / custom "Country" dropdown | 2.1.1 | Not reachable via Tab at all — built with `<div onClick>`, no `tabindex`, no keyboard handler | Rebuild using the [Accessible Combobox](02-accessible-combobox-scenario.md) pattern, or swap for a native `<select>` |
| 3 | High | Cart / quantity stepper | 2.1.2 | Focus becomes trapped inside the stepper's +/- buttons after using them 3+ times — likely the same class of bug as [Keyboard Trap Bug — Find and Fix](04-keyboard-trap-bug-fix.md) | Audit the stepper's keydown handler for the missing-boundary-check pattern |
| 4 | High | Payment / validation errors | 4.1.3 | Inline validation errors appear visually but are never announced — no live region | Add a live region per [Live Region Announcements](03-live-region-async-announcements.md) |
| 5 | Medium | Confirmation page | 4.1.3 | Order success state has no announcement — screen reader user has no confirmation the order went through beyond what's visually on screen | Announce via a polite live region or move focus to the confirmation heading |
| — | *Not covered this pass* | — | — | VoiceOver/Safari not tested (NVDA/Chrome only); zoom/reflow at 400% not tested; no testing with actual customer-specified AT if different from NVDA | Flag for a follow-up pass with the compliance-required AT combination |

## Gotchas

**Treating a clean automated-scanner report as "accessible."** Automated tools catch a meaningful but partial slice of real issues (commonly cited estimates put it around 30-40% of WCAG failures) — a page that passes axe cleanly can still have a completely broken focus order, an unannounced live region, or a keyboard trap, none of which most automated scanners can detect, because they require understanding *interaction and meaning*, not just static markup validity.

**Spending the whole time budget perfecting one flow and never reaching the others.** A checkout process is only as accessible as its worst step — thoroughly auditing cart while never touching payment (where the transaction actually completes) risks missing the highest-stakes blocker in the entire flow.

**Producing a flat, unprioritized list of every issue found.** A findings list with fifteen equally-weighted bullet points forces the reading team to independently re-derive severity before they can act — the audit's value is largely in doing that triage work, not just surfacing raw findings.

**Not stating what wasn't covered.** Especially relevant here given the VPAT/compliance framing — an audit that implicitly reads as "this app is now verified accessible" because it doesn't say otherwise, when really only one AT/browser combination and a fixed time budget were covered, sets up a false sense of completeness that becomes a real liability if surfaced later.

**Confusing "this passed my manual test" with "this is definitely fine" for anything genuinely ambiguous.** Two hours doesn't allow for deep expertise on every edge case — flagging something as "needs specialist review" is a legitimate, honest output, not a cop-out, when a finding is genuinely unclear (e.g., "I'm not certain if this reading order issue is a real problem or just unusual, needs a second opinion").

## Follow-up Questions

**Q (High): Why isn't an automated scanner passing cleanly sufficient to call a page accessible?**

Answer: Automated tools are fundamentally limited to what's mechanically detectable from static markup and computed styles — missing `alt` attributes, insufficient contrast ratios, invalid ARIA attribute usage, unlabeled form fields, and similar structural checks. They cannot evaluate anything that requires understanding *interaction over time* or *meaning*: whether Tab order matches logical reading order, whether a live region actually announces at the right moment with the right content, whether a focus trap releases correctly, or whether an `aria-label` that's technically present is actually *accurate and useful* rather than just non-empty. A page can have zero axe violations and still be substantially broken for a screen reader or keyboard-only user in ways only manual testing surfaces — commonly cited figures put automated coverage around a third to 40% of real-world WCAG failures, which is valuable (it's fast and catches genuine issues) but explicitly partial, not comprehensive.

The trap: treating "zero automated findings" as equivalent to "accessible" in a status update or compliance claim — this significantly overstates what was actually verified and is a common, risky shortcut under time pressure.

---

**Q (High): The customer specifically requires a VPAT (Voluntary Product Accessibility Template). How does that change what you'd produce compared to an internal engineering punch list?**

Answer: A VPAT is structured very differently from an engineering triage list — it's organized by individual WCAG success criterion (each one gets an explicit conformance level: Supports / Partially Supports / Does Not Support / Not Applicable), not by page or component, because that's the format the requesting customer's procurement/legal team will actually read it against. My two-hour audit's findings would need to be re-mapped into that structure as a follow-up step, not produced in that format directly during the time-constrained pass itself — trying to fill out a full VPAT live, criterion by criterion, during the same two hours as the actual testing would meaningfully cut into testing time for a document format that's better assembled *from* findings than generated *during* discovery. I'd be explicit that the two-hour output is an input to a VPAT, not the VPAT itself, and that a genuine VPAT typically also requires broader flow coverage and AT-combination coverage than a two-hour spot-check can responsibly claim.

The trap: conflating a fast internal triage pass with the rigor a customer-facing compliance document actually requires — presenting a two-hour audit's findings as if they constitute VPAT-level conformance verification overstates the coverage in a way that has real legal/contractual weight if it's wrong.

---

**Q (Medium): How would you prioritize fixing the findings afterward — what determines what gets fixed first?**

Answer: Severity (does it block task completion vs. degrade the experience) crossed with reach (how many users/how much of the flow does it affect) is the primary axis, with a checkout flow's step-order also mattering specifically: a blocker earlier in the flow (say, the shipping form) effectively also blocks everything after it for an affected user, even if the payment step downstream is itself flawless, so fixing earlier-flow blockers first has outsized value beyond their individual severity rating. I'd also weight fixes that address a *pattern*, not just an instance, higher than they'd otherwise rank — if the quantity-stepper's keyboard trap turns out to be the same underlying bug class used in several other custom controls across the app (not just this one instance), fixing and documenting the pattern once has much higher leverage than fixing this one occurrence and leaving the same bug elsewhere.

The trap: prioritizing purely by "how many issues are on this page" (a raw count) rather than by task-blocking severity and flow position — a page with many minor issues can rank below a page with one single blocker, and a flat issue-count metric obscures that.

---

**Q (Medium): The engineering team says "we don't have time to fix all of this before the customer deadline." How do you help them triage what's realistic?**

Answer: I'd separate the findings into "blockers that make the flow legally/functionally non-compliant if shipped as-is" versus "real issues that don't block task completion" — for a checkout flow specifically, an unreachable or unlabeled required field is existential (a portion of users literally cannot complete a purchase), while a suboptimal-but-functional heading structure is a genuine issue but not one that prevents task completion. I'd push for the blockers to be non-negotiable for the deadline regardless of remaining capacity, and treat the rest as a scoped, dated follow-up backlog — visible and tracked, not silently dropped, echoing the same "make the trade-off explicit and owned" principle from handling stakeholder pushback ([Designer Pushback on Color Contrast](05-designer-pushback-on-color-contrast.md)) rather than either engineering unilaterally deciding what's "good enough" or leadership being unaware real gaps remain.

The trap: treating "we're out of time" as license to silently deprioritize genuine blockers alongside genuine nice-to-haves without distinguishing them — the honest answer draws a hard line at task-blocking issues and treats everything past that line as a negotiable, but explicitly tracked, scope decision.

---

**Q (Low): Would your process change if this were a mobile app instead of a responsive web checkout flow?**

Answer: The same layered structure (automated scan → keyboard/switch-control-equivalent pass → screen reader pass → zoom/text-scaling pass) still applies, but the specific tools and some of the checks shift: "keyboard-only" becomes testing with a physical switch control or the platform's built-in "Full Keyboard Access" (iOS) / equivalent, automated scanning tools differ (Xcode's Accessibility Inspector, Android's Accessibility Scanner, rather than axe/Lighthouse which are web-specific), and screen reader testing uses VoiceOver on iOS or TalkBack on Android rather than NVDA, with real gesture-based navigation differences from desktop screen reader arrow-key exploration. The severity × reach triage framework and the "state what wasn't covered" discipline carry over unchanged — those are process principles independent of platform, only the specific tools and interaction models differ.

The trap: assuming web accessibility testing knowledge transfers completely unchanged to mobile — the underlying principles do, but the specific tools, gesture models, and some platform-specific checks (like dynamic type / text-scaling support) are genuinely different and worth naming specifically rather than hand-waving as "basically the same."

---

## Self-Assessment

- [ ] Can lay out a time-boxed audit process (automated → keyboard → screen reader → zoom/reflow) and justify the ordering
- [ ] Can explain specifically why a clean automated scan isn't sufficient evidence of accessibility, citing what it structurally can't detect
- [ ] Can produce a severity × reach triage framework and apply it to rank a mixed list of findings
- [ ] Can explain the difference between an engineering punch list and a VPAT's required structure
- [ ] Can name what to do when time runs out before full coverage (state the gap explicitly, don't imply completeness)
- [ ] Can prioritize a realistic fix list under a hard deadline without silently dropping genuine blockers

---
*Next: Accessible Drag-and-Drop Alternative for Keyboard Users — a specific, common finding from exactly this kind of audit (a reorderable list with no non-pointer alternative) turned into a full implementation scenario.*
