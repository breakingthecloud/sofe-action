<p align="center">
  <img alt="SOFE Action" src="https://img.shields.io/badge/🤖-SOFE_Evaluate-6C2D82?style=for-the-badge" height="50">
</p>

<p align="center">
  <b>Run FinOps policy evaluations in CI/CD</b><br>
  Cloud mode (live AWS scan) or Terraform mode (pre-deploy plan scan).
</p>

<p align="center">
  <a href="#quick-start">Quick Start</a>
  ·
  <a href="#inputs">Inputs</a>
  ·
  <a href="#how-it-works">How It Works</a>
  ·
  <a href="#ecosystem">Ecosystem</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Marketplace-SOFE_Evaluate-6C2D82?style=flat-square&logo=github" alt="Marketplace">
  <img src="https://img.shields.io/badge/license-Apache_2.0-6C2D82?style=flat-square" alt="License">
  <img src="https://img.shields.io/badge/modes-Cloud+TF-6C2D82?style=flat-square" alt="Cloud + Terraform">
  <img src="https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat-square" alt="PRs">
</p>

---

Catch cost violations before they reach production. Two modes:
- **Cloud mode**: Scan live AWS resources for idle resources, tagging violations, and savings
- **Terraform mode**: Scan `tfplan.json` pre-deploy to catch policy violations before `terraform apply`

```yaml
- uses: breakingthecloud/sofe-action@v2
  with:
    api-key: ${{ secrets.SOFE_API_KEY }}
    fail-on: high
```

## Quick Start — Cloud Mode

```yaml
- uses: breakingthecloud/sofe-action@v2
  with:
    api-key: ${{ secrets.SOFE_API_KEY }}
    mode: cloud
    fail-on: high
```

## Quick Start — Terraform Mode

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

      - name: Terraform Plan
        run: |
          cd infra/
          terraform init && terraform plan -out=tfplan
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
| `api-key` | SOFE API key (`sk_sofe_xxx`) | ✅ | — | both |
| `mode` | `cloud` or `terraform` | ❌ | `cloud` | — |
| `plan-file` | Path to `tfplan.json` | ✅ (tf) | — | terraform |
| `fail-on` | Fail if findings ≥ severity | ❌ | (none) | both |
| `resource-types` | Filter resource types | ❌ | all | cloud |
| `format` | Output: table, json, markdown | ❌ | `table` | both |
| `version` | CLI version to install | ❌ | `latest` | cloud |

## Outputs

| Output | Description |
|--------|-------------|
| `findings-count` | Number of findings detected |
| `resources-scanned` | Number of resources scanned |
| `failed` | `true` if fail-on threshold was exceeded |

## How It Works

### Cloud Mode
1. Installs the SOFE CLI binary
2. Runs `sofe evaluate --cloud` with your API key
3. SOFE evaluates your connected AWS account
4. Returns findings + optional pipeline failure

### Terraform Mode
1. Reads your `tfplan.json` file (no CLI install needed)
2. POSTs it to `api.sofe.dev/terraform/scan`
3. Evaluates 6 pre-deploy policies (tags, sizing, encryption, etc.)
4. Returns findings + optional pipeline failure

**Key advantage:** Same policies work pre AND post deploy. Write once, enforce everywhere.

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

```diff
-- uses: breakingthecloud/sofe-action@v1
+- uses: breakingthecloud/sofe-action@v2
   with:
     api-key: ${{ secrets.SOFE_API_KEY }}
+    mode: cloud
     fail-on: high
```

## Prerequisites

- SOFE account at [platform.sofe.dev](https://platform.sofe.dev)
- API key (add as `SOFE_API_KEY` secret in your repo)
- **Cloud mode**: AWS account connected in the SOFE platform
- **Terraform mode**: `terraform show -json` output as `.json` file

## Ecosystem

| Project | Description |
|---------|-------------|
| [sofe](https://github.com/breakingthecloud/sofe) | Python engine (collectors + policies) |
| [sofe-server](https://github.com/breakingthecloud/sofe-server) | REST API server |
| [sofe-cli](https://github.com/breakingthecloud/sofe-cli) | Go CLI (19 commands, TUI) |
| [platform.sofe.dev](https://platform.sofe.dev) | SaaS dashboard (free tier) |
| [sofe.dev/docs](https://sofe.dev/docs) | Documentation |

## License

Apache 2.0 — see [LICENSE](LICENSE).

---

<p align="center">
  <a href="https://sofe.dev">sofe.dev</a> · <a href="https://github.com/breakingthecloud/sofe">Engine</a> · <a href="https://finoptix.dev">finoptix.dev</a>
</p>
<p align="center">
  <sub>Catch cost violations before they reach production.</sub>
</p>
