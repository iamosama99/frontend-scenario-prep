# Project Context — Frontend Scenario Interview Prep

> Hand this file to any new conversation to continue exactly where we left off.

---

## What This Project Is

A personal frontend engineering **scenario-based** interview prep repository for a **senior frontend engineer**. Companion to [`Frontend concepts prep`](../Frontend%20concepts%20prep/CONTEXT.md), which covers *what things are and why* — this project drills *applying that knowledge under interview conditions*: machine coding on a shared editor, debugging someone else's broken component, designing a system on a whiteboard, narrating how you'd handle a production incident.

The goal is 122 scenarios across 13 phases, one scenario at a time, at the learner's pace. Quality over speed — there is no rush.

Phases are ordered by how frequently they show up across senior frontend loops and how strong a signal they are — see the rationale in the README's Progress table. Within each phase, scenarios are ordered High → Medium → Low importance.

Each file is a standalone, self-contained walkthrough of one scenario — written the way a strong senior candidate would actually work through it out loud, not as a model-answer transcript to memorize.

---

## Who You're Teaching

- Senior frontend engineer level — use correct terminology freely, don't oversimplify
- Already has the conceptual grounding from `Frontend concepts prep`; this project is about *performance under interview conditions* — narrating trade-offs, handling ambiguity, writing correct code live
- Wants to see the reasoning a senior candidate narrates out loud, not just a finished answer — the clarifying questions matter as much as the solution
- No caps on length, no caps on number of follow-up questions — go as deep as the scenario warrants
- Code examples: **plain JavaScript for Phases 1–2** (machine coding is almost always plain JS/TS on a shared editor), **React + TypeScript from Phase 3 onward**

---

## How to Continue

When the user says **"next"**, generate the next scenario in sequence (see progress tracker below).
When the user names a **specific scenario or phase**, jump directly to it.

Write the scenario as a markdown file and save it to the correct path (listed per scenario below).
After writing, commit it with `git add <file> && git commit`.

---

## Content Format — Follow This For Every Scenario

Each scenario file must follow this structure. Do not skip sections that apply. There is no length limit.

```
# [Scenario Name]

## Quick Reference

A small table (2–5 rows) or tight bullet list placed immediately after the title.
Maps the core technique/decision → mechanism → why it's the right call, at a glance.
A reader returning after weeks should be able to re-anchor in under 10 seconds
without re-reading the whole file.

## The Scenario

Phrase the prompt exactly as an interviewer would actually say it — first person,
conversational, slightly underspecified on purpose (real prompts are underspecified;
that's the point being tested).

## Clarifying Questions

What a senior candidate asks before writing a line of code or drawing a single box.
For machine coding: edge cases, expected inputs, environment constraints.
For system design: scale, users, devices, must-have vs. nice-to-have features.
For debugging: repro steps, when it started, what changed, scope of impact.
Explain WHY each question matters — what wrong assumption it prevents.

## Approach & Trade-offs

The reasoning a senior candidate narrates out loud before/while coding or designing.
State the options considered and why one was chosen over the others. This is the
section that most separates senior from mid-level — showing the trade-off, not just
picking an answer.

## Solution

The actual code (machine coding / debugging) or design (system design) — built
incrementally where that helps the reader follow the reasoning, not dropped as one
finished block with no narration. Include comments only where genuinely non-obvious.

> **Check yourself:** [A question testing whether the reader could reproduce this
> approach unprompted, not just recognize it.]

## [Other sections as needed — use judgment]
E.g., for debugging scenarios: "Root Cause", "The Fix", "How to Prevent This Class of Bug".
For system design: "Data Model", "Component Breakdown", "Scaling Considerations".
For architecture/migration: "Rollout Plan", "Risk Mitigation".
Name sections for what they actually cover. Only create a section if it adds value.

## Gotchas

Where candidates lose points even with technically working code/design — the missed
edge case, the trade-off not mentioned, the follow-up that reveals a shallow answer.
No padding. If there are two, write two. If there are six, write six.
Every gotcha that's interview-worthy must also appear as a follow-up question below.

## Follow-up Questions

Each question has an importance label: `High` (asked in almost every senior interview
that reaches this scenario), `Medium` (common but company-dependent), or `Low`
(good depth signal, asked less often).

ORDER QUESTIONS from most to least important: High → Medium → Low.

**Q (Medium): [Question text]**

Answer: [What a senior candidate would say. Be complete — this is the reference answer.]

The trap: [What the interviewer is watching for. What weaker candidates say or miss.]

[Repeat for every follow-up the scenario genuinely warrants. No artificial cap.]

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.
Leave unchecked anything you'd need to read to answer — that's what to revisit.

- [ ] [Concrete capability: "Can state the approach and trade-off in one breath, unprompted"]
- [ ] [Concrete capability: "Can write/sketch the solution from memory"]
- [ ] [Concrete capability: "Can answer the top 2 follow-ups without notes"]
- [ ] [Add 2–4 more items specific to the scenario — aim for 4–6 total]

---
*Next: [Next scenario name] — [One sentence on why it follows naturally, or what new
angle it adds if the connection isn't obvious.]*
```

---

## Style Rules

- Frame every scenario as the interviewer would actually phrase it — conversational, slightly underspecified
- Clarifying questions come before any code or design — never skip straight to a solution
- Always narrate the trade-off, not just the chosen answer — that narration is the actual senior signal
- Use plain language but exact terminology — never dumb down, never use vague words when a precise term exists
- Write "Approach & Trade-offs" as a senior engineer thinking out loud, not as documentation
- Connect scenarios to each other and to `Frontend concepts prep` topics when relevant (link by name)
- Never pad. If a section doesn't apply, don't invent content to fill it
- Follow-up questions: write as many as the scenario genuinely warrants. Include the answer AND the trap

---

## Repository Structure

```
/Users/osama/Developer/Projects/frontend scenario prep/
├── CONTEXT.md                                    ← this file
├── README.md                                     ← full scenario index with checkboxes
├── .gitignore
├── phase-01-js-machine-coding-async/             # plain JS
├── phase-02-component-machine-coding/            # plain JS / TS
├── phase-03-react-debugging-scenarios/           # React + TS ← switch to React+TS here
├── phase-04-frontend-system-design/              # React + TS
├── phase-05-performance-debugging/               # React + TS
├── phase-06-state-architecture-tradeoffs/        # React + TS
├── phase-07-networking-data-layer/               # React + TS
├── phase-08-css-layout-debugging/                # HTML/CSS (+ React where relevant)
├── phase-09-accessibility-scenarios/             # React + TS
├── phase-10-security-scenarios/                  # React + TS
├── phase-11-testing-strategy-scenarios/          # React + TS
├── phase-12-architecture-migration-judgment/     # prose-heavy, code where relevant
└── phase-13-ai-integrated-frontend/              # React + TS
```

---

## Progress Tracker

Legend: ✅ Done | ⬜ Not started | 👉 **Next up**

### Phase 1 — JavaScript Machine Coding & Async Mechanics (16 scenarios)

| # | Scenario | File | Status |
|---|----------|------|--------|
| 1 | Debounce — Leading/Trailing/Cancel | `phase-01-js-machine-coding-async/01-debounce-leading-trailing-cancel.md` | ✅ |
| 2 | Throttle — Leading/Trailing | `phase-01-js-machine-coding-async/02-throttle-leading-trailing.md` | ✅ |
| 3 | Deep Clone — Circular References & Special Types | `phase-01-js-machine-coding-async/03-deep-clone-circular-refs.md` | ✅ |
| 4 | Deep Equal | `phase-01-js-machine-coding-async/04-deep-equal.md` | ✅ |
| 5 | Curry, Compose & Pipe | `phase-01-js-machine-coding-async/05-curry-compose-pipe.md` | ✅ |
| 6 | Memoize With TTL & Cache Eviction | `phase-01-js-machine-coding-async/06-memoize-with-ttl-eviction.md` | ✅ |
| 7 | Custom EventEmitter (on/off/once/emit) | `phase-01-js-machine-coding-async/07-custom-event-emitter.md` | ✅ |
| 8 | Promise.all / allSettled / race / any Polyfills | `phase-01-js-machine-coding-async/08-promise-all-allsettled-race-any-polyfills.md` | ✅ |
| 9 | Custom Promise From Scratch | `phase-01-js-machine-coding-async/09-custom-promise-from-scratch.md` | ✅ |
| 10 | Retry With Exponential Backoff | `phase-01-js-machine-coding-async/10-retry-with-exponential-backoff.md` | ✅ |
| 11 | LRU Cache | `phase-01-js-machine-coding-async/11-lru-cache.md` | ✅ |
| 12 | Concurrency-limited Task Queue (Promise Pool) | `phase-01-js-machine-coding-async/12-concurrency-limited-task-queue-promise-pool.md` | ✅ |
| 13 | Pub/Sub System | `phase-01-js-machine-coding-async/13-pub-sub-system.md` | ✅ |
| 14 | Array Method Polyfills: map/filter/reduce/flat | `phase-01-js-machine-coding-async/14-array-polyfills-map-filter-reduce-flat.md` | ✅ |
| 15 | call/apply/bind Polyfills | `phase-01-js-machine-coding-async/15-call-apply-bind-polyfills.md` | ✅ |
| 16 | Event Loop Output Prediction — Tricky Async | `phase-01-js-machine-coding-async/16-event-loop-output-prediction-tricky-async.md` | ✅ |

### Phase 2 — Component Machine Coding (16 scenarios)

| # | Scenario | File | Status |
|---|----------|------|--------|
| 1 | Autocomplete / Typeahead — Debounced & Cancellable | `phase-02-component-machine-coding/01-autocomplete-typeahead-debounced-cancellable.md` | ✅ |
| 2 | Infinite Scroll List | `phase-02-component-machine-coding/02-infinite-scroll-list.md` | ✅ |
| 3 | Virtualized List (Windowing) From Scratch | `phase-02-component-machine-coding/03-virtualized-list-windowing-from-scratch.md` | ✅ |
| 4 | Accessible Modal With Focus Trap | `phase-02-component-machine-coding/04-accessible-modal-focus-trap.md` | ✅ |
| 5 | Accessible Tabs — Keyboard Navigation | `phase-02-component-machine-coding/05-accessible-tabs-keyboard-nav.md` | ✅ |
| 6 | Accordion Component | `phase-02-component-machine-coding/06-accordion-component.md` | ✅ |
| 7 | Accessible Combobox / Dropdown | `phase-02-component-machine-coding/07-accessible-combobox-dropdown.md` | ✅ |
| 8 | Multi-step Form Wizard With Validation | `phase-02-component-machine-coding/08-multi-step-form-wizard-validation.md` | ✅ |
| 9 | Data Table — Sort, Filter, Paginate | `phase-02-component-machine-coding/09-data-table-sort-filter-paginate.md` | ✅ |
| 10 | Drag-and-Drop Sortable List | `phase-02-component-machine-coding/10-drag-and-drop-sortable-list.md` | ✅ |
| 11 | Toast / Notification Queue System | `phase-02-component-machine-coding/11-toast-notification-queue-system.md` | ✅ |
| 12 | Star Rating Component | `phase-02-component-machine-coding/12-star-rating-component.md` | ✅ |
| 13 | OTP / PIN Input | `phase-02-component-machine-coding/13-otp-pin-input.md` | ✅ |
| 14 | Image Carousel / Gallery | `phase-02-component-machine-coding/14-image-carousel-gallery.md` | ✅ |
| 15 | Undo/Redo Stack for an Editor UI | `phase-02-component-machine-coding/15-undo-redo-stack-editor.md` | ✅ |
| 16 | Nested Comments — Recursive Tree Rendering | `phase-02-component-machine-coding/16-nested-comments-recursive-tree.md` | ✅ |

### Phase 3 — React Debugging & Behavioral Scenarios (10 scenarios) — React + TS starts here

| # | Scenario | File | Status |
|---|----------|------|--------|
| 1 | Why Is This Component Re-rendering Constantly? | `phase-03-react-debugging-scenarios/01-why-is-this-re-rendering.md` | ✅ |
| 2 | Stale Closure in useEffect/useCallback | `phase-03-react-debugging-scenarios/02-stale-closure-useeffect-usecallback.md` | ✅ |
| 3 | Race Condition in Fetch (Autocomplete Overwrite Bug) | `phase-03-react-debugging-scenarios/03-race-condition-fetch-autocomplete.md` | ✅ |
| 4 | Memory Leak From Uncleaned Subscriptions | `phase-03-react-debugging-scenarios/04-memory-leak-uncleaned-subscriptions.md` | ✅ |
| 5 | Context Causing App-wide Re-renders | `phase-03-react-debugging-scenarios/05-context-causing-app-wide-re-renders.md` | ✅ |
| 6 | Key Prop Misuse — State Bleeding Between List Items | `phase-03-react-debugging-scenarios/06-key-prop-misuse-state-bleed.md` | ✅ |
| 7 | Controlled vs. Uncontrolled Input Bug | `phase-03-react-debugging-scenarios/07-controlled-vs-uncontrolled-input-bug.md` | ✅ |
| 8 | Infinite Render Loop | `phase-03-react-debugging-scenarios/08-infinite-render-loop.md` | ✅ |
| 9 | Prop Drilling Causing Stale Sibling State | `phase-03-react-debugging-scenarios/09-prop-drilling-stale-sibling-state.md` | ✅ |
| 10 | Error Boundary Not Catching an Error — Why | `phase-03-react-debugging-scenarios/10-error-boundary-not-catching-error.md` | ✅ |

### Phase 4 — Frontend System Design (15 scenarios)

| # | Scenario | File | Status |
|---|----------|------|--------|
| 1 | Design a News Feed (Facebook/LinkedIn-style) | `phase-04-frontend-system-design/01-design-a-news-feed.md` | ✅ |
| 2 | Design an Autocomplete / Search-as-you-type System | `phase-04-frontend-system-design/02-design-autocomplete-search-system.md` | ✅ |
| 3 | Design an E-commerce Product Listing + Filters Page | `phase-04-frontend-system-design/03-design-ecommerce-plp-filters.md` | ✅ |
| 4 | Design a Chat Application (WhatsApp Web-style) | `phase-04-frontend-system-design/04-design-chat-application.md` | ✅ |
| 5 | Design a Notification System (In-app + Push) | `phase-04-frontend-system-design/05-design-notification-system.md` | ✅ |
| 6 | Design an Image/Video Gallery With Lazy Loading | `phase-04-frontend-system-design/06-design-image-video-gallery.md` | ✅ |
| 7 | Design a Collaborative Document Editor (OT/CRDT Basics) | `phase-04-frontend-system-design/07-design-collaborative-document-editor.md` | ✅ |
| 8 | Design a Live Comments/Reactions Feed | `phase-04-frontend-system-design/08-design-live-comments-reactions-feed.md` | ✅ |
| 9 | Design a Resumable, Chunked File Uploader | `phase-04-frontend-system-design/09-design-resumable-chunked-file-uploader.md` | ✅ |
| 10 | Design a Configurable Dashboard With Widgets | `phase-04-frontend-system-design/10-design-configurable-dashboard-widgets.md` | ✅ |
| 11 | Design a Ticket/Seat Booking UI | `phase-04-frontend-system-design/11-design-ticket-seat-booking-ui.md` | ✅ |
| 12 | Design a Polling/Voting Widget | `phase-04-frontend-system-design/12-design-polling-voting-widget.md` | ✅ |
| 13 | Design an Instagram Stories-style Component | `phase-04-frontend-system-design/13-design-stories-component.md` | ✅ |
| 14 | Design a Schema-driven Form Builder | `phase-04-frontend-system-design/14-design-schema-driven-form-builder.md` | ✅ |
| 15 | Design a Component Library / Design System From Scratch | `phase-04-frontend-system-design/15-design-a-component-library-design-system.md` | ✅ |

### Phase 5 — Performance Debugging Scenarios (8 scenarios)

| # | Scenario | File | Status |
|---|----------|------|--------|
| 1 | Diagnosing Poor LCP (e.g., "LCP is 4.2s") | `phase-05-performance-debugging/01-diagnosing-poor-lcp.md` | ✅ |
| 2 | Janky Scroll/Animation — Find and Fix | `phase-05-performance-debugging/02-janky-scroll-animation.md` | ✅ |
| 3 | Bundle Size Regression After a Release | `phase-05-performance-debugging/03-bundle-size-regression-after-release.md` | ✅ |
| 4 | Production Memory Leak Triage | `phase-05-performance-debugging/04-production-memory-leak-triage.md` | ✅ |
| 5 | Slow Initial Load on 3G / Low-end Device | `phase-05-performance-debugging/05-slow-load-low-end-device-3g.md` | ✅ |
| 6 | Redundant Network Requests on a Page | `phase-05-performance-debugging/06-redundant-network-requests-on-a-page.md` | ✅ |
| 7 | Long Tasks Blocking the Main Thread | `phase-05-performance-debugging/07-long-tasks-blocking-main-thread.md` | ✅ |
| 8 | High INP / Unresponsive Interactions | `phase-05-performance-debugging/08-high-inp-unresponsive-interactions.md` | ✅ |

### Phase 6 — State Management & Architecture Trade-offs (8 scenarios)

| # | Scenario | File | Status |
|---|----------|------|--------|
| 1 | Where Does This State Live? (Server/Client/URL/Form Sort) | `phase-06-state-architecture-tradeoffs/01-where-does-this-state-live.md` | ✅ |
| 2 | State Design: Filters + Saved Views + Real-time Counters | `phase-06-state-architecture-tradeoffs/02-state-design-filters-saved-views-realtime-counters.md` | ✅ |
| 3 | Optimistic Update With Rollback | `phase-06-state-architecture-tradeoffs/03-optimistic-update-with-rollback.md` | ✅ |
| 4 | Undo/Redo — Architecture Decision | `phase-06-state-architecture-tradeoffs/04-undo-redo-architecture-decision.md` | ✅ |
| 5 | Cross-tab State Sync | `phase-06-state-architecture-tradeoffs/05-cross-tab-state-sync.md` | ✅ |
| 6 | Conflict Resolution for Concurrent Edits | `phase-06-state-architecture-tradeoffs/06-conflict-resolution-concurrent-edits.md` | ✅ |
| 7 | Redux → Lightweight State — Migration Trade-offs | `phase-06-state-architecture-tradeoffs/07-redux-to-lightweight-state-migration-tradeoffs.md` | ✅ |
| 8 | Feature-flag-driven Rollout Design | `phase-06-state-architecture-tradeoffs/08-feature-flag-driven-rollout-design.md` | ✅ |

### Phase 7 — Networking & Data Layer Scenarios (8 scenarios)

| # | Scenario | File | Status |
|---|----------|------|--------|
| 1 | Data-fetching Layer for N Related Resources (Waterfalls → Parallelization) | `phase-07-networking-data-layer/01-data-fetching-layer-for-related-resources.md` | ✅ |
| 2 | Cache Invalidation Strategy | `phase-07-networking-data-layer/02-cache-invalidation-strategy.md` | ✅ |
| 3 | Pagination Data Model: Cursor vs. Offset | `phase-07-networking-data-layer/03-pagination-data-model-cursor-vs-offset.md` | ✅ |
| 4 | WebSocket Reconnection & Backoff | `phase-07-networking-data-layer/04-websocket-reconnection-backoff.md` | ✅ |
| 5 | Offline-first Sync With Conflict Resolution | `phase-07-networking-data-layer/05-offline-first-sync-conflict-resolution.md` | ✅ |
| 6 | Streaming LLM Response UI: SSE vs. WebSocket | `phase-07-networking-data-layer/06-streaming-llm-response-ui-sse-vs-websocket.md` | ✅ |
| 7 | Graceful Degradation for a Flaky Third-party API | `phase-07-networking-data-layer/07-graceful-degradation-flaky-third-party-api.md` | ✅ |
| 8 | GraphQL Client-side N+1 / Over-fetching | `phase-07-networking-data-layer/08-graphql-client-n-plus-1-overfetching.md` | ✅ |

### Phase 8 — CSS & Layout Debugging Scenarios (8 scenarios)

| # | Scenario | File | Status |
|---|----------|------|--------|
| 1 | Holy Grail Layout | `phase-08-css-layout-debugging/01-holy-grail-layout.md` | ✅ |
| 2 | Sticky Header Broken on Mobile Safari | `phase-08-css-layout-debugging/02-sticky-header-broken-mobile-safari.md` | ✅ |
| 3 | CLS From Late-loading Images & Fonts | `phase-08-css-layout-debugging/03-cls-from-late-loading-images-fonts.md` | ✅ |
| 4 | Responsive Grid for an Unknown Item Count | `phase-08-css-layout-debugging/04-responsive-grid-unknown-item-count.md` | ✅ |
| 5 | Z-index / Stacking Context Bug | `phase-08-css-layout-debugging/05-z-index-stacking-context-bug.md` | ✅ |
| 6 | RTL Layout Breaking | `phase-08-css-layout-debugging/06-rtl-layout-breaking.md` | ✅ |
| 7 | Nested Scroll Container Overflow Trap | `phase-08-css-layout-debugging/07-nested-scroll-container-overflow-trap.md` | ✅ |
| 8 | Dark Mode & Print Stylesheet Edge Cases | `phase-08-css-layout-debugging/08-dark-mode-print-stylesheet-edge-cases.md` | ✅ |

### Phase 9 — Accessibility Scenarios (7 scenarios)

| # | Scenario | File | Status |
|---|----------|------|--------|
| 1 | Accessible Modal — Full Scenario Walkthrough | `phase-09-accessibility-scenarios/01-accessible-modal-scenario.md` | ✅ |
| 2 | Accessible Combobox — Full Scenario Walkthrough | `phase-09-accessibility-scenarios/02-accessible-combobox-scenario.md` | ✅ |
| 3 | Live Region Announcements for Async Updates | `phase-09-accessibility-scenarios/03-live-region-async-announcements.md` | ✅ |
| 4 | Keyboard Trap Bug — Find and Fix | `phase-09-accessibility-scenarios/04-keyboard-trap-bug-fix.md` | ✅ |
| 5 | Designer Pushback on Color Contrast — How You Handle It | `phase-09-accessibility-scenarios/05-designer-pushback-on-color-contrast.md` | ✅ |
| 6 | Auditing an Existing App for Accessibility | `phase-09-accessibility-scenarios/06-auditing-an-app-for-accessibility.md` | ✅ |
| 7 | Accessible Drag-and-Drop Alternative for Keyboard Users | `phase-09-accessibility-scenarios/07-accessible-drag-and-drop-alternative.md` | ✅ |

### Phase 10 — Security Scenarios (6 scenarios)

| # | Scenario | File | Status |
|---|----------|------|--------|
| 1 | Stored XSS Found in Production | `phase-10-security-scenarios/01-stored-xss-found-in-production.md` | 👉 **Next** |
| 2 | Auth Token Storage — a Breach Scenario | `phase-10-security-scenarios/02-auth-token-storage-breach-scenario.md` | ⬜ |
| 3 | Open Redirect Found in Code Review | `phase-10-security-scenarios/03-open-redirect-in-code-review.md` | ⬜ |
| 4 | Third-party Script Breaks Your CSP | `phase-10-security-scenarios/04-third-party-script-breaks-csp.md` | ⬜ |
| 5 | Clickjacking Risk on an Embeddable Widget | `phase-10-security-scenarios/05-clickjacking-embeddable-widget.md` | ⬜ |
| 6 | Dependency With a Known CVE in Production | `phase-10-security-scenarios/06-dependency-cve-in-production-response.md` | ⬜ |

### Phase 11 — Testing Strategy Scenarios (6 scenarios)

| # | Scenario | File | Status |
|---|----------|------|--------|
| 1 | Flaky E2E Test — Triage and Fix | `phase-11-testing-strategy-scenarios/01-flaky-e2e-test-triage.md` | ⬜ |
| 2 | "How Would You Test This Component?" Exercise | `phase-11-testing-strategy-scenarios/02-how-would-you-test-this-component.md` | ⬜ |
| 3 | Testing a Race-condition-prone Async Component | `phase-11-testing-strategy-scenarios/03-testing-a-race-condition-prone-component.md` | ⬜ |
| 4 | Mocking the Network Layer for Integration Tests (MSW) | `phase-11-testing-strategy-scenarios/04-mocking-network-layer-msw.md` | ⬜ |
| 5 | Visual Regression False Positives — Handling Them | `phase-11-testing-strategy-scenarios/05-visual-regression-false-positives.md` | ⬜ |
| 6 | Testing an Accessibility Requirement | `phase-11-testing-strategy-scenarios/06-testing-accessibility-requirements.md` | ⬜ |

### Phase 12 — Architecture, Migration & Engineering Judgment (8 scenarios)

| # | Scenario | File | Status |
|---|----------|------|--------|
| 1 | Migrating a Legacy jQuery/AngularJS App to React — Incrementally | `phase-12-architecture-migration-judgment/01-migrating-legacy-jquery-angularjs-to-react.md` | ⬜ |
| 2 | Migrating a CSR App to SSR for SEO | `phase-12-architecture-migration-judgment/02-migrating-csr-to-ssr-for-seo.md` | ⬜ |
| 3 | Introducing TypeScript to a Large Untyped JS Codebase | `phase-12-architecture-migration-judgment/03-introducing-typescript-to-large-js-codebase.md` | ⬜ |
| 4 | Monolith Frontend to Micro-frontends — When and How | `phase-12-architecture-migration-judgment/04-monolith-to-micro-frontends.md` | ⬜ |
| 5 | Choosing a Rendering Strategy for a New Product | `phase-12-architecture-migration-judgment/05-choosing-rendering-strategy-for-new-product.md` | ⬜ |
| 6 | Handling a Breaking API Change From Another Team | `phase-12-architecture-migration-judgment/06-handling-a-breaking-api-change-from-another-team.md` | ⬜ |
| 7 | Rolling Out a Risky Refactor Without Breaking Prod | `phase-12-architecture-migration-judgment/07-rolling-out-risky-refactor-without-breaking-prod.md` | ⬜ |
| 8 | Reviewing a PR With a Subtle Race Condition | `phase-12-architecture-migration-judgment/08-reviewing-a-pr-with-a-subtle-race-condition.md` | ⬜ |

### Phase 13 — AI-integrated Frontend Scenarios (6 scenarios)

| # | Scenario | File | Status |
|---|----------|------|--------|
| 1 | Streaming an LLM Chat Response Into the UI | `phase-13-ai-integrated-frontend/01-streaming-llm-chat-response-into-ui.md` | ⬜ |
| 2 | Choosing SSE vs. WebSocket for a Real-time Feature | `phase-13-ai-integrated-frontend/02-choosing-sse-vs-websocket-for-realtime-feature.md` | ⬜ |
| 3 | Rendering Streaming Markdown/Code Safely | `phase-13-ai-integrated-frontend/03-rendering-streaming-markdown-safely.md` | ⬜ |
| 4 | Cancelling a Long-running AI Request Cleanly | `phase-13-ai-integrated-frontend/04-cancelling-a-long-running-ai-request.md` | ⬜ |
| 5 | Designing UI for Agentic / Tool-use Flows | `phase-13-ai-integrated-frontend/05-designing-ui-for-agentic-tool-use-flows.md` | ⬜ |
| 6 | Handling AI Latency and Failure Gracefully in the UI | `phase-13-ai-integrated-frontend/06-handling-ai-latency-and-failure-gracefully.md` | ⬜ |

---

## How to Update This File

After each scenario is completed:
1. Change its status from ⬜ to ✅ in the progress tracker above
2. Move the 👉 **Next** marker to the following scenario
3. Commit: `git add CONTEXT.md && git commit -m "Update progress: [scenario name] done"`
