# KICS Security Governance Workflow Plan

This document defines how KICS (IaC security scanning) is integrated into the repository to ensure continuous Terraform security validation across all environments.

It introduces:

- Scheduled recurring scans
- Pull Request (PR) security scans
- Merge blocking on CRITICAL findings
- Automated alerts on CRITICAL issues

---

# Objectives

1. Shift security left (scan before merge).
2. Prevent insecure Terraform from reaching `main`.
3. Continuously scan the full repository.
4. Alert security/DevOps on CRITICAL misconfigurations.
5. Maintain visibility via SARIF reports in GitHub Artifact.

---

# Scan Types

We implement **two KICS execution modes**:

| Scan Type | Trigger | Scope | Blocks Merge? | Alerts? |
|------------|----------|--------|---------------|---------|
| PR Scan | pull_request | Changed Terraform code | YES (on CRITICAL) | YES |
| Scheduled Scan | schedule (cron) | Entire repository | No | YES |

---

# Pull Request KICS Scan (Blocking)

## Trigger

```yaml
on:
  pull_request:
    branches:
      - main
      - develop
      ...
```

## Scope

- Scan:
  - `environments/`
  - `modules/`
- Optionally limit to changed files only.

## Behavior

- Run KICS before allowing merge.
- Fail workflow if any CRITICAL findings are detected.
- Upload SARIF report to GitHub Security tab.
- Send alert on CRITICAL findings.

## PR Scan Flowchart

```text
          ┌─────────────────────────────┐
          │ Pull Request Created/Update │
          └──────────────┬──────────────┘
                         │
                         ▼
          ┌─────────────────────────────┐
          │      Checkout PR Code       │
          └──────────────┬──────────────┘
                         │
                         ▼
          ┌─────────────────────────────┐
          │        Run KICS Scan        │
          │      (envs + modules)       │
          └──────────────┬──────────────┘
                         │
                         ▼
          ┌─────────────────────────────┐
          │    Any CRITICAL Findings?   │
          └──────────────┬──────────────┘
                         │
           ┌─────────────┴─────────────┐
           │                           │
           ▼                           ▼
        YES (Found)                  NO
           │                           │
           ▼                           ▼
  ┌─────────────────────┐      ┌─────────────────────┐
  │    Fail Workflow    │      │ Upload SARIF Report │
  │    Block PR Merge   │      │     Allow Merge     │
  │    Send Alert       │      └─────────────────────┘
  └─────────────────────┘
```

## Enforcement

Use **Branch Protection Rules**:

- Require status check: `kics-pr-scan`
- Prevent merge if workflow fails

This guarantees:
- No CRITICAL IaC misconfigurations reach protected branches.

---

# Scheduled KICS Scan (Continuous Security)

## Trigger

```yaml
on:
  schedule:
    - cron: '0 2 * * 1'   # Every Monday at 02:00 UTC
```

## Scope

- Entire repository:
  - environments/
  - modules/
  - lambda/ (if scanning IaC references)

## Purpose

- Detect:
  - Legacy insecure configurations
  - Newly introduced risky patterns
  - Misconfigurations introduced outside PR flow
- Provide continuous compliance posture

## Scheduled Scan Flowchart

```text
          ┌─────────────────────────────┐
          │ Scheduled Trigger (Cron)    │
          └──────────────┬──────────────┘
                         │
                         ▼
          ┌─────────────────────────────┐
          │ Checkout Repository         │
          └──────────────┬──────────────┘
                         │
                         ▼
          ┌─────────────────────────────┐
          │ Run Full KICS Scan          │
          │ (All Terraform Code)        │
          └──────────────┬──────────────┘
                         │
                         ▼
          ┌─────────────────────────────┐
          │ Any CRITICAL Findings?      │
          └──────────────┬──────────────┘
                         │
           ┌─────────────┴─────────────┐
           │                           │
           ▼                           ▼
        YES (Found)                  NO
           │                           │
           ▼                           ▼
  ┌─────────────────────┐      ┌─────────────────────┐
  │     Send Alert      │      │ Upload SARIF Report │
  └─────────────────────┘      |        End          │
                               └─────────────────────┘
```

Scheduled scans do NOT block merges but ensure continuous monitoring.

---

# Recommended KICS Execution Configuration

Run with:

```bash
kics scan \
  -p . \
  --report-formats "sarif,json" \
  --fail-on HIGH \
  --severity-threshold CRITICAL
```

Recommended enforcement:

- PR scans:
  - `--fail-on CRITICAL`
- Scheduled scans:
  - Do not fail pipeline
  - Send alert instead

---

# Security Gate Position in CI/CD

## For PRs

Security scan must run:

Before:
- Terraform Plan
- Terraform Apply
- Merge to main

Final order:

```
PR Created
  ↓
Terraform Validate
  ↓
KICS Scan (BLOCKING)
  ↓
If Clean → Allow Merge
```

---

# Governance Model

| Environment | PR Scan | Scheduled Scan | Merge Blocking |
|------------|----------|----------------|----------------|
| dev | Yes | Yes | Yes |
| uat | Yes | Yes | Yes |
| prod | Yes | Yes | Yes |

Security policy is enforced equally across all environments.

---

# Architecture Alignment With Repository Structure

KICS scans:

- `environments/dev`
- `environments/uat`
- `environments/prod`
- `modules/`

Because:

- Each environment is a Terraform root.
- Each module can introduce security risks reused across environments.
- Accounts are isolated, but code is shared.

---

# Complete Governance Flow

```text
                       ┌─────────────────────────────┐
                       │        Developer PR         │
                       └──────────────┬──────────────┘
                                      │
                                      ▼
                           ┌──────────────────────┐
                           │     KICS PR Scan     │
                           └──────────┬───────────┘
                                      │
                        ┌─────────────┴───────────────┐
                        │                             │
                        ▼                             ▼
                CRITICAL Found?                    Clean?
                        │                             │
                        ▼                             ▼
              Block Merge + Alert                Allow Merge
                        │
                        ▼
                       END

--------------------------------------------------------------

                     Scheduled Weekly Scan
                              │
                              ▼
                     Full Repo KICS Scan
                              │
                              ▼
                     CRITICAL Found?
                              │
                 ┌────────────┴────────────┐
                 ▼                         ▼
          Send Alert + Issue           Upload Report
```

---

# Resulting Security Posture

✔ No insecure Terraform reaches protected branches  
✔ Continuous compliance validation  
✔ Enterprise-ready governance  
✔ Clear separation between prevention (PR) and monitoring (Scheduled)  
✔ Fully aligned with multi-account environment isolation  

