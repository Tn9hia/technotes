---
title: Vagrant Core Concepts
tags:
  - vagrant
  - iac
  - core
date: 2026-07-18
---

# Vagrant Core Concepts

## Cách hoạt động

```
Vagrantfile (Ruby DSL)
    │
    ▼
vagrant up          ← đọc config, gọi provider API (VirtualBox/libvirt/Docker...)
    │
    ▼
Provider tạo VM/container từ box (base image)
    │
    ▼
Vagrant chạy provisioners (shell/ansible/docker...) lần đầu boot
    │
    ▼
.vagrant/           ← lưu machine ID, SSH config để lệnh sau tái sử dụng
```

Vagrant **không có state file** kiểu Terraform — nó hỏi trực tiếp provider "VM này còn sống không, ID bao nhiêu" mỗi lần chạy lệnh. `.vagrant/machines/<name>/<provider>/id` chỉ lưu ID để tra cứu nhanh.

---

## Vagrantfile

File cấu hình chính, viết bằng Ruby nhưng hầu hết chỉ cần biết cú pháp DSL — không cần biết Ruby sâu.

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/bionic64"
  config.vm.box_version = "1.0.282"        # pin version — tránh box tự update phá vỡ reproducibility

  config.vm.hostname = "dev-server"

  config.vm.network "private_network", ip: "192.168.56.10"
  config.vm.network "forwarded_port", guest: 80, host: 8080

  config.vm.synced_folder ".", "/vagrant"   # default, có thể override/disable

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
    vb.cpus   = 2
    vb.name   = "dev-server-vbox"
  end

  config.vm.provision "shell", inline: "apt-get update && apt-get install -y nginx"
end
```

**`"2"` trong `Vagrant.configure("2")`** là config version (API v2, dùng từ Vagrant 1.1+ tới hiện tại) — không phải version của Vagrant binary.

### Config precedence (khi có nhiều nguồn config)

```
1. Vagrantfile trong project (cùng thư mục chạy `vagrant up`)
2. ~/.vagrant.d/Vagrantfile          ← global, áp dụng mọi project
3. Box's Vagrantfile                 ← default config đóng gói sẵn trong box
```

Vagrant merge theo thứ tự: box default → global → project (project override tất cả). Đây là lý do một box "chạy khác nhau" giữa các máy nếu máy đó có `~/.vagrant.d/Vagrantfile` custom.

---

## Boxes

Box = base image (giống AMI của AWS hoặc Docker image) — chứa OS đã cài sẵn + Vagrant metadata.

```bash
# Thêm box vào local cache
vagrant box add hashicorp/bionic64
vagrant box add ubuntu/jammy64 --provider virtualbox

# Danh sách box đã có
vagrant box list

# Update box (khi Vagrantfile không pin version)
vagrant box update

# Xoá box không dùng
vagrant box remove hashicorp/bionic64 --box-version 1.0.282
```

### Nguồn box

```ruby
# Vagrant Cloud (registry chính thức, giống Docker Hub)
config.vm.box = "hashicorp/bionic64"

# URL trực tiếp tới file .box
config.vm.box = "custom-box"
config.vm.box_url = "https://example.com/boxes/custom.box"

# Box local đã build bằng Packer
config.vm.box = "my-custom-box"
config.vm.box_url = "file:///path/to/package.box"
```

**Pin `box_version`** luôn luôn — nếu không, `vagrant up` trên máy khác có thể pull version box mới hơn → môi trường không còn identical giữa các dev.

---

## Machine Lifecycle & States

```
NOT CREATED ──vagrant up──► RUNNING ──vagrant halt──► POWEROFF
     ▲                         │                          │
     │                         │──vagrant suspend──► SAVED│
     │                         │                          │
     └──────vagrant destroy────┴───────vagrant up──────────┘
```

| Lệnh | Tác động |
|---|---|
| `vagrant up` | Tạo mới (nếu chưa có) hoặc start lại VM |
| `vagrant halt` | Shutdown gracefully, giữ disk — boot lại nhanh hơn tạo mới |
| `vagrant suspend` / `resume` | Lưu RAM state ra disk — resume gần như tức thì, nhưng chiếm dung lượng lớn |
| `vagrant destroy` | Xoá VM + disk hoàn toàn — không thể resume |
| `vagrant reload` | `halt` + `up` — cần khi đổi network/provider config (không cần cho provisioner) |

---

## Multi-Machine Setup

Một Vagrantfile định nghĩa nhiều VM — hữu ích để test cluster (web + db, hoặc mô phỏng multi-node).

```ruby
Vagrant.configure("2") do |config|
  config.vm.define "web" do |web|
    web.vm.box = "hashicorp/bionic64"
    web.vm.network "private_network", ip: "192.168.56.10"
    web.vm.provision "shell", path: "provisioning/web.sh"
  end

  config.vm.define "db" do |db|
    db.vm.box = "hashicorp/bionic64"
    db.vm.network "private_network", ip: "192.168.56.11"
    db.vm.provision "shell", path: "provisioning/db.sh"
  end

  # Config chung cho tất cả machines (đặt ngoài define block)
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 1024
  end
end
```

```bash
vagrant up               # tạo tất cả machines
vagrant up web           # chỉ tạo machine "web"
vagrant ssh web
vagrant ssh db
vagrant halt db
vagrant destroy -f web
```

**Machine đầu tiên định nghĩa** là default cho các lệnh không chỉ định tên (`vagrant ssh` → SSH vào machine đầu tiên).

---

## Triggers

Chạy script tại các thời điểm trong lifecycle — không cần tới trong VM (khác provisioner, chạy trên **host**).

```ruby
config.vm.provision "shell", inline: "echo hello"

config.trigger.before :up do |trigger|
  trigger.info = "Đang chuẩn bị khởi động VM..."
  trigger.run = { inline: "echo 'pre-up hook trên host'" }
end

config.trigger.after :destroy do |trigger|
  trigger.info = "VM đã bị xoá — dọn dẹp local resources"
  trigger.run = { path: "scripts/cleanup.sh" }
end
```

Dùng cho: cleanup local file sau `destroy`, notify Slack khi `up` xong, backup trước `halt`.

---

## Environment Variables & `.env`

```ruby
# Đọc biến môi trường trong Vagrantfile (thuần Ruby)
memory = ENV['VM_MEMORY'] || "2048"

config.vm.provider "virtualbox" do |vb|
  vb.memory = memory
end
```

```bash
VM_MEMORY=4096 vagrant up
```

---

## Gotchas

- **Không pin `box_version` → drift giữa các máy**: box tự update ngầm khi provider default fetch "latest" — 2 dev chạy `vagrant up` khác thời điểm có thể có OS patch level khác nhau. Luôn set `config.vm.box_version`.
- **`vagrant reload` cần thiết sau khi đổi network/provider config**: chỉ sửa Vagrantfile không tự áp dụng cho VM đang chạy — phải `reload` (hoặc `destroy` + `up`) để nhận config mới. Sửa provisioner thì chỉ cần `vagrant provision`.
- **`.vagrant/` chứa absolute path**: metadata trong `.vagrant/` tham chiếu tuyệt đối tới thư mục project. Di chuyển/rename folder project → Vagrant "mất" VM (phải `vagrant global-status --prune` hoặc trỏ lại).
- **Machine ID trùng khi copy project**: copy cả `.vagrant/` sang máy khác/project khác → Vagrant nghĩ VM đã tồn tại nhưng thực ra provider không có VM đó → lỗi. Luôn `.gitignore` `.vagrant/`.
- **`Vagrant.configure("2")` không phải version binary**: nhầm lẫn phổ biến — đây là API config version, gần như luôn luôn là `"2"` bất kể Vagrant version đang cài.
