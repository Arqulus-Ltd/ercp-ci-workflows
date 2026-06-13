# ercp-ci-workflows (scaffold)

Reusable GitHub Actions workflows (`on: workflow_call`) shared by the domain source repos.
Source-free: this repo holds CI templates, never application source. Staged under `bootstrap/`;
becomes `Arqulus-Ltd/ercp-ci-workflows`. Implements guide §5.2.

## Why this split

> **Automation reads source; humans don't.** Domain repos call these templates; the per-job
> `GITHUB_TOKEN` (scoped to the calling repo, minted per run) does the checkout. There is **no
> standing source-read credential**, and the DevOps-held `ercp-runners` App has no source scope.
> DevOps maintains these templates + the runner platform, never the application source.

## Workflows

| Workflow | Purpose | Key inputs / secrets |
|---|---|---|
| `build-and-push.yml` | Build image, push to ECR **by digest** via OIDC; outputs the digest | inputs: `service`, `dockerfile`, `image-repo`, `runner-label`, `aws-region`; secret: `deploy-role-arn` |
| `test.yml` | `go build/vet/test` on Go 1.26.4 | inputs: `runner-label`, `go-version` |
| `scan.yml` | gitleaks + gosec + govulncheck + Trivy (fail on HIGH/CRITICAL) | inputs: `runner-label`, `go-version` |

## `uses:` contract

A caller (in its domain repo) must grant the permissions the called workflow needs:

```yaml
permissions:
  id-token: write   # required by build-and-push (OIDC)
  contents: read
jobs:
  build-and-push:
    uses: Arqulus-Ltd/ercp-ci-workflows/.github/workflows/build-and-push.yml@v1
    with: { service: ..., dockerfile: ..., image-repo: ..., runner-label: ubuntu-latest }
    secrets: { deploy-role-arn: ${{ secrets.AWS_DEPLOY_ROLE_ARN }} }
```

See `examples/caller-workflow.yml` for a full example.

## Notes

- **No PAT, no `contents:write`** — checkout is the per-job token; the deploy identity is an OIDC role ARN passed as a secret, never a static AWS key.
- **`runner-label` is an input** so the same template runs on `ubuntu-latest` today and on ARC self-hosted runners later — no hardcoded `runs-on`.
- Pin `@v1` (a tag/release of this repo) in callers; bump deliberately.
