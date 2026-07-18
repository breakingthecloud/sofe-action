# SOFE Evaluate — GitHub Action

> Run FinOps policy evaluations in CI/CD. Two modes:
> - **Cloud mode**: Scan live AWS resources for idle resources, tagging violations, and savings
> - **Terraform mode**: Scan `tfplan.json` pre-deploy to catch policy violations before `terraform apply`

## Quick Start — Cloud Mode (v1 compatible)

```yaml
- uses: breakingthecloud/sofe-action@v2
  with:
    api-key: ${{ secrets.SOFE_API_KEY }}
    fail-on: high
```

## Quick Start — Terraform Mode (v2)

```yaml
- run: terraform plan -out=tfplan && terraform show -json tfplan > tfplan.json
- uses: breakingthecloud/sofe-action@v2
  with:
    api-key: ${{ secrets.SOFE_API_KEY }}
    mode: terraform
    plan-file: tfplan.json
    fail-on: high
```

## Full Example — Terraform FinOps Gate

```yaml
name: FinOps Pre-Deploy Gate
on: pull_request

jobs:
  terraform-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.x

      - name: Terraform Plan
        run: |
          cd infra/
          terraform init
          terraform plan -out=tfplan
          terraform show -json tfplan > tfplan.json

      - name: SOFE Pre-Deploy Scan
        uses: breakingthecloud/sofe-action@v2
        id: sofe
        with:
          api-key: ${{ secrets.SOFE_API_KEY }}
          mode: terraform
          plan-file: infra/tfplan.json
          fail-on: high

      - name: Comment findings on PR
        if: always() && github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const findings = '${{ steps.sofe.outputs.findings-count }}';
            const resources = '${{ steps.sofe.outputs.resources-scanned }}';
            const failed = '${{ steps.sofe.outputs.failed }}';
            const icon = failed === 'true' ? '❌' : '✅';
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## ${icon} SOFE Terraform Scan\n| Metric | Value |\n|--------|-------|\n| Resources scanned | ${resources} |\n| Findings | ${findings} |\n| Status | ${failed === 'true' ? 'FAILED (high+ severity)' : 'PASSED'} |`
            })
```

## Full Example — Cloud Mode

```yaml
name: FinOps Gate
on: [push, pull_request]

jobs:
  sofe-evaluate:
    runs-on: ubuntu-latest
    steps:
      - uses: breakingthecloud/sofe-action@v2
        id: sofe
        with:
          api-key: ${{ secrets.SOFE_API_KEY }}
          mode: cloud
          fail-on: high
          format: table

      - name: Comment findings on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## SOFE FinOps Report\n- Findings: ${{ steps.sofe.outputs.findings-count }}\n- Resources scanned: ${{ steps.sofe.outputs.resources-scanned }}`
            })
```

## Inputs

| Input | Description | Required | Default | Modes |
|-------|-------------|:--------:|---------|:-----:|
| `api-key` | SOFE API key (`sk_sofe_xxx`) or Bearer token | ✅ | — | both |
| `mode` | `cloud` (live AWS scan) or `terraform` (plan file scan) | ❌ | `cloud` | — |
| `plan-file` | Path to `tfplan.json` | ✅ (terraform) | — | terraform |
| `fail-on` | Fail if findings ≥ severity (`critical\|high\|medium\|low`) | ❌ | (none) | both |
| `resource-types` | Filter resource types (comma-sep) | ❌ | all | cloud |
| `format` | Output: `table`, `json`, or `markdown` | ❌ | `table` | both |
| `version` | CLI version to install | ❌ | `latest` | cloud |

## Outputs

| Output | Description |
|--------|-------------|
| `findings-count` | Number of findings detected |
| `resources-scanned` | Number of resources scanned |
| `failed` | `true` if fail-on threshold was exceeded |

## How It Works

### Cloud Mode
1. Installs the SOFE CLI binary (from GitHub Releases)
2. Runs `sofe evaluate --cloud` with your API key
3. SOFE evaluates your connected AWS account (via api.sofe.dev)
4. Returns findings + optional pipeline failure

### Terraform Mode
1. Reads your `tfplan.json` file (no CLI install needed)
2. POSTs it to `api.sofe.dev/terraform/scan`
3. Engine evaluates 6 pre-deploy policies (tags, sizing, encryption, etc.)
4. Returns findings + optional pipeline failure

**Key advantage:** Same policies work pre AND post deploy. Write once, enforce everywhere.

## Prerequisites

- SOFE account at [platform.sofe.dev](https://platform.sofe.dev)
- API key (add as `SOFE_API_KEY` secret in your repo)
- For **cloud mode**: AWS account connected in the SOFE platform
- For **terraform mode**: `terraform show -json` output as `.json` file

## Terraform Policy Examples

| Policy | What it catches |
|--------|----------------|
| `require-cost-tags` | Resource without owner/env/costcenter tag |
| `no-oversized-staging` | Large instance in non-prod environment |
| `s3-encryption-required` | Bucket without SSE configured |
| `no-public-s3` | Bucket with public access |
| `rds-multi-az-prod` | Production RDS without Multi-AZ |
| `naming-convention` | Resource not following naming standard |

## Migration from v1

`@v1` continues to work unchanged (cloud mode is the default). To upgrade:

```diff
-- uses: breakingthecloud/sofe-action@v1
+- uses: breakingthecloud/sofe-action@v2
   with:
     api-key: ${{ secrets.SOFE_API_KEY }}
+    mode: cloud  # optional, cloud is default
     fail-on: high
```

## Links

- [Get API key](https://platform.sofe.dev/keys)
- [Connect AWS account](https://platform.sofe.dev/accounts)
- [Terraform scan UI](https://platform.sofe.dev/terraform)
- [SOFE Docs](https://sofe.dev/docs)
- [CLI Reference](https://sofe.dev/docs/cli)

## License

Apache 2.0
