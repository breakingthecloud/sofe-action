# SOFE Evaluate — GitHub Action

> Run FinOps policy evaluations on your AWS account in CI/CD. Detect idle resources, enforce tagging, and gate deployments based on cost findings.

## Quick Start

```yaml
- uses: breakingthecloud/sofe-action@v1
  with:
    api-key: ${{ secrets.SOFE_API_KEY }}
    fail-on: high
```

## Full Example

```yaml
name: FinOps Gate
on: [push, pull_request]

jobs:
  sofe-evaluate:
    runs-on: ubuntu-latest
    steps:
      - uses: breakingthecloud/sofe-action@v1
        id: sofe
        with:
          api-key: ${{ secrets.SOFE_API_KEY }}
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

| Input | Description | Required | Default |
|-------|-------------|:--------:|---------|
| `api-key` | SOFE API key (`sk_sofe_xxx`) | ✅ | — |
| `fail-on` | Fail if findings ≥ severity | ❌ | (none) |
| `resource-types` | Filter types (comma-sep) | ❌ | all |
| `format` | Output: table/json/markdown | ❌ | table |
| `version` | CLI version to install | ❌ | latest |

## Outputs

| Output | Description |
|--------|-------------|
| `findings-count` | Number of findings detected |
| `resources-scanned` | Number of AWS resources scanned |
| `failed` | `true` if fail-on threshold was exceeded |

## How It Works

1. Installs the SOFE CLI binary (from GitHub Releases)
2. Runs `sofe evaluate --cloud` with your API key
3. SOFE evaluates your connected AWS account (via api.sofe.dev)
4. Returns findings count + optional pipeline failure

## Prerequisites

- SOFE account at [platform.sofe.dev](https://platform.sofe.dev)
- API key (add as `SOFE_API_KEY` secret in your repo)
- AWS account connected in the SOFE platform

## Links

- [Get API key](https://platform.sofe.dev/keys)
- [Connect AWS account](https://platform.sofe.dev/accounts)
- [SOFE Docs](https://sofe.dev/docs)
- [CLI Reference](https://sofe.dev/docs/cli)

## License

Apache 2.0
