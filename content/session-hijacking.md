---
title: "Session Hijacking"
description: "Predictable session identifiers and how they enable account takeover."
keywords:
  - Session Hijacking
  - CWE-330
  - CWE-340
  - OWASP A07:2025
  - Session Management
exports:
  - format: pdf
    template: plain_latex
    output: session-hijacking.pdf
---

---

## Introduction

This case study reproduces a session hijacking vulnerability against WebGoat `v2025.3` by identifying weaknesses in the identifier generation logic and exploiting the keyspace collapse that follows from pairing a global counter with a wall-clock timestamp. The objective is to demonstrate the vulnerability and the mitigations in a controlled environment and shine a light on the importance of modern session management practices in the context of web application security.

Session hijacking is a type of attack where an attacker gains access to a user's session by hijacking the session identifier [@owasp-top10-2025-a07; @owasp-cs-session-management]. This can be done by predicting the session identifier or by intercepting the session identifier and modifying it. The underlying weakness when identifiers are predictable is insufficient entropy in their generation [@cwe-330], a concern that session management guidance in NIST SP 800-63B-4 addresses directly [@nist-sp-800-63b-4].

A web session is a sequence of requests and responses between a client and a server that lasts for a period of time. During this time, every interaction between the client and the server will include a series of variables that contain the session state and allow for improvements in the user experience, security, and functionality. Usually, these variables are stored in an HTTP cookie and are used to to ensure that the user is authenticated and authorized to access the resources they need without having to re-authenticate for each request [@owasp-cs-session-management].

Session hijacking can be classified into two forms: **targeted** and **generic**. In a targeted attack, the attacker knows the victim's session identifier and is able to predict it. In a generic attack, the attacker does not know the victim's session identifier but is able to intercept the network traffic and modify the identifier. This case study demonstrates a targeted attack that identifies weaknesses in the identifier generation logic by observing behavior during normal login, and exploits the keyspace collapse that follows from pairing a global counter with a wall-clock timestamp.

Mitigation strategies for session hijacking for modern web applications include:

- **Generic Session ID Name.** Using a generic session ID name such as `id` instead of a specific or default name like `PHPSESSID` or `JSESSIONID` to avoid fingerprinting the application.
- **Sufficient Entropy.** Using a cryptographically secure random number generator (CSPRNG) to generate the session identifier to ensure that the identifier is unpredictable and unique.
- **Session ID Length.** Must be sufficiently long to make it difficult for an attacker to guess or enumerate.
- **Session ID Content.** Must be meaningless and opaque to the attacker to prevent information leakage about the session identifier and state.

Additional session management practices include using built-in mechanisms provided by the web application framework or server to manage sessions, and implementing TLS for the entire session to prevent man-in-the-middle attacks.

:::{include} _materials/webgoat.md
:::

:::{include} _materials/burp.md
:::

## Methods

This case study reproduces a predictable-identifier session hijacking against the WebGoat `HijackSession` lesson at the pinned tag `v2025.3`. The investigation proceeds in three phases: observe the structure of the `hijack_session` cookie issued during normal login, force the server to mint fresh identifiers in sequence to characterize the generation logic, then detect a counter increment that skips a value (indicating another client minted an identifier between two of the attacker's own) and predict that missing value to hijack the corresponding session. Requests are observed through Burp Suite Community Edition (see Materials). The success criterion is recovery and successful submission of a session identifier the server did not issue to the attacker.

## Results

The attack hinges on a single observation: the server allocates session identifiers from a shared counter, so a victim's identifier sits between two of the attacker's own. The flow is summarized in [](#sh-attack-flow).

:::{figure}
:label: sh-attack-flow

```{mermaid}
sequenceDiagram
  participant A as Attacker
  participant S as WebGoat
  participant V as Victim

  A->>S: request (no cookie)
  S-->>A: Set-Cookie: `N-T₀`
  V->>S: login
  S-->>V: Set-Cookie: `(N+1)-T₁`
  A->>S: request (no cookie)
  S-->>A: Set-Cookie: `(N+2)-T₂`
  Note over A: Skip from N to N+2
  Note over A: Infer victim: N+1,<br/>T₀ ≤ T₁ ≤ T₂
  A->>S: request w/ `(N+1)-T₁`
  S-->>A: Victim's session
```

Counter-skip observation enabling identifier prediction: the attacker's two observations bracket the victim's allocation between $N$ at $T_0$ and $N+2$ at $T_2$.
:::

The login page POSTs to `/WebGoat/HijackSession/login`. The form source confirms the endpoint and shows no client-side token, nonce, or anti-tampering mechanism. Whatever protects authenticated requests has to live in the session identifier itself.

```{figure} https://bac.cdn.laplacef.me/figures/session-hijacking/01-login-form.png
:label: sh-login-form
:alt: WebGoat Session Hijacking lesson login form
:width: 100%
:align: center

The WebGoat Session Hijacking lesson login form.
```

```{figure} https://bac.cdn.laplacef.me/figures/session-hijacking/02-login-form-source-code.png
:label: sh-login-form-source
:alt: WebGoat Session Hijacking lesson login form source code
:width: 100%
:align: center

Login form source confirms no client-side token or anti-tampering mechanism; protection relies on the session identifier alone.
```

Intercepting the login surfaces a `Cookie: hijack_session=<a>-<b>` header on the request [@mdn-set-cookie]. The value is two numeric fields separated by a dash, with the left field noticeably longer than the right. Two questions follow: what generates each field, and which (if either) is predictable.

```{figure} https://bac.cdn.laplacef.me/figures/session-hijacking/03-intercept-login.png
:label: sh-intercept-login
:alt: WebGoat Session Hijacking lesson login request intercepted
:width: 100%
:align: center

The intercepted login request showing the `hijack_session=<a>-<b>` cookie.
```

The right-hand field is the more legible of the two. Its thirteen-digit magnitude lines up with a Unix timestamp in milliseconds [@wikipedia-unix-time], and it advances by roughly the wall-clock interval elapsed between requests. That accounts for half of the identifier.

The left-hand field is more interesting. Replaying the same request, logging in with different credentials, and logging in and out of WebGoat all return the same value. As long as a valid session exists on the server, the cookie is stable. That's normal session-store behavior and reveals nothing about how the value is allocated.

Forcing the server to mint a fresh identifier is the next move. Deleting the cookie entirely and re-issuing the request returns a new `Set-Cookie`: the timestamp moves forward by however many seconds have passed, and the left-hand field increments by one.

```{figure} https://bac.cdn.laplacef.me/figures/session-hijacking/04-enumerate-identifier.png
:label: sh-enumerate-identifier
:alt: Enumeration of the `hijack_session` identifier
:width: 100%
:align: center

Deleting and re-issuing the request returns a new `Set-Cookie` with the left field incremented by one.
```

Repeating the deletion-and-reissue loop confirms the pattern. The left-hand field is a counter, not a random value, and it increments globally rather than per-client.

```{figure} https://bac.cdn.laplacef.me/figures/session-hijacking/05-enumerate-identifier-skip-two-1.png
:label: sh-skip-two-1
:alt: Sequential increment of the `hijack_session` counter
:width: 100%
:align: center

Sequential increment of the `hijack_session` counter across repeated reissues.
```

Every ten requests or so, the counter increments by two instead of one. The server has effectively handed out an identifier in between two of the attacker's own, and the only plausible recipient is another client logging on during that gap.

```{figure} https://bac.cdn.laplacef.me/figures/session-hijacking/06-enumerate-identifier-skip-two-2.png
:label: sh-skip-two-2
:alt: Counter skips by two when another session is allocated mid-loop
:width: 100%
:align: center

Every ten requests or so the counter skips by two, indicating another client received the missing value.
```

That observation collapses the keyspace. After seeing $N$ at timestamp $T_0$ and $N+2$ at timestamp $T_2$, the missing identifier is exactly $N+1$, and its timestamp lies in $[T_0, T_2]$. The candidate space is one counter value paired with the millisecond values in that window, on the order of a few thousand candidates for a sub-second gap and well within reach of automated submission. Submitting `hijack_session=(N+1)-T_0` as the first guess lands the victim's session: $T_1$ happened to equal $T_0$, meaning the missing identifier was minted within the same millisecond as the attacker's first observation. When that first guess misses, iterating `(N+1)-T` across the remaining milliseconds in $[T_0, T_2]$ converges within a small number of attempts.

```{figure} https://bac.cdn.laplacef.me/figures/session-hijacking/07-hijacked-session.png
:label: sh-hijacked-session
:alt: Hijacked session after submitting the inferred identifier
:width: 100%
:align: center

Submitting the inferred `(N+1)-T` identifier returns the victim's session.
```


## Discussion

The vulnerable components live under `src/main/java/org/owasp/webgoat/lessons/hijacksession/` at the `v2025.3` tag [@webgoat-v2025-3-source], split across the assignment controller and a `cas/` sub-package containing the authentication abstractions. Three pieces are load-bearing: a static-field-and-supplier pair that mints identifiers, a queue that validates them, and an auto-login routine that exposes minted identifiers to external observation.

Identifier generation is concentrated in two lines of [`HijackSessionAuthenticationProvider`](https://github.com/WebGoat/WebGoat/blob/v2025.3/src/main/java/org/owasp/webgoat/lessons/hijacksession/cas/HijackSessionAuthenticationProvider.java#L29-L34):

:::{code-block} java
:label: sh-listing-generator
:caption: Session identifier generation in `HijackSessionAuthenticationProvider`.

private static long id = new Random().nextLong() & Long.MAX_VALUE;

private static final Supplier<String> GENERATE_SESSION_ID =
    () -> ++id + "-" + Instant.now().toEpochMilli();
:::

The counter `id` is a process-global `static long`, seeded once at class load with `new Random().nextLong()` masked to non-negative values. Each invocation of the supplier pre-increments `id` and concatenates it with the current Unix epoch in milliseconds, producing the `<counter>-<epochMillis>` wire format observed in Results. Three weaknesses follow directly from this construction. The counter portion is sequential, providing no entropy in the per-identifier output; the timestamp portion is observable from any wall clock; and the seed itself is drawn from `java.util.Random` rather than `java.security.SecureRandom`, so once any single value is observed and the increment pattern recognized, all subsequent values are derivable without recovering the seed. The increment is also non-atomic: `++id` on a non-`volatile`, non-`AtomicLong` static is a torn-write hazard under concurrent invocation, though predictability rather than consistency is the dominant failure.

The login entry point is `HijackSessionAssignment.login(...)`, the [`@PostMapping("/HijackSession/login")` handler](https://github.com/WebGoat/WebGoat/blob/v2025.3/src/main/java/org/owasp/webgoat/lessons/hijacksession/HijackSessionAssignment.java#L47-L70). When the incoming `hijack_cookie` is empty, the handler builds an `Authentication` carrying the submitted credentials, passes it through `provider.authenticate(...)`, and writes the resulting identifier back to the response via `setCookie(...)`. Inside `authenticate(...)`, the supplier fires only when `authentication.getId()` is empty; the just-minted identifier is then returned to the caller but is **not** itself added to the queue used for validation. Instead, `authorizedUserAutoLogin()` runs immediately after and, with probability `~0.25` per call, mints a *separate* authenticated session via `AUTHENTICATION_SUPPLIER` (which itself invokes `GENERATE_SESSION_ID`) and enqueues that identifier into an instance-level `Queue<String> sessions` of capacity 50. Validation of an incoming cookie is then a literal `sessions.contains(cookieValue)` membership check, with no binding to the requesting client's IP, fingerprint, or prior session state.

The intuitive reading recorded in Results (a counter that occasionally skips a value, with the missing identifier behaving as if another client had received it) is faithful to how this generation logic would fail in a deployed system. A real concurrent user logging in between two of the attacker's observations would produce the same counter skip, and the attacker's predicted value would remain valid against that user's session. The WebGoat implementation synthesizes the concurrent traffic deterministically with `authorizedUserAutoLogin` so a single-operator exercise can reproduce the pattern, but the predictability of `GENERATE_SESSION_ID` is independent of that scaffolding. The empirical demonstration in Results therefore generalizes: any deployment of this supplier exposes counter increments to any external observer who can request a new cookie, and the lab's autologin produces the same observable pattern that real user traffic would.

Three counterfactual controls would close the demonstrated path. First, replacing the sequential-plus-timestamp construction with a 128-bit value drawn from `java.security.SecureRandom`, URL-safe-base64 encoded, removes the predictability that the counter-skip observation exploits; the cookie format on the wire is independent of the supplier, so only `GENERATE_SESSION_ID` needs to change [@owasp-cs-session-management; @nist-sp-800-63b-4]. Second, the identifier should be rotated immediately after successful authentication and bound to the authenticated principal, so that membership in a server-side set is necessary but not sufficient for authorization. This is the second-layer failure that survives even an entropy fix. A stronger identifier accepted on possession alone is still hijackable by any party who comes to possess it [@owasp-cs-authorization]. Third, identifiers should never be minted along paths the authentication flow does not exercise. The literal `authorizedUserAutoLogin` is a teaching device, but the underlying anti-pattern (populating a session store from a code path the legitimate user never traverses) recurs in production systems as fixture loaders, admin tooling, and session-replay debug aids that quietly broaden the keyspace an attacker can target.

Taxonomically the case sits at the intersection of two CWE classes governing identifier predictability: insufficient randomness in the seed and counter [@cwe-330], and predictability of the resulting numeric output [@cwe-340]. The validation-by-membership failure shades into the broader category of authentication failures captured by the OWASP A07:2025 entry [@owasp-top10-2025-a07]. Mitigation guidance for each of the counterfactuals above is consolidated in the Session Management Cheat Sheet [@owasp-cs-session-management] and in the identity-and-session sections of NIST SP 800-63B-4 [@nist-sp-800-63b-4].

## Conclusion

The case study reproduces a targeted session hijacking against WebGoat `v2025.3` by predicting a single missing identifier in a globally-incrementing counter paired with a millisecond timestamp. The keyspace collapse is empirical, not theoretical: two observations bracketing a counter skip reduce the candidate space to one counter value and the milliseconds in $[T_0, T_2]$, which automated submission resolves within a small number of attempts. The vulnerability does not depend on the lab's synthetic concurrent traffic. Any deployment of the same supplier exposes the same pattern to any client that can request a fresh cookie.

Read against the mitigations named in the Introduction, the failure is structural. The identifier carries neither sufficient entropy (the counter is sequential and the seed is drawn from `java.util.Random` rather than a CSPRNG) nor opaque content (the `<counter>-<epochMillis>` format reveals its own allocation order). Length is incidental once content is predictable: a thirteen-digit timestamp concatenated with a sequential counter is no harder to guess than the counter itself. Closing the demonstrated path requires a 128-bit opaque value drawn from `java.security.SecureRandom`, rotated immediately after successful authentication and bound to the authenticated principal, with no identifier minting along code paths the legitimate user does not traverse. The Session Management Cheat Sheet and NIST SP 800-63B-4 specify these properties together because each one is necessary and none is sufficient on its own.
