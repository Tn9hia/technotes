---
title: Vagrant
tags:
  - vagrant
  - iac
  - infrastructure-as-code
  - index
date: 2026-07-18
---

# Vagrant

Vagrant (HashiCorp) là tool để build và quản lý **local development environments** — VM/container reproducible, define bằng code (`Vagrantfile`, Ruby DSL). Vagrant không provision cloud infrastructure như Terraform; nó orchestrate provider (VirtualBox, libvirt, Docker, Hyper-V...) để tạo môi trường dev/test giống hệt nhau trên mọi máy — "it works on my machine" → "it works on every machine".

## Contents

| File | Nội dung |
|------|----------|
| [[Core Concepts]] | Vagrantfile, boxes, machine lifecycle, multi-machine setup, config precedence, triggers |
| [[Networking & Synced Folders]] | Private/public network, port forwarding, synced folder types (VirtualBox/NFS/rsync/SMB) |
| [[Providers & Provisioners]] | VirtualBox/libvirt/Docker/Hyper-V providers; Shell/Ansible/Docker provisioners |
| [[CLI Reference]] | Bảng lệnh tra cứu nhanh, box management, plugin, debug |

---

## Quick Reference

### Workflow cơ bản

```bash
vagrant init hashicorp/bionic64   # tạo Vagrantfile mẫu với box chỉ định
vagrant up                        # tạo + boot VM (download box nếu chưa có)
vagrant ssh                       # SSH vào máy (mặc định machine đầu tiên)
vagrant halt                      # shutdown VM (giữ disk)
vagrant suspend                   # save state, tắt nhanh (giống hibernate)
vagrant resume                    # resume từ suspend
vagrant reload                    # halt + up (áp dụng lại Vagrantfile config)
vagrant reload --provision        # reload + chạy lại provisioners
vagrant destroy                   # xoá VM hoàn toàn (confirm)
vagrant destroy -f                # xoá không hỏi confirm
```

### Trạng thái & thông tin

```bash
vagrant status                    # trạng thái VM trong project hiện tại
vagrant global-status             # tất cả VM Vagrant đang quản lý (mọi project)
vagrant global-status --prune     # dọn entry của VM đã bị xoá thủ công
vagrant box list                  # box đã download local
```

### Provisioning riêng lẻ

```bash
vagrant provision                 # chạy lại provisioners trên VM đang chạy
vagrant provision --provision-with shell   # chỉ chạy provisioner tên "shell"
```

---

## Vagrant vs Terraform vs Ansible

| | Vagrant | Terraform | Ansible |
|---|---|---|---|
| Mục đích chính | Local dev/test environment | Cloud/infra provisioning | Configuration management |
| Đơn vị quản lý | VM/container trên máy dev | Cloud resources (VPC, EC2, RDS...) | Config state trên existing host |
| State tracking | Không (dựa vào provider, `.vagrant/` metadata) | `.tfstate` (source of truth) | Không (idempotent re-check mỗi lần) |
| Chạy ở đâu | Máy local (laptop, CI runner) | Local hoặc CI, gọi cloud API | Control node → SSH tới target |
| Scope thường gặp | 1 dev, sandbox, demo, CI test env | Team, production infra | Cả dev lẫn production config |

**Kết hợp thực tế:** Vagrant tạo VM (`vagrant up`) → Vagrant gọi Ansible provisioner (`config.vm.provision "ansible"`) → Ansible configure VM y hệt cách sẽ configure production server. Đây là pattern phổ biến nhất để **test Ansible playbook locally trước khi chạy trên production**.

---

## Folder Structure điển hình

```
my-project/
├── Vagrantfile              ← định nghĩa machine(s), provider, network, provisioning
├── .vagrant/                 ← metadata (machine ID, SSH config...) — KHÔNG commit
├── provisioning/
│   ├── bootstrap.sh          ← shell provisioner
│   └── playbook.yml          ← ansible provisioner
└── .gitignore                 ← ignore .vagrant/
```

`.vagrant/` chứa state runtime (VM ID của provider, SSH keys sinh ra) — tương tự mục đích với `.terraform/`, luôn add vào `.gitignore`.

---

## Cross-References

- Provisioning bằng Ansible: xem [[Providers & Provisioners]] + [[Ansible - Overview]] (control node chạy trực tiếp từ host, không qua Vagrant khi target là production)
- So sánh với cách Terraform quản lý lifecycle: [[State Management]]
- Packer build custom base box (bake sẵn dependencies) để dùng làm `config.vm.box`: xem [[Packer]]
