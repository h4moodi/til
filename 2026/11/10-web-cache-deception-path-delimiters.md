# Web Cache Deception via Path Delimiter Discrepancies

*Web application vulnerabilities & exploitation · 2026-11-10*

## What it is

Web cache deception (WCD) happens when a CDN/reverse-proxy cache and the origin server parse the same URL differently. If the origin treats a character such as `;` as a path delimiter but the cache doesn't, a request for `/my-account;x.js` is served by the origin as `/my-account` (the victim's private page) while the cache sees a "static" `.js` file and stores it. Anyone who requests that exact URL afterwards gets the victim's cached response.

## Example

Probe in an authorized test (Burp Repeater or curl), with your own session:

```bash
# 1. Does the origin ignore everything after ';'? (Spring matrix vars, Java ;jsessionid)
curl -s -o /dev/null -w '%{http_code}\n' -b 'session=ME' 'https://target/my-account;abc'

# 2. Does the cache key it as static and store it?
curl -sI -b 'session=ME' 'https://target/my-account;wcd123.js' | grep -iE 'x-cache|cf-cache-status|age|cache-control'
curl -sI                  'https://target/my-account;wcd123.js' | grep -iE 'x-cache|cf-cache-status|age'
```

- `;abc` returning the same 200 page means the origin truncates at `;`.
- The `.js` suffix triggers an extension-based cache rule on the CDN.
- A `HIT` / growing `Age` on the **unauthenticated** second request confirms a cached private response.

Other delimiters worth fuzzing: `%3F` (encoded `?`), `%00`, `%23`, `!`, `$`, `.` (Rails format), and encoded `/` (`%2F`) combined with `..%2F` for normalization variants.

## When to use it / why it matters

- **Offense:** high impact with zero injection: session-bound data (API keys, CSRF tokens, PII) leaks just by getting a victim to load a link. Use a unique cache buster per test so you never serve real users' data to others.
- **Defense:**
  - Send `Cache-Control: no-store, private` on every authenticated/dynamic response, and configure the CDN to **respect origin headers** rather than override by extension.
  - Make the origin strict: 404 on unexpected path suffixes instead of silently routing them.
  - Enable CDN checks that the `Content-Type` matches the extension (e.g. Cloudflare Cache Deception Armor).
- **Gotcha:** each poisoned URL is its own cache key, so impact is per-URL, but the attacker picks the URL and can mint fresh ones per victim.

## Sources

- [PortSwigger Research: Gotta cache 'em all: bending the rules of web cache exploitation](https://portswigger.net/research/gotta-cache-em-all)
- [PortSwigger Web Security Academy: Web cache deception](https://portswigger.net/web-security/web-cache-deception)
- [Lab: Exploiting path delimiters for web cache deception](https://portswigger.net/web-security/web-cache-deception/lab-wcd-exploiting-path-delimiters)
- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
