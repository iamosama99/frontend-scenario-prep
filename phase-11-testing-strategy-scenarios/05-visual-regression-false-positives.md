# Visual Regression False Positives — Handling Them

## Quick Reference

| Source of false positive | Fix | Why |
|---|---|---|
| Nondeterministic content (dates, avatars, ads, random data) | Freeze time, seed data, mock network, mask regions | Same input → same pixels |
| Rendering differences (fonts, antialiasing, OS/GPU) | Run in one pinned container image, wait for fonts | Pixel output depends on the rendering stack |
| Animations/transitions/carets/video | Disable animations, hide caret, pause media | Capture a settled, stable frame |
| Overly strict thresholds | Tune per-test `maxDiffPixelRatio`/threshold, scope to components | Tolerate sub-pixel noise, not real change |
| Capture timing | Wait for network idle + `document.fonts.ready` + stable layout | Avoid screenshotting mid-render |

## The Scenario

"We adopted visual regression testing (screenshot diffs on every PR). Three months in, engineers just click 'approve all' because ~30% of diffs are noise: fonts shift a pixel, a timestamp changes, a skeleton loader is caught mid-animation. How do you rescue this, and would you keep it at all?"

## Clarifying Questions

- **Where does it run — local dev machines, CI, a hosted service (Chromatic, Percy)?** If baselines are generated on macOS laptops and compared on Linux CI, you'll have noise by construction.
- **What are we capturing — full pages or isolated components/stories?** Full-page shots are far noisier (more dynamic content, more surface); component-level shots are tight and stable.
- **What's the review workflow — who approves, is it blocking?** The social failure mode ("approve all") matters as much as the technical one.
- **What's the ratio of false positives to true catches — do we have data?** Justifies whether it's worth keeping.
- **What did visual tests catch that nothing else would have?** The value case: CSS regressions, z-index/overflow bugs, dark-mode breakage.

## Approach & Trade-offs

**The real failure is trust, not pixels.** Once reviewers approve reflexively, the suite has zero value — worse, it has negative value (CI time, review friction). So the goal is a *low-noise* signal: a diff should almost always mean "a human should look at this."

**Diagnose by classifying the noise.** Cluster the last N false positives: dynamic data, font/antialiasing, animation, timing. Usually 2–3 causes make up most of it. Fix causes, not symptoms.

**Make renders deterministic.**

- *Environment:* run baselines and comparisons in the *same pinned Docker image* (e.g., the Playwright image), never mixing local and CI baselines. Font rasterization differs across OSes.
- *Data:* mock the network, fix `Date.now`, seed randomness, use static avatars.
- *Time and motion:* disable CSS animations/transitions (`animations: 'disabled'` in Playwright), hide the text caret, pause video.
- *Fonts:* wait for `document.fonts.ready`; self-host fonts to avoid CDN variance.

**Narrow the blast radius.** Prefer component/story-level snapshots to full-page ones; mask volatile regions (`mask: [locator]`) rather than diffing them; capture specific states (hover, error, dark mode) deliberately.

**Thresholds are a blunt instrument.** Raising global tolerance hides real subtle regressions (1px misalignments). Prefer fixing determinism, then a small per-test threshold only where antialiasing is inherently fuzzy.

**Process fixes.** Make review meaningful: diffs grouped per component with side-by-side/overlay views, PR-level ownership (the author of the change explains each diff), and fail—not warn—on unreviewed changes. Delete low-value snapshots.

**Trade-off: is it worth it at all?** For a design-system/component library or a CSS-heavy app, absolutely — it's the only automated check for visual correctness. For a small app with churning UI, the maintenance may outweigh the benefit; then scope to a handful of critical pages/components.

## Solution

### Deterministic Playwright config and test

```ts
// playwright.config.ts
export default defineConfig({
  expect: {
    toHaveScreenshot: {
      animations: 'disabled',
      caret: 'hide',
      maxDiffPixelRatio: 0.001,
    },
  },
  use: { colorScheme: 'light', locale: 'en-US', timezoneId: 'UTC', deviceScaleFactor: 1 },
});
```

```ts
test('invoice card', async ({ page }) => {
  await page.clock.setFixedTime(new Date('2025-01-15T10:00:00Z'));   // freeze time
  await page.route('**/api/invoice/42', r => r.fulfill({ json: fixtures.invoice }));
  await page.route('**/*.{png,jpg}', r => r.fulfill({ path: 'fixtures/avatar.png' }));
  await page.goto('/invoice/42');

  await page.evaluate(() => document.fonts.ready);
  await expect(page.getByRole('status')).toBeHidden();              // spinner gone

  await expect(page.getByTestId('invoice-card')).toHaveScreenshot('invoice-card.png', {
    mask: [page.getByTestId('relative-timestamp')],                 // inherently volatile
  });
});
```

### Pin the environment

```bash
# Run everywhere (local + CI) inside the same image so baselines match
docker run --rm -v $PWD:/work -w /work mcr.microsoft.com/playwright:v1.49.0-jammy \
  npx playwright test --update-snapshots   # only when intentionally rebaselining
```

### Component-level via Storybook

Snapshot each story state (default, loading, error, long text, RTL, dark) — tiny, isolated, and stable compared with whole pages.

### Triage workflow

1. Tag each failing diff with a cause (data / fonts / animation / timing / real).
2. Fix top cause; re-measure false-positive rate weekly.
3. Target: <2–3% of diffs are noise; track it as a metric.

> **Check yourself:** Why is "increase the diff threshold" usually the wrong first move, and what are the first three determinism fixes you'd apply?

## Gotchas

- **Baselines from a different OS than CI.** Guaranteed font/antialiasing noise; generate baselines only in the pinned container.
- **Screenshotting before fonts/images settle.** Wait deterministically, not with sleeps.
- **Raising the global threshold.** Silently lets real 1–2px regressions through.
- **Full-page snapshots of dynamic pages.** High surface, high noise; scope down.
- **Auto-approving.** The cultural failure; fix with ownership, small diffs, and metrics.
- **Not testing states.** A snapshot of only the default state misses error/hover/long-text/RTL regressions — where visual bugs live.
- **Blind rebaselining** (`--update-snapshots` on failure) turns the suite into a rubber stamp.

## Follow-up Questions

**Q (High): What causes most visual-diff false positives and how do you eliminate them?**

Answer: Dynamic content (dates, random data, remote images), rendering-environment differences (fonts, OS, GPU antialiasing), animations/carets, and capture-timing races. Eliminate with frozen time/mocked data, a single pinned container for baselines and CI, disabled animations, awaiting fonts and stable layout, and masking genuinely volatile regions.

The trap: reaching for threshold tuning first, or retrying until it passes.

**Q (High): How do you keep reviewers from blindly approving?**

Answer: Reduce noise so a diff means something; scope snapshots to components so each diff is small and attributable; require the PR author to own and explain each diff; use overlay/side-by-side review UI; and track false-positive rate publicly. Blocking merge on un-reviewed changes, not on diff existence.

The trap: treating it purely as a tooling problem and ignoring review culture.

**Q (Medium): Pixel diff vs. DOM/semantic checks — when to use which?**

Answer: Pixel diffs catch what DOM assertions can't (overlap, clipping, z-index, spacing, color) but are noisy and say nothing about *why*. DOM/role/computed-style assertions are stable and precise but blind to rendering. Use computed-style or DOM tests for specific invariants (e.g., button contrast token), and visual regression as a coarse net for layout/styling.

The trap: either-or thinking.

**Q (Medium): Would you run it on every PR, or on a schedule?**

Answer: Component-level snapshots on every PR (fast, scoped, affects the author directly). Full-page/cross-browser matrices on main or nightly, where slower and noisier runs don't block development, with failures triaged. Balance feedback speed against cost.

The trap: running a huge matrix on every PR and wondering why it's ignored.

**Q (Low): How do you handle cross-browser visual testing?**

Answer: Treat each browser as a separate baseline set (they legitimately render differently), limit to Chromium for broad coverage plus WebKit for known Safari risks, and avoid comparing across engines. Hosted services render in a controlled farm.

The trap: expecting pixel-identical output across engines.

## Self-Assessment

- [ ] Can name the five false-positive sources and one fix for each
- [ ] Can explain why baselines must be generated in the same pinned environment
- [ ] Can configure animations, caret, masking, and frozen time in a screenshot test
- [ ] Can explain why threshold inflation is a poor default
- [ ] Can describe the cultural fix for "approve all"
- [ ] Can argue when to keep vs. drop visual regression

---
*Next: Testing an Accessibility Requirement — closes the phase by asking what automation can and can't prove about a requirement that's partly human.*
