---
title: Terraform CLI Reference
tags:
  - terraform
  - cli
  - cheatsheet
  - reference
date: 2026-07-17
---

# Terraform CLI Reference

> Bảng tra cứu nhanh flag, environment variable, và debug recipe. Xem [[Core Concepts]], [[State Management]], [[CI-CD & Operations]] cho nội dung chi tiết theo chủ đề.

---

## Useful Flags Reference

| Flag | Áp dụng cho | Mô tả |
|------|------------|-------|
| `-auto-approve` | apply, destroy | Bỏ qua confirm |
| `-target=<addr>` | plan, apply, destroy | Chỉ thao tác 1 resource/module |
| `-var="key=val"` | plan, apply | Override biến inline |
| `-var-file=x.tfvars` | plan, apply | Load biến từ file |
| `-out=tfplan` | plan | Lưu plan ra file |
| `-input=false` | init, plan, apply | Không hỏi input (CI/CD) |
| `-refresh=false` | plan, apply | Bỏ qua refresh state |
| `-replace=<addr>` | plan, apply | Buộc recreate resource (thay thế `taint`) |
| `-parallelism=N` | plan, apply | Số resource xử lý song song (default 10) |
| `-detailed-exitcode` | plan | Exit code 2 nếu có changes |
| `-json` | show, output | Output dạng JSON |
| `-raw` | output | Raw string (bash-friendly) |
| `-recursive` | fmt | Apply cho tất cả subfolders |
| `-check` | fmt | Chỉ kiểm tra, không sửa |
| `-upgrade` | init | Upgrade providers |
| `-reconfigure` | init | Reset backend config |

---

## Environment Variables

```bash
# Logging
export TF_LOG=DEBUG                  # TRACE | DEBUG | INFO | WARN | ERROR
export TF_LOG_PATH=./tf.log          # Ghi log ra file

# Input variables (tự động map vào var.*)
export TF_VAR_region="us-east-1"     # → var.region = "us-east-1"
export TF_VAR_db_password="secret"   # → var.db_password = "secret"

# Workspace
export TF_WORKSPACE=production       # Override workspace

# CLI config
export TF_CLI_ARGS_plan="-input=false -refresh=false"  # Default args cho plan
export TF_CLI_ARGS_apply="-input=false"                # Default args cho apply
export TF_IN_AUTOMATION=1            # Suppress interactive prompts (CI mode)

# Plugin cache (tránh download lại providers)
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"
```

---

## Quick Debug Recipes

```bash
# "Tại sao resource này sẽ bị replace?"
terraform plan -out=tfplan
terraform show -json tfplan | jq '.resource_changes[] | select(.change.actions | contains(["delete"])) | {address: .address, before: .change.before, after: .change.after}'

# "Resource nào đang trong state?"
terraform state list

# "Chi tiết resource X trông như thế nào trong state?"
terraform state show aws_instance.web

# "Output value là gì?"
terraform output -json | jq '.'

# "Providers nào đang được dùng và version bao nhiêu?"
terraform providers
terraform version

# "Xem log chi tiết khi plan bị fail"
TF_LOG=DEBUG terraform plan 2>&1 | grep -E "ERROR|WARN|RequestId"

# "State bị lock, lấy lock ID"
# Lock ID xuất hiện trong error message:
# Error: Error locking state: ... Lock Info: ID: <LOCK_ID>
terraform force-unlock <LOCK_ID>
```
