---
title: "Insecure Direct Object References"
description: "Accessing other users' resources by tampering with identifiers exposed in the URL or response body."
exports:
  - format: pdf
    template: plain_latex
    output: insecure-direct-object-references.pdf
---

---

## Objective

Intercept and modify requests to access other users' profile information and to edit values in the database.

## Requirements

- **WebGoat image:** `webgoat/webgoat:v2025.3`
- **Proxy tool:** `mitmproxy v12.2.0`

## Introduction

Insecure direct object references (IDOR) are a class of broken access control vulnerability, listed as **API1:2023 Broken Object Level Authorization (BOLA)** in the OWASP API Security Top 10. They occur when a web application exposes an object identifier (in a URL path, query parameter, or response body) and trusts the client-supplied identifier without verifying that the authenticated user is allowed to act on that specific object. The consequence is unauthorized read or write of resources that belong to other users.

:::{iframe} https://www.youtube-nocookie.com/embed/rloqMGcPMkI
:width: 100%
:::

_Video: [PwnFunction](https://www.youtube.com/@PwnFunction)._

### Types of IDOR Attacks

- **URL tampering.** The most common form. The attacker swaps the identifier in a URL or path parameter to reach a resource owned by someone else.
- **Body or document manipulation.** The identifier lives in the request body (JSON field, form input, hidden field). Tampering with it has the same effect as URL tampering.
- **File access.** The identifier maps to a file path or storage key. Predictable or directly-referenced paths let the attacker read files outside their own scope.

### Mitigation Strategies

- **Authorize every object access on the server.** For every request that touches an object, verify the authenticated user is allowed to perform the requested action on that specific object. This is the only control that closes IDOR; everything else is defense in depth.
- **Use opaque, unguessable identifiers.** UUIDs or random tokens raise the cost of enumeration but do not replace authorization checks. An attacker who learns a single valid identifier still gets through if no authorization runs.

## Walkthrough

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

The first part of the lab requires the user to authenticate using `tom` and `cat` for the username and password respectively. Simply entering the credentials and clicking the `Submit` button will authenticate the user.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/01-login-form.png
:alt: WebGoat IDOR lesson login form with credentials tom and cat entered
:width: 100%
:align: center
```

After authenticating, **observe** what the application actually returns. The question to keep asking is: what does the UI show versus what the server sends back?

Clicking on the `View Profile` button will reveal the user's profile. In this case, it reveals three pieces of information through the UI: the user's `name`, `color`, and `size`.

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

Confirming that `tom`'s own `userId` resolves through the path means the same mechanism can be turned against other accounts. Intercept the `View Profile` request, swap the `userId` for one that belongs to someone else, and the server returns their profile.

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

The actual `userId` of the next user in the database is not known, therefore the request will need to be repeated until the next user is found. This can be done manually by incrementing the `userId` by one and adding it to the path until the next user is found. Alternatively, a script can be written to automate the process or a tool with this functionality can be used (e.g. [Burp Suite](https://portswigger.net/burp) or [mitmproxy](https://docs.mitmproxy.org/stable/mitmproxytutorial-replayrequests/)).

In this case, after replaying the request multiple times while simultaneously incrementing the `userId` by one, the path `/WebGoat/IDOR/profile/2342388` reveals the profile information of the user `Buffalo Bill`.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/10-enumerated-profile.png
:alt: Profile response for /WebGoat/IDOR/profile/2342388 showing Buffalo Bill's account data after enumerating userId values
:width: 100%
:align: center
```

Lastly, it may be possible to edit values in the database by completely manipulating the type of request and passing different values for known fields.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/11-edit-values-prompt.png
:alt: WebGoat lesson prompt asking the user to modify values in another account's profile
:width: 100%
:align: center
```

So far the lab has used `GET` to read profiles; the next step uses `PUT` to update one. By convention `GET` retrieves a representation of a resource without side effects, while `PUT` replaces the resource at a given path with the payload in the request body.

WebGoat models privilege as an inverse number: a lower `role` value means higher privilege. The lab's goal is to escalate `Buffalo Bill` by lowering their `role` from `4` to `1`, and to change their `color` to `red` as a side-effect that proves the write succeeded. This can be done by sending a `PUT` request to the `/WebGoat/IDOR/profile/2342388` endpoint with the following payload:

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

Insecure direct object references (IDOR) is a type of broken access control vulnerability that allows attackers to access sensitive data or modify data without proper authorization. They occur when a web application allows users to access or modify objects directly by manipulating the object's identifier, either through the URL path or the request body.

It is critical to always be aware of the identifiers that are exposed in the URL or response body and to always be suspicious of them. Additionally, it is important to always be aware of the type of request being made and the fields that are being passed in the request body.

## References

- [OWASP Top 10 — A01:2025 Broken Access Control](https://owasp.org/Top10/2025/A01_2025-Broken_Access_Control/)
- [OWASP API Security Top 10 — API1:2023 Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [OWASP Cheat Sheet — Authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [CWE-639: Authorization Bypass Through User-Controlled Key](https://cwe.mitre.org/data/definitions/639.html)
- [MDN — Insecure Direct Object References](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/IDOR)
