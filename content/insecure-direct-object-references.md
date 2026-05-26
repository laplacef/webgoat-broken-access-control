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

The case study empirically reproduces an insecure direct object reference vulnerability in WebGoat `v2025.3` by identifying weaknesses in the identifier generation logic and exploiting the keyspace collapse that follows from pairing a global counter with a wall-clock timestamp. The objective is to demonstrate the vulnerability and the mitigations in a controlled environment and shine a light on the importance of modern session management practices in the context of web application security.

Insecure direct object reference (IDOR) is a class of broken access control vulnerability, listed as **API1:2023 Broken Object Level Authorization (BOLA)** in the OWASP API Security Top 10 [@owasp-api-top10-2023-api1; @mdn-idor]. It occurs when a web application exposes an object identifier (in a URL path, query parameter, or response body) and trusts the client-supplied identifier without verifying that the authenticated user is allowed to act on that specific object. The consequence is unauthorized read or write of resources that belong to other users.

IDOR presents in three forms:

- **URL tampering.** The most common form. The attacker swaps the identifier in a URL or path parameter to reach a resource owned by someone else.
- **Body or document manipulation.** The identifier lives in the request body (JSON field, form input, hidden field). Tampering with it has the same effect as URL tampering.
- **File access.** The identifier maps to a file path or storage key. Predictable or directly-referenced paths let the attacker read files outside their own scope.

Two controls mitigate IDOR:

- **Authorize every object access on the server** [@owasp-cs-authorization]. For every request that touches an object, verify the authenticated user is allowed to perform the requested action on that specific object. This is the only control that closes IDOR; everything else is defense in depth.
- **Use opaque, unguessable identifiers.** UUIDs or random tokens raise the cost of enumeration but do not replace authorization checks. An attacker who learns a single valid identifier still gets through if no authorization runs.

## Materials

:::{include} _materials/webgoat.md
:::

:::{include} _materials/mitmproxy.md
:::

## Methods

This case study reproduces the IDOR vulnerability in the WebGoat profile endpoint at the pinned tag `v2025.3`. The investigation proceeds in three phases: authenticate against the lesson with the documented test credentials, intercept the `/IDOR/profile` request and inspect the response for fields the UI does not surface, then substitute the recovered identifier in subsequent requests to read and modify other accounts' profiles. The read and write steps rely on standard HTTP method semantics [@rfc-9110], with `GET` retrieving the profile representation and `PUT` replacing it. Requests are observed through mitmproxy in reverse-proxy mode (see Materials). The success criterion is the unauthorized read of another account's profile and an unauthorized write that escalates that account's `role`.

## Results

The exploit hinges on two observations: the profile response body carries an identifier the UI hides, and the parameterized endpoint returns any profile that identifier names. The full attack flow is summarized in [](#idor-attack-flow).

:::{figure}
:label: idor-attack-flow

```{mermaid}
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
```

Attack flow against the IDOR profile endpoint, from authentication through unauthorized read and privilege escalation.
:::

Authentication requires the credentials `tom` and `cat` for the username and password respectively. Submitting them through the login form establishes the session.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/01-login-form.png
:label: idor-login-form
:alt: WebGoat IDOR lesson login form with credentials tom and cat entered
:width: 100%
:align: center

Submitting `tom` / `cat` against the WebGoat IDOR lesson login form.
```

After authentication, the application's responses diverge from what the UI presents. The operative question is what the server sends back versus what the UI renders.

Clicking the `View Profile` button reveals the user's profile. The UI surfaces three pieces of information: the user's `name`, `color`, and `size`.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/02-profile-ui.png
:label: idor-profile-ui
:alt: Profile panel rendered in the WebGoat UI showing only the name, color, and size fields
:width: 100%
:align: center

The `View Profile` panel surfaces only `name`, `color`, and `size`.
```

Inspecting the source code of the page reveals the endpoint that retrieves the profile: `GET /WebGoat/IDOR/profile`.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/03-source-endpoint.png
:label: idor-source-endpoint
:alt: Browser developer tools showing the page source with the GET /WebGoat/IDOR/profile endpoint highlighted
:width: 100%
:align: center

Page source reveals the endpoint `GET /WebGoat/IDOR/profile` behind the `View Profile` button.
```

Intercepting the endpoint and examining the response shows that the application is returning additional information that is not visible in the UI, including the user's `role` and `userId`.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/04-intercept-profile-get.png
:label: idor-intercept-profile-get
:alt: mitmproxy capture of the GET /WebGoat/IDOR/profile request
:width: 100%
:align: center

The mitmproxy capture of the `GET /WebGoat/IDOR/profile` request.
```

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/05-response-hidden-fields.png
:label: idor-response-hidden-fields
:alt: Profile response body in mitmproxy showing extra role and userId fields beyond what the UI rendered
:width: 100%
:align: center

The response body carries `role` and `userId` beyond what the UI surfaces.
```

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/06-submit-userid.png
:label: idor-submit-userid
:alt: WebGoat lesson answer form filled with the userId recovered from the intercepted response
:width: 100%
:align: center

The lesson answer form accepts the `userId` recovered from the intercepted response.
```

With the hidden `userId` exposed in the response body, it may be possible to access other users' profile information through path manipulation. The application is not enforcing server-side access control on the profile endpoint, nor does it use an opaque identifier that would resist guessing.

Adding `tom`'s own `userId` to the path reveals the profile information of their account.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/07-self-userid-path.png
:label: idor-self-userid-path
:alt: Profile response after appending tom's own userId to the request path, confirming the endpoint accepts a path-based identifier
:width: 100%
:align: center

Appending `tom`'s own `userId` to the path returns the same profile, confirming the endpoint accepts a path-based identifier.
```

Confirming that `tom`'s own `userId` resolves through the path means the same mechanism can be turned against other accounts. Intercepting the `View Profile` request and substituting a different account's `userId` causes the server to return that account's profile.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/08-view-profile-button.png
:label: idor-view-profile-button
:alt: View Profile button in the WebGoat IDOR lesson UI
:width: 100%
:align: center

The `View Profile` button that triggers the parameterized profile request.
```

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/09-intercept-view-profile.png
:label: idor-intercept-view-profile
:alt: mitmproxy intercept of the View Profile request, paused so the path can be edited before forwarding
:width: 100%
:align: center

The intercepted `View Profile` request, paused for path modification before forwarding.
```

Incrementing or decrementing the `userId` reveals the profile information of the next or previous user in the database. Repeating this process enumerates additional profiles.

The next user's `userId` is not known a priori; incrementing from `tom`'s known value converges on `/WebGoat/IDOR/profile/2342388`, which returns the profile of `Buffalo Bill`.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/10-enumerated-profile.png
:label: idor-enumerated-profile
:alt: Profile response for /WebGoat/IDOR/profile/2342388 showing Buffalo Bill's account data after enumerating userId values
:width: 100%
:align: center

Substituting `2342388` in the path returns `Buffalo Bill`'s profile.
```

Finally, the database state can be modified by changing the request method and supplying different values for known fields.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/11-edit-values-prompt.png
:label: idor-edit-values-prompt
:alt: WebGoat lesson prompt asking the user to modify values in another account's profile
:width: 100%
:align: center

The lesson prompts for modification of another account's profile.
```

WebGoat models privilege as an inverse number: a lower `role` value means higher privilege. Escalating `Buffalo Bill` therefore requires lowering their `role` from `4` to `1`; setting `color` to `red` in the same request serves as a side-effect that confirms the write succeeded. Sending a `PUT` request to the `/WebGoat/IDOR/profile/2342388` endpoint with the following payload performs the escalation:

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
:label: idor-original-get
:alt: mitmproxy capture of the original GET request prior to method and body modification
:width: 100%
:align: center

The original `GET` request captured before method rewriting.
```

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/13-modified-put.png
:label: idor-modified-put
:alt: Same request rewritten in mitmproxy as a PUT with a JSON body and Content-Type application/json
:width: 100%
:align: center

The same request rewritten as a `PUT` with a JSON body and `Content-Type: application/json`.
```

The response from the server shows that the request was successful and the `role` of `Buffalo Bill` was updated to `1` and the `color` was changed to `red`.

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/14-put-success-response.png
:label: idor-put-success-response
:alt: Server response confirming the PUT succeeded, returning Buffalo Bill's updated profile with role 1 and color red
:width: 100%
:align: center

The server response confirms the write: `Buffalo Bill`'s `role` is now `1` and `color` is `red`.
```

```{figure} https://bac.cdn.laplacef.me/figures/insecure-direct-object-references/15-lesson-complete.png
:label: idor-lesson-complete
:alt: WebGoat lesson UI marking the profile-modification step complete
:width: 100%
:align: center

The WebGoat lesson UI marks the profile-modification step complete.
```

## Discussion

The two vulnerable handlers live in single-class controllers under `org/owasp/webgoat/lessons/idor/` at the `v2025.3` tag. [`IDORViewOtherProfile.completed`](https://github.com/WebGoat/WebGoat/blob/v2025.3/src/main/java/org/owasp/webgoat/lessons/idor/IDORViewOtherProfile.java) maps `GET /IDOR/profile/{userId}` ([](#idor-view-other)), and [`IDOREditOtherProfile.completed`](https://github.com/WebGoat/WebGoat/blob/v2025.3/src/main/java/org/owasp/webgoat/lessons/idor/IDOREditOtherProfile.java) maps `PUT /IDOR/profile/{userId}` ([](#idor-edit-other)). Both accept the path variable directly and pass it to `new UserProfile(userId)` to perform the lookup and (for the `PUT` handler) the field-level mutation, without comparing the path-supplied `userId` to the session-bound principal stored in `LessonSession` under the key `idor-authenticated-user-id`. The source comments name the omission explicitly: the view handler concedes that "secure code would ensure there was a horizontal access control check prior to dishing up the requested profile", and the edit handler is introduced with "accepting the user submitted ID and assuming it will be the same as the logged in userId and not checking for proper authorization".

:::{code-block} java
:label: idor-view-other
:caption: Vulnerable `GET /IDOR/profile/{userId}` handler from `IDORViewOtherProfile`.

@GetMapping(path = "/IDOR/profile/{userId}", produces = {"application/json"})
@ResponseBody
public AttackResult completed(@PathVariable("userId") String userId) {
    // ...
    UserProfile requestedProfile = new UserProfile(userId);
    // secure code would ensure there was a horizontal access control check
    // prior to dishing up the requested profile
    // ...
}
:::

:::{code-block} java
:label: idor-edit-other
:caption: Vulnerable `PUT /IDOR/profile/{userId}` handler from `IDOREditOtherProfile`.

@PutMapping(path = "/IDOR/profile/{userId}", consumes = "application/json")
@ResponseBody
public AttackResult completed(
    @PathVariable("userId") String userId,
    @RequestBody UserProfile userSubmittedProfile) {

    String authUserId = (String) userSessionData.getValue("idor-authenticated-user-id");
    // this is where it starts ... accepting the user submitted ID and assuming
    // it will be the same as the logged in userId and not checking for proper
    // authorization
    UserProfile currentUserProfile = new UserProfile(userId);
    // ...
    currentUserProfile.setColor(userSubmittedProfile.getColor());
    currentUserProfile.setRole(userSubmittedProfile.getRole());
}
:::

That the omission is structural rather than a framework limitation is evident from the sibling handler [`IDORViewOwnProfile.invoke`](https://github.com/WebGoat/WebGoat/blob/v2025.3/src/main/java/org/owasp/webgoat/lessons/idor/IDORViewOwnProfile.java) ([](#idor-view-own)), which maps `GET /IDOR/profile` with no path variable and uses the session-bound identity as the lookup parameter. The session-based identity is already present in the request context; the parameterized handlers simply ignore it in favor of the client-supplied path key. This is the defining shape of **CWE-639 (Authorization Bypass Through User-Controlled Key)** [@cwe-639]: the request carries the key, the server trusts the key, and no binding between the key and the authenticated principal is enforced before the privileged operation runs.

:::{code-block} java
:label: idor-view-own
:caption: Safe sibling handler `IDORViewOwnProfile.invoke`, deriving identity from `LessonSession`.

@GetMapping(path = {"/IDOR/own", "/IDOR/profile"}, produces = {"application/json"})
@ResponseBody
public Map<String, Object> invoke() {
    String authUserId = (String) userSessionData.getValue("idor-authenticated-user-id");
    UserProfile userProfile = new UserProfile(authUserId);
    // ...
}
:::

The counterfactual is a single guard at the top of each parameterized handler ([](#idor-counterfactual)), before the `UserProfile` is constructed from the path-supplied `userId`. For an endpoint scoped to per-user profile data, ownership equality is the correct rule, and the guard applies symmetrically to both the read and the write path. Cases with a legitimate cross-user mode (administrators editing other accounts) would extend the guard with a role check rather than weaken it. The fix is independent of the rest of the lesson logic and changes nothing about the request shape that the client sees.

:::{code-block} java
:label: idor-counterfactual
:caption: Minimal ownership guard closing both the read enumeration and the privilege-escalation write.

String authUserId = (String) userSessionData.getValue("idor-authenticated-user-id");
if (!userId.equals(authUserId)) {
    throw new AccessDeniedException("not authorized for this profile");
}
:::

The taxonomy mapping follows directly. The weakness class is **CWE-639**, the user-controlled-key specialization of the more general CWE-285 (Improper Authorization) [@cwe-285]; CWE-639 is the better fit because the defining feature of the empirical finding is that a client-supplied key is trusted without authorization, not that authorization is misconfigured in some broader sense. The API-surface category is **OWASP API1:2023 Broken Object Level Authorization**, where the missing control is per-object rather than per-function. The umbrella entry is **OWASP A01:2025 Broken Access Control** [@owasp-top10-2025-a01].

## Conclusion

The demonstration produced both unauthorized reads and an unauthorized write against the WebGoat profile endpoint because neither control named in the introduction was present. The endpoint accepted a path-supplied `userId` without verifying that it belonged to the authenticated session, making every other user's profile reachable; sequential numeric identifiers reduced enumeration to a tractable counting exercise, converging on `Buffalo Bill` at `userId` `2342388`. Pivoting from `GET` to `PUT` against the same path lowered that account's `role` from `4` to `1`, confirming the absent ownership check applied to writes as well as reads.

Of the two mitigations named at the outset, server-side per-object authorization is the load-bearing control: an explicit ownership assertion before the lookup and before the update would have closed both the read enumeration and the privilege-escalation write. Opaque identifiers function as defense in depth rather than a substitute for authorization. They would have raised the cost of enumeration, but on their own would still have admitted any client that learned a single valid identifier.
