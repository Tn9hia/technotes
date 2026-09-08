---
title: Terraform
tags:
  - terraform
  - iac
  - infrastructure-as-code
  - index
date: 2026-04-27
---

# Terraform

Terraform (HashiCorp) là IaC tool dùng HCL (HashiCorp Configuration Language) để define và provision infrastructure theo phong cách declarative. Terraform so sánh desired state (code) với current state (tfstate) và tạo ra execution plan để đồng bộ.

## Contents

| File | Nội dung |
|------|----------|
| [[Core Concepts]] | Providers, resources, data sources, variables, outputs, locals, functions, count/for_each, dynamic blocks, modules, workspaces, provisioners |
| [[State Management]] | tfstate structure, remote backends (S3/MinIO/GitLab/TF Cloud), state locking, state commands (mv/rm/import), moved block, refresh-only, cross-workspace data |
| [[Modules & Patterns]] | Module design principles, repo structure, module composition, Terragrunt, Kubernetes+Helm provider, Vault integration |
| [[CI-CD & Operations]] | Validation & formatting, security scanning (Checkov/tfsec), GitHub Actions pipeline, GitLab CI, Atlantis, Terratest, operations cheat sheet |
| [[CLI Reference]] | Bảng flag tra cứu nhanh, environment variables, debug recipes |

---

## Quick Reference

### Workflow cơ bản

```bash
terraform init          # download providers + modules
terraform fmt           # format code
terraform validate      # syntax + logic check (không cần credentials)
terraform plan          # preview changes
terraform apply         # apply changes (confirm)
terraform apply -auto-approve
terraform destroy       # destroy all (cẩn thận!)
```

### Plan options thường dùng

```bash
terraform plan -out=tfplan                    # save plan file
terraform plan -target='module.networking'    # chỉ plan specific module
terraform plan -var="env=production"          # override variable
terraform plan -refresh-only                  # xem drift
terraform plan -destroy                       # preview destroy
```

### State inspection

```bash
terraform state list
terraform state show 'module.networking.aws_vpc.main'
terraform output
terraform output -json
```

### State surgery

```bash
terraform state mv 'aws_vpc.main' 'module.networking.aws_vpc.main'
terraform state rm 'module.old'
terraform import aws_vpc.main vpc-0abc123

# Terraform 1.5+ — declarative import
# thêm import block vào .tf file, rồi:
terraform plan -generate-config-out=generated.tf
terraform apply
```

### Debug

```bash
TF_LOG=DEBUG terraform plan 2> debug.log
TF_LOG_PATH=/tmp/tf.log TF_LOG=TRACE terraform apply
```

### Apply options

```bash
terraform apply -target='aws_instance.web'    # specific resource
terraform apply -replace='aws_instance.web'   # force recreate (replaces taint)
terraform apply -parallelism=20
```

---

## Folder Structure chuẩn

```
infrastructure/
├── modules/                    ← reusable (không có env config)
│   ├── networking/
│   ├── kubernetes-cluster/
│   └── postgres-rds/
└── environments/
    ├── production/
    │   ├── networking/         ← separate state per component
    │   ├── kubernetes/
    │   └── apps/
    ├── staging/
    └── dev/
```

Mỗi component directory có: `main.tf`, `variables.tf`, `outputs.tf`, `backend.tf`, `terraform.tfvars`.

---

## CI/CD Overview

```
PR opened
  │
  ├─ terraform fmt -check      ← CI block nếu không formatted
  ├─ terraform validate
  ├─ checkov / tfsec           ← security scan
  ├─ terraform plan            ← post as PR comment
  │
Review + Approve
  │
Merge → main
  │
  └─ terraform apply           ← auto hoặc Atlantis
```

**Atlantis:** comment `atlantis plan` / `atlantis apply` trong PR → không cần config CI riêng.

---

## Cross-References

- Vault secrets trong Terraform: [[Kubernetes Integration]]
- PostgreSQL CloudNativePG provisioning: [[PostgreSQL on Kubernetes]]
- Kubernetes cluster provisioning (EKS/GKE): [[Production Ops]]
- ArgoCD + Terraform GitOps: [[ArgoCD]]
- Ansible vs Terraform: Terraform = provisioning infrastructure; Ansible = configuration management
