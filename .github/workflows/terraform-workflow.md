# Terraform GitHub Actions Workflow Design

## Workflow Trigger

The workflow is manually triggered using `workflow_dispatch`.

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Select environment"
        required: true
        type: choice
        options:
          - dev
          - uat
          - prod

      action:
        description: "Select Terraform action"
        required: true
        type: choice
        options:
          - plan
          - apply
          - destroy
```

---

## High-Level Workflow Architecture

The workflow is designed in structured phases:

1. Validate inputs
2. Select environment directory
3. Configure AWS credentials for selected environment
4. Terraform init
5. Terraform validate & fmt
6. Terraform plan
7. Conditional apply or destroy
8. Production approval (if prod)

---

## Flowchart

```text
               ┌────────────────────────────┐
               │  Manual Trigger (Dispatch) │
               │  Select: env + action      │
               └──────────────┬─────────────┘
                              │
                              ▼
               ┌────────────────────────────┐
               │ Validate Inputs            │
               │ (env in dev/uat/prod)      │
               │ (action allowed)           │
               └──────────────┬─────────────┘
                              │
                              ▼
               ┌────────────────────────────┐
               │ Set Environment Variables  │
               │ WORKDIR = environments/env │
               └──────────────┬─────────────┘
                              │
                              ▼
               ┌────────────────────────────┐
               │ KICS Terraform Scan        │
               │ (Fail on High/Critical)    │
               └──────────────┬─────────────┘
                              │
                              ▼
               ┌────────────────────────────┐
               │ Configure AWS Credentials  │
               │ (OIDC or secrets)          │
               │ Based on selected env      │
               └──────────────┬─────────────┘
                              │
                              ▼
               ┌────────────────────────────┐
               │ Terraform Init             │
               │ (uses env/backend.tf)      │
               └──────────────┬─────────────┘
                              │
                              ▼
               ┌────────────────────────────┐
               │ Terraform Validate         │
               │ Terraform Format Check     │
               └──────────────┬─────────────┘
                              │
                              ▼
               ┌────────────────────────────┐
               │ Terraform Plan             │
               │ -var-file=env.tfvars       │
               │ Save plan output (tfplan)  │
               └──────────────┬─────────────┘
                              │
                 ┌────────────┴─────────────┐
                 │                          │
                 ▼                          ▼
          action == plan            action == apply/destroy?
                 │                          │
                 ▼                          ▼
        Upload Plan Artifact        Is env == prod?
                 │                          │
                 ▼                          ▼
                END               Require Manual Approval?
                                            │
                                            ▼
                              ┌─────────────────────────┐
                              │   GitHub Environment    │
                              │   Protection Rule       │
                              └─────────────┬───────────┘
                                            │
                                            ▼
                              ┌─────────────────────────┐
                              │ Terraform Apply/Destroy │
                              │ Using Saved Plan        │
                              └─────────────┬───────────┘
                                            │
                                            ▼
                                           END
```

---

## Job Structure Design

The workflow should be structured into three logical jobs.

### 1. Validate Job

**Purpose:**
- Validate workflow inputs
- Set working directory
- Check Terraform formatting
- Run `terraform validate`

**Steps:**
- Checkout repository
- Set working directory to `environments/${{ inputs.environment }}`
- Run:

```bash
terraform fmt -check
terraform validate
```

---

### 2. Plan Job

**Purpose:**
- Authenticate to correct AWS account
- Initialize Terraform backend
- Generate execution plan
- Upload plan artifact

**Steps:**
- Configure AWS credentials (OIDC recommended)
- Change directory to selected environment
- Run:

```bash
terraform init
terraform plan -var-file=<env>.tfvars -out=tfplan
```

- Upload `tfplan` as artifact

**This ensures:**
- Plan is preserved
- Apply uses exact reviewed plan
- No drift between plan and apply

---

### 3. Apply or Destroy Job

Runs only if:

```
action != plan
```

**Steps:**
- Download plan artifact
- If environment == prod:
  - Require manual approval (GitHub Environment protection)
- Run:

```bash
terraform apply tfplan
```

OR

```bash
terraform destroy -var-file=<env>.tfvars
```

---

## Terraform Execution Model

For selected environment:

```bash
cd environments/<env>
terraform init
terraform plan -var-file=<env>.tfvars -out=tfplan
terraform apply tfplan
```

Backend configuration automatically uses:
- Correct S3 bucket
- Correct AWS account
- Correct state isolation

---

## Plan Artifact Strategy (Best Practice)

Always:

```bash
terraform plan -out=tfplan
```

Upload artifact.

For apply:

```bash
terraform apply tfplan
```

**Advantages:**
- Apply uses reviewed plan
- No drift
- Auditable
- Enterprise compliant

---

## Complete Execution Summary

```
Trigger
  ↓
Validate Inputs
  ↓
Select AWS Account (based on environment)
  ↓
Terraform Init
  ↓
Terraform Validate + Format Check
  ↓
Terraform Plan
  ↓
If PLAN → End
If APPLY/DESTROY →
        If PROD → Require Approval
        Execute Action
  ↓
Complete
```