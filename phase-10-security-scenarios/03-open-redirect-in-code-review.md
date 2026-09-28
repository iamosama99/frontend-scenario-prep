# Open Redirect Found in Code Review

## Quick Reference

| Bypass attempt | Why a naive check misses it | Correct defense |
|---|---|---|
| `?returnTo=//evil.com` | `startsWith('/')` is true — this is a *protocol-relative* URL, and browsers resolve it against the current protocol as `https://evil.com` | Explicitly reject any value starting with `//` in addition to requiring a single leading `/` |
| `?returnTo=/\evil.com` | Some browsers normalize a leading backslash to a slash, turning this into a protocol-relative URL too | Parse with the `URL` API against a known base and compare `origin`, don't string-match |
| `?returnTo=https://trusted.com@evil.com` | Naive `includes('trusted.com')` matches — but the actual host, per URL spec, is `evil.com` (everything before `@` is userinfo) | Never validate via substring match; parse the URL and check `.hostname` explicitly |
| `?returnTo=https://evil.com` | `indexOf('http') === -1` catches this one, but only because it forgot the case above | Use an allowlist of exact permitted origins, or restrict to same-origin relative paths only |

## The Scenario

"Here's a PR from another engineer on the team — a post-login redirect feature: after signing in, the user gets sent back to whatever page they were trying to reach, using a `returnTo` query parameter. Review it. What do you flag, and what would you ask the author to change before approving?"

The diff being reviewed:

```tsx
function LoginPage() {
  const [params] = useSearchParams();
  const returnTo = params.get('returnTo') ?? '/dashboard';

  async function handleLogin(credentials: Credentials) {
    await login(credentials);
    // Author's reasoning, per the PR description: "returnTo always starts with '/'
    // since it comes from our own app's links, so this is safe."
    window.location.href = returnTo;
  }

  return <LoginForm onSubmit={handleLogin} />;
}
```

This scenario is deliberately framed as *finding* the bug in someone else's code during review, rather than debugging a live report — it tests whether the candidate can spot the vulnerability class from a diff alone, before it ever ships, which is a different (and in some ways harder) skill than diagnosing a known-broken system.

## Clarifying Questions

- **Is `returnTo` ever expected to point somewhere other than same-origin — e.g., to a trusted partner site after an SSO-style login?** If the legitimate use case is strictly "come back to a page within our own app," the fix is a strict same-origin/relative-path check. If there's a genuine need to redirect to specific external partners, the fix has to be an explicit allowlist of those exact origins instead — a meaningfully different (and more work to maintain) shape of fix.
- **Is this redirect happening client-side (`window.location.href`, as shown) or does the server also issue a redirect (a 302) using a similar parameter?** Both are exploitable, but a server-side open redirect is often more valuable to an attacker — it can make a phishing link look like it starts entirely on the trusted domain (`https://real-app.com/login?returnTo=evil.com` redirecting via a server 302), which is more convincing than a client-side redirect that at least briefly shows the trusted domain in the address bar before navigating away.
- **Does this app have an OAuth or SSO flow anywhere that also takes a redirect-style parameter (`redirect_uri`, `next`, `continue`)?** Open redirect bugs are disproportionately dangerous specifically when they sit near an OAuth flow, because a redirect target attacker control combined with an authorization flow can leak authorization codes or tokens to the attacker's server — I'd want to know if this `returnTo` pattern is reused anywhere near auth-token-bearing redirects, since the severity is much higher there than for a plain post-login landing page.
- **Has this exact `returnTo` ?? `/dashboard'` pattern been copy-pasted anywhere else in the codebase?** A code-review finding on one instance is a good moment to check for the same anti-pattern elsewhere, since "the author reasoned it was safe because it starts with `/`" is a mistake likely to recur wherever the same shortcut gets reused.

## Approach & Trade-offs

**I'd flag this in review before it merges, not wait for it to reach production** — this is exactly the kind of bug where the fix is a few lines and the review is the cheapest point in the entire lifecycle to catch it, versus finding it via a bug bounty report or, worse, an actual phishing campaign later.

**The core problem with the author's reasoning ("`returnTo` always starts with `/`, so it's safe") is that it conflates "what our own app's links look like" with "what an attacker can put in this query parameter."** The parameter's value is entirely attacker-controlled the moment it's read from the URL — nothing stops someone from crafting `https://real-app.com/login?returnTo=//evil.com` and sending that link to a victim. The victim sees a link starting with the real, trusted domain (which is exactly why this is valuable for phishing — it survives a cautious user's "hover over the link and check the domain" habit), logs in for real, and is then silently redirected to `evil.com`, which can now show a convincing fake "session expired, please log in again" page to harvest credentials with the victim's guard already down.

**Trade-off: strict same-origin/relative-path validation vs. an explicit external-domain allowlist.** If there's no legitimate need to ever redirect off-site, I'd push for the strictest option — validate that `returnTo` is a same-origin relative path and nothing else, full stop, which closes off the entire vulnerability class rather than trying to enumerate every malicious external URL shape. If there's a genuine product requirement to sometimes redirect to a specific trusted partner (common in SSO handoff flows), that requires an explicit allowlist of exact permitted origins, checked via proper URL parsing — a real but justified increase in complexity, and one I'd want the PR to call out explicitly rather than quietly add later once "just this one partner domain" creeps into "several partner domains, half of them added without review."

**I'd insist on parsing with the `URL` API rather than string methods (`startsWith`, `includes`, `indexOf`) for any of this validation.** String matching against a URL is exactly the class of check that's bypassable via protocol-relative URLs, userinfo tricks (`https://trusted.com@evil.com`), and similar parser-confusion techniques — the fix isn't "add more string checks," it's "stop reasoning about URLs as strings" and let a spec-compliant parser tell you the actual resolved hostname/origin.

## The Fix

```tsx
function getSafeReturnTo(raw: string | null, fallback = '/dashboard'): string {
  if (!raw) return fallback;

  // Reject protocol-relative and absolute URLs outright: must be a same-origin path.
  if (!raw.startsWith('/') || raw.startsWith('//') || raw.startsWith('/\\')) {
    return fallback;
  }

  try {
    // Resolve against a known origin and confirm it didn't escape same-origin.
    const resolved = new URL(raw, window.location.origin);
    if (resolved.origin !== window.location.origin) {
      return fallback;
    }
    return resolved.pathname + resolved.search + resolved.hash;
  } catch {
    return fallback;
  }
}

function LoginPage() {
  const [params] = useSearchParams();
  const returnTo = getSafeReturnTo(params.get('returnTo'));

  async function handleLogin(credentials: Credentials) {
    await login(credentials);
    window.location.href = returnTo; // now guaranteed same-origin or the fallback
  }

  return <LoginForm onSubmit={handleLogin} />;
}
```

If external-partner redirects are a genuine requirement, the allowlist variant replaces the same-origin check:

```ts
const ALLOWED_REDIRECT_ORIGINS = new Set([
  'https://partner-a.example.com',
  'https://partner-b.example.com',
]);

function getSafeReturnTo(raw: string | null, fallback = '/dashboard'): string {
  if (!raw) return fallback;
  try {
    const resolved = new URL(raw, window.location.origin);
    if (resolved.origin === window.location.origin || ALLOWED_REDIRECT_ORIGINS.has(resolved.origin)) {
      return resolved.toString();
    }
  } catch {
    /* fall through to fallback */
  }
  return fallback;
}
```

Both versions share the same underlying discipline: parse the value into a real `URL`, inspect its `.origin`/`.hostname` explicitly, and compare against a known-good set — never infer safety from what the raw string merely starts with or contains.

> **Check yourself:** Walk through, character by character, why `new URL('//evil.com', 'https://real-app.com').origin` evaluates to `'https://evil.com'` rather than something involving `real-app.com` — this is the exact mechanism the `raw.startsWith('//')` check exists to catch before the value ever reaches `new URL`.

## Gotchas

**`startsWith('/')` alone is the single most common version of this bug, and it looks correct at a glance.** `//evil.com` genuinely does start with `/` — it's a protocol-relative URL, and browsers resolve `//host/path` against the *current page's protocol*, producing a fully external URL. A reviewer who doesn't specifically know to check for the double-slash case will often approve exactly this line.

**Blocklisting known-bad prefixes (`http://`, `https://`) instead of allowlisting known-good shapes.** This is the denylist-vs-allowlist trap showing up again (see the [Stored XSS](01-stored-xss-found-in-production.md) scenario for the same principle applied to HTML): there's always another way to express an absolute or protocol-relative URL that a specific blocklist entry doesn't anticompate — the `//evil.com`, backslash-normalization, and userinfo (`@`) tricks in the Quick Reference table are three separate ways to defeat a naive `indexOf('http')` check, and there's no guarantee that list is exhaustive.

**Not connecting this to OAuth `redirect_uri` validation when it's relevant.** If the app has any OAuth-adjacent flow, an open redirect bug elsewhere in the app can sometimes be chained with it — e.g., if the OAuth provider's `redirect_uri` validation is itself loose (matches by prefix rather than exact value) and permits sending the user through the app's *own* vulnerable open-redirect endpoint, the combination can leak authorization codes or tokens to an attacker even when the OAuth provider's redirect validation looks correct in isolation. A reviewer who only reasons about this one PR's blast radius in isolation, without asking whether it interacts with auth flows elsewhere, misses that broader risk.

**Approving the PR because "it's just a redirect, not data exposure."** Open redirect is routinely underrated in review because nothing appears to leak directly — no data is displayed, no script executes — but its entire value to an attacker is *social engineering leverage*: a phishing link that starts with a real, trusted domain is dramatically more effective than one that doesn't, and that's a real, exploitable property even though the bug itself looks inert in isolation.

## Follow-up Questions

**Q (High): This "just redirects" — no data is exposed and no script executes. Why does that make it a real vulnerability worth blocking a PR over?**

Answer: The danger isn't in the redirect mechanism itself, it's in what the redirect enables socially: a link like `https://real-app.com/login?returnTo=//evil.com` starts with the genuine, trusted domain — the part users are taught to check before clicking, and the part shown in email clients, chat previews, and a quick hover-to-check. A victim who does exactly the right due diligence (checks the domain, sees it's legitimate) still ends up on `evil.com` after a real, successful login — with their guard already down from having just authenticated somewhere they trust. That's dramatically more convincing than a raw phishing link to an unknown domain, and is the actual reason open redirect bugs are treated as security-relevant even though, taken in isolation, "the browser navigated somewhere" doesn't sound alarming.

The trap: dismissing the finding as low-severity because no data was technically exposed by the redirect itself — the exploit value is entirely in the phishing/social-engineering leverage it hands an attacker, not in any direct data leak, and a reviewer who only checks for direct data exposure will wave this through.

---

**Q (High): Why does `raw.startsWith('/') && !raw.startsWith('//')` still not fully solve this — what else needs checking?**

Answer: That pair of checks does correctly close off the plain protocol-relative case, but it's still string-based reasoning about something that should be resolved by an actual URL parser. Depending on the browser and how the value later gets used, a value like `/\evil.com` (backslash instead of the second forward slash) can be normalized by the browser into a protocol-relative URL during navigation even though it doesn't literally start with `//` as a string — some browsers treat backslashes interchangeably with forward slashes in URLs for legacy-compatibility reasons. The robust fix is to resolve the value with `new URL(raw, window.location.origin)` and then compare `.origin` against the known-good origin explicitly, letting the platform's own spec-compliant URL-parsing logic (which already knows about these normalization quirks) make the safety determination, rather than trying to enumerate every string pattern that could be coerced into an external URL by hand.

The trap: treating a fixed set of string checks as a complete fix — every string-pattern-based defense against URL parsing tricks is playing catch-up against a parser that has more edge cases than a manually maintained blocklist will ever cover; the fix is to stop parsing manually, not to add one more check to the list.

---

**Q (High): How would you implement the allowlist version if the product genuinely needs to support redirecting to a couple of trusted partner domains?**

Answer: Parse the candidate URL with `new URL(raw, window.location.origin)`, then check `resolved.origin` — not a substring of the raw string — against an explicit `Set` of exact permitted origins (scheme + host + port), defined once as an allowlist. I'd insist the comparison target the parsed `.origin`, since that's what actually determines where the browser navigates, rather than checking `.hostname` alone (which misses the scheme — an `http://` variant of an otherwise-allowed `https://` origin is a downgrade worth catching) or, worse, doing a substring/`includes()` check against the raw string (defeated by the userinfo trick: `https://trusted-partner.com@evil.com` contains the substring `trusted-partner.com` but resolves to host `evil.com`). I'd also want this allowlist reviewed whenever it changes, since every entry added is a domain this app is explicitly willing to send authenticated users to post-login.

The trap: implementing the allowlist check via `raw.includes(allowedDomain)` or `raw.startsWith(allowedDomain)` — both are defeated by the userinfo-in-URL trick and similar tricks; the check has to operate on the *parsed* origin, never the raw string.

---

**Q (Medium): Does it matter whether this redirect happens client-side (`window.location.href`) versus the server issuing an HTTP 302 based on a similar parameter?**

Answer: Both are exploitable and both deserve the same fix in principle, but a server-issued 302 is often the more valuable version for an attacker to find, because it can make the entire redirect chain look like it happened on the trusted server without the browser ever rendering an intermediate trusted page — a crafted link to `https://real-app.com/some-redirect-endpoint?returnTo=evil.com` can 302 straight to the attacker's domain without the victim ever seeing `real-app.com`'s UI at all, which is a slightly different but still effective abuse of the domain's reputation (works even in previews/crawlers that follow server redirects but don't execute JavaScript, which the client-side `window.location.href` version wouldn't affect). Practically, if a codebase has both a client-side pattern like this one and any server-side redirect endpoints taking a similar parameter, I'd flag both for the same fix, since finding one is a strong signal to specifically go looking for the other.

The trap: treating "client-side only, so lower severity" as a reason to deprioritize — the phishing value described above works regardless of which layer issues the redirect, and server-side redirects have the additional property of working against tools that don't execute JavaScript.

---

**Q (Medium): How does an open redirect become dangerous specifically in combination with an OAuth authorization code flow?**

Answer: OAuth flows rely on the authorization server redirecting the user's browser back to a `redirect_uri` registered and validated ahead of time, carrying a sensitive value (an authorization code, or in the older implicit flow, an access token directly) in the URL. If a legitimate, registered `redirect_uri` on the app's own domain itself contains an open-redirect vulnerability — or if the OAuth provider's own `redirect_uri` matching is loose (e.g., prefix-matching instead of exact-match, allowing `https://real-app.com/anything` to pass validation) — an attacker can craft an authorization request whose `redirect_uri` points at the app's vulnerable open-redirect endpoint, which then forwards the sensitive authorization code or token onward to an attacker-controlled domain, all while the OAuth provider's own redirect validation technically "passed" because the URI it checked was, in fact, on the legitimate app's domain. This is a well-known real-world chaining pattern and is exactly why open redirect findings are worth escalating with extra urgency in any codebase that also implements OAuth, rather than treating them as isolated, contained bugs.

The trap: evaluating an open redirect bug purely on its own, immediate blast radius (phishing) without considering whether it sits near, or could be chained with, an OAuth flow elsewhere in the same application — the combined risk is materially higher than either piece in isolation.

---

**Q (Low): The userinfo trick — `https://trusted.com@evil.com` — resolves to host `evil.com`. Why, syntactically?**

Answer: Per the URL specification, the general form of a URL's authority component is `userinfo@host`, where everything before an `@` (if present) is credentials information (historically used for embedding a username/password directly in a URL, e.g., `ftp://user:pass@host`) and everything after it, up to the next `/`, `?`, `#`, or end of string, is the actual host. So `https://trusted.com@evil.com` parses as: scheme `https`, userinfo `trusted.com` (browsers ignore or warn about this rather than using it as literal credentials, but it's still syntactically valid userinfo), and host `evil.com` — the string *contains* `trusted.com` and even visually leads with it, but it is not, and was never, the host the browser will navigate to. This is precisely why any validation approach based on substring matching or `.includes()`/`.startsWith()` against the raw string is unsound — the human-readable order of characters in a URL string doesn't correspond to which part is authoritative for navigation, and only a real URL parser correctly separates them.

The trap: assuming any check that looks for the trusted domain's name "somewhere in the string, near the front" is good enough — the userinfo trick specifically exploits that assumption, placing the trusted-looking string exactly where a human (or a naive regex) would expect the host to be.

---

## Self-Assessment

- [ ] Can spot the `startsWith('/')`-only check as insufficient in a code review, without being told what to look for
- [ ] Can explain the protocol-relative URL (`//evil.com`) bypass mechanism precisely
- [ ] Can explain the userinfo (`@`) trick and why substring matching is fundamentally unsound for URL validation
- [ ] Can write the `URL`-API-based same-origin check and the allowlist variant from memory
- [ ] Can articulate why open redirect matters despite "not exposing any data" directly
- [ ] Can explain how this chains with OAuth `redirect_uri` handling to become more severe

---
*Next: Third-party Script Breaks Your CSP — turns from "catching a vulnerability" to "diagnosing a defense mechanism that's now in the way," a different kind of debugging under a security constraint.*
