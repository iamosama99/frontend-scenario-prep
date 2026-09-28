# Dark Mode & Print Stylesheet Edge Cases

## Quick Reference

| Context | Mechanism | Common Gotcha |
|---|---|---|
| Dark mode, OS-driven | `@media (prefers-color-scheme: dark)` | Hardcoded colors (not tokens/custom properties) don't respond at all |
| Dark mode, user-toggleable (overrides OS) | A `data-theme`/class attribute on `<html>`, CSS custom properties keyed off it, JS syncing to `localStorage` | Flash of wrong theme on load if the theme is applied after first paint instead of before |
| Print | `@media print` | Fixed/sticky positioning, backgrounds, and off-screen/hidden interactive elements behave differently in print than screen |
| Images/icons across themes | `currentColor` for SVG icons, or theme-specific image sources | An SVG icon with a hardcoded `fill="#000"` stays black in dark mode, invisible against a dark background |
| Forms/inputs across themes | Native form controls' `color-scheme` CSS property | Without `color-scheme: dark`, native controls (checkboxes, date pickers, scrollbars) can stay light-themed and look out of place |

## The Scenario

"Two separate but related asks came in this sprint. First: users want dark mode — respecting their OS-level preference by default, with an in-app toggle to override it. Second: someone tried to print an invoice page and got a mess — a sticky header repeating on every printed page, a dark-mode-styled background wasting ink, and a 'Download PDF' button rendered uselessly in the middle of the printed page. Handle both: build dark mode properly, and fix the print output so a printed invoice looks like a clean document, not a scraped screenshot of the web page."

## Clarifying Questions

- **For dark mode: should the in-app toggle's choice persist across sessions (saved to `localStorage`/a user preference on the backend), and should it override the OS preference indefinitely once set, or only for the current session?** This affects both the storage mechanism and the precedence logic between `prefers-color-scheme` and the explicit toggle — a common, sensible default is "OS preference by default until the user explicitly overrides it via the toggle, then remember and respect that override going forward, but still let them switch back to 'follow system.'"
- **Does the color system already use CSS custom properties/design tokens for color values, or are colors hardcoded throughout the component styles?** This is the single most consequential unknown for scoping the dark-mode work — a codebase already using semantic color tokens (`--color-background`, `--color-text-primary`) needs "only" to define a second value set for each token under the dark condition; a codebase with hardcoded hex/rgb values scattered through component CSS needs a genuine refactor before dark mode is feasible at all, which is a much larger scoping conversation than the prompt's phrasing suggests.
- **For print: is the invoice page's only use case "print this specific document," or does the print stylesheet need to generalize across the whole app (any page a user might hit Cmd+P on)?** An invoice-specific print stylesheet can be tailored narrowly (hide the entire app chrome, show only the invoice content, in a known structure); a general-purpose print stylesheet for an arbitrary app page needs more defensive, broadly-applicable rules (hide anything interactive/nav-like by pattern, not by knowing exactly what's on the page).
- **Should printed output preserve any of the app's branding/color, or is "clean document, minimal ink usage" the explicit goal, implying grayscale/high-contrast print styling regardless of what theme (light/dark) was active on screen at print time?** This determines whether the print stylesheet should derive from the light theme's colors (common default, and implied by "wasting ink" in the prompt) or be its own explicitly-authored achromatic/print-optimized palette independent of either screen theme.
- **Does the invoice need to be printable/exportable as an actual PDF via a "Download PDF" flow as well as browser print, and if so, is that generated server-side (e.g., via a headless-browser rendering service) or purely via the browser's native print-to-PDF using this same print stylesheet?** If a server-side PDF generation pipeline already exists or is planned, the print stylesheet's scope might be "make browser Cmd+P output reasonable" as a secondary path, with the primary "Download PDF" button producing a separately-controlled, more reliably-formatted document — worth clarifying so effort isn't spent perfecting browser print rendering if it's not actually the primary path users are expected to take.

## Approach & Trade-offs

**Dark mode should be built on CSS custom properties as semantic tokens (`--color-surface`, `--color-text-primary`), not per-component hardcoded color overrides, because the token layer is what makes "respond to OS preference AND a manual toggle" tractable as a single mechanism instead of two.** Defining color *once*, as a small set of semantically-named custom properties at the `:root` level, and having every component reference those tokens (`background: var(--color-surface)`) rather than literal color values, means the entire theming problem reduces to "redefine the token values under two different conditions" — the components themselves never need theme-awareness at all, they just consume whatever the current token values resolve to. Without this layer, dark mode requires either a parallel `[data-theme="dark"] .component { ... }` override for every component (doubling color-related CSS and requiring perfect discipline to keep both in sync as components evolve) or, worse, inline/JS-computed color logic scattered through components — both of which don't scale and are exactly the kind of thing that silently breaks (a new component ships with a hardcoded color, doesn't respond to the toggle, isn't caught until a user reports it) the same way the physical-vs-logical-property gap did for RTL in [[06-rtl-layout-breaking]]. This is the same underlying lesson as that scenario: token-based/semantic values that get reinterpreted by context beat literal values that need to be manually mirrored everywhere they're used.

**Respect `prefers-color-scheme` as the default, and layer an explicit user toggle on top via a `data-theme` attribute that, when present, takes precedence over the OS-media-query-driven value — this ordering (OS default, explicit override takes precedence) matches both user expectation and is straightforward to express in CSS's cascade.** The OS-level preference should govern by default because it's already an explicit signal of the user's general preference across their whole system, and respecting it out of the box is both good UX and, for accessibility, sometimes not just a preference but a genuine need (some users configure dark mode for light-sensitivity/migraine reasons). Once the user explicitly interacts with an in-app toggle, that specific, deliberate choice should override the ambient OS signal until they change it again — implemented by having the dark-theme custom-property values apply either under the `prefers-color-scheme: dark` media query OR under a `[data-theme="dark"]` attribute selector (whichever is present takes effect; explicit attribute-based rules naturally take precedence in the cascade over media-query-scoped ones when both would otherwise apply, provided the attribute-based rule is written with at least equal specificity and appears appropriately in source order, or more robustly, by not relying on `prefers-color-scheme` for tokens at all once a `data-theme` attribute is explicitly set by JS on first load).

**Print stylesheets need a fundamentally different mental model than screen stylesheets — "hide everything that's interactive or navigational, show only the actual content, and don't assume any of the screen layout mechanisms (sticky, fixed, viewport units) behave the same way" — because print has no viewport, no scroll, and no interaction at all.** The reported bugs (sticky header repeating per page, dark background wasting ink, a button rendered uselessly) are all instances of screen-specific CSS being carried over into a context where its underlying assumptions don't hold: `position: sticky`/`fixed` has no meaning without a scrollable viewport (a printed page is not a scrolling context), and different browsers/print engines handle this differently — often re-rendering a `position: fixed` element on every physical printed page, which is exactly the "sticky header repeating on every printed page" bug described; a dark background color, if not explicitly suppressed, prints literally as ink-heavy dark background (browsers do have a "background graphics" print setting users can toggle, but relying on the user to know to disable it isn't a real fix — the print stylesheet itself should not require the background image/color to be suppressed to look reasonable); and any element whose entire purpose is a screen-only interaction (a "Download PDF" button, that has no possible function on an already-printed page) should simply not print at all.

**Build the print stylesheet by inverting the usual instinct — instead of trying to make everything look "print-friendly" through incremental tweaks, explicitly and aggressively hide everything not part of the actual content, then style what remains.** A page's full screen layout — navigation, sidebars, buttons, headers with app chrome — is almost entirely irrelevant to a printed document; the correct default posture for `@media print` is closer to "hide the whole app shell, show only the substantive content region" (`display: none` on nav/header/footer/buttons/any element with a clear "interactive, not content" semantic role) rather than trying to individually neutralize dozens of screen-specific properties (undo the sticky positioning, undo the dark background, undo the button's visual prominence) one at a time — the latter approach is exactly how the reported bugs accumulated in the first place, incrementally, without anyone stepping back to design the print output as its own deliberate thing.

## Solution — Dark Mode

**Step 1 — define semantic color tokens as CSS custom properties, with light values as the default and dark values under both the media query and an explicit override attribute:**

```css
:root {
  --color-background: #ffffff;
  --color-surface: #f5f5f5;
  --color-text-primary: #1a1a1a;
  --color-text-secondary: #595959;
  --color-border: #e0e0e0;
  color-scheme: light; /* tells the browser to style native controls (scrollbars, checkboxes) to match */
}

@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --color-background: #121212;
    --color-surface: #1e1e1e;
    --color-text-primary: #f0f0f0;
    --color-text-secondary: #b0b0b0;
    --color-border: #333333;
    color-scheme: dark;
  }
}

:root[data-theme="dark"] {
  --color-background: #121212;
  --color-surface: #1e1e1e;
  --color-text-primary: #f0f0f0;
  --color-text-secondary: #b0b0b0;
  --color-border: #333333;
  color-scheme: dark;
}
```

The `:root:not([data-theme="light"])` guard under the media query means: apply dark tokens when the OS prefers dark, *unless* the user has explicitly chosen light via the toggle (which sets `data-theme="light"`) — this is what lets an explicit light-mode choice override a dark OS preference, symmetric to the explicit dark-mode override.

**Step 2 — every component consumes tokens, never literal colors:**

```css
.card {
  background: var(--color-surface);
  color: var(--color-text-primary);
  border: 1px solid var(--color-border);
}
```

**Step 3 — the toggle sets `data-theme` on `<html>` and persists the choice, applied before first paint to avoid a flash of the wrong theme:**

```html
<!-- Inline, blocking script in <head>, before any stylesheet/content renders -->
<script>
  (function () {
    const saved = localStorage.getItem('theme'); // 'light' | 'dark' | null (= follow system)
    if (saved) document.documentElement.setAttribute('data-theme', saved);
  })();
</script>
```

```tsx
function ThemeToggle() {
  const [theme, setTheme] = useState<'light' | 'dark' | 'system'>(
    () => (localStorage.getItem('theme') as 'light' | 'dark') ?? 'system'
  );

  const applyTheme = (next: 'light' | 'dark' | 'system') => {
    setTheme(next);
    if (next === 'system') {
      document.documentElement.removeAttribute('data-theme');
      localStorage.removeItem('theme');
    } else {
      document.documentElement.setAttribute('data-theme', next);
      localStorage.setItem('theme', next);
    }
  };

  return (/* renders a three-way control: light / dark / system */);
}
```

Running the `data-theme` restoration as an inline, synchronous, render-blocking script in `<head>` — before the rest of the page's CSS/JS loads — is what prevents a "flash of wrong theme" (briefly rendering light, then snapping to dark once React hydrates and reads `localStorage`), a common and jarring dark-mode implementation bug.

**Step 4 — fix theme-unaware SVG icons and native form controls:**

```css
.icon {
  fill: currentColor; /* inherits the surrounding text color, which already responds to theme via the token system */
}
```

`color-scheme: light`/`dark` (set on `:root` alongside the token blocks in Step 1) is what makes native browser-rendered controls — scrollbars, checkboxes, date/color pickers, the default focus ring on some browsers — match the active theme automatically, without needing to restyle each one manually.

> **Check yourself:** Why is it important that the inline theme-restoration script in Step 3 runs synchronously and before any stylesheet, rather than as a normal `<script>` at the end of `<body>` or inside a React `useEffect`?

## Solution — Print Stylesheet

**Step 1 — hide the entire app shell by default in print, showing only the actual invoice content:**

```css
@media print {
  .app-header,
  .app-sidebar,
  .app-footer,
  .action-buttons,
  .download-pdf-button,
  nav,
  [data-print-hide] {
    display: none !important;
  }

  .invoice-content {
    display: block;
    width: 100%;
    margin: 0;
    padding: 0;
  }
}
```

**Step 2 — neutralize screen-specific positioning and coloring that has no valid meaning on a printed page:**

```css
@media print {
  * {
    position: static !important; /* sticky/fixed have no scrolling context to attach to on paper */
  }

  body {
    background: #ffffff !important; /* explicitly white, regardless of active screen theme (light or dark) at print time */
    color: #000000 !important;
  }

  .card,
  .invoice-content * {
    box-shadow: none !important; /* shadows print as gray smudges, pure visual noise on paper */
  }
}
```

Explicitly forcing `background: white` / `color: black` (rather than relying on the print stylesheet inheriting whichever theme tokens happen to be active) ensures the printed output is consistent and ink-conscious *regardless* of whether the user was in dark mode on screen at the moment they printed — this is the fix for the "dark-mode-styled background wasting ink" bug specifically, and is deliberately not derived from the token system at all, since print has its own, separate correctness requirement independent of screen theme.

**Step 3 — add print-specific affordances that only make sense on paper (page URLs, print date) if relevant, and control page-break behavior for multi-page invoices:**

```css
@media print {
  .invoice-content {
    page-break-inside: avoid; /* don't split an invoice's line-item table awkwardly across a page boundary, where reasonably avoidable */
  }

  .invoice-line-items tr {
    page-break-inside: avoid; /* keep individual rows intact rather than split mid-row */
  }

  a[href]::after {
    content: " (" attr(href) ")"; /* a link's destination is otherwise invisible/useless on paper */
  }
}
```

**Step 4 — verify with the browser's actual print preview, not just visual inspection of the screen with `@media print` styles toggled on in DevTools, since some behaviors (page breaks, headers/footers the browser itself adds, `position: fixed` repeating per page) only fully manifest in real print rendering:**

Chrome/Firefox DevTools can force-apply print media styles for quick iteration, but the *actual* page-break computation and any browser-injected print headers/footers (URL, page number, date — controllable by the user in the print dialog, but worth knowing about) are only visible in the real print preview dialog, which is why final verification needs to happen there, not purely in the emulated view.

> **Check yourself:** Why does `position: static !important` in the print stylesheet specifically address the "sticky header repeating on every printed page" bug — what was actually causing the repetition in the first place?

## Gotchas

**Building dark mode by adding a parallel set of hardcoded dark colors per-component (`[data-theme="dark"] .card { background: #1e1e1e }`) instead of a token layer** — this technically works for the initial ship but means every future component needs the same manual dark-mode override remembered and written correctly, with no structural guarantee it happens; the token-based approach makes correctness the default, not something that has to be separately re-earned for every new component.

**Forgetting `color-scheme` on `:root`**, leaving native form controls, scrollbars, and default focus indicators visually light-themed against a dark-themed page — a subtle but very noticeable "looks unfinished" signal, especially for scrollbars, which many engineers don't think to check since they're not typically styled through ordinary component CSS at all.

**A flash of incorrect theme on page load** caused by restoring the saved theme preference only after JS hydrates/mounts (a `useEffect` in a component, rather than a synchronous blocking script in `<head>`) — the page renders in the default/OS theme for a frame or more, then visibly snaps to the user's actually-saved preference, which reads as a bug/flicker even though the "wrong" theme was only ever shown extremely briefly.

**Testing dark mode and print in isolation from each other and missing the interaction case this scenario specifically calls out** — a user who has dark mode active on screen and prints without deliberate print-specific color overrides gets a dark, ink-heavy printed page unless the print stylesheet explicitly forces light colors regardless of the active screen theme; this is exactly the kind of cross-cutting edge case ("what happens when both conditions apply at once") that's easy to miss when each feature is built and tested independently.

**Relying on `display: none` alone to hide interactive elements from print, without considering that some browsers' print dialogs let users still choose to print background graphics/colors**, meaning a page that *visually* relies on a background-color print toggle behaving a specific way, rather than an explicit `!important` override in the print stylesheet, is at the mercy of a setting most users never touch (default is often "don't print background graphics," but this isn't guaranteed, and explicit stylesheet control removes the ambiguity entirely).

**Not using `!important` deliberately (or not understanding why it's more defensible here than in ordinary component CSS)** in the print stylesheet specifically — `@media print` rules often need to override a wide, unpredictable variety of screen-specific styling (inline styles, high-specificity component CSS, third-party widget styles) that the print-stylesheet author doesn't want to have to individually out-specificity; `!important` is one of the few places in CSS where its use is broadly considered acceptable practice precisely because print styles are meant to be an unconditional, final override layer, not part of the app's normal cascading style architecture.

## Follow-up Questions

**Q (High): Why does `position: fixed`/`sticky` cause an element to repeat on every physical printed page, and why does `position: static` in the print stylesheet fix it?**

Answer: `position: fixed` positions an element relative to the viewport (or, in a browser's print-rendering model, relative to *each individual printed page*, which the browser's print engine treats analogously to a viewport for layout purposes) — since a multi-page printed document is conceptually a sequence of individual "viewports" (pages) rather than one continuous scrollable surface, an element fixed relative to "the viewport" gets recomputed and re-rendered at that same fixed position on *every* page the browser lays out content across, which is exactly the "header repeats on every printed page" symptom — it's not a bug in the sense of behaving randomly, it's the fixed-positioning model being applied literally and consistently to a context (multi-page print) where "the viewport" now means something different (and plural) than it did on screen (singular, scrollable). Forcing `position: static` (or `relative`) in the print stylesheet removes the element from being positioned relative to any page/viewport at all, returning it to normal document flow — where it appears exactly once, in its natural place in the content order, and simply flows onto whichever single page that content naturally lands on, like any other static content.

The trap: describing this as an inconsistent or buggy browser behavior rather than a logical (if surprising) consequence of what `position: fixed` actually means once "the viewport" is reinterpreted as "each printed page" — understanding it this way is what makes the fix (`position: static !important` in print) an obvious, principled choice rather than a discovered-by-trial-and-error workaround.

---

**Q (High): A component ships with `background-color: white` hardcoded (not a token) because "the design always wanted it white regardless of theme." Is this ever legitimately correct, and how would you distinguish that case from a bug during a dark-mode code review?**

Answer: Yes, this can be legitimately correct — some content genuinely should stay a fixed color regardless of theme (a brand logo mark with a specific required background, an embedded photo's letterboxing, a color swatch whose entire purpose is showing an exact, theme-independent color value) — the distinguishing question during review isn't "is this a hardcoded color" (which is sometimes fine) but "does the semantic *purpose* of this color depend on which theme is active, or is it inherently theme-independent." A card background, a body text color, a border — these represent generic UI surface/content roles whose whole reason for having a specific color value *is* theme-dependent (their purpose is "the background of this content area," and what that should look like is exactly what changes between light and dark) — those should always be tokens. A logo's specific brand color, or a genuinely fixed-purpose swatch, has a purpose that doesn't change based on theme, and hardcoding it is the more correct choice, not a shortcut. The practical review heuristic: ask "if we shipped a third theme (e.g., a high-contrast mode) tomorrow, would this specific color need to change along with the theme, or does it need to stay exactly what it is regardless?" — if the former, it should be a token; if the latter, hardcoding it is deliberate and correct, not an oversight.

The trap: treating "no hardcoded colors, ever" as an absolute rule during code review, flagging every literal color value as a bug — a thoughtful reviewer distinguishes semantically theme-dependent color usage (should be tokenized) from genuinely theme-independent, fixed-purpose color usage (legitimately hardcoded), rather than applying a blanket rule that doesn't hold up for real design requirements like brand marks.

---

**Q (Medium): The inline theme-restoration script in `<head>` runs before React hydrates and reads `localStorage` synchronously to set `data-theme` on `<html>`. Does this create any issue with server-side rendering (SSR), where the server doesn't have access to the browser's `localStorage` at render time?**

Answer: Yes — this is a real SSR-specific edge case worth naming explicitly. The server, rendering the initial HTML, has no access to the browser's `localStorage` and therefore cannot know the user's saved theme preference at render time — it can only render with some default assumption (typically no `data-theme` attribute at all, letting the `prefers-color-scheme` media query govern until the client-side inline script runs and potentially overrides it). This means there's an unavoidable brief window, specific to SSR, where the server-rendered HTML reflects either the OS-preference-driven theme (if `prefers-color-scheme` alone is used server-side, which itself requires reading a request header like `Sec-CH-Prefers-Color-Scheme` if using client hints, or simply not attempting to guess server-side at all) rather than the user's actual saved override — the inline script, running as early as possible on the client, minimizes but can't fully eliminate this gap for SSR specifically, in a way that a pure client-side-rendered app (where nothing is visible until JS runs anyway) doesn't have to contend with. Some frameworks/setups mitigate this further by reading a theme preference from a cookie (sent with the initial request, so the server *can* see it, unlike `localStorage`) instead of or in addition to `localStorage`, specifically to close this SSR gap.

The trap: assuming the inline-script approach from Step 3 fully eliminates flash-of-wrong-theme in *every* rendering setup — it's a strong fix for client-side-rendered/hydration timing, but SSR specifically introduces an additional, distinct gap (server has no `localStorage` access) that the same technique doesn't fully close without an additional mechanism like a theme-preference cookie readable server-side.

---

**Q (Medium): Someone suggests handling print entirely differently — generating a server-side PDF (e.g., via a headless browser or a PDF-generation library) instead of relying on the browser's native print stylesheet. What's the trade-off?**

Answer: A server-side PDF generation pipeline (headless Chrome/Puppeteer rendering a dedicated print-optimized HTML template, or a purpose-built PDF library assembling the document programmatically) gives materially more reliable, consistent output — the same PDF renders identically regardless of which browser or OS the *user* is on, sidesteps browser-specific print-engine quirks (like the fixed-position-repeats-per-page behavior, or inconsistent page-break handling across browsers) entirely, and can support richer layout control (precise page numbering, headers/footers, consistent typography) than CSS `@media print` easily achieves. The trade-off is real infrastructure cost and complexity: it requires a server-side rendering service (or a PDF-generation library with its own learning curve and limitations), adds latency (generating a PDF server-side and downloading it isn't instant the way browser-native print is), and creates a second rendering pipeline that needs to be kept in sync with the actual invoice content/design as it evolves — effectively maintaining two representations of the same document (the on-screen HTML/CSS version and the server-rendered PDF version) rather than one. For a low-volume, "occasionally a user wants a printed copy" feature, the CSS print-stylesheet approach is usually proportionate; for a business-critical, must-look-pixel-perfect-every-time document (a legal invoice, a compliance document), the server-side approach's reliability often justifies its added cost.

The trap: treating one approach as strictly superior in all cases — server-side PDF generation solves real cross-browser inconsistency problems that a hand-tuned `@media print` stylesheet can't fully guarantee away, but "always use server-side PDF generation" ignores that it's meaningfully more infrastructure for a use case that a well-built print stylesheet may already serve adequately, and the right choice depends on how business-critical exact, consistent output actually is for this specific document type.

---

**Q (Low): Would `prefers-contrast` or `forced-colors` (Windows High Contrast Mode) need separate handling from the dark-mode work done here, or does the token-based dark-mode system already cover them?**

Answer: They need separate, explicit handling — `prefers-color-scheme` and `prefers-contrast`/`forced-colors` are distinct media features addressing different user needs, and a dark-mode implementation doesn't automatically satisfy high-contrast requirements just because both involve "non-default" color schemes. `prefers-contrast: more` signals a user wants higher contrast than the default (light or dark) theme provides, which the existing token system could reasonably extend to serve (defining a third set of token values tuned for higher contrast ratios, under `@media (prefers-contrast: more)`), but isn't automatically satisfied by having only "light" and "dark" token sets, since a dark theme isn't inherently high-contrast (dark-on-dark low-contrast UI is a common real accessibility failure). `forced-colors: active` (triggered by Windows High Contrast Mode and similar OS-level accessibility features) is a more fundamentally different mode — the OS overrides author-specified colors with a constrained system palette entirely, and author CSS needs to specifically avoid relying on background images/colors for conveying meaning and use the `forced-colors` media query along with system color keywords (`Canvas`, `CanvasText`, `LinkText`, etc.) to ensure the UI remains usable when the browser is actively overriding most color declarations — a fundamentally different mechanism than the custom-property token-swapping approach used for light/dark mode.

The trap: assuming "we support dark mode" implies broader color-accessibility needs are automatically covered — `prefers-contrast` and `forced-colors` are separate media features addressing separate, real user needs (some users need high contrast regardless of light/dark preference; some need the OS's forced-colors mode to work correctly with the app at all), and treating dark-mode support as a complete answer to "does this app handle user color preferences" undersells the distinct work still needed for those other cases.

---

## Self-Assessment

- [ ] Can explain why a CSS custom property token layer is what makes both OS-preference-driven and manually-toggled dark mode tractable as one mechanism
- [ ] Can explain the flash-of-wrong-theme bug and why the fix requires a synchronous, render-blocking script, not a `useEffect`
- [ ] Can explain precisely why `position: fixed`/`sticky` causes repetition across printed pages, and why `position: static` in print fixes it
- [ ] Can explain why print colors need to be explicitly forced (not inherited from the active screen theme) regardless of light/dark mode
- [ ] Can articulate when a hardcoded (non-token) color is legitimately correct vs. a dark-mode bug during review
- [ ] Can name `prefers-contrast`/`forced-colors` as distinct, additional concerns not automatically solved by light/dark theming

---
*Next: Phase 9, Scenario 1 — Accessible Modal, Full Scenario Walkthrough. Closes the CSS/layout-debugging phase and opens accessibility as its own dedicated phase — a natural continuation, since several bugs in this phase (the modal's background-scroll containment in [[07-nested-scroll-container-overflow-trap]], the `dir`-aware focus order implications in [[06-rtl-layout-breaking]]) already touched accessibility concerns that phase 9 treats as the primary subject rather than a side note.*
