---
title: Packer Provisioners & Post-Processors
tags:
  - packer
  - provisioners
  - post-processors
date: 2026-07-18
---

# Packer Provisioners & Post-Processors

## Provisioners — Cài đặt bên trong image tạm

Chạy **trong lúc build**, trên instance/VM/container tạm mà builder vừa tạo — y hệt vai trò provisioner trong Vagrant, chỉ khác là kết quả được **đóng băng vào image** thay vì tồn tại trên máy chạy lâu dài.

### Shell Provisioner

```hcl
provisioner "shell" {
  script = "scripts/install-deps.sh"
}

provisioner "shell" {
  inline = [
    "sudo apt-get update",
    "sudo apt-get install -y nginx",
  ]
}

# Nhiều script theo thứ tự
provisioner "shell" {
  scripts = [
    "scripts/01-base.sh",
    "scripts/02-app.sh",
    "scripts/03-cleanup.sh",
  ]
}

# Truyền biến vào script qua env
provisioner "shell" {
  script = "scripts/install.sh"
  environment_vars = [
    "APP_VERSION=${var.app_version}",
  ]
}
```

### Ansible Provisioner

```hcl
provisioner "ansible" {
  playbook_file = "provisioning/playbook.yml"
  extra_arguments = [
    "--extra-vars", "app_version=${var.app_version}",
  ]
}
```

**Cách hoạt động:** Packer tự tạo inventory tạm trỏ về instance/VM đang build (qua SSH ephemeral connection), rồi chạy `ansible-playbook` từ máy host — cần cài Ansible trên máy chạy `packer build`, tương tự provisioner `ansible` (không phải `ansible_local`) của Vagrant.

### File Provisioner

```hcl
provisioner "file" {
  source      = "configs/app.conf"
  destination = "/tmp/app.conf"
}
```

### Thứ tự thực thi

Giống Vagrant — provisioners chạy **tuần tự theo thứ tự khai báo** trong `build` block, dùng để đảm bảo dependency (copy file config trước, chạy script đọc config sau).

```hcl
build {
  sources = ["source.amazon-ebs.app"]

  provisioner "file" {
    source      = "configs/app.conf"
    destination = "/tmp/app.conf"
  }

  provisioner "shell" {
    inline = ["sudo mv /tmp/app.conf /etc/app/app.conf"]
  }

  provisioner "ansible" {
    playbook_file = "provisioning/site.yml"
  }
}
```

---

## Post-Processors — Đóng gói kết quả sau khi provision xong

Chạy **sau khi** provisioners xong, **trước khi** builder cleanup instance tạm — dùng để convert/đóng gói/publish artifact.

### vagrant (build ra `.box`)

```hcl
post-processor "vagrant" {
  output = "output/app-{{.Provider}}.box"
}
```

Convert kết quả từ builder (VD `virtualbox-iso`) thành file `.box` sẵn sàng dùng trong `config.vm.box_url`.

### compress

```hcl
post-processor "compress" {
  output = "output/app-image.tar.gz"
}
```

### manifest (ghi lại metadata build — AMI ID, timestamp)

```hcl
post-processor "manifest" {
  output     = "manifest.json"
  strip_path = true
}
```

```json
// manifest.json — dùng để Terraform/CI đọc lại AMI ID vừa build
{
  "builds": [
    {
      "name": "app-image",
      "builder_type": "amazon-ebs",
      "build_time": 1721289600,
      "artifact_id": "ap-southeast-1:ami-0abc123def456"
    }
  ]
}
```

### docker-tag / docker-push

```hcl
post-processor "docker-tag" {
  repository = "myregistry.internal/app"
  tags       = [var.app_version, "latest"]
}

post-processor "docker-push" {
  # dùng docker login trước, hoặc ECR credential helper
}
```

### Chaining post-processors

```hcl
post-processors {
  post-processor "compress" {
    output = "output/app.tar.gz"
  }
  post-processor "docker-tag" {
    repository = "myregistry.internal/app"
    tags       = ["latest"]
  }
}
```

Dùng `post-processors { ... }` (số nhiều, có ngoặc nhọn bao ngoài) khi muốn các post-processor chạy **nối tiếp trên cùng 1 artifact** (output của cái trước là input cái sau). Khai báo rời từng `post-processor` riêng lẻ thì chúng chạy **độc lập song song** trên artifact gốc.

---

## Multi-Target Build Pattern (Vagrant + Terraform từ 1 template)

```hcl
build {
  sources = [
    "source.virtualbox-iso.dev",     # → build .box cho Vagrant
    "source.amazon-ebs.prod",        # → build AMI cho Terraform
  ]

  # Provisioner CHUNG cho cả 2 target — đảm bảo giống hệt nhau
  provisioner "shell" {
    script = "scripts/install-deps.sh"
  }

  provisioner "ansible" {
    playbook_file = "provisioning/site.yml"
  }

  post-processor "vagrant" {
    only   = ["virtualbox-iso.dev"]
    output = "output/dev-{{.Provider}}.box"
  }

  post-processor "manifest" {
    only   = ["amazon-ebs.prod"]
    output = "manifest.json"
  }
}
```

`only`/`except` trong post-processor giới hạn post-processor đó chỉ áp dụng cho source cụ thể — cần thiết vì `vagrant` post-processor không có ý nghĩa với artifact từ `amazon-ebs`.

---

## Gotchas

- **Ansible provisioner cần Ansible trên máy chạy `packer build`**: giống hệt lưu ý ở Vagrant — nếu CI runner build image không có Ansible cài sẵn, provisioner này sẽ fail ngay bước đầu.
- **`manifest` post-processor ghi đè file cũ mỗi build**: nếu không đổi `output` path hoặc không archive lại, build sau sẽ mất thông tin AMI ID của build trước — quan trọng khi cần audit trail image nào đang chạy production.
- **`only`/`except` áp theo tên `<builder_type>.<source_name>`, không phải tên biến**: dễ gõ sai (VD `amazon-ebs.app` thay vì đúng tên khai báo trong `source "amazon-ebs" "app"`) → post-processor âm thầm không chạy mà không báo lỗi rõ ràng, nên luôn `packer validate` trước khi build thật.
- **`docker-push` cần đăng nhập registry trước khi `packer build`**: post-processor không tự xử lý auth — phải `docker login` (hoặc cấu hình credential helper) từ trước, nếu không build "thành công" ở bước tag nhưng fail ở bước push.
- **Cleanup trước khi provisioner kết thúc, không phải sau post-processor**: log file, cache, temp secret tạo ra trong lúc provisioner chạy sẽ bị đóng băng vào image nếu không dọn — nên thêm bước `provisioner "shell" { inline = ["cleanup commands"] }` là provisioner **cuối cùng**, trước khi tới post-processor.
