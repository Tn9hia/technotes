---
title: Vagrant Networking & Synced Folders
tags:
  - vagrant
  - networking
  - synced-folders
date: 2026-07-18
---

# Vagrant Networking & Synced Folders

## Networking Modes

```
┌─────────────────────────────────────────────────────────┐
│  Host machine                                            │
│                                                            │
│  forwarded_port: host:8080 ──────┐                        │
│                                    │                       │
│  private_network: 192.168.56.x ───┼──► ┌────────────────┐│
│                                    │    │   Guest VM      ││
│  public_network (bridged) ────────┴───►│                 ││
│                                         └────────────────┘│
└─────────────────────────────────────────────────────────┘
```

### Forwarded Port

Map port của guest VM ra port trên host — dùng khi cần truy cập service từ host qua `localhost`.

```ruby
config.vm.network "forwarded_port", guest: 80, host: 8080
config.vm.network "forwarded_port", guest: 443, host: 8443

# auto_correct: nếu host port bị chiếm, tự tìm port khác thay vì fail
config.vm.network "forwarded_port", guest: 3000, host: 3000, auto_correct: true

# Chỉ forward khi guest bind trên 0.0.0.0 (không phải chỉ 127.0.0.1 trong guest)
```

```bash
curl http://localhost:8080   # từ host, tới service chạy trên guest:80
```

### Private Network

Tạo network riêng giữa host và guest (hoặc giữa nhiều guest) — không expose ra ngoài LAN thật. Dùng cho multi-machine cluster hoặc khi cần IP cố định để test.

```ruby
# Static IP
config.vm.network "private_network", ip: "192.168.56.10"

# DHCP (Vagrant tự cấp IP trong dải private)
config.vm.network "private_network", type: "dhcp"
```

Dùng static IP khi nhiều machine cần biết địa chỉ nhau trước (VD: web machine cấu hình trỏ tới db machine ở IP cố định).

### Public Network (Bridged)

Guest VM nhận IP trực tiếp trên LAN thật (bridged tới network interface của host) — máy khác trong mạng LAN truy cập được guest như một máy vật lý.

```ruby
config.vm.network "public_network", bridge: "en0: Wi-Fi (AirPort)"

# Static IP trên network thật (thay vì DHCP từ router)
config.vm.network "public_network", ip: "192.168.1.50"
```

Vagrant sẽ hỏi chọn network interface nếu không chỉ định `bridge` — cần khi có nhiều NIC (Ethernet + Wi-Fi).

---

## So sánh 3 loại network

| | Forwarded Port | Private Network | Public Network |
|---|---|---|---|
| Truy cập từ | Chỉ host (qua localhost) | Host + guest khác trong cùng private net | Toàn bộ LAN |
| Cần IP cố định | Không (dùng port) | Thường có (multi-machine) | Tuỳ |
| Use case | Web app dev, expose 1 service | Multi-machine cluster, service-to-service | Demo cho máy khác trong LAN, test mobile app |
| Isolation | Cao nhất | Trung bình (chỉ giữa VMs của Vagrant) | Thấp nhất (visible toàn LAN) |

---

## Synced Folders

Đồng bộ file giữa host và guest — sửa code trên host (IDE quen thuộc), chạy trong guest.

```ruby
# Default: thư mục project (nơi có Vagrantfile) → /vagrant trong guest
# Disable nếu không cần:
config.vm.synced_folder ".", "/vagrant", disabled: true

# Custom mapping
config.vm.synced_folder "./app", "/home/vagrant/app"

# Đọc-only
config.vm.synced_folder "./config", "/etc/app-config", mount_options: ["ro"]
```

### Loại synced folder theo provider

```ruby
# VirtualBox shared folder (default với provider VirtualBox)
# — chậm nhất, nhưng không cần config gì thêm, hoạt động cross-platform

# NFS — nhanh hơn nhiều, chỉ hoạt động tốt trên Linux/macOS host
config.vm.synced_folder ".", "/vagrant", type: "nfs", nfs_udp: false

# rsync — copy 1 chiều host→guest, không realtime, cần `vagrant rsync-auto`
config.vm.synced_folder ".", "/vagrant", type: "rsync",
  rsync__exclude: [".git/", "node_modules/"]

# SMB — dùng khi host là Windows
config.vm.synced_folder ".", "/vagrant", type: "smb"
```

```bash
# rsync cần trigger thủ công hoặc chạy watcher
vagrant rsync              # sync 1 lần
vagrant rsync-auto         # watch file changes, tự sync liên tục
```

---

## So sánh Synced Folder Types

| Type | Tốc độ | Realtime | Yêu cầu |
|---|---|---|---|
| VirtualBox (default) | Chậm | Có | Guest Additions cài trong box |
| NFS | Nhanh | Có | NFS server (host), không cần trên Windows host |
| rsync | Nhanh (vì copy, không mount) | Không (cần `rsync-auto` hoặc trigger) | rsync binary trên host |
| SMB | Trung bình | Có | Windows host, hoặc Samba trên host khác |

---

## Gotchas

- **VirtualBox synced folder chậm với I/O nhiều file nhỏ**: `node_modules/`, `.git/` qua VirtualBox shared folder có thể chậm gấp 10-50 lần so với native filesystem. Cân nhắc NFS hoặc rsync cho project có nhiều file, hoặc exclude các thư mục nặng I/O.
- **NFS cần host cho phép NFS server**: lần đầu `vagrant up` với `type: "nfs"` sẽ yêu cầu sudo password trên host để cấu hình `/etc/exports` — bình thường, không phải lỗi.
- **rsync không tự động 2 chiều**: sửa file trong guest sẽ KHÔNG sync ngược lại host. Luôn coi host là source of truth khi dùng rsync.
- **`forwarded_port` guest phải bind `0.0.0.0`**: nếu service trong guest chỉ listen `127.0.0.1:80`, forward từ host sẽ không kết nối được dù port map đúng — service phải bind trên tất cả interface của guest.
- **Bridged network và VPN**: `public_network` bridged có thể không hoạt động khi host đang kết nối VPN (VPN thường chiếm quyền route). Tắt VPN hoặc dùng `private_network` thay thế khi gặp vấn đề.
- **Nhiều private_network cùng dải IP giữa các project khác nhau**: 2 project Vagrant riêng biệt cùng dùng `192.168.56.10` có thể conflict nếu chạy đồng thời — đổi dải IP theo project khi cần chạy song song.
