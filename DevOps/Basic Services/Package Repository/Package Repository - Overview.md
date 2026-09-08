---
title: Package Repository & Mirror
tags:
  - apt
  - mirror
  - infra
  - basic-services
  - GlobalTechJSC
date: 2026-04-26
status: in-progress
---

# Package Repository & Mirror — apt-mirror / aptly + Nginx

Tags: #apt #mirror #infra #basic-services
Last updated: 2026-04-26

---

## 1. What — Nó là cái gì?

**Package Repository Mirror** là bản sao local của một upstream APT repository (như `archive.ubuntu.com`, `debian.org`). Thay vì mỗi server kéo package từ internet, tất cả kéo từ mirror nội bộ.

Hai thành phần:
- **Mirroring tool** (`apt-mirror`, `aptly`, `debmirror`): sync package từ upstream về local disk
- **HTTP server** (`nginx`): serve thư mục đó như một APT repo endpoint

> Về bản chất: mày đang clone một subset của Debian/Ubuntu repo về máy, rồi dùng nginx làm "web server" để các client khác `apt install` từ đó.

---

## 2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?

Nếu không có internal mirror:

- **Tốn bandwidth**: 50 server cùng `apt upgrade` → mỗi cái tải riêng từ internet → bandwidth tốn gấp 50 lần
- **Air-gap không hoạt động**: môi trường không có internet (compliance, classified) hoàn toàn không install được package
- **Phụ thuộc upstream**: upstream bị down hoặc chậm → toàn bộ CI/CD pipeline bị block
- **Không reproducible**: hôm nay install `nginx=1.24.0`, tuần sau upstream đã lên `1.26.0` — không còn install được version cũ

**Use case trong On-Premise:**
- Offline/air-gap lab: không có internet
- Bandwidth tiết kiệm: sync 1 lần từ internet, distribute nội bộ
- Snapshot repo cho compliance: đóng băng version tại thời điểm audit
- CI/CD reproducibility: pin repo tại specific snapshot

---

## 3. When — Dùng khi nào / KHÔNG dùng khi nào?

**Dùng khi:**
- Môi trường air-gap hoặc restricted internet
- Số lượng server >= 5 (dưới đó dùng Squid cache APT đơn giản hơn)
- Cần reproducible build / audit trail
- Bandwidth nội bộ nhanh hơn đường internet đáng kể

**KHÔNG dùng khi:**
- Chỉ 1-2 server, đường internet tốt → overhead không đáng
- Không có người maintain mirror (sync fail, disk full, stale packages) → nguy hiểm hơn không có
- Chỉ cần cache (không cần offline) → Squid transparent proxy đơn giản hơn nhiều

> [!warning] Mirror cần người maintain
> Mirror stale (không sync) nguy hiểm hơn không có mirror: team tin tưởng mirror → không biết thiếu security patch. Phải có monitoring + alerting cho sync job.

---

## 4. Architecture — Nó nằm ở đâu trong hệ thống?

```
[Internet]
    |
    | HTTPS — archive.ubuntu.com, security.ubuntu.com
    v
[Mirror Server]
  ┌─────────────────────────────────┐
  │  aptly / apt-mirror (sync job)  │  ← cron hàng đêm
  │  /srv/mirror/ubuntu/            │  ← local disk (100–500 GB)
  │  nginx (:80)                    │  ← serve HTTP
  └─────────────────────────────────┘
    |
    | HTTP :80 — internal LAN
    v
[All Internal Servers]
  /etc/apt/sources.list.d/internal.list
  → deb http://mirror.internal.com/ubuntu jammy main restricted universe
```

### Repo structure trên disk

```
/srv/mirror/ubuntu/
├── dists/
│   └── jammy/
│       ├── InRelease          ← signed metadata (GPG)
│       ├── Release
│       ├── main/
│       │   └── binary-amd64/
│       │       ├── Packages.gz
│       │       └── Packages.xz
│       └── security/
└── pool/
    ├── main/
    │   └── n/nginx/
    │       └── nginx_1.24.0-1_amd64.deb
    └── universe/
```

→ Deep dive: [[APT Repository Structure]]

---

## 5. How — Cơ chế hoạt động

### 5.1 APT Client Flow

```
apt install nginx
  │
  ├─ 1. Đọc /etc/apt/sources.list → URL của repo
  ├─ 2. Fetch dists/jammy/InRelease → verify GPG signature
  ├─ 3. Fetch dists/jammy/main/binary-amd64/Packages.xz → package index
  ├─ 4. Tìm nginx trong index → lấy path đến .deb file
  ├─ 5. Fetch pool/main/n/nginx/nginx_1.24.0-1_amd64.deb
  └─ 6. dpkg -i → install
```

### 5.2 Mirror Sync Flow (aptly)

```
aptly mirror create ubuntu-jammy \
  http://archive.ubuntu.com/ubuntu jammy main restricted universe
  │
  ├─ 1. Fetch InRelease → verify signature
  ├─ 2. Fetch Packages index → danh sách tất cả .deb
  ├─ 3. Download .deb files (chỉ những gì chưa có)
  └─ 4. aptly snapshot create → immutable snapshot của repo tại thời điểm này
         │
         └─ aptly publish snapshot → tạo repo structure trên disk
                                      → nginx serve
```

### 5.3 GPG Signing

APT client verify InRelease bằng GPG public key. Khi publish mirror:
- Dùng GPG key của upstream (re-publish signed metadata) → client cần trust upstream key
- Hoặc sign lại bằng key riêng → client cần import key riêng của mình

→ Deep dive: [[APT Repository Structure#GPG Signing]]

---

## 6. Key Config — Cấu hình cần nhớ

### 6.1 aptly — Mirror setup

```bash
# Khởi tạo aptly config
cat > ~/.aptly.conf << 'EOF'
{
  "rootDir": "/srv/mirror",
  "downloadConcurrency": 4,
  "architectures": ["amd64"],
  "gpgDisableSign": false,
  "gpgDisableVerify": false
}
EOF

# Import GPG key của Ubuntu
gpg --keyserver keyserver.ubuntu.com --recv-keys 871920D1991BC93C
gpg --export 871920D1991BC93C | aptly key add -

# Tạo mirror
aptly mirror create ubuntu-jammy-main \
  http://archive.ubuntu.com/ubuntu \
  jammy main restricted universe

aptly mirror create ubuntu-jammy-security \
  http://security.ubuntu.com/ubuntu \
  jammy-security main restricted universe

# Sync lần đầu (lâu — vài chục GB)
aptly mirror update ubuntu-jammy-main
aptly mirror update ubuntu-jammy-security

# Tạo snapshot
aptly snapshot create jammy-main-$(date +%Y%m%d) \
  from mirror ubuntu-jammy-main

# Publish snapshot
aptly publish snapshot jammy-main-$(date +%Y%m%d) \
  -distribution=jammy \
  -component=main \
  filesystem:ubuntu:
```

### 6.2 apt-mirror — config đơn giản hơn

```bash
# /etc/apt/mirror.list
set base_path    /srv/mirror
set nthreads     10
set _tilde       0

# Ubuntu 22.04 jammy
deb http://archive.ubuntu.com/ubuntu jammy main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu jammy-updates main restricted universe multiverse
deb http://security.ubuntu.com/ubuntu jammy-security main restricted universe multiverse

# Chỉ amd64
deb-amd64 http://archive.ubuntu.com/ubuntu jammy main restricted

# Chạy sync
apt-mirror
```

### 6.3 Nginx serve mirror

```nginx
# /etc/nginx/sites-available/apt-mirror
server {
    listen 80;
    server_name mirror.internal.com;

    root /srv/mirror/public;   # aptly publish path
    # hoặc /srv/mirror/mirror  # apt-mirror path

    autoindex on;   # cho phép browse directory

    location / {
        try_files $uri $uri/ =404;
    }

    # Tắt access log cho file nhỏ (Packages, Release)
    # để giảm noise — tuỳ chọn
    location ~* \.(gz|xz|bz2)$ {
        access_log off;
    }

    # Cache headers cho .deb files
    location ~* \.deb$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

### 6.4 Client config

```bash
# /etc/apt/sources.list.d/internal-mirror.list
deb http://mirror.internal.com/ubuntu jammy main restricted universe
deb http://mirror.internal.com/ubuntu jammy-security main restricted universe
deb http://mirror.internal.com/ubuntu jammy-updates main restricted universe

# Comment out hoặc xoá sources.list gốc
# mv /etc/apt/sources.list /etc/apt/sources.list.bak

# Import GPG key (nếu mirror tự sign)
curl -fsSL http://mirror.internal.com/gpg.key | \
  gpg --dearmor | \
  tee /etc/apt/trusted.gpg.d/internal-mirror.gpg

# Test
apt update
apt install nginx
```

### 6.5 Cron sync job

```bash
# /etc/cron.d/apt-mirror-sync
# Sync hàng đêm lúc 2:00 AM
0 2 * * * root /usr/bin/apt-mirror >> /var/log/apt-mirror.log 2>&1

# aptly version (thêm bước snapshot + publish)
0 2 * * * root /usr/local/bin/aptly-sync.sh >> /var/log/aptly-sync.log 2>&1
```

```bash
#!/bin/bash
# /usr/local/bin/aptly-sync.sh
set -e
DATE=$(date +%Y%m%d)

aptly mirror update ubuntu-jammy-main
aptly snapshot create jammy-main-${DATE} from mirror ubuntu-jammy-main

# Switch published snapshot
aptly publish switch jammy filesystem:ubuntu: jammy-main-${DATE}

# Cleanup snapshots cũ hơn 30 ngày
aptly snapshot list -raw | grep jammy-main | head -n -30 | \
  xargs -I{} aptly snapshot drop {}
```

> [!tip] Config hay bị sai
> - Nginx `root` trỏ sai path → 404 khi client `apt update`
> - Quên import GPG key vào aptly → sync fail với "signature verification failed"
> - Disk full giữa chừng khi sync → repo corrupt → client bị lỗi lạ
> - `autoindex on` thiếu → apt không browse được directory listing

---

## 7. Security Considerations

### Attack surface

- **Mirror poisoning**: attacker replace .deb files với malicious version → client install malware
- **GPG key compromise**: nếu dùng key riêng để sign, key bị lộ → attacker sign malicious packages
- **HTTP (non-TLS)**: ai đó trên mạng nội bộ có thể MITM và inject .deb giả (hiếm nhưng có thể)
- **Stale security packages**: mirror không sync → client không nhận được security patch

### Hardening checklist

- [ ] Luôn verify GPG signature của upstream (không dùng `gpgDisableVerify: true` trên production)
- [ ] Nếu dùng HTTP nội bộ: đảm bảo network segment được kiểm soát
- [ ] Dùng HTTPS nếu có thể (nginx + cert nội bộ)
- [ ] Monitor disk space — cron sync sẽ fail silently nếu disk full
- [ ] Alert khi sync job fail — stale repo là silent risk
- [ ] Không expose mirror port ra internet
- [ ] Giới hạn write access vào `/srv/mirror` — chỉ mirror service user

---

## 8. Ops Runbook — Production Notes

### Health check

```bash
# Test client có thể apt update được không
curl -s http://mirror.internal.com/ubuntu/dists/jammy/InRelease | head -5

# Kiểm tra nginx đang serve
curl -I http://mirror.internal.com/ubuntu/dists/jammy/Release

# Kiểm tra disk space
df -h /srv/mirror

# aptly: xem danh sách mirror và trạng thái
aptly mirror list
aptly mirror show ubuntu-jammy-main

# Kiểm tra published repos
aptly publish list
```

### Log quan trọng

```bash
# apt-mirror sync log
tail -f /var/log/apt-mirror.log
# Tìm lỗi download
grep -E "ERROR|Failed|404" /var/log/apt-mirror.log

# Nginx access log — theo dõi client activity
tail -f /var/log/nginx/access.log | grep "apt"

# Kiểm tra sync hoàn thành chưa
ls -lh /srv/mirror/ubuntu/mirror/archive.ubuntu.com/ubuntu/dists/jammy/InRelease
```

### Metrics cần monitor

| Metric | Alert khi |
|--------|-----------|
| Disk usage `/srv/mirror` | > 85% |
| Cron sync job exit code | != 0 |
| `InRelease` file mtime | > 25h (sync không chạy) |
| Nginx 404 rate | Tăng đột biến |

### Disk usage estimate

| Distro | Components | Dung lượng xấp xỉ |
|--------|------------|-------------------|
| Ubuntu 22.04 main+restricted | amd64 only | ~80 GB |
| Ubuntu 22.04 + universe | amd64 only | ~200 GB |
| Ubuntu 22.04 full (all arch) | tất cả | ~500 GB |
| Debian 12 main | amd64 only | ~60 GB |

---

## 9. Gotchas & Lessons Learned

> Phần này điền thêm khi có kinh nghiệm thực tế.

- **apt-mirror không có snapshot**: sync xong là overwrite ngay → không rollback được. Dùng aptly nếu cần reproducibility.
- **Lần sync đầu mất rất lâu**: Ubuntu main+universe ~200GB, với đường 100Mbps mất 4-5 tiếng. Lên kế hoạch trước, không chạy lúc giờ cao điểm.
- **`jammy-updates` và `jammy-security` phải sync riêng**: nhiều người chỉ mirror `jammy` mà quên `jammy-security` → server không nhận security patch.
- **Client cần `apt-get clean` sau khi đổi mirror**: cache .deb cũ từ internet repo có thể conflict.
- **aptly publish sau khi switch snapshot**: `aptly publish switch` update symlink nhanh, client thấy repo mới ngay mà không cần restart nginx.

---

## 10. Resources

- [aptly Official Docs](https://www.aptly.info/doc/)
- [apt-mirror man page](https://manpages.debian.org/apt-mirror)
- [Debian Repository Format](https://wiki.debian.org/DebianRepository/Format)
- [Ubuntu Archive Structure](https://help.ubuntu.com/community/Repositories)
- [Nginx as APT repo server](https://www.digitalocean.com/community/tutorials/how-to-create-a-simple-apt-repository)
