---
title: Terraform Core Concepts
tags:
  - terraform
  - iac
  - core
date: 2026-04-29
---

# Terraform Core Concepts

## Cách hoạt động

```
.tf files (HCL)
    │
    ▼
terraform plan      ← diff desired state vs current state (tfstate)
    │
    ▼
terraform apply     ← call provider APIs để reconcile
    │
    ▼
terraform.tfstate   ← ghi lại current state sau khi apply
```

Terraform **declarative**: bạn mô tả *what*, Terraform tính *how* và thứ tự.

---

## Providers

Provider = plugin biết cách talk tới một API (AWS, GCP, Kubernetes, Vault, PostgreSQL…).

```hcl
# versions.tf
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"      # ~> 5.0 = >= 5.0, < 6.0
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.25.0"
    }
    vault = {
      source  = "hashicorp/vault"
      version = "~> 4.0"
    }
  }
}

# providers.tf
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "terraform"
      Project     = var.project_name
    }
  }
}
```

### Multiple Provider Configs (alias)

```hcl
# Hai AWS accounts, hoặc hai regions
provider "aws" {
  alias  = "primary"
  region = "ap-southeast-1"
}

provider "aws" {
  alias  = "dr"
  region = "ap-southeast-2"
}

# Chỉ định provider khi dùng resource
resource "aws_s3_bucket" "dr_backup" {
  provider = aws.dr
  bucket   = "my-dr-backup"
}
```

---

## Resources

Resource là block khai báo infrastructure object.

```hcl
# resource "<TYPE>" "<LOCAL_NAME>" { ... }
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"

  tags = {
    Name = "web-server"
  }
}

# Tham chiếu: <TYPE>.<LOCAL_NAME>.<ATTRIBUTE>
output "web_ip" {
  value = aws_instance.web.private_ip
}
```

### Meta-arguments

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  # depends_on: explicit dependency (dùng khi Terraform không tự detect)
  depends_on = [aws_iam_role_policy.web_policy]

  # lifecycle: control create/update/delete behavior
  lifecycle {
    create_before_destroy = true    # tạo mới trước khi destroy old (zero-downtime replace)
    prevent_destroy       = true    # block terraform destroy (production DBs, etc.)
    ignore_changes        = [tags]  # ignore external changes tới specific attributes
    replace_triggered_by  = [null_resource.trigger]   # force replace khi trigger changes
  }
}
```

---

## Data Sources

Data source đọc existing resources — không create hay manage chúng.

```hcl
# Đọc existing VPC
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["main-vpc"]
  }
}

# Đọc latest Ubuntu AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]    # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}

# Đọc secret từ Vault
data "vault_kv_secret_v2" "db_creds" {
  mount = "secret"
  name  = "production/database"
}

# Dùng data source
resource "aws_instance" "web" {
  ami    = data.aws_ami.ubuntu.id
  vpc_id = data.aws_vpc.main.id
}
```

---

## Variables & Outputs

### Variables

```hcl
# variables.tf
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "environment must be dev, staging, or production."
  }
}

variable "instance_count" {
  type    = number
  default = 2
}

variable "enable_monitoring" {
  type    = bool
  default = true
}

variable "allowed_cidrs" {
  type    = list(string)
  default = ["10.0.0.0/8"]
}

variable "tags" {
  type    = map(string)
  default = {}
}

# Object type với nested validation
variable "database_config" {
  type = object({
    instance_class    = string
    allocated_storage = number
    multi_az          = bool
  })
  default = {
    instance_class    = "db.t3.medium"
    allocated_storage = 100
    multi_az          = false
  }
}

# Sensitive variable — không hiện trong plan/apply output
variable "db_password" {
  type      = string
  sensitive = true
}
```

### Cách truyền variable values

```bash
# 1. terraform.tfvars (auto-loaded)
environment    = "production"
instance_count = 3

# 2. *.auto.tfvars (auto-loaded)
# production.auto.tfvars
enable_monitoring = true

# 3. CLI flag
terraform apply -var="environment=staging" -var="instance_count=2"

# 4. Var file
terraform apply -var-file="production.tfvars"

# 5. Environment variable
export TF_VAR_environment=production
export TF_VAR_db_password=supersecret
```

### Outputs

```hcl
# outputs.tf
output "instance_ip" {
  value       = aws_instance.web.private_ip
  description = "Private IP of web server"
}

output "db_connection_string" {
  value       = "postgres://${var.db_user}@${aws_db_instance.main.endpoint}/${var.db_name}"
  sensitive   = true    # ẩn khỏi console output, nhưng vẫn trong state
}

# Output từ module
output "vpc_id" {
  value = module.networking.vpc_id
}
```

```bash
# Đọc output
terraform output instance_ip
terraform output -json    # tất cả outputs dưới dạng JSON
terraform output -raw db_connection_string   # raw value (no quotes)
```

---

## Locals & Expressions

Locals là computed values trong module — không phải input (variables) hay output.

```hcl
locals {
  # Simple computation
  is_production = var.environment == "production"
  name_prefix   = "${var.project}-${var.environment}"

  # Conditional
  instance_type = local.is_production ? "t3.large" : "t3.micro"

  # List manipulation
  availability_zones = slice(data.aws_availability_zones.available.names, 0, 3)

  # Map merge
  common_tags = merge(var.tags, {
    Environment = var.environment
    ManagedBy   = "terraform"
    UpdatedAt   = timestamp()
  })
}

resource "aws_instance" "web" {
  instance_type = local.instance_type
  tags          = local.common_tags
}
```

---

## Built-in Functions

```hcl
# ─── String ───
local {
  upper_env = upper(var.environment)          # "PRODUCTION"
  trimmed   = trimspace("  hello  ")          # "hello"
  joined    = join(",", ["a", "b", "c"])      # "a,b,c"
  replaced  = replace("foo-bar", "-", "_")   # "foo_bar"
  formatted = format("%-10s = %d", "count", 5)
  templated = templatefile("user_data.sh.tpl", { name = var.name })
}

# ─── Collection ───
locals {
  first     = element(var.list, 0)
  length    = length(var.list)
  flat      = flatten([[1, 2], [3, 4]])          # [1, 2, 3, 4]
  distinct  = distinct(["a", "b", "a"])           # ["a", "b"]
  keys_list = keys(var.map)
  vals_list = values(var.map)
  zipped    = zipmap(["a", "b"], [1, 2])         # {a=1, b=2}

  # Filter: chỉ lấy elements thoả điều kiện
  prod_instances = [for i in var.instances : i if i.env == "production"]

  # toset, tolist, tomap: type conversion
  unique_azs = toset(var.availability_zones)
}

# ─── Numeric ───
locals {
  max_val = max(1, 5, 3)    # 5
  min_val = min(1, 5, 3)    # 1
  ceil    = ceil(1.2)        # 2
  floor   = floor(1.9)       # 1
}

# ─── Encoding ───
locals {
  b64     = base64encode("hello")
  json    = jsonencode({ key = "value" })
  parsed  = jsondecode(file("config.json"))
  yamled  = yamlencode({ key = "value" })
}

# ─── Filesystem ───
locals {
  script  = file("${path.module}/scripts/init.sh")
  files   = fileset(path.module, "scripts/*.sh")
}

# ─── Type check ───
locals {
  safe_value = try(var.optional_config.key, "default")
  # try(): trả về giá trị đầu tiên không error
  # coalesce(): trả về giá trị non-null đầu tiên
  first_set = coalesce(var.override, var.default, "fallback")
}
```

---

## Count & for_each

### count

```hcl
# Tạo N copies của resource
resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = var.ami_id
  instance_type = "t3.medium"

  tags = {
    Name = "web-${count.index}"    # count.index: 0, 1, 2, ...
  }
}

# Conditional resource (0 hoặc 1)
resource "aws_cloudwatch_log_group" "app" {
  count = var.enable_logging ? 1 : 0
  name  = "/aws/app/${var.name}"
}

# Reference: aws_instance.web[0], aws_instance.web[1], ...
output "web_ips" {
  value = aws_instance.web[*].private_ip   # splat expression
}
```

**Hạn chế của count:** nếu xóa phần tử ở giữa list → Terraform re-index → destroy và recreate nhiều resources. Dùng `for_each` cho collections.

### for_each

```hcl
# for_each với map: key stable → không bị re-index
variable "instances" {
  type = map(object({
    instance_type = string
    zone          = string
  }))
  default = {
    "web-1" = { instance_type = "t3.medium", zone = "ap-southeast-1a" }
    "web-2" = { instance_type = "t3.large",  zone = "ap-southeast-1b" }
  }
}

resource "aws_instance" "web" {
  for_each          = var.instances
  ami               = var.ami_id
  instance_type     = each.value.instance_type   # each.key, each.value
  availability_zone = each.value.zone

  tags = {
    Name = each.key    # "web-1", "web-2"
  }
}

# for_each với set of strings
resource "aws_iam_user" "team" {
  for_each = toset(["alice", "bob", "charlie"])
  name     = each.key
}

# Reference: aws_instance.web["web-1"].private_ip
output "ips" {
  value = { for k, v in aws_instance.web : k => v.private_ip }
}
```

### for expressions

```hcl
locals {
  # List comprehension
  instance_ids = [for instance in aws_instance.web : instance.id]

  # Map comprehension
  instance_map = { for k, v in aws_instance.web : k => v.private_ip }

  # Filter
  large_instances = [for k, v in var.instances : k if v.instance_type == "t3.large"]

  # Nested for
  sg_rules = flatten([
    for port in var.ports : [
      for cidr in var.allowed_cidrs : {
        port = port
        cidr = cidr
      }
    ]
  ])
}
```

### dynamic blocks

```hcl
# Thay vì lặp lại nhiều block giống nhau
resource "aws_security_group" "app" {
  name = "app-sg"

  dynamic "ingress" {
    for_each = var.ingress_rules    # list of objects
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ingress.value.cidrs
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

variable "ingress_rules" {
  type = list(object({
    port  = number
    cidrs = list(string)
  }))
  default = [
    { port = 80,  cidrs = ["0.0.0.0/0"] },
    { port = 443, cidrs = ["0.0.0.0/0"] },
    { port = 22,  cidrs = ["10.0.0.0/8"] },
  ]
}
```

---

## Modules

Module = reusable group of resources với defined inputs (variables) và outputs.

### Module Structure

```
modules/
  networking/
    main.tf        ← resources
    variables.tf   ← inputs
    outputs.tf     ← outputs
    versions.tf    ← provider requirements
    README.md
  kubernetes-cluster/
    main.tf
    variables.tf
    outputs.tf
```

### Tạo và dùng module

```hcl
# modules/networking/variables.tf
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}
variable "environment" { type = string }
variable "az_count"    { type = number; default = 3 }

# modules/networking/main.tf
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags       = { Name = "${var.environment}-vpc" }
}

resource "aws_subnet" "private" {
  count             = var.az_count
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}

# modules/networking/outputs.tf
output "vpc_id"         { value = aws_vpc.main.id }
output "private_subnets" { value = aws_subnet.private[*].id }

# ─── Dùng module ───
# environments/production/main.tf
module "networking" {
  source = "../../modules/networking"

  vpc_cidr    = "10.10.0.0/16"
  environment = "production"
  az_count    = 3
}

# Tham chiếu output của module
resource "aws_eks_cluster" "main" {
  vpc_config {
    subnet_ids = module.networking.private_subnets
  }
}
```

### Module Sources

```hcl
# Local path
source = "../../modules/networking"

# Terraform Registry (public)
source  = "terraform-aws-modules/vpc/aws"
version = "~> 5.0"

# Git
source = "git::https://github.com/myorg/tf-modules.git//networking?ref=v2.1.0"
# Hoặc SSH
source = "git::ssh://git@github.com/myorg/tf-modules.git//networking?ref=main"

# Private registry
source  = "registry.terraform.internal/myorg/vpc/aws"
version = "~> 1.0"
```

---

## Workspaces

Workspace cho phép dùng cùng config với different state files — theo môi trường.

```bash
# Tạo và switch workspace
terraform workspace new staging
terraform workspace new production
terraform workspace list
terraform workspace select production

# Dùng trong config
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "production" ? "t3.large" : "t3.micro"
  count         = terraform.workspace == "production" ? 3 : 1
}
```

**Giới hạn:** Workspaces dùng cùng backend location — không isolate state tốt. Production pattern ưa dùng **separate directories** (environments/prod, environments/staging) thay vì workspaces.

---

## Lifecycle & Provisioners

### Lifecycle rules

```hcl
resource "aws_db_instance" "main" {
  # ...

  lifecycle {
    # Tạo replacement trước khi xóa old (giảm downtime cho resources như LB)
    create_before_destroy = true

    # Block mọi destroy — yêu cầu remove block trước khi destroy
    prevent_destroy = true

    # Bỏ qua changes bên ngoài Terraform
    ignore_changes = [
      tags["LastUpdated"],    # external system update tag này
      ami,                    # không force replace khi AMI mới
    ]

    # Force replace resource khi specific value changes
    replace_triggered_by = [
      aws_s3_object.user_data_script    # deploy mới khi script thay đổi
    ]

    # Precondition: validate trước khi create
    precondition {
      condition     = var.instance_type != "t2.micro" || var.environment != "production"
      error_message = "t2.micro is not allowed in production."
    }

    # Postcondition: validate sau khi create
    postcondition {
      condition     = self.private_ip != ""
      error_message = "Instance must have a private IP."
    }
  }
}
```

### Provisioners (dùng hạn chế)

```hcl
# Provisioners là escape hatch — chỉ dùng khi không có provider tốt hơn
# Không idempotent, không trong plan output → khó debug

resource "aws_instance" "web" {
  # ...

  # local-exec: chạy command trên machine đang run Terraform
  provisioner "local-exec" {
    command = "ansible-playbook -i '${self.private_ip},' playbook.yml"
    environment = {
      ANSIBLE_HOST_KEY_CHECKING = "false"
    }
  }

  # remote-exec: chạy command trên remote instance
  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
    ]
    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip
    }
  }

  # on destroy: chạy khi resource bị destroyed
  provisioner "local-exec" {
    when    = destroy
    command = "curl -X DELETE https://api.internal/deregister/${self.id}"
  }
}
```

---

## Gotchas

- **`count` vs `for_each`**: Không convert giữa hai cái mà không destroy. `count = 3` → resources tại `[0]`, `[1]`, `[2]`. Xóa index 1 → `[2]` trở thành `[1]` → Terraform destroy + recreate `[1]`. `for_each` tránh vấn đề này vì địa chỉ dựa trên key.
- **`depends_on` ảnh hưởng plan**: `depends_on` trên module → toàn bộ module plan là unknown cho đến khi dependency resolved. Làm plan chậm và less precise. Chỉ dùng khi thực sự cần.
- **`sensitive = true` không mã hóa state**: `sensitive` chỉ ẩn giá trị khỏi console output. Trong `terraform.tfstate`, giá trị vẫn là plaintext. Luôn encrypt remote state.
- **`lifecycle.ignore_changes` và drift**: `ignore_changes = [tags]` → Terraform không bao giờ fix drift trong `tags`. Nếu external system xóa tags quan trọng → Terraform không restore. Dùng cẩn thận.
- **Provider version `~>` và breaking changes**: `~> 4.0` allow `4.x` updates. Nhưng provider có thể có breaking changes trong minor versions. Luôn check CHANGELOG trước `terraform init -upgrade`.
