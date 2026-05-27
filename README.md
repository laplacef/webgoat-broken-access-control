# webgoat-broken-access-control

[![Cases](https://img.shields.io/github/v/release/laplacef/webgoat-broken-access-control?label=cases&color=2962ff)](https://laplacef.github.io/webgoat-broken-access-control/)
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.20407989.svg)](https://doi.org/10.5281/zenodo.20407989)
[![WebGoat](https://img.shields.io/badge/WebGoat-v2025.3-ff6f00)](https://github.com/WebGoat/WebGoat)
[![Burp Suite](https://img.shields.io/badge/Burp_Suite-v2025.10.6-3eaf7c)](https://portswigger.net/burp)
[![mitmproxy](https://img.shields.io/badge/mitmproxy-v12.2.0-purple)](https://mitmproxy.org)
[![Content license](https://img.shields.io/badge/content-CC%20BY--NC--SA%204.0-lightgrey?logo=creativecommons)](LICENSE.md)
[![Code license](https://img.shields.io/badge/code-Apache%202.0-d22128)](LICENSE.md)

---

An empirical case series on **Broken Access Control** vulnerabilities, reproduced and analysed against OWASP WebGoat at a pinned release. Each case study pairs empirical demonstration of the exploit with source-level analysis of the WebGoat implementation, counterfactual identification of the missing or inadequate control, and explicit mapping to OWASP and CWE taxonomy.

## Case Studies

- [Session Hijacking](content/session-hijacking.md). Predictable session identifier generation (CWE-330, CWE-340) and the keyspace collapse that follows from pairing a global counter with a wall-clock timestamp.
- [Insecure Direct Object References](content/insecure-direct-object-references.md). Missing object-level authorization (CWE-639, API1:2023 BOLA) on a profile endpoint that leaks the very identifier required to enumerate and modify other users' records.

## How to Cite

**DOI:** <https://doi.org/10.5281/zenodo.20407989>

Per-release DOIs and pre-formatted citations are available on the [Zenodo deposit page](https://doi.org/10.5281/zenodo.20407989).

## Acknowledgments

Tools and target application referenced throughout the case series:

- [WebGoat](https://github.com/WebGoat/WebGoat) (target application)
- [Burp Suite](https://portswigger.net/burp) (interception proxy)
- [mitmproxy](https://mitmproxy.org) (interception proxy)
- [mystmd](https://mystmd.org) (publication framework)

## License

- **Content** (case study prose and screenshots): [CC BY-NC-SA 4.0](LICENSE.md)
- **Code** (everything else in the repo): [Apache-2.0](LICENSE.md)
