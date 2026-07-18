# Contributing to SOFE Action

A composite GitHub Action that runs SOFE FinOps evaluations in CI/CD.

## How It Works

1. Installs the [sofe-cli](https://github.com/breakingthecloud/sofe-cli) binary
2. Runs `sofe evaluate --cloud` with the provided API key
3. Outputs findings count, resources scanned, and pass/fail status

## Files

```
sofe-action/
├── action.yml        # Action definition (inputs, outputs, steps)
├── README.md         # Usage documentation
├── LICENSE           # Apache 2.0
└── CONTRIBUTING.md   # This file
```

## Making Changes

- Edit `action.yml` for input/output changes or step modifications
- Edit `README.md` for documentation updates
- Tag releases: `git tag -f v1 && git push origin v1 --force` (force-update v1 tag)

## What We Accept

- ✅ New inputs/outputs
- ✅ Better error handling in the bash steps
- ✅ Documentation improvements
- ✅ Support for additional platforms

## What We Don't Accept

- ❌ JavaScript/TypeScript actions (keeping it as composite/bash for simplicity)
- ❌ Embedded credentials
- ❌ Dependencies on external services beyond sofe-cli + api.sofe.dev

## Questions?

Open an issue or see [sofe.dev/docs/ci-cd](https://sofe.dev/docs/ci-cd).
