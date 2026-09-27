# Design a Configurable Dashboard With Widgets

## Quick Reference

| Decision Point | Choice | Why |
|---|---|---|
| Widget architecture | Registry of independently-implemented widget components, each self-contained (owns its own data fetching, loading/error state), composed by a generic layout shell | A monolithic dashboard component that knows about every widget's internals doesn't scale past a handful of widget types and makes adding a new widget a change to shared code rather than an isolated addition |
| Layout persistence | Serializable layout config (grid position/size per widget instance) persisted server-side per user, decoupled entirely from widget content/data | Layout (where things are) and data (what a widget shows) are orthogonal concerns — conflating them means a data refresh or a widget's internal state change has no business touching the persisted layout, and vice versa |
| Per-widget data fetching | Each widget instance fetches and caches its own data independently, keyed by its own config (not a single dashboard-wide fetch-everything-up-front waterfall) | Widgets are heterogeneous (different data sources, different refresh cadences, some user-added later) — a shared fetch-all-at-mount pattern doesn't scale to a dynamic, user-configurable set and forces irrelevant widgets to block on each other |
| Widget crashing | Per-widget error boundary, isolated so one widget's runtime error doesn't take down the rest of the dashboard | A single misbehaving or unexpectedly-erroring widget (bad data shape from a flaky API, a third-party embed) shouldn't render the entire dashboard blank — this is the same isolation principle as [Error Boundary Not Catching an Error](../phase-03-react-debugging-scenarios/10-error-boundary-not-catching-error.md), scoped per widget |
| Reflow/drag performance with many widgets | Virtualize off-screen widgets' rendering (mount placeholder, defer actual content mount until near-viewport) once widget count is large, and throttle layout recalculation during active drag | Dozens of simultaneously data-fetching, independently-rendering widgets is a real performance and network-request-volume concern, distinct from the drag-interaction performance concern of the grid itself |

## The Scenario

"Design a customizable dashboard — think something like a Grafana or Datadog dashboard, or a personalized analytics home page. Users can add, remove, resize, and rearrange widgets (charts, stat tiles, tables, whatever), and their layout should persist across sessions. New widget types get added to the product over time. Walk me through how you'd architect this, both the widget system itself and how layout and data fit together."

## Clarifying Questions

- **Is the set of widget types fixed and known at build time, or does the system need to support widgets being added/registered dynamically (e.g., by other teams, or even third-party plugin-style widgets)?** A fixed, build-time-known widget catalog can use a simple switch/lookup registry; a dynamically extensible one (new widget types shipped independently of the dashboard shell's own release cycle, or genuinely pluggable third-party widgets) pushes toward a more formal registration contract and possibly code-splitting per widget type, and raises the sandboxing/isolation question much more seriously if third parties are involved.
- **Does each widget fetch its own data independently, or is there a shared, dashboard-level data layer that widgets subscribe to (e.g., all widgets on this dashboard are viewing the same underlying dataset, just visualized differently)?** This changes the data-fetching architecture significantly — independent per-widget fetching is simpler and more decoupled but can produce redundant requests if multiple widgets need overlapping data; a shared data layer avoids duplication but couples widgets to a common data-fetching contract they must all agree to use.
- **Is layout a free-form drag-and-resize grid (arbitrary pixel/grid-cell positioning), or a simpler fixed set of layout slots/columns the user picks from?** Free-form grid layout (react-grid-layout-style) is a materially more complex UI/interaction problem (collision detection, resize handles, responsive breakpoint behavior) than choosing from a small set of predefined slot arrangements — worth confirming which is actually in scope before designing either.
- **Can a user add the same widget type more than once (e.g., two separate "line chart" widgets, each configured differently), or is each widget type a singleton per dashboard?** Determines whether widget identity in the persisted layout needs to be a widget-instance ID distinct from its type (supporting multiple instances of one type, each with independent config) or can key directly off type.
- **What happens when a widget type is deprecated/removed from the product, but a user's persisted layout still references it?** A real, easily-missed edge case — the layout-rendering logic needs an explicit fallback (e.g., an "this widget is no longer available" placeholder) rather than crashing or silently dropping the slot when it encounters a widget type it no longer recognizes.
- **Is there a size/performance ceiling on how many widgets a single dashboard can hold, or should the design assume dashboards can grow arbitrarily large?** Shapes whether virtualization of off-screen widgets and per-widget fetch throttling are necessary from day one or a later scaling concern.

## Approach & Trade-offs

**The widget system needs a clean registry-based architecture, where the dashboard shell knows nothing about any specific widget's internals — only a common contract every widget implements.** Each widget type registers itself (component to render, a config schema for its settings, a default size) against a type key; the dashboard shell's job is purely to read the user's persisted layout (a list of widget instances, each with a type, a config, and a grid position/size) and, for each entry, look up and render the corresponding registered component, passing it its own config. Adding a new widget type becomes an isolated addition (implement the widget, register it) rather than a change to shared dashboard-shell code — this is the same registry pattern that underlies extensible plugin systems generally, and is the architectural property that lets "new widget types get added over time" (as stated in the scenario) not require touching the shell itself.

**Layout and data are orthogonal concerns and must be decoupled in both the data model and the persistence strategy.** The persisted layout is purely structural — which widget instances exist, their type, their grid position/size, and their widget-specific configuration (e.g., "this chart widget is configured to show the 'signups' metric over the last 30 days") — and contains no actual fetched data. Each widget instance, once rendered by the shell with its config, is independently responsible for fetching whatever data its config specifies and rendering it. This separation matters because layout changes (dragging a widget to a new position) and data changes (a widget's underlying metric refreshing) happen on entirely different cadences and through entirely different code paths — conflating them (e.g., persisting a widget's last-fetched data as part of the "layout" object) creates a stale-data bug the moment the underlying data changes without a corresponding layout save, and needlessly couples an unrelated layout-persistence API to the shape of every widget type's data.

**Per-widget independent data fetching is the right default, accepting some potential request duplication in exchange for decoupling — with a shared data layer only introduced if duplication becomes a measured problem.** Because widgets are heterogeneous (different data sources, different refresh intervals, dynamically added/removed by the user), having each widget own its own fetch (using whatever this repo's other scenarios establish as the standard client-side data-fetching pattern — a cache-aware fetch hook, keyed by the widget's own config) keeps every widget type fully self-contained and addable/removable without any shared-fetch-orchestration code needing to know about it. The trade-off is that if two widgets on the same dashboard happen to need overlapping data (e.g., two different chart widgets both pulling from the same underlying metrics endpoint with different visualizations), each fetches independently rather than sharing one request — an acceptable inefficiency in most cases, and one a shared cache layer (e.g., a request-deduplicating fetch library keyed by URL/params, as covered in [Redundant Network Requests on a Page](../phase-05-performance-debugging/06-redundant-network-requests-on-a-page.md)) can transparently eliminate later without requiring the widget architecture itself to change — deduplication becomes an implementation detail of the shared fetch layer, not a structural coupling between widgets.

**Each widget must be wrapped in its own error boundary, isolating a runtime failure to that one widget's slot rather than the whole dashboard.** A dashboard is, almost by definition, aggregating heterogeneous, often third-party-adjacent or independently-maintained pieces of UI — some widget encountering an unexpected data shape, a rendering bug, or an uncaught exception is a realistic, expected occurrence at scale, and it should degrade to "this one widget shows an error state" rather than crashing the entire page (the same principle as [Error Boundary Not Catching an Error — Why](../phase-03-react-debugging-scenarios/10-error-boundary-not-catching-error.md), applied per-widget rather than at the page root, since a page-root-only boundary would still take down every other, perfectly healthy widget alongside the one that actually failed).

**Free-form grid layout (drag/resize) needs its own dedicated interaction handling, typically via a purpose-built grid library rather than hand-rolled from scratch, but the underlying persisted data model is simple regardless of which library implements the interaction.** The layout data itself reduces to an array of `{ widgetId, x, y, w, h }` grid-cell coordinates per widget instance — how the user interactively produces those coordinates (drag handles, resize corners, collision/reflow logic preventing overlaps) is a substantial, largely solved interaction problem that's rarely worth re-implementing from scratch in an interview or in practice; what is worth designing explicitly is the boundary between that interaction layer and the rest of the system — the grid library emits layout-change events, which the dashboard shell debounces and persists, entirely decoupled from anything about what the widgets being rearranged actually contain.

## Solution

**Widget registry — the core extensibility mechanism:**

```tsx
interface WidgetDefinition<TConfig = unknown> {
  type: string;
  displayName: string;
  defaultSize: { w: number; h: number };
  Component: React.ComponentType<{ config: TConfig; instanceId: string }>;
  ConfigEditor?: React.ComponentType<{ config: TConfig; onChange: (next: TConfig) => void }>;
}

const widgetRegistry = new Map<string, WidgetDefinition<any>>();

function registerWidget(def: WidgetDefinition<any>) {
  widgetRegistry.set(def.type, def);
}

registerWidget({
  type: 'line-chart',
  displayName: 'Line Chart',
  defaultSize: { w: 4, h: 3 },
  Component: LineChartWidget,
  ConfigEditor: LineChartConfigEditor,
});
```

**Layout data model — purely structural, no fetched data embedded:**

```ts
interface WidgetInstance {
  instanceId: string;   // unique per placed widget — NOT the same as widget type, supports multiple instances of one type
  type: string;         // key into widgetRegistry
  config: unknown;      // widget-type-specific — opaque to the shell
  layout: { x: number; y: number; w: number; h: number };
}

interface DashboardLayout {
  dashboardId: string;
  widgets: WidgetInstance[];
  updatedAt: string;
}
```

**The shell — knows nothing about any specific widget, only the registry contract:**

```tsx
function DashboardShell({ layout }: { layout: DashboardLayout }) {
  return (
    <GridLayout onLayoutChange={(next) => persistLayoutChange(layout.dashboardId, next)}>
      {layout.widgets.map((instance) => {
        const def = widgetRegistry.get(instance.type);
        if (!def) {
          return <UnknownWidgetPlaceholder key={instance.instanceId} type={instance.type} />; // deprecated/removed widget type
        }
        return (
          <WidgetErrorBoundary key={instance.instanceId} widgetType={instance.type}>
            <def.Component config={instance.config} instanceId={instance.instanceId} />
          </WidgetErrorBoundary>
        );
      })}
    </GridLayout>
  );
}
```

**A widget, fully self-contained — owns its own data fetching and loading/error state:**

```tsx
function LineChartWidget({ config, instanceId }: { config: LineChartConfig; instanceId: string }) {
  const { data, isLoading, error } = useMetricData(config.metricKey, config.rangeDays); // this widget's own fetch, own cache key

  if (isLoading) return <WidgetSkeleton />;
  if (error) return <WidgetErrorState message="Couldn't load this chart's data." />;
  return <LineChart data={data} />;
}
```

**Per-widget error isolation:**

```tsx
class WidgetErrorBoundary extends React.Component<{ widgetType: string; children: React.ReactNode }, { hasError: boolean }> {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  componentDidCatch(error: Error) { logWidgetError(this.props.widgetType, error); }
  render() {
    if (this.state.hasError) return <WidgetErrorState message="This widget encountered an error." />;
    return this.props.children;
  }
}
```

> **Check yourself:** Without looking above, explain why `instanceId` must be distinct from `type` in the layout data model, and describe a concrete scenario where conflating the two breaks.

## Layout Persistence & Debounced Save

```tsx
const persistLayoutChange = useDebouncedCallback((dashboardId: string, newLayout: WidgetInstance[]) => {
  saveLayoutToServer(dashboardId, newLayout); // fires once dragging/resizing has settled, not on every intermediate pixel of movement
}, 800);
```

Debouncing the persisted-layout save (rather than saving on every intermediate drag-frame update) is the same principle as debouncing any high-frequency UI event before it reaches the network — the grid library's `onLayoutChange` fires continuously during an active drag, and persisting every intermediate position would be both wasteful and unnecessary, since only the final settled position matters for what gets saved.

## Scaling Considerations

**Virtualize/defer rendering of off-screen widgets once a dashboard can hold many.** A dashboard with 30+ widgets, each independently fetching and rendering its own content, means 30+ simultaneous network requests and mounted chart/table components on initial load if nothing is deferred — the same windowing principle as [Virtualized List (Windowing) From Scratch](../phase-02-component-machine-coding/03-virtualized-list-windowing-from-scratch.md) applies here at the widget-grid level: widgets sufficiently far below the fold can render a lightweight placeholder and defer their actual data-fetch-and-mount until they scroll near-viewport (via `IntersectionObserver`), meaningfully reducing initial load's request volume and render cost without changing anything about how any individual widget is implemented.

**Throttle layout recalculation during active drag/resize, independent of the debounced persistence save.** The two are separate performance concerns — persistence debouncing controls network write volume; a separate throttle on how often the grid's collision-detection/reflow logic actually recomputes during a drag controls interaction smoothness (60fps dragging needs the reflow calculation itself to be cheap and not run more often than each animation frame warrants), and most mature grid libraries handle this internally, but it's worth naming as a distinct concern from persistence.

**Consider a request-deduplicating shared cache layer if per-widget independent fetching produces measurable redundant-request volume in practice**, rather than defaulting to a shared data layer up front — this is explicitly a "measure, then optimize" decision, not a default architectural requirement, per the Approach section above.

## Gotchas

**Embedding a widget's fetched data inside the persisted layout object.** Produces stale data the moment the underlying data changes without a corresponding layout save, and needlessly couples the layout-persistence API's shape to every widget type's data shape — layout must persist only structural placement/config, never fetched content.

**No error boundary per widget, only a single page-root boundary (or none at all).** One widget's runtime error takes down the entire dashboard rather than degrading just that widget's slot — a real, likely occurrence at scale given a dashboard's heterogeneous widget set.

**No fallback for an unrecognized/deprecated widget type still referenced in a persisted layout.** Either crashes the shell (attempting to render `undefined` as a component) or silently drops that slot with no indication to the user why a widget they expect to see is missing — an explicit "this widget is no longer available" placeholder is the honest default.

**Persisting layout changes on every intermediate drag-frame event rather than debouncing to the settled final position.** Produces excessive network write volume for no benefit, since only the final position is ever actually meaningful to persist.

**Using widget `type` as the layout's unique key instead of a distinct `instanceId`.** Breaks the moment a user wants two instances of the same widget type configured differently — a design that doesn't anticipate this from the start often requires a breaking data-model migration to retrofit multiple-instances-per-type support later.

**A shared dashboard-wide fetch-everything-at-mount waterfall, rather than each widget owning its own fetch.** Doesn't scale to a dynamic, user-configurable widget set — adding a new widget type would require also updating shared fetch-orchestration code to know about its data needs, defeating the registry pattern's whole purpose of isolated, independent widget additions.

## Follow-up Questions

**Q (High): Why should each widget fetch its own data independently rather than the dashboard shell orchestrating one shared fetch for all widgets' data up front?**

Answer: A shared, dashboard-level fetch-everything-up-front approach requires the orchestrating code to know, for every widget on the dashboard, what data it needs and how to fetch it — which directly breaks the registry pattern's core promise that adding a new widget type is an isolated addition, not a change to shared code. It also creates an artificial coupling where an unrelated widget's slow or failing data fetch can block or complicate the loading state of the entire dashboard, rather than each widget independently showing its own loading/error/success state on its own timeline. Independent per-widget fetching accepts a real but usually minor cost (potential redundant requests if multiple widgets need overlapping data) in exchange for keeping every widget type fully self-contained and pluggable — and that redundancy, if it becomes a measured problem, is solvable transparently at the shared-fetch-library layer (request deduplication/caching keyed by URL) without requiring the widget architecture itself to change.

The trap: proposing a shared fetch-all-at-mount layer as the default, more "efficient" design without acknowledging what it costs architecturally — the efficiency gain (avoiding some redundant requests) is real but secondary to the coupling cost it introduces to the extensibility the scenario explicitly asks for ("new widget types get added over time").

---

**Q (High): A user's dashboard references a widget type that was since removed from the product (e.g., it was deprecated last quarter). What should happen when their layout renders, and how do you prevent this from crashing the page?**

Answer: The shell's render loop, when looking up a widget instance's `type` against the registry, must explicitly handle a lookup miss (`widgetRegistry.get(instance.type)` returning `undefined`) as a distinct, expected case — rendering a dedicated placeholder component (something like "this widget is no longer available — remove it?") in that grid slot rather than either crashing (attempting to render an undefined component) or silently omitting the slot with no explanation. This is a straightforward defensive check in the shell's rendering logic, but it's the kind of edge case that's easy to overlook if the registry lookup is written assuming every persisted `type` will always resolve — a design that ships new widget types over time (as the scenario states) will, with near certainty, eventually also deprecate/remove some, and stale layouts referencing them are a predictable, not hypothetical, occurrence.

The trap: treating this as a rare edge case not worth designing for explicitly, or handling it only via a crash boundary (technically prevents a full page crash via the per-widget error boundary, but produces a generic "this widget encountered an error" message rather than the more honest and actionable "this widget type no longer exists" — recognizing the distinction between a runtime error and a genuinely-missing registry entry is the stronger answer).

---

**Q (High): How would you handle a widget that needs to communicate with or affect another widget on the same dashboard — e.g., clicking a data point in one chart widget should filter what a table widget below it displays?**

Answer: This introduces genuine cross-widget coupling that the fully-decoupled, independently-fetching widget model doesn't handle for free, and needs an explicit, deliberately narrow mechanism rather than ad hoc direct communication between widget components (which would break the registry's isolation guarantees). The cleanest approach introduces a small, dashboard-level shared "interaction state" (e.g., a `selectedFilter` value, or a `crossFilter` object) that any widget can optionally read from and/or write to via a well-defined hook (e.g., `useDashboardFilter()`), decoupled from any specific pair of widget types knowing about each other directly — a chart widget's click handler writes to this shared state, and any other widget (including one added later, of a type that didn't exist when the chart widget was built) can independently choose to read and react to it. This keeps the registry's core isolation property intact (no widget needs to know about any specific other widget by type or identity) while still enabling optional, opt-in cross-widget interaction through a shared, generic channel.

The trap: implementing direct widget-to-widget references or callbacks (widget A's config holds a reference to widget B's instance ID and calls into it directly) — this works for the specific pair being built but doesn't generalize, breaks if either widget is removed or reconfigured, and reintroduces exactly the tight coupling the registry pattern was designed to avoid.

---

**Q (Medium): How would you support third-party or externally-developed widgets, versus widgets built entirely in-house?**

Answer: True third-party widgets (code not trusted or maintained by the same team as the dashboard shell) raise a sandboxing/isolation concern well beyond what an in-house widget registry needs — an iframe-based sandbox (each third-party widget rendered inside its own iframe, communicating with the shell only via `postMessage`) is the standard approach, since it provides genuine script/style/DOM isolation that a same-document React component registration does not; a misbehaving or malicious third-party widget can't reach into the rest of the page's DOM or global state from inside its own iframe. This is a meaningfully larger scope than the in-house registry pattern described above (which assumes all widget code is trusted, same-origin, and reviewed through the same process as the rest of the app) — worth explicitly flagging as a different problem with a different solution shape, not an incremental extension of the same registry.

The trap: assuming the same in-process component-registry pattern extends naturally to genuinely untrusted third-party code — it doesn't; untrusted code sharing the same JS realm and DOM as the rest of the dashboard is a real security and stability risk (an errant third-party widget could, for instance, still access `document` or global variables even inside a React error boundary, since error boundaries only catch render-time exceptions, not all forms of misbehavior), which is exactly the class of problem iframe sandboxing (or similar full isolation) is for.

---

**Q (Medium): Should widget configuration (e.g., a chart widget's selected metric and date range) live in the same persisted layout object as position/size, or somewhere separate?**

Answer: Reasonable to keep it in the same `WidgetInstance` record (as modeled above) since both are structural/configuration data about a specific widget instance, updated relatively infrequently by explicit user action (as opposed to fetched data, which is a fundamentally different, much-higher-frequency-changing category) — the key distinguishing property for the layout/data split discussed throughout this scenario isn't "position vs. everything else," it's "user-authored configuration/structure vs. server-fetched content." Config and position are both the former; only the widget's actual rendered data is the latter, and that's the piece that must never be part of the persisted layout object.

The trap: over-splitting config and position into separate persisted objects/APIs on the theory that "layout" should mean only geometric position — this adds unnecessary complexity (two round trips or two data structures to keep in sync for what's conceptually one "how is this widget instance set up" record) without addressing the actual problem the layout/data separation exists to solve, which is keeping ephemeral fetched content out of persisted structural state.

---

**Q (Low): How would you let a user preview what a new widget will look like before adding it to their dashboard?**

Answer: The registry's per-widget-type metadata (already holding `displayName`, `defaultSize`, and the component itself) can be extended with an optional preview affordance — rendering the actual widget `Component` with representative/sample config in a picker UI (a "widget gallery" modal), rather than a hand-maintained static screenshot or description per widget type, keeps the preview automatically accurate as the widget's real implementation evolves, since it's literally the same component being rendered, just with placeholder config instead of a real, saved instance's config.

The trap: maintaining separate static preview images or descriptions per widget type in the picker UI — this drifts out of sync with the widget's actual current appearance the moment the widget's implementation changes, and is genuinely unnecessary extra maintenance given the registry already has everything needed to render a live, accurate preview directly.

---

## Self-Assessment

- [ ] Can design a widget registry and explain why the dashboard shell should have zero widget-type-specific knowledge
- [ ] Can explain why layout and data are orthogonal and must be persisted/updated through entirely separate paths
- [ ] Can justify per-widget independent data fetching over a shared dashboard-wide fetch, and name when a shared cache layer becomes worth introducing
- [ ] Can design per-widget error boundary isolation and explain why a single page-root boundary is insufficient
- [ ] Can handle an unrecognized/deprecated widget type in a persisted layout without crashing or silently dropping it
- [ ] Can explain the sandboxing gap between in-house and genuinely third-party widgets, and why an iframe boundary is needed for the latter

---
*Next: Design a Ticket/Seat Booking UI — shifts from a composition/extensibility problem to a correctness-under-contention problem: multiple users racing to reserve the same finite, uniquely-identified resource (a seat), where the central new concerns become optimistic-locking UX, hold/expiry timers, and real-time availability sync across concurrent viewers.*
