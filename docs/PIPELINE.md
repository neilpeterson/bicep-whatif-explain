# CI/CD Pipeline Integration

Integration guide for using `whatif-explain` as an automated deployment gate in CI/CD pipelines.

## Overview

`whatif-explain` provides **platform auto-detection** for seamless CI/CD integration:

- ✅ **Automatic CI mode** - Detects GitHub Actions or Azure DevOps and enables safety gates
- ✅ **Zero configuration** - Extracts PR metadata from environment automatically
- ✅ **Automatic PR comments** - Posts detailed analysis without manual commands
- ✅ **Exit code gating** - Blocks unsafe deployments automatically (exit code 1)

### What Gets Auto-Detected

**GitHub Actions:**
- CI mode automatically enabled
- PR title and description from event file
- Git diff reference from PR base branch
- PR comments posted when `GITHUB_TOKEN` available

**Azure DevOps:**
- CI mode automatically enabled
- PR ID from environment variables
- Git diff reference from target branch
- PR comments posted when `SYSTEM_ACCESSTOKEN` available

## GitHub Actions

### Minimal Setup (Recommended)

Complete PR review workflow in ~50 lines:

```yaml
name: PR Review - Bicep What-If Analysis

on:
  pull_request:
    branches: [main]
    paths:
      - 'bicep/**'

permissions:
  id-token: write
  contents: read
  pull-requests: write

env:
  BICEP_TEMPLATE: bicep/main.bicep
  BICEP_PARAMS: bicep/main.bicepparam

jobs:
  whatif-review:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - run: pip install whatif-explain[anthropic]

      - env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          az deployment group what-if \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP }} \
            --template-file ${{ env.BICEP_TEMPLATE }} \
            --parameters ${{ env.BICEP_PARAMS }} \
            --exclude-change-types NoChange Ignore \
            | whatif-explain
```

**That's it!** Everything is automatic:
- CI mode detected
- PR metadata extracted
- Diff reference set to `origin/main`
- PR comment posted
- Deployment blocked if high risk

### Required Secrets

Add these to **Settings → Secrets and variables → Actions:**

| Secret | Description | How to get |
|--------|-------------|------------|
| `AZURE_CLIENT_ID` | App registration ID | Azure AD app registration |
| `AZURE_TENANT_ID` | Azure AD tenant ID | `az account show` |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID | `az account show` |
| `ANTHROPIC_API_KEY` | Anthropic API key | https://console.anthropic.com/ |

**See [GITHUB_ACTIONS_SETUP.md](./GITHUB_ACTIONS_SETUP.md) for complete Azure authentication setup guide.**

### Adjusting Risk Thresholds

By default, deployments are blocked only on **high** risk. Adjust by adding flags:

```yaml
# More strict - block on medium risk
| whatif-explain \
  --drift-threshold medium \
  --intent-threshold medium \
  --operations-threshold medium

# Very strict - block on any risk
| whatif-explain \
  --drift-threshold low \
  --intent-threshold low \
  --operations-threshold low
```

### Using Different Providers

**Azure OpenAI:**
```yaml
- run: pip install whatif-explain[azure]

- env:
    AZURE_OPENAI_API_KEY: ${{ secrets.AZURE_OPENAI_KEY }}
    AZURE_OPENAI_ENDPOINT: ${{ vars.AZURE_OPENAI_ENDPOINT }}
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    az deployment group what-if ... | whatif-explain --provider azure-openai
```

**Ollama (self-hosted):**
```yaml
- run: pip install whatif-explain[ollama]

- env:
    OLLAMA_HOST: http://localhost:11434
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    az deployment group what-if ... | whatif-explain --provider ollama
```

## Azure DevOps Pipelines

### Minimal Setup (Recommended)

Complete PR review pipeline:

```yaml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - bicep/*

pr:
  branches:
    include:
      - main
  paths:
    include:
      - bicep/*

pool:
  vmImage: ubuntu-latest

variables:
  BICEP_TEMPLATE: bicep/main.bicep
  BICEP_PARAMS: bicep/main.bicepparam

stages:
  - stage: WhatIfReview
    displayName: 'What-If Review'
    jobs:
      - job: Review
        displayName: 'AI Safety Review'
        steps:
          - checkout: self
            fetchDepth: 0

          - task: AzureCLI@2
            displayName: 'What-If Analysis & AI Review'
            inputs:
              azureSubscription: 'my-service-connection'
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                pip install whatif-explain[anthropic]

                az deployment group what-if \
                  --resource-group $(RESOURCE_GROUP) \
                  --template-file $(BICEP_TEMPLATE) \
                  --parameters $(BICEP_PARAMS) \
                  --exclude-change-types NoChange Ignore \
                  | whatif-explain
            env:
              ANTHROPIC_API_KEY: $(ANTHROPIC_API_KEY)
              SYSTEM_ACCESSTOKEN: $(System.AccessToken)
```

**Auto-detection includes:**
- CI mode enabled
- PR ID from `SYSTEM_PULLREQUEST_PULLREQUESTID`
- Diff reference from `SYSTEM_PULLREQUEST_TARGETBRANCH`
- PR comments posted when `SYSTEM_ACCESSTOKEN` available

### Required Variables

Add these in **Pipelines → Library → Variable groups:**

| Variable | Type | Description |
|----------|------|-------------|
| `ANTHROPIC_API_KEY` | Secret | From https://console.anthropic.com/ |
| `RESOURCE_GROUP` | Plain | Your Azure resource group name |

**Important:** Set `SYSTEM_ACCESSTOKEN` in pipeline YAML as shown above to enable PR comments.

### Adjusting Risk Thresholds

```yaml
| whatif-explain \
  --drift-threshold medium \
  --intent-threshold medium \
  --operations-threshold medium
```

## Risk Assessment System

### Three Independent Risk Buckets

Each deployment is evaluated across three risk categories:

1. **Infrastructure Drift**
   - Detects changes in What-If output not present in code diff
   - Catches out-of-band modifications (manual portal changes)
   - **High risk:** Security settings, stateful resources
   - **Medium risk:** Multiple resources drifting
   - **Low risk:** Tags, display names

2. **PR Intent Alignment**
   - Compares What-If changes to PR title/description
   - Catches unintended side effects
   - **High risk:** Destructive changes not mentioned
   - **Medium risk:** Changes misaligned with PR purpose
   - **Low risk:** Minor scope differences

3. **Risky Operations**
   - Evaluates inherent danger of Azure operations
   - Independent of code/PR context
   - **High risk:** Deletions, security changes, public access
   - **Medium risk:** Behavioral changes, new endpoints
   - **Low risk:** New resources, monitoring, tags

### Risk Thresholds

Each bucket has an independent threshold (low, medium, high):

```bash
# Example: Strict on drift, lenient on operations
--drift-threshold low \
--intent-threshold high \
--operations-threshold high
```

**Deployment fails if ANY bucket exceeds its threshold.**

## Example PR Comment

When a PR is created, you'll see:

```markdown
## What-If Deployment Review

### Risk Assessment

| Risk Bucket | Risk Level | Key Concerns |
|-------------|------------|--------------|
| Infrastructure Drift | Low | All changes present in code |
| PR Intent Alignment | Low | Changes match PR description |
| Risky Operations | Medium | Creates new public endpoint |

### Resource Changes

| # | Resource | Type | Action | Risk | Summary |
|---|----------|------|--------|------|---------|
| 1 | app-service | Microsoft.Web/sites | Create | Medium | New App Service with public endpoint |
| 2 | app-plan | Microsoft.Web/serverfarms | Create | Low | Consumption plan for App Service |

**Summary:** This deployment creates new App Service resources as described in PR.

### Verdict: ✅ SAFE

**Overall Risk Level:** Medium
**Highest Risk Bucket:** Operations
**Reasoning:** New public endpoint is documented in PR and includes planned firewall rules.

---
*Generated by whatif-explain*
```

## Troubleshooting

### GitHub Actions: No PR comment posted

**Cause:** Missing `pull-requests: write` permission

**Fix:**
```yaml
permissions:
  pull-requests: write
```

### Azure DevOps: No PR comment posted

**Cause:** Missing `SYSTEM_ACCESSTOKEN` environment variable

**Fix:**
```yaml
env:
  SYSTEM_ACCESSTOKEN: $(System.AccessToken)
```

### "CI mode not detected"

**Cause:** Running outside GitHub Actions or Azure DevOps

**Fix:** Manually enable CI mode:
```bash
| whatif-explain --ci
```

### High risk detected unexpectedly

**Cause:** Infrastructure drift (deployed resources don't match code)

**Fix:**
1. Check drift explanation in PR comment
2. Update code to match infrastructure, OR
3. Deploy from main branch to sync infrastructure

## Advanced Configuration

### Multi-Environment Setup

Use different thresholds per environment:

**Development:**
```yaml
| whatif-explain \
  --drift-threshold low \      # Catch all drift
  --intent-threshold medium \
  --operations-threshold medium
```

**Production:**
```yaml
| whatif-explain \
  --drift-threshold high \     # Only block critical drift
  --intent-threshold high \
  --operations-threshold high
```

### Manual PR Metadata (Optional)

If auto-detection fails, manually provide PR details:

```bash
| whatif-explain \
  --pr-title "Add monitoring resources" \
  --pr-description "This PR adds Application Insights diagnostics"
```

### Specify Bicep Source Directory

Include Bicep source files for better analysis:

```bash
| whatif-explain --bicep-dir bicep/modules/
```

## Other CI Platforms

For platforms without built-in auto-detection (GitLab, Jenkins, etc.), manually enable CI mode:

### GitLab CI

```yaml
bicep_review:
  stage: review
  script:
    - pip install whatif-explain[anthropic]
    - |
      az deployment group what-if ... | whatif-explain \
        --ci \
        --diff-ref origin/$CI_MERGE_REQUEST_TARGET_BRANCH_NAME \
        --pr-title "$CI_MERGE_REQUEST_TITLE" \
        --pr-description "$CI_MERGE_REQUEST_DESCRIPTION"
  only:
    - merge_requests
  variables:
    ANTHROPIC_API_KEY: $ANTHROPIC_API_KEY
```

### Jenkins

```groovy
stage('What-If Review') {
  steps {
    sh '''
      pip install whatif-explain[anthropic]
      az deployment group what-if ... | whatif-explain \
        --ci \
        --diff-ref origin/${CHANGE_TARGET} \
        --pr-title "${CHANGE_TITLE}"
    '''
  }
  environment {
    ANTHROPIC_API_KEY = credentials('anthropic-api-key')
  }
}
```

## Additional Resources

- [GitHub Actions Setup Guide](./GITHUB_ACTIONS_SETUP.md) - Complete setup with Azure authentication
- [Workflow Comparison](./WORKFLOW_COMPARISON.md) - Before/after examples
- [Platform Auto-Detection](./PLATFORM_AUTO_DETECTION_PLAN.md) - How auto-detection works
- [Main Documentation](../README.md) - Full feature reference
