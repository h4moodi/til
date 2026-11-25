# Pwn Requests and the New `pull_request_target` Guardrails

*CI/CD & software supply-chain security · 2026-11-25*

## What it is

`pull_request_target` runs the workflow file from the base branch with the base repo's `GITHUB_TOKEN`, secrets and cache scope. A "pwn request" happens when that privileged job checks out the fork's head and executes it (`npm ci`, `make test`, a build script), handing secrets to whoever opened the PR. GitHub is now closing this from two sides: `actions/checkout` v7 (backported to floating major tags in July 2026) refuses to fetch fork PR code under `pull_request_target`/`workflow_run`, and from **December 2, 2026** public repos block `pull_request_target` runs entirely unless an Actions event policy allows them.

## Example

Split untrusted work from privileged work:

```yaml
# ci.yml: untrusted, no secrets for forks
on: pull_request
permissions: { contents: read }
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with: { persist-credentials: false }
      - run: npm ci && npm test
      - uses: actions/upload-artifact@v4
        with: { name: results, path: results.json }
```

```yaml
# report.yml: privileged, never runs PR code
on:
  workflow_run: { workflows: [ci], types: [completed] }
permissions: { pull-requests: write }
jobs:
  comment:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with: { name: results, run-id: ${{ github.event.workflow_run.id }}, github-token: ${{ github.token }} }
      - run: node post-comment.js   # treat results.json as untrusted *data*
```

- `pull_request` gets a read-only token and no secrets for fork PRs, so running their code there is fine.
- `workflow_run` carries the write permission but only consumes the artifact, never executes it.
- `allow-unsafe-pr-checkout: true` is the v7 escape hatch; grep for it in review like you would `--no-verify`.

## When to use it / why it matters

- **Audit now:** check the repo's Actions Insights for runs the Dec 2 rule would block, then either migrate to the split above or allow specific workflow files in an event policy.
- **The checkout guard is narrow:** it only stops `actions/checkout` from fetching the fork head/merge commit. `run: gh pr checkout`, `git fetch origin pull/N/head`, or interpolating `${{ github.event.pull_request.title }}` into a shell are still exploitable.
- **Artifacts are attacker-controlled:** validate them, don't `unzip` into the workspace and `source` them.
- Offensive angle: when assessing a target's CI, `pull_request_target` + explicit `ref:` to the head SHA is still the first pattern to look for in older pinned-SHA workflows.

## Sources

- [GitHub Changelog: pull_request_target blocked by default December 2](https://github.blog/changelog/2026-11-02-pull_request_target-blocked-by-default-december-2/)
- [GitHub Changelog: Safer pull_request_target defaults for actions/checkout](https://github.blog/changelog/2026-06-18-safer-pull_request_target-defaults-for-github-actions-checkout/)
- [GitHub Docs: Securely using pull_request_target](https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target)
- [actions/checkout README](https://github.com/actions/checkout)
