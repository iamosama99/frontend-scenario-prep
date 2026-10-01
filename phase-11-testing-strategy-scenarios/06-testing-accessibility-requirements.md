# Testing an Accessibility Requirement

## Quick Reference

| Layer | Tool | Catches | Misses |
|---|---|---|---|
| Static/lint | `eslint-plugin-jsx-a11y` | Missing alt, bad ARIA props at author time | Anything runtime or behavioral |
| Automated rules | `axe-core` (jest-axe, `@axe-core/playwright`) | Contrast, missing labels, invalid ARIA, landmarks (~30–40% of issues) | Whether it *makes sense* or *works* |
| Behavioral | RTL/Playwright with role queries, keyboard, focus assertions | Tab order, focus management, Escape, announcements wired | Real AT behavior quirks |
| Manual | Keyboard-only pass, screen reader (VoiceOver/NVDA), zoom, reduced motion | Everything else — meaning, flow, clarity | Doesn't scale; do per release/feature |

## The Scenario

"Product has a new requirement: 'The new Add Team Member dialog must be accessible — WCAG 2.2 AA.' How do you turn that into tests, and how do you know you're actually done?"

## Clarifying Questions

- **What does "accessible" mean concretely — which standard and level, which assistive tech and browsers must be supported?** A requirement like "WCAG 2.2 AA, NVDA+Firefox and VoiceOver+Safari" is testable; "accessible" is not.
- **What are the interactive elements and states in the dialog?** Fields, validation errors, async submit, success announcement, focus on open/close — each yields concrete criteria.
- **Is there a design system with already-verified primitives (Dialog, Field)?** If so, test composition rather than re-deriving behavior of tested components.
- **Who owns accessibility sign-off — a specialist, or the team?** Determines whether manual screen reader testing is available.
- **Is this a gate (blocks release) or an ongoing quality bar?** Informs CI strategy.

## Approach & Trade-offs

**Translate the vague requirement into verifiable acceptance criteria.** For a modal: focus moves into the dialog on open and returns to the trigger on close; Tab/Shift+Tab are trapped; Escape closes; the dialog has an accessible name and `role="dialog"` with `aria-modal`; background is inert; every input has a label; errors are programmatically associated and announced; contrast and target size meet AA; no information relies on color alone; works at 200% zoom and with reduced motion.

**Layer the tests by what each can prove.** Automated axe checks are cheap and catch real violations, but they only cover a minority of WCAG issues — they can tell you a button has no name, not that the name is *meaningful* or the flow is *usable*. So automation is a floor, not a certificate.

**Use behavioral tests for what axe cannot see.** Keyboard and focus behavior is testable: simulate Tab, Escape, and assert `document.activeElement`. Querying by `getByRole('dialog', { name })` doubles as a semantics assertion — if the role or name is wrong, the test fails.

**Be honest about the manual part.** Screen reader output, reading order, and "does this make sense" need humans. I'd schedule a short manual pass (keyboard-only; VoiceOver and NVDA) per feature or release, and put the results in the PR. Trying to fully automate it creates false confidence.

**Trade-off: CI gating strictness.** Failing CI on any axe violation is clean but can block on false positives (e.g., color-contrast in jsdom, which can't compute it — axe disables that rule there). Run contrast checks in a real browser (Playwright + axe) and keep an explicit, reviewed allowlist for justified exceptions.

## Solution

### Automated: axe in unit and E2E

```tsx
import { axe } from 'jest-axe';

it('has no detectable a11y violations', async () => {
  const { container } = render(<AddMemberDialog open onClose={() => {}} />);
  expect(await axe(container)).toHaveNoViolations();
});
```

```ts
// Playwright — real browser, so contrast is evaluated
import AxeBuilder from '@axe-core/playwright';

test('dialog passes axe', async ({ page }) => {
  await page.goto('/team');
  await page.getByRole('button', { name: 'Add team member' }).click();
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag22aa'])
    .include('[role="dialog"]')
    .analyze();
  expect(results.violations).toEqual([]);
});
```

### Behavioral: semantics, focus, keyboard

```tsx
it('manages focus and keyboard correctly', async () => {
  const user = userEvent.setup();
  render(<TeamPage />);
  const trigger = screen.getByRole('button', { name: /add team member/i });

  await user.click(trigger);
  const dialog = screen.getByRole('dialog', { name: /add team member/i });
  expect(dialog).toHaveAttribute('aria-modal', 'true');
  expect(within(dialog).getByLabelText(/email/i)).toHaveFocus();   // initial focus

  // Focus trap: Shift+Tab from first goes to last, Tab from last wraps to first
  await user.tab({ shift: true });
  expect(within(dialog).getByRole('button', { name: /cancel/i })).toHaveFocus();
  await user.tab();
  expect(within(dialog).getByLabelText(/email/i)).toHaveFocus();

  await user.keyboard('{Escape}');
  expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
  expect(trigger).toHaveFocus();                                   // focus restored
});

it('associates and announces validation errors', async () => {
  const user = userEvent.setup();
  render(<AddMemberDialog open onClose={() => {}} />);
  await user.click(screen.getByRole('button', { name: /^add$/i }));

  const email = screen.getByLabelText(/email/i);
  expect(email).toBeInvalid();
  expect(email).toHaveAccessibleDescription(/enter a valid email/i);
  expect(screen.getByRole('alert')).toBeInTheDocument();           // or a polite live region
});
```

### Manual checklist (recorded in the PR)

- Keyboard only: reach, operate, escape, no traps, visible focus indicator (WCAG 2.4.7/2.4.11)
- VoiceOver+Safari and NVDA+Firefox: dialog name announced on open, fields announced with labels, error announced, success announced on close
- 200% zoom / 320px reflow, no horizontal scroll (1.4.10)
- `prefers-reduced-motion` respected
- Target size ≥ 24×24 CSS px (2.5.8)

### Preventing regressions

- `eslint-plugin-jsx-a11y` in lint
- axe on component/story tests in CI; Storybook a11y addon for fast feedback
- Primitives (Dialog, Field) from the design system tested once, thoroughly

> **Check yourself:** Write five concrete, testable acceptance criteria for "the dialog is accessible," and say which layer verifies each.

## Gotchas

- **"axe passes" ≠ accessible.** Automation catches a fraction of issues and no judgment calls.
- **jsdom can't evaluate contrast or layout.** Color-contrast axe rule is skipped there; check in a real browser.
- **`getByTestId` everywhere.** Throws away the free semantics assertion that role/name queries give.
- **Testing the happy path only.** Error, loading, and success announcements are where screen reader users get lost.
- **Focus restoration forgotten.** Trigger unmounts or loses focus → user dumped at top of the page.
- **Hiding with `display:none` vs. `aria-hidden` confusion.** Tests must verify real exposure to the accessibility tree.
- **One-time audit.** Accessibility regresses; it needs CI guards and recurring manual checks.

## Follow-up Questions

**Q (High): Can you automate accessibility testing fully?**

Answer: No. Tools like axe detect roughly a third of WCAG issues — the machine-checkable ones (missing names, invalid ARIA, contrast). They can't judge whether a label is meaningful, the reading order makes sense, focus movement is logical, or announcements are timely. So: automation for the floor and regression prevention, behavioral tests for keyboard/focus, and manual screen reader and keyboard passes for the rest.

The trap: "We run axe in CI, so we're compliant."

**Q (High): How do you test focus management in an automated test?**

Answer: Drive the real keyboard with `userEvent`/Playwright and assert `document.activeElement` (or `toHaveFocus`) at each step: initial focus on open, trap on Tab/Shift+Tab, Escape closes, focus returns to the trigger. In Playwright this runs in a real browser, catching inert/focus-visible behavior jsdom can't.

The trap: asserting only that the dialog renders.

**Q (Medium): How do you test that a screen reader announces something?**

Answer: Automated tests can assert the *mechanism*: a live region (`role="status"`/`alert`) exists and its text updates, or `aria-describedby` points at the right text. Whether a screen reader actually speaks it, and when, requires manual testing with real AT (or an emerging tool like Guidepup driving VoiceOver/NVDA in CI for smoke checks).

The trap: claiming a unit test proves announcement.

**Q (Medium): Where do you put accessibility in the pipeline?**

Answer: Shift left: lint rules and Storybook a11y checks give authors immediate feedback; axe in component tests and Playwright runs in CI gates regressions; scheduled manual audits plus a recorded screen reader pass for significant features. Tested design-system primitives reduce per-feature burden.

The trap: a single pre-launch audit as the whole strategy.

**Q (Low): How do you handle an axe rule you disagree with or a false positive?**

Answer: Verify it's actually a false positive (often it isn't), then disable narrowly for the specific element with a comment explaining why and linking the issue — never globally. Review the allowlist periodically.

The trap: disabling rules wholesale to get CI green.

## Self-Assessment

- [ ] Can convert "make it accessible" into concrete, verifiable criteria
- [ ] Can state what axe catches, what it misses, and why
- [ ] Can write focus-trap/Escape/focus-restore tests with real keyboard events
- [ ] Can use role/name queries as semantic assertions
- [ ] Can describe the manual testing that remains and how to record it
- [ ] Can explain why contrast checks need a real browser

---
*Phase 11 complete. Next: Phase 12 — Architecture, Migration & Engineering Judgment, starting with "Migrating a Legacy jQuery/AngularJS App to React — Incrementally," which shifts from verifying code to deciding how large systems evolve.*
