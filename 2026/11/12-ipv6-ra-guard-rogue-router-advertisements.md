# Rogue IPv6 Router Advertisements and RA-Guard

*Networking & protocols · 2026-11-12*

## What it is

Any host on a link can send an ICMPv6 Router Advertisement (type 134) and become the default gateway or push a prefix/DNS server to every SLAAC client, even on a network you think is IPv4-only, since Windows and most Linux desktops prefer IPv6 when they get it. RA-Guard (RFC 6105) is the L2 switch control that only allows RAs on ports you mark as router-facing. RFC 7113 documents how naive RA-Guard can be bypassed with IPv6 extension headers or fragmentation, and what a correct implementation has to do about it.

## Example

Cisco IOS-style switch policy:

```
ipv6 nd raguard policy HOSTS
 device-role host                 ! RAs arriving on these ports are dropped
interface range Gi1/0/1 - 46
 ipv6 nd raguard attach-policy HOSTS
interface Gi1/0/48
 ipv6 nd raguard attach-policy UPLINK   ! policy with device-role router
```

Host-side hardening and detection on Linux:

```bash
# Servers with static addressing: ignore RAs entirely
sysctl -w net.ipv6.conf.eth0.accept_ra=0
# Watch for RAs on the wire (who is advertising, and which prefixes)
tcpdump -ni eth0 -vv 'icmp6 and ip6[40] == 134'
```

- `device-role host`: the switch drops RAs (and redirects) received on access ports.
- `accept_ra=0`: the kernel ignores RAs, so no rogue default route or SLAAC address appears.
- `ip6[40] == 134`: matches the ICMPv6 type byte right after the 40-byte fixed header.

## When to use it / why it matters

- **Offensive (authorized tests):** tools like mitm6 combine rogue RA/DHCPv6 to become a victim's DNS server, which is often the first step in NTLM relay chains on AD networks.
- **Defensive:** turn on RA-Guard (plus DHCPv6 Guard) on every access port, even if you "don't use IPv6".
- **Gotcha 1:** RA-Guard that only checks the first header is bypassed by putting a Destination Options header in front of the ICMPv6, or by fragmenting so the ICMPv6 header lands in the second fragment. Per RFC 7113, the switch must walk the whole header chain and drop packets where it can't find the upper-layer header.
- **Gotcha 2:** the tcpdump filter above misses those same evasive packets. Better detection: alert on any RA from a MAC that isn't a known router.

## Sources

- [RFC 6105: IPv6 Router Advertisement Guard](https://datatracker.ietf.org/doc/html/rfc6105)
- [RFC 7113: Implementation Advice for IPv6 RA-Guard](https://www.rfc-editor.org/rfc/rfc7113.html)
- [RFC 6104: Rogue IPv6 Router Advertisement Problem Statement](https://datatracker.ietf.org/doc/rfc6104/)
- [fox-it/mitm6](https://github.com/fox-it/mitm6)
