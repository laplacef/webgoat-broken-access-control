# webgoat-broken-access-control

[![Live site](https://img.shields.io/badge/live-laplacef.github.io-2962ff)](https://laplacef.github.io/webgoat-broken-access-control/)
[![WebGoat](https://img.shields.io/badge/WebGoat-v2025.3-ff6f00)](https://github.com/WebGoat/WebGoat)
[![mitmproxy](https://img.shields.io/badge/mitmproxy-v12.2.0-3eaf7c)](https://mitmproxy.org)
[![Content license](https://img.shields.io/badge/content-CC%20BY--NC--SA%204.0-lightgrey?logo=creativecommons)](LICENSE.md)
[![Code license](https://img.shields.io/badge/code-Apache%202.0-d22128)](LICENSE.md)

---

Hands-on walkthrough of the [OWASP WebGoat](https://github.com/WebGoat/WebGoat) **Broken Access Control** section. [Read the published walkthrough](https://laplacef.github.io/webgoat-broken-access-control/).

## Lab setup

Match the pinned versions to keep findings deterministic across the series.

### WebGoat `v2025.3`

```bash
docker run --rm -p 8080:8080 -p 9090:9090 webgoat/webgoat:v2025.3
```

### mitmproxy `v12.2.0`

Run mitmproxy in **reverse mode** in front of WebGoat:

```bash
mitmproxy --mode reverse:http://localhost:8080 --listen-port 8888
```

## Citations

This walkthrough is grounded in the following sources:

- [OWASP Top 10 2021 — A01: Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [OWASP API Security Top 10 — API1:2023 Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [OWASP Cheat Sheet — Authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [OWASP Cheat Sheet — Access Control](https://cheatsheetseries.owasp.org/cheatsheets/Access_Control_Cheat_Sheet.html)
- [OWASP WebGoat](https://github.com/WebGoat/WebGoat) — the lab application
- [mitmproxy](https://mitmproxy.org) — used to intercept and replay lab traffic
- [mystmd](https://mystmd.org) — documentation engine ([Cockett et al., 2026](https://zenodo.org/records/20301071))

## License

Dual-licensed:

- **Content** (walkthrough prose and screenshots): [CC BY-NC-SA 4.0](LICENSE.md)
- **Code** (everything else in the repo): [Apache-2.0](LICENSE.md)

See [LICENSE.md](LICENSE.md) for the full deed and commercial-licensing path.
