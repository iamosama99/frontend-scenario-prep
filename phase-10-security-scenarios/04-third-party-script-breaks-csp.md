# Third-party Script Breaks Your CSP

## Quick Reference

| Violated directive | Typical symptom | Fix (in order of preference) |
|---|---|---|
| `script-src` | Widget's `<script>` tag silently fails to load; console shows a CSP violation, not a network error | Add the vendor's exact script origin to `script-src` — never `'unsafe-inline'`/`'unsafe-eval'` as a first resort |
| `style-src` | Widget renders unstyled or not at all; injected `<style>`/inline `style=` attributes get dropped | Prefer a nonce the vendor can accept, or the vendor's own stylesheet URL; `'unsafe-inline'` on `style-src` only is a smaller, more defensible concession than on `script-src` |
| `connect-src` | Widget loads visually but never receives data — `fetch`/`XHR`/WebSocket calls silently fail | Add the vendor's API origin(s) explicitly; check whether they use a CDN with rotating/many subdomains |
| `frame-src` | An embedded iframe (chat window, video) refuses to render | Add the vendor's frame origin explicitly, ideally scoped as narrowly as the vendor's docs allow |

## The Scenario

"Marketing added a live-chat widget to the site last week via a `<script>` snippet the vendor gave them. Since then, our CSP violation reports have exploded, and support is getting complaints that the chat bubble never shows up. Diagnose it, and tell me how you'd fix it — without just disabling our CSP to make the errors go away."

The explicit "without just disabling CSP" framing in the prompt is intentional — it's there to see whether the candidate reaches for the blunt, security-defeating fix under time pressure or reasons through what the CSP is actually protecting against and preserves that protection while accommodating the vendor.

## Clarifying Questions

- **What does the actual CSP violation report say — which directive, and what blocked URI/inline resource?** "CSP violation" isn't specific enough to act on; I need to see whether it's `script-src` (the script itself can't load or execute), `style-src` (injected styles are being stripped), `connect-src` (the script loads but its network calls are blocked), or `frame-src` (it's trying to open an iframe) — each has a different fix, and a widget commonly violates more than one simultaneously.
- **Is our CSP currently in `Content-Security-Policy` (enforcing) mode or `Content-Security-Policy-Report-Only` mode?** This determines whether the described symptom ("chat bubble never shows up") is actually being caused by CSP at all — in report-only mode, violations are logged but nothing is actually blocked, so if the widget is *also* broken, something else might be going on, and I don't want to spend the review chasing a CSP fix for a bug CSP isn't actually causing.
- **Does the widget's script dynamically inject further scripts, styles, or iframes at runtime (as most chat/support widgets do), or is it a single static `<script src>` tag?** A vendor script that only loads once from a known URL is a much smaller CSP surface to accommodate than one that dynamically constructs a whole UI with inline styles and secondary requests — I need to know the shape of what it's actually doing before I can write policy for it.
- **Do we have any visibility into what domains/behavior the vendor's script needs, beyond reverse-engineering it from violation reports?** Reputable vendors usually publish a documented CSP snippet for exactly this reason; I'd rather get an authoritative list from their integration docs than infer one purely from whatever happens to show up in our violation log, which only tells me what's been tried so far, not everything the script might do under different conditions.
- **How much do we trust this specific vendor, and is this widget handling anything sensitive (payment info, PII) on the page it's embedded on?** A vendor requiring `'unsafe-eval'` or broad wildcard origins is a materially bigger trust extension than one that just needs a couple of specific origins added — the answer to "how far do we compromise the policy for this vendor" should scale with how much we're choosing to trust their code running on our origin.

## Approach & Trade-offs

**Switch to report-only mode first, if not already there, to get the complete list of violations before making any changes.** Fixing CSP reactively — patch one violation, redeploy, wait for the next report, repeat — is slow and risks shipping a series of narrow patches that never quite catch up to everything the widget does. `Content-Security-Policy-Report-Only` alongside the existing enforced policy lets me collect every violation the widget triggers across a normal usage window (including code paths only hit on certain user interactions, like opening the chat window or submitting a message) without breaking anything further in the meantime, then write one complete, deliberate policy update instead of a series of whack-a-mole patches.

**Add the minimum set of exact origins needed, not broad wildcards or blanket `'unsafe-*'` keywords.** The instinct under pressure — "just add `'unsafe-inline'` to `script-src` and the errors stop" — defeats the actual purpose of having a CSP in the first place: `script-src 'unsafe-inline'` re-opens exactly the injected-script execution vector that a strict CSP exists to close (this is the same defense-in-depth mechanism discussed in the [Stored XSS](01-stored-xss-found-in-production.md) scenario, in reverse — there, a strict CSP was the safety net catching a sanitization bug; here, weakening it back to `'unsafe-inline'` removes that safety net for the entire site, not just for the widget). The correct, narrower fix is adding the vendor's actual script origin(s) to `script-src` by exact domain, which permits the specific script we've chosen to trust while leaving the policy's protection against *arbitrary* injected inline scripts fully intact.

**`style-src 'unsafe-inline'` is a materially smaller concession than `script-src 'unsafe-inline'`, and sometimes the pragmatic one to accept.** If the widget dynamically injects `<style>` tags or inline `style=` attributes it doesn't control the exact content of ahead of time (common for a UI that positions itself based on runtime measurements), a nonce or hash-based allowlist may not be achievable without the vendor's cooperation. Inline CSS can't execute JavaScript, so `'unsafe-inline'` scoped *only* to `style-src` reopens CSS-based risks (data exfiltration via attribute selectors in some old/unusual configurations, UI redressing) rather than script execution — a real but much narrower risk than the same keyword on `script-src`, and one I'd accept as a documented, deliberate trade-off rather than treat as equivalent to loosening script execution.

**Consider sandboxing the widget in an iframe with its own separate, restrictive CSP if the vendor's script needs are broad and trust is uncertain.** If accommodating the widget's actual behavior would require meaningfully weakening the *main page's* policy — the page that also handles login, payment, or other sensitive flows — the better trade-off is often isolating the widget inside an `<iframe sandbox>` with its own CSP header, scoped tightly to only what that widget needs, communicating with the parent page via `postMessage` if any interaction is required. This costs real integration complexity (the widget vendor has to support being framed this way, and any parent-page interaction needs explicit message-passing) but means a compromised or misbehaving third-party script is contained to its own frame's origin and policy rather than inheriting the full page's trust level.

## The Fix

Given a violation log showing the widget loading a script from `https://cdn.chatvendor.com`, injecting inline styles for its floating bubble, and making API calls to `https://api.chatvendor.com`:

```
# Before — widget silently broken, or console flooded with violations
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self'; connect-src 'self';

# After — minimum additions to accommodate the vendor, nothing broader
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.chatvendor.com;
  style-src 'self' https://cdn.chatvendor.com 'nonce-r4nd0mPerRequestValue';
  connect-src 'self' https://api.chatvendor.com;
  frame-src https://chat.chatvendor.com;
```

The `style-src` nonce specifically only helps if the vendor's script supports reading a nonce we generate per-request and applying it to the `<style>` tags it injects — most reputable vendors document exactly this ("pass our snippet a `data-csp-nonce` attribute and we'll apply it"), because they've hit this exact problem with enough of their customers to have solved it. If the vendor's script has no such support and injects raw inline styles with no way to attach a nonce, the realistic options narrow to: `style-src 'unsafe-inline'` as a scoped, documented concession; asking the vendor whether they offer a CSS-file-based version of their widget instead of inline-injected styles; or the iframe-sandboxing approach above if the trust/risk calculus doesn't favor either compromise.

> **Check yourself:** Explain why `script-src 'self' https://cdn.chatvendor.com` is meaningfully safer than `script-src 'self' 'unsafe-inline'` even though both "make the vendor's script work" — what specific attack does the first version still block that the second doesn't?

## Gotchas

**Reaching for `'unsafe-inline'`/`'unsafe-eval'` as the fast, generic fix.** It's almost always the fastest way to make CSP violation errors disappear, and that's exactly the trap — it disappears the *symptom* by removing the *protection*, silently reopening the injected-script execution vector a strict CSP exists specifically to close for the entire origin, not just for the widget causing the errors.

**Using a wildcard like `https://*.chatvendor.com` for convenience instead of the exact subdomain(s) actually needed.** This is more convenient if the vendor rotates or adds subdomains, but it also means trusting *any* current or future subdomain the vendor (or anyone who compromises a piece of their infrastructure) ever stands up — a real widening of the trust boundary that should be a deliberate choice, not a default reached for to avoid maintaining an exact list.

**Not checking whether the CSP is in report-only or enforcing mode before diagnosing the "widget doesn't work" symptom.** If it's report-only, CSP isn't blocking anything — the widget being visually broken is a separate bug, and chasing a CSP fix for it wastes time and risks masking the actual cause.

**Treating a CSP fix for one vendor script version as permanent.** Vendors update their scripts, sometimes adding new origins, new inline-injection patterns, or new API endpoints without notifying integrators — a CSP tuned to today's version of the widget can silently start failing again after the vendor's next release, which argues for keeping the report-only reporting endpoint active alongside the enforced policy long-term, not just during the initial fix.

**Confusing CSP with Subresource Integrity (SRI) as solving the same problem.** CSP controls *where* scripts/styles/connections are allowed to come from; SRI (a `integrity="sha384-..."` attribute on a `<script>`/`<link>` tag) verifies that the file actually fetched from an allowed origin hasn't been tampered with in transit or at the CDN. Adding a vendor's origin to `script-src` without SRI means the policy trusts *that origin's content, whatever it currently is* — SRI is the complementary control for when a script is served from a source I don't fully control and want to pin to a specific, verified version.

## Follow-up Questions

**Q (High): Why is adding the vendor's exact script origin to `script-src` meaningfully safer than adding `'unsafe-inline'`, if both make the widget work?**

Answer: `script-src 'self' https://cdn.chatvendor.com` permits execution of scripts loaded from exactly two trusted origins and nothing else — an attacker who manages to inject an inline `<script>` tag into the page via some other bug (an XSS vulnerability elsewhere, as in the [Stored XSS](01-stored-xss-found-in-production.md) scenario) still can't get their injected script to execute, because inline scripts remain blocked regardless of this addition. `script-src 'unsafe-inline'`, by contrast, permits *any* inline script anywhere on the page to execute — it doesn't just accommodate the vendor's legitimate script, it removes CSP's protection against inline script injection entirely, for the whole origin, for every current and future bug. The vendor-origin approach narrows trust to a specific, named third party whose script content is presumably reviewed/trusted at integration time; the `'unsafe-inline'` approach extends trust to "any script that ends up inline in this page's HTML, from any source, forever."

The trap: treating both as equivalent because "the errors stop either way" — the interviewer is checking whether the candidate understands CSP's actual protective mechanism (restricting *sources* of script) well enough to see that one fix preserves it and the other discards it.

---

**Q (High): The vendor's widget injects inline `<style>` tags dynamically and doesn't support nonces. Walk through the actual options and their trade-offs.**

Answer: Nonce-based allowlisting requires the vendor's own script to read a server-generated nonce (typically via a `data-*` attribute we pass into their snippet) and apply it to the `<style>` tags it creates — if their script has no support for this, we can't retroactively add a nonce to content their code generates, since the nonce has to be present in the actual `<style>` tag's attribute at creation time. Hash-based allowlisting (`style-src 'sha256-...'`) only works for *static, unchanging* inline content, since the hash has to match exactly — a widget that generates styles dynamically (e.g., positioning based on runtime measurements) will produce different content each time, making a fixed hash unworkable. That leaves three realistic paths: (1) ask the vendor if they offer an external-stylesheet-based integration instead of inline-injected styles — the cleanest fix if available; (2) accept `style-src 'unsafe-inline'` as a scoped, documented trade-off, reasoning that inline CSS can't execute arbitrary JavaScript the way `script-src 'unsafe-inline'` would, so the residual risk (CSS-based data exfiltration via attribute selectors, UI redressing) is real but narrower; or (3) isolate the widget in a sandboxed iframe with its own separate, more permissive CSP, containing the concession to that frame rather than the whole page. Which of the three is right depends on how sensitive the rest of the page is and how much the team trusts this specific vendor.

The trap: assuming a nonce can just be "added" without the vendor's script cooperating — the nonce has to be present as an attribute on the inline element at the moment it's created, which is entirely under the third-party script's control, not something our own CSP configuration can retrofit onto content we don't generate.

---

**Q (High): What's the actual difference between `Content-Security-Policy` and `Content-Security-Policy-Report-Only`, and how would you use both together to roll this fix out safely?**

Answer: `Content-Security-Policy` is enforced — any violation is actually blocked (the script doesn't execute, the style doesn't apply, the connection doesn't fire) and reported if a `report-uri`/`report-to` endpoint is configured. `Content-Security-Policy-Report-Only` describes a policy that's evaluated and *reported* on violation but never actually blocks anything — it's a dry run. The safe rollout pattern for a change like this is to send both headers simultaneously during the transition: keep the current, working enforced policy live (so nothing actually breaks further for real users) while adding a `Report-Only` header carrying the *proposed* new, widened policy, and let that run in production for a representative window (long enough to capture less-common code paths — opening the chat window, submitting a message, whatever triggers parts of the widget's behavior that a quick manual test might miss). Once the report-only log shows the proposed policy produces zero further violations across that window, promote it to the enforced header. This avoids the two failure modes of doing it live: shipping a policy that's still too narrow (breaks the widget again) or shipping one that's needlessly broad because it was guessed rather than measured.

The trap: assuming report-only mode is only useful for initially rolling out CSP on a site that's never had one — it's equally valuable, and arguably more valuable, for validating *changes* to an already-enforced policy without a live-breakage risk during the transition.

---

**Q (Medium): Where does Subresource Integrity fit in relative to the CSP fix here, and would you add it?**

Answer: SRI and CSP solve different, complementary problems — CSP's `script-src` addition says "scripts from `cdn.chatvendor.com` are allowed to run," while SRI's `integrity` attribute says "and specifically, only if the fetched content's hash matches this exact expected value," protecting against the CDN itself being compromised, a man-in-the-middle substituting different content, or the vendor's own infrastructure serving something unexpected. I'd add SRI here if the vendor publishes a versioned, pinned script URL with a stable hash (many do, alongside their documented CSP snippet) — it's a low-cost addition that meaningfully narrows what "trusting `cdn.chatvendor.com`" actually means, from "whatever that origin currently serves" to "specifically this exact, verified file." The practical limitation: if the vendor serves a script designed to auto-update without a version-pinned URL (common for widgets that want to ship fixes without every integrator re-deploying), SRI isn't compatible with that model, since the hash would break on every vendor-side update — that's a trade-off to have explicitly with the vendor relationship, not a reason to skip SRI silently where it is available.

The trap: treating CSP's `script-src` addition as making SRI redundant — they answer different questions ("is this origin allowed to serve scripts here" vs. "is this specific content what I expect from that origin"), and a compromised-but-still-same-origin CDN defeats CSP's origin check while SRI would still catch it.

---

**Q (Medium): When would you choose to sandbox the widget in its own iframe with a separate CSP instead of widening the main page's policy?**

Answer: When the concessions the widget needs are broad enough, or the vendor is trusted little enough, that widening the *main* page's policy to accommodate it would meaningfully weaken protection for everything else on that page — particularly if the same page also handles sensitive flows like login or payment. Isolating the widget in a sandboxed iframe with its own, separately-scoped CSP means the concession (say, a looser `script-src` or `'unsafe-inline'` for styles) only applies within that frame's own security context, not to the parent document. The real cost: any interaction the widget needs with the parent page (passing user info into the chat, resizing the iframe to fit content, closing itself) now has to go through explicit, deliberately-designed `postMessage` communication rather than direct DOM access, which is real integration work the vendor's script has to support (most chat-widget vendors do support an iframe-embeddable mode for exactly this reason, since many of their customers make this same trade-off).

The trap: assuming iframe-sandboxing is a "free" containment strategy with no trade-off — it works well specifically because it restricts what the widget can do, which means any legitimate interaction the widget needs with the host page has to be explicitly re-plumbed through message-passing rather than assumed to work automatically.

---

**Q (Low): The violation report shows the widget attempting to call `eval()` internally. Should you add `'unsafe-eval'` to accommodate it?**

Answer: I'd treat this as a strong signal to push back on the vendor rather than accommodate it by default — `'unsafe-eval'` permits `eval()`, `new Function()`, and similar string-to-code execution anywhere on the page, which is one of the more dangerous CSP concessions available, since it reopens a broad class of injection techniques that don't even require getting an inline `<script>` tag into the DOM (a string passed to a vulnerable `eval()` call elsewhere in the page becomes exploitable again). Many legitimate reasons a vendor's bundler-era build might touch something `eval`-adjacent turn out, on closer inspection, to be avoidable (older webpack `eval` devtool source maps accidentally shipped to production is a common real-world cause) — I'd ask the vendor directly whether this is required for their widget to function or is an artifact of their build config, since the latter is often just a vendor-side fix away, and I'd treat granting `'unsafe-eval'` broadly as a last resort reserved for cases where the vendor genuinely can't avoid it and the trust/risk trade-off has been explicitly discussed, not something to grant reflexively to silence the violation.

The trap: treating `'unsafe-eval'` as just another origin to allowlist, similar in weight to adding a script-src domain — it's categorically broader (it's not scoped to any origin at all, it permits the dangerous *operation* anywhere script already runs) and deserves proportionally more scrutiny before being granted.

---

## Self-Assessment

- [ ] Can identify which CSP directive a given symptom (blocked script, missing styles, silent network failures) points to
- [ ] Can explain, unprompted, why `script-src 'unsafe-inline'` is a worse fix than adding the vendor's exact origin
- [ ] Can describe the report-only-then-promote rollout pattern for changing an already-enforced CSP
- [ ] Can explain why nonces don't work for third-party inline content the vendor's script doesn't cooperate with
- [ ] Can articulate the iframe-sandboxing trade-off (containment vs. postMessage integration cost)
- [ ] Can distinguish what CSP protects against from what SRI protects against

---
*Next: Clickjacking Risk on an Embeddable Widget — shifts from "a script you embed" to "your own widget being embedded elsewhere," the mirror-image integration-security problem.*
