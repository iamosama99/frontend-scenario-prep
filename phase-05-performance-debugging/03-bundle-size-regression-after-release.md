# Bundle Size Regression After a Release

## Quick Reference

| Regression Source | How You Spot It | Fix |
|---|---|---|
| A new dependency pulled in whole | Bundle analyzer shows a large new module graph rooted at one import | Check for a lighter alternative, or import only the specific function needed |
| Barrel-file import defeating tree-shaking | Importing `{ Button } from 'ui-lib'` pulls in the entire library graph | Import from the specific submodule path, or fix the library's `sideEffects`/ESM config |
| Duplicate library versions bundled | Analyzer shows the same package name twice at different versions | Dedupe via lockfile/`resolutions`, align versions across the dependency tree |
| Code that should be route-split ended up in the main chunk | A feature only used on one route inflates the shared/main bundle | Dynamic `import()` at the route or feature boundary, verify it's a separate chunk |
| Polyfills/transpile target regressed (broader browser support added) | Main bundle grew uniformly, matching a `browserslist`/babel-config change | Narrow the target, or serve differential bundles (modern vs. legacy) |

## The Scenario

"Someone on the team flagged that our main JS bundle jumped from 340KB to 510KB gzipped after last week's release. Nobody remembers 'adding 170KB of stuff' on purpose. Find out what actually caused it and how you'd fix it — and tell me how you'd prevent this from silently happening again."

## Clarifying Questions

- **Do we have a bundle size measurement from before and after the release specifically — CI build artifacts, a bundle-analyzer report, or a bundle-size-tracking tool (bundlesize, size-limit, a Lighthouse CI budget)?** Without a stored before/after artifact I'd have to reconstruct it (check out the pre-release commit, build, compare), which works but is slower and less precise than diffing two already-generated bundle analyzer reports.
- **Was this release a single deploy or does it bundle several merged PRs — and do we have the list of what merged in that window?** A single-PR release narrows the search immediately (diff that PR's dependency changes and new imports). A larger release requires binary-searching across the merged commits (bisecting by rebuilding at intermediate commits) or, more efficiently, diffing the lockfile and import graph directly rather than reading through PR diffs individually.
- **Is the 170KB increase concentrated in one chunk (e.g., the main/vendor bundle) or spread across many route-based chunks?** Concentrated in one shared chunk points at something imported broadly (a new top-level dependency, a change to a shared utility/config) — spread evenly across many chunks points more at something systemic, like a build tool config change (a broadened Babel/polyfill target, a changed tree-shaking setting) affecting every chunk uniformly.
- **Did the lockfile change in ways beyond what the feature PR would obviously need** — a new top-level dependency, or existing dependencies bumping major versions that pulled in new transitive dependencies? I'd want to diff `package-lock.json`/`yarn.lock`/`pnpm-lock.yaml` between the two releases directly, since a transitive bump (a dependency's dependency updating to a version with new sub-dependencies) is a very common, easy-to-miss cause that doesn't show up in the application code diff at all.
- **Is "510KB gzipped" measured on the actual production build output, or is this a dev-build/uncompressed number being compared against a gzipped one?** Comparing incompatible measurements (dev vs. prod build, or gzipped vs. raw) can manufacture an apparent regression that isn't real, or mask a real one — I'd confirm both numbers came from the same build mode and compression settings before treating the delta as meaningful.

## Approach & Trade-offs

**Use a bundle analyzer to see the actual composition, rather than guessing from the PR list.** Tools like `webpack-bundle-analyzer`, `vite-bundle-visualizer`, or `source-map-explorer` render the bundle as a proportionally-sized treemap based on the actual compiled/minified output and its source maps — this shows exactly which modules contribute how many bytes, which is dramatically more reliable than reading a diff of application code and guessing what got heavier, since the actual bloat is very often in a transitive dependency nobody directly touched.

**Diff two analyzer reports (before/after) rather than reading one in isolation.** A single treemap shows *what's in* the bundle, but not *what changed* — generating a report at the pre-release commit and one at the current commit, then comparing them (several tools support this directly; otherwise, diffing the two module lists and their sizes manually) directly answers "what's new or grew," which is the actual question, instead of re-deriving it from a static snapshot.

**Distinguish four categorically different causes early, since each has a different fix and a different owner.** (1) A genuinely new, necessary dependency that's just heavy — the fix is evaluating lighter alternatives or accepting the cost if justified. (2) An existing import that used to tree-shake cleanly and now doesn't (a barrel-file import pulling in a whole library, a package losing its `sideEffects: false` marking in an update) — the fix is import-path hygiene, not removing functionality. (3) Duplicate versions of the same library bundled side by side because of a version mismatch somewhere in the dependency tree — the fix is deduping via the lockfile. (4) A build-tool/config regression (broadened browser targets pulling in more polyfills, a tree-shaking setting accidentally disabled) affecting the whole bundle uniformly — the fix is in build config, not application code at all. Treating all four as "reduce bundle size, generically" leads to solving the wrong one.

**Prevention is a CI gate, not a retrospective habit.** Catching this after a user-visible release (or after someone happens to notice) means the regression already shipped; the actual fix for *recurrence* is a bundle-size budget enforced in CI (`size-limit`, `bundlesize`, or a custom check comparing the PR's build output against `main`) that fails the build (or at least flags loudly) when a chunk crosses a threshold or grows by more than some percentage — turning "someone eventually notices" into "the PR that caused it gets flagged before merge, with the specific diff attached."

## Solution — the diagnostic + fix walkthrough

**Step 1 — generate before/after bundle analyzer reports.**

```bash
git checkout <pre-release-commit>
npm run build -- --analyze   # or: npx source-map-explorer dist/main.*.js --html before.html
git checkout <post-release-commit>
npm run build -- --analyze   # → after.html
```

**Step 2 — diff the two treemaps.** Say the diff shows a single new top-level entry:

```
+ moment-timezone (168 KB gzipped) — new, under node_modules/moment-timezone
+ moment (72 KB gzipped) — new, moment-timezone's peer dependency
```

**Step 3 — trace it to the actual code change.** `grep`-ing the release's diff for the import:

```ts
// New in this release, added for a single "convert to user's timezone" feature
import moment from 'moment';
import 'moment-timezone';

function formatInUserTz(date: Date, tz: string) {
  return moment(date).tz(tz).format('MMM D, h:mm A');
}
```

`moment` plus `moment-timezone` (which bundles IANA timezone data) is a well-known heavy combination — added for what's functionally a single date-formatting utility used in one place.

**Step 4 — replace with a lighter, purpose-built alternative** (native `Intl.DateTimeFormat` covers this exact use case with zero bundle cost, since it's a browser built-in):

```ts
function formatInUserTz(date: Date, tz: string) {
  return new Intl.DateTimeFormat('en-US', {
    timeZone: tz,
    month: 'short', day: 'numeric', hour: 'numeric', minute: '2-digit',
  }).format(date);
}
```

This removes both `moment` and `moment-timezone` from the graph entirely — no polyfill needed for the timezone-database aspect, since `Intl.DateTimeFormat` with an IANA `timeZone` string is natively supported in all evergreen browsers and Node.

**Step 5 — re-measure and confirm the delta is accounted for**, then check whether anything *else* in the diff (beyond this one obvious addition) also contributed — a single treemap-visible spike doesn't guarantee it's the *only* regression; I'd sum the diff's contributions against the full observed 170KB delta and keep looking if there's an unexplained remainder (e.g., a second, smaller transitive bump elsewhere).

**Step 6 — add the CI guardrail** so this class of regression is caught automatically going forward:

```json
// size-limit config (package.json)
"size-limit": [
  {
    "path": "dist/main.*.js",
    "limit": "350 KB"
  }
]
```

```yaml
# CI step
- run: npx size-limit
```

This fails the PR's CI run (rather than a later release) the moment a change pushes the main bundle over budget, with `size-limit`'s own diff output naming the specific delta right in the PR, before merge.

> **Check yourself:** If the analyzer diff had instead shown the *same* library appearing twice at two different versions (rather than one clean new addition), what would that tell you about the cause, and what would you check in the lockfile to confirm it?

## Gotchas

**Fixing the one obviously-heavy import the analyzer highlights, without checking whether the full delta is accounted for.** A 170KB regression might be 150KB from one obvious new dependency and 20KB from a second, smaller, easy-to-miss change (a transitive bump, a barrel-import regression) elsewhere in the same release — declaring victory after removing the big one without reconciling the total against the original measurement can leave a real, smaller regression shipped.

**Comparing bundle sizes measured under different build configurations** (dev vs. production mode, different minifier settings, gzip vs. brotli vs. raw) and treating the delta as meaningful — this can manufacture a phantom regression or mask a real one; the two measurements need to come from identical build/compression settings to be comparable.

**Removing a heavy dependency's import in one file while it's still imported elsewhere in the codebase.** Bundlers don't remove a package from the graph just because *one* call site stopped using it — if `moment` is imported in three other files, the fix in Step 4 above only removes it once all call sites are migrated; verifying the package is actually gone from the *post-fix* analyzer output (not just assuming it based on the one file changed) confirms the fix is complete.

**Adding a bundle-size CI budget that's either too loose to catch real regressions or so strict it blocks legitimate growth.** A budget set once and never revisited either drifts irrelevant (too loose, if the app has genuinely grown since) or becomes a constant false-positive nuisance developers learn to ignore/override (too strict) — budgets need periodic review as the app's baseline legitimately changes, and should distinguish "grew because of an intentional, reviewed feature" from "grew because of an unnoticed dependency bloat."

**Treating tree-shaking as an all-or-nothing property of "using ES modules," rather than something that can silently break per-import.** A library can be fully tree-shakeable in general, but a specific import pattern (`import { Button } from 'ui-lib'` where `ui-lib`'s root index re-exports everything from a single barrel file with side effects, or without a `sideEffects: false` marking in its `package.json`) can defeat tree-shaking for that one import while the rest of the app's imports shake fine — the analyzer output (seeing the *entire* library graph pulled in for a single-component import) is what actually reveals this, not an assumption based on the library's general reputation.

## Follow-up Questions

**Q (High): Walk through exactly how you'd use a bundle analyzer to find the cause of an unexplained size regression, from a cold start with no prior data.**

Answer: First, reproduce a comparable "before" build — check out the commit/tag from before the regression was introduced (from release history or CI artifacts if available, otherwise the last commit before the suspected release window) and build it with the analyzer enabled (`webpack-bundle-analyzer`, `source-map-explorer`, or the bundler's built-in equivalent), saving that treemap. Then build the current (regressed) commit the same way. With both treemaps in hand, diff them — either using a tool that supports direct before/after diffing, or manually comparing the module lists and sizes (most analyzers can export a JSON stats file, which is scriptable: sort both by size, diff the sets of module names, sum the size deltas for anything present in both but changed). The diff directly names new/grown modules by their `node_modules` path, which traces unambiguously back to a specific dependency (and, from there, `git log`/`git blame` on the lockfile or the import site identifies which PR/commit introduced it) — this is a much more direct path than reading through the release's application-code diff hoping to spot the cause by inspection.

The trap: describing this as "I'd look at the bundle analyzer" without the actual before/after diffing step — a single analyzer snapshot shows composition, not *change*, and the regression's cause is defined by what changed, not by what's merely present.

---

**Q (High): A barrel-file import (`import { Button } from 'design-system'`) is pulling in the entire design system library instead of just the `Button` component, even though the library is written with ES modules. Explain the mechanism, and how you'd fix it from the consuming code's side versus the library's side.**

Answer: Tree-shaking relies on the bundler being able to statically determine that unused exports have no side effects and can be safely dropped — a barrel file (`index.js` that does `export * from './Button'; export * from './Modal'; export * from './Table'; ...`) means importing anything from the barrel technically imports the whole barrel module first, and if the library's `package.json` doesn't declare `"sideEffects": false` (or lists the barrel file as having side effects, or the bundler can't prove the re-exported modules are side-effect-free), the bundler conservatively keeps the entire graph reachable from that barrel file rather than risk dropping something with an observable side effect (e.g., a module that registers a global on import). From the consuming code's side, the immediate fix is importing directly from the specific submodule path instead of the barrel — `import Button from 'design-system/Button'` — which bypasses the barrel file's module graph entirely and only pulls in what `Button`'s own module actually imports. From the library's side, the durable fix is ensuring `package.json` correctly declares `"sideEffects": false` (or an accurate array of the specific files that do have side effects) so bundlers can safely tree-shake through the barrel for *all* consumers, without every consumer needing to know to avoid the barrel.

The trap: treating this purely as "tree-shaking is broken, nothing to do about it" — the deep-import workaround is available immediately on the consuming side, and the sideEffects fix is a concrete, well-defined change on the library side, not a vague "tree-shaking is unreliable" shrug.

---

**Q (High): The analyzer shows `lodash` appearing twice in the bundle at two different versions (4.17.15 and 4.17.21). What does this indicate, and how do you actually fix it (not just "update the version")?**

Answer: This indicates two different parts of the dependency tree depend on incompatible semver ranges of `lodash` that the package manager's deduplication couldn't collapse into a single shared version — commonly, the application's own `package.json` pins one range while some third-party dependency's own `package.json` pins a different, non-overlapping range, so the lockfile ends up installing (and the bundler ends up bundling) both copies side by side, each fully included since they're technically different modules from the bundler's perspective. Fixing it means finding *why* the second version exists — `npm ls lodash` (or `yarn why lodash` / `pnpm why lodash`) shows the full dependency chain responsible for each installed version — and then either updating the offending dependency to a version whose own `lodash` range overlaps with the app's, or, if that's not available, forcing a single resolved version across the tree via the package manager's override mechanism (`resolutions` in Yarn, `overrides` in npm, `pnpm.overrides` in pnpm), verifying afterward that the build actually collapses to one `lodash` entry in the analyzer output and that nothing breaks due to the forced version being outside what some dependency originally expected.

The trap: saying "just run `npm update`" without identifying *which* dependency's nested requirement is pinning the older version — a blind update often does nothing here, since the older pinned version is usually coming from a third-party package's own lockfile-level requirement, not the application's direct `package.json`, and needs the override mechanism or an upstream fix, not just bumping the top-level version.

---

**Q (Medium): How would you determine whether a bundle size increase came from application code growth versus a broadened Babel/browserslist transpilation target?**

Answer: A broadened target (e.g., `browserslist` config changing to support older browsers, or a Babel preset-env target regression) tends to show up as many small, uniformly-distributed size increases across most/all chunks — additional polyfills (`core-js` entries for features the new target doesn't natively support) and less-optimized, more-verbose transpiled output for modern syntax (arrow functions, classes, optional chaining getting downleveled) appear scattered throughout the bundle rather than as one large new module. I'd check this by diffing the `browserslist` config / Babel config between the before/after commits directly (a config diff, not a code diff) and separately confirming in the analyzer whether the size increase is concentrated in a few new/grown top-level modules (pointing at a dependency-level cause) versus spread thinly and consistently across many existing modules that didn't change in application logic (pointing at a transpilation-target cause) — a `core-js`/`regenerator-runtime` polyfill entry appearing or growing in the analyzer output specifically confirms the polyfill-target theory.

The trap: assuming any bundle growth must come from "someone added code" — a build-tooling/config regression affecting the whole compilation target is a distinct, easy-to-overlook category that requires diffing config files, not just application source.

---

**Q (Medium): Your bundle-size CI check (`size-limit`) is now in place. A legitimate new feature genuinely needs 40KB of new, necessary code and will fail the existing budget. How do you handle this without it becoming "the check everyone just overrides"?**

Answer: The budget itself should be treated as a deliberate, reviewed decision each time it needs to move, not an obstacle to route around silently — raising the limit should require the same PR (or a linked one) to state *why* the increase is justified (what feature, what was evaluated as an alternative, whether it's route-split so it doesn't hit all users) as part of the review, the same way a schema migration or an infra change gets called out explicitly rather than snuck through. Practically, I'd also push back on *where* the 40KB lands before accepting a blanket limit increase — if the feature is only used on one route/page, the actual fix is often route-splitting it into its own chunk rather than growing the shared main bundle's budget, which keeps the cost paid only by users of that feature rather than by everyone on every page load. A budget that's raised silently, without that conversation, degrades into exactly the "regression nobody remembers approving" scenario this whole investigation started from — the check is only useful if crossing it triggers a real decision, not an automatic bump.

The trap: treating "the check is blocking my legitimate PR" as proof the check is wrong and should just be raised — sometimes it correctly identifies that the new code belongs in a separate chunk rather than the shared bundle, which is a design decision, not a budget-tuning problem.

---

**Q (Low): Would switching from gzip to Brotli compression change how you'd interpret or set these bundle size budgets?**

Answer: Brotli generally compresses JS text noticeably better than gzip (often 15–25% smaller on typical JS payloads) at the same content, so a budget defined against gzipped size and a budget defined against Brotli-compressed size of the *identical* code are not directly comparable numbers — switching compression without adjusting the budget's baseline would make an unchanged bundle appear to have "improved," which could mask a real code-level regression happening at the same time as the compression migration. I'd re-baseline the budget immediately after switching compression algorithms (measure the current, already-accepted bundle under the new compression, and set the budget relative to that), and make sure whatever tool enforces the budget (`size-limit`, CI script) is actually measuring the same compression the production CDN/server serves — a budget checked against gzip in CI while production actually serves Brotli would be validating a number nobody's users experience.

The trap: assuming raw JS size (or one compression algorithm's output) is a stable, comparable unit forever — compression settings are part of the measurement, and changing them without re-baselining silently invalidates prior budget numbers.

---

## Self-Assessment

- [ ] Can describe the before/after bundle-analyzer-diff workflow for finding a size regression's cause, not just "open the analyzer"
- [ ] Can name and distinguish the four common regression categories (new heavy dependency, broken tree-shaking, duplicate versions, build-config/polyfill target change) and their distinct fixes
- [ ] Can explain the barrel-file/`sideEffects` mechanism that defeats tree-shaking and fix it from both the consumer and library side
- [ ] Can explain what duplicate library versions in a bundle indicate and how to actually resolve them (not just "update")
- [ ] Can design a CI bundle-size budget that catches regressions without becoming a rubber-stamped nuisance

---
*Next: Production Memory Leak Triage — a load-time/bundle-size problem gives way to a runtime, accumulating-over-a-session problem, using heap snapshots rather than a bundle analyzer as the diagnostic tool.*
