# Auth Token Storage — a Breach Scenario

## Quick Reference

| Storage location | Exposed to XSS? | Exposed to CSRF? | Survives page reload? | Notes |
|---|---|---|---|---|
| `localStorage` | Yes — any script on the page can read it | No | Yes | Convenient, but any XSS anywhere on the origin is a full token theft |
| `sessionStorage` | Yes — same as localStorage | No | Only within the tab | Slightly narrower blast radius (per-tab), still fully XSS-readable |
| Cookie, no flags | Yes (readable via `document.cookie` unless `HttpOnly`) | Yes | Yes | Worst of both worlds if flags are omitted |
| Cookie, `HttpOnly` + `Secure` + `SameSite=Strict/Lax` | Not directly readable by script | Substantially reduced, not eliminated | Yes | Right home for a long-lived **refresh** token |
| In-memory (JS variable, never persisted) | Only if attacker's script runs *while* it's in memory | No | No — lost on reload | Right home for a short-lived **access** token |

## The Scenario

"We just got an incident report: an XSS bug in a third-party analytics widget we embed let an attacker read `localStorage` on our site for about six hours before we caught it, and our auth tokens live in `localStorage`. We need to (1) respond to this breach right now, and (2) redesign token storage so this class of breach is fundamentally less damaging next time — not just patch this one XSS hole. Walk me through both."

This scenario deliberately has two halves — incident response for *this* breach, and an architectural redesign for *next* time — because conflating them (jumping straight to "let's move everything to cookies" without first handling the live exposure) is a common and telling mistake under interview pressure.

## Clarifying Questions

- **What kind of token is in `localStorage` — a short-lived access token, a long-lived refresh token, or both?** This changes the urgency and the shape of the fix dramatically: a leaked access token with a 15-minute expiry is a bounded problem that resolves itself shortly after the exposure window closes; a leaked refresh token with a 30-day expiry is a standing compromise that persists until explicitly revoked.
- **Is the token a stateless JWT or an opaque token the server can look up and invalidate?** A JWT's validity is normally checked by verifying its signature and expiry alone, with no server-side lookup — which means there's often no way to invalidate one early short of maintaining a revocation list or rotating the signing key (which invalidates *every* outstanding token, not just the compromised ones). This single fact determines whether "revoke the stolen token" is a five-minute database update or a much bigger operation.
- **Does the frontend need to call any third-party or cross-origin APIs with this token, or only our own first-party backend?** If the token is sent to third-party origins, cookie-based storage becomes awkward or impossible (cookies are same-origin-scoped by default and third parties generally shouldn't receive them anyway) — this constrains which storage redesign options are actually viable.
- **Is there a mobile app sharing the same backend and token scheme?** A redesign centered on browser cookies doesn't translate to a native app, which has its own secure-storage primitives (Keychain/Keystore) — I'd want to know if the fix needs to be browser-specific or needs a parallel native-side equivalent.
- **Do we have request logs covering the six-hour exposure window?** This determines whether "assume the worst" is the only option or whether we can actually narrow down which accounts, if any, show suspicious activity (logins from new locations/devices, unusual API call patterns) during that window.

## Approach & Trade-offs

**Two clocks, two workstreams.** The breach response (what do we do about the tokens that were *already* exposed) and the architectural fix (how do we store tokens going forward) are both necessary but shouldn't block each other. I'd drive the breach response first since it's time-critical and mostly involves configuration/operational changes (revoke tokens, patch the immediate XSS hole, force re-auth) rather than a redesign, and treat the storage redesign as the follow-up work item that prevents a repeat, not something that needs to ship in the same hour.

**`localStorage` vs. cookies is not "one is secure, one isn't" — it's a trade-off between two different attack surfaces.** `localStorage` is fully exposed to any XSS on the page — no flags exist to protect it, since by design any JavaScript running on the origin can read and write it. An `HttpOnly` cookie can't be read by JavaScript at all, which closes off the *direct* token-theft vector this exact breach exploited — but cookies are automatically attached by the browser to matching requests, which reopens CSRF (a malicious site tricking the browser into making an authenticated request the user didn't intend) as an attack surface that `localStorage`-based tokens don't have, since those require an attacker's script to explicitly read and attach the token. Neither option is strictly safer in general — the right choice depends on which attack surface is easier to close for a given token type and application shape.

**Split the token by lifetime, and store each half appropriately — this is the actual redesign, not just "pick localStorage or cookies."** The pattern I'd land on: a short-lived **access token** (minutes, not days) kept purely in memory — a module-level variable or a value in application state, never written to any persistent storage — and a long-lived **refresh token** stored in an `HttpOnly`, `Secure`, `SameSite=Lax` (or `Strict`, if the flow allows it) cookie, used only to hit a dedicated refresh endpoint. This means: even if the exact same XSS bug recurs, injected script can no longer read the refresh token at all (`HttpOnly` blocks it), and the access token it *could* theoretically grab from memory during the attack window is only ever useful for the few minutes before it naturally expires — turning "attacker has a working token for six hours" into "attacker has a working token for, at most, the token's own short TTL, and only while their injected script is actively running." That's a fundamentally smaller blast radius from the same underlying XSS bug, which is precisely the "fix this class of problem" ask.

**Trade-off this introduces: losing the access token on every page reload.** Keeping the access token purely in memory means a full page refresh loses it — the app has to silently call the refresh endpoint (which relies on the `HttpOnly` cookie the browser sends automatically) on load to get a new access token before the user can do anything authenticated. This adds a brief "am I logged in yet?" loading state on every hard reload that didn't exist when the token was simply sitting in `localStorage` ready to use instantly — a real UX cost, worth naming explicitly rather than presenting the redesign as free.

**If the team's ambitions go further: a Backend-for-Frontend (BFF) pattern removes tokens from the browser entirely.** In this model, the browser only ever holds an `HttpOnly` session cookie scoped to a same-origin BFF layer; the actual API access tokens are held server-side and never sent to the browser at all. This is the strongest posture against exactly this breach — there's no token in the browser for any XSS to steal, full stop — but it's a meaningfully bigger architectural change (introducing or extending a backend layer purely for this) than the split-token approach, so I'd frame it as the answer to "if we were designing this from scratch" rather than something to retrofit under incident pressure this week.

## The Redesign

```ts
// authClient.ts — access token lives ONLY here, never in localStorage/sessionStorage
let accessToken: string | null = null;

export function getAccessToken() {
  return accessToken;
}

export async function refreshAccessToken(): Promise<boolean> {
  // Cookie is sent automatically (HttpOnly + SameSite); no token is read or attached manually here
  const res = await fetch('/api/auth/refresh', { method: 'POST', credentials: 'include' });
  if (!res.ok) {
    accessToken = null;
    return false;
  }
  const { accessToken: newToken } = await res.json();
  accessToken = newToken; // held in memory only
  return true;
}

export async function apiFetch(input: RequestInfo, init: RequestInit = {}) {
  let res = await fetch(input, {
    ...init,
    headers: { ...init.headers, Authorization: `Bearer ${accessToken}` },
  });

  if (res.status === 401) {
    // Access token expired mid-session — try one silent refresh, then retry once
    const refreshed = await refreshAccessToken();
    if (!refreshed) throw new Error('Session expired');
    res = await fetch(input, {
      ...init,
      headers: { ...init.headers, Authorization: `Bearer ${accessToken}` },
    });
  }
  return res;
}
```

```
Set-Cookie: refreshToken=<opaque-token>; HttpOnly; Secure; SameSite=Lax; Path=/api/auth/refresh; Max-Age=2592000
```

Key properties of this design: the refresh token cookie is scoped to `Path=/api/auth/refresh` specifically (not `/`), so it isn't even sent along with every other API request — narrowing exactly which endpoint could leak it via, say, a CSRF-adjacent misconfiguration elsewhere. The access token never touches `localStorage`, `sessionStorage`, or a cookie — an XSS payload running *right now* could still call `getAccessToken()` and grab the current value (in-memory storage isn't immune to a script executing *during* the compromise), but it can't extract the long-lived refresh token, and it gets nothing at all once the access token's short TTL elapses — the persistent, reusable-later value is exactly what's protected.

> **Check yourself:** Explain out loud why this design doesn't eliminate the value of the XSS bug entirely — what can an attacker still do with injected script running live on the page, even with tokens split this way?

## Breach Response (This Incident, Right Now)

- **Revoke or rotate every outstanding refresh token**, forcing full re-authentication for all users — even without confirmed exploitation beyond the report, a six-hour exposure window on a token store is treated as "assume compromised, prove otherwise" rather than "assume safe until proven compromised."
- **If tokens are stateless JWTs with no revocation list**, the only clean options are rotating the signing key (invalidates everyone, forces global re-auth — blunt but certain) or shortening remaining TTL isn't retroactively possible, which is itself the argument for why *opaque, server-checkable* refresh tokens (not JWTs) are usually the better choice for anything long-lived enough to need revocation.
- **Patch the specific XSS hole in the third-party widget** — likely by sandboxing it (an `<iframe>` with a restrictive `sandbox` attribute and no access to the parent page's DOM/storage) or removing it until the vendor ships a fix, since the storage redesign doesn't matter if the injection vector generating the theft opportunity is still live.
- **Audit logs for the exposure window** for anomalous activity (new-device logins, unusual API usage) tied to the timeframe, to move from "assume all sessions compromised" toward "these specific accounts show signs of actual misuse" for any user-facing disclosure decision.
- **Decide on user communication with security/legal**, informed by what the log audit turns up — not something to decide unilaterally as the engineer driving the technical fix.

## Gotchas

**Treating "move tokens to cookies" as a complete, self-contained fix.** It closes the direct-read XSS vector but reopens CSRF if `SameSite` and endpoint design aren't handled deliberately — a redesign proposed without mentioning CSRF mitigation is missing half the trade-off.

**Assuming `HttpOnly` cookies make XSS harmless.** As covered in the [Stored XSS](01-stored-xss-found-in-production.md) scenario, injected script running live on the page can still make authenticated same-origin requests using cookies the browser attaches automatically — `HttpOnly` stops *token theft for later reuse*, not *misuse during the attack itself*.

**Forgetting that a JWT's "expiry" isn't the same as "revocability."** A stolen JWT with 29 days left on its TTL is valid for 29 days unless the system has a separate revocation mechanism — proposing "just check the token's expiry" as the safety net for a breach misses that the whole point of a breach response is invalidating tokens *before* their natural expiry.

**Proposing the in-memory-access-token design without acknowledging the reload/silent-refresh UX cost.** Every non-trivial security trade-off has a cost; presenting this one as strictly free reads as not having thought through the full picture.

**Conflating this scenario's browser-based storage fix with what a native mobile app should do.** A mobile app doesn't have `localStorage`/cookies in the same sense — the equivalent redesign for a companion app would center on Keychain (iOS) / Keystore (Android), which is a different (though analogous) implementation with its own constraints.

## Follow-up Questions

**Q (High): Why does moving the token from `localStorage` to an `HttpOnly` cookie not fully solve the security problem — what's the new risk it introduces?**

Answer: `HttpOnly` cookies close the direct-read vector (injected JavaScript can no longer call something equivalent to `document.cookie` and get the token value out), which is exactly what this breach exploited. But cookies are attached by the browser automatically to any matching request, including ones initiated by a malicious third-party site the victim happens to visit while still logged in — that's CSRF: the attacker's page can't read the cookie, but it can still cause the browser to send an authenticated request carrying it. Mitigating that requires `SameSite=Lax` or `Strict` (which stops the cookie being sent on most cross-site requests) plus, for anything `SameSite` doesn't fully cover, a CSRF token pattern (a per-session token included in a custom header or hidden form field that a cross-site attacker can't read or forge, verified server-side on state-changing requests). Moving to cookies without addressing this is trading one solved problem for one newly reopened one.

The trap: presenting "just use cookies" as a complete fix — a senior answer names the CSRF trade-off unprompted, since it's the direct consequence of the exact property (automatic attachment) that makes cookies attractive here.

---

**Q (High): The refresh token is a JWT and we have no revocation list. How do you actually invalidate the specific tokens that may have been stolen during the six-hour window, right now?**

Answer: If there's genuinely no server-side state tracking valid tokens, the only ways to force early invalidation are: (1) rotate the JWT signing key/secret, which invalidates literally every outstanding token — including tokens belonging to users who were never at risk — forcing a full re-auth across the entire user base, a blunt but immediate and certain fix; or (2) if the JWT includes a version/generation claim tied to the user record, bump that value for all users and have the verification step reject tokens with a stale version — narrower than a full key rotation but requires that claim to already exist in the token schema, which it may not. Absent either, there is no partial, targeted revocation available for a pure stateless JWT — which is exactly the argument for why long-lived, security-sensitive tokens (refresh tokens especially) are usually better implemented as opaque, server-side-checkable tokens rather than JWTs: the ability to invalidate one specific token on demand is worth the extra database lookup on every refresh call.

The trap: suggesting "just wait for them to expire" as sufficient for a breach response — that's acceptable for routine token hygiene, not for a confirmed exposure window where the standing assumption should be "revoke now, don't wait out the clock."

---

**Q (High): Would the access-token-in-memory design have prevented this specific breach from being as damaging?**

Answer: Meaningfully, yes, though not perfectly. In this design, the leaked six-hour exposure would have only ever exposed whatever access token happened to be live in memory at the moments the attacker's script executed — each such token expiring in minutes, not persisting for reuse after the fact — rather than a `localStorage`-resident token/refresh-token pair valid for the token's full lifetime (which could be weeks) sitting there ready to be read at any point during the six-hour window and reused indefinitely afterward. The refresh token, being `HttpOnly`, would never have been readable by the injected script at all under this design, meaning the attacker's window of usable access would have been bounded by the access token's short TTL rather than the refresh token's long one. It's not a complete prevention — a sufficiently active attacker script running live during the exposure window could still make authenticated requests using whatever access token was current at that instant — but it converts "unbounded token theft for the refresh token's full lifetime" into "bounded misuse only during the live attack window," which is the actual goal of "make this class of breach less damaging."

The trap: claiming this design would have prevented the breach entirely — it narrows blast radius significantly, it does not make XSS harmless, and overstating that distinction misses the point of defense-in-depth reasoning the interviewer is checking for.

---

**Q (Medium): What is the Backend-for-Frontend (BFF) pattern, and why is it the "gold standard" answer here even though you wouldn't recommend it as this week's incident fix?**

Answer: In a BFF architecture, the browser never holds any access or refresh token for the actual downstream APIs at all — it holds only an `HttpOnly` session cookie scoped to a same-origin backend layer (the BFF) that the frontend talks to. The BFF itself holds the real tokens server-side (in a database or server-side session store) and attaches them to outbound requests to the actual APIs on the browser's behalf. From the browser's perspective, there is simply no bearer token anywhere for an XSS payload to steal — the worst an XSS bug can do is make requests as the logged-in user through the BFF while the session is live, the same residual risk any cookie-based scheme has, but with no long-lived credential material exposed to the browser at all. It's the strongest posture against token theft specifically, but it's a bigger lift — introducing or extending an actual backend proxy layer — which is why it's the right answer for "how would you design this system from scratch" rather than "what do we ship this week" during an active incident with existing infrastructure.

The trap: recommending a full BFF migration as the immediate incident response — it's the correct long-term architectural answer, but proposing it as this week's fix ignores the time-criticality established in the scenario and conflates the two clocks the approach section explicitly separates.

---

**Q (Medium): Why is a short access-token TTL (e.g., 15 minutes) combined with silent refresh better than just using one long-lived token and calling it done?**

Answer: A single long-lived token means any theft of it — via XSS, a logging mistake, a misconfigured error report that included headers, a compromised browser extension — is valid and reusable for its entire lifetime, which is usually the actual damage multiplier in a breach post-mortem ("the token was valid for 30 more days"). Splitting into a short-lived access token plus a longer-lived, better-protected (`HttpOnly`) refresh token means the token that's actually exposed to in-browser risk (XSS, extensions, etc., since it has to be attached to requests somehow) has a short natural shelf life, while the token that's valid for a long time is kept out of reach of the riskiest exposure vector. The cost is added complexity — a refresh flow, handling concurrent requests during a refresh, handling refresh failure — which is a real engineering cost worth naming, not a free improvement.

The trap: proposing "just make the single token's TTL very short" without introducing a refresh mechanism — that just makes users re-enter their password every 15 minutes, trading a security improvement for an unusable product; the point of the split is getting the short TTL's security benefit *without* that UX cost, via the refresh flow.

---

**Q (Low): How would this redesign differ for a native mobile app sharing the same backend?**

Answer: The web-specific mechanics (cookies, `SameSite`, CSRF) don't apply to a native app the same way — there's no browser enforcing same-origin/`SameSite` semantics for a mobile HTTP client. The equivalent goal (don't let a long-lived, highly damaging credential sit in easily-exfiltrated storage) is instead achieved via the platform's secure credential storage: iOS Keychain and Android Keystore, both of which are OS-level, hardware-backed-where-available stores specifically designed for exactly this (encrypted at rest, tied to app identity, not readable by other apps). The access/refresh split concept still applies conceptually — short-lived access token in memory, longer-lived refresh token in Keychain/Keystore rather than a cookie — but the storage primitive changes to match the platform's actual threat model, since "someone reads app-local storage via a debugger or a jailbroken/rooted device" is the mobile analogue of "someone reads `localStorage` via XSS."

The trap: assuming the web redesign (cookies specifically) has a direct 1:1 equivalent on mobile — cookies are a browser-and-HTTP-client concept tied to `SameSite`/CSRF semantics that don't carry over; the right mobile answer names the platform-specific secure storage primitives instead.

---

## Self-Assessment

- [ ] Can separate "respond to this breach" from "redesign for next time" as two distinct workstreams, unprompted
- [ ] Can explain the localStorage-vs-cookie trade-off as "different attack surfaces," not "one is secure, one isn't"
- [ ] Can describe the access-token-in-memory + refresh-token-in-HttpOnly-cookie split and its UX cost (reload → silent refresh)
- [ ] Can explain why a stateless JWT is hard to revoke early, and what to do about it during an active breach
- [ ] Can name CSRF as the trade-off introduced by moving to cookies, and how SameSite/CSRF tokens address it
- [ ] Can describe the BFF pattern and correctly frame it as the long-term answer, not the incident-week fix

---
*Next: Open Redirect Found in Code Review — a smaller, single-bug scenario, but one that tests whether the candidate can spot a vulnerability from a diff alone, before it ever reaches production.*
