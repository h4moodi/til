# JWT Algorithm Confusion: Pin the Algorithm to the Key

*Cryptography, authentication & identity · 2026-11-28*

## What it is

A JWT's header names its own signing algorithm (`alg`). If a verifier lets that header pick the algorithm, an attacker can take a service that signs with RS256 and send a token labelled HS256, signed with HMAC using the server's *public* RSA key as the shared secret. A generic `verify(token, key)` then computes HMAC with the same public key bytes, the signature matches, and forged claims are accepted. Root cause: attacker input chooses how a key is used.

## Example

```python
import jwt  # PyJWT >= 2.4

PUBLIC_KEY = open("issuer_pub.pem").read()

claims = jwt.decode(
    token,
    PUBLIC_KEY,
    algorithms=["RS256"],          # allow-list exactly one algorithm for this key
    audience="api.example.com",    # also bind the token to this service
    issuer="https://idp.example.com",
    options={"require": ["exp", "iat", "sub"]},
)
```

- `algorithms=["RS256"]`: the header's `alg` must match this list, so `HS256` and `none` are rejected before any crypto runs.
- One key, one algorithm: never pass a list like `["RS256", "HS256"]` with a single key object. That list is exactly how the bug gets in.
- `audience` / `issuer` / `require`: so a validly signed token minted for another service still fails.

To test a target (authorized only): grab the public key from `/.well-known/jwks.json` or derive it from two signed tokens, re-sign a modified payload as HS256 with the PEM bytes, and see whether it's accepted. Burp's JWT Editor extension does this.

## When to use it / why it matters

- Defensive: any verifier that reads `alg` from the token to pick a code path is suspect. Grep for `decode(` calls with no algorithm argument or with mixed symmetric and asymmetric algorithms.
- Gotcha: PyJWT tries to refuse public keys as HMAC secrets by blocklisting PEM/SSH formats, but that guard has been bypassed more than once (OpenSSH Ed25519 keys in CVE-2022-29217, later DER-encoded keys). Format sniffing is not a fix; pinning the algorithm is.
- Same family: `alg: none`, `kid` path traversal or SQLi, and `jku`/`x5u` header URLs pointing at attacker-hosted keys. The fix pattern is the same: the server decides the key and algorithm, never the token.
- Detection: log and alert on tokens whose `alg` differs from the issuer's configured algorithm. Legitimate clients never send those.

## Sources

- [RFC 8725: JSON Web Token Best Current Practices, §2.1 and §3.1](https://www.rfc-editor.org/rfc/rfc8725)
- [PortSwigger Web Security Academy: Algorithm confusion attacks](https://portswigger.net/web-security/jwt/algorithm-confusion)
- [PyJWT usage docs](https://pyjwt.readthedocs.io/en/stable/usage.html)
- [GHSA-p4g4-x82p-q773: PyJWT accepts DER public keys as HMAC secrets](https://github.com/jpadilla/pyjwt/security/advisories/GHSA-p4g4-x82p-q773)
