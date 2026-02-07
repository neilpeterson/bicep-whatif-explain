# whatif-explain

> Azure What-If deployment analyzer using LLMs for human-friendly summaries and safety reviews

`whatif-explain` is a Python CLI tool that accepts Azure Bicep/ARM What-If output via stdin, sends it to an LLM for analysis, and renders a human-friendly summary table in the terminal. In CI mode, it acts as an automated deployment gate with risk assessment and PR comments.

## Table of Contents

- [Features](#features)
- [Quick Start](#quick-start)
- [Standard vs CI Mode](#standard-vs-ci-mode)
- [CI Mode: Deployment Safety Gate](#ci-mode-deployment-safety-gate)
- [CLI Reference](#cli-reference)
- [Usage Examples](#usage-examples)
- [PR Intent Analysis Feature](#pr-intent-analysis-feature)
- [CI/CD Integration](#cicd-integration)
- [Providers](#providers)
- [Troubleshooting](#troubleshooting)

## Features

- 📊 **Human-Friendly Summaries** - Colored tables with plain English explanations of infrastructure changes
- 🔒 **Deployment Safety Gates** - Automated risk assessment for CI/CD pipelines
- 🤖 **Multiple LLM Providers** - Anthropic Claude, Azure OpenAI, or local Ollama
- 📝 **Multiple Output Formats** - Table, JSON, or Markdown
- 🚦 **PR Integration** - Post summaries directly to GitHub or Azure DevOps pull requests
- ⚡ **Fast & Lightweight** - Minimal dependencies, works anywhere Python runs

## Quick Start

### Installation

```bash
# Install with Anthropic Claude support (recommended)
pip install whatif-explain[anthropic]

# Or with Azure OpenAI
pip install whatif-explain[azure]

# Or with local Ollama
pip install whatif-explain[ollama]

# Or install all providers
pip install whatif-explain[all]
```

### Basic Usage

```powershell
# Set your API key
$env:ANTHROPIC_API_KEY = "your-api-key"

# Pipe What-If output to whatif-explain
az deployment group what-if `
  --resource-group my-rg `
  --template-file main.bicep `
  --parameters params.json | whatif-explain
```

### Example Output

```
╭──────┬───────────────────────────┬──────────────────────┬────────┬─────────────────────────────────────╮
│ #    │ Resource                  │ Type                 │ Action │ Summary                             │
├──────┼───────────────────────────┼──────────────────────┼────────┼─────────────────────────────────────┤
│ 1    │ applicationinsights       │ APIM Diagnostic      │ Create │ Configures App Insights logging     │
│      │                           │                      │        │ with custom JWT headers and 100%    │
│      │                           │                      │        │ sampling.                           │
├──────┼───────────────────────────┼──────────────────────┼────────┼─────────────────────────────────────┤
│ 2    │ policy                    │ APIM Global Policy   │ Modify │ Updates global inbound policy to    │
│      │                           │                      │        │ validate Front Door header and      │
│      │                           │                      │        │ include JWT parsing fragment.       │
├──────┼───────────────────────────┼──────────────────────┼────────┼─────────────────────────────────────┤
│ 3    │ sce-jwt-parsing-logging   │ APIM Policy Fragment │ Create │ Reusable fragment that parses       │
│      │                           │                      │        │ Bearer tokens and extracts claims   │
│      │                           │                      │        │ into logging headers.               │
╰──────┴───────────────────────────┴──────────────────────┴────────┴─────────────────────────────────────╯

Summary: This deployment creates JWT authentication policies, updates diagnostic
logging, and enhances API security with Front Door validation.
```

## Standard vs CI Mode

`whatif-explain` operates in two distinct modes:

### Standard Mode (Default)

**Purpose:** Human-readable summary of infrastructure changes

**Usage:**
```bash
az deployment group what-if ... | whatif-explain
```

**Features:**
- ✅ Summarizes What-If output in plain English
- ✅ Colored table or JSON/Markdown output
- ✅ No risk assessment
- ✅ Always exits with code 0 (success)
- ✅ No git context needed

**Best for:**
- Interactive terminal usage
- Quick understanding of what will change
- Documentation and reporting
- Local development

### CI Mode (--ci flag)

**Purpose:** Automated deployment safety gate for pipelines

**Usage:**
```bash
az deployment group what-if ... | whatif-explain --ci --diff-ref origin/main
```

**Features:**
- ✅ Everything in Standard Mode, plus:
- ✅ Risk assessment for each resource (none/low/medium/high/critical)
- ✅ Git diff analysis to see code changes
- ✅ PR intent analysis to catch scope creep
- ✅ Overall safety verdict
- ✅ Configurable risk threshold
- ✅ Exit code 0 (safe) or 1 (unsafe) for deployment gates
- ✅ Optional PR comment posting

**Best for:**
- CI/CD pipelines
- Automated deployment gates
- Pull request reviews
- Team collaboration and safety

### Quick Comparison

| Feature | Standard Mode | CI Mode |
|---------|---------------|---------|
| Summary table | ✅ | ✅ |
| Risk assessment | ❌ | ✅ |
| Git diff analysis | ❌ | ✅ |
| PR intent validation | ❌ | ✅ |
| Deployment verdict | ❌ | ✅ |
| Blocks unsafe deploys | ❌ | ✅ |
| PR comments | ❌ | ✅ (optional) |
| Exit codes | Always 0 | 0 (safe) or 1 (unsafe) |
| Use case | Local dev | CI/CD |

## CI Mode: Deployment Safety Gate

CI mode (`--ci`) transforms `whatif-explain` from a summary tool into an **automated deployment safety gate** for your CI/CD pipelines. It analyzes infrastructure changes, assesses risk, and can automatically block unsafe deployments.

### How CI Mode Works

1. **Input Collection**
   - Reads Azure What-If output (infrastructure changes)
   - Fetches git diff to see code changes
   - Optionally loads Bicep source files for context
   - Captures PR title and description for intent analysis

2. **Risk Assessment**
   - Each resource change is assigned a risk level: `none`, `low`, `medium`, `high`, or `critical`
   - Risk is based on action type, resource type, and alignment with PR intent
   - Overall deployment risk is the highest individual resource risk

3. **Intent Analysis** ✨ NEW
   - Compares infrastructure changes against stated PR intent
   - Flags changes not mentioned in PR description
   - Elevates risk for destructive changes not explicitly documented
   - Catches scope creep and unintended side effects

4. **Deployment Decision**
   - Compares overall risk against `--risk-threshold`
   - Exit code 0 = safe to deploy (proceed with deployment)
   - Exit code 1 = unsafe (block deployment)
   - Optionally posts detailed analysis to PR as comment

### Risk Levels Explained

| Risk Level | Examples | Typical Actions |
|------------|----------|-----------------|
| **🔴 CRITICAL** | Database deletions, storage account deletions, key vault deletions, RBAC/identity deletions, disabling encryption, broad network security changes | **BLOCK** - Requires manual review |
| **🟠 HIGH** | Production resource deletions, firewall rule changes, authentication/authorization modifications, SKU downgrades on critical services | **BLOCK** (default threshold) |
| **🟡 MEDIUM** | Behavioral changes to existing resources, new public endpoints, policy modifications, scaling config changes | **WARN** - Proceed with caution |
| **🟢 LOW** | Adding new resources, adding tags, diagnostic/monitoring resources, cosmetic changes | **SAFE** - Proceed |
| **⚪ NONE** | NoChange, Ignore actions, no actual modifications | **SAFE** - Proceed |

### Intent Mismatch Detection

When PR metadata is provided (`--pr-title`, `--pr-description`), the AI evaluates whether infrastructure changes align with the stated purpose:

**Risk Elevation Rules:**
- 🔴 **CRITICAL**: Destructive changes (Delete) not mentioned in PR
- 🟠 **HIGH**: Security/auth/network changes not mentioned in PR
- 🟡 **MEDIUM**: Resource modifications not aligned with PR intent
- 🟢 **LOW**: New resources not mentioned but contextually aligned

**Example Scenario:**
```
PR Title: "Add Application Insights logging"
PR Description: "Configuring diagnostic settings to send logs to App Insights"

What-If Shows:
✅ APIM diagnostic settings     → LOW RISK (matches intent)
⚠️ SQL Database deletion        → CRITICAL RISK (not mentioned in PR!)
⚠️ Storage firewall rule change → HIGH RISK (not mentioned in PR!)
```

The AI will flag the database deletion and firewall change as **intent mismatches** and block the deployment.

### CI Mode Output

In CI mode, the output includes risk assessment:

```
╭──────┬──────────────────┬────────────────┬────────┬──────────┬────────────────────────────────╮
│ #    │ Resource         │ Type           │ Action │ Risk     │ Summary                        │
├──────┼──────────────────┼────────────────┼────────┼──────────┼────────────────────────────────┤
│ 1    │ myDatabase       │ SQL Database   │ Delete │ CRITICAL │ ⚠️ Deleting production        │
│      │                  │                │        │          │ database - data loss will      │
│      │                  │                │        │          │ occur! Not mentioned in PR.    │
├──────┼──────────────────┼────────────────┼────────┼──────────┼────────────────────────────────┤
│ 2    │ myStorage        │ Storage Acct   │ Modify │ MEDIUM   │ Changing access tier from      │
│      │                  │                │        │          │ Cool to Hot. Not mentioned     │
│      │                  │                │        │          │ in PR description.             │
╰──────┴──────────────────┴────────────────┴────────┴──────────┴────────────────────────────────╯

🔴 CRITICAL RISK DETECTED
Verdict: UNSAFE - Deployment blocked

Concerns:
• Database deletion 'myDatabase' will cause data loss
• Change not mentioned in PR: "Add logging configuration"

Recommendations:
• Update PR description to document database deletion and justification
• Ensure database backup exists before proceeding
• Consider if this change belongs in a separate PR
```

## CLI Reference

### Basic Flags

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--provider` | `-p` | `anthropic` | LLM provider: `anthropic`, `azure-openai`, `ollama` |
| `--model` | `-m` | Provider default | Override model name |
| `--format` | `-f` | `table` | Output format: `table`, `json`, `markdown` |
| `--verbose` | `-v` | _(disabled)_ | Include property-level change details for modified resources |
| `--no-color` | | _(disabled)_ | Disable colored output (useful for logs) |
| `--version` | | | Print version and exit |
| `--help` | `-h` | | Print help message |

### CI Mode Flags

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--ci` | | _(disabled)_ | **Enable CI mode with risk assessment and deployment gate** |
| `--diff` | `-d` | Auto-detect | Path to git diff file (auto-runs `git diff` if not provided) |
| `--diff-ref` | | `HEAD~1` | Git reference to diff against (e.g., `origin/main`, `HEAD~3`) |
| `--risk-threshold` | | `high` | Fail deployment at this risk level or above: `low`, `medium`, `high`, `critical` |
| `--post-comment` | | _(disabled)_ | Post summary as PR comment (GitHub or Azure DevOps) |
| `--pr-url` | | Auto-detect | PR URL for posting comments (auto-detected from environment if not provided) |
| `--bicep-dir` | | `.` | Path to Bicep source files for additional context |
| `--pr-title` | | Auto-detect | Pull request title for intent analysis (auto-fetched in GitHub Actions) |
| `--pr-description` | | Auto-detect | Pull request description for intent analysis (auto-fetched in GitHub Actions) |

## Environment Variables

### Provider Credentials

**Anthropic:**
```powershell
$env:ANTHROPIC_API_KEY = "sk-ant-..."
```

**Azure OpenAI:**
```powershell
$env:AZURE_OPENAI_ENDPOINT = "https://your-resource.openai.azure.com/"
$env:AZURE_OPENAI_API_KEY = "your-key"
$env:AZURE_OPENAI_DEPLOYMENT = "your-deployment-name"
```

**Ollama:**
```powershell
$env:OLLAMA_HOST = "http://localhost:11434"  # Optional, this is the default
```

### Optional Overrides

```powershell
$env:WHATIF_PROVIDER = "anthropic"  # Override default provider
$env:WHATIF_MODEL = "claude-sonnet-4-20250514"  # Override default model
```

## Usage Examples

### Different Output Formats

```bash
# Table (default, colored)
az deployment group what-if ... | whatif-explain

# JSON (for scripting)
az deployment group what-if ... | whatif-explain --format json

# Markdown (for documentation)
az deployment group what-if ... | whatif-explain --format markdown

# Pipe JSON to jq for filtering
az deployment group what-if ... | whatif-explain -f json | jq '.resources[] | select(.action == "Delete")'
```

### Verbose Mode

```bash
# Show property-level changes for modified resources
az deployment group what-if ... | whatif-explain --verbose
```

### Different Providers

```bash
# Use Azure OpenAI
az deployment group what-if ... | whatif-explain --provider azure-openai

# Use local Ollama
az deployment group what-if ... | whatif-explain --provider ollama --model llama3.1

# Use a different Claude model
az deployment group what-if ... | whatif-explain --model claude-opus-4
```

### CI Mode Examples

#### Basic Deployment Gate

```bash
# Run What-If analysis
az deployment group what-if \
  --resource-group my-rg \
  --template-file main.bicep \
  --parameters params.json > whatif-output.txt

# Analyze with CI mode
cat whatif-output.txt | whatif-explain \
  --ci \
  --diff-ref origin/main \
  --risk-threshold high

# Exit code 0 = safe, exit code 1 = unsafe
if [ $? -eq 0 ]; then
  echo "✅ Safe to deploy - proceeding..."
  az deployment group create \
    --resource-group my-rg \
    --template-file main.bicep \
    --parameters params.json
else
  echo "❌ Deployment blocked due to high risk"
  exit 1
fi
```

#### With PR Comments (GitHub Actions)

```bash
# In GitHub Actions, PR metadata is auto-detected
cat whatif-output.txt | whatif-explain \
  --ci \
  --diff-ref origin/main \
  --bicep-dir ./infrastructure \
  --format markdown \
  --pr-title "${{ steps.pr-details.outputs.pr_title }}" \
  --pr-description "${{ steps.pr-details.outputs.pr_body }}" \
  > analysis.md

# Post comment using gh CLI
gh pr comment ${{ github.event.pull_request.number }} --body-file analysis.md
```

#### Custom Risk Threshold

```bash
# Block on medium risk or higher (more strict)
cat whatif-output.txt | whatif-explain \
  --ci \
  --risk-threshold medium

# Only block on critical risk (more permissive)
cat whatif-output.txt | whatif-explain \
  --ci \
  --risk-threshold critical
```

#### With Bicep Source Context

```bash
# Include Bicep source files for better analysis
cat whatif-output.txt | whatif-explain \
  --ci \
  --diff-ref origin/main \
  --bicep-dir ./bicep \
  --format markdown
```

#### Different Diff References

```bash
# Compare against main branch
cat whatif-output.txt | whatif-explain --ci --diff-ref origin/main

# Compare against specific commit
cat whatif-output.txt | whatif-explain --ci --diff-ref abc123

# Compare against previous commit
cat whatif-output.txt | whatif-explain --ci --diff-ref HEAD~1

# Compare against tag
cat whatif-output.txt | whatif-explain --ci --diff-ref v1.0.0
```

## PR Intent Analysis Feature

The PR intent analysis feature helps catch unintended changes, scope creep, and configuration mistakes by comparing infrastructure changes against the stated purpose of your pull request.

### How It Works

1. **Extract Intent**: Reads PR title and description to understand what the change is supposed to do
2. **Compare Changes**: Analyzes What-If output to see what will actually change
3. **Flag Mismatches**: Identifies changes that don't align with the stated intent
4. **Elevate Risk**: Increases risk level for unmentioned or unexpected changes

### Real-World Examples

#### Example 1: Caught Unintended Deletion

**PR Description:**
```
Title: Update API Management logging configuration
Description: Configuring diagnostic settings to send logs to Application Insights
```

**What-If Shows:**
- ✅ APIM diagnostic settings modified
- ⚠️ SQL Database 'prod-customer-db' deleted

**AI Analysis:**
```
🔴 CRITICAL RISK - Intent Mismatch Detected

The PR claims to only update logging configuration, but the deployment
will DELETE a production database. This is not mentioned anywhere in the
PR description.

Risk: CRITICAL (deployment blocked)
Recommendation: Update PR description to explain database deletion or
revert the unintended change.
```

#### Example 2: Scope Creep Detection

**PR Description:**
```
Title: Add environment tags to storage account
Description: Adding standard Environment and ManagedBy tags for compliance
```

**What-If Shows:**
- ✅ Tags added to storage account
- ⚠️ Storage account access tier changed from Cool to Hot

**AI Analysis:**
```
🟡 MEDIUM RISK - Potential Scope Creep

While tag additions match the PR intent, the access tier change is not
mentioned. This may be unintentional or forgotten documentation.

Risk: MEDIUM (proceed with caution)
Recommendation: Update PR description to document the access tier change
and its impact on costs.
```

#### Example 3: Security Change Not Documented

**PR Description:**
```
Title: Update APIM policy fragments
Description: Refactoring JWT parsing logic into reusable fragments
```

**What-If Shows:**
- ✅ APIM policy fragments created
- ⚠️ Network security group rules modified (port 443 opened to 0.0.0.0/0)

**AI Analysis:**
```
🔴 HIGH RISK - Security Change Not Documented

The PR describes policy refactoring but doesn't mention the NSG rule
change that opens port 443 to the internet. This could be a security
risk or configuration error.

Risk: HIGH (deployment blocked)
Recommendation: Document the network security change in the PR or
investigate if this is an unintended side effect.
```

### Best Practices

To get the most value from intent analysis:

1. **Write Descriptive PR Titles**
   - ❌ "Update Bicep files"
   - ✅ "Add Application Insights logging to API Management"

2. **Document All Significant Changes**
   ```markdown
   ## Changes
   - Configure APIM diagnostic settings for App Insights
   - Add JWT parsing policy fragment
   - Update global policy to validate Front Door headers
   ```

3. **Explain Destructive Operations**
   ```markdown
   ## ⚠️ Destructive Changes
   - Deleting test database 'dev-temp-db' (no longer needed)
   - Backup verified and saved to storage account
   ```

4. **Call Out Security/Network Changes**
   ```markdown
   ## Security Changes
   - Opening port 443 on NSG for Front Door integration
   - Restricting access to Front Door IP ranges only
   ```

## Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success (or safe deployment in CI mode) |
| `1` | Error or unsafe deployment (in CI mode, risk threshold exceeded) |
| `2` | Invalid input (no piped input, empty stdin, malformed What-If output) |

## Providers

### Azure OpenAI

Use your own Azure OpenAI deployment.

```powershell
pip install whatif-explain[azure]
$env:AZURE_OPENAI_ENDPOINT = "https://..."
$env:AZURE_OPENAI_API_KEY = "..."
$env:AZURE_OPENAI_DEPLOYMENT = "..."
```

### Anthropic Claude

Fast, accurate, and easy to set up.

```powershell
pip install whatif-explain[anthropic]
$env:ANTHROPIC_API_KEY = "sk-ant-..."
```

Default model: `claude-sonnet-4-20250514`

### Ollama (Local)

Run LLMs locally without API calls.

```bash
pip install whatif-explain[ollama]
ollama pull llama3.1
ollama serve
```

Default model: `llama3.1`

## CI/CD Integration

### GitHub Actions Example

Complete workflow with PR intent analysis:

```yaml
name: PR Review - Infrastructure Changes

on:
  pull_request:
    branches: [main]
    paths: ['infrastructure/**']

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Needed for git diff

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install whatif-explain
        run: pip install whatif-explain[anthropic]

      - name: Get PR Details
        id: pr
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          echo "title=$(gh pr view ${{ github.event.pull_request.number }} --json title --jq '.title')" >> $GITHUB_OUTPUT
          echo "body<<EOF" >> $GITHUB_OUTPUT
          gh pr view ${{ github.event.pull_request.number }} --json body --jq '.body' >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Run What-If Analysis
        run: |
          az deployment group what-if \
            --resource-group ${{ vars.RESOURCE_GROUP }} \
            --template-file infrastructure/main.bicep \
            --parameters infrastructure/params.json \
            --exclude-change-types NoChange Ignore \
            > whatif-output.txt

      - name: AI Safety Analysis
        id: analysis
        continue-on-error: true
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          cat whatif-output.txt | whatif-explain \
            --ci \
            --diff-ref origin/main \
            --bicep-dir infrastructure/ \
            --pr-title "${{ steps.pr.outputs.title }}" \
            --pr-description "${{ steps.pr.outputs.body }}" \
            --risk-threshold high \
            --format markdown > analysis.md

      - name: Post PR Comment
        if: always()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh pr comment ${{ github.event.pull_request.number }} --body-file analysis.md

      - name: Check Safety Verdict
        if: steps.analysis.outcome == 'failure'
        run: |
          echo "::error::Deployment blocked due to high risk"
          exit 1
```

### Azure DevOps Pipeline Example

```yaml
trigger:
  branches:
    include: [main]
  paths:
    include: ['infrastructure/*']

pr:
  branches:
    include: [main]
  paths:
    include: ['infrastructure/*']

pool:
  vmImage: 'ubuntu-latest'

steps:
  - task: AzureCLI@2
    displayName: 'Run What-If Analysis'
    inputs:
      azureSubscription: 'My-Service-Connection'
      scriptType: 'bash'
      scriptLocation: 'inlineScript'
      inlineScript: |
        az deployment group what-if \
          --resource-group $(RESOURCE_GROUP) \
          --template-file infrastructure/main.bicep \
          --parameters infrastructure/params.json \
          > whatif-output.txt

  - task: UsePythonVersion@0
    inputs:
      versionSpec: '3.11'

  - script: |
      pip install whatif-explain[anthropic]
    displayName: 'Install whatif-explain'

  - script: |
      cat whatif-output.txt | whatif-explain \
        --ci \
        --diff-ref origin/main \
        --bicep-dir infrastructure/ \
        --risk-threshold high \
        --format markdown \
        --post-comment
    displayName: 'AI Safety Analysis'
    env:
      ANTHROPIC_API_KEY: $(ANTHROPIC_API_KEY)
      SYSTEM_ACCESSTOKEN: $(System.AccessToken)
```

See [PIPELINE.md](docs/PIPELINE.md) for complete CI/CD integration guides.

## Troubleshooting

### "No input detected" error

Make sure you're piping What-If output to the command:

```bash
# ❌ Wrong - no piped input
whatif-explain

# ✅ Correct - piped input
az deployment group what-if ... | whatif-explain
```

### "ANTHROPIC_API_KEY environment variable not set"

Set your API key:

```powershell
$env:ANTHROPIC_API_KEY = "sk-ant-..."
```

### "Cannot reach Ollama" error

Make sure Ollama is running:

```bash
ollama serve
```

### LLM returns invalid JSON

The tool attempts to extract JSON from malformed responses. If this fails consistently:
- Try a different model with `--model`
- Check if your What-If output is extremely large (truncated to 100k chars)
- Use a more capable model (e.g., Claude Opus)

## Contributing

Issues and pull requests are welcome! Please see the repository for contribution guidelines.

## License

MIT License - see [LICENSE](LICENSE) for details.

## Links

- Anthropic API: https://console.anthropic.com/
- Azure OpenAI: https://azure.microsoft.com/products/ai-services/openai-service
- Ollama: https://ollama.com/
