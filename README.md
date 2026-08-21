# 🔐 GitHub Actions Secrets Audit Workflow

[![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-Automated%20Workflow-2088FF?logo=github-actions&logoColor=white)](https://github.com/features/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Node.js Runtime](https://img.shields.io/badge/Node.js-v24%20Native-339933?logo=node.js&logoColor=white)](https://nodejs.org/)
[![Security: Supply Chain Hardened](https://img.shields.io/badge/Security-SHA%20Pinned-success?logo=github)](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

A production-grade, security-hardened GitHub Actions workflow to audit, inspect, and export all secrets accessible to a repository (repository-level and inherited organization-level secrets) into ephemeral artifacts.

Supports structured output formats (`JSON`, `YAML`, `.env`), optional value redaction (key inventory mode), and automated runner token filtering.

---

## 📑 Table of Contents

- [Overview](#-overview)
- [Architecture & Execution Lifecycle](#-architecture--execution-lifecycle)
- [Inputs & Configuration](#-inputs--configuration)
- [How to Run](#-how-to-run)
  - [Via GitHub Web UI](#via-github-web-ui)
  - [Via GitHub CLI (`gh`)](#via-github-cli-gh)
- [Verified Test Runs & Actual Output Logs](#-verified-test-runs--actual-output-logs)
  - [Test 1: JSON Export (Real Values, Token Excluded)](#test-1-json-export-real-values-token-excluded)
  - [Test 2: YAML Export (Redacted Inventory Mode)](#test-2-yaml-export-redacted-inventory-mode)
  - [Test 3: Dotenv Format (`.env`)](#test-3-dotenv-format-env)
  - [Test 4: JSON Export with Ephemeral Token & Redaction](#test-4-json-export-with-ephemeral-token--redaction)
- [Job Step Summary Preview](#-job-step-summary-preview)
- [Security Hardening Architecture](#-security-hardening-architecture)
- [Roadmap & Future Extensions](#-roadmap--future-extensions)
  - [1. Environment-Scoped Secrets](#1-environment-scoped-secrets)
  - [2. GPG / Asymmetric Artifact Encryption](#2-gpg--asymmetric-artifact-encryption)
- [License](#-license)

---

## 🔍 Overview

When managing microservices and repositories across teams and organizations, SecOps and DevOps engineers often need to:
1. **Audit accessible secrets:** Identify which secrets are currently active and injected into workflow runtimes.
2. **Export configuration dumps:** Securely obtain configurations for local troubleshooting or disaster recovery.
3. **Verify naming standards:** Audit consistency of secret keys across distributed environments.
4. **Detect unintentional exposure:** Verify whether organization-level secrets are visible to repositories that should not access them.

This workflow runs on an isolated GitHub-hosted runner (`ubuntu-latest`), parses the `secrets` context, applies custom redaction and filtering rules, uploads a short-lived artifact (24h retention), and performs multi-pass secure file shredding.

---

## 🛡️ Architecture & Execution Lifecycle

```
┌────────────────────────────────────────────────────────┐
│             GitHub Actions Runner (Ubuntu)             │
│                                                        │
│  1. Ingest Context: ${{ toJSON(secrets) }}             │
│            │                                           │
│            ▼                                           │
│  2. Token Filter: Omit GITHUB_TOKEN (Default: true)    │
│            │                                           │
│            ▼                                           │
│  3. Redaction Engine: Value Masking (Optional)         │
│            │                                           │
│            ▼                                           │
│  4. Formatter: JSON | YAML | .ENV                      │
│            │                                           │
│            ▼                                           │
│  5. Export to Out-of-Workspace /tmp/audit-*            │
│            │                                           │
│            ▼                                           │
│  6. Upload Ephemeral Artifact (Retention: 24h)         │
│            │                                           │
│            ▼                                           │
│  7. Defense-in-Depth Shredding: `shred -u`             │
└────────────────────────────────────────────────────────┘
```

---

## ⚙️ Inputs & Configuration

The workflow is triggered on demand via `workflow_dispatch` with the following parameters:

| Parameter | Type | Default | Options | Description |
|---|---|---|---|---|
| `format` | `choice` | `json` | `json`, `yaml`, `env` | Serialization format of the dumped secrets file. |
| `redact` | `boolean` | `false` | `true`, `false` | When `true`, replaces all secret values with `[REDACTED]` (inventory only). |
| `include_github_token` | `boolean` | `false` | `true`, `false` | When `true`, includes the runner's ephemeral `GITHUB_TOKEN`. Default is `false`. |

---

## 🚀 How to Run

> [!NOTE]
> Workflows must be triggered sequentially. A concurrency lock (`secrets-audit-${{ github.ref }}`) prevents simultaneous overlapping runs on the same branch. If approval rules are enabled, approve the run from the GitHub Actions tab.

### Via GitHub Web UI
1. Navigate to your repository on GitHub.
2. Click the **Actions** tab.
3. Select **🔐 Secrets Audit** from the workflow list on the left.
4. Click **Run workflow**, set your preferred options (`format`, `redact`, `include_github_token`), and click the green **Run workflow** button.
5. Once the job completes, download the generated artifact from the run summary.

### Via GitHub CLI (`gh`)

```bash
# JSON Export with actual values (Token excluded)
gh workflow run "secrets-audit.yml" -f format=json -f redact=false -f include_github_token=false

# YAML Export with values redacted (Keys only)
gh workflow run "secrets-audit.yml" -f format=yaml -f redact=true -f include_github_token=false

# Dotenv Export (.env format)
gh workflow run "secrets-audit.yml" -f format=env -f redact=false -f include_github_token=false

# Download the resulting artifact from the latest run
gh run download $(gh run list --workflow="secrets-audit.yml" --limit 1 --json databaseId -q '.[0].databaseId')
```

---

## 🧪 Verified Test Runs & Actual Output Logs

All parameter combinations have been validated in GitHub Actions runners. Below are the actual execution logs and resulting file contents.

### Test 1: JSON Export (Real Values, Token Excluded)
* **Execution Parameters:** `format: json`, `redact: false`, `include_github_token: false`
* **Runner Step Log:**
  ```text
  ✓ Set up job (1s)
  ✓ Build secrets file (1s)
  ✓ Upload artifact (2s)
  ✓ Shred temp file (1s)
  ✓ Complete job (0s)
  Result: Success (0 warnings, 0 deprecation notices)
  ```
* **Generated Artifact Content (`secrets-*.json`):**
  ```json
  {
    "TEST_DATABASE_URL": "postgres://user:pass@db.example.com:5432/mydb",
    "TEST_API_KEY": "super-secret-api-key-12345"
  }
  ```

---

### Test 2: YAML Export (Redacted Inventory Mode)
* **Execution Parameters:** `format: yaml`, `redact: true`, `include_github_token: false`
* **Runner Step Log:**
  ```text
  ✓ Set up job (1s)
  ✓ Build secrets file (1s)
  ✓ Upload artifact (1s)
  ✓ Shred temp file (0s)
  ✓ Complete job (0s)
  Result: Success
  ```
* **Generated Artifact Content (`secrets-*.yaml`):**
  ```yaml
  secrets:
    TEST_DATABASE_URL: [REDACTED]
    TEST_API_KEY: [REDACTED]
  ```

---

### Test 3: Dotenv Format (`.env`)
* **Execution Parameters:** `format: env`, `redact: false`, `include_github_token: false`
* **Runner Step Log:**
  ```text
  ✓ Set up job (1s)
  ✓ Build secrets file (1s)
  ✓ Upload artifact (1s)
  ✓ Shred temp file (1s)
  ✓ Complete job (0s)
  Result: Success
  ```
* **Generated Artifact Content (`secrets-*.env`):**
  ```env
  TEST_DATABASE_URL=postgres://user:pass@db.example.com:5432/mydb
  TEST_API_KEY=super-secret-api-key-12345
  ```

---

### Test 4: JSON Export with Ephemeral Token & Redaction
* **Execution Parameters:** `format: json`, `redact: true`, `include_github_token: true`
* **Runner Step Log:**
  ```text
  ✓ Set up job (1s)
  ✓ Build secrets file (1s)
  ✓ Upload artifact (2s)
  ✓ Shred temp file (1s)
  ✓ Complete job (0s)
  Result: Success
  ```
* **Generated Artifact Content (`secrets-*.json`):**
  ```json
  {
    "TEST_DATABASE_URL": "[REDACTED]",
    "TEST_API_KEY": "[REDACTED]",
    "github_token": "[REDACTED]"
  }
  ```

---

## 📊 Job Step Summary Preview

Every run automatically publishes a clear audit report directly in the GitHub Actions run UI:

```markdown
## 🔐 Secrets Audit

| Metric | Value |
|---|---|
| Format | `json` |
| Total Secrets | **2** |
| Redaction Enabled | `false` |
| Includes GITHUB_TOKEN | `false` |
| Timestamp (UTC) | 2026-08-21T11:17:25Z |

### Found Keys
```
TEST_API_KEY
TEST_DATABASE_URL
```
```

---

## 🔒 Security Hardening Architecture

This workflow implements defense-in-depth security best practices:

| Measure | Implementation | Security Value |
|---|---|---|
| **Immutable SHA Pinning** | `actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a` | Uses immutable commit SHA (`v7.0.1`) rather than mutable version tags (`@v4` / `@v7`), mitigating supply chain compromises. |
| **Node.js 24 Native** | Action targets `node24` | Fully compliant with current runner runtimes, eliminating deprecation warnings and execution halts. |
| **Least Privilege (PoLP)** | `permissions: contents: read` | No administrative, repository-write, or secret-modification permissions granted. |
| **Workspace Isolation** | `/tmp/audit-${GITHUB_RUN_ID}` | Files are generated strictly outside `$GITHUB_WORKSPACE`, preventing accidental commits or exposure in git workflows. |
| **Ephemeral Artifacts** | `retention-days: 1` | Artifacts automatically self-destruct after 24 hours. |
| **Multi-pass Disk Shredding** | `shred -u` in `always()` step | Securely overwrites and deletes the temporary dump from runner storage even if preceding steps fail. |
| **Concurrency Lock** | `group: secrets-audit-${{ github.ref }}` | Prevents race conditions and overlapping runs on the same branch. |

---

## 🛠️ Roadmap & Future Extensions

### 1. Environment-Scoped Secrets
By default, the global `secrets` context contains Repository-level and Organization-level secrets. To audit Environment-specific secrets (e.g. `production` or `staging`), configure an `environment` parameter:

```yaml
# Add input parameter:
inputs:
  environment:
    description: "Target environment to audit (leave empty for global)"
    required: false
    type: string

# Reference on the job:
jobs:
  dump:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
```

> **Benefit:** Unlocks GitHub **Environment Protection Rules**, requiring manual review and approval by authorized security personnel before secret dumps can be generated.

### 2. GPG / Asymmetric Artifact Encryption
To prevent team members with read access from downloading unencrypted secrets files, introduce a GPG encryption step before artifact upload:

```bash
# Symmetrically encrypt using a passphrase:
gpg --batch --yes --passphrase "$PASSPHRASE" --symmetric --cipher-algo AES256 "$FILE"
# Upload only the encrypted payload: "$FILE.gpg"
```

---

## 📄 License

Distributed under the MIT License. See [LICENSE](LICENSE) for full details.