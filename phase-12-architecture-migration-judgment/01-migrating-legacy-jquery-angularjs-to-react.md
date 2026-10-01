# Migrating a Legacy jQuery/AngularJS App to React — Incrementally

## Quick Reference

| Decision | Mechanism | Why it's the right call |
|---|---|---|
| Don't rewrite; strangle | Strangler-fig: new features and touched screens in React, legacy shrinks over time | Ships value continuously; no 18-month big-bang with no feedback |
| Coexist at a seam | Mount React islands inside legacy pages (or legacy inside React shell) via a thin bridge | Lets both worlds run in one app while migrating |
| Migrate by route/feature, leaf-first | Pick isolated, high-churn, low-coupling screens first | Highest payoff, lowest risk, builds team muscle |
| Freeze the legacy surface | No new features in legacy; bug-fix only; lint/guard against growth | Prevents the migration target from moving |

## The Scenario

"We have a 9-year-old AngularJS 1.x app, about 400 views, with jQuery plugins sprinkled through it. AngularJS is end-of-life, hiring is painful, and the product team doesn't want a feature freeze. Leadership asks you to get us onto React. How do you approach it?"

## Clarifying Questions

- **What's the actual driver — security (EOL), hiring, velocity, UX limits?** The driver sets the finish line; "get off EOL framework" differs from "improve UX," and tells me how aggressive and how complete the migration must be.
- **Can we afford a feature freeze? How much headcount, what timeline, what's the business tolerance for risk?** The prompt says no freeze — so incremental is mandatory, not a preference.
- **How is the app built and deployed — one bundle, server-rendered templates, a single page?** Determines how React can be mounted next to the legacy code.
- **How coupled is the code — shared `$scope`/services, global state, jQuery DOM manipulation, directives used everywhere?** Coupling decides migration order and whether a shared-state bridge is needed.
- **What's the test coverage?** If coverage is thin, characterization tests (or E2E on critical journeys) come before touching anything.
- **Which areas change most often vs. rarely?** Migrate hot areas first; code that's stable and ugly may never need migrating.

## Approach & Trade-offs

**Rewrite vs. incremental.** A big-bang rewrite is seductive and almost always wrong at this size: the old app keeps changing while you rebuild it, the new one reaches parity late if ever, risk is concentrated at cutover, and you've shipped zero value for a year. Incremental migration (strangler-fig pattern) trades elegance for continuous delivery and reversibility. The cost: you live with two frameworks for a long time, so the *seam* quality and the discipline to eventually finish matter enormously.

**Choosing the seam.** Options:

1. *React inside Angular:* wrap React components as AngularJS directives (via `react2angular` or a small custom bridge) and render them within existing templates. Good for migrating from the leaves up (a datepicker, a data table) while Angular owns routing.
2. *Angular inside React / new shell:* build a React shell that owns routing and layout, embedding legacy pages in an iframe or a mounted legacy app per route. Good when you want the long-term architecture (routing, auth, design system) to be new from day one.
3. *Route-level split:* old and new apps live side by side behind the same domain/reverse proxy, navigation between them is a full page load, shared auth via cookie/session. Simplest isolation, worst UX at the boundaries.

My default: **React islands in leaves first, then flip routing ownership to a React shell once enough of the tree is converted.** Leaf-first gets value early with low coupling; the shell flip happens when the ratio justifies it.

**Order of work.** (1) Safety net: E2E smoke on critical journeys + characterization tests. (2) Foundations: build tooling that bundles both (Webpack/Vite), shared design tokens/CSS so UI looks coherent, a typed API client usable from both worlds, a shared auth/session. (3) Pick an *exemplar* migration (one meaningful, contained screen) to establish patterns and reveal surprises. (4) Rule: new features in React; touched legacy screens migrated opportunistically ("Boy Scout" rule) or via budgeted allocation (e.g., 20%). (5) Track progress with a visible metric (% of routes/LOC on React) and a deadline for removing the bridge.

**State is the hard part.** Angular `$scope`/services and React state don't talk to each other. Avoid two-way entanglement: extract shared state/services into framework-agnostic modules (plain TS + a small store/event bus) that both sides consume. Migrating a screen means moving its state out of `$scope` first.

**Trade-offs I'd make explicit.** Bundle size grows while both frameworks ship — accept temporarily, code-split the legacy bundle. Performance dips at the bridge (two change-detection systems). Duplicate UI components exist for a while. Mitigate with a hard rule: *no new Angular code*, and an explicit exit date.

**What I'd say no to.** Migrating everything. Some stable, rarely touched, internal screens may be cheaper to leave on a quarantined legacy bundle (or retire) than to rewrite. "Done" means no security exposure and no new development in legacy — not necessarily zero lines of AngularJS.

## Solution

### Rollout plan

| Phase | Work | Exit criteria |
|---|---|---|
| 0 — Safety net | E2E on top ~10 journeys, monitoring/error tracking baseline, characterization tests | Can detect regressions in legacy |
| 1 — Foundations | Dual-bundle tooling, shared tokens, typed API client, auth/session, bridge component | Can render a React component in a legacy page |
| 2 — Exemplar | Migrate one hot, contained screen end-to-end | Pattern doc, perf/bundle impact measured |
| 3 — Leaf-first expansion | New features in React; migrate touched screens; shared components → design system | % of traffic through React rising |
| 4 — Shell flip | React router owns navigation; legacy screens mounted as legacy islands | Routing/auth/layout fully new |
| 5 — Tail & removal | Migrate or retire remaining screens; delete bridge and AngularJS | No AngularJS in prod bundle |

### The bridge: a React component inside an AngularJS template

```ts
// react-bridge.ts
import { createRoot, Root } from 'react-dom/client';

angular.module('app').directive('reactMount', ['$injector', ($injector) => ({
  restrict: 'E',
  scope: { component: '@', props: '=' },
  link(scope: any, el: JQLite) {
    const Component = registry[scope.component];
    const root: Root = createRoot(el[0]);
    const render = () => root.render(<Component {...scope.props} />);
    render();
    scope.$watch('props', render, true);        // push legacy state down
    scope.$on('$destroy', () => root.unmount()); // avoid leaks on view teardown
  },
})]);
```

```html
<react-mount component="UserTable" props="vm.userTableProps"></react-mount>
```

### Shared state out of `$scope`

```ts
// shared/session-store.ts — framework-agnostic
export const sessionStore = createStore<Session>(initial);   // zustand/vanilla or similar

// AngularJS side
angular.module('app').factory('Session', () => sessionStore);
// React side
const user = useStore(sessionStore, s => s.user);
```

### Guard rails against legacy growth

```js
// .eslintrc — block new AngularJS code in changed files
"no-restricted-imports": ["error", { "patterns": ["angular", "**/legacy/**"] }]
// CI: fail PRs that add lines under /legacy beyond bug-fix allowlist
```

> **Check yourself:** Why leaf-first rather than starting with the app shell, and what must be true before you flip routing ownership to React?

## Gotchas

- **The rewrite temptation.** Proposing a parallel rewrite signals inexperience with long-tail risk; show you know why it fails.
- **No exit date → permanent dual-stack.** The bridge becomes load-bearing; set a removal milestone and metric.
- **Two sources of truth for state.** Syncing `$scope` and React state both ways produces loops and stale views.
- **Leaks at the bridge.** Forgetting to unmount React roots on Angular `$destroy`.
- **Migrating by technology, not by value.** Moving stable internal screens first burns budget; follow churn and pain.
- **Visual inconsistency.** Mixed UIs without shared tokens look broken to users.
- **Ignoring tests.** Migrating untested code without a safety net means regressions are found by customers.
- **Bundle bloat unmanaged.** Ship both frameworks without code-splitting and watch performance regress.

## Follow-up Questions

**Q (High): Why not just rewrite it from scratch?**

Answer: Rewrites hold risk until the end, deliver no value for a long time, chase a moving target as the legacy app keeps evolving, and routinely lose undocumented behavior embedded in the old code. Incremental strangler migration ships continuously, can be paused or reprioritized, and each step is reversible. A rewrite is defensible only for small apps or when the legacy is truly unsalvageable and well-specified.

The trap: "cleaner" as the rationale — ignoring delivery risk and lost business logic.

**Q (High): How do React and AngularJS coexist on one page without two systems fighting?**

Answer: Mount React roots inside legacy-owned DOM via a bridge directive; legacy owns the surrounding view and passes props down, React emits callbacks up. Keep ownership boundaries clean (no React touching Angular-managed DOM and vice versa), unmount on `$destroy`, and move shared state to framework-agnostic stores so neither system depends on the other's internals.

The trap: two-way binding between `$scope` and React state, or letting jQuery mutate React-managed DOM.

**Q (High): How do you decide what to migrate first?**

Answer: Score by change frequency (where the team spends time), business criticality, coupling, and risk. Start with a high-churn, loosely coupled leaf to prove the pattern; avoid the most entangled core early; leave stable, rarely touched code last or retire it.

The trap: migrating in directory order or by "easiest first" with no business value.

**Q (Medium): How do you keep the team from sliding back into writing legacy code?**

Answer: Make the right thing easy and the wrong thing visible: a "new code in React" rule enforced by lint/CI, an exemplar and templates, and a shared component library. Track migration progress on a dashboard and budget migration capacity explicitly rather than hoping it happens "when we have time."

The trap: relying on a policy document with no enforcement or capacity.

**Q (Medium): How do you manage performance and bundle size during the dual-stack period?**

Answer: Code-split so React and AngularJS load only where needed, lazy-load legacy bundles for legacy routes, monitor bundle size and Web Vitals per route in CI, and avoid redundant libraries (e.g., two date libs). Accept a temporary cost but cap it with budgets.

The trap: ignoring the temporary regression until users complain.

**Q (Low): What does "done" look like?**

Answer: AngularJS removed from the production bundle, no active development in legacy, and the bridge deleted. If a small stable remainder isn't worth migrating, isolate it, own the security risk explicitly, or retire it — a conscious decision rather than neglect.

The trap: declaring victory at "80%" and leaving the bridge forever.

## Self-Assessment

- [ ] Can argue incremental over rewrite in under a minute with concrete risks
- [ ] Can compare the three seam options and pick one with reasons
- [ ] Can describe a React-in-AngularJS bridge including cleanup
- [ ] Can explain how shared state should be handled across frameworks
- [ ] Can lay out phased rollout with exit criteria
- [ ] Can say what "done" means and how to prevent permanent dual-stack

---
*Next: Migrating a CSR App to SSR for SEO — another incremental migration, but this time the constraint is rendering model and infrastructure rather than framework choice.*
