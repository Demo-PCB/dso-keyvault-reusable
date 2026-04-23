# Azure KeyVault Reusable Workflows

Enterprise-grade GitHub Actions reusable workflows for managing Azure KeyVault deployments with CI/CD automation, security scanning, and artifact management.

## Overview

This repository provides reusable workflows for:
- **CI**: Validate, package, and publish KeyVault changes to Artifactory
- **CD**: Deploy configuration to Azure KeyVault with dry-run preview
- **Rollback**: Restore previous configuration versions

### Key Features

- ✅ **OIDC Authentication**: Secure Azure authentication using Managed Identity (no static credentials)
- ✅ **Dry Run Preview**: Preview changes before import using AZ CLI
- ✅ **Artifact Management**: Version-controlled artifacts stored in JFrog Artifactory
- ✅ **Environment Promotion**: DEV → UAT → PRD deployment chain with manual gates
- ✅ **Production Approval**: Manual approval required for production deployments
- ✅ **Git Tagging**: Automatic version tagging in UAT and release tagging in PRD
- ✅ **Rollback Support**: Quick rollback to any previous version
- ✅ **Security Scanning**: Secret scanning with Gitleaks
- ✅ **Audit Trail**: Complete deployment history with change order tracking

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                          CI Workflow                            │
├─────────────────────────────────────────────────────────────────┤
│  prepare → validate → security-scan → package → publish-summary │
│                              ↓                                  │
│                     Artifactory (versioned zip)                 │
└─────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                         CD Workflows                            │
├─────────────────────────────────────────────────────────────────┤
│  DEV: prepare → download → dry-run → deploy → post-deploy       │
│                              ↓                                  │
│  UAT: prepare → download → dry-run → deploy → create-tag        │
│                              ↓                                  │
│  PRD: prepare → approval → download → dry-run → deploy → release│
└─────────────────────────────────────────────────────────────────┘
```

## Quick Start

### 1. Repository Structure

Your consuming repository should have:

```
your-repo/
├── .github/
│   └── workflows/
│       ├── ci.yml              # CI caller workflow
│       ├── cd-dev.yml          # CD DEV caller workflow
│       ├── cd-uat.yml          # CD UAT caller workflow
│       ├── cd-prd.yml          # CD PRD caller workflow
│       ├── rollback.yml        # Rollback caller workflow
│       └── rollback-prd.yml    # PRD rollback caller workflow
├── data.json                   # Configuration key-value pairs
└── config.yml                  # Version and environment config
```

### 2. Configuration Files

**data.json** - Configuration to import:
```json
{
  "key1": "value1",
  "key2": "value2",
  "nested:key": "nested-value"
}
```

**config.yml** - Version and environment settings:
```yaml
version: 1.0.0

dev:
  keyvault-name: myapp-keyvault-dev
  resource-group: rg-myapp-dev

uat:
  keyvault-name: myapp-keyvault-uat
  resource-group: rg-myapp-uat

prd:
  keyvault-name: myapp-keyvault-prd
  resource-group: rg-myapp-prd
```

### 3. Caller Workflow Examples

Copy workflows from `caller-examples/` to your repository's `.github/workflows/` directory.

**CI Workflow** (`.github/workflows/ci.yml`):
```yaml
name: CI - KeyVault

on:
  push:
    branches: [main, develop]
    paths: ['data.json', 'config.yml']

jobs:
  ci:
    uses: armedinag/dso-keyvault-reusable/.github/workflows/_ci-keyvault.yml@main
    with:
      application: 'my-application'
    secrets:
      DSO_ARTIFACTORY_TOKEN: ${{ secrets.DSO_ARTIFACTORY_TOKEN }}
```

**CD Workflow** (`.github/workflows/cd-dev.yml`):
```yaml
name: CD - KeyVault (DEV)

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy'
        required: true

jobs:
  deploy:
    uses: armedinag/dso-keyvault-reusable/.github/workflows/_cd-keyvault-dev.yml@main
    with:
      application: 'my-application'
      version: ${{ inputs.version }}
    secrets:
      DSO_ARTIFACTORY_TOKEN: ${{ secrets.DSO_ARTIFACTORY_TOKEN }}
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID_DEV }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID_DEV }}
```

## Workflows Reference

### CI Workflow: `_ci-keyvault.yml`

Validates, packages, and publishes KeyVault changes.

#### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `application` | ✅ | - | Application name |
| `component-name` | ❌ | `repo name` | Component name |
| `data-file-path` | ❌ | `data.json` | Path to data file |
| `config-file-path` | ❌ | `config.yml` | Path to config file |
| `FLAG_RUN_SCHEMA_VALIDATION` | ❌ | `true` | Enable schema validation |
| `FLAG_RUN_SECRET_SCAN` | ❌ | `true` | Enable secret scanning |
| `FLAG_PUBLISH_ARTIFACTS` | ❌ | `true` | Enable artifact publishing |

#### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `DSO_ARTIFACTORY_TOKEN` | ✅ | Artifactory token |

#### Outputs

| Output | Description |
|--------|-------------|
| `version` | Computed version (e.g., `1.0.0-42`) |
| `artifact-name` | Name of published artifact |
| `artifact-path` | Artifactory path to artifact |

### CD Workflows: `_cd-keyvault-{env}.yml`

Deploy configuration to Azure KeyVault.

#### Inputs (Common)

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `application` | ✅ | - | Application name |
| `version` | ✅ | - | Version to deploy |
| `component-name` | ❌ | `repo name` | Component name |
| `content-type` | ❌ | `application/json` | Content type for values |
| `label` | ❌ | - | Label for configuration keys |
| `separator` | ❌ | `:` | Separator for nested keys |
| `prefix` | ❌ | - | Prefix for all keys |
| `FLAG_DRY_RUN_ONLY` | ❌ | `false` | Only perform dry run |

#### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `DSO_ARTIFACTORY_TOKEN` | ✅ | Artifactory token |
| `AZURE_CLIENT_ID` | ✅ | Azure AD Application ID |
| `AZURE_TENANT_ID` | ✅ | Azure AD Tenant ID |
| `AZURE_SUBSCRIPTION_ID` | ✅ | Azure Subscription ID |

#### Environment-Specific Features

| Environment | Git Tag | Approval | Additional Inputs |
|-------------|---------|----------|-------------------|
| DEV | ❌ | ❌ | - |
| UAT | ✅ (version tag) | ❌ | `FLAG_SKIP_GIT_TAG` |
| PRD | ✅ (release tag) | ✅ | `change-order`, `artifact-version` |

### Rollback Workflows

#### `_keyvault-rollback.yml` (DEV/UAT)

| Input | Required | Description |
|-------|----------|-------------|
| `application` | ✅ | Application name |
| `version` | ✅ | Version to rollback to |
| `environment` | ✅ | Target environment (dev/uat) |

#### `_keyvault-rollback-prd.yml` (PRD)

| Input | Required | Description |
|-------|----------|-------------|
| `application` | ✅ | Application name |
| `version` | ✅ | Version to rollback to |
| `change-order` | ❌ | Change order for audit |
| `rollback-reason` | ❌ | Reason for rollback |

## Azure OIDC Configuration

### Prerequisites

1. **Azure AD Application** with federated credentials for GitHub Actions
2. **KeyVault Data Owner** role on Azure KeyVault resources
3. **Resource Group Reader** role on resource groups

### Setting Up OIDC

1. Create Azure AD Application:
```bash
az ad app create --display-name "GitHub-keyvault-OIDC"
```

2. Create Service Principal:
```bash
az ad sp create --id <app-id>
```

3. Add Federated Credential:
```bash
az ad app federated-credential create \
  --id <app-id> \
  --parameters '{
    "name": "github-actions",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:<org>/<repo>:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

4. Assign Roles:
```bash
az role assignment create \
  --assignee <app-id> \
  --role "KeyVault Data Owner" \
  --scope /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.keyvaulturation/configurationStores/<name>
```

## Feature Flags Reference

| Flag | Default | Description |
|------|---------|-------------|
| `FLAG_RUN_SCHEMA_VALIDATION` | `true` | Validate JSON/YAML syntax |
| `FLAG_RUN_SECRET_SCAN` | `true` | Run Gitleaks secret scan |
| `FLAG_PUBLISH_ARTIFACTS` | `true` | Publish to Artifactory |
| `FLAG_DRY_RUN_ONLY` | `false` | Preview without import |
| `FLAG_SKIP_GIT_TAG` | `false` | Skip UAT git tagging |
| `FLAG_SKIP_RELEASE_TAG` | `false` | Skip PRD release tagging |
| `FLAG_STRICT_MODE` | `false` | Fail on warnings |

## Secrets Reference

Configure these secrets in your repository or organization:

| Secret | Scope | Description |
|--------|-------|-------------|
| `DSO_ARTIFACTORY_TOKEN` | All | Artifactory API token |
| `AZURE_TENANT_ID` | All | Azure AD Tenant ID |
| `AZURE_CLIENT_ID_DEV` | DEV | Azure App ID for DEV |
| `AZURE_SUBSCRIPTION_ID_DEV` | DEV | Azure Subscription for DEV |
| `AZURE_CLIENT_ID_UAT` | UAT | Azure App ID for UAT |
| `AZURE_SUBSCRIPTION_ID_UAT` | UAT | Azure Subscription for UAT |
| `AZURE_CLIENT_ID_PRD` | PRD | Azure App ID for PRD |
| `AZURE_SUBSCRIPTION_ID_PRD` | PRD | Azure Subscription for PRD |

## Variables Reference

Configure this repository/organization variable:

| Variable | Scope | Description |
|----------|-------|-------------|
| `IBK_ARTIFACTORY_URL` | All | JFrog Artifactory base URL |

## Deployment Flow

### Standard Deployment

1. **CI**: Push changes to `main` or `develop` branch
   - Validates `data.json` and `config.yml`
   - Packages into versioned zip
   - Publishes to Artifactory
   - Outputs version (e.g., `1.0.0-42`)

2. **DEV**: Trigger CD DEV workflow (optional)
   - Input version from CI output
   - Deploys to DEV KeyVault

3. **UAT**: Trigger CD UAT workflow
   - Input version from CI output
   - Deploys to UAT KeyVault
   - Creates git tag (e.g., `1.0.0-42`)

4. **PRD**: Trigger CD PRD workflow
   - Input release version (e.g., `1.0.0`)
   - Requires manual approval
   - Deploys to PRD KeyVault
   - Creates release tag and GitHub Release

### Rollback

1. **DEV/UAT**: Trigger rollback workflow
   - Select environment
   - Input version to rollback to
   - Restores previous configuration

2. **PRD**: Trigger PRD rollback workflow
   - Requires manual approval
   - Input change order for audit
   - Restores previous configuration

## Troubleshooting

### Common Issues

**"No keyvault-name found for environment"**
- Verify `config.yml` has the environment section with `keyvault-name` and `resource-group`

**"Failed to download artifact"**
- Verify the artifact exists in Artifactory
- Check Artifactory credentials are correct

**"OIDC authentication failed"**
- Verify Azure AD Application federated credentials
- Check subject claim matches repository and branch

**"Access denied to KeyVault"**
- Verify service principal has "KeyVault Data Owner" role
- Check scope includes the target KeyVault resource

### Debug Mode

Enable debug logging by setting repository secret:
```
ACTIONS_STEP_DEBUG=true
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make changes following existing patterns
4. Test with a consuming repository
5. Submit a pull request

## License

MIT License - See LICENSE file for details.
