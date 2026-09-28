# Frontend Scenario Interview Prep

Senior frontend engineer scenario-based interview prep — 122 scenarios across 13 phases.

Companion to [Frontend Concepts Prep](../Frontend%20concepts%20prep) — that project builds the *mental models*; this one drills the *application of those models under interview conditions*: machine coding, debugging a broken component, designing a system on a whiteboard, walking an interviewer through a production incident.

---

## How This Works

Every scenario is framed the way an interviewer would actually say it — a prompt, not a lecture. The file then works through it the way a strong senior candidate would: clarifying questions first, a stated approach with trade-offs, a working solution, then the follow-up questions an interviewer throws in to probe depth.

**Workflow:** One scenario at a time, at your pace. Say "next" to continue in sequence or name any scenario to jump to it.

**Code examples:** Plain JavaScript for machine-coding scenarios (Phases 1–2), since that's what's expected on a whiteboard/CoderPad. React + TypeScript from Phase 3 onward, since that's the realistic shape of the codebase these bugs and designs live in.

**Scenario labels:** Each scenario is tagged with its interview importance — `High` (asked in most senior loops for this category), `Medium` (common but company-dependent), `Low` (asked at some companies, good depth signal when it comes up). Within each phase, scenarios are ordered High → Medium → Low.

### Every scenario file contains

| Section | Purpose |
|---|---|
| **Quick Reference** | The core technique/decision at a glance — re-anchor in 10 seconds |
| **The Scenario** | The prompt, phrased as an interviewer would actually say it |
| **Clarifying Questions** | What a senior candidate asks before writing a line of code |
| **Approach & Trade-offs** | The reasoning a senior candidate narrates out loud |
| **Solution** | Working code / design, built incrementally where it helps |
| **Follow-up Questions** | The interviewer's probes, ordered High → Medium → Low, with answers + the trap |
| **Gotchas** | Where candidates lose points even with working code |
| **Self-Assessment** | End-of-file checklist — what you own vs. what to revisit |

---

## Progress

| Phase | Scenarios | Area | Why This Matters at Senior Level |
|-------|-----------|------|-----------------------------------|
| [Phase 1](#phase-1--javascript-machine-coding--async-mechanics) | 16 | JavaScript machine coding & async mechanics | The universal filter — asked in nearly every loop regardless of stack |
| [Phase 2](#phase-2--component-machine-coding) | 16 | Component machine coding | Build-a-widget rounds; tests API design instinct, not just syntax |
| [Phase 3](#phase-3--react-debugging--behavioral-scenarios) | 10 | React debugging & behavioral scenarios | The #1 senior differentiator — anyone can write code, fewer can debug someone else's |
| [Phase 4](#phase-4--frontend-system-design) | 15 | Frontend system design | Standard for senior/staff loops — architecture instinct at scale |
| [Phase 5](#phase-5--performance-debugging-scenarios) | 8 | Performance debugging scenarios | Systematic diagnosis under ambiguity, not tool trivia |
| [Phase 6](#phase-6--state-management--architecture-trade-offs) | 8 | State management & architecture trade-offs | "Where does this state live" — the question that separates levels |
| [Phase 7](#phase-7--networking--data-layer-scenarios) | 8 | Networking & data layer scenarios | Real-time, caching, and resilience under flaky networks |
| [Phase 8](#phase-8--css--layout-debugging-scenarios) | 8 | CSS & layout debugging scenarios | Common at generalist / product companies, still shows up for seniors |
| [Phase 9](#phase-9--accessibility-scenarios) | 7 | Accessibility scenarios | Increasingly weighted as a senior-vs-junior differentiator |
| [Phase 10](#phase-10--security-scenarios) | 6 | Security scenarios | Trust signal — can you be handed prod without a security review babysitting you |
| [Phase 11](#phase-11--testing-strategy-scenarios) | 6 | Testing strategy scenarios | Process judgment — what to test, at which layer, and why |
| [Phase 12](#phase-12--architecture-migration--engineering-judgment) | 8 | Architecture, migration & engineering judgment | Staff-adjacent scenarios — reserved for senior+ loops specifically |
| [Phase 13](#phase-13--ai-integrated-frontend-scenarios) | 6 | AI-integrated frontend scenarios | Emerging in 2026 loops — streaming UI, LLM integration patterns |
| **Total** | **122** | | |

---

## Phase 1 — JavaScript Machine Coding & Async Mechanics

> The universal filter. Closures, timers, and promise composition — asked regardless of what framework the job actually uses.

- [x] [Debounce — Leading/Trailing/Cancel](phase-01-js-machine-coding-async/01-debounce-leading-trailing-cancel.md)
- [x] [Throttle — Leading/Trailing](phase-01-js-machine-coding-async/02-throttle-leading-trailing.md)
- [x] [Deep Clone — Circular References & Special Types](phase-01-js-machine-coding-async/03-deep-clone-circular-refs.md)
- [x] [Deep Equal](phase-01-js-machine-coding-async/04-deep-equal.md)
- [x] [Curry, Compose & Pipe](phase-01-js-machine-coding-async/05-curry-compose-pipe.md)
- [x] [Memoize with TTL & Cache Eviction](phase-01-js-machine-coding-async/06-memoize-with-ttl-eviction.md)
- [x] [Custom EventEmitter (on/off/once/emit)](phase-01-js-machine-coding-async/07-custom-event-emitter.md)
- [x] [Promise.all / allSettled / race / any Polyfills](phase-01-js-machine-coding-async/08-promise-all-allsettled-race-any-polyfills.md)
- [x] [Custom Promise From Scratch](phase-01-js-machine-coding-async/09-custom-promise-from-scratch.md)
- [x] [Retry With Exponential Backoff](phase-01-js-machine-coding-async/10-retry-with-exponential-backoff.md)
- [x] [LRU Cache](phase-01-js-machine-coding-async/11-lru-cache.md)
- [x] [Concurrency-limited Task Queue (Promise Pool)](phase-01-js-machine-coding-async/12-concurrency-limited-task-queue-promise-pool.md)
- [x] [Pub/Sub System](phase-01-js-machine-coding-async/13-pub-sub-system.md)
- [x] [Array Method Polyfills: map/filter/reduce/flat](phase-01-js-machine-coding-async/14-array-polyfills-map-filter-reduce-flat.md)
- [x] [call/apply/bind Polyfills](phase-01-js-machine-coding-async/15-call-apply-bind-polyfills.md)
- [x] [Event Loop Output Prediction — Tricky Async](phase-01-js-machine-coding-async/16-event-loop-output-prediction-tricky-async.md)

---

## Phase 2 — Component Machine Coding

> "Build this widget in 40 minutes." Tests component API design and edge-case coverage, not just markup.

- [x] [Autocomplete / Typeahead — Debounced & Cancellable](phase-02-component-machine-coding/01-autocomplete-typeahead-debounced-cancellable.md)
- [x] [Infinite Scroll List](phase-02-component-machine-coding/02-infinite-scroll-list.md)
- [x] [Virtualized List (Windowing) From Scratch](phase-02-component-machine-coding/03-virtualized-list-windowing-from-scratch.md)
- [x] [Accessible Modal With Focus Trap](phase-02-component-machine-coding/04-accessible-modal-focus-trap.md)
- [x] [Accessible Tabs — Keyboard Navigation](phase-02-component-machine-coding/05-accessible-tabs-keyboard-nav.md)
- [x] [Accordion Component](phase-02-component-machine-coding/06-accordion-component.md)
- [x] [Accessible Combobox / Dropdown](phase-02-component-machine-coding/07-accessible-combobox-dropdown.md)
- [x] [Multi-step Form Wizard With Validation](phase-02-component-machine-coding/08-multi-step-form-wizard-validation.md)
- [x] [Data Table — Sort, Filter, Paginate](phase-02-component-machine-coding/09-data-table-sort-filter-paginate.md)
- [x] [Drag-and-Drop Sortable List](phase-02-component-machine-coding/10-drag-and-drop-sortable-list.md)
- [x] [Toast / Notification Queue System](phase-02-component-machine-coding/11-toast-notification-queue-system.md)
- [x] [Star Rating Component](phase-02-component-machine-coding/12-star-rating-component.md)
- [x] [OTP / PIN Input](phase-02-component-machine-coding/13-otp-pin-input.md)
- [x] [Image Carousel / Gallery](phase-02-component-machine-coding/14-image-carousel-gallery.md)
- [x] [Undo/Redo Stack for an Editor UI](phase-02-component-machine-coding/15-undo-redo-stack-editor.md)
- [x] [Nested Comments — Recursive Tree Rendering](phase-02-component-machine-coding/16-nested-comments-recursive-tree.md)

---

## Phase 3 — React Debugging & Behavioral Scenarios

> "Here's a component. It's misbehaving. Find out why." The single biggest senior-vs-junior signal.

- [x] [Why Is This Component Re-rendering Constantly?](phase-03-react-debugging-scenarios/01-why-is-this-re-rendering.md)
- [x] [Stale Closure in useEffect/useCallback](phase-03-react-debugging-scenarios/02-stale-closure-useeffect-usecallback.md)
- [x] [Race Condition in Fetch (Autocomplete Overwrite Bug)](phase-03-react-debugging-scenarios/03-race-condition-fetch-autocomplete.md)
- [x] [Memory Leak From Uncleaned Subscriptions](phase-03-react-debugging-scenarios/04-memory-leak-uncleaned-subscriptions.md)
- [x] [Context Causing App-wide Re-renders](phase-03-react-debugging-scenarios/05-context-causing-app-wide-re-renders.md)
- [x] [Key Prop Misuse — State Bleeding Between List Items](phase-03-react-debugging-scenarios/06-key-prop-misuse-state-bleed.md)
- [x] [Controlled vs. Uncontrolled Input Bug](phase-03-react-debugging-scenarios/07-controlled-vs-uncontrolled-input-bug.md)
- [x] [Infinite Render Loop](phase-03-react-debugging-scenarios/08-infinite-render-loop.md)
- [x] [Prop Drilling Causing Stale Sibling State](phase-03-react-debugging-scenarios/09-prop-drilling-stale-sibling-state.md)
- [x] [Error Boundary Not Catching an Error — Why](phase-03-react-debugging-scenarios/10-error-boundary-not-catching-error.md)

---

## Phase 4 — Frontend System Design

> The whiteboard round. Requirements gathering, component/data architecture, and the trade-offs at scale.

- [x] [Design a News Feed (Facebook/LinkedIn-style)](phase-04-frontend-system-design/01-design-a-news-feed.md)
- [x] [Design an Autocomplete / Search-as-you-type System](phase-04-frontend-system-design/02-design-autocomplete-search-system.md)
- [x] [Design an E-commerce Product Listing + Filters Page](phase-04-frontend-system-design/03-design-ecommerce-plp-filters.md)
- [x] [Design a Chat Application (WhatsApp Web-style)](phase-04-frontend-system-design/04-design-chat-application.md)
- [x] [Design a Notification System (In-app + Push)](phase-04-frontend-system-design/05-design-notification-system.md)
- [x] [Design an Image/Video Gallery With Lazy Loading](phase-04-frontend-system-design/06-design-image-video-gallery.md)
- [x] [Design a Collaborative Document Editor (OT/CRDT Basics)](phase-04-frontend-system-design/07-design-collaborative-document-editor.md)
- [x] [Design a Live Comments/Reactions Feed](phase-04-frontend-system-design/08-design-live-comments-reactions-feed.md)
- [x] [Design a Resumable, Chunked File Uploader](phase-04-frontend-system-design/09-design-resumable-chunked-file-uploader.md)
- [x] [Design a Configurable Dashboard With Widgets](phase-04-frontend-system-design/10-design-configurable-dashboard-widgets.md)
- [x] [Design a Ticket/Seat Booking UI](phase-04-frontend-system-design/11-design-ticket-seat-booking-ui.md)
- [x] [Design a Polling/Voting Widget](phase-04-frontend-system-design/12-design-polling-voting-widget.md)
- [x] [Design an Instagram Stories-style Component](phase-04-frontend-system-design/13-design-stories-component.md)
- [x] [Design a Schema-driven Form Builder](phase-04-frontend-system-design/14-design-schema-driven-form-builder.md)
- [x] [Design a Component Library / Design System From Scratch](phase-04-frontend-system-design/15-design-a-component-library-design-system.md)

---

## Phase 5 — Performance Debugging Scenarios

> "Here's a metric that's bad. Walk me through your diagnosis." Systematic process over tool trivia.

- [x] [Diagnosing Poor LCP (e.g., "LCP is 4.2s")](phase-05-performance-debugging/01-diagnosing-poor-lcp.md)
- [x] [Janky Scroll/Animation — Find and Fix](phase-05-performance-debugging/02-janky-scroll-animation.md)
- [x] [Bundle Size Regression After a Release](phase-05-performance-debugging/03-bundle-size-regression-after-release.md)
- [x] [Production Memory Leak Triage](phase-05-performance-debugging/04-production-memory-leak-triage.md)
- [x] [Slow Initial Load on 3G / Low-end Device](phase-05-performance-debugging/05-slow-load-low-end-device-3g.md)
- [x] [Redundant Network Requests on a Page](phase-05-performance-debugging/06-redundant-network-requests-on-a-page.md)
- [x] [Long Tasks Blocking the Main Thread](phase-05-performance-debugging/07-long-tasks-blocking-main-thread.md)
- [x] [High INP / Unresponsive Interactions](phase-05-performance-debugging/08-high-inp-unresponsive-interactions.md)

---

## Phase 6 — State Management & Architecture Trade-offs

> "Where does this state live?" — the question that reveals whether you have architecture instincts or just API knowledge.

- [x] [Where Does This State Live? (Server/Client/URL/Form Sort)](phase-06-state-architecture-tradeoffs/01-where-does-this-state-live.md)
- [x] [State Design: Filters + Saved Views + Real-time Counters](phase-06-state-architecture-tradeoffs/02-state-design-filters-saved-views-realtime-counters.md)
- [x] [Optimistic Update With Rollback](phase-06-state-architecture-tradeoffs/03-optimistic-update-with-rollback.md)
- [x] [Undo/Redo — Architecture Decision](phase-06-state-architecture-tradeoffs/04-undo-redo-architecture-decision.md)
- [x] [Cross-tab State Sync](phase-06-state-architecture-tradeoffs/05-cross-tab-state-sync.md)
- [x] [Conflict Resolution for Concurrent Edits](phase-06-state-architecture-tradeoffs/06-conflict-resolution-concurrent-edits.md)
- [x] [Redux → Lightweight State — Migration Trade-offs](phase-06-state-architecture-tradeoffs/07-redux-to-lightweight-state-migration-tradeoffs.md)
- [x] [Feature-flag-driven Rollout Design](phase-06-state-architecture-tradeoffs/08-feature-flag-driven-rollout-design.md)

---

## Phase 7 — Networking & Data Layer Scenarios

> Data fetching, caching, and resilience when the network is slow, flaky, or adversarial.

- [x] [Data-fetching Layer for N Related Resources (Waterfalls → Parallelization)](phase-07-networking-data-layer/01-data-fetching-layer-for-related-resources.md)
- [x] [Cache Invalidation Strategy](phase-07-networking-data-layer/02-cache-invalidation-strategy.md)
- [x] [Pagination Data Model: Cursor vs. Offset](phase-07-networking-data-layer/03-pagination-data-model-cursor-vs-offset.md)
- [x] [WebSocket Reconnection & Backoff](phase-07-networking-data-layer/04-websocket-reconnection-backoff.md)
- [x] [Offline-first Sync With Conflict Resolution](phase-07-networking-data-layer/05-offline-first-sync-conflict-resolution.md)
- [x] [Streaming LLM Response UI: SSE vs. WebSocket](phase-07-networking-data-layer/06-streaming-llm-response-ui-sse-vs-websocket.md)
- [x] [Graceful Degradation for a Flaky Third-party API](phase-07-networking-data-layer/07-graceful-degradation-flaky-third-party-api.md)
- [x] [GraphQL Client-side N+1 / Over-fetching](phase-07-networking-data-layer/08-graphql-client-n-plus-1-overfetching.md)

---

## Phase 8 — CSS & Layout Debugging Scenarios

> "It looks fine on my machine." Layout bugs that only show up in specific browsers, viewports, or content states.

- [x] [Holy Grail Layout](phase-08-css-layout-debugging/01-holy-grail-layout.md)
- [x] [Sticky Header Broken on Mobile Safari](phase-08-css-layout-debugging/02-sticky-header-broken-mobile-safari.md)
- [x] [CLS From Late-loading Images & Fonts](phase-08-css-layout-debugging/03-cls-from-late-loading-images-fonts.md)
- [x] [Responsive Grid for an Unknown Item Count](phase-08-css-layout-debugging/04-responsive-grid-unknown-item-count.md)
- [x] [Z-index / Stacking Context Bug](phase-08-css-layout-debugging/05-z-index-stacking-context-bug.md)
- [x] [RTL Layout Breaking](phase-08-css-layout-debugging/06-rtl-layout-breaking.md)
- [x] [Nested Scroll Container Overflow Trap](phase-08-css-layout-debugging/07-nested-scroll-container-overflow-trap.md)
- [x] [Dark Mode & Print Stylesheet Edge Cases](phase-08-css-layout-debugging/08-dark-mode-print-stylesheet-edge-cases.md)

---

## Phase 9 — Accessibility Scenarios

> Real-world a11y failures and the fixes — plus the softer scenario of pushing back on design/product.

- [x] [Accessible Modal — Full Scenario Walkthrough](phase-09-accessibility-scenarios/01-accessible-modal-scenario.md)
- [x] [Accessible Combobox — Full Scenario Walkthrough](phase-09-accessibility-scenarios/02-accessible-combobox-scenario.md)
- [x] [Live Region Announcements for Async Updates](phase-09-accessibility-scenarios/03-live-region-async-announcements.md)
- [x] [Keyboard Trap Bug — Find and Fix](phase-09-accessibility-scenarios/04-keyboard-trap-bug-fix.md)
- [x] [Designer Pushback on Color Contrast — How You Handle It](phase-09-accessibility-scenarios/05-designer-pushback-on-color-contrast.md)
- [x] [Auditing an Existing App for Accessibility](phase-09-accessibility-scenarios/06-auditing-an-app-for-accessibility.md)
- [x] [Accessible Drag-and-Drop Alternative for Keyboard Users](phase-09-accessibility-scenarios/07-accessible-drag-and-drop-alternative.md)

---

## Phase 10 — Security Scenarios

> "You found this in production. What do you do?" Incident response as much as prevention.

- [ ] [Stored XSS Found in Production](phase-10-security-scenarios/01-stored-xss-found-in-production.md)
- [ ] [Auth Token Storage — a Breach Scenario](phase-10-security-scenarios/02-auth-token-storage-breach-scenario.md)
- [ ] [Open Redirect Found in Code Review](phase-10-security-scenarios/03-open-redirect-in-code-review.md)
- [ ] [Third-party Script Breaks Your CSP](phase-10-security-scenarios/04-third-party-script-breaks-csp.md)
- [ ] [Clickjacking Risk on an Embeddable Widget](phase-10-security-scenarios/05-clickjacking-embeddable-widget.md)
- [ ] [Dependency With a Known CVE in Production](phase-10-security-scenarios/06-dependency-cve-in-production-response.md)

---

## Phase 11 — Testing Strategy Scenarios

> Not "do you write tests" but "what do you test, where, and why" — plus triage of tests that already exist.

- [ ] [Flaky E2E Test — Triage and Fix](phase-11-testing-strategy-scenarios/01-flaky-e2e-test-triage.md)
- [ ] ["How Would You Test This Component?" Exercise](phase-11-testing-strategy-scenarios/02-how-would-you-test-this-component.md)
- [ ] [Testing a Race-condition-prone Async Component](phase-11-testing-strategy-scenarios/03-testing-a-race-condition-prone-component.md)
- [ ] [Mocking the Network Layer for Integration Tests (MSW)](phase-11-testing-strategy-scenarios/04-mocking-network-layer-msw.md)
- [ ] [Visual Regression False Positives — Handling Them](phase-11-testing-strategy-scenarios/05-visual-regression-false-positives.md)
- [ ] [Testing an Accessibility Requirement](phase-11-testing-strategy-scenarios/06-testing-accessibility-requirements.md)

---

## Phase 12 — Architecture, Migration & Engineering Judgment

> Staff-adjacent scenarios. Reserved for senior+ loops, but high-signal when they come up — this is where "senior" gets proven.

- [ ] [Migrating a Legacy jQuery/AngularJS App to React — Incrementally](phase-12-architecture-migration-judgment/01-migrating-legacy-jquery-angularjs-to-react.md)
- [ ] [Migrating a CSR App to SSR for SEO](phase-12-architecture-migration-judgment/02-migrating-csr-to-ssr-for-seo.md)
- [ ] [Introducing TypeScript to a Large Untyped JS Codebase](phase-12-architecture-migration-judgment/03-introducing-typescript-to-large-js-codebase.md)
- [ ] [Monolith Frontend to Micro-frontends — When and How](phase-12-architecture-migration-judgment/04-monolith-to-micro-frontends.md)
- [ ] [Choosing a Rendering Strategy for a New Product](phase-12-architecture-migration-judgment/05-choosing-rendering-strategy-for-new-product.md)
- [ ] [Handling a Breaking API Change From Another Team](phase-12-architecture-migration-judgment/06-handling-a-breaking-api-change-from-another-team.md)
- [ ] [Rolling Out a Risky Refactor Without Breaking Prod](phase-12-architecture-migration-judgment/07-rolling-out-risky-refactor-without-breaking-prod.md)
- [ ] [Reviewing a PR With a Subtle Race Condition](phase-12-architecture-migration-judgment/08-reviewing-a-pr-with-a-subtle-race-condition.md)

---

## Phase 13 — AI-integrated Frontend Scenarios

> The newest category in senior loops — streaming UI and LLM-integration patterns, tested as system design or debugging.

- [ ] [Streaming an LLM Chat Response Into the UI](phase-13-ai-integrated-frontend/01-streaming-llm-chat-response-into-ui.md)
- [ ] [Choosing SSE vs. WebSocket for a Real-time Feature](phase-13-ai-integrated-frontend/02-choosing-sse-vs-websocket-for-realtime-feature.md)
- [ ] [Rendering Streaming Markdown/Code Safely](phase-13-ai-integrated-frontend/03-rendering-streaming-markdown-safely.md)
- [ ] [Cancelling a Long-running AI Request Cleanly](phase-13-ai-integrated-frontend/04-cancelling-a-long-running-ai-request.md)
- [ ] [Designing UI for Agentic / Tool-use Flows](phase-13-ai-integrated-frontend/05-designing-ui-for-agentic-tool-use-flows.md)
- [ ] [Handling AI Latency and Failure Gracefully in the UI](phase-13-ai-integrated-frontend/06-handling-ai-latency-and-failure-gracefully.md)
