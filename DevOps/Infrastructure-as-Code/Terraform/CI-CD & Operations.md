---
title: Terraform CI/CD & Operations
tags:
  - terraform
  - cicd
  - atlantis
  - testing
  - operations
date: 2026-04-29
---

# Terraform CI/CD & Operations

## Workflow Chuẩn

```
Developer
  │
  ├─ git checkout -b feature/add-subnet
  ├─ viết .tf files
  ├─ terraform fmt && terraform validate (local)
  ├─ git push → open PR
  │
  ▼
CI Pipeline (trên PR)
  ├─ terraform fmt -check       ← fail nếu không formatted
  ├─ terraform validate         ← syntax + logic check
  ├─ tfsec / checkov            ← security scan
  ├─ terraform plan             ← post plan as PR comment
  └─ (block merge nếu fail)
  │
  ▼
Code Review
  ├─ Review plan output trong PR comment
  ├─ Approve PR
  │
  ▼
Merge → main
  │
  ▼
CI Pipeline (trên main)
  └─ terraform apply            ← auto-apply hoặc manual trigger
```

---

## Validation & Formatting

```bash
# Format tất cả .tf files (recursive)
terraform fmt -recursive

# Check formatting (exit 1 nếu không đúng — dùng trong CI)
terraform fmt -check -recursive -diff

# Validate cấu trúc và logic (không cần cloud credentials)
terraform validate

# Plan (cần credentials)
terraform plan -out=tfplan        # save plan file
terraform show -json tfplan       # xem plan dưới dạng JSON

# Apply saved plan (không ask for confirmation)
terraform apply tfplan

# Destroy preview
terraform plan -destroy
```

### Variable Validation trong Config

```hcl
variable "environment" {
  type = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "environment must be one of: dev, staging, production."
  }
}

variable "instance_type" {
  type = string

  validation {
    condition     = can(regex("^t3\\.", var.instance_type))
    error_message = "Only t3.* instance types are allowed."
  }
}

variable "cidr_block" {
  type = string

  validation {
    condition     = can(cidrnetmask(var.cidr_block))
    error_message = "cidr_block must be a valid CIDR notation."
  }
}

variable "replica_count" {
  type = number

  validation {
    condition     = var.replica_count >= 1 && var.replica_count <= 10
    error_message = "replica_count must be between 1 and 10."
  }
}
```

---

## Security Scanning

### Checkov

```bash
# Install
pip install checkov

# Scan Terraform directory
checkov -d ./terraform/ --framework terraform

# Scan và output SARIF (upload to GitHub Security)
checkov -d ./terraform/ \
  --output sarif \
  --output-file checkov-results.sarif \
  --soft-fail-on LOW,MEDIUM    # pass cho LOW/MEDIUM, fail cho HIGH/CRITICAL

# Skip specific check (false positive hoặc accepted risk)
checkov -d ./terraform/ --skip-check CKV_AWS_123

# Inline suppression trong HCL
resource "aws_s3_bucket" "internal" {
  bucket = "internal-data"

  #checkov:skip=CKV_AWS_18:Access logging not required for internal bucket
  #checkov:skip=CKV_AWS_144:Cross-region replication not needed for dev
}
```

### tfsec

```bash
# Install
brew install tfsec

# Scan
tfsec .
tfsec . --minimum-severity HIGH

# Output formats
tfsec . --format json | jq '.results[] | select(.severity == "CRITICAL")'
tfsec . --format sarif > tfsec-results.sarif

# Ignore trong code
resource "aws_security_group_rule" "allow_all_egress" {
  #tfsec:ignore:aws-ec2-no-public-egress-sgr
  type        = "egress"
  cidr_blocks = ["0.0.0.0/0"]
}
```

---

## GitHub Actions Pipeline

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [main]
  pull_request:
    paths:
      - 'infrastructure/**'
      - '.github/workflows/terraform.yml'

env:
  TF_VERSION: "1.7.5"
  AWS_REGION: "ap-southeast-1"
  WORKING_DIR: "infrastructure/environments/production/networking"

jobs:
  # ─── Validate & Scan (luôn chạy) ───
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Format check
        run: terraform fmt -check -recursive
        working-directory: infrastructure/

      - name: Validate
        run: |
          terraform init -backend=false
          terraform validate
        working-directory: ${{ env.WORKING_DIR }}

      - name: Checkov Security Scan
        uses: bridgecrewio/checkov-action@master
        with:
          directory: infrastructure/
          framework: terraform
          soft_fail: false
          output_format: sarif
          output_file_path: checkov.sarif

      - name: Upload Checkov results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: checkov.sarif

  # ─── Plan (chỉ trên PR) ───
  plan:
    runs-on: ubuntu-latest
    needs: validate
    if: github.event_name == 'pull_request'
    permissions:
      pull-requests: write
      id-token: write    # OIDC auth với AWS

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/TerraformPlanRole
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: terraform init
        working-directory: ${{ env.WORKING_DIR }}

      - name: Terraform Plan
        id: plan
        run: |
          terraform plan -no-color -out=tfplan 2>&1 | tee plan.txt
          echo "exitcode=$?" >> $GITHUB_OUTPUT
        working-directory: ${{ env.WORKING_DIR }}
        continue-on-error: true

      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('${{ env.WORKING_DIR }}/plan.txt', 'utf8');
            const maxLength = 65000;
            const truncated = plan.length > maxLength
              ? plan.substring(0, maxLength) + '\n... (truncated)'
              : plan;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Terraform Plan — \`${{ env.WORKING_DIR }}\`
            \`\`\`hcl
            ${truncated}
            \`\`\``
            });

      - name: Fail if plan failed
        if: steps.plan.outputs.exitcode != '0'
        run: exit 1

  # ─── Apply (chỉ trên merge tới main) ───
  apply:
    runs-on: ubuntu-latest
    needs: validate
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production    # require manual approval nếu configured
    permissions:
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/TerraformApplyRole
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: terraform init
        working-directory: ${{ env.WORKING_DIR }}

      - name: Terraform Apply
        run: terraform apply -auto-approve
        working-directory: ${{ env.WORKING_DIR }}
```

### GitLab CI

```yaml
# .gitlab-ci.yml
variables:
  TF_ROOT: "infrastructure/environments/production/networking"
  TF_VERSION: "1.7.5"

stages:
  - validate
  - plan
  - apply

image:
  name: hashicorp/terraform:$TF_VERSION
  entrypoint: [""]

cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - $TF_ROOT/.terraform/

fmt:
  stage: validate
  script:
    - cd $TF_ROOT
    - terraform fmt -check -recursive

validate:
  stage: validate
  script:
    - cd $TF_ROOT
    - terraform init -backend=false
    - terraform validate

plan:
  stage: plan
  script:
    - cd $TF_ROOT
    - terraform init
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - $TF_ROOT/tfplan
    expire_in: 7 days
  environment:
    name: production
    action: prepare

apply:
  stage: apply
  script:
    - cd $TF_ROOT
    - terraform init
    - terraform apply tfplan
  dependencies:
    - plan
  when: manual          # require manual trigger
  only:
    - main
  environment:
    name: production
    action: start
```

---

## Atlantis — GitOps cho Terraform

Atlantis tự động chạy `plan` khi PR mở và `apply` khi PR merge — không cần config CI riêng.

### Install

```yaml
# atlantis-deployment.yaml (trong K8s)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: atlantis
spec:
  template:
    spec:
      containers:
        - name: atlantis
          image: ghcr.io/runatlantis/atlantis:v0.28.0
          args:
            - server
            - --gh-user=atlantis-bot
            - --gh-token=$(GH_TOKEN)
            - --gh-webhook-secret=$(GH_WEBHOOK_SECRET)
            - --repo-allowlist=github.com/myorg/*
            - --atlantis-url=https://atlantis.internal
          env:
            - name: GH_TOKEN
              valueFrom:
                secretKeyRef:
                  name: atlantis-secrets
                  key: github-token
            - name: AWS_ROLE_ARN
              value: arn:aws:iam::123456789:role/AtlantisRole
```

### atlantis.yaml

```yaml
# atlantis.yaml — ở root của repo
version: 3
automerge: false        # không tự merge PR sau apply
delete_source_branch_on_merge: false

projects:
  - name: production-networking
    dir: infrastructure/environments/production/networking
    workspace: default
    autoplan:
      when_modified:
        - "**/*.tf"
        - "**/*.tfvars"
        - "../../../modules/networking/**/*.tf"
      enabled: true
    apply_requirements:
      - approved              # cần approval trước khi apply
      - mergeable             # PR phải mergeable (no conflicts)

  - name: production-kubernetes
    dir: infrastructure/environments/production/kubernetes
    depends_on:
      - production-networking    # plan sau khi networking apply xong

  - name: staging-networking
    dir: infrastructure/environments/staging/networking
    autoplan:
      enabled: true
    apply_requirements: []      # staging: không cần approval
```

### Atlantis Workflow

```
Developer opens PR
  │
  ├─ Atlantis detects changed .tf files
  ├─ Atlantis runs: terraform plan
  ├─ Posts plan as PR comment
  │
Reviewer sees plan → approves PR
  │
  ├─ PR author comments: atlantis apply
  ├─ Atlantis runs: terraform apply
  ├─ Posts apply output as PR comment
  │
  └─ Merge PR
```

```bash
# Atlantis commands trong PR comment
atlantis plan                   # run plan cho tất cả projects trong PR
atlantis plan -p production-networking   # plan specific project
atlantis apply                  # apply sau khi plan + approval
atlantis apply -p production-networking
atlantis unlock                 # unlock nếu plan stuck
```

---

## Terratest — Testing

Terratest cho phép viết integration tests cho Terraform modules bằng Go.

```go
// modules/networking/tests/networking_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestNetworkingModule(t *testing.T) {
    t.Parallel()

    region := "ap-southeast-1"

    terraformOptions := &terraform.Options{
        // Path to module
        TerraformDir: "../",

        // Input variables
        Vars: map[string]interface{}{
            "environment": "test",
            "vpc_cidr":    "10.99.0.0/16",
            "az_count":    2,
            "enable_nat_gw": false,
        },

        // AWS region
        EnvVars: map[string]string{
            "AWS_REGION": region,
        },
    }

    // Cleanup after test
    defer terraform.Destroy(t, terraformOptions)

    // Init và apply
    terraform.InitAndApply(t, terraformOptions)

    // Get outputs
    vpcId         := terraform.Output(t, terraformOptions, "vpc_id")
    privateSubnets := terraform.OutputList(t, terraformOptions, "private_subnets")

    // Assert
    assert.NotEmpty(t, vpcId)
    assert.Len(t, privateSubnets, 2)

    // Verify AWS resources directly
    vpc := aws.GetVpcById(t, vpcId, region)
    assert.Equal(t, "10.99.0.0/16", aws.GetVpcById(t, vpcId, region).CidrBlock)

    // Verify subnets are in correct AZs
    for _, subnetId := range privateSubnets {
        subnet := aws.GetSubnetById(t, subnetId, region)
        assert.False(t, subnet.MapPublicIpOnLaunch)
    }
}
```

```bash
# Chạy tests
cd modules/networking/tests
go test -v -timeout 30m -run TestNetworkingModule

# Parallel tests (nếu có nhiều test files)
go test -v -timeout 60m -parallel 5 ./...
```

---

## Operations Cheat Sheet

```bash
# ─── Daily operations ───
terraform init          # download providers + modules
terraform fmt           # format code
terraform validate      # syntax check
terraform plan          # preview changes
terraform apply         # apply changes
terraform destroy       # destroy all (cẩn thận!)

# ─── State inspection ───
terraform state list
terraform state show '<resource>'
terraform output
terraform output -json

# ─── State surgery ───
terraform state mv '<old>' '<new>'
terraform state rm '<resource>'
terraform import '<address>' '<id>'

# ─── Debugging ───
TF_LOG=DEBUG terraform plan 2> debug.log
TF_LOG_PATH=/tmp/tf-debug.log TF_LOG=TRACE terraform apply

# ─── Plan options ───
terraform plan -target='module.networking'    # chỉ plan specific module
terraform plan -var="instance_count=5"
terraform plan -refresh=false                 # skip refresh (faster, less accurate)
terraform plan -compact-warnings              # compact warning output

# ─── Apply options ───
terraform apply -auto-approve              # không hỏi confirm
terraform apply -target='aws_instance.web' # chỉ apply specific resource
terraform apply -parallelism=20            # tăng concurrent operations

# ─── Forced replace ───
terraform apply -replace='aws_instance.web["web-1"]'
# equivalent của taint trong Terraform 0.x
```

---

## Gotchas

- **`terraform apply -target` và dependencies**: `-target` chỉ apply target và direct dependencies. Có thể tạo state inconsistency nếu dùng thường xuyên. Chỉ dùng cho emergency hotfix, không phải normal workflow.
- **Parallel apply và race conditions**: Terraform tự tính dependency graph. Nhưng với external dependencies (ví dụ: 2 modules đều configure same S3 bucket) → race condition. Cần `depends_on` hoặc sequence bằng `terraform_data`.
- **Atlantis và long plans**: Atlantis có timeout cho plan (default 30s/60s). Với nhiều resources → timeout → plan fail. Tune `--tf-distribution=parallelism` và Atlantis `parallel_plan_enabled`.
- **`terraform destroy` trong production**: Không có "undo". Luôn dùng `prevent_destroy = true` cho critical resources. Require manual step + backup trước khi destroy.
- **Provider auth trong CI**: Dùng OIDC (GitHub Actions → AWS, GitLab → AWS) thay vì long-lived access keys. OIDC tokens expire sau job → không thể bị leaked. Không bao giờ hardcode AWS credentials trong workflow files.
- **Checkov false positives**: Một số Checkov rules không phù hợp với context (internal VPC resources không cần TLS, dev buckets không cần replication). Maintain `.checkov.yaml` hoặc inline suppression với comment giải thích. Document tất cả suppressions để audit.
