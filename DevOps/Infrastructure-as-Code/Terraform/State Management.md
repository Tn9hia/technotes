---
title: Terraform State Management
tags:
  - terraform
  - state
  - backend
  - remote-state
date: 2026-04-29
---

# Terraform State Management

## terraform.tfstate — Cấu trúc

State file là "source of truth" về những resources Terraform đang quản lý.

```json
{
  "version": 4,
  "terraform_version": "1.7.5",
  "serial": 42,
  "lineage": "abc123-def456-...",
  "outputs": {
    "vpc_id": {
      "value": "vpc-0abc123",
      "type": "string",
      "sensitive": false
    }
  },
  "resources": [
    {
      "module": "module.networking",
      "mode": "managed",
      "type": "aws_vpc",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "vpc-0abc123",
            "cidr_block": "10.0.0.0/16",
            "tags": { "Environment": "production" }
          }
        }
      ]
    }
  ]
}
```

**Quan trọng:**
- `serial`: tăng mỗi lần state thay đổi — dùng để detect conflicts
- `lineage`: unique ID cho state — backend dùng để prevent mixing states
- Sensitive values: **plaintext trong state** dù `sensitive = true` trong config

---

## Remote Backend

Local state (`terraform.tfstate`) không phù hợp cho team — không có locking, không share được.

### S3 + DynamoDB (AWS)

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "production/networking/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true                        # server-side encryption
    kms_key_id     = "arn:aws:kms:..."          # CMK encryption
    dynamodb_table = "terraform-state-lock"     # locking table

    # Assume role (cross-account)
    role_arn = "arn:aws:iam::123456789:role/TerraformBackendRole"
  }
}
```

```bash
# Tạo S3 bucket và DynamoDB table (bootstrap — làm 1 lần)
aws s3api create-bucket \
  --bucket mycompany-terraform-state \
  --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1

# Enable versioning (rollback state)
aws s3api put-bucket-versioning \
  --bucket mycompany-terraform-state \
  --versioning-configuration Status=Enabled

# Block public access
aws s3api put-public-access-block \
  --bucket mycompany-terraform-state \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# DynamoDB table cho locking (key phải là "LockID")
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-southeast-1
```

### MinIO (On-Premise — S3-compatible)

```hcl
# On-premise với MinIO
terraform {
  backend "s3" {
    bucket                      = "terraform-state"
    key                         = "production/app/terraform.tfstate"
    region                      = "us-east-1"     # MinIO bỏ qua region nhưng cần set
    endpoint                    = "http://minio.internal:9000"
    access_key                  = var.minio_access_key
    secret_key                  = var.minio_secret_key
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
    force_path_style            = true    # MinIO dùng path-style, không virtual-hosted
  }
}
```

### Terraform Cloud / HCP Terraform

```hcl
terraform {
  cloud {
    organization = "myorg"
    workspaces {
      name = "production-networking"
      # hoặc: tags = ["production"]
    }
  }
}
```

### HTTP Backend (tự host — GitLab, etc.)

```hcl
terraform {
  backend "http" {
    address        = "https://gitlab.internal/api/v4/projects/42/terraform/state/production"
    lock_address   = "https://gitlab.internal/api/v4/projects/42/terraform/state/production/lock"
    unlock_address = "https://gitlab.internal/api/v4/projects/42/terraform/state/production/lock"
    username       = "gitlab-ci-token"
    password       = var.gitlab_token
    lock_method    = "POST"
    unlock_method  = "DELETE"
  }
}
```

---

## State Locking

Locking ngăn concurrent applies — hai người apply cùng lúc có thể corrupt state.

```bash
# DynamoDB locking: khi apply, Terraform tạo item trong DynamoDB
# Item: { LockID: "mycompany-terraform-state/production/networking/terraform.tfstate" }
# Khi done: xóa item

# Nếu apply bị interrupt → lock còn lại → lần apply sau bị block
# Force unlock (cẩn thận — chỉ dùng khi chắc chắn không có apply đang chạy)
terraform force-unlock <LOCK_ID>

# Lock ID thấy trong error message:
# Error: Error acquiring the state lock
# Lock ID: "abc123-def456-..."
```

---

## State Commands

### terraform state list / show

```bash
# List tất cả resources trong state
terraform state list
# module.networking.aws_vpc.main
# module.networking.aws_subnet.private[0]
# aws_instance.web["web-1"]

# Show details của một resource
terraform state show 'module.networking.aws_vpc.main'
# resource "aws_vpc" "main" {
#   cidr_block = "10.0.0.0/16"
#   id         = "vpc-0abc123"
#   ...
# }
```

### terraform state mv

```bash
# Rename resource trong state (không destroy/recreate)
# Use case: refactor HCL mà không muốn recreate resource

# Rename resource
terraform state mv aws_instance.web aws_instance.app

# Move resource vào module
terraform state mv aws_vpc.main module.networking.aws_vpc.main

# Move từ module ra
terraform state mv 'module.networking.aws_vpc.main' aws_vpc.main

# Move giữa resources (count → for_each migration)
terraform state mv 'aws_instance.web[0]' 'aws_instance.web["web-1"]'
```

### terraform state rm

```bash
# Remove resource khỏi state (Terraform sẽ không quản lý nữa)
# Resource KHÔNG bị xóa trên thực tế — chỉ xóa khỏi state
# Use case: adopt resource về management của team khác, hoặc stop managing

terraform state rm aws_instance.legacy_server
terraform state rm 'module.old_module.aws_vpc.main'

# Remove tất cả resources trong module
terraform state rm 'module.old_module'
```

### terraform state pull / push

```bash
# Pull current state về local
terraform state pull > backup.tfstate

# Push local state lên remote (nguy hiểm — override remote state)
# Chỉ dùng cho disaster recovery
terraform state push backup.tfstate
```

---

## terraform import

Import existing resources vào Terraform management mà không recreate.

### Legacy import (command)

```bash
# Syntax: terraform import <ADDRESS> <RESOURCE_ID>
terraform import aws_vpc.main vpc-0abc123
terraform import 'aws_instance.web["web-1"]' i-0abc123def456
terraform import 'module.networking.aws_security_group.app' sg-0abc123

# Sau khi import:
# 1. Terraform state có resource
# 2. Phải viết HCL config matching resource (không auto-generate)
# 3. terraform plan phải cho thấy "No changes" nếu config đúng
```

### Import block (Terraform 1.5+ — preferred)

```hcl
# import.tf — declarative import
import {
  to = aws_vpc.main
  id = "vpc-0abc123"
}

import {
  to = module.networking.aws_security_group.app
  id = "sg-0abc123"
}

# Với for_each import
import {
  for_each = {
    "web-1" = "i-0abc123def456"
    "web-2" = "i-0def789abc012"
  }
  id = each.value
  to = aws_instance.web[each.key]
}
```

```bash
# Generate config tự động (Terraform 1.5+)
terraform plan -generate-config-out=generated.tf
# → Terraform viết HCL cho imported resources vào generated.tf
# → Review và clean up generated config trước khi dùng

terraform apply    # apply import
# Sau khi done: xóa import blocks (chỉ cần 1 lần)
```

### Chiến lược import existing infrastructure

```
1. terraform plan -generate-config-out=generated.tf
   → có được draft config

2. Review và refactor generated.tf:
   - Xóa computed attributes (id, arn, etc.)
   - Thêm variables cho dynamic values
   - Organize vào proper files/modules

3. terraform plan → kiểm tra "No changes" hoặc chỉ có acceptable diffs

4. Dọn dẹp: xóa import blocks, commit code
```

---

## moved Block

```hcl
# Khai báo resource đã moved (refactor không destroy)
# Terraform 1.1+

# Rename
moved {
  from = aws_instance.web
  to   = aws_instance.app_server
}

# Move vào module
moved {
  from = aws_vpc.main
  to   = module.networking.aws_vpc.main
}

# count → for_each
moved {
  from = aws_instance.web[0]
  to   = aws_instance.web["web-primary"]
}

# terraform plan sẽ show "moved" action (không phải destroy+create)
# Sau khi apply: xóa moved blocks
```

---

## terraform refresh (deprecated)

```bash
# Sync state với real infrastructure (không change infra)
# DEPRECATED trong Terraform 1.x — dùng:
terraform apply -refresh-only

# Xem drift giữa state và reality
terraform plan -refresh-only
# → thấy những gì bị thay đổi bên ngoài Terraform
```

---

## Sensitive Values trong State

```bash
# State chứa plaintext sensitive values
# → Phải protect state file!

# Best practices:
# 1. Remote backend với encryption (S3 + KMS)
# 2. Restrict access tới state bucket (IAM policies)
# 3. Enable S3 bucket versioning (rollback state)
# 4. Không commit state file vào git (.gitignore)
# 5. Audit log cho state bucket access
```

```hcl
# Partial config: không hardcode credentials trong backend config
# Backend config từ file hoặc env vars
terraform {
  backend "s3" {}    # empty — values từ -backend-config hoặc CLI
}
```

```bash
# Pass backend config lúc init (không commit credentials)
terraform init \
  -backend-config="bucket=mycompany-state" \
  -backend-config="key=production/app.tfstate" \
  -backend-config="region=ap-southeast-1"

# Hoặc từ file (gitignored)
terraform init -backend-config=backend.conf
# backend.conf (gitignored):
# bucket = "mycompany-state"
# access_key = "AKIA..."
# secret_key = "..."
```

---

## terraform_remote_state (Cross-Module Data)

```hcl
# Đọc outputs từ state của workspace khác
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "production/networking/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

# Dùng output từ networking workspace
resource "aws_eks_cluster" "main" {
  vpc_config {
    subnet_ids = data.terraform_remote_state.networking.outputs.private_subnets
  }
}
```

**Lưu ý:** `terraform_remote_state` tạo tight coupling giữa workspaces. Alternative: dùng data sources trực tiếp (e.g., `data "aws_subnet" "..."`) — linh hoạt hơn.

---

## Gotchas

- **State không phải backup**: State tracking Terraform-managed resources. Không replace infrastructure backup (Velero cho K8s, etcd snapshot). State chỉ là Terraform's view, không phải data backup.
- **`terraform state push` nguy hiểm**: Override remote state bằng local state. Nếu local state cũ hơn remote (lower serial) → có thể mất changes. Chỉ dùng trong disaster recovery, sau khi verify carefully.
- **Locking và CI timeouts**: CI job bị kill (timeout, cancel) → lock không được release → tiếp theo bị block. Implement lock timeout hoặc CI cleanup step. DynamoDB TTL không clean up locks tự động.
- **`ignore_changes` và import**: Khi import resource có config mismatch, thêm tạm `ignore_changes = [attribute]` để plan clean. Sau đó gradually remove ignore và fix config.
- **Cross-team state dependencies**: `terraform_remote_state` yêu cầu quyền đọc state bucket của team khác. Cân nhắc: expose outputs qua SSM Parameter Store hoặc Consul thay vì direct state access.
