# Reusable GitHub Workflows and Actions
Reusable workflows intended to be called from other repositories via `workflow_call`.

## Workflows

| Workflow                  | Required Technologies | Optional Technologies    | Notes                                                                                                                                                                                                                |
|---------------------------|-----------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| test.yml                  | Python, pytest        | CodeArtifact, pre-commit | Can optionally authenticate to CodeArtifact to install private python packages and turn on pre-commit                                                                                                                |
| push-code-artifact.yml    | Python, CodeArtifact  |                          | See [this](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services#updating-your-github-actions-workflow) if using OpenID Connect in AWS |

## Example usage

### `.github/workflows/test.yml`

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  tests:
    uses: danb27/reusable-actions/.github/workflows/test.yml@main
    with:
      python-versions: '["3.11", "3.12"]'
      run_pre_commit: true
      enable-codeartifact-login: false
```

### `.github/workflows/push-code-artifact.yml`

```yaml
name: Publish Package

on:
  workflow_dispatch:
  push:
    tags:
      - "v*"

jobs:
  publish:
    uses: danb27/reusable-actions/.github/workflows/push-code-artifact.yml@main
    with:
      python-version: "3.11"
    secrets:
      AWS_ROLE: ${{ secrets.AWS_ROLE }}
      AWS_REGION: ${{ secrets.AWS_REGION }}
      AWS_CODEARTIFACT_DOMAIN: ${{ secrets.AWS_CODEARTIFACT_DOMAIN }}
      AWS_CODEARTIFACT_REPOSITORY: ${{ secrets.AWS_CODEARTIFACT_REPOSITORY }}
```
