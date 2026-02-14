# Azure DevOps Pipeline Examples

This directory contains sample Azure DevOps pipeline YAML files for integrating `bicep-whatif-advisor` into your CI/CD workflow.

## Files

### `bicep-whatif-azure-repos.yml`
**Use this when:** Your code is hosted in **Azure Repos Git** (native Azure DevOps repositories)

**Authentication:** Uses `SYSTEM_ACCESSTOKEN` to post PR comments
- ✅ Automatically available via `$(System.AccessToken)`
- ✅ No external tokens needed
- ⚠️ Requires "Contribute to pull requests" permission for build service

### `bicep-whatif-github-repo.yml`
**Use this when:** Your code is hosted in **GitHub** but you're using Azure DevOps Pipelines

**Authentication:** Uses `GITHUB_TOKEN` to post PR comments
- ⚠️ Requires GitHub Personal Access Token (PAT)
- ⚠️ Must be added as pipeline variable
- ⚠️ Token needs `repo` scope or pull requests write permission

## How to Choose

Check your repository provider in Azure DevOps:

1. Go to your pipeline → Edit
2. Look at the repository source in the UI
3. Choose the appropriate sample:
   - **"Azure Repos Git"** → Use `bicep-whatif-azure-repos.yml`
   - **"GitHub"** → Use `bicep-whatif-github-repo.yml`

## Quick Comparison

| Feature | Azure Repos | GitHub Repo |
|---------|-------------|-------------|
| Repository Provider | `TfsGit` | `GitHub` |
| PR Comment Token | `SYSTEM_ACCESSTOKEN` | `GITHUB_TOKEN` |
| Token Setup | Automatic | Manual (PAT required) |
| Build Service Permissions | Required | Not applicable |

## Error: "404 Not Found" when posting comments

This error occurs when you use the wrong authentication method:

**Symptom:**
```
Warning: Azure DevOps API error: 404 Client Error: Not Found for url: https://dev.azure.com/.../pullRequests/.../threads
```

**Cause:** Using `SYSTEM_ACCESSTOKEN` with a GitHub repository

**Fix:** Switch to `GITHUB_TOKEN` as shown in `bicep-whatif-github-repo.yml`

## Additional Resources

- [CI/CD Integration Guide](../docs/guides/CICD_INTEGRATION.md) - Complete setup instructions
- [Getting Started Guide](../docs/guides/GETTING_STARTED.md) - Installation and basic usage
- [CLI Reference](../docs/guides/CLI_REFERENCE.md) - All command-line options
