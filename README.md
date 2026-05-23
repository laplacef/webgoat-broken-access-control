# webgoat-broken-access-control

[![Lessons](https://img.shields.io/badge/lessons-v1.0.0-2962ff)](https://laplacef.github.io/webgoat-broken-access-control/)
[![WebGoat](https://img.shields.io/badge/WebGoat-v2025.3-ff6f00)](https://github.com/WebGoat/WebGoat)
[![mitmproxy](https://img.shields.io/badge/mitmproxy-v12.2.0-purple)](https://mitmproxy.org)
[![burpsuite](https://img.shields.io/badge/burpsuite-v2025.10.6-3eaf7c)](https://portswigger.net/burp)
[![Content license](https://img.shields.io/badge/content-CC%20BY--NC--SA%204.0-lightgrey?logo=creativecommons)](LICENSE.md)
[![Code license](https://img.shields.io/badge/code-Apache%202.0-d22128)](LICENSE.md)

---

Introductory lessons on exploiting **Broken Access Control** vulnerabilities, with hands-on walkthroughs of the [OWASP WebGoat](https://github.com/WebGoat/WebGoat) labs.

[Read the released walkthroughs](https://laplacef.github.io/webgoat-broken-access-control/).

## Lab setup

### Target application

**WebGoat** `v2025.3`

```bash
docker run --rm -p 8080:8080 -p 9090:9090 webgoat/webgoat:v2025.3
```

### Proxy tools

**mitmproxy** `v12.2.0`

```bash
mitmproxy --mode reverse:http://localhost:8080 --listen-port 8888
```

**Burp Suite** `v2025.10.6`

```bash
burpsuite --user-config-file=burp-config.json
```

## Citations

- [OWASP Top 10 2021 — A01: Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [OWASP API Security Top 10 — API1:2023 Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [OWASP Cheat Sheet — Authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [OWASP Cheat Sheet — Access Control](https://cheatsheetseries.owasp.org/cheatsheets/Access_Control_Cheat_Sheet.html)
- [OWASP WebGoat](https://github.com/WebGoat/WebGoat)
- [Burp Suite](https://portswigger.net/burp)
- [mitmproxy](https://mitmproxy.org)
- [mystmd](https://mystmd.org)

## License

- **Content** (walkthrough prose and screenshots): [CC BY-NC-SA 4.0](LICENSE.md)
- **Code** (everything else in the repo): [Apache-2.0](LICENSE.md)

See [LICENSE.md](LICENSE.md) for the full deed and commercial-licensing path.
