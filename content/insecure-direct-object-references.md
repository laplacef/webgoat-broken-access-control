---
title: "Insecure Direct Object References"
description: "Accessing other users' resources by tampering with identifiers exposed in the URL or response body."
keywords:
  - IDOR
  - CWE-639
  - OWASP API1:2023
  - Broken Object Level Authorization
exports:
  - format: pdf
    template: plain_latex
    output: insecure-direct-object-references.pdf
---

---

## Introduction

Insecure direct object reference (IDOR) is a class of broken access control vulnerability, listed as **API1:2023 Broken Object Level Authorization (BOLA)** in the OWASP API Security Top 10. It occurs when a web application exposes an object identifier (in a URL path, query parameter, or response body) and trusts the client-supplied identifier without verifying that the authenticated user is allowed to act on that specific object. The consequence is unauthorized read or write of resources that belong to other users.

### Forms of IDOR

- **URL tampering.** The most common form. The attacker swaps the identifier in a URL or path parameter to reach a resource owned by someone else.
- **Body or document manipulation.** The identifier lives in the request body (JSON field, form input, hidden field). Tampering with it has the same effect as URL tampering.
- **File access.** The identifier maps to a file path or storage key. Predictable or directly-referenced paths let the attacker read files outside their own scope.

### Mitigation Strategies

- **Authorize every object access on the server.** For every request that touches an object, verify the authenticated user is allowed to perform the requested action on that specific object. This is the only control that closes IDOR; everything else is defense in depth.
- **Use opaque, unguessable identifiers.** UUIDs or random tokens raise the cost of enumeration but do not replace authorization checks. An attacker who learns a single valid identifier still gets through if no authorization runs.

### Materials

:::{include} _materials/webgoat.md
:::

:::{include} _materials/mitmproxy.md
:::

## Methods

The attack flow has three phases: authenticate, discover that the profile endpoint leaks an identifier the UI never shows, then abuse that identifier to read and modify other users' profiles.

:::{mermaid}
sequenceDiagram
    autonumber
    participant U as Attacker
    participant W as WebGoat
    U->>W: POST /login (tom:cat)
    W-->>U: 200 OK + session
    U->>W: GET /WebGoat/IDOR/profile
    W-->>U: profile JSON (includes hidden userId)
    U->>W: GET /WebGoat/IDOR/profile/{userId} (enumerate)
    W-->>U: another user's profile
    U->>W: PUT /WebGoat/IDOR/profile/{userId} (privilege change)
    W-->>U: 200 OK, role updated
:::

Authentication requires the credentials `tom` and `cat` for the username and password respectively. Submitting them through the login form establishes the session.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/01-login-form.png
:alt: WebGoat IDOR lesson login form with credentials tom and cat entered
:width: 100%
:align: center
```

After authentication, the application's responses diverge from what the UI presents. The operative question is what the server sends back versus what the UI renders.

Clicking the `View Profile` button reveals the user's profile. The UI surfaces three pieces of information: the user's `name`, `color`, and `size`.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/02-profile-ui.png
:alt: Profile panel rendered in the WebGoat UI showing only the name, color, and size fields
:width: 100%
:align: center
```

Inspecting the source code of the page reveals the endpoint that retrieves the profile: `GET /WebGoat/IDOR/profile`.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/03-source-endpoint.png
:alt: Browser developer tools showing the page source with the GET /WebGoat/IDOR/profile endpoint highlighted
:width: 100%
:align: center
```

Intercepting the endpoint and examining the response shows that the application is returning additional information that is not visible in the UI, including the user's `role` and `userId`.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/04-intercept-profile-get.png
:alt: mitmproxy capture of the GET /WebGoat/IDOR/profile request
:width: 100%
:align: center
```

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/05-response-hidden-fields.png
:alt: Profile response body in mitmproxy showing extra role and userId fields beyond what the UI rendered
:width: 100%
:align: center
```

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/06-submit-userid.png
:alt: WebGoat lesson answer form filled with the userId recovered from the intercepted response
:width: 100%
:align: center
```

With the hidden `userId` exposed in the response body, it may be possible to access other users' profile information through path manipulation. The application is not enforcing server-side access control on the profile endpoint, nor does it use an opaque identifier that would resist guessing.

By testing the hypothesis, it is true that adding the known `userId` of `tom` to the path reveals the profile information of their account.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/07-self-userid-path.png
:alt: Profile response after appending tom's own userId to the request path, confirming the endpoint accepts a path-based identifier
:width: 100%
:align: center
```

Confirming that `tom`'s own `userId` resolves through the path means the same mechanism can be turned against other accounts. Intercepting the `View Profile` request and substituting a different account's `userId` causes the server to return that account's profile.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/08-view-profile-button.png
:alt: View Profile button in the WebGoat IDOR lesson UI
:width: 100%
:align: center
```

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/09-intercept-view-profile.png
:alt: mitmproxy intercept of the View Profile request, paused so the path can be edited before forwarding
:width: 100%
:align: center
```

Incrementing or decrementing the `userId` reveals the profile information of the next or previous user in the database. Successfully repeating this process could potentially reveal the profile information of all users in the database.

The actual `userId` of the next user in the database is not known, therefore the request will need to be repeated until the next user is found. This can be done by incrementing the `userId` by one and adding it to the path until the next user is found.

In this case, after replaying the request multiple times while simultaneously incrementing the `userId` by one, the path `/WebGoat/IDOR/profile/2342388` reveals the profile information of the user `Buffalo Bill`.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/10-enumerated-profile.png
:alt: Profile response for /WebGoat/IDOR/profile/2342388 showing Buffalo Bill's account data after enumerating userId values
:width: 100%
:align: center
```

Finally, the database state can be modified by changing the request method and supplying different values for known fields.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/11-edit-values-prompt.png
:alt: WebGoat lesson prompt asking the user to modify values in another account's profile
:width: 100%
:align: center
```

All preceding requests used `GET` to read profiles; the next request uses `PUT` to update one. By convention `GET` retrieves a representation of a resource without side effects, while `PUT` replaces the resource at a given path with the payload in the request body.

WebGoat models privilege as an inverse number: a lower `role` value means higher privilege. The objective at this step is to escalate `Buffalo Bill` by lowering their `role` from `4` to `1`, and to change their `color` to `red` as a side-effect that proves the write succeeded. Sending a `PUT` request to the `/WebGoat/IDOR/profile/2342388` endpoint with the following payload performs the escalation:

```json
{
  "role": 1,
  "color": "red",
  "size": "large",
  "name": "Buffalo Bill",
  "userId": 2342388
}
```

The server is expecting a `json` object with the following fields: `role`, `color`, `size`, `name`, and `userId`. Therefore, all fields must be passed in the payload. Additionally, the `Content-Type` request header must be set to `application/json`.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/12-original-get.png
:alt: mitmproxy capture of the original GET request prior to method and body modification
:width: 100%
:align: center
```

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/13-modified-put.png
:alt: Same request rewritten in mitmproxy as a PUT with a JSON body and Content-Type application/json
:width: 100%
:align: center
```

The response from the server shows that the request was successful and the `role` of `Buffalo Bill` was updated to `1` and the `color` was changed to `red`.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/14-put-success-response.png
:alt: Server response confirming the PUT succeeded, returning Buffalo Bill's updated profile with role 1 and color red
:width: 100%
:align: center
```

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/15-lesson-complete.png
:alt: WebGoat lesson UI marking the profile-modification step complete
:width: 100%
:align: center
```

## Conclusion

The demonstration produced both unauthorized reads and an unauthorized write against the WebGoat profile endpoint because neither control named in the introduction was present. The endpoint accepted a path-supplied `userId` without verifying that it belonged to the authenticated session, making every other user's profile reachable; sequential numeric identifiers reduced enumeration to a tractable counting exercise, converging on `Buffalo Bill` at `userId` `2342388`. Pivoting from `GET` to `PUT` against the same path lowered that account's `role` from `4` to `1`, confirming the absent ownership check applied to writes as well as reads.

Of the two mitigations named at the outset, server-side per-object authorization is the load-bearing control: an explicit ownership assertion before the lookup and before the update would have closed both the read enumeration and the privilege-escalation write. Opaque identifiers function as defense in depth rather than a substitute for authorization. They would have raised the cost of enumeration, but on their own would still have admitted any client that learned a single valid identifier.

## References

- [OWASP Top 10 — A01:2025 Broken Access Control](https://owasp.org/Top10/2025/A01_2025-Broken_Access_Control/)
- [OWASP API Security Top 10 — API1:2023 Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [OWASP Cheat Sheet — Authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [CWE-639: Authorization Bypass Through User-Controlled Key](https://cwe.mitre.org/data/definitions/639.html)
- [MDN — Insecure Direct Object References](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/IDOR)
