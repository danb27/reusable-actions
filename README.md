# Reusable GitHub Workflows and Actions
Reusable workflows intended to be called from other repositories via `workflow_call`.

## Workflows

| Workflow                  | Required Technologies | Optional Technologies    | Notes                                                                                                                                                                                                                |
|---------------------------|-----------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| test.yml                  | Python, pytest        | CodeArtifact, pre-commit | Can optionally authenticate to CodeArtifact to install private python packages and turn on pre-commit                                                                                                                |
| push-code-artifact.yml    | Python, CodeArtifact  |                          | See [this](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services#updating-your-github-actions-workflow) if using OpenID Connect in AWS |
| terraform-plan.yml        | Terraform, AWS OIDC   | Cloudflare               | Formats, validates and plans, then comments the plan on the pull request. Runs with `-lock=false` so a plan can never block an apply                                                                                 |
| terraform-apply.yml       | Terraform, AWS OIDC   | Cloudflare               | Applies and returns non-sensitive outputs as JSON. Set `environment` to put a required-reviewer gate in front of it                                                                                                  |

## Terraform workflows

Both take AWS credentials by OIDC — no long-lived keys — and both accept an
optional `TF_VARS_JSON` secret: a JSON object of variable values written to
`ci.auto.tfvars.json` in the working directory, which Terraform picks up
automatically. Use it for values that shouldn't live in the repository.

```json
{ "cloudflare_account_id": "abc123", "allowed_emails": ["you@example.com"] }
```

One blob rather than a named secret per variable is a deliberate trade: it keeps
the workflow interface stable as configurations grow, at the cost of granularity.
Provider credentials that aren't Terraform variables — `CLOUDFLARE_API_TOKEN` —
stay named secrets so GitHub masks them individually.

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

### `.github/workflows/terraform-plan.yml`

```yaml
name: PR

on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  id-token: write
  pull-requests: write

jobs:
  plan:
    uses: danb27/reusable-actions/.github/workflows/terraform-plan.yml@main
    with:
      working-directory: infra
    secrets:
      AWS_ROLE: ${{ secrets.AWS_ROLE }}
      TF_VARS_JSON: ${{ secrets.TF_VARS_JSON }}
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

### `.github/workflows/terraform-apply.yml`

```yaml
name: Deploy

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write

concurrency:
  group: deploy
  cancel-in-progress: false

jobs:
  apply:
    uses: danb27/reusable-actions/.github/workflows/terraform-apply.yml@main
    with:
      working-directory: infra
    secrets:
      AWS_ROLE: ${{ secrets.AWS_ROLE }}
      TF_VARS_JSON: ${{ secrets.TF_VARS_JSON }}
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

Applies should always run under a `concurrency` group so two merges cannot
apply the same state at once.
