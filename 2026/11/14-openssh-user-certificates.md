# Short-Lived OpenSSH User Certificates

*Cryptography, authentication & identity · 2026-11-14*

## What it is

Instead of copying public keys into every server's `authorized_keys`, you run a small SSH CA: a keypair whose only job is to sign users' public keys. The resulting certificate binds the key to a key ID, a list of *principals* (allowed login names or roles), a validity window, and options such as forced commands or source-address limits. Servers trust the CA's public key once, so access is granted and revoked centrally, and expiry does most of the revocation work.

## Example

```bash
# One-time: create the CA (keep the private half offline or in an HSM/agent)
ssh-keygen -t ed25519 -f user_ca -C "user-ca"

# Sign alice's key: 8h lifetime, principals "alice,deploy", locked to the office range
ssh-keygen -s user_ca -I alice@laptop-2026-11-14 -n alice,deploy \
  -V +8h -O source-address=10.20.0.0/16 -z 1042 id_ed25519.pub
#   -s CA key   -I key ID (appears in sshd logs)   -n principals
#   -V validity -O cert option                      -z serial (for revocation)

ssh-keygen -L -f id_ed25519-cert.pub   # inspect: principals, validity, options
```

Server side (`/etc/ssh/sshd_config`):

```
TrustedUserCAKeys /etc/ssh/user_ca.pub
AuthorizedPrincipalsFile /etc/ssh/principals/%u   # e.g. /etc/ssh/principals/deploy contains "deploy"
RevokedKeys /etc/ssh/revoked_keys                  # key revocation list (KRL)
```

Revoke serial 1042 early: `echo "serial: 1042" > spec && ssh-keygen -k -f revoked_keys -s user_ca.pub spec`. The client picks up `id_ed25519-cert.pub` automatically.

## When to use it / why it matters

- **Defensive:** an 8-hour cert stolen from a laptop is dead by tomorrow, and offboarding means "stop signing", not editing `authorized_keys` on hundreds of hosts. Key ID and serial appear in sshd logs, so attribution beats a bare fingerprint.
- **Gotcha, principals:** without `AuthorizedPrincipalsFile`, sshd accepts any cert whose principals include the target username. A sloppy `-n root` is a skeleton key; map principals per account deliberately.
- **Gotcha, no `-V`:** omit it and the cert never expires.
- **Pentest angle:** a readable CA private key (CI secret, bastion home dir, config repo) is a critical finding: it mints access to every host that trusts it. Remediation is signing behind a service (step-ca, Vault SSH, HSM) with short TTLs.
- Host certs (`-h` + `@cert-authority` in `known_hosts`) remove blind trust-on-first-use prompts.

## Sources

- [ssh-keygen(1): CERTIFICATES](https://man.openbsd.org/ssh-keygen#CERTIFICATES)
- [sshd_config(5): TrustedUserCAKeys / AuthorizedPrincipalsFile](https://man.openbsd.org/sshd_config#TrustedUserCAKeys)
- [OpenSSH PROTOCOL.certkeys](https://cvsweb.openbsd.org/src/usr.bin/ssh/PROTOCOL.certkeys?rev=HEAD)
