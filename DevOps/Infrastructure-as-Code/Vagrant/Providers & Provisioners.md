---
title: Vagrant Providers & Provisioners
tags:
  - vagrant
  - providers
  - provisioners
date: 2026-07-18
---

# Vagrant Providers & Provisioners

## Providers là gì

Provider = backend thực sự tạo và chạy VM/container (giống Terraform provider, nhưng ở đây là virtualization backend thay vì cloud API).

```
Vagrantfile
    │
    ▼
vagrant up --provider=virtualbox
    │
    ▼
Provider plugin gọi API tương ứng:
  - VirtualBox   → VBoxManage CLI
  - libvirt/KVM  → libvirt API
  - Docker       → docker CLI
  - Hyper-V      → PowerShell/WMI
  - Parallels    → Parallels Desktop CLI (macOS)
```

---

## VirtualBox (default, cross-platform)

Provider mặc định, miễn phí, chạy được trên Windows/macOS/Linux — lựa chọn phổ biến nhất cho local dev.

```ruby
config.vm.provider "virtualbox" do |vb|
  vb.name   = "my-dev-vm"        # tên hiển thị trong VirtualBox GUI
  vb.memory = 4096
  vb.cpus   = 2
  vb.gui    = false               # true để mở GUI window (debug)

  # Custom VBoxManage options
  vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  vb.customize ["modifyvm", :id, "--cableconnected1", "on"]
end
```

**Yêu cầu:** cài VirtualBox riêng (Vagrant chỉ là orchestrator, không bundle hypervisor).

---

## libvirt/KVM (Linux)

Nhanh hơn VirtualBox trên Linux host (dùng KVM native thay vì type-2 hypervisor), nhưng cần plugin riêng.

```bash
vagrant plugin install vagrant-libvirt
```

```ruby
config.vm.provider "libvirt" do |lv|
  lv.memory = 4096
  lv.cpus   = 2
  lv.driver = "kvm"
  lv.storage_pool_name = "default"
end
```

```bash
vagrant up --provider=libvirt
```

---

## Docker

Dùng container thay vì full VM — nhanh hơn nhiều, nhưng phù hợp khi workload không cần kernel riêng hoặc full OS simulation.

```ruby
config.vm.provider "docker" do |d|
  d.image = "ubuntu:22.04"
  d.has_ssh = true              # cần base image có sshd để `vagrant ssh` hoạt động
end
```

**Hạn chế:** không isolate ở mức kernel như VM thật — không phù hợp để test kernel module, systemd behavior đầy đủ, hoặc mô phỏng sát production bare-metal/VM.

---

## Hyper-V (Windows) & Parallels (macOS)

```ruby
# Hyper-V — cần Windows Pro/Enterprise, Hyper-V feature bật sẵn
config.vm.provider "hyperv" do |h|
  h.memory = 4096
  h.cpus   = 2
end

# Parallels Desktop — macOS, thường nhanh hơn VirtualBox trên Apple Silicon
config.vm.provider "parallels" do |p|
  p.memory = 4096
  p.cpus   = 2
end
```

**Apple Silicon (M1/M2/M3):** VirtualBox hỗ trợ ARM còn hạn chế — Parallels hoặc UTM/libvirt thường ổn định hơn cho box ARM64.

---

## So sánh Providers

| Provider | OS host | Tốc độ | Isolation | Ghi chú |
|---|---|---|---|---|
| VirtualBox | Windows/macOS/Linux | Trung bình | VM đầy đủ | Default, dễ setup nhất, free |
| libvirt/KVM | Linux only | Nhanh | VM đầy đủ | Cần plugin, native tới kernel Linux |
| Docker | Mọi OS có Docker | Rất nhanh | Container (yếu hơn VM) | Không sát production nếu cần test OS-level |
| Hyper-V | Windows Pro+ | Nhanh | VM đầy đủ | Conflict với VirtualBox nếu bật cùng lúc |
| Parallels | macOS | Nhanh | VM đầy đủ | Trả phí, tốt cho Apple Silicon |

---

## Provisioners

Provisioner = script/tool chạy **bên trong guest** sau khi VM boot lần đầu (và khi `vagrant provision`) — để cài đặt, configure guest.

### Shell Provisioner

```ruby
# Inline script
config.vm.provision "shell", inline: <<-SHELL
  apt-get update
  apt-get install -y nginx
  systemctl enable nginx
SHELL

# External script file
config.vm.provision "shell", path: "provisioning/bootstrap.sh"

# Chạy với privilege khác (mặc định root)
config.vm.provision "shell", path: "setup.sh", privileged: false

# Truyền argument vào script
config.vm.provision "shell", path: "setup.sh", args: ["production", "8080"]

# Chỉ chạy 1 lần dù `vagrant provision` nhiều lần
config.vm.provision "shell", path: "once.sh", run: "once"     # default
config.vm.provision "shell", path: "always.sh", run: "always"  # chạy mỗi lần up/provision
```

### Ansible Provisioner (2 chế độ)

```ruby
# ansible_local: Ansible cài BÊN TRONG guest, chạy playbook nhắm vào chính guest đó
# → không cần cài Ansible trên host, phù hợp Windows host hoặc CI không có Ansible
config.vm.provision "ansible_local" do |ansible|
  ansible.playbook = "provisioning/playbook.yml"
  ansible.install_mode = "pip"
  ansible.version = "latest"
end

# ansible: Ansible chạy TỪ HOST, SSH vào guest như target bình thường
# → cần cài Ansible trên host, nhưng dùng chung playbook y hệt production
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "provisioning/playbook.yml"
  ansible.inventory_path = "provisioning/inventory"
  ansible.limit = "all"
  ansible.extra_vars = { environment: "vagrant" }
end
```

**Chọn cái nào:** `ansible` (từ host) khi muốn dùng đúng 1 bộ playbook cho cả Vagrant lẫn production — đây là pattern khuyến nghị để "test Ansible playbook trước khi apply production". `ansible_local` khi host không có Ansible cài được (VD Windows không WSL) hoặc cần đảm bảo version Ansible cố định trong guest.

### Docker Provisioner

```ruby
config.vm.provision "docker" do |d|
  d.pull_images "redis:7"
  d.run "redis", image: "redis:7", args: "-p 6379:6379"
end
```

### File Provisioner

```ruby
# Copy file/folder từ host vào guest trước khi chạy provisioner khác
config.vm.provision "file", source: "~/.gitconfig", destination: ".gitconfig"
config.vm.provision "file", source: "./configs/app.conf", destination: "/tmp/app.conf"
```

### Thứ tự thực thi

Provisioners chạy theo **thứ tự khai báo trong Vagrantfile**, tuần tự, mỗi cái phải xong mới tới cái tiếp theo — dùng để đảm bảo dependency (VD: `file` provisioner copy config trước, `shell` provisioner đọc config đó sau).

```ruby
config.vm.provision "file", source: "app.conf", destination: "/tmp/app.conf"
config.vm.provision "shell", inline: "cp /tmp/app.conf /etc/app/app.conf"
config.vm.provision "ansible", playbook: "site.yml"
```

---

## Gotchas

- **`ansible` provisioner cần Ansible trên HOST, không phải guest**: lỗi phổ biến nhất — quên rằng chế độ `ansible` (không phải `ansible_local`) chạy từ máy host, nên host (macOS/Linux) phải cài `ansible` package trước.
- **Docker provider `has_ssh = true` cần base image có sshd**: image Docker thông thường (VD `ubuntu:22.04` gốc) không có SSH server — phải build image riêng có cài `openssh-server` hoặc dùng box Docker chuyên dụng, nếu không `vagrant ssh` sẽ fail.
- **`run: "once"` không tự re-run khi script thay đổi**: sửa nội dung shell script rồi `vagrant up` lại (VM đã tồn tại) sẽ KHÔNG chạy lại provisioner mặc định — phải `vagrant provision` thủ công hoặc `vagrant up --provision`.
- **VirtualBox + Hyper-V không chạy đồng thời trên Windows**: bật Hyper-V feature (kể cả gián tiếp qua WSL2/Docker Desktop) có thể làm VirtualBox chạy chậm hẳn hoặc lỗi — do cả hai đều cần quyền truy cập hardware virtualization (VT-x) độc quyền.
- **Provisioner Ansible tự tạo inventory nếu không set `inventory_path`**: Vagrant tự sinh inventory tạm dựa trên machine trong Vagrantfile — tiện cho quick test, nhưng nếu muốn dùng đúng inventory production-like thì phải set `ansible.inventory_path` rõ ràng.
