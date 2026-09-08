---
title: Vagrant CLI Reference
tags:
  - vagrant
  - cli
  - cheatsheet
  - reference
date: 2026-07-18
---

# Vagrant CLI Reference

> Bảng tra cứu nhanh lệnh, plugin, và debug recipe. Xem [[Core Concepts]], [[Networking & Synced Folders]], [[Providers & Provisioners]] cho nội dung chi tiết theo chủ đề.

---

## Lifecycle Commands

| Lệnh | Mô tả |
|------|-------|
| `vagrant init <box>` | Tạo Vagrantfile mẫu với box chỉ định |
| `vagrant up` | Tạo mới hoặc start VM |
| `vagrant up --provider=X` | Chỉ định provider (nếu Vagrantfile hỗ trợ nhiều) |
| `vagrant halt` | Shutdown, giữ disk |
| `vagrant suspend` / `resume` | Save RAM state / resume |
| `vagrant reload` | halt + up (áp dụng lại config Vagrantfile) |
| `vagrant reload --provision` | reload + chạy lại provisioners |
| `vagrant destroy` / `-f` | Xoá VM hoàn toàn (force = không hỏi confirm) |
| `vagrant provision` | Chạy lại provisioners trên VM đang chạy |
| `vagrant provision --provision-with X` | Chỉ chạy provisioner tên X |

## Info & Status

| Lệnh | Mô tả |
|------|-------|
| `vagrant status` | Trạng thái machine trong project hiện tại |
| `vagrant global-status` | Tất cả VM Vagrant quản lý, mọi project |
| `vagrant global-status --prune` | Dọn entry của VM đã xoá thủ công qua provider |
| `vagrant ssh-config` | In OpenSSH config để dùng với `ssh` trực tiếp |

## SSH & Files

| Lệnh | Mô tả |
|------|-------|
| `vagrant ssh [name]` | SSH vào machine (mặc định machine đầu tiên) |
| `vagrant ssh -c "command"` | Chạy 1 command qua SSH rồi thoát |
| `vagrant rsync` | Sync 1 lần (synced folder type rsync) |
| `vagrant rsync-auto` | Watch file changes, tự sync liên tục |

## Box Management

| Lệnh | Mô tả |
|------|-------|
| `vagrant box add <name>` | Thêm box vào local cache |
| `vagrant box add <name> --provider X` | Chỉ định provider khi box có nhiều variant |
| `vagrant box list` | Danh sách box đã có local |
| `vagrant box update` | Update box lên version mới nhất (nếu không pin) |
| `vagrant box outdated` | Kiểm tra box hiện tại có bản mới không |
| `vagrant box remove <name> --box-version X` | Xoá 1 version cụ thể |
| `vagrant package` | Đóng gói VM đang chạy thành `.box` mới (tự build box) |

## Plugin Management

```bash
vagrant plugin install vagrant-libvirt
vagrant plugin install vagrant-vbguest    # tự update VirtualBox Guest Additions
vagrant plugin list
vagrant plugin update
vagrant plugin uninstall vagrant-libvirt
```

## Snapshot (thử nghiệm nhanh, rollback)

```bash
vagrant snapshot save before-upgrade
vagrant snapshot list
vagrant snapshot restore before-upgrade
vagrant snapshot delete before-upgrade
```

Hữu ích trước khi thử thay đổi rủi ro (upgrade OS package, chạy script chưa test) — rollback nhanh hơn `destroy` + `up` lại từ đầu.

---

## Environment Variables

```bash
export VAGRANT_LOG=debug              # info | debug — bật verbose logging
export VAGRANT_CWD=/path/to/project    # chạy lệnh như đang ở thư mục khác
export VAGRANT_DOTFILE_PATH=.vagrant-custom   # đổi vị trí .vagrant/ (multi-provider testing)
export VAGRANT_DEFAULT_PROVIDER=libvirt        # provider mặc định khi không chỉ định
export VAGRANT_HOME=~/.vagrant.d       # nơi lưu box cache + global config
```

---

## Quick Debug Recipes

```bash
# "VM không boot được, xem chi tiết"
VAGRANT_LOG=debug vagrant up 2>&1 | tee vagrant-debug.log

# "Provider nào đang thực sự dùng cho machine này?"
vagrant status

# "SSH bằng tay thay vì `vagrant ssh` (khi cần debug SSH agent/key)"
vagrant ssh-config > /tmp/vssh.conf
ssh -F /tmp/vssh.conf default

# "Box bị lỗi, thử tạo VM sạch"
vagrant destroy -f
vagrant box update
vagrant up

# "Xem VM thật trong VirtualBox có khớp Vagrant nghĩ không"
VBoxManage list vms
vagrant global-status

# "Provisioner chạy lỗi giữa chừng, muốn chạy lại từ đầu không rebuild VM"
vagrant provision
```
