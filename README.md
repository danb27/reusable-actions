# Reusable GitHub Workflows and Actions
Reusable workflows intended to be called from other repositories via `workflow_call`.

## Workflows

| Workflow                  | Required Technologies | Optional Technologies    | Notes                                                                                                                                                                                                                |
|---------------------------|-----------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| test.yml                  | Python, pytest        | CodeArtifact, pre-commit | Can optionally authenticate to CodeArtifact to install private python packages and turn on pre-commit                                                                                                                |
| push-code-artifact.yml    | Python, CodeArtifact  |                          | See [this](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services#updating-your-github-actions-workflow) if using OpenID Connect in AWS |
| pr-title.yml              |                       |                          | Enforces a conventional-commit pull request title. Needed wherever squash merging feeds release-please                                                                                                              |

## Why pr-title.yml exists

A repository that squash-merges gets one commit per pull request, and that
commit's subject is the pull request title. release-please reads those commits
to decide the version bump and write the changelog — so the title is no longer
cosmetic. A title that isn't a conventional commit is silently skipped, which is
a quiet way to lose a release.

## What is deliberately not here

Terraform plan and apply workflows. An earlier draft of this repo had them;
[dflook/terraform-github-actions](https://github.com/dflook/terraform-github-actions)
does the same job and one thing ours did not: `terraform-apply` applies the plan
that was attached to the pull request and **fails if it is missing or has
changed**, so what gets applied is what was reviewed. Ours re-planned at merge
time, which means the plan you read and the plan that ran were two different
plans that usually agreed.

Consuming repositories call those actions directly. Wrapping a well-designed
action in a thin workflow of our own would add a layer without adding anything.

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

### `.github/workflows/pr-title.yml`

```yaml
name: PR Title

on:
  pull_request:
    types: [opened, edited, reopened, synchronize]

permissions:
  pull-requests: read

jobs:
  title:
    uses: danb27/reusable-actions/.github/workflows/pr-title.yml@main
```
