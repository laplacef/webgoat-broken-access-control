---
title: "Broken Access Control"
description: "Empirical reproduction and source-level analysis of representative vulnerabilities from the OWASP WebGoat Broken Access Control section."
exports:
  - format: pdf
    template: plain_latex
    output: broken-access-control-overview.pdf
---

+++ { "part": "abstract" }

This case study empirically reproduces representative vulnerabilities from OWASP A01:2025 (Broken Access Control) against WebGoat `v2025.3`. Each case demonstrates exploitation under controlled conditions and traces the vulnerability to its implementation in the WebGoat source to identify the missing or inadequate authorization control. The work is intended both as a learning artifact for practitioners and as a reproducible reference for the broken access control category.

+++

## Introduction

A01:2025 places Broken Access Control at the top of the OWASP Top 10 [@owasp-top10-2025-a01], ranking it the most prevalent category of web application vulnerability in the most recent edition. The category subsumes a wide range of underlying weaknesses: missing object-level authorization [@cwe-639], insufficient entropy in session identifiers [@cwe-330], and absent function-level controls [@cwe-284], among others. Mitigation guidance from OWASP unifies the class under a small number of principles: deny by default, enforce authorization at the server, and avoid client-controlled keys [@owasp-cs-access-control; @owasp-cs-authorization]. What unites the weaknesses is a server's failure to enforce *who is allowed to do what on which resource*. The consequence is consistent across the class: unauthorized read, modification, or escalation of resources that should belong to other users.

The vulnerability class is well-documented at the level of taxonomy and mitigation principle. What is less commonly available in one place is a side-by-side reproduction in a controlled target paired with source-level explanation of why the implementation permits each exploit. This case study addresses that gap.

WebGoat itself is widely adopted in cybersecurity coursework at undergraduate and graduate levels as a hands-on target for teaching web application vulnerabilities, and a body of community resources has grown around it. Detailed walkthroughs [@balluff-webgoat-2025; @an1604-webgoat-solutions; @webgoat-wiki-solutions] document procedural reproduction for selected lessons, and broader markdown collections [@vernjan-webgoat; @fmauri-webgoat-solutions; @jawitold-high-level-vulnerabilities] catalogue solutions and notes across larger lesson sets covering a range of WebGoat versions. This case series treats each WebGoat vulnerability as an academic case study: empirical exploit reproduction at a pinned target version paired with source-level investigation of the WebGoat implementation, counterfactual identification of the missing or inadequate control, and explicit mapping to OWASP and CWE taxonomy. The repository is the first installment of a multi-repo series taking this approach, opening with A01:2025 Broken Access Control and tracing each finding to the WebGoat source.

## Methods

Each case study follows a consistent seven-section structure modelled on academic case-report writing:

1. **Introduction** establishes the vulnerability's framing, taxonomy placement (OWASP / CWE), the mitigation principles relevant to the class, and the literature position of the case
2. **Materials** declares the pinned target application and proxy tool versions via shared include snippets from `content/_materials/`
3. **Methods** names the investigation phases, the tools used, and the success criterion that defines a complete empirical demonstration for the case
4. **Results** reproduces the exploit empirically, opening with a Mermaid attack-flow diagram and proceeding through intercepted requests and annotated screenshots
5. **Discussion** analyses the WebGoat implementation at the source level: the vulnerable code path at the pinned tag, the absent or inadequate authorization control, and the counterfactual change that would close the vulnerability
6. **Conclusion** synthesises the empirical result against the mitigations named in the introduction
7. **References** auto-rendered from the project bibliography (`references.bib`)

The target application is pinned at WebGoat `v2025.3` to ensure reproducibility against a specific implementation. Proxy tools (`mitmproxy v12.2.0`, `Burp Suite Community Edition v2025.10.6`) vary per case based on which surfaces the relevant request flow most clearly. Pinned versions and setup commands are declared inline in each case via the shared snippets in [`content/_materials/`](https://github.com/laplacef/webgoat-broken-access-control/tree/main/content/_materials).

## Scope

The case series is bounded to vulnerabilities within the OWASP A01 (Broken Access Control) category as expressed in WebGoat `v2025.3`. Infrastructure-level concerns (network controls, transport security, container hardening, supply chain), client-side weaknesses outside the access-control class, and vulnerabilities specific to WebGoat versions other than the pinned release are out of scope. Findings generalise to the weakness class as documented in CWE and OWASP, but the source-level analysis is specific to the WebGoat implementation under study.

## Case Studies

- [Session Hijacking](./content/session-hijacking.md) — predictable session identifier generation (CWE-330) and the keyspace collapse that follows from pairing a global counter with a wall-clock timestamp.
- [Insecure Direct Object References](./content/insecure-direct-object-references.md) — missing object-level authorization (CWE-639, API1:2023 BOLA) on a profile endpoint that leaks the very identifier required to enumerate and modify other users' records.

+++ { "part": "data_availability" }

Source materials for this case series are available at <https://github.com/laplacef/webgoat-broken-access-control>. Reference tool configurations for reproducing the cases live in the [`tools/`](https://github.com/laplacef/webgoat-broken-access-control/tree/main/tools) directory. Content is licensed under CC BY-NC-SA 4.0; build tooling and configuration are licensed under Apache-2.0. The target application analysed throughout is WebGoat at tag `v2025.3` [@webgoat-v2025-3-source].

+++
