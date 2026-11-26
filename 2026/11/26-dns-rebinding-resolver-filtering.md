# Blocking DNS Rebinding at the Resolver

*Networking & protocols · 2026-11-26*

## What it is

DNS rebinding gets around the same-origin policy by changing what a hostname points to while the page is still open. The victim loads `attacker.example`, which first resolves to the attacker's server. A later lookup, after a very short TTL, returns an internal address like `192.168.1.1` or `127.0.0.1`. The browser still treats that as the same origin, so the attacker's JavaScript can read responses from routers, admin panels and localhost dev servers that never meant to be reachable from the internet. The simplest network-wide defence is a resolver that won't hand out private addresses in answers for public names.

## Example

`/etc/dnsmasq.d/rebind.conf`:

```ini
stop-dns-rebind                 # drop upstream answers that point into RFC1918, link-local, ULA, etc.
rebind-localhost-ok             # keep 127.0.0.0/8 and ::1 answers (needed by some DNSBLs)
rebind-domain-ok=/corp.internal/ # split-horizon zones that really do resolve to private IPs
log-queries                     # rejected answers show up as "possible DNS-rebind attack detected"
```

The same thing in Unbound (`unbound.conf`):

```yaml
server:
  private-address: 10.0.0.0/8
  private-address: 172.16.0.0/12
  private-address: 192.168.0.0/16
  private-address: 169.254.0.0/16
  private-address: fd00::/8
  private-address: fe80::/10
  private-domain: "corp.internal"   # allow private answers for this zone only
```

Test it: `dig @127.0.0.1 7f000001.c0a80001.rbndr.us` should return nothing (or NXDOMAIN) once filtering is on.

## When to use it / why it matters

- **Offense (authorized tests):** rebinding still works against unauthenticated device APIs and services that trust localhost, like debug ports, IoT web UIs and local AI or model servers. Tools like `rbndr` and NCC's Singularity are the usual way to prove a finding.
- **Defense in layers:** filtering at the resolver only covers clients that use that resolver. DoH in the browser or a hard-coded `8.8.8.8` skips it. The fix that actually works is on the service: check the `Host` header against an allowlist, require authentication even on localhost, and bind to a Unix socket where you can.
- **Browsers are catching up:** Chrome's Local Network Access work replaces Private Network Access preflights with a user permission prompt before a public site can reach private or loopback addresses. It's a useful extra layer, but don't treat it as the whole fix.
- **Gotcha:** split-horizon DNS breaks quietly unless you exempt internal zones with `rebind-domain-ok` / `private-domain`.

## Sources

- [dnsmasq(8) man page](https://manpages.debian.org/trixie/dnsmasq-base/dnsmasq.8.en.html)
- [pfSense docs: DNS Rebinding Protections](https://docs.netgate.com/pfsense/en/latest/services/dns/rebinding.html)
- [Chrome for Developers: Local Network Access](https://developer.chrome.com/blog/local-network-access)
- [APNIC Blog: State of DNS rebinding in 2023](https://blog.apnic.net/2023/06/27/state-of-dns-rebinding-in-2023/)
