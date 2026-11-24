# Client-Side Path Traversal (CSPT) as a CSRF Primitive

*Web application vulnerabilities & exploitation · 2026-11-24*

## What it is

Client-side path traversal happens when frontend JavaScript drops user-controlled input into the *path* of an API call (`fetch(`/api/items/${id}`)`). Because the browser resolves `../` before sending, an attacker can steer that request to a different same-origin endpoint. The request still carries the victim's cookies, bearer header, and any anti-CSRF token the SPA attaches, so classic CSRF defenses don't stop it. Doyensec named the chain CSPT2CSRF.

## Example

Vulnerable SPA code:

```js
// /item?id=123  ->  GET /api/items/123, then POST /api/items/123/star
const id = new URLSearchParams(location.search).get("id");
fetch(`/api/items/${id}/star`, {
  method: "POST",
  headers: { "X-CSRF-Token": csrfToken },   // token added automatically
  credentials: "include",
});
```

Lure link (authorized test only):

```
https://app.example/item?id=../../admin/users/42/promote%3F
```

- `../../` walks out of `/api/items/` to `/api/`.
- `admin/users/42/promote` is the endpoint you actually want hit.
- `%3F` decodes to `?` in `URLSearchParams`, so the trailing `/star` becomes a harmless query string.

Result: the victim's browser sends a token-bearing `POST /api/admin/users/42/promote`.

Fix:

```js
if (!/^[0-9a-f-]{1,36}$/i.test(id)) throw new Error("bad id");
fetch(`/api/items/${encodeURIComponent(id)}/star`, ...);
```

The allowlist matters most. `encodeURIComponent` alone helps, but some servers or proxies decode `%2F` back into `/`.

## When to use it / why it matters

- **Offense:** grep JS bundles for template-literal URLs built from `location`, route params, or stored values (a stored ID gives you a one-click or zero-click variant). Doyensec's Burp extension flags reflected params that land in request paths. The impact is capped by the HTTP method and body the sink sends, so map which sensitive endpoints accept that method and ignore or tolerate that body.
- **Defense:** validate IDs before building URLs, reject `.`/`/`/`%` in path segments server-side, and don't count SameSite cookies or CSRF tokens as protection here: the request is same-origin by design.
- **Gotcha:** GET sinks matter too. They can pull an attacker-uploaded JSON file into the app (CSPT to XSS) or leak data through the response handler.

## Sources

- [Doyensec: Exploiting CSPT to Perform CSRF (CSPT2CSRF)](https://blog.doyensec.com/2024/07/02/cspt2csrf.html)
- [Doyensec CSPT2CSRF whitepaper (PDF)](https://www.doyensec.com/resources/Doyensec_CSPT2CSRF_Whitepaper.pdf)
- [Doyensec: Bypassing file upload restrictions to exploit CSPT](https://blog.doyensec.com/2025/01/09/cspt-file-upload.html)
- [Doyensec: CSPT resources](https://blog.doyensec.com/2025/03/27/cspt-resources.html)
