# GitHub Actions Deployment Setup

This guide explains how to configure GitHub Actions to deploy the Bicep template to Azure.

## Prerequisites

- Azure subscription
- GitHub repository with this code
- Resource group already created in Azure
- Existing APIM instance and Application Insights logger (as required by the Bicep template)

## Setup Instructions

This workflow uses **OIDC (OpenID Connect)** authentication - a passwordless, secure method that works with GitHub-hosted runners. This is similar to Azure DevOps service connections.

### Benefits of OIDC
- ✅ No passwords or secrets stored
- ✅ Uses federated identity (like ADO service connections)
- ✅ Works with GitHub-hosted runners
- ✅ Automatically rotates credentials

## OIDC Authentication Setup

### Step 1: Create Azure AD Application

```powershell
# Create the Azure AD application
$APP_NAME = "github-actions-bicep-deploy"
$APP_ID = az ad app create --display-name "github-actions-bicep-deploy" --query appId -o tsv

# Create service principal
az ad sp create --id $APP_ID

# Get tenant and subscription IDs
$TENANT_ID = az account show --query tenantId -o tsv
$SUBSCRIPTION_ID = az account show --query id -o tsv
$SP_OBJECT_ID = az ad sp show --id $APP_ID --query id -o tsv

Write-Host "Client ID (AZURE_CLIENT_ID): $APP_ID"
Write-Host "Tenant ID (AZURE_TENANT_ID): $TENANT_ID"
Write-Host "Subscription ID (AZURE_SUBSCRIPTION_ID): $SUBSCRIPTION_ID"
```

### Step 2: Configure Federated Credentials

Federated credentials allow GitHub Actions to authenticate to Azure without storing passwords. You'll create one or both depending on your workflow needs:

- **Main Branch Credential (Required)**: Allows workflows to deploy when running on the main branch (e.g., after PR merge)
- **Pull Request Credential (Optional)**: Allows workflows to run What-If analysis on PRs for preview and validation

```powershell
# Replace with your GitHub username/organization and repository name
$GITHUB_ORG = "your-github-username"
$REPO_NAME = "bicep-whatif-explain"

# Main Branch Credential (REQUIRED)
# Used by: deploy-bicep.yml workflow when triggered on push to main
# Purpose: Authenticate for actual deployments after code is merged
$federatedCredMain = @{
  name = "github-main-branch"
  issuer = "https://token.actions.githubusercontent.com"
  subject = "repo:$GITHUB_ORG/$REPO_NAME:ref:refs/heads/main"
  audiences = @("api://AzureADTokenExchange")
} | ConvertTo-Json -Compress

az ad app federated-credential create `
  --id $APP_ID `
  --parameters $federatedCredMain

# Pull Request Credential (OPTIONAL)
# Used by: Future What-If preview workflows triggered on pull_request events
# Purpose: Run What-If analysis and post summaries to PRs before merging
# Recommended: Add now even if not using yet - easier than adding later
$federatedCredPR = @{
  name = "github-pull-requests"
  issuer = "https://token.actions.githubusercontent.com"
  subject = "repo:$GITHUB_ORG/$REPO_NAME:pull_request"
  audiences = @("api://AzureADTokenExchange")
} | ConvertTo-Json -Compress

az ad app federated-credential create `
  --id $APP_ID `
  --parameters $federatedCredPR
```

#### Verify Federated Credentials

After creating the credentials, verify the subject identifiers are correct:

```powershell
# List all federated credentials for the app
az ad app federated-credential list --id $APP_ID --query "[].{name:name, subject:subject}" -o table
```

**Expected output:**
```
Name                    Subject
----------------------  ------------------------------------------------------------
github-main-branch      repo:YOUR-USERNAME/bicep-whatif-explain:ref:refs/heads/main
github-pull-requests    repo:YOUR-USERNAME/bicep-whatif-explain:pull_request
```

#### Common Issues and Tips

**Problem: Subject shows incorrect format** (e.g., `repo:email@domain.com//heads/main`)

**Cause:** Variables not set correctly or using wrong shell (bash instead of PowerShell)

**Solutions:**

1. **Verify variables before running commands:**
   ```powershell
   Write-Host "GitHub Org: $GITHUB_ORG"
   Write-Host "Repo Name: $REPO_NAME"
   Write-Host "Full subject: repo:$GITHUB_ORG/$REPO_NAME:ref:refs/heads/main"
   ```

2. **Use hardcoded values to avoid variable expansion issues:**
   ```powershell
   # Replace YOUR-USERNAME with your actual GitHub username
   $params = '{"name":"github-main-branch","issuer":"https://token.actions.githubusercontent.com","subject":"repo:YOUR-USERNAME/bicep-whatif-explain:ref:refs/heads/main","audiences":["api://AzureADTokenExchange"]}'

   az ad app federated-credential create --id $APP_ID --parameters $params
   ```

3. **Manual setup via Azure Portal** (easiest if commands aren't working):
   - Go to Azure Portal → Azure AD → App registrations → `github-actions-bicep-deploy`
   - Click "Certificates & secrets" → "Federated credentials" tab
   - Click "+ Add credential"
   - Select "Other issuer"
   - Fill in:
     - **Issuer:** `https://token.actions.githubusercontent.com`
     - **Subject identifier:** `repo:YOUR-USERNAME/bicep-whatif-explain:ref:refs/heads/main`
     - **Name:** `github-main-branch`
     - **Audience:** `api://AzureADTokenExchange`

**Important:** The subject identifier format must be exact:
- Main branch: `repo:ORG/REPO:ref:refs/heads/main`
- Pull requests: `repo:ORG/REPO:pull_request`
- Use your GitHub **username**, NOT your email address
- Include the full repository name

### Step 3: Assign Azure Permissions

```powershell
# Get your resource group name
$RESOURCE_GROUP = "rg-apim-nepeters-vs"

# Assign Contributor role to the service principal on the resource group
az role assignment create `
  --assignee $APP_ID `
  --role Contributor `
  --scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP
```

### Step 4: Configure GitHub Secrets and Variables

In your GitHub repository, go to **Settings → Secrets and variables → Actions**:

**Secrets** (Settings → Secrets and variables → Actions → Secrets):
- `AZURE_CLIENT_ID`: The application (client) ID from step 1
- `AZURE_TENANT_ID`: Your Azure tenant ID
- `AZURE_SUBSCRIPTION_ID`: Your Azure subscription ID

**Variables** (Settings → Secrets and variables → Actions → Variables):
- `AZURE_RESOURCE_GROUP`: Your target resource group name (e.g., `rg-api-gateway-tme-two`)

## Testing the Deployment

### Manual Trigger

1. Go to **Actions** tab in GitHub
2. Select "Deploy Bicep Template" workflow
3. Click "Run workflow"
4. Select branch and run

### Automatic Trigger

Push changes to the `bicep-sample/` directory on the main branch:

```powershell
git add bicep-sample/
git commit -m "Update Bicep template"
git push origin main
```

## Troubleshooting

### Error: "Login failed - Not all values are present"

**Symptoms:** GitHub Actions fails with "client-id and tenant-id are not supplied"

**Causes:**
- Secrets not configured in GitHub
- Secrets configured in an Environment but workflow doesn't reference it

**Solutions:**
1. Verify secrets exist in GitHub (Settings → Secrets and variables → Actions):
   - `AZURE_CLIENT_ID`
   - `AZURE_TENANT_ID`
   - `AZURE_SUBSCRIPTION_ID`

2. If secrets are in an Environment (e.g., `azure-personal-subscription`), update workflow:
   ```yaml
   jobs:
     deploy:
       environment: azure-personal-subscription  # Add this line
   ```

3. Verify variable exists:
   - `AZURE_RESOURCE_GROUP` (in Variables, not Secrets)

### OIDC Error: "No matching federated identity record found"

**Symptoms:** Authentication fails with "AADSTS70021: No matching federated identity record found"

**Causes:**
- Federated credential subject doesn't match the GitHub repository
- Subject identifier has wrong format (common with variable expansion issues)

**Solutions:**
1. **Check subject format in Azure Portal:**
   - Go to App registration → Certificates & secrets → Federated credentials
   - Verify subject shows: `repo:USERNAME/bicep-whatif-explain:ref:refs/heads/main`
   - NOT: `repo:email@domain.com//heads/main` ❌
   - NOT: `repo:USERNAME//:ref:refs/heads/main` ❌

2. **List credentials via CLI to verify:**
   ```powershell
   az ad app federated-credential list --id $APP_ID --query "[].{name:name, subject:subject}" -o table
   ```

3. **Fix incorrect credentials:**
   - Delete the incorrect credential in Azure Portal
   - Recreate manually through the portal UI (see Step 2 tips above)
   - Or use hardcoded JSON string approach to avoid variable issues

### Error: "The client does not have authorization"

- Verify the app registration has Contributor role on the resource group
- Check that the resource group name in GitHub variables matches exactly
- Confirm role assignment: `az role assignment list --assignee $APP_ID -o table`

### Error: "Resource not found" for APIM or App Insights

- Ensure the APIM instance and Application Insights logger exist
- Verify the names in `bicep-sample/tme-lab.bicepparam` match your Azure resources
- Check that resources are in the same resource group specified in parameters

### Storage Account Name Conflict

- Storage account names must be globally unique
- Update `storageAccountName` in `bicep-sample/tme-lab.bicepparam` if needed
- Use lowercase letters and numbers only (no hyphens or underscores)

## Next Steps

Once basic deployment is working, you can:
- Add What-If analysis step before deployment
- Integrate `whatif-explain` for LLM-powered change summaries
- Add approval gates for production deployments
- Configure branch protection rules
