# GitHub Actions Deployment Setup

This guide walks you through setting up GitHub Actions to deploy Bicep templates to Azure using OIDC authentication with GitHub Environments.

## Overview

The workflow uses:
- **OIDC (OpenID Connect)**: Passwordless authentication to Azure
- **GitHub Environment**: Stores secrets and variables for your Azure subscription
- **Federated Credentials**: Allows GitHub to authenticate without storing passwords

## Prerequisites

- Azure subscription with permissions to create app registrations
- GitHub repository with the Bicep workflow
- Resource group already created in Azure
- Azure CLI installed locally

## Setup Steps

### Step 1: Create Azure AD App Registration

Run these commands in PowerShell to create the app registration:

```powershell
# Create the Azure AD application
$APP_ID = az ad app create --display-name "github-actions-bicep-deploy" --query appId -o tsv

# Create service principal
az ad sp create --id $APP_ID

# Get tenant and subscription IDs
$TENANT_ID = az account show --query tenantId -o tsv
$SUBSCRIPTION_ID = az account show --query id -o tsv

# Display the values (you'll need these for GitHub)
Write-Host "============================================"
Write-Host "Save these values for GitHub Environment:"
Write-Host "============================================"
Write-Host "AZURE_CLIENT_ID: $APP_ID"
Write-Host "AZURE_TENANT_ID: $TENANT_ID"
Write-Host "AZURE_SUBSCRIPTION_ID: $SUBSCRIPTION_ID"
Write-Host "============================================"
```

**Keep this PowerShell window open** - you'll need `$APP_ID` for the next steps.

### Step 2: Create GitHub Environment and Add Secrets

1. Go to your GitHub repository: **Settings → Environments**

2. Click **New environment** and name it: `azure-personal-subscription` (or your preferred name)

3. Click **Add environment**

4. In the environment, add these **Secrets**:

   Click **Add secret** for each:

   | Secret Name | Value |
   |-------------|-------|
   | `AZURE_CLIENT_ID` | The `$APP_ID` from Step 1 |
   | `AZURE_TENANT_ID` | The `$TENANT_ID` from Step 1 |
   | `AZURE_SUBSCRIPTION_ID` | The `$SUBSCRIPTION_ID` from Step 1 |

5. In the same environment, add this **Variable**:

   Click **Add variable**:

   | Variable Name | Value |
   |---------------|-------|
   | `AZURE_RESOURCE_GROUP` | Your resource group name (e.g., `rg-apim-nepeters-vs`) |

### Step 3: Create Federated Credential

**Important:** The federated credential subject must match your GitHub environment name.

#### Option A: Using Azure Portal (Recommended)

1. Go to **Azure Portal → Azure Active Directory → App registrations**

2. Find and click **github-actions-bicep-deploy**

3. Click **Certificates & secrets** → **Federated credentials** tab

4. Click **+ Add credential**

5. Fill in the form:
   - **Federated credential scenario:** Other issuer
   - **Issuer:** `https://token.actions.githubusercontent.com`
   - **Subject identifier:** `repo:YOUR-GITHUB-USERNAME/bicep-whatif-explain:environment:azure-personal-subscription`
     - Replace `YOUR-GITHUB-USERNAME` with your actual GitHub username
     - Replace `azure-personal-subscription` if you used a different environment name
   - **Name:** `github-environment-azure-personal-subscription`
   - **Audience:** `api://AzureADTokenExchange`

6. Click **Add**

#### Option B: Using PowerShell

If you prefer using the command line (in the same PowerShell window from Step 1):

```powershell
# Replace with your actual GitHub username
$GITHUB_USERNAME = "neilpeterson"

# Replace with your environment name if different
$ENVIRONMENT_NAME = "azure-personal-subscription"

# Build the federated credential
$envCred = @{
  name = "github-environment-$ENVIRONMENT_NAME"
  issuer = "https://token.actions.githubusercontent.com"
  subject = "repo:$GITHUB_USERNAME/bicep-whatif-explain:environment:$ENVIRONMENT_NAME"
  audiences = @("api://AzureADTokenExchange")
} | ConvertTo-Json -Compress

# Create the credential
az ad app federated-credential create --id $APP_ID --parameters $envCred
```

#### Verify the Credential

```powershell
# List federated credentials to verify
az ad app federated-credential list --id $APP_ID --query "[].{name:name, subject:subject}" -o table
```

**Expected subject:** `repo:YOUR-USERNAME/bicep-whatif-explain:environment:azure-personal-subscription`

### Step 4: Assign Azure Permissions

Grant the app registration permissions to deploy to your resource group:

```powershell
# Set your resource group name (same as in GitHub environment variable)
$RESOURCE_GROUP = "rg-apim-nepeters-vs"

# Assign Contributor role
az role assignment create `
  --assignee $APP_ID `
  --role Contributor `
  --scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP

Write-Host "Permissions assigned successfully!"
```

### Step 5: Update Workflow to Use Environment

Ensure your workflow file (`.github/workflows/deploy-bicep.yml`) references the environment:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: azure-personal-subscription  # Must match your environment name

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      # ... rest of workflow
```

### Step 6: Test the Deployment

1. Go to **Actions** tab in your GitHub repository

2. Select **Deploy Bicep Template** workflow

3. Click **Run workflow** → **Run workflow**

4. The workflow should authenticate successfully and deploy your Bicep template

## Quick Reference

### What You Created

| Component | Purpose |
|-----------|---------|
| Azure AD App Registration | Identity that GitHub uses to authenticate |
| Service Principal | Enables the app to access Azure resources |
| Federated Credential | Passwordless trust between GitHub and Azure |
| GitHub Environment | Stores secrets and variables for Azure |
| Role Assignment | Grants deployment permissions to resource group |

### Subject Identifier Format

When using GitHub Environments, the subject format is:
```
repo:GITHUB_USERNAME/REPO_NAME:environment:ENVIRONMENT_NAME
```

Example:
```
repo:neilpeterson/bicep-whatif-explain:environment:azure-personal-subscription
```

**Common mistakes:**
- ❌ Using email instead of username: `repo:user@example.com/...`
- ❌ Missing environment part: `repo:user/repo:ref:refs/heads/main`
- ❌ Wrong environment name: Must match exactly what's in your workflow

## Troubleshooting

### Error: "Login failed - client-id and tenant-id are not supplied"

**Cause:** Secrets not found in the environment

**Fix:**
1. Check environment name in workflow matches environment in GitHub settings
2. Verify secrets exist in the environment (not repository secrets)
3. Check secret names are exactly: `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`

### Error: "No matching federated identity record found"

**Cause:** Subject identifier doesn't match what GitHub is sending

**Fix:**
1. Go to Azure Portal → App registration → Certificates & secrets → Federated credentials
2. Check the **Subject identifier** field
3. It should show: `repo:YOUR-USERNAME/bicep-whatif-explain:environment:ENVIRONMENT-NAME`
4. If incorrect, delete it and recreate with correct subject (see Step 3)

### Error: "The client does not have authorization to perform action"

**Cause:** App registration doesn't have permissions on the resource group

**Fix:**
```powershell
# Verify role assignment exists
az role assignment list --assignee $APP_ID -o table

# If missing, create it (use your resource group name)
az role assignment create `
  --assignee $APP_ID `
  --role Contributor `
  --scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/YOUR-RESOURCE-GROUP
```

### How to Verify Everything is Configured Correctly

Run these commands to check your setup:

```powershell
# 1. Verify app registration exists
az ad app show --id $APP_ID --query "{displayName:displayName, appId:appId}"

# 2. Verify service principal exists
az ad sp show --id $APP_ID --query "{displayName:displayName}"

# 3. Verify federated credentials
az ad app federated-credential list --id $APP_ID --query "[].{name:name, subject:subject}" -o table

# 4. Verify role assignment
az role assignment list --assignee $APP_ID -o table
```

## Optional: Add Pull Request Support

To run What-If analysis on pull requests (recommended for safety reviews), add an additional federated credential:

**Subject identifier:** `repo:YOUR-USERNAME/bicep-whatif-explain:pull_request`

```powershell
# Add PR support federated credential
$prCred = @{
  name = "github-pr-support"
  issuer = "https://token.actions.githubusercontent.com"
  subject = "repo:$GITHUB_USERNAME/bicep-whatif-explain:pull_request"
  audiences = @("api://AzureADTokenExchange")
} | ConvertTo-Json -Compress

az ad app federated-credential create --id $APP_ID --parameters $prCred
```

Then update your workflow to run on pull requests:

```yaml
on:
  pull_request:
    branches: [main]
    paths:
      - 'bicep-sample/**'
  push:
    branches: [main]

jobs:
  whatif-review:
    runs-on: ubuntu-latest
    environment: azure-personal-subscription
    permissions:
      contents: read
      pull-requests: write
    steps:
      # ... Azure login and What-If steps ...

      - name: AI Safety Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          cat whatif-output.txt | whatif-explain \
            --ci \
            --diff-ref origin/main \
            --drift-threshold high \
            --intent-threshold high \
            --operations-threshold high \
            --pr-title "${{ github.event.pull_request.title }}" \
            --pr-description "${{ github.event.pull_request.body }}" \
            --post-comment

  deploy:
    runs-on: ubuntu-latest
    needs: whatif-review
    if: github.ref == 'refs/heads/main'
    environment: azure-personal-subscription
    steps:
      # ... actual deployment steps ...
```

This setup ensures:
- Every PR gets AI-powered safety review
- PR intent is compared against actual changes
- Deployment only happens on main branch after review passes

## Next Steps

Once your deployment workflow is working:
- Add What-If analysis with `whatif-explain` (see below)
- Add deployment protection rules to the environment (require approvals)
- Configure branch protection on main branch

### Integrate whatif-explain for AI-Powered Safety Gates

Add AI-powered deployment analysis to your workflow:

#### 1. Add Anthropic API Key to Environment

Go to **Settings → Environments → [your-environment]**:

Add this secret:
- **Secret Name:** `ANTHROPIC_API_KEY`
- **Value:** Your Anthropic API key from https://console.anthropic.com/

#### 2. Update Workflow with What-If Analysis

Add this step before deployment:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: azure-personal-subscription
    permissions:
      contents: read
      pull-requests: write  # Required for PR comments

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Required for git diff analysis

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Install whatif-explain
        run: pip install whatif-explain[anthropic]

      - name: Run What-If Analysis
        run: |
          az deployment group what-if \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP }} \
            --template-file bicep-sample/main.bicep \
            --parameters bicep-sample/parameters.bicepparam \
            --exclude-change-types NoChange Ignore \
            > whatif-output.txt

      - name: AI Safety Review & Deployment Gate
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          cat whatif-output.txt | whatif-explain \
            --ci \
            --diff-ref origin/main \
            --drift-threshold high \
            --intent-threshold high \
            --operations-threshold high \
            --post-comment \
            --format markdown

      - name: Deploy to Azure (only if safe)
        if: success() && github.ref == 'refs/heads/main'
        run: |
          az deployment group create \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP }} \
            --template-file bicep-sample/main.bicep \
            --parameters bicep-sample/parameters.bicepparam
```

#### 3. Risk Bucket Configuration

The three independent risk thresholds control deployment safety:

**Production (Recommended):**
```yaml
--drift-threshold high \       # Block on infrastructure drift
--intent-threshold high \      # Block on intent misalignment
--operations-threshold high    # Block on dangerous operations
```

**Development/Staging:**
```yaml
--drift-threshold medium \
--intent-threshold medium \
--operations-threshold medium
```

**Strict (catch any drift):**
```yaml
--drift-threshold low \        # Very sensitive to drift
--intent-threshold high \
--operations-threshold high
```

#### 4. Expected PR Comment Output

When a PR is created, `whatif-explain` will post a comment like:

```markdown
## What-If Deployment Review

### Risk Assessment

| Risk Bucket | Risk Level | Key Concerns |
|-------------|------------|--------------|
| Infrastructure Drift | Low | None |
| PR Intent Alignment | Low | None |
| Risky Operations | Low | None |

### Resource Changes

| # | Resource | Type | Action | Risk | Summary |
|---|----------|------|--------|------|---------|
| 1 | applicationinsights | APIM Diagnostic | Create | Low | Configures App Insights logging... |

**Summary:** This deployment adds monitoring and diagnostic resources.

### Verdict: ✅ SAFE

**Overall Risk Level:** Low
**Reasoning:** All changes are additive with no modifications to existing infrastructure.

---
*Generated by whatif-explain*
```

The workflow will:
- ✅ **Pass** if all risk buckets are below their thresholds → Deployment proceeds
- ❌ **Fail** if any bucket exceeds threshold → Deployment blocked, PR shows which bucket failed

## Additional Resources

- [GitHub OIDC Documentation](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Azure Workload Identity Federation](https://learn.microsoft.com/entra/workload-id/workload-identity-federation)
- [Azure Login Action](https://github.com/Azure/login)
