# Enforcing SHA-Pinned GitHub Actions
*CI/CD & software supply-chain security · 2026-11-11*

## What it is
A `uses: owner/action@v4` reference resolves a **mutable** git tag at run time, so whoever controls that repo (or steals its token) decides what code runs with your `GITHUB_TOKEN` and secrets. That is exactly how the tj-actions/changed-files compromise (CVE-2025-30066, March 2025) worked: the attacker repointed existing version tags to a commit that dumped runner memory, secrets included, into the build logs. Since August 2025, GitHub's "allowed actions" policy can **require full-length commit SHAs**, so any workflow referencing an action by tag or branch fails instead of running.

## Example
Workflow reference that passes the policy:
```yaml
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```
- 40-char SHA: immutable pointer; a repointed tag can't change what runs.
- `# v4.3.0` comment: keeps it readable, and Dependabot / Renovate bump both the SHA and the comment.

Turn it on: *Settings → Actions → General → Allowed actions and reusable workflows → "Require actions to be pinned to a full-length commit SHA"* (repo, org, or enterprise level). Bulk-convert existing workflows first, e.g.:
```bash
pinact run          # rewrites @tag refs to @sha # tag across .github/workflows
```

## When to use it / why it matters
- **Defensive:** cheap, high-value control for any org consuming third-party actions; it kills the "tag hijack" class outright. Set it at org level so new repos inherit it.
- **Gotcha – impostor commits:** GitHub resolves a SHA across the whole fork network, so a SHA that only exists in an attacker's fork can still be "pinned". Verify pins come from the upstream repo (zizmor and similar linters flag this).
- **Gotcha – it isn't transitive trust:** a pinned composite or Docker action can itself pull `@main` refs or `:latest` images internally. Review what you pin.
- **Pentest angle:** in an authorized CI/CD review, grep workflows for `uses: .*@(v?[0-9]|main|master)` and check whether the policy is enforced; unpinned third-party actions running with write tokens or cloud credentials are a reportable finding.
- The same policy also supports **blocking** specific actions (`!owner/action@*`), which is useful for fast incident response.

## Sources
- [GitHub Changelog: Actions policy now supports blocking and SHA pinning](https://github.blog/changelog/2025-08-15-github-actions-policy-now-supports-blocking-and-sha-pinning-actions/)
- [GitHub Docs: Secure use reference (pinning actions)](https://docs.github.com/en/actions/reference/security/secure-use)
- [GHSA-mrrh-fwg8-r2c3: tj-actions/changed-files compromise](https://github.com/advisories/GHSA-mrrh-fwg8-r2c3)
