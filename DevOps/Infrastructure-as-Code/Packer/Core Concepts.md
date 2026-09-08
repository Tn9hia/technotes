---
title: Packer Core Concepts
tags:
  - packer
  - iac
  - core
date: 2026-07-18
---

# Packer Core Concepts

## Template Structure (HCL2)

Từ Packer 1.7+, template chuẩn viết bằng **HCL2** (giống Terraform), file đuôi `.pkr.hcl`. JSON template cũ (`.json`) vẫn chạy được nhưng deprecated cho template mới.

```
template.pkr.hcl
    │
    ├── packer {}              ← required_plugins, version constraint
    ├── variable "x" {}         ← input, giống Terraform variable
    ├── source "builder-type" "name" {}   ← config của 1 builder cụ thể
    └── build {                ← liên kết sources + provisioners + post-processors
          sources = [...]
          provisioner "shell" {...}
          post-processor "vagrant" {...}
        }
```

```hcl
packer {
  required_plugins {
    amazon = {
      version = ">= 1.2.8"
      source  = "github.com/hashicorp/amazon"
    }
    virtualbox = {
      version = ">= 1.0.5"
      source  = "github.com/hashicorp/virtualbox"
    }
  }
}
```

---

## Variables

```hcl
variable "instance_type" {
  type    = string
  default = "t3.medium"
}

variable "aws_region" {
  type    = string
  default = "ap-southeast-1"
}

variable "ssh_username" {
  type      = string
  default   = "ubuntu"
  sensitive = false
}

variable "app_version" {
  type    = string
  # không default → bắt buộc truyền khi build
}

# Dùng trong source block
source "amazon-ebs" "app" {
  region        = var.aws_region
  instance_type = var.instance_type
}
```

```bash
packer build -var="app_version=1.4.0" template.pkr.hcl
packer build -var-file="prod.pkrvars.hcl" template.pkr.hcl
```

`locals {}` cũng dùng được y hệt Terraform để tính giá trị derived (VD ghép tag name từ version + timestamp).

---

## Source Blocks (Builders)

`source` = cấu hình cho một loại builder cụ thể — tương đương "provider config" trong Terraform. Builder quyết định Packer tạo temporary instance/VM ở đâu để chạy provisioner.

### amazon-ebs (build AMI)

```hcl
source "amazon-ebs" "app" {
  region        = var.aws_region
  instance_type = "t3.medium"
  ssh_username  = "ubuntu"

  # Base image để build từ đó
  source_ami_filter {
    filters = {
      name                = "ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"
      virtualization-type = "hvm"
      root-device-type    = "ebs"
    }
    most_recent = true
    owners      = ["099720109477"]   # Canonical
  }

  ami_name = "app-${var.app_version}-${formatdate("YYYYMMDDhhmm", timestamp())}"

  tags = {
    Name      = "app-image"
    Version   = var.app_version
    ManagedBy = "packer"
  }
}
```

**Cách hoạt động:** Packer boot 1 EC2 instance tạm từ `source_ami_filter`, SSH vào, chạy provisioners, sau đó **snapshot instance thành AMI mới** rồi terminate instance tạm.

### virtualbox-iso (build box cho Vagrant)

```hcl
source "virtualbox-iso" "ubuntu" {
  guest_os_type    = "Ubuntu_64"
  iso_url          = "https://releases.ubuntu.com/22.04/ubuntu-22.04.4-live-server-amd64.iso"
  iso_checksum     = "file:https://releases.ubuntu.com/22.04/SHA256SUMS"

  ssh_username     = "vagrant"
  ssh_password     = "vagrant"
  shutdown_command = "echo 'vagrant' | sudo -S shutdown -P now"

  disk_size        = 20000    # MB
  memory           = 2048
  cpus             = 2

  # Cần autoinstall/preseed file để cài OS không tương tác
  boot_command = ["<esc><wait>", "linux /install/vmlinuz auto-install/enable=true ..."]
  http_directory = "http"     # serve autoinstall config qua HTTP tạm
}
```

**Khác biệt so với amazon-ebs:** builder ISO phải tự cài cả OS từ đầu (boot ISO, chạy installer tự động qua `boot_command`/preseed) — nặng hơn vì AMI builder chỉ kế thừa OS đã cài sẵn từ base AMI.

### docker (build Docker image)

```hcl
source "docker" "app" {
  image  = "ubuntu:22.04"
  commit = true             # commit container thành image mới sau khi provision
}
```

### qemu (build image cho KVM/libvirt)

```hcl
source "qemu" "app" {
  iso_url          = "https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img"
  iso_checksum     = "none"
  disk_image       = true    # dùng cloud image thay vì cài từ ISO installer
  output_directory = "output-qemu"
  accelerator      = "kvm"
  disk_size        = "20G"
  format           = "qcow2"
  ssh_username     = "ubuntu"
}
```

---

## Build Block — Liên kết mọi thứ

```hcl
build {
  name = "app-image"

  sources = [
    "source.amazon-ebs.app",
    "source.virtualbox-iso.ubuntu",
  ]

  provisioner "shell" {
    script = "scripts/install-deps.sh"
  }

  provisioner "ansible" {
    playbook_file = "provisioning/playbook.yml"
  }

  post-processor "manifest" {
    output = "manifest.json"
  }
}
```

**Multi-source build:** khai báo nhiều `source` trong 1 `build` block → Packer build **song song** ra nhiều target (AMI + VirtualBox box) từ **cùng bộ provisioner** — đây chính là cách đảm bảo dev image và production image giống hệt nhau.

### Chỉ build 1 source cụ thể (khi có nhiều)

```bash
packer build -only="amazon-ebs.app" template.pkr.hcl
packer build -except="virtualbox-iso.ubuntu" template.pkr.hcl
```

---

## Plugins

Packer core chỉ có logic điều phối — mọi builder/provisioner/post-processor cụ thể (trừ shell, file) đều là **plugin** cài riêng.

```hcl
packer {
  required_plugins {
    amazon = {
      version = ">= 1.2.8"
      source  = "github.com/hashicorp/amazon"
    }
  }
}
```

```bash
packer init template.pkr.hcl     # tự động cài plugin khai báo trong required_plugins
packer plugins install github.com/hashicorp/ansible
packer plugins installed         # danh sách plugin đã cài
```

---

## Gotchas

- **Quên `packer init` trước `packer build`**: từ khi chuyển sang plugin architecture (Packer 1.7+), builder như `amazon-ebs` không còn bundle sẵn trong binary — phải `packer init` để tải plugin trước, nếu không `packer build` báo lỗi "Unknown builder type".
- **`source_ami_filter` với `most_recent = true` không pin version**: giống vấn đề box Vagrant không pin version — base AMI có thể đổi ngầm giữa các lần build, image ra không còn reproducible. Cân nhắc pin cụ thể AMI ID cho build production quan trọng.
- **ISO builder (`virtualbox-iso`, `qemu` với ISO) rất chậm**: phải cài cả OS từ đầu qua `boot_command`. Nếu chỉ cần image dựa trên OS có sẵn, dùng cloud image (`disk_image = true` với qemu, hoặc base AMI có sẵn) nhanh hơn nhiều.
- **`shutdown_command` sai → build treo vô thời hạn**: nếu builder không nhận được tín hiệu VM đã shutdown đúng cách (do sai user/password trong `shutdown_command`), Packer chờ timeout rất lâu trước khi fail — kiểm tra kỹ credential trong config.
- **Provisioner chạy trên máy tạm, không phải image cuối**: dễ nhầm — script provisioner chạy trong lúc build (trên VM/container tạm thời), không phải sau khi image đã deploy. Bất kỳ state runtime nào tạo ra lúc build (log file, temp cache) nên dọn sạch trước khi Packer snapshot, nếu không sẽ "đóng băng" vào image.
