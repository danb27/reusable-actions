# Reusable GitHub Workflows and Actions
Reusable workflows intended to be called from other repositories via `workflow_call`.

## Workflows

| Workflow                  | Required Technologies | Optional Technologies    | Notes                                                                                                                                                                                                                |
|---------------------------|-----------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| test.yml                  | Python, pytest        | CodeArtifact, pre-commit | Can optionally authenticate to CodeArtifact to install private python packages and turn on pre-commit                                                                                                                |
| push-code-artifact.yml    | Python, CodeArtifact  |                          | See [this](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services#updating-your-github-actions-workflow) if using OpenID Connect in AWS |
| terraform-plan.yml        | Terraform, AWS OIDC   | Any TF provider          | Checks formatting and plans, then comments the plan on the pull request. Runs with `-lock=false` so a plan can never block an apply                                                                                  |
| terraform-apply.yml       | Terraform, AWS OIDC   | Any TF provider          | Applies on merge. Callers should set a `concurrency` group so two merges cannot apply the same state at once                                                                                                         |

## Terraform workflows

Both take AWS credentials by OIDC — no long-lived keys — and both accept an
optional `TF_VARS_JSON` secret: a JSON object of variable values written to
`ci.auto.tfvars.json` in the working directory, which Terraform picks up
automatically. Use it for values that shouldn't live in the repository.

```json
{ "cloudflare_account_id": "abc123", "allowed_emails": ["you@example.com"] }
```

Provider credentials aren't Terraform variables, so they take a second blob,
`PROVIDER_ENV_JSON`, whose keys become environment variables:

```json
{ "CLOUDFLARE_API_TOKEN": "...", "DATADOG_API_KEY": "..." }
```

A named secret per provider would mean editing this workflow every time a
downstream repo adopts one. GitHub masks a secret as a whole but not the values
*inside* a JSON blob, so both workflows mask each value explicitly with
`::add-mask::` before exporting it.

One blob rather than a named secret per value is a deliberate trade: it keeps
the workflow interface stable as configurations grow, at the cost of
granularity.

Terraform version and AWS region are hardcoded to `1.14.3` and `us-west-2`.
Neither is an input, because nothing needed a second value yet; both are a
one-line change when something does.

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
      PROVIDER_ENV_JSON: ${{ secrets.PROVIDER_ENV_JSON }}
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
      PROVIDER_ENV_JSON: ${{ secrets.PROVIDER_ENV_JSON }}
```

Applies should always run under a `concurrency` group so two merges cannot
apply the same state at once.
