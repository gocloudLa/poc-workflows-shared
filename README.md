# Centralized CI/CD Workflows

This repository contains reusable GitHub Actions workflows for standardizing the CI/CD pipeline across multiple services and programming languages in a large enterprise environment.

## Overview

This centralized workflow system provides:

- **Standardized pipelines** for all services regardless of programming language
- **Multi-language support**: PHP, Python, JavaScript (Node.js), and Go
- **PCI/ISO27001 compliance** with security checks and approval workflows
- **Flexible bypass mechanisms** for teams to skip or extend workflows
- **Observability integration** with Datadog
- **Automated versioning and releases**

## Repository Structure

```
.github/
  workflows/
    # Main orchestrator workflows
    shared-pull-request-events.yml
    shared-develop-push-events.yml
    shared-main-push-events.yml
    
    # Individual job workflows
    conventional-commit-check.yml
    linter-php.yml
    linter-python.yml
    linter-javascript.yml
    linter-go.yml
    security-scan.yml
    unit-tests.yml
    coverage-validation.yml
    build-and-push.yml
    deploy-dev.yml
    deploy-staging.yml
    deploy-production.yml
    deploy-verification.yml
    health-check.yml
    staging-validation.yml
    rollback-mechanism.yml
    release-and-changelog.yml
```

## Workflow Flows

### Feature Branch → Pull Request to `develop`

When a PR is opened against the `develop` branch, the following validations run:

1. **Conventional Commit Check**: Validates PR title follows conventional commit format
2. **Linter**: Runs language-specific linter (auto-detected or specified)
3. **Security Scan**: CodeQL analysis, dependency review, and container scanning
4. **Unit Tests**: Executes unit tests for the detected language
5. **Coverage Validation**: Ensures test coverage meets threshold (default: 80%)

### `develop` Branch (Push)

When code is merged to `develop`, the following pipeline runs:

1. **Security Scan**: Additional security checks
2. **Build & Tag**: Builds Docker image and pushes to ECR with tags:
   - `{commit-sha-short}` (e.g., `a1b2c3d`)
   - `develop-latest`
3. **Deploy to Dev**: Deploys to development environment
4. **Deploy Verification**: Verifies deployment status
5. **Health Check**: Waits 3 minutes and performs health check

### `main` Branch (Push)

When code is merged to `main`, the following pipeline runs:

1. **Release & Changelog**: Generates release PR with changelog (using release-please)
2. **Build & Tag**: Builds Docker image and pushes to ECR with tags:
   - `{version}` (e.g., `v1.2.3`)
   - `{version}-stg`
   - `{version}-prd`
3. **Deploy to Staging**: Deploys to staging environment
4. **Staging Verification**: Verifies deployment status
5. **Staging Health Check**: Performs health check
6. **Staging Validation**: Validates error rate, latency, and APM metrics (Datadog)
7. **Rollback Mechanism**: Documents rollback capability (if validation fails)
8. **Manual Approval**: Requires approval for production deployment (PCI compliance)
9. **Deploy to Production**: Deploys to production environment
10. **Production Verification**: Verifies deployment status
11. **Production Health Check**: Performs health check
12. **Rollback Mechanism**: Documents rollback capability (if health check fails)

## Usage

### In Your Service Repository

Create three workflow files in `.github/workflows/`:

#### `pull-request-events.yml`

```yaml
name: "Pull Request Events Workflow"

on:
  pull_request:
    branches: [develop]
    types: [opened, edited, reopened, synchronize, ready_for_review]

permissions:
  pull-requests: write
  id-token: write
  contents: read

jobs:
  pull-request-events:
    uses: gocloudLa/poc-workflows-shared/.github/workflows/shared-pull-request-events.yml@main
    with:
      language: php  # or python, javascript, go
    secrets: inherit
```

#### `develop-push-events.yml`

```yaml
name: "Develop Push Events Workflow"

on:
  push:
    branches: [develop]

permissions:
  pull-requests: write
  id-token: write
  contents: read

jobs:
  develop-push-events:
    uses: gocloudLa/poc-workflows-shared/.github/workflows/shared-develop-push-events.yml@main
    with:
      ecr_repository_url: ${{ secrets.ECR_REPOSITORY_URL }}
      aws_role_arn_shared: ${{ secrets.AWS_ROLE_ARN_SHARED }}
      aws_role_arn_dev: ${{ secrets.AWS_ROLE_ARN_DEV }}
      aws_region: ${{ secrets.AWS_REGION || 'us-east-1' }}
      service_name: your-service-name
    secrets: inherit
```

#### `main-push-events.yml`

```yaml
name: "Main Push Events Workflow"

on:
  push:
    branches: [main]

permissions:
  pull-requests: write
  id-token: write
  contents: write

jobs:
  main-push-events:
    uses: gocloudLa/poc-workflows-shared/.github/workflows/shared-main-push-events.yml@main
    with:
      ecr_repository_url: ${{ secrets.ECR_REPOSITORY_URL }}
      aws_role_arn_shared: ${{ secrets.AWS_ROLE_ARN_SHARED }}
      aws_role_arn_staging: ${{ secrets.AWS_ROLE_ARN_STAGING }}
      aws_role_arn_prod: ${{ secrets.AWS_ROLE_ARN_PROD }}
      aws_region: ${{ secrets.AWS_REGION || 'us-east-1' }}
      service_name: your-service-name
    secrets: inherit
```

## Configuration

### Required Repository Secrets

Configure these secrets in your service repository:

- `ECR_REPOSITORY_URL`: ECR repository URL (e.g., `123456789012.dkr.ecr.us-east-1.amazonaws.com/my-service`)
- `AWS_ROLE_ARN_SHARED`: IAM role ARN for ECR access
- `AWS_ROLE_ARN_DEV`: IAM role ARN for dev environment deployments
- `AWS_ROLE_ARN_STAGING`: IAM role ARN for staging environment deployments
- `AWS_ROLE_ARN_PROD`: IAM role ARN for production environment deployments
- `AWS_REGION`: AWS region (optional, defaults to `us-east-1`)

### GitHub Environments

Configure the following environments in your repository (Settings → Environments):

#### `dev` Environment
- **Protection rules**: None (automatic deployment)
- **Secrets**: Optional environment-specific secrets

#### `staging` Environment
- **Protection rules**: None (automatic deployment)
- **Secrets**: Optional environment-specific secrets

#### `prd` Environment
- **Protection rules**:
  - **Required reviewers**: Add security team or development team (based on PCI/non-PCI)
  - **Deployment branches**: Only `main`
- **Secrets**: Environment-specific secrets for production

### AWS IAM Trust Policy

Each IAM role must have a trust policy allowing GitHub Actions OIDC:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
      }
    }
  }]
}
```

## Bypassing Jobs

All workflows support `skip_*` flags to bypass specific jobs:

```yaml
jobs:
  pull-request-events:
    uses: gocloudLa/poc-workflows-shared/.github/workflows/shared-pull-request-events.yml@main
    with:
      language: php
      skip_linter: true
      skip_coverage_validation: true
    secrets: inherit
```

Available skip flags:
- `skip_conventional_commit`
- `skip_linter`
- `skip_security_scan`
- `skip_unit_tests`
- `skip_coverage_validation`
- `skip_build`
- `skip_deploy`
- `skip_deploy_verification`
- `skip_health_check`
- `skip_staging_validation`
- `skip_release`

## Language Detection

The workflows automatically detect the programming language by checking for:
- **PHP**: `composer.json`
- **JavaScript**: `package.json`
- **Python**: `requirements.txt`, `setup.py`, or `pyproject.toml`
- **Go**: `go.mod`

You can also explicitly specify the language:

```yaml
with:
  language: php  # Overrides auto-detection
```

## Customization

### Extending Workflows

You can add additional jobs to your workflow files:

```yaml
jobs:
  pull-request-events:
    uses: gocloudLa/poc-workflows-shared/.github/workflows/shared-pull-request-events.yml@main
    with:
      language: php
    secrets: inherit

  custom-job:
    needs: pull-request-events
    runs-on: ubuntu-latest
    steps:
      - name: Custom Step
        run: echo "Custom job"
```

### Overriding Defaults

You can override default values:

```yaml
with:
  coverage_threshold: 90  # Default is 80
  conventional_commit_types: 'fix feat chore docs'  # Custom types
  require_scope: true  # Require scope in commits
```

## Observability Integration

### Datadog

The workflows include Datadog integration points:

- **Deployment Events**: Logged when deployments occur
- **Health Checks**: Results logged to Datadog
- **Staging Validation**: Queries Datadog for error rate, latency, and APM metrics

To enable Datadog integration, configure the Datadog API key in your repository secrets and update the workflow steps to use the Datadog API.

## Security Features

### GitHub Advanced Security

- **CodeQL**: Static analysis for JavaScript, Python, and Go
- **Dependency Review**: Scans dependencies for vulnerabilities
- **Container Scanning**: Trivy scans Docker images for vulnerabilities

### PCI Compliance

- **Manual Approval**: Production deployments require manual approval
- **Audit Logging**: All deployments are logged
- **Secrets Management**: Uses GitHub Secrets and Environments
- **OIDC Authentication**: No static credentials stored

## Versioning

The system uses semantic versioning:

- **develop branch**: Tags with `{commit-sha-short}` and `develop-latest`
- **main branch**: Tags with `{version}`, `{version}-stg`, and `{version}-prd`

Releases are managed by `release-please`, which:
1. Analyzes commits for conventional commit messages
2. Generates a release PR with changelog
3. Creates a GitHub release when the PR is merged

## Rollback Mechanism

The rollback mechanism is documented but simulated. In a real scenario, it would:

- **ArgoCD**: `argocd app rollback <app-name> <revision>`
- **Argo Rollouts**: `kubectl argo rollouts undo <rollout-name>`
- **ECS**: Revert to previous task definition revision
- **Lambda**: Update to previous version/alias

## Branch Protection Rules

Configure branch protection rules in GitHub (Settings → Branches):

### `develop` Branch
- Require pull request reviews
- Require status checks to pass
- Require branches to be up to date
- Allow specified actors to bypass (team leaders)

### `main` Branch
- Require pull request reviews
- Require status checks to pass
- Require branches to be up to date
- Require linear history
- Allow specified actors to bypass (team leaders)

## Example Repositories

See the example repositories for complete implementations:

- [PHP Example](../poc-workflows-php-repo)
- [Node.js Example](../poc-workflows-node-repo)
- [Python Example](../poc-workflows-python-repo)
- [Go Example](../poc-workflows-go-repo)

## Troubleshooting

### Workflow Fails on Language Detection

If language detection fails, explicitly specify the language:

```yaml
with:
  language: php  # Explicitly set language
```

### Build Fails

Ensure your `Dockerfile` is in the repository root and is valid.

### Deployment Fails

Check:
1. AWS IAM roles are correctly configured
2. OIDC trust policy is set up correctly
3. GitHub environment secrets are configured
4. ECR repository exists and is accessible

### Security Scan Fails

Security scans may find issues. Review the GitHub Security tab for details. To bypass (not recommended):

```yaml
with:
  skip_security_scan: true
```

## Contributing

When adding new workflows or modifying existing ones:

1. Ensure all workflows have `skip_execution` support
2. Add proper error handling and logging
3. Update this README with new features
4. Test with example repositories

## License

[Add your license here]

