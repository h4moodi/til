# LLMNR and NBT-NS Poisoning: Spotting and Killing the Fallback

*Networking & protocols · 2026-11-19*

## What it is

When a Windows host can't resolve a name through DNS (a typo like `\\fileserv`, a stale mapped drive, a WPAD lookup), it falls back to Link-Local Multicast Name Resolution (LLMNR, UDP 5355 to `224.0.0.252` / `ff02::1:3`, RFC 4795) and NetBIOS Name Service (NBT-NS, UDP 137 broadcast). Neither protocol authenticates the responder, so anyone on the same segment can answer "that's me". The victim then connects to the attacker and often offers NTLM authentication, which can be captured for offline cracking or relayed (MITRE ATT&CK T1557.001).

## Example

Audit first, without poisoning anything (authorized engagement, your own segment):

```bash
sudo responder -I eth0 -A
```

- `-I eth0`: interface to listen on.
- `-A`: analyze mode; logs which hosts send LLMNR/NBT-NS/mDNS queries and for what names, but never answers.

Fix on the Windows side (GPO equivalents in comments):

```powershell
# Computer Config > Admin Templates > Network > DNS Client > "Turn off multicast name resolution" = Enabled
New-Item 'HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient' -Force |
  New-ItemProperty -Name EnableMulticast -Value 0 -PropertyType DWord -Force

# NBT-NS has no GPO switch: set NetbiosOptions=2 (disable) on every adapter
Get-ChildItem 'HKLM:\SYSTEM\CurrentControlSet\Services\NetBT\Parameters\Interfaces' |
  ForEach-Object { Set-ItemProperty $_.PSPath -Name NetbiosOptions -Value 2 }
```

- `EnableMulticast = 0`: stops the DNS client from sending LLMNR queries.
- `NetbiosOptions = 2`: disables NetBIOS over TCP/IP per interface (`0` = follow DHCP, `1` = enabled). Your DHCP server can also push this via vendor option 001.

## When to use it / why it matters

- Offensively, it's one of the first things checked on an internal test: no exploit, just a listener and a misconfigured fallback. Analyze mode alone tells the client how exposed they are.
- Defensively, disabling the fallbacks removes the bait, but pair it with SMB signing / EPA and NTLM restrictions so a different coercion path can't be relayed either.
- Gotchas: the NetBT setting is per adapter, so new NICs/VPN adapters come back enabled unless you push it via DHCP or a startup script. mDNS (UDP 5353) is a third fallback on modern Windows, so check it too. Detection: alert on hosts answering 5355/137 that aren't DNS/WINS servers, or seed a fake name lookup and flag anyone who replies.

## Sources

- [MITRE ATT&CK T1557.001: LLMNR/NBT-NS Poisoning and SMB Relay](https://attack.mitre.org/techniques/T1557/001/)
- [RFC 4795: Link-Local Multicast Name Resolution](https://www.rfc-editor.org/rfc/rfc4795)
- [CISA CM0053: Disable LLMNR](https://www.cisa.gov/eviction-strategies-tool/info-countermeasures/CM0053)
- [Responder (lgandx) on GitHub](https://github.com/lgandx/Responder)
