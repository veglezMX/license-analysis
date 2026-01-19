# FOSSA Pipeline Setup Documentation

> **Documentation Index**
> 
> This documentation covers everything teams need to integrate FOSSA scanning into their CI/CD pipelines and connect to the Compliance Dashboard.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Quick Start](#quick-start)
4. [Azure DevOps Setup](#azure-devops-setup)
5. [GitLab CI Setup](#gitlab-ci-setup)
6. [GitHub Actions Setup](#github-actions-setup)
7. [Configuration Reference](#configuration-reference)
8. [Dashboard Integration](#dashboard-integration)
9. [Troubleshooting](#troubleshooting)
10. [FAQ](#faq)

---

# Overview

## What is FOSSA?

FOSSA is a license compliance and security scanning tool that analyzes your project's dependencies to:

- **Detect license violations** — Identify copyleft licenses (GPL, AGPL) that may conflict with your distribution model
- **Find vulnerabilities** — Match dependencies against CVE databases
- **Generate SBOMs** — Create Software Bill of Materials for compliance requirements

## How It Works

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Your CI/CD     │────▶│  FOSSA Cloud    │◀────│  Compliance     │
│  Pipeline       │     │                 │     │  Dashboard      │
│                 │     │  • Analyzes     │     │                 │
│  Runs:          │     │  • Stores       │     │  • Displays     │
│  • fossa analyze│     │  • Evaluates    │     │  • Tracks       │
│  • fossa test   │     │    policies     │     │  • Audits       │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

1. **Your pipeline** runs the FOSSA CLI to analyze dependencies
2. **FOSSA Cloud** processes the data and evaluates against policies
3. **Compliance Dashboard** fetches and displays results for review

## Architecture

```
Developer pushes code
        ↓
CI/CD pipeline triggered
        ↓
┌───────────────────────────────────────────┐
│  FOSSA Scan Stage                         │
│  ┌─────────────────────────────────────┐  │
│  │ 1. Install FOSSA CLI                │  │
│  │ 2. Run fossa analyze               │  │
│  │ 3. Run fossa test (policy check)   │  │
│  │ 4. Notify Compliance Dashboard     │  │
│  └─────────────────────────────────────┘  │
└───────────────────────────────────────────┘
        ↓
Results visible in Compliance Dashboard
```

---

# Prerequisites

## Before You Begin

Ensure you have the following:

### 1. FOSSA Account Access

You need access to your organization's FOSSA account at [app.fossa.com](https://app.fossa.com).

**Don't have access?** Contact your Compliance Team or IT administrator.

### 2. FOSSA API Key

You'll need a **Push-Only API key** for your CI/CD pipeline.

#### How to Get Your API Key

1. Log in to [app.fossa.com](https://app.fossa.com)
2. Click your profile icon → **Account Settings**
3. Navigate to **Integrations** → **API Tokens**
4. Click **Add API Token**
5. Configure the token:
   - **Name:** `CI Pipeline - [your-repo-name]`
   - **Token Type:** `Push Only` (recommended for CI)
6. Click **Create**
7. **Copy the token immediately** — you won't see it again!

```
┌─────────────────────────────────────────────────────────────┐
│  ⚠️  IMPORTANT: Token Security                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  • Never commit your API key to source control              │
│  • Always use CI/CD secrets/variables                       │
│  • Use Push-Only tokens for pipelines (principle of         │
│    least privilege)                                         │
│  • Rotate tokens periodically                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3. CI/CD Access

You need permissions to:
- Edit pipeline configuration files
- Add secrets/variables to your CI/CD system

### 4. Compliance Dashboard Token (Optional)

If you want your pipeline to notify the Compliance Dashboard when scans complete:

1. Log in to the Compliance Dashboard
2. Go to **Settings** → **API Keys**
3. Create a new key with **Webhook** scope
4. Save this as `COMPLIANCE_DASHBOARD_TOKEN` in your CI/CD secrets

---

# Quick Start

## 5-Minute Setup

Choose your CI/CD platform and follow the steps:

### Azure DevOps

```yaml
# azure-pipelines.yml
resources:
  repositories:
    - repository: templates
      type: git
      name: DevOps/azure-pipeline-templates
      ref: refs/tags/v1.0.0

stages:
  - template: fossa/v1/scan.yml@templates
```

**Required secret:** `FOSSA_API_KEY`

[→ Full Azure DevOps Guide](#azure-devops-setup)

---

### GitLab CI

```yaml
# .gitlab-ci.yml
include:
  - project: 'devops/gitlab-ci-templates'
    ref: 'v1.0.0'
    file: '/fossa/v1/.fossa-scan.yml'

stages:
  - fossa-scan
```

**Required variable:** `FOSSA_API_KEY` (masked)

[→ Full GitLab CI Guide](#gitlab-ci-setup)

---

### GitHub Actions

```yaml
# .github/workflows/fossa.yml
name: FOSSA Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  fossa-scan:
    uses: your-org/github-actions-templates/.github/workflows/fossa-scan.yml@v1.0.0
    secrets:
      FOSSA_API_KEY: ${{ secrets.FOSSA_API_KEY }}
```

**Required secret:** `FOSSA_API_KEY`

[→ Full GitHub Actions Guide](#github-actions-setup)

---

# Azure DevOps Setup

## Step 1: Add the Template Repository

Your organization maintains reusable pipeline templates. First, reference this repository:

```yaml
# azure-pipelines.yml

trigger:
  branches:
    include:
      - main
      - develop

resources:
  repositories:
    - repository: templates
      type: git
      name: DevOps/azure-pipeline-templates    # ← Your templates repo
      ref: refs/tags/v1.0.0                    # ← Pin to version
```

## Step 2: Configure the FOSSA API Key

### Option A: Pipeline Variable (Single Pipeline)

1. Go to **Pipelines** → Select your pipeline → **Edit**
2. Click **Variables** → **New Variable**
3. Configure:
   - **Name:** `FOSSA_API_KEY`
   - **Value:** `[paste your token]`
   - **Keep this value secret:** ✅ Checked
4. Click **OK** → **Save**

### Option B: Variable Group (Multiple Pipelines)

1. Go to **Pipelines** → **Library** → **+ Variable group**
2. Name it: `fossa-credentials`
3. Add variable:
   - **Name:** `FOSSA_API_KEY`
   - **Value:** `[paste your token]`
   - Click the 🔒 lock icon to make it secret
4. **Save**
5. In each pipeline, reference the group:

```yaml
variables:
  - group: fossa-credentials
```

## Step 3: Add the FOSSA Scan Stage

### Minimal Configuration

```yaml
# azure-pipelines.yml

trigger:
  - main

resources:
  repositories:
    - repository: templates
      type: git
      name: DevOps/azure-pipeline-templates
      ref: refs/tags/v1.0.0

variables:
  - group: fossa-credentials

stages:
  - stage: Build
    jobs:
      - job: Build
        steps:
          - script: npm install && npm run build

  # Add FOSSA scanning
  - template: fossa/v1/scan.yml@templates
```

### Full Configuration

```yaml
# azure-pipelines.yml

trigger:
  branches:
    include:
      - main
      - develop
      - release/*

pr:
  branches:
    include:
      - main

resources:
  repositories:
    - repository: templates
      type: git
      name: DevOps/azure-pipeline-templates
      ref: refs/tags/v1.0.0

variables:
  - group: fossa-credentials
  - group: compliance-dashboard-credentials  # Optional

stages:
  - stage: Build
    jobs:
      - job: BuildAndTest
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - script: npm ci
          - script: npm run build
          - script: npm test

  # FOSSA Scan with full configuration
  - template: fossa/v1/scan.yml@templates
    parameters:
      # Project identification
      projectName: 'my-service'              # Defaults to repo name
      branch: '$(Build.SourceBranchName)'
      
      # Behavior
      failOnIssues: true                     # Fail pipeline on violations
      skipTest: false                        # Run policy check
      
      # Dashboard notification
      notifyDashboard: true
      dashboardWebhookUrl: 'https://compliance.yourcompany.com/api/webhooks/fossa-scan'
      dashboardTokenSecret: 'COMPLIANCE_DASHBOARD_TOKEN'
      
      # Advanced
      fossaCliVersion: 'latest'
      timeout: 600
      additionalAnalyzeArgs: ''
      additionalTestArgs: ''

  - stage: Deploy
    dependsOn: 
      - Build
      - FOSSA_Scan
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - job: Deploy
        steps:
          - script: ./deploy.sh
```

## Step 4: Verify the Setup

1. **Commit and push** your pipeline changes
2. **Check the pipeline run** in Azure DevOps
3. **Verify in FOSSA:** Log in to app.fossa.com and check your project appears
4. **Verify in Dashboard:** Check the Compliance Dashboard shows your component

---

# GitLab CI Setup

## Step 1: Configure CI/CD Variables

1. Go to your project → **Settings** → **CI/CD**
2. Expand **Variables**
3. Click **Add variable**:
   - **Key:** `FOSSA_API_KEY`
   - **Value:** `[paste your token]`
   - **Type:** Variable
   - **Flags:** 
     - ✅ Mask variable
     - ✅ Protect variable (optional, if only scanning protected branches)
4. Click **Add variable**

### For Dashboard Notifications (Optional)

Add another variable:
- **Key:** `COMPLIANCE_DASHBOARD_TOKEN`
- **Value:** `[paste your dashboard token]`
- **Flags:** ✅ Mask variable

## Step 2: Include the Template

### Minimal Configuration

```yaml
# .gitlab-ci.yml

include:
  - project: 'devops/gitlab-ci-templates'
    ref: 'v1.0.0'
    file: '/fossa/v1/.fossa-scan.yml'

stages:
  - build
  - test
  - fossa-scan    # From template
  - deploy

build:
  stage: build
  script:
    - npm ci
    - npm run build

test:
  stage: test
  script:
    - npm test

# fossa-scan job is automatically included from template

deploy:
  stage: deploy
  script:
    - ./deploy.sh
  only:
    - main
```

### Full Configuration

```yaml
# .gitlab-ci.yml

include:
  - project: 'devops/gitlab-ci-templates'
    ref: 'v1.0.0'
    file: '/fossa/v1/.fossa-scan.yml'

# Override template variables
variables:
  # Project identification
  FOSSA_PROJECT_NAME: "my-service"
  FOSSA_BRANCH: "${CI_COMMIT_REF_NAME}"
  
  # Behavior
  FOSSA_FAIL_ON_ISSUES: "true"
  FOSSA_SKIP_TEST: "false"
  FOSSA_TEST_TIMEOUT: "600"
  
  # Dashboard notification
  FOSSA_NOTIFY_DASHBOARD: "true"
  COMPLIANCE_DASHBOARD_URL: "https://compliance.yourcompany.com/api/webhooks/fossa-scan"
  
  # Advanced
  FOSSA_CLI_VERSION: "latest"
  FOSSA_ADDITIONAL_ANALYZE_ARGS: ""
  FOSSA_ADDITIONAL_TEST_ARGS: ""

stages:
  - build
  - test
  - fossa-scan
  - deploy

build:
  stage: build
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/

test:
  stage: test
  script:
    - npm test

# The fossa-scan job runs automatically from template

deploy:
  stage: deploy
  script:
    - ./deploy.sh
  only:
    - main
  needs:
    - build
    - test
    - fossa-scan
```

## Step 3: Customize Job Rules (Optional)

If you want to control when FOSSA scans run:

```yaml
# Override the fossa-scan job's rules
fossa-scan:
  rules:
    # Always run on main and develop
    - if: $CI_COMMIT_BRANCH == "main"
      when: always
    - if: $CI_COMMIT_BRANCH == "develop"
      when: always
    # Run on merge requests to main
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "main"
      when: always
    # Run on tags
    - if: $CI_COMMIT_TAG
      when: always
    # Don't run on feature branches
    - when: never
```

## Step 4: Verify the Setup

1. **Commit and push** your `.gitlab-ci.yml`
2. **Check the pipeline** in GitLab CI/CD → Pipelines
3. **Verify in FOSSA:** Check your project at app.fossa.com
4. **Verify in Dashboard:** Check the Compliance Dashboard

---

# GitHub Actions Setup

## Step 1: Add the FOSSA API Key Secret

1. Go to your repository → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Configure:
   - **Name:** `FOSSA_API_KEY`
   - **Secret:** `[paste your token]`
4. Click **Add secret**

### For Dashboard Notifications (Optional)

Add another secret:
- **Name:** `COMPLIANCE_DASHBOARD_TOKEN`
- **Secret:** `[paste your dashboard token]`

## Step 2: Create the Workflow File

### Minimal Configuration

```yaml
# .github/workflows/fossa.yml

name: FOSSA License Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  fossa-scan:
    uses: your-org/github-actions-templates/.github/workflows/fossa-scan.yml@v1.0.0
    secrets:
      FOSSA_API_KEY: ${{ secrets.FOSSA_API_KEY }}
```

### Full Configuration (Inline)

If you prefer not to use reusable workflows:

```yaml
# .github/workflows/fossa.yml

name: FOSSA License Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:  # Allow manual trigger

env:
  FOSSA_PROJECT_NAME: ${{ github.event.repository.name }}
  FOSSA_FAIL_ON_ISSUES: 'true'
  COMPLIANCE_DASHBOARD_URL: 'https://compliance.yourcompany.com/api/webhooks/fossa-scan'

jobs:
  fossa-scan:
    name: FOSSA Analysis
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install FOSSA CLI
        run: |
          curl -H 'Cache-Control: no-cache' \
            https://raw.githubusercontent.com/fossas/fossa-cli/master/install-latest.sh | bash
          fossa --version

      - name: Run FOSSA Analyze
        env:
          FOSSA_API_KEY: ${{ secrets.FOSSA_API_KEY }}
        run: |
          fossa analyze \
            --project "$FOSSA_PROJECT_NAME" \
            --revision "${{ github.sha }}" \
            --branch "${{ github.ref_name }}"

      - name: Run FOSSA Test
        id: fossa-test
        env:
          FOSSA_API_KEY: ${{ secrets.FOSSA_API_KEY }}
        run: |
          set +e
          fossa test --timeout 600
          FOSSA_EXIT_CODE=$?
          echo "exit_code=$FOSSA_EXIT_CODE" >> $GITHUB_OUTPUT
          
          if [ $FOSSA_EXIT_CODE -ne 0 ]; then
            echo "::warning::FOSSA found policy violations"
            if [ "$FOSSA_FAIL_ON_ISSUES" = "true" ]; then
              exit 1
            fi
          fi

      - name: Notify Compliance Dashboard
        if: always()
        env:
          COMPLIANCE_DASHBOARD_TOKEN: ${{ secrets.COMPLIANCE_DASHBOARD_TOKEN }}
          WEBHOOK_SECRET: ${{ secrets.COMPLIANCE_WEBHOOK_SECRET }}
        run: |
          if [ "${{ steps.fossa-test.outputs.exit_code }}" = "0" ]; then
            SCAN_STATUS="passed"
          else
            SCAN_STATUS="failed"
          fi
          
          TIMESTAMP=$(date +%s)
          PAYLOAD=$(cat <<EOF
          {
            "event": "fossa.scan.completed",
            "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
            "repository": {
              "name": "${{ github.event.repository.name }}",
              "url": "${{ github.event.repository.html_url }}",
              "provider": "github"
            },
            "scan": {
              "revision": "${{ github.sha }}",
              "branch": "${{ github.ref_name }}",
              "status": "${SCAN_STATUS}"
            },
            "pipeline": {
              "id": "${{ github.run_id }}",
              "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}",
              "provider": "github-actions"
            }
          }
          EOF
          )
          
          SIGNATURE=$(echo -n "${TIMESTAMP}.${PAYLOAD}" | openssl dgst -sha256 -hmac "${WEBHOOK_SECRET}" | cut -d' ' -f2)
          
          curl -X POST "${COMPLIANCE_DASHBOARD_URL}" \
            -H "Content-Type: application/json" \
            -H "X-Webhook-Signature: sha256=${SIGNATURE}" \
            -H "X-Webhook-Timestamp: ${TIMESTAMP}" \
            -d "${PAYLOAD}" || echo "Warning: Failed to notify dashboard"
```

## Step 3: Verify the Setup

1. **Commit and push** the workflow file
2. **Check Actions tab** in GitHub
3. **Verify in FOSSA:** Check your project at app.fossa.com
4. **Verify in Dashboard:** Check the Compliance Dashboard

---

# Configuration Reference

## Template Parameters

### Azure DevOps Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `fossaApiKeySecret` | string | `FOSSA_API_KEY` | Name of secret containing FOSSA token |
| `projectName` | string | Repo name | FOSSA project identifier |
| `branch` | string | Current branch | Branch name for tracking |
| `failOnIssues` | boolean | `true` | Fail pipeline on policy violations |
| `skipTest` | boolean | `false` | Skip policy check (only analyze) |
| `notifyDashboard` | boolean | `true` | Send webhook to dashboard |
| `dashboardWebhookUrl` | string | See template | Dashboard webhook URL |
| `dashboardTokenSecret` | string | `COMPLIANCE_DASHBOARD_TOKEN` | Secret name for dashboard auth |
| `fossaCliVersion` | string | `latest` | FOSSA CLI version |
| `workingDirectory` | string | Source directory | Path to analyze (for monorepos) |
| `additionalAnalyzeArgs` | string | `''` | Extra args for `fossa analyze` |
| `additionalTestArgs` | string | `''` | Extra args for `fossa test` |
| `timeout` | number | `600` | Test timeout in seconds |

### GitLab CI Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `FOSSA_API_KEY` | (required) | FOSSA API token |
| `FOSSA_PROJECT_NAME` | `$CI_PROJECT_NAME` | FOSSA project identifier |
| `FOSSA_BRANCH` | `$CI_COMMIT_REF_NAME` | Branch name |
| `FOSSA_REVISION` | `$CI_COMMIT_SHA` | Commit SHA |
| `FOSSA_FAIL_ON_ISSUES` | `true` | Fail on violations |
| `FOSSA_SKIP_TEST` | `false` | Skip policy check |
| `FOSSA_TEST_TIMEOUT` | `600` | Test timeout (seconds) |
| `FOSSA_NOTIFY_DASHBOARD` | `true` | Send webhook |
| `COMPLIANCE_DASHBOARD_URL` | See template | Webhook URL |
| `COMPLIANCE_DASHBOARD_TOKEN` | (optional) | Dashboard auth token |
| `FOSSA_CLI_VERSION` | `latest` | CLI version |
| `FOSSA_WORKING_DIRECTORY` | `$CI_PROJECT_DIR` | Analysis directory |
| `FOSSA_ADDITIONAL_ANALYZE_ARGS` | `''` | Extra analyze args |
| `FOSSA_ADDITIONAL_TEST_ARGS` | `''` | Extra test args |

## FOSSA CLI Arguments

### `fossa analyze` Options

| Argument | Description |
|----------|-------------|
| `--project NAME` | Project name in FOSSA |
| `--revision SHA` | Commit/revision identifier |
| `--branch NAME` | Branch name |
| `--only-direct` | Only analyze direct dependencies |
| `--detect-vendored` | Detect vendored dependencies (C/C++) |
| `--debug` | Verbose output |

### `fossa test` Options

| Argument | Description |
|----------|-------------|
| `--timeout SECONDS` | Max wait time for results |
| `--suppress-issues TYPES` | Ignore specific issue types |

---

# Dashboard Integration

## Linking Your Component

After your first successful scan:

1. **Log in** to the Compliance Dashboard
2. **Create a new component** (or edit existing)
3. **Select your FOSSA project** from the dropdown
4. **Save** — results will sync automatically

## Real-Time Updates

When your pipeline runs:

1. FOSSA CLI uploads scan results
2. Pipeline sends webhook to dashboard
3. Dashboard fetches latest results from FOSSA
4. Your component's status updates immediately

## Webhook Configuration

The pipeline templates automatically notify the dashboard. If you need to configure manually:

**Webhook URL:**
```
https://compliance.yourcompany.com/api/webhooks/fossa-scan
```

**Required Headers:**
```
Content-Type: application/json
X-Webhook-Signature: sha256=<hmac_signature>
X-Webhook-Timestamp: <unix_timestamp>
```

**Payload:**
```json
{
  "event": "fossa.scan.completed",
  "repository": {
    "url": "https://github.com/your-org/your-repo"
  },
  "scan": {
    "revision": "<commit_sha>",
    "branch": "<branch_name>",
    "status": "passed|failed"
  }
}
```

---

# Troubleshooting

## Common Issues

### "FOSSA_API_KEY not set"

**Cause:** The secret/variable isn't configured or has the wrong name.

**Solution:**
1. Verify the secret exists in your CI/CD system
2. Check the name matches exactly (case-sensitive)
3. Ensure the secret is not protected when running on unprotected branches

---

### "fossa analyze failed: no targets found"

**Cause:** FOSSA couldn't detect your project's package manager.

**Solution:**
1. Ensure you have a manifest file (`package.json`, `pom.xml`, `requirements.txt`, etc.)
2. Run `npm install` / `pip install` before `fossa analyze`
3. Check the working directory is correct

---

### "fossa test timed out"

**Cause:** FOSSA took too long to analyze your project.

**Solution:**
1. Increase the timeout parameter
2. Check if you have an unusually large number of dependencies
3. Contact FOSSA support if the issue persists

---

### "Policy violations found" but you want to continue

**Solution:** Set `failOnIssues: false` (Azure) or `FOSSA_FAIL_ON_ISSUES: "false"` (GitLab)

This will warn but not fail the pipeline.

---

### "Template not found"

**Cause:** The template repository or ref doesn't exist.

**Solution:**
1. Verify the repository name is correct
2. Check you have access to the templates repository
3. Verify the version tag exists

---

### "Webhook failed to notify dashboard"

**Cause:** Network issue or invalid credentials.

**Solution:**
1. Check the dashboard URL is correct
2. Verify the dashboard token is valid
3. Check network connectivity from your CI/CD runner
4. Review dashboard logs for rejected webhooks

---

### Component not appearing in Dashboard

**Cause:** The FOSSA project locator doesn't match.

**Solution:**
1. Check the project name in FOSSA matches what's in the dropdown
2. Ensure the first scan completed successfully
3. Try refreshing the project list in the dashboard

---

## Debug Mode

Enable verbose output to troubleshoot issues:

**Azure DevOps:**
```yaml
parameters:
  additionalAnalyzeArgs: '--debug'
```

**GitLab CI:**
```yaml
variables:
  FOSSA_ADDITIONAL_ANALYZE_ARGS: "--debug"
```

---

## Getting Help

1. **Check FOSSA Docs:** [docs.fossa.com](https://docs.fossa.com)
2. **Internal Support:** Contact the Compliance Team via Slack `#compliance-support`
3. **Template Issues:** File an issue in the `DevOps/pipeline-templates` repository

---

# FAQ

## General

### Q: Do I need a FOSSA account for each developer?

**A:** No. The CI/CD pipeline uses a shared API key. Individual developers don't need FOSSA accounts unless they want to view results directly in FOSSA's UI.

---

### Q: How often should I scan?

**A:** We recommend scanning on:
- Every push to `main`/`develop`
- Every merge request/pull request
- Release tags

The templates are configured for this by default.

---

### Q: Will FOSSA slow down my pipeline?

**A:** Typical scan times:
- Small project: 1-2 minutes
- Medium project: 2-5 minutes
- Large project: 5-10 minutes

FOSSA scans run in parallel with other stages, so they rarely block deployments.

---

## Policies

### Q: What happens if my scan fails?

**A:** By default, the pipeline fails. You can:
1. Fix the violation (update/remove the dependency)
2. Get an approval in the Compliance Dashboard
3. Set `failOnIssues: false` to make scans non-blocking (not recommended for production)

---

### Q: How do I get an approval for a violation?

**A:**
1. Go to the Compliance Dashboard
2. Find your component
3. Click on the issue
4. Click "Request Approval" and provide justification
5. Your Compliance Officer will review

---

### Q: Can I ignore certain licenses?

**A:** Yes, configure this in FOSSA's policy settings. Contact your Compliance Officer to request policy changes.

---

## Monorepos

### Q: How do I scan multiple projects in a monorepo?

**A:** Use the `workingDirectory` parameter to scan each project separately:

**Azure DevOps:**
```yaml
stages:
  - template: fossa/v1/scan.yml@templates
    parameters:
      projectName: 'myapp-backend'
      workingDirectory: '$(Build.SourcesDirectory)/backend'

  - template: fossa/v1/scan.yml@templates
    parameters:
      projectName: 'myapp-frontend'
      workingDirectory: '$(Build.SourcesDirectory)/frontend'
```

**GitLab CI:**
```yaml
fossa-backend:
  extends: .fossa-base
  variables:
    FOSSA_PROJECT_NAME: "myapp-backend"
    FOSSA_WORKING_DIRECTORY: "${CI_PROJECT_DIR}/backend"

fossa-frontend:
  extends: .fossa-base
  variables:
    FOSSA_PROJECT_NAME: "myapp-frontend"
    FOSSA_WORKING_DIRECTORY: "${CI_PROJECT_DIR}/frontend"
```

---

## Security

### Q: Is my source code sent to FOSSA?

**A:** No. FOSSA CLI only uploads dependency metadata (package names, versions, licenses). Your source code stays in your CI/CD environment.

---

### Q: Can I use a self-hosted FOSSA instance?

**A:** Yes, if your organization has FOSSA Enterprise. Update the `--endpoint` parameter in the templates.

---

## Upgrades

### Q: How do I upgrade to a new template version?

**A:**
1. Check the CHANGELOG for breaking changes
2. Update the version reference:
   - Azure: `ref: refs/tags/v1.1.0`
   - GitLab: `ref: 'v1.1.0'`
3. Test in a non-production branch first
4. Roll out to all pipelines

---

### Q: What's the current stable version?

**A:** Check the templates repository for the latest tag. Current stable: **v1.0.0**

---

# Changelog

## v1.0.0 (2025-01-15)

- Initial release
- Azure DevOps template
- GitLab CI template
- Dashboard webhook integration
- Monorepo support

---

# Support

| Channel | Use For |
|---------|---------|
| `#compliance-support` (Slack) | Quick questions |
| `DevOps/pipeline-templates` (Issues) | Bug reports, feature requests |
| compliance-team@yourcompany.com | Policy questions |
| FOSSA Support | FOSSA product issues |
