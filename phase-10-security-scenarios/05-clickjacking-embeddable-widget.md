# Clickjacking Risk on an Embeddable Widget

## Quick Reference

| Defense | What it stops | Limitation |
|---|---|---|
| `X-Frame-Options: DENY` / `SAMEORIGIN` | Framing entirely, or framing by a different origin | Only one allowed origin, no list — can't support multiple named partners |
| `Content-Security-Policy: frame-ancestors <list>` | Same goal, modern replacement | Supports an explicit allowlist of multiple origins — the right tool when there are known partners, useless if *anyone* must be able to embed |
| Frame-busting JS (`if (top !== self) top.location = self.location`) | Nothing reliable | Trivially defeated by an attacker embedding the page inside a `sandbox` iframe lacking `allow-top-navigation` |
| Forcing the sensitive action to a top-level navigation/popup instead of an in-iframe click | Clickjacking even when arbitrary embedding is a hard requirement | Real UX/integration cost — the "click to authorize" can no longer be a single inline click |

## The Scenario

"We're launching an embeddable widget partners can drop into their sites — a single button that lets their users connect their bank account to our platform. Security review just flagged it: 'this is directly clickjackable — a malicious page could overlay this iframe under a fake "claim your prize" button and trick users into authorizing a connection they never meant to make.' We do want this to be broadly embeddable — some partners are known and vetted, but we're also considering letting any website embed a lighter public version. How do you actually fix this, for both cases?"

The scenario is deliberately split into two sub-cases — known/vetted partners versus arbitrary public embedding — because the correct defense is genuinely different for each, and conflating them is the most common mistake candidates make here.

## Clarifying Questions

- **Does the widget perform a sensitive, state-changing action on a single click (authorize a bank connection, in this case), or is it purely informational?** Clickjacking is a real but low-severity nuisance for a purely display-only widget; it's a serious vulnerability specifically because a single deceived click here triggers an authorization the user never consciously intended.
- **Who is actually allowed to embed this — a known, finite list of vetted partners, or literally any site on the internet?** This is the single question that determines which category of fix applies at all: a finite, known partner list can be enforced with a `frame-ancestors` allowlist at the HTTP layer; "anyone can embed this" makes header-based origin restriction structurally unusable, since there's no fixed list to write, and pushes the fix toward redesigning the sensitive action itself.
- **Is the widget currently frameable from any origin, or does it already have some framing restriction that this review is asking to tighten?** I want to know the current baseline — whether this is "add a missing header" or "the header exists but is misconfigured/too permissive" changes where I start looking.
- **Do partners need to visually customize the widget (their own CSS, transparency, custom sizing)?** Heavy customization support, especially anything allowing transparency or custom positioning, makes it easier for a legitimate partner's own page (or a compromised one) to inadvertently or deliberately create exactly the kind of deceptive overlay the review is worried about — worth knowing whether that flexibility is already a requirement I have to preserve.
- **Has anyone specced what "the public embed version" is actually allowed to do — is it the same one-click authorize action, or a reduced, view-only version?** If the public tier is meant to have reduced capability specifically because it can't be origin-restricted, that's a product decision worth surfacing explicitly rather than assuming the full authorize flow is available in both tiers.

## Approach & Trade-offs

**For the known-partner case: `Content-Security-Policy: frame-ancestors` is the correct, modern header — not `X-Frame-Options`.** `X-Frame-Options` only supports `DENY`, `SAMEORIGIN`, or a single origin via the largely unsupported and non-standard `ALLOW-FROM` value — it simply cannot express "allow these five specific partner origins," which is exactly the shape of the known-partner requirement here. `frame-ancestors` (a CSP directive) natively supports a space-separated list of permitted origins, is enforced by all modern browsers, and is the header I'd write as the actual fix for that case. I'd still send `X-Frame-Options: SAMEORIGIN` alongside it as a defense-in-depth fallback for any legacy browser that doesn't honor CSP's `frame-ancestors` — it won't allow the multi-partner list, but a same-origin-only fallback is strictly safer than no fallback at all for browsers that miss the modern header.

**For the "anyone can embed this" public case, header-based allowlisting is structurally the wrong tool, and I'd say so directly rather than trying to force it to work.** There is no finite list to write into `frame-ancestors` if the requirement is genuinely "any website" — leaving it wide open (`frame-ancestors *` or omitting the header) is exactly the vulnerability the review flagged, but restricting it to a list defeats the "any site" requirement. The actual fix here has to be structural, not header-based: redesign the sensitive action so a single in-iframe click can never, by itself, complete the authorization. Concretely, that usually means the "Connect your bank" click inside the iframe opens a genuine top-level popup window or performs a top-level navigation (not a nested iframe interaction) to complete the actual authorization — the same pattern used by "Sign in with Google"-style buttons, which is not a coincidence; those buttons face exactly this same "must be embeddable by literally anyone" constraint and solved it the same way. A click that only *triggers* a top-level popup, rather than *completing* the sensitive action itself, can't be silently redirected by an invisible overlay in the same way, because the user is now interacting with a window the embedding page fundamentally can't control the framing/visibility of.

**This is a real UX and integration cost, and I'd present it as one rather than pretending it's free.** Moving the sensitive action out of the inline iframe click and into a popup/top-level flow means partners integrating the public widget get a meaningfully different (and slightly more friction-heavy) integration than a simple "click the button, done" experience — that's the actual price of supporting genuinely arbitrary embedding safely, and it's worth having that trade-off be an explicit, informed product decision rather than something engineering quietly absorbs by weakening the security requirement instead.

**I would not rely on frame-busting JavaScript as a defense at all, for either case.** The classic pattern (`if (window.top !== window.self) { window.top.location = window.self.location; }`) is trivially defeated by an attacker embedding the page inside an iframe with a `sandbox` attribute that omits `allow-top-navigation` — the sandboxed frame's script is simply prevented by the browser from navigating the top-level page, so the busting script silently fails to do anything, while the frame (and the clickjacking setup around it) stays exactly as the attacker arranged it. This is a well-known, long-superseded technique; the header-based (`frame-ancestors`) and structural (top-level-navigation-for-sensitive-actions) approaches above are the actual current best practice, and I'd flag frame-busting JS as a red flag if I found it already in the codebase as the *only* defense.

## The Fix

**Known-partner tier:**

```
Content-Security-Policy: frame-ancestors https://partner-a.example.com https://partner-b.example.com;
X-Frame-Options: SAMEORIGIN
```

Every new partner requires an explicit addition to this list, reviewed the same way any trust boundary expansion should be — this is a deliberate friction point, not overhead to streamline away.

**Public tier — structural redesign of the sensitive action:**

```tsx
function ConnectBankWidget() {
  function handleConnectClick() {
    // Does NOT perform the authorization inline. Opens a genuine top-level
    // popup the embedding page cannot invisibly overlay or intercept clicks on.
    const popup = window.open(
      'https://auth.ourplatform.com/connect',
      'connect-bank',
      'width=480,height=640'
    );

    // Listen for the popup to report completion via postMessage,
    // validating the origin explicitly.
    function onMessage(event: MessageEvent) {
      if (event.origin !== 'https://auth.ourplatform.com') return;
      if (event.data?.type === 'connect-complete') {
        popup?.close();
        window.removeEventListener('message', onMessage);
        // proceed with the connected-account UI state
      }
    }
    window.addEventListener('message', onMessage);
  }

  return <button onClick={handleConnectClick}>Connect your bank account</button>;
}
```

The click inside the (still arbitrarily embeddable, still clickjackable-in-principle) iframe now only ever *opens* a popup — it cannot, by itself, complete an authorization. Even if an attacker's overlay tricks a user into clicking this button when they meant to click something else, the worst outcome is an unwanted popup opening (annoying, visible, and immediately actionable by the user — they can just close it), not a silently completed bank-account authorization. The actual sensitive flow happens inside `auth.ourplatform.com`'s own popup, which — being a genuine top-level browsing context under our own control — can itself carry a strict `frame-ancestors 'none'` (it should never be framed at all) and its own full, undiluted set of protections.

> **Check yourself:** Explain why moving the sensitive action to a popup specifically defeats clickjacking, when the *trigger* button (the one an attacker's overlay is trying to deceive the user into clicking) is still sitting in an arbitrarily-embeddable iframe exactly as before.

## Gotchas

**Treating `X-Frame-Options` as sufficient once it's set to `SAMEORIGIN`, without checking whether multiple external partners actually need to embed the widget.** `SAMEORIGIN` blocks *all* cross-origin framing, including the vetted partners the product explicitly wants to support — a candidate who reaches for `SAMEORIGIN` without checking the actual embedding requirements will break the feature entirely, not just the attack.

**Relying on frame-busting JavaScript as a standalone defense.** As covered above, it's defeated by an attacker's `sandbox` attribute omitting `allow-top-navigation` — this is common enough real-world knowledge that citing it unprompted is a strong signal, and *not* knowing it (proposing frame-busting JS as "the fix") is a clear gap.

**Assuming HTTPS/a valid certificate has any bearing on clickjacking risk.** It doesn't — clickjacking is a UI-deception attack (visually or positionally hiding the real, legitimate, correctly-served-over-HTTPS iframe under attacker-controlled content), entirely orthogonal to transport security. A candidate who mentions "but we're on HTTPS" as a mitigating factor here is conflating two unrelated threat models.

**Confusing clickjacking defenses with CSRF defenses.** `SameSite` cookies and CSRF tokens protect against a different threat: a forged *request* sent without the user's knowledge or intent at all. Clickjacking is about tricking a user into knowingly clicking *something*, just not the thing they think they're clicking — the request that fires is often not forged in the CSRF sense (it carries real, correctly-scoped cookies, because the user really is on the real site's iframe), it's just triggered by a deceived click rather than an intended one. Proposing `SameSite=Strict` as a clickjacking fix misses that the attack doesn't rely on cross-site request forgery at all.

**Not asking who's actually allowed to embed the widget before proposing a fix.** This is the single fork in the road between "write a `frame-ancestors` allowlist" and "redesign the interaction to require top-level navigation" — proposing the allowlist fix without confirming there's a finite list to write, or proposing the heavier structural redesign when a simple allowlist would have sufficed, both suggest the question wasn't asked.

## Follow-up Questions

**Q (High): Why is `Content-Security-Policy: frame-ancestors` preferred over `X-Frame-Options` for the known-partner case specifically?**

Answer: `X-Frame-Options` predates the need many sites have to be embeddable by more than one external origin — its value space is `DENY`, `SAMEORIGIN`, or the non-standard `ALLOW-FROM <origin>`, which most modern browsers don't actually implement (and even where implemented historically, only supported a single origin, not a list). `frame-ancestors`, as a CSP directive, was designed with exactly this multi-origin case in mind and takes a space-separated list of permitted framing origins natively, is well-supported across all current browsers, and additionally composes with the rest of a site's CSP rather than being a separate, older header with its own quirks. For a widget that needs to be embeddable by several named, vetted partners, `frame-ancestors` is simply the only one of the two that can correctly express the requirement at all — `X-Frame-Options` would force a choice between `DENY` (breaks the feature) or `SAMEORIGIN` (still breaks it, since partners are cross-origin by definition).

The trap: proposing `X-Frame-Options: SAMEORIGIN` as if it solves the multi-partner case — it's the right instinct for "restrict framing" in general, but it structurally cannot support more than the site's own origin, which doesn't match a scenario with multiple external partners needing to embed the widget.

---

**Q (High): The widget also needs to support a "public" embed tier where literally any website can embed it. You can't write an allowlist for "any origin." How do you defend against clickjacking there?**

Answer: Header-based origin restriction is the wrong tool for this case by definition — there's no finite allowlist to write if the requirement is genuinely unrestricted embedding, and leaving `frame-ancestors` open (or absent) to satisfy that requirement reopens exactly the vulnerability under review. The fix has to change what a single in-iframe click can actually *do*: rather than the click completing the sensitive action (authorizing a bank connection) directly inside the arbitrarily-embeddable frame, the click should only open a genuine top-level popup or perform a top-level navigation to complete the sensitive flow — the same structural pattern established OAuth "Sign in with X" buttons use for exactly this reason. A deceived click on the embedded trigger button, at worst, opens an unwanted (but fully visible, user-dismissable) popup; it can no longer, by itself, complete an authorization the user never intended. This is a real integration/UX cost compared to a single inline click completing everything, and that cost is the actual price of supporting arbitrary embedding safely — not a limitation to work around, but the trade-off itself.

The trap: trying to find some header or meta-tag configuration that makes public, unrestricted embedding "safe" without changing the interaction model — no such header exists; if literally anyone can frame the widget, the fix has to be in what a click can accomplish, not in restricting who can frame it (since that's explicitly not restricted).

---

**Q (Medium): Why is frame-busting JavaScript (`if (top !== self) top.location = self.location`) not a reliable defense, even as a supplement to headers?**

Answer: An attacker constructing a clickjacking page controls exactly how they embed the target iframe, including its `sandbox` attribute. Embedding the target inside `<iframe sandbox="allow-scripts">` (deliberately omitting `allow-top-navigation`) allows the framed page's own JavaScript to run — so the busting script does execute — but the browser blocks precisely the one operation the script needs to escape the frame: navigating the top-level browsing context. The busting attempt silently fails, the clickjacking setup remains fully intact, and the defender gets no signal that anything went wrong. Because this bypass requires nothing more exotic than a standard, well-documented `sandbox` attribute value on the attacker's own page, frame-busting JS is considered broadly unreliable rather than merely "imperfect" — it's the textbook example given for why header-based (`frame-ancestors`) or structural (top-level-navigation-for-sensitive-actions) defenses replaced it as best practice.

The trap: treating frame-busting JS as "better than nothing" defense-in-depth — since the bypass is trivial and well-known, its presence can give a false sense of having addressed clickjacking when it hasn't meaningfully raised the bar for an attacker who knows this specific technique, which is not an exotic or advanced one.

---

**Q (Medium): Does serving the widget over HTTPS with a valid certificate provide any protection against clickjacking?**

Answer: No — the two are orthogonal. HTTPS protects the integrity and confidentiality of the widget's content *in transit* between server and browser (preventing tampering or eavesdropping by a network-level attacker), and confirms the browser is really talking to the legitimate origin. Clickjacking is entirely a client-side, post-delivery UI deception: the real, correctly-served, unmodified, HTTPS-delivered widget is rendered exactly as intended, just positioned invisibly or disguised underneath attacker-controlled content on a different page, so that a user's click lands on it without realizing it. The certificate and encryption have already done their job correctly by the time the browser renders the (legitimate) content — the attack happens entirely in how that legitimate content is visually composed on the page, which HTTPS has no mechanism to detect or prevent.

The trap: citing "we're on HTTPS" or "the padlock is green" as a mitigating factor when discussing clickjacking risk — a fairly common confusion, since both are broadly filed under "web security," but they defend against completely different threat models and neither substitutes for the other.

---

**Q (Low): Some sites use a "wait a moment before the button becomes clickable" or "require the mouse to actually move over the button first" trick as an additional clickjacking mitigation. Is this worth adding here, and what are its limits?**

Answer: These are heuristic, UX-layer mitigations aimed at a specific clickjacking variant — one where the attacker pre-positions the victim's cursor near the hidden button and relies on an instantaneous, un-considered click (often paired with a decoy "click here" prompt timed to the moment the hidden real button is under the cursor). A short render delay before the button becomes interactive, or requiring genuine pointer movement (checked via `mousemove` events) rather than accepting an instantaneous click, raises the bar against that specific timing-dependent variant, and costs very little to add. But they're not a structural fix — a patient attacker can design an overlay that accounts for a fixed delay, or simulate more convincing pointer movement, and these heuristics do nothing at all against a straightforward "genuinely invisible iframe positioned exactly under a large, honestly-labeled-looking fake button, clicked normally" setup, which is the baseline clickjacking case the header-based and structural fixes actually address. I'd consider these reasonable, cheap, additional layers on top of the real fixes discussed above, never as a substitute for them.

The trap: presenting timing/movement heuristics as if they were a real fix for clickjacking — they raise the cost of one specific attack variant marginally, but the actual defenses (origin restriction where a finite partner list exists, or structural top-level-navigation redesign where it doesn't) are what the interviewer is checking for; leading with the heuristic instead of the structural fix under-delivers on the question.

---

## Self-Assessment

- [ ] Can explain why `X-Frame-Options` can't support a multi-partner allowlist and why `frame-ancestors` can
- [ ] Can articulate why header-based origin restriction is structurally unusable for "anyone can embed this" and what replaces it
- [ ] Can describe the top-level-popup redesign pattern and why it defeats clickjacking even while the trigger stays embeddable
- [ ] Can explain precisely why frame-busting JS is defeated by the `sandbox` attribute
- [ ] Can distinguish clickjacking's threat model from CSRF's, and from HTTPS's guarantees
- [ ] Can name the real UX/integration cost the structural fix introduces, not just its security benefit

---
*Next: Dependency With a Known CVE in Production — moves from "a UI-level attack surface" to "a supply-chain risk already sitting in `node_modules`," testing triage judgment under a very different kind of time pressure.*
