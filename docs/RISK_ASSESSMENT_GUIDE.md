# Risk Assessment Guide

A simple explanation of how `whatif-explain` evaluates deployment safety and makes decisions.

## Overview

When running in CI mode, `whatif-explain` acts as a deployment safety gate by evaluating your infrastructure changes across three independent risk categories. Each category gets its own risk level, and you set thresholds to control when deployments should be blocked.

## The Decision Flow

```
Azure What-If Output + Code Diff
           ↓
    AI Analysis (LLM)
           ↓
   Three Risk Assessments
           ↓
    ┌──────────────┬──────────────┬──────────────┐
    │              │              │              │
    │   Drift      │   Intent     │  Operations  │
    │   Bucket     │   Bucket     │   Bucket     │
    │              │              │              │
    │  Risk: Low   │  Risk: Med   │  Risk: High  │
    │              │              │              │
    └──────────────┴──────────────┴──────────────┘
           ↓
    Compare to Thresholds
           ↓
    Deployment Verdict
           ↓
    Exit Code (0 or 1)
```

## Three Risk Buckets

Each deployment is evaluated independently across three categories:

### 1. Infrastructure Drift 🔄

**What it detects:** Changes appearing in What-If output that aren't in your code diff.

**Why it matters:** Indicates someone manually changed infrastructure in Azure (via Portal, CLI, etc.) and those changes will be overwritten by your deployment.

**Examples:**
- **High Risk:** Storage account was manually set to "private" but template will revert to "public"
- **Medium Risk:** Multiple resources have configuration drift
- **Low Risk:** Only tags or display names differ

**Common causes:**
- Manual fixes in production
- Emergency changes not committed to code
- Configuration drift over time

### 2. PR Intent Alignment 🎯

**What it detects:** Misalignment between what the PR says it does and what it actually does.

**Why it matters:** Catches unintended side effects, scope creep, or changes that weren't discussed in the PR.

**Examples:**
- **High Risk:** PR says "add monitoring" but it's also deleting a database
- **Medium Risk:** PR says "update tags" but it's modifying network rules
- **Low Risk:** PR says "add App Service" and it adds related resources not mentioned

**Common causes:**
- Unintended side effects
- Forgotten about previous changes
- Template dependencies creating extra changes

**Note:** This bucket is only evaluated if PR title/description are available (auto-detected or manual).

### 3. Risky Operations ⚠️

**What it detects:** Inherently dangerous Azure operations regardless of context.

**Why it matters:** Some operations are risky by nature (deletions, security changes) and deserve extra scrutiny.

**Examples:**
- **High Risk:** Deleting a database, removing RBAC roles, enabling public access on storage
- **Medium Risk:** Changing firewall rules, modifying encryption settings, new public endpoints
- **Low Risk:** Creating new resources, updating tags, adding monitoring

**Common causes:**
- Intentional but risky changes
- Refactoring infrastructure
- Security configuration updates

## Risk Levels

Each bucket receives one of three risk levels:

| Level | Meaning | Symbol |
|-------|---------|--------|
| **Low** | Safe, routine change | ● Green |
| **Medium** | Needs attention but not critical | ● Yellow |
| **High** | Dangerous, requires review | ● Red |

The AI evaluates each bucket independently and assigns a risk level based on the specific concerns found.

## Thresholds (Your Control)

You set independent thresholds for each bucket to control when deployments are blocked:

```bash
whatif-explain \
  --drift-threshold high \       # Block only on high drift
  --intent-threshold high \      # Block only on high misalignment
  --operations-threshold high    # Block only on high-risk operations
```

### How Thresholds Work

**Deployment is BLOCKED if ANY bucket exceeds its threshold:**

```
Drift: Medium     (threshold: high)   → ✅ Pass (below threshold)
Intent: Low       (threshold: high)   → ✅ Pass (below threshold)
Operations: High  (threshold: high)   → ❌ FAIL (meets threshold)

Result: Deployment BLOCKED (exit code 1)
```

### Threshold Strategies

**Conservative (Recommended for Production):**
```bash
--drift-threshold high \
--intent-threshold high \
--operations-threshold high
```
- Blocks only critical issues
- Allows routine changes
- Good for production

**Moderate (Staging/Pre-Production):**
```bash
--drift-threshold medium \
--intent-threshold high \
--operations-threshold medium
```
- More sensitive to drift
- Catches moderate operational risks
- Good for staging

**Strict (Development/Audit):**
```bash
--drift-threshold low \
--intent-threshold low \
--operations-threshold low
```
- Blocks on any risk
- Maximum safety
- Good for learning or high-security environments

## Exit Codes

The tool uses standard exit codes to communicate results:

| Code | Meaning | When it happens | What to do |
|------|---------|-----------------|------------|
| **0** | Success / Safe | All risk buckets below thresholds | Deploy safely ✅ |
| **1** | Unsafe / Blocked | One or more buckets exceed thresholds | Review PR comment, fix issues ⚠️ |
| **2** | Error / Invalid | Bad input or tool error | Check logs, fix command ❌ |

### Exit Code Examples

**Scenario 1: Safe Deployment**
```bash
$ az deployment group what-if ... | whatif-explain
# Analysis shows:
#   Drift: Low
#   Intent: Low
#   Operations: Medium
#
# Thresholds: all "high"
# Result: Exit code 0 (safe)
```

**Scenario 2: Blocked Deployment**
```bash
$ az deployment group what-if ... | whatif-explain
# Analysis shows:
#   Drift: High (storage account security regression)
#   Intent: High (PR about monitoring, but security changing)
#   Operations: Medium
#
# Thresholds: all "high"
# Result: Exit code 1 (blocked - drift and intent both high)
```

**Scenario 3: Tool Error**
```bash
$ echo "invalid data" | whatif-explain
# No valid What-If output detected
# Result: Exit code 2 (invalid input)
```

## How CI/CD Pipelines Use Exit Codes

### GitHub Actions

```yaml
- run: az deployment group what-if ... | whatif-explain
  # Exit code 1 automatically fails the workflow
  # Blocks PR from being merged if branch protection enabled
```

GitHub Actions treats any non-zero exit code as a failure, so:
- Exit 0 → Workflow succeeds → PR can merge
- Exit 1 → Workflow fails → PR blocked
- Exit 2 → Workflow fails → Check logs

### Azure DevOps

```yaml
- script: az deployment group what-if ... | whatif-explain
  # Exit code 1 automatically fails the pipeline
  # Sets build status to failed
```

Azure DevOps behavior:
- Exit 0 → Build succeeds → Can proceed
- Exit 1 → Build fails → Deployment blocked
- Exit 2 → Build fails → Check logs

## Complete Example

Let's walk through a complete scenario:

### The Setup

You have a PR titled "Add Application Insights monitoring" with this workflow:

```yaml
- env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
  run: |
    az deployment group what-if ... | whatif-explain \
      --drift-threshold high \
      --intent-threshold high \
      --operations-threshold high
```

### What Happens

1. **Azure What-If runs:** Shows changes to deploy
2. **AI analyzes changes:** Evaluates three risk buckets
3. **Drift check:** Compares What-If to git diff
   - Finds: Storage account `publicNetworkAccess` changing Disabled → Enabled
   - But: No code changes to this property
   - **Result: High drift risk** ⚠️
4. **Intent check:** Compares changes to PR title
   - PR says: "Add Application Insights"
   - What-If shows: Adding App Insights + Changing storage security
   - **Result: High intent misalignment** ⚠️
5. **Operations check:** Evaluates operation risk
   - Enabling public network access is dangerous
   - **Result: High operations risk** ⚠️

### The Verdict

```
Risk Assessment:
  Drift: High       (threshold: high)   → ❌ FAIL
  Intent: High      (threshold: high)   → ❌ FAIL
  Operations: High  (threshold: high)   → ❌ FAIL

Verdict: UNSAFE
Exit Code: 1
```

### PR Comment Posted

```markdown
## What-If Deployment Review

### Risk Assessment

| Risk Bucket | Risk Level | Key Concerns |
|-------------|------------|--------------|
| Infrastructure Drift | High | Storage account publicNetworkAccess changed manually |
| PR Intent Alignment | High | PR is about monitoring, not security changes |
| Risky Operations | High | Enabling public network access on storage |

### Verdict: ❌ UNSAFE

**Reasoning:** Storage account was manually secured (publicNetworkAccess: Disabled)
outside of this PR, but the Bicep template will revert it to an insecure state
(Enabled). This security regression is unrelated to the PR's stated monitoring
purpose and appears to be unintentional.
```

### What You Should Do

1. **Review the drift:** Check why storage account was manually changed
2. **Update Bicep code:** Add `publicNetworkAccess: 'Disabled'` to template
3. **Update PR:** Explain the security fix
4. **Re-run workflow:** Will now pass with code matching infrastructure

## FAQ

### Q: What if I disagree with the risk assessment?

**A:** You can:
1. Adjust thresholds to be more lenient: `--drift-threshold high` → `--intent-threshold medium`
2. Manually override by bypassing the check (not recommended)
3. Review the PR comment reasoning and address the concerns
4. Use `--ci` flag locally to test before pushing

### Q: Can I see what caused each risk level?

**A:** Yes! The PR comment includes:
- Key concerns for each bucket
- Specific resources flagged
- Reasoning for the verdict
- Recommendations

### Q: Does it block safe changes?

**A:** With default thresholds (all "high"), it only blocks critical issues:
- Security regressions
- Unintended deletions
- Major drift
- Complete misalignment with PR intent

Routine changes like adding resources, updating tags, or modifying configurations typically get low/medium risk and pass.

### Q: What if there's no PR description?

**A:** The Intent bucket is skipped. Risk assessment continues with just:
- Drift detection
- Operations risk

This is fine - you can still catch drift and risky operations.

### Q: Can I test locally?

**A:** Yes! Use `--ci` flag manually:

```bash
az deployment group what-if ... | whatif-explain \
  --ci \
  --diff-ref origin/main \
  --pr-title "My test PR"
```

This lets you see the risk assessment before pushing.

## Summary

**Simple mental model:**

1. **Three checks** (drift, intent, operations)
2. **Each gets a risk level** (low, medium, high)
3. **You set thresholds** for each check
4. **Deployment blocked** if ANY check exceeds its threshold
5. **Exit code 0** = safe, **1** = blocked, **2** = error
6. **PR comment** explains everything

**Default behavior (recommended):**
- Thresholds all set to "high"
- Only critical issues block deployments
- Routine changes pass through
- You get detailed PR comments either way

**Key insight:** The three-bucket system catches different types of problems:
- **Drift** → "Does code match reality?"
- **Intent** → "Does this match what the PR claims?"
- **Operations** → "Is this operation inherently risky?"

All three checks together provide comprehensive deployment safety! 🛡️
