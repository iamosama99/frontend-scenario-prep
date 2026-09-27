# Design a Component Library / Design System From Scratch

## Quick Reference

| Decision Point | Choice | Why |
|---|---|---|
| Foundational layer | Design tokens (color, spacing, typography, radii, etc. as named, themeable values) beneath every component, not hardcoded values inside individual components | Tokens are what make theming (dark mode, brand variants, white-labeling) a data change instead of a per-component code change — the same "behavior driven by data, not hardcoded per-instance" principle underlying the Dashboard and Form Builder scenarios' registries, applied to visual design instead of structure |
| Component API design | Composable, unstyled-behavior-plus-styled-primitives approach (compound components exposing sub-parts, sensible prop-based variants) over large, monolithic, heavily-configurable single components | A monolithic component accumulates an ever-growing prop list trying to cover every consumer's layout need; composable sub-parts let consumers arrange things themselves while the library still owns behavior, accessibility, and styling primitives |
| Accessibility | Built into each component's default implementation (correct ARIA roles/attributes, keyboard interaction, focus management) as a non-optional baseline, not left to each consumer to add per-usage | A design system used across many teams/products only actually raises the accessibility floor org-wide if consumers get it for free by using the component correctly — accessibility that's "possible to add" per-consumer gets skipped under deadline pressure far too often to be a reliable strategy |
| Versioning & breaking changes | Semantic versioning with an explicit, tooled deprecation path (old API still works with a console warning for a announced window, codemods where feasible) before a breaking major version | A design system has many, often numerous and slow-moving, downstream consumers — an abrupt breaking change with no transition path either blocks every consumer from upgrading or forces a painful, uncoordinated simultaneous migration across the whole org |
| Theming mechanism | CSS custom properties (or an equivalent runtime-swappable token layer) rather than build-time-only theme variants (e.g., separate compiled CSS bundles per theme) | Runtime-swappable tokens support a user toggling dark mode instantly with no page reload or separate bundle download, and support arbitrary brand/white-label theming without needing a separate build per theme |

## The Scenario

"Your organization has multiple product teams each building their own UI, with growing inconsistency and duplicated effort — buttons, inputs, and modals that look and behave slightly differently everywhere. You've been asked to design a shared component library / design system to fix this. Walk me through how you'd architect it — the component API design, theming, accessibility, and how you'd roll it out and evolve it without breaking every team that adopts it."

## Clarifying Questions

- **How many consuming teams/products, and how tightly or loosely coordinated are their release cycles?** A design system consumed by a handful of tightly-coordinated teams on a shared release cadence can tolerate more frequent, faster-paced breaking changes than one consumed by dozens of independently-shipping teams across different codebases and release schedules — this materially affects how conservative the versioning/deprecation strategy needs to be.
- **Does the system need to support multiple visually distinct brands/products (a genuine multi-brand or white-label requirement), or is theming primarily about a single brand's light/dark mode?** Multi-brand theming is a meaningfully bigger design requirement on the token architecture than light/dark mode alone — it needs the token layer to support swapping an entire visual identity, not just inverting a limited light/dark palette.
- **Is this greenfield (no existing shared components, starting from zero) or a migration effort (existing, currently-duplicated per-team components need to be consolidated into the new shared library over time)?** A migration context adds a whole additional dimension to the rollout strategy — incremental adoption alongside existing bespoke components, codemods to convert existing usage, and coexistence strategies during a long transition period — that a genuinely greenfield build doesn't need to solve for.
- **What level of customization do consuming teams need per-instance — pure "use it as designed with no customization," prop-based variants only, or full style-override escape hatches for cases the library didn't anticipate?** This shapes the component API philosophy directly: an API with zero escape hatches is simpler to reason about and keep visually consistent but frustrates teams with a legitimate one-off need; some form of sanctioned "escape hatch" (a `className`/style-override prop, or slot-based composition) is almost always needed in practice, but how permissive it is is a real design decision.
- **Who owns accessibility compliance ultimately — is it entirely the design system team's responsibility to get right once inside each component, or do consuming teams still need to do their own accessibility review on top?** Establishes whether "accessible by default" is being treated as a hard guarantee the library provides (baked into every component, tested as part of the library's own CI) or a starting point consumers are still expected to independently verify for their specific composition/usage.
- **Is this a React-specific library (matching this repo's stated React + TypeScript context from Phase 3 onward), or does it need to support multiple frontend frameworks used across the org?** A multi-framework requirement pushes toward separating framework-agnostic logic (tokens, and potentially unstyled behavioral logic) from framework-specific component implementations, a substantially larger scope than a single-framework library.

## Approach & Trade-offs

**Design tokens are the foundational layer everything else builds on, and this is the single decision that most determines whether theming is ever tractable.** Every visual value a component might use — colors, spacing units, font sizes/weights, border radii, shadow depths, animation durations — should be expressed as a reference to a named, centrally-defined token (`color.background.primary`, `space.md`, `radius.default`) rather than a hardcoded literal value baked directly into a component's styles. This is structurally the same "behavior/appearance driven by data, not hardcoded per-instance" principle this repo has applied repeatedly to structural concerns (the widget and field-type registries) — here applied to visual design specifically: changing what `color.background.primary` actually resolves to (for dark mode, for a different brand, for an accessibility-driven high-contrast mode) is a single data change at the token layer, requiring zero changes to any individual component that references that token, rather than needing to hunt down and update every place a color was hardcoded.

**Component API design should favor composable primitives exposing sub-parts over large, monolithic, prop-driven components trying to anticipate every layout need through configuration — this is a real trade-off, and both extremes have genuine costs.** A monolithic approach (a single `<Modal title="..." footer={...} showCloseButton onClose={...} size="lg" ...>` component trying to support every consumer's layout variation through an ever-growing prop surface) becomes increasingly awkward as real-world usage diversity grows — eventually some consumer needs a layout combination the prop API didn't anticipate, and the fix is either an even larger, more tangled prop API, or the consumer bypassing the component entirely. A composable approach (`<Modal><Modal.Header>...</Modal.Header><Modal.Body>...</Modal.Body><Modal.Footer>...</Modal.Footer></Modal>`, a "compound component" pattern) lets consumers arrange sub-parts in whatever structure their specific case needs, while the library still fully owns each sub-part's behavior, accessibility wiring, and base styling — the consumer composes structure; the library still owns everything about how each piece actually behaves and looks. The trade-off is that composable APIs ask slightly more of a first-time consumer (learning the sub-part vocabulary) than a single all-in-one prop-driven component, but this is broadly the industry-converged answer for a reason — it scales to diverse real-world usage far better than a monolithic API's ever-expanding prop list.

**Accessibility must be built into each component's own default implementation as a non-optional baseline, not documented as something consumers are responsible for adding themselves.** Correct semantic HTML, ARIA roles/states/properties, keyboard interaction patterns (tab order, arrow-key navigation within composite widgets, Escape-to-close, focus trapping where appropriate — the same concerns covered individually in [Accessible Modal With Focus Trap](../phase-02-component-machine-coding/04-accessible-modal-focus-trap.md), [Accessible Tabs](../phase-02-component-machine-coding/05-accessible-tabs-keyboard-nav.md), and [Accessible Combobox / Dropdown](../phase-02-component-machine-coding/07-accessible-combobox-dropdown.md)), and correct focus management on state changes should all be handled inside the library's own component implementations, exercised by the library's own tests, and essentially invisible/automatic to a consumer who just uses the component as intended. The reasoning for insisting on this as a hard baseline rather than "possible to add correctly if a consumer follows the docs" is pragmatic, not merely idealistic: a shared design system's entire value proposition is raising a baseline consistently across every team that adopts it, and accessibility work that depends on each individual consuming team remembering to add it correctly, under whatever deadline pressure they're independently facing, will reliably be skipped or done incorrectly by at least some fraction of consumers — building it into the shared component is the only strategy that reliably raises the floor org-wide rather than merely making a good outcome possible for the most diligent teams.

**Versioning needs an explicit, tooled deprecation path before any breaking change ships in a major version, because a design system's consumer base is typically numerous, independently-scheduled, and slow to all move in lockstep.** Following semantic versioning (patch/minor releases are additive/non-breaking; breaking changes only ship in a major version bump) is necessary but not sufficient on its own — the more important practice is announcing and supporting a deprecated old API alongside its replacement for a defined transition window (the old prop/component still works, but triggers a development-mode console warning pointing at the migration path) before the old API is actually removed in the next major version, giving every consuming team a real opportunity to migrate on their own schedule rather than being forced into an abrupt, uncoordinated simultaneous rewrite the moment a new major version is adopted. Where the migration is mechanical (a renamed prop, a restructured but equivalent API), providing an automated codemod that rewrites consumer code directly is a meaningfully lower-friction path to adoption than expecting every team to manually find and update every usage themselves — and directly reduces the real organizational cost (and thus the real resistance) to the design system team ever being able to ship a breaking change at all.

**Theming needs to be runtime-swappable, which points toward CSS custom properties (or an equivalent runtime token mechanism) rather than build-time-only theme variants.** If tokens resolve to CSS custom properties (`--color-background-primary: #fff;`, overridden to a different value under a `[data-theme="dark"]` selector or similar), switching themes at runtime — a user toggling dark mode, or a white-labeled product loading a different brand's theme based on runtime configuration — is simply swapping which set of custom-property values is active, requiring no new bundle download and no page reload, and works uniformly across every component built on those tokens with zero component-level awareness of theming at all (a component just references `var(--color-background-primary)`; it has no idea a theme even exists, let alone which one is active). A build-time-only approach (separate compiled CSS bundles per theme, selected at build or deploy time) can't support a user-toggleable runtime switch without a full page reload (or complex, redundant bundle-swapping logic) and doesn't scale gracefully to an arbitrary, dynamically-configured number of brand themes the way a data-driven runtime token layer does.

## Solution

**Design tokens — the foundational, themeable layer, exposed as CSS custom properties:**

```css
:root {
  --color-background-primary: #ffffff;
  --color-text-primary: #1a1a1a;
  --space-sm: 8px;
  --space-md: 16px;
  --radius-default: 6px;
}

[data-theme="dark"] {
  --color-background-primary: #1a1a1a;
  --color-text-primary: #f5f5f5;
  /* spacing/radii tokens intentionally unchanged — only color-scheme-dependent tokens are redefined per theme */
}
```

```tsx
// A component references tokens, never hardcoded values, and has NO awareness that theming even exists:
const Button = styled.button`
  background: var(--color-background-primary);
  color: var(--color-text-primary);
  padding: var(--space-sm) var(--space-md);
  border-radius: var(--radius-default);
`;
```

**Composable compound-component API — the library owns behavior/accessibility; the consumer owns arrangement:**

```tsx
function Modal({ isOpen, onClose, children }: { isOpen: boolean; onClose: () => void; children: React.ReactNode }) {
  const dialogRef = useFocusTrap(isOpen); // library-owned: focus trap, Escape-to-close, aria-modal wiring — invisible to the consumer
  if (!isOpen) return null;
  return (
    <div role="dialog" aria-modal="true" ref={dialogRef} onKeyDown={(e) => e.key === 'Escape' && onClose()}>
      {children}
    </div>
  );
}
Modal.Header = function ModalHeader({ children }: { children: React.ReactNode }) {
  return <div className="modal-header">{children}</div>;
};
Modal.Body = function ModalBody({ children }: { children: React.ReactNode }) {
  return <div className="modal-body">{children}</div>;
};
Modal.Footer = function ModalFooter({ children }: { children: React.ReactNode }) {
  return <div className="modal-footer">{children}</div>;
};

// Consumer freely arranges sub-parts for THEIR specific layout need — no prop explosion required:
function DeleteConfirmationModal({ isOpen, onClose, onConfirm }: DeleteModalProps) {
  return (
    <Modal isOpen={isOpen} onClose={onClose}>
      <Modal.Header>Delete this item?</Modal.Header>
      <Modal.Body>This action can't be undone.</Modal.Body>
      <Modal.Footer>
        <Button variant="secondary" onClick={onClose}>Cancel</Button>
        <Button variant="danger" onClick={onConfirm}>Delete</Button>
      </Modal.Footer>
    </Modal>
  );
}
```

**Deprecation path — old API kept working, with a clear migration signal, ahead of eventual removal:**

```tsx
function Button({ variant, kind, ...props }: ButtonProps & { kind?: string /* @deprecated use `variant` */ }) {
  if (kind !== undefined && process.env.NODE_ENV !== 'production') {
    console.warn(
      `Button: the "kind" prop is deprecated and will be removed in v3.0. Use "variant" instead. ` +
      `See migration guide: https://design-system.example/migrations/kind-to-variant`
    );
  }
  const resolvedVariant = variant ?? kind; // old prop still fully functional during the deprecation window
  return <ButtonImpl variant={resolvedVariant} {...props} />;
}
```

> **Check yourself:** Without looking above, explain why a component built on design tokens (like the `Button` styled-component above) needs zero awareness that theming exists at all, and describe what would have to change about that component if theming were instead implemented as build-time-only separate CSS bundles per theme.

## Rollout & Governance

**Incremental, opt-in adoption alongside existing bespoke components, not a forced big-bang migration**, is the realistic rollout model for a non-greenfield context — new features and actively-touched code paths adopt the shared library going forward, while existing, stable, rarely-touched bespoke components are migrated opportunistically or left alone if the cost of migrating clearly outweighs the benefit, rather than mandating every team stop and rewrite everything simultaneously.

**A contribution/governance model deciding how new components get added or existing ones evolve** — commonly a small core team owning the library's architecture, tokens, and accessibility standards, with a defined process (an RFC-style proposal, a design review) for consuming teams to request or contribute new components/variants, balancing consistency (a small, deliberate core team prevents every team independently adding slightly-inconsistent one-off components to the shared library) against responsiveness (consuming teams need a real path to get genuinely needed new components added, not an indefinite bottleneck).

**Automated visual regression testing and accessibility auditing as part of the library's own CI**, catching an unintended visual or accessibility regression in a shared component before it ships to every single consumer simultaneously — the blast radius of a bug in a shared design-system component is every product using it, which raises the bar for how rigorously the library's own test suite needs to guard against regressions compared to a single product team's own component.

## Gotchas

**Hardcoding visual values directly in components instead of referencing design tokens.** Makes theming (dark mode, multi-brand) require hunting down and rewriting every hardcoded value across every component, rather than a single data change at the token layer.

**A single, ever-growing monolithic component API trying to support every consumer's layout need through more and more props.** Becomes increasingly unwieldy and eventually forces consumers to bypass the component entirely for cases its prop API didn't anticipate — composable sub-parts avoid this by letting consumers arrange structure themselves.

**Treating accessibility as documentation/guidance for consumers to implement themselves, rather than building it into each component's own default behavior.** Reliably produces inconsistent accessibility outcomes across consuming teams, since it depends on every team independently getting it right under their own deadline pressure — the whole point of a shared library is raising the floor automatically, not just making a good outcome possible.

**Shipping a breaking change with no deprecation window, console warning, or migration path.** Forces every consuming team into an uncoordinated, simultaneous, all-at-once migration the moment they need any other update from the library — a major, avoidable source of organizational friction and resistance to ever upgrading at all.

**Build-time-only theme variants (separate compiled bundles per theme) when a runtime-toggleable theme (e.g., user-switchable dark mode) is actually required.** Can't support an instant, reload-free theme switch, and scales poorly to an arbitrary or dynamically-configured number of brand themes compared to a data-driven, runtime custom-property-based token layer.

**No sanctioned escape hatch for genuine one-off customization needs.** An API with zero flexibility for cases the library didn't anticipate pushes consuming teams toward forking or bypassing the shared component entirely, quietly reintroducing the exact inconsistency-and-duplication problem the design system was built to solve.

## Follow-up Questions

**Q (High): Why is the compound-component (composable sub-parts) pattern generally preferred over a single, highly-configurable monolithic component for something like a Modal, and when might the monolithic approach actually be the better choice?**

Answer: The compound pattern scales better to diverse, hard-to-fully-anticipate real-world layout needs because the library only needs to own and guarantee correctness for each individual sub-part's behavior (a header that's just a styled container, a footer that's just a styled container, the outer dialog owning focus trap/ARIA/keyboard behavior) — consumers freely compose these sub-parts into whatever structure their specific case needs without the library ever needing to have anticipated that exact combination through a dedicated prop. A monolithic API, by contrast, needs an explicit prop for every variation it wants to support (a `footerAlignment` prop, a `hideCloseButton` prop, a `customHeaderContent` prop...), and inevitably reaches a real case its authors didn't anticipate, at which point a consumer either petitions for yet another prop or works around the component entirely. That said, a monolithic, more tightly-constrained API can be the *better* choice specifically when consistency matters more than flexibility for a given component — a component intentionally meant to always look and behave nearly identically everywhere it's used (say, a toast/notification component, where allowing arbitrary internal composition would undermine the very consistency the component exists to enforce) is a reasonable candidate for a smaller, more constrained, monolithic-style API precisely because *not* offering much compositional flexibility is the intended design constraint, not a limitation.

The trap: presenting the compound-component pattern as an unconditionally superior default with no acknowledgment that a more constrained, monolithic API is sometimes the deliberately correct choice — recognizing that the right API shape depends on whether the component's purpose benefits from consumer-controlled flexibility or from library-enforced rigidity is the stronger, more senior answer.

---

**Q (High): How would you actually ship a breaking change to a widely-adopted component (e.g., renaming a core prop) without forcing every consuming team to migrate simultaneously?**

Answer: The old prop should continue to fully function, not merely be accepted and ignored, for a defined deprecation window spanning at least one full minor-version release cycle (giving consuming teams real time to notice and act, rather than a change that's announced and removed in the same release) — internally mapping the deprecated prop's value onto the new one's behavior, as shown in the solution's `Button` example above, so existing consumer code keeps working exactly as before with zero required immediate action. Alongside this, the library should surface a clear, actionable development-mode warning (a `console.warn` naming exactly what's deprecated, what to use instead, and linking a migration guide) so teams discover the need to migrate through their own normal development process rather than needing to proactively read a changelog. Where the migration is mechanically expressible (a straightforward prop rename, a restructured-but-equivalent API), providing an automated codemod that rewrites consumer source code directly removes even the manual-effort barrier to migrating, meaningfully increasing how many consuming teams actually complete the migration within the deprecation window rather than continuing to rely on the deprecated path indefinitely until forced. Only once the deprecation window has passed (and, realistically, telemetry/usage data confirms remaining usage of the deprecated path is negligible) does the next major version actually remove the old prop entirely.

The trap: treating a version bump alone (shipping the breaking change directly in a new major version, with semantic versioning as the only signal) as sufficient — semantic versioning correctly communicates "this is a breaking change you must review before upgrading," but does nothing to reduce the actual migration effort or provide a transition window, which is the part that determines whether consuming teams can and will actually adopt the new major version in any reasonable timeframe.

---

**Q (High): The interviewer asks: "One consuming team says the design system's Button component doesn't support the exact shadow/border style their specific marketing landing page needs. How do you handle this without either breaking consistency everywhere or blocking that team indefinitely?"**

Answer: This is precisely the situation a sanctioned, deliberate escape hatch exists for — rather than either rigidly refusing any customization (pushing the team to fork or bypass the component, quietly recreating the exact fragmentation problem the design system exists to prevent) or reflexively adding a new bespoke prop to the shared `Button` component for a genuinely one-off need (bloating the shared API for a case that may not generalize to any other consumer), the library should offer a constrained, explicit override mechanism — commonly a `className`/`style`-prop escape hatch, or in a CSS-custom-property-based token system, the ability for a consumer to locally override specific token values within their own scoped context (e.g., wrapping their marketing page section in a container that redefines `--radius-default` and a shadow-related token just for that scope) — that lets this one team achieve their specific visual need without requiring a change to the shared component's core API or behavior, and without that customization leaking into or affecting any other consumer. If, over time, multiple unrelated teams request the same customization independently, that's a real signal the shared component's default API genuinely should be extended to support that variant more formally — but a single one-off request is better served by the escape hatch than by growing the core API preemptively.

The trap: either refusing all customization outright (driving consumers to bypass or fork the component, undermining the design system's adoption and value) or immediately adding a new bespoke prop to the shared component's core API for what might be a genuinely one-off need — the escape-hatch pattern is specifically what avoids having to choose between those two costly extremes for every individual customization request.

---

**Q (Medium): How would you validate that accessibility is actually correctly implemented across the library's components, rather than just assuming it is because the components were built with accessibility in mind?**

Answer: Automated accessibility auditing (tools like axe-core integrated into the library's own component test suite, catching common, detectable issues — missing labels, insufficient color contrast against the current theme's tokens, incorrect ARIA attribute usage) should run as part of the library's CI on every change, catching a meaningful class of accessibility regressions automatically before any release ships to every consumer simultaneously. This needs to be supplemented, not replaced, by manual testing that automated tooling fundamentally cannot verify — actual keyboard-only navigation testing (can every interactive component be fully operated without a mouse, in a sensible tab order, with correct visible focus indication) and real screen-reader testing (does the experience actually make sense read aloud, not just "does it have the technically-correct ARIA attributes present") — automated tools reliably catch missing/incorrect attributes but cannot verify that the actual experience they produce is genuinely usable.

The trap: treating automated accessibility linting/auditing tooling as sufficient on its own — these tools are valuable and should absolutely be run continuously, but they detect a specific, limited class of technically-detectable issues and cannot verify the deeper, more important question of whether the actual interactive experience is genuinely usable via keyboard or screen reader, which requires real manual testing to confirm.

---

**Q (Medium): Would you build this design system as a single framework-specific (e.g., React-only) library, or design it to be usable across multiple frontend frameworks used across the org?**

Answer: This depends heavily on the clarified scope from earlier — if every consuming team is on the same framework (React + TypeScript, per this repo's stated context), a framework-specific library is the pragmatic default, since it can fully leverage that framework's own component/composition model (as the compound-component pattern above does) without the added complexity of framework-agnostic abstraction layers. If genuinely multiple frameworks are in active use across the org, a common approach separates framework-agnostic concerns (design tokens as plain CSS custom properties, usable identically regardless of framework; potentially shared, framework-agnostic behavioral logic via Web Components or headless/unstyled-behavior libraries) from framework-specific component wrappers implemented separately per framework on top of that shared foundation — the tokens and core interaction logic are written once and shared; the actual component implementations (and their idiomatic APIs) are framework-specific, since forcing one framework's component model onto every framework used in the org tends to produce an awkward, lowest-common-denominator API that fits none of them particularly well.

The trap: assuming a single-framework answer is always sufficient without first confirming the org's actual framework diversity, or, in the other direction, over-engineering a fully framework-agnostic architecture (e.g., building everything as Web Components) for an org that's actually entirely on one framework, incurring real complexity cost for a multi-framework requirement that doesn't actually exist.

---

**Q (Low): How would you measure whether the design system is actually succeeding at its goal (reducing inconsistency and duplicated effort across teams), beyond just "teams are using it"?**

Answer: Meaningful signals go beyond raw adoption/usage counts (which can be misleadingly high even if usage is shallow or heavily overridden) — worth tracking include the rate of consumer teams needing to override or bypass shared components' default styling/behavior (a high override rate suggests the library's defaults aren't actually meeting real needs, undermining the consistency goal even where the component is nominally "adopted"), the time/effort required for a team to build a new UI surface using the library versus a historical baseline before it existed (a direct measure of the "duplicated effort" reduction goal), and accessibility audit pass rates across products using the library's components versus those that aren't (a direct measure of whether the "raises the accessibility floor org-wide" goal is actually being realized in practice, not just in theory).

The trap: measuring only surface-level adoption (how many teams have installed/imported the library) without also measuring the depth and fidelity of that adoption (how much overriding/bypassing is happening underneath nominal usage) — a design system can show impressive adoption numbers while still failing at its actual underlying goal if teams are widely overriding its defaults to the point that visual/behavioral consistency across products hasn't meaningfully improved.

---

## Self-Assessment

- [ ] Can explain why design tokens are the foundational layer and why a component built on them needs zero awareness that theming exists
- [ ] Can articulate the compound-component vs. monolithic-API trade-off and name a case where the monolithic approach is actually preferable
- [ ] Can explain why accessibility must be built into each component's default implementation rather than left to consumers, and why that's a pragmatic (not just idealistic) argument
- [ ] Can design a deprecation path (dual-support window, warning, codemod) for a breaking API change and explain why a version bump alone is insufficient
- [ ] Can explain why runtime-swappable theming (CSS custom properties) is preferred over build-time-only theme bundles
- [ ] Can describe a sanctioned escape-hatch mechanism for one-off customization needs and explain why it's preferable to either refusing customization or growing the core API per request

---
*Phase 4 (all 15 scenarios) now complete. Next up: Phase 5 — Performance Debugging Scenarios, starting with "Diagnosing Poor LCP" — shifts from architecture/design-under-ambiguity to systematic diagnosis of a stated, measurable symptom, the skill of narrowing from a broad performance metric down to its actual root cause under interview time pressure.*
