# Dependency Cooldowns with `minimumReleaseAge`

*CI/CD & software supply-chain security · 2026-11-18*

## What it is

A dependency cooldown tells the package manager to refuse any version that was published less than N time ago. Most malicious npm releases (hijacked maintainer accounts, worm-style self-propagating packages) are spotted and pulled within hours to a couple of days, so making installs lag behind the newest versions by a few days avoids most of them. pnpm has it as `minimumReleaseAge` (added in 10.16, default of one day in recent docs), npm as `min-release-age` (11.10+), and Dependabot/Renovate can apply the same delay to update PRs.

## Example

`pnpm-workspace.yaml`:

```yaml
minimumReleaseAge: 4320        # minutes: ignore versions younger than 3 days
minimumReleaseAgeExclude:
  - '@myorg/*'                 # your own packages can ship right away
```

`.npmrc` (npm):

```ini
min-release-age=3              # days, not minutes
```

`.github/dependabot.yml`, so update PRs follow the same rule:

```yaml
- package-ecosystem: npm
  directory: /
  schedule: { interval: daily }
  cooldown:
    default-days: 3            # wait this long after a version is published before opening a PR
```

- The age check runs on **transitive** dependencies too, which is where most compromised versions arrive.
- If you set `minimumReleaseAge` explicitly in pnpm, strict mode is on: resolution fails instead of quietly picking a version that's too new.

## When to use it / why it matters

- **Defensive:** cheap, broad protection against the "publish malware, get pulled in by `^` ranges within minutes" pattern. Use it together with a committed lockfile, `npm ci` / `pnpm install --frozen-lockfile` in CI, and disabled install scripts. On its own it's not enough.
- **Gotcha: units differ.** pnpm uses minutes and npm uses days. If you copy `1440` into `.npmrc`, you get a four-year freeze.
- **Gotcha: security fixes.** A cooldown also delays the patched version of a CVE. Have an exclusion path ready (exclude list or a one-off override) for urgent upgrades, and keep it short-lived.
- **Gotcha: it only matters at resolution time.** An existing lockfile already pins exact versions. The cooldown matters when a lockfile is created or updated, so it's important that the bot opening update PRs enforces it too.

## Sources

- [pnpm settings: dependency resolution (`minimumReleaseAge`)](https://pnpm.io/settings/dependency-resolution)
- [Socket: pnpm 10.16 adds a setting for delayed dependency updates](https://socket.dev/blog/pnpm-10-16-adds-new-setting-for-delayed-dependency-updates)
- [Datadog Security Labs: Dependency cooldowns](https://securitylabs.datadoghq.com/articles/dependency-cooldowns/)
- [Renovate docs: Minimum release age](https://docs.renovatebot.com/key-concepts/minimum-release-age/)
