---
title: Terraform Modules & Patterns
tags:
  - terraform
  - modules
  - patterns
  - best-practices
date: 2026-04-29
---

# Terraform Modules & Patterns

## Module Design Principles

```
Module tốt:
  ✓ Single responsibility (1 module = 1 concern)
  ✓ Explicit inputs/outputs (không hard-code values)
  ✓ Sensible defaults (override chỉ khi cần)
  ✓ Idempotent (apply nhiều lần = same result)
  ✓ Self-contained (không import từ parent scope)

Module xấu:
  ✗ "God module" — tạo tất cả resources của cả hệ thống
  ✗ Expose toàn bộ provider config qua variables
  ✗ Hard-code region, account, environment
  ✗ Nested modules quá sâu (>3 levels)
```

---

## Recommended Repository Structure

```
infrastructure/
├── modules/                    ← reusable modules (không có environment-specific config)
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   └── README.md
│   ├── kubernetes-cluster/
│   ├── postgres-rds/
│   └── app-service/
│
└── environments/               ← environment-specific roots
    ├── production/
    │   ├── networking/         ← mỗi component = separate state
    │   │   ├── main.tf
    │   │   ├── variables.tf
    │   │   ├── outputs.tf
    │   │   ├── backend.tf
    │   │   └── terraform.tfvars
    │   ├── kubernetes/
    │   ├── databases/
    │   └── apps/
    │       ├── myapp/
    │       └── payments/
    ├── staging/
    │   ├── networking/
    │   └── ...
    └── dev/
        └── ...
```

**Tại sao separate directories thay vì workspaces?**
- Mỗi component có state riêng → blast radius nhỏ hơn
- Permissions granular (team A chỉ có quyền apply `apps/`)
- Dễ audit: ai apply gì, khi nào
- Tránh accidental cross-env apply

---

## Module Composition — Ví dụ Thực tế

### Module: `networking`

```hcl
# modules/networking/variables.tf
variable "environment"   { type = string }
variable "vpc_cidr"      { type = string; default = "10.0.0.0/16" }
variable "az_count"      { type = number; default = 3 }
variable "enable_nat_gw" { type = bool;   default = true }

# modules/networking/main.tf
locals {
  name = "${var.environment}-vpc"
  azs  = slice(data.aws_availability_zones.available.names, 0, var.az_count)
}

data "aws_availability_zones" "available" { state = "available" }

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = local.name }
}

resource "aws_subnet" "private" {
  count             = var.az_count
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index)
  availability_zone = local.azs[count.index]
  tags = { Name = "${local.name}-private-${count.index + 1}", Tier = "private" }
}

resource "aws_subnet" "public" {
  count                   = var.az_count
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4, count.index + var.az_count)
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "${local.name}-public-${count.index + 1}", Tier = "public" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${local.name}-igw" }
}

resource "aws_eip" "nat" {
  count  = var.enable_nat_gw ? var.az_count : 0
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  count         = var.enable_nat_gw ? var.az_count : 0
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  tags          = { Name = "${local.name}-nat-${count.index + 1}" }
}

# modules/networking/outputs.tf
output "vpc_id"          { value = aws_vpc.main.id }
output "private_subnets" { value = aws_subnet.private[*].id }
output "public_subnets"  { value = aws_subnet.public[*].id }
output "vpc_cidr"        { value = aws_vpc.main.cidr_block }
```

### Module: `app-service` (compose các modules)

```hcl
# modules/app-service/variables.tf
variable "name"        { type = string }
variable "environment" { type = string }
variable "image"       { type = string }
variable "replicas"    { type = number; default = 2 }
variable "cpu"         { type = string; default = "250m" }
variable "memory"      { type = string; default = "256Mi" }
variable "namespace"   { type = string }
variable "secrets"     { type = map(string); default = {} }

# modules/app-service/main.tf — Kubernetes resources
resource "kubernetes_deployment" "app" {
  metadata {
    name      = var.name
    namespace = var.namespace
    labels    = { app = var.name, environment = var.environment }
  }

  spec {
    replicas = var.replicas

    selector {
      match_labels = { app = var.name }
    }

    template {
      metadata {
        labels = { app = var.name }
      }
      spec {
        container {
          name  = var.name
          image = var.image

          resources {
            requests = { cpu = var.cpu, memory = var.memory }
            limits   = { cpu = var.cpu, memory = var.memory }
          }

          dynamic "env" {
            for_each = var.secrets
            content {
              name = env.key
              value_from {
                secret_key_ref {
                  name = kubernetes_secret.app.metadata[0].name
                  key  = env.key
                }
              }
            }
          }
        }
      }
    }
  }
}

resource "kubernetes_secret" "app" {
  metadata {
    name      = "${var.name}-secrets"
    namespace = var.namespace
  }
  data = var.secrets
}

resource "kubernetes_service" "app" {
  metadata {
    name      = var.name
    namespace = var.namespace
  }
  spec {
    selector = { app = var.name }
    port {
      port        = 80
      target_port = 8080
    }
  }
}
```

### Root: `environments/production/apps/myapp`

```hcl
# environments/production/apps/myapp/backend.tf
terraform {
  backend "s3" {
    bucket = "mycompany-terraform-state"
    key    = "production/apps/myapp/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

# environments/production/apps/myapp/main.tf
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "production/networking/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

data "vault_kv_secret_v2" "app_secrets" {
  mount = "secret"
  name  = "production/myapp"
}

module "myapp" {
  source = "../../../../modules/app-service"

  name        = "myapp"
  environment = "production"
  image       = "ghcr.io/myorg/myapp:${var.image_tag}"
  replicas    = 3
  cpu         = "500m"
  memory      = "512Mi"
  namespace   = "production"
  secrets     = data.vault_kv_secret_v2.app_secrets.data
}

# environments/production/apps/myapp/terraform.tfvars
image_tag = "sha-abc1234"
```

---

## Provider Version Management

```hcl
# versions.tf — pin tất cả providers
terraform {
  required_version = ">= 1.7.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.50"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.30"
    }
    vault = {
      source  = "hashicorp/vault"
      version = "~> 4.4"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.14"
    }
  }
}
```

```bash
# terraform.lock.hcl — auto-generated, COMMIT vào git
# Lock file đảm bảo mọi người dùng cùng provider version

# Update providers (phải explicit)
terraform init -upgrade

# Check lock file
cat .terraform.lock.hcl
# provider "registry.terraform.io/hashicorp/aws" {
#   version     = "5.52.0"
#   constraints = "~> 5.50"
#   hashes = [
#     "h1:AbCd...",   ← verify integrity
#     "zh:AbCd...",
#   ]
# }
```

---

## Null Provider & Random Provider

```hcl
# null_resource: trigger side effects (không create real resources)
resource "null_resource" "bootstrap" {
  triggers = {
    cluster_version = var.cluster_version    # re-run khi cluster version thay đổi
    always_run      = timestamp()            # re-run mỗi apply (cẩn thận!)
  }

  provisioner "local-exec" {
    command = "kubectl apply -f base-manifests/ --context=${var.kube_context}"
  }
}

# terraform_data (Terraform 1.4+ — thay thế null_resource)
resource "terraform_data" "bootstrap" {
  triggers_replace = [var.cluster_version]

  provisioner "local-exec" {
    command = "kubectl apply -f base-manifests/"
  }
}

# random provider: generate unique names, passwords
resource "random_id" "suffix" {
  byte_length = 4
}

resource "random_password" "db_password" {
  length           = 32
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>?"
}

resource "random_pet" "server_name" {
  length    = 2
  separator = "-"
  # → "happy-panda"
}

resource "aws_s3_bucket" "state" {
  bucket = "mycompany-state-${random_id.suffix.hex}"
  # → "mycompany-state-a1b2c3d4"
}
```

---

## Terragrunt (DRY cho Multi-Environment)

Terragrunt là wrapper giảm code duplication khi nhiều environments.

```hcl
# terragrunt.hcl (root)
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "mycompany-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

inputs = {
  project = "mycompany"
}
```

```hcl
# environments/production/networking/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules//networking"
}

inputs = {
  environment  = "production"
  vpc_cidr     = "10.10.0.0/16"
  az_count     = 3
  enable_nat_gw = true
}
```

```bash
# Apply toàn bộ production
terragrunt run-all apply --terragrunt-working-dir environments/production

# Plan specific component
cd environments/production/networking
terragrunt plan

# Dependency ordering: Terragrunt detect cross-component dependencies
# environments/production/kubernetes/terragrunt.hcl
dependency "networking" {
  config_path = "../networking"
  mock_outputs = {
    vpc_id          = "vpc-00000000"
    private_subnets = ["subnet-00000000"]
  }
}

inputs = {
  vpc_id     = dependency.networking.outputs.vpc_id
  subnet_ids = dependency.networking.outputs.private_subnets
}
```

---

## Common Provider Patterns

### Kubernetes + Helm

```hcl
# Authenticate với EKS cluster
data "aws_eks_cluster" "main" {
  name = "production-cluster"
}

data "aws_eks_cluster_auth" "main" {
  name = "production-cluster"
}

provider "kubernetes" {
  host                   = data.aws_eks_cluster.main.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.main.certificate_authority[0].data)
  token                  = data.aws_eks_cluster_auth.main.token
}

provider "helm" {
  kubernetes {
    host                   = data.aws_eks_cluster.main.endpoint
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.main.certificate_authority[0].data)
    token                  = data.aws_eks_cluster_auth.main.token
  }
}

# Install Helm chart
resource "helm_release" "ingress_nginx" {
  name             = "ingress-nginx"
  repository       = "https://kubernetes.github.io/ingress-nginx"
  chart            = "ingress-nginx"
  version          = "4.10.0"
  namespace        = "ingress-nginx"
  create_namespace = true

  values = [
    file("${path.module}/values/ingress-nginx.yaml")
  ]

  set {
    name  = "controller.replicaCount"
    value = 2
  }

  set_sensitive {
    name  = "controller.extraEnvs[0].value"
    value = var.webhook_token
  }
}

# Kubernetes namespace
resource "kubernetes_namespace" "production" {
  metadata {
    name = "production"
    labels = {
      "pod-security.kubernetes.io/enforce" = "restricted"
    }
  }
}
```

### Vault + Terraform

```hcl
provider "vault" {
  address = "https://vault.internal:8200"
  # VAULT_TOKEN env var hoặc:
  auth_login_kubernetes {
    mount = "kubernetes"
    role  = "terraform-production"
    # JWT tự lấy từ service account trong K8s pod
  }
}

resource "vault_policy" "myapp" {
  name = "myapp-production"
  policy = <<EOT
path "secret/data/production/myapp/*" {
  capabilities = ["read", "list"]
}
EOT
}

resource "vault_kubernetes_auth_backend_role" "myapp" {
  backend                          = "kubernetes"
  role_name                        = "myapp-production"
  bound_service_account_names      = ["myapp"]
  bound_service_account_namespaces = ["production"]
  token_policies                   = [vault_policy.myapp.name]
  token_ttl                        = 3600
}
```

---

## Gotchas

- **Module source với double slash (`//`)**: `git::https://github.com/org/repo.git//modules/vpc?ref=v1.0` — double slash tách repo URL và subdirectory path. Single slash là invalid.
- **Circular dependencies giữa modules**: Module A output → Module B input → Module A input → cycle → error. Redesign: extract shared resource ra module C.
- **`terraform_remote_state` và permissions**: Consumer cần read access toàn bộ state file — bao gồm sensitive values. Prefer data sources over `terraform_remote_state` để limit exposure.
- **Helm provider và Kubernetes API timing**: Sau khi tạo EKS cluster, K8s API server cần vài phút để ready. Nếu Helm resource chạy ngay sau `aws_eks_cluster` → connection refused. Thêm `depends_on` hoặc `terraform_data` với `sleep`.
- **Terragrunt mock outputs**: `mock_outputs` chỉ dùng khi `terragrunt plan` (để plan không fail vì dependency chưa apply). Khi `apply`, Terragrunt đọc actual outputs. Đảm bảo mock types match actual types.
