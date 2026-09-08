---
title: Packer CLI Reference
tags:
  - packer
  - cli
  - cheatsheet
  - reference
date: 2026-07-18
---

# Packer CLI Reference

> Bảng tra cứu nhanh lệnh, flag, và debug recipe. Xem [[Core Concepts]], [[Provisioners & Post-Processors]] cho nội dung chi tiết theo chủ đề.

---

## Core Commands

| Lệnh | Mô tả |
|------|-------|
| `packer init template.pkr.hcl` | Tải plugin khai báo trong `required_plugins` |
| `packer fmt template.pkr.hcl` | Format code |
| `packer fmt -check template.pkr.hcl` | Chỉ kiểm tra, không sửa (dùng trong CI) |
| `packer validate template.pkr.hcl` | Syntax + logic check, không build thật |
| `packer build template.pkr.hcl` | Build image |
| `packer inspect template.pkr.hcl` | In ra variables, builders, provisioners khai báo trong template |
| `packer console` | REPL để test expression HCL2 (giống `terraform console`) |

## Build Options

| Flag | Mô tả |
|------|-------|
| `-var="key=val"` | Override variable inline |
| `-var-file=prod.pkrvars.hcl` | Load variables từ file |
| `-only=<builder>.<name>` | Chỉ build source cụ thể |
| `-except=<builder>.<name>` | Build tất cả trừ source chỉ định |
| `-force` | Xoá artifact cũ trùng tên trước khi build (VD box VirtualBox cũ) |
| `-on-error=abort\|cleanup\|ask` | Hành động khi provisioner fail (default: cleanup) |
| `-debug` | Build từng bước, dừng lại chờ Enter sau mỗi bước (debug provisioner) |
| `-parallel-builds=N` | Số source build song song (default: không giới hạn) |
| `-color=false` | Tắt màu output (CI log) |

## Plugin Management

```bash
packer plugins install github.com/hashicorp/ansible
packer plugins installed
packer plugins remove github.com/hashicorp/ansible
```

---

## Environment Variables

```bash
export PACKER_LOG=1                      # bật logging
export PACKER_LOG_PATH="./packer.log"    # ghi log ra file

export PKR_VAR_app_version="1.4.0"       # tương đương TF_VAR_ — map vào var.app_version
export PKR_VAR_aws_region="ap-southeast-1"

export PACKER_CACHE_DIR="$HOME/.cache/packer"   # cache ISO/image đã download
export PACKER_CONFIG_DIR="$HOME/.packer.d"      # nơi lưu plugin đã cài
```

---

## Quick Debug Recipes

```bash
# "Template có lỗi cú pháp/logic gì không, trước khi build thật"
packer validate template.pkr.hcl

# "Xem toàn bộ config đã resolve (biến, source) mà không build"
packer inspect template.pkr.hcl

# "Build lỗi giữa chừng, muốn xem chi tiết"
PACKER_LOG=1 packer build template.pkr.hcl 2>&1 | tee packer-debug.log

# "Muốn dừng lại xem instance tạm trước khi Packer cleanup, để SSH vào debug"
packer build -debug template.pkr.hcl
# Packer in ra SSH command để connect vào instance tạm giữa các bước

# "Provisioner fail, giữ lại instance tạm để debug thay vì tự xoá"
packer build -on-error=abort template.pkr.hcl

# "Build lại nhưng artifact cũ (box/AMI tag) đã tồn tại, bị chặn"
packer build -force template.pkr.hcl

# "Chỉ build 1 target trong nhiều source (VD chỉ AMI, bỏ qua .box)"
packer build -only="amazon-ebs.prod" template.pkr.hcl
```
