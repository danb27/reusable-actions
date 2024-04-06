# Reusable GitHub Workflows and Actions
My not-so-private stash of reusable GitHub workflows and actions.


## Usage

### Workflows

| Workflow                  | Required Technologies | Optional Technologies    | Notes                                                                                                                                                                                                                |
|---------------------------|-----------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| test.yml                  | Python, pytest        | CodeArtifact, pre-commit | Can optionally authenticate to CodeArtifact to install private python packages and turn on pre-commit                                                                                                                |
| push-to-code-artifact.yml | Python, CodeArtifact  |                          | See [this](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services#updating-your-github-actions-workflow) if using OpenID Connect in AWS |
