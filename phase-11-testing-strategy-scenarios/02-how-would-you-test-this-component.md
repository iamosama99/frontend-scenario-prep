# "How Would You Test This Component?" Exercise

## Quick Reference

| Decision | Mechanism | Why it's the right call |
|---|---|---|
| Test through the user's interface | React Testing Library: query by role/label, interact with `userEvent` | Tests survive refactors; they fail only when behavior breaks |
| Pick the layer by risk | Pure logic → unit; component + state + network → integration; critical journeys → few E2E | Cheapest layer that can catch the bug; avoid inverting the pyramid |
| Mock at the boundary, not inside | Mock the network (MSW), not your own hooks/children | Keeps real wiring under test |
| Enumerate behaviors, not lines | Happy path, validation, async states, errors, a11y, edge inputs | Coverage % measures execution, not confidence |

## The Scenario

"Here's a `<CouponForm>`: the user types a coupon code, clicks Apply, we call `POST /api/coupons/validate`, and on success the order total updates; on failure we show an error. How would you test this? Walk me through what you'd write and what you wouldn't."

## Clarifying Questions

- **Who owns the total — this component, or a parent/cart store?** Determines whether "total updates" is asserted here (via a callback/prop) or in an integration test with the real cart.
- **What are the server's failure modes — invalid code, expired, already used, network down, 500?** Each is a distinct user-visible state worth a test; "error" is not one case.
- **Is the button disabled while pending? Is double-submit possible?** Behavior I'd want pinned down because it's where real bugs live.
- **Is there existing test infrastructure (MSW, Testing Library, Playwright)?** I'll match conventions rather than introduce a new stack.
- **What's the cost of a bug here?** Money-adjacent flows justify one E2E smoke path; a cosmetic widget doesn't.

## Approach & Trade-offs

**Start from behaviors, not implementation.** I list what a user can observe: enters a code and applies it → sees the discount; submits an empty code → sees validation, no request fired; code rejected → sees a specific message and can retry; request in flight → button disabled/announced; network failure → recoverable error; keyboard-only and screen reader can complete it.

**Choose the layer deliberately.**

- *Unit* for any pure logic extracted from the component (code normalization — trim, uppercase — and money math). Fast, exhaustive.
- *Component integration* (RTL + MSW) as the workhorse: render the real component, drive it like a user, mock only the HTTP boundary. This catches wiring bugs unit tests can't, without a browser.
- *E2E* for one happy path through real cart + checkout. Slow and flaky-prone, so a single smoke test, not every error branch.

**What I deliberately don't test:** that `useState` updates, internal function calls, CSS class names, snapshot of the whole DOM tree, or that a child received certain props. These couple tests to implementation and break on harmless refactors while missing real bugs.

**Trade-off: mocking `fetch` vs. mocking the module.** Mocking `validateCoupon` (the module) is simpler but leaves the request-building and response-parsing untested. Intercepting at the network with MSW tests the whole client path and survives swapping fetch for a different client. See the dedicated MSW scenario later in this phase.

## Solution

```tsx
// CouponForm.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { http, HttpResponse } from 'msw';
import { server } from '../test/server';
import { CouponForm } from './CouponForm';

function setup(onApplied = vi.fn()) {
  const user = userEvent.setup();
  render(<CouponForm onApplied={onApplied} />);
  return { user, onApplied };
}

it('applies a valid code and reports the discount', async () => {
  server.use(http.post('/api/coupons/validate', () =>
    HttpResponse.json({ code: 'SAVE10', discountCents: 1000 })));
  const { user, onApplied } = setup();

  await user.type(screen.getByLabelText(/coupon code/i), '  save10 ');
  await user.click(screen.getByRole('button', { name: /apply/i }));

  expect(await screen.findByText(/\$10\.00 off/i)).toBeInTheDocument();
  expect(onApplied).toHaveBeenCalledWith({ code: 'SAVE10', discountCents: 1000 });
});

it('does not call the API for an empty code', async () => {
  const spy = vi.fn();
  server.use(http.post('/api/coupons/validate', () => { spy(); return HttpResponse.json({}); }));
  const { user } = setup();

  await user.click(screen.getByRole('button', { name: /apply/i }));

  expect(screen.getByRole('alert')).toHaveTextContent(/enter a code/i);
  expect(spy).not.toHaveBeenCalled();
});

it('shows a specific error for an expired code and lets the user retry', async () => {
  server.use(http.post('/api/coupons/validate', () =>
    HttpResponse.json({ error: 'EXPIRED' }, { status: 422 })));
  const { user } = setup();

  await user.type(screen.getByLabelText(/coupon code/i), 'OLD');
  await user.click(screen.getByRole('button', { name: /apply/i }));

  expect(await screen.findByRole('alert')).toHaveTextContent(/expired/i);
  expect(screen.getByRole('button', { name: /apply/i })).toBeEnabled();
});

it('prevents double-submit while pending', async () => {
  let hits = 0;
  server.use(http.post('/api/coupons/validate', async () => {
    hits++; await delay(50); return HttpResponse.json({ code: 'A', discountCents: 1 });
  }));
  const { user } = setup();

  await user.type(screen.getByLabelText(/coupon code/i), 'A');
  const btn = screen.getByRole('button', { name: /apply/i });
  await user.click(btn);
  await user.click(btn);

  await screen.findByText(/off/i);
  expect(hits).toBe(1);
});

it('recovers from a network failure', async () => {
  server.use(http.post('/api/coupons/validate', () => HttpResponse.error()));
  const { user } = setup();
  await user.type(screen.getByLabelText(/coupon code/i), 'A');
  await user.click(screen.getByRole('button', { name: /apply/i }));
  expect(await screen.findByRole('alert')).toHaveTextContent(/try again/i);
});
```

Plus: a `jest-axe` assertion on the rendered form, and one Playwright test: add item → apply real coupon on staging data → total reflects the discount at checkout.

> **Check yourself:** For each of the five tests above, which layer is it, and why would a unit test with a mocked `fetch` have missed the bug it targets?

## Gotchas

- **Querying by test id or class first.** Role/label queries double as an accessibility check; `getByTestId` is the last resort.
- **Asserting on internal state or call counts of your own hooks.** Breaks on refactor, proves nothing about behavior.
- **Not awaiting async UI.** Using `getBy` right after a click instead of `findBy`/`waitFor` produces flaky or false-pass tests.
- **Whole-component snapshots.** Huge, noisy, rubber-stamped on update.
- **Chasing 100% coverage.** Covered ≠ asserted; a test with no meaningful assertion still counts.
- **Testing the happy path only.** The error, pending, and empty states are where production bugs are.
- **Using `fireEvent` where `userEvent` is right.** `fireEvent.change` skips focus, keystrokes, and pointer sequences, missing real interaction bugs.

## Follow-up Questions

**Q (High): Why test through the DOM instead of calling methods or checking state?**

Answer: Because the user's contract is the rendered output and interactions. Tests coupled to implementation fail on safe refactors (false negatives for the refactor) and pass when the UI is actually broken (the state is right but it isn't rendered). Testing Library's guiding principle: the more your tests resemble how the software is used, the more confidence they give.

The trap: defending enzyme-style `wrapper.state()` assertions.

**Q (High): What do you mock, and what do you refuse to mock?**

Answer: Mock the boundaries you don't own or that are slow/nondeterministic: network, time, randomness, browser APIs missing in jsdom. Don't mock your own child components, hooks, or utility modules — that tests the mock. If a unit is hard to test without mocking internals, that's a design signal, not a testing problem.

The trap: shallow rendering everything and mocking every dependency until the test verifies nothing.

**Q (Medium): Where does this stop being a component test and become an E2E test?**

Answer: When correctness depends on real integration the component test fakes: the real cart store, routing, auth, the actual backend contract, or cross-page behavior. I keep one E2E happy path for those seams and leave exhaustive branches to the faster layer.

The trap: either E2E-ing every branch (slow, flaky) or never verifying the real seam.

**Q (Medium): jsdom doesn't do layout. What can't you test here?**

Answer: Anything layout/visual: positioning, overflow, responsive breakpoints, focus-visible rendering, real scrolling. Those go to a real browser: Playwright component tests, E2E, or visual regression. jsdom also lacks some APIs (IntersectionObserver, ResizeObserver) that need polyfills/mocks.

The trap: asserting `toBeVisible` on CSS-driven visibility and trusting it.

**Q (Low): How do you test the a11y requirement for this form?**

Answer: Use role/label queries (they fail if the semantics are wrong), assert `role="alert"` for errors, run jest-axe for static violations, and manually/E2E verify keyboard flow and screen reader announcements — automation catches roughly a third of issues. Covered more in the final scenario of this phase.

The trap: "axe passes, so it's accessible."

## Self-Assessment

- [ ] Can list the observable behaviors of a component before writing any test
- [ ] Can assign each behavior to unit / integration / E2E and justify it
- [ ] Can write an RTL + MSW test with `userEvent` and `findBy` correctly
- [ ] Can name what I deliberately don't test and why
- [ ] Can explain what jsdom can't verify and where that coverage lives

---
*Next: Testing a Race-condition-prone Async Component — takes the "pending/error" states from this exercise and tackles the hardest one: out-of-order responses.*
