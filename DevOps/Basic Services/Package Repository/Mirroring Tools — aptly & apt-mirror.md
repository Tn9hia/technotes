---
title: Mirroring Tools — aptly & apt-mirror
tags:
  - apt
  - aptly
  - apt-mirror
  - mirror
  - deep-dive
date: 2026-04-26
---

# Mirroring Tools — aptly & apt-mirror

## So sánh tổng quan

| | apt-mirror | aptly |
|---|---|---|
| Snapshot | Không | Có (immutable) |
| Rollback | Không | Có |
| API | Không | REST API |
| Filter packages | Không | Có (query) |
| Multiple distros | Có | Có |
| Disk dedup | Không | Có (content-addressed) |
| Complexity | Đơn giản | Phức tạp hơn |
| Use case | Mirror đơn giản, 1 version | Production, audit, CI/CD |

**Khi nào dùng cái nào:**
- `apt-mirror`: lab, internal mirror đơn giản, không cần rollback
- `aptly`: production, cần snapshot/rollback, cần filter package, CI/CD pipeline

---

## apt-mirror

### Cài đặt

```bash
apt-get install apt-mirror
```

### Config `/etc/apt/mirror.list`

```bash
# Thư mục lưu mirror
set base_path    /srv/mirror

# Số parallel download
set nthreads     20

# Không tạo thư mục ~
set _tilde       0

# Ubuntu 22.04
deb http://archive.ubuntu.com/ubuntu jammy main restricted universe
deb http://archive.ubuntu.com/ubuntu jammy-updates main restricted universe
deb http://security.ubuntu.com/ubuntu jammy-security main restricted universe

# Chỉ amd64 (bỏ arm64, i386)
deb-amd64 http://archive.ubuntu.com/ubuntu jammy main restricted
```

### Chạy sync

```bash
apt-mirror

# Log output:
# Downloading 12345 index files using 20 threads...
# ...
# 45.2 GiB will be downloaded into archive.
# Downloading 8921 archive files using 20 threads...
# [100%]
```

### Structure sau sync

```
/srv/mirror/
├── mirror/
│   └── archive.ubuntu.com/
│       └── ubuntu/
│           ├── dists/
│           └── pool/
├── skel/        ← metadata chỉ
└── var/         ← state files
```

Nginx root trỏ vào `/srv/mirror/mirror/archive.ubuntu.com/ubuntu/`

### Cleanup package cũ

```bash
# apt-mirror tạo script cleanup tự động
bash /srv/mirror/var/clean.sh
```

---

## aptly

### Cài đặt

```bash
# Thêm repo aptly
echo "deb http://repo.aptly.info/ squeeze main" > /etc/apt/sources.list.d/aptly.list
curl -fsSL https://www.aptly.info/pubkey.txt | gpg --dearmor > \
  /etc/apt/trusted.gpg.d/aptly.gpg
apt-get update && apt-get install aptly
```

### Config `~/.aptly.conf`

```json
{
  "rootDir": "/srv/mirror",
  "downloadConcurrency": 8,
  "downloadSpeedLimit": 0,
  "architectures": ["amd64"],
  "dependencyFollowSuggests": false,
  "dependencyFollowRecommends": false,
  "gpgDisableSign": false,
  "gpgDisableVerify": false,
  "gpgProvider": "gpg",
  "storageBackend": "filesystem"
}
```

### Workflow cơ bản

```
mirror create → mirror update → snapshot create → publish snapshot
                     ↑               ↑
                  (cron)        (immutable point-in-time)
```

#### 1. Import GPG key

```bash
# Ubuntu key
gpg --no-default-keyring \
  --keyring /usr/share/keyrings/ubuntu-archive-keyring.gpg \
  --export | aptly key add -

# Hoặc import trực tiếp
gpg --keyserver hkp://keyserver.ubuntu.com:80 \
  --recv-keys 871920D1991BC93C 3B4FE6ACC0B21F32
gpg --export 871920D1991BC93C 3B4FE6ACC0B21F32 | aptly key add -
```

#### 2. Tạo mirror

```bash
aptly mirror create \
  -architectures=amd64 \
  ubuntu-jammy-main \
  http://archive.ubuntu.com/ubuntu \
  jammy \
  main restricted universe

aptly mirror create \
  -architectures=amd64 \
  ubuntu-jammy-security \
  http://security.ubuntu.com/ubuntu \
  jammy-security \
  main restricted universe

aptly mirror create \
  -architectures=amd64 \
  ubuntu-jammy-updates \
  http://archive.ubuntu.com/ubuntu \
  jammy-updates \
  main restricted universe

# Xem danh sách mirror
aptly mirror list
```

#### 3. Sync (update)

```bash
aptly mirror update ubuntu-jammy-main
aptly mirror update ubuntu-jammy-security
aptly mirror update ubuntu-jammy-updates
```

#### 4. Tạo snapshot

```bash
DATE=$(date +%Y%m%d)

aptly snapshot create jammy-main-${DATE} from mirror ubuntu-jammy-main
aptly snapshot create jammy-security-${DATE} from mirror ubuntu-jammy-security
aptly snapshot create jammy-updates-${DATE} from mirror ubuntu-jammy-updates

# Merge snapshots thành 1 (optional — để publish 1 endpoint)
aptly snapshot merge jammy-full-${DATE} \
  jammy-main-${DATE} \
  jammy-security-${DATE} \
  jammy-updates-${DATE}

# Xem snapshots
aptly snapshot list
```

#### 5. Publish snapshot

```bash
# Lần đầu: publish
aptly publish snapshot \
  -distribution=jammy \
  -component=main \
  jammy-full-${DATE} \
  filesystem:ubuntu:

# Lần sau: switch sang snapshot mới
aptly publish switch \
  -component=main \
  jammy \
  filesystem:ubuntu: \
  jammy-full-${DATE}
```

#### 6. Cleanup snapshot cũ

```bash
# List và drop snapshot cũ hơn 30 ngày
aptly snapshot list -raw | grep "jammy-full" | \
  sort | head -n -30 | \
  xargs -I{} aptly snapshot drop {}

# aptly tự cleanup orphaned packages
aptly db cleanup
```

### Filter packages (tính năng độc đáo của aptly)

```bash
# Chỉ lấy packages cần thiết (tiết kiệm disk)
aptly mirror create \
  -filter="Name (nginx), Name (curl), Name (vim)" \
  -filter-with-deps \
  ubuntu-jammy-minimal \
  http://archive.ubuntu.com/ubuntu \
  jammy \
  main

# Loại bỏ debug packages
aptly mirror create \
  -filter="! Name (% -dbg), ! Name (% -dbgsym)" \
  ubuntu-jammy-nodebug \
  http://archive.ubuntu.com/ubuntu \
  jammy main
```

### aptly REST API

```bash
# Bật API server
aptly api serve -listen=:8080

# Tạo mirror qua API
curl -X POST http://localhost:8080/api/mirrors \
  -H "Content-Type: application/json" \
  -d '{
    "Name": "ubuntu-jammy",
    "ArchiveURL": "http://archive.ubuntu.com/ubuntu",
    "Distribution": "jammy",
    "Components": ["main", "restricted"]
  }'

# Trigger update
curl -X PUT http://localhost:8080/api/mirrors/ubuntu-jammy/packages

# List snapshots
curl http://localhost:8080/api/snapshots
```

---

## Production sync script (aptly)

```bash
#!/bin/bash
# /usr/local/bin/mirror-sync.sh
set -euo pipefail

LOG=/var/log/aptly-sync.log
DATE=$(date +%Y%m%d_%H%M%S)
MIRRORS=(ubuntu-jammy-main ubuntu-jammy-security ubuntu-jammy-updates)
SNAPSHOTS=()

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a $LOG; }

log "=== Mirror sync started ==="

# Update tất cả mirrors
for mirror in "${MIRRORS[@]}"; do
    log "Updating mirror: $mirror"
    aptly mirror update "$mirror" >> $LOG 2>&1
    
    snap="${mirror}-${DATE}"
    log "Creating snapshot: $snap"
    aptly snapshot create "$snap" from mirror "$mirror" >> $LOG 2>&1
    SNAPSHOTS+=("$snap")
done

# Merge
merged="jammy-full-${DATE}"
log "Merging snapshots → $merged"
aptly snapshot merge "$merged" "${SNAPSHOTS[@]}" >> $LOG 2>&1

# Switch published
log "Publishing $merged"
aptly publish switch jammy filesystem:ubuntu: "$merged" >> $LOG 2>&1

# Cleanup cũ hơn 30 snapshots
log "Cleaning old snapshots"
aptly snapshot list -raw | grep "jammy-full-" | \
    sort | head -n -30 | \
    xargs -r -I{} aptly snapshot drop {} >> $LOG 2>&1

aptly db cleanup >> $LOG 2>&1

log "=== Mirror sync completed ==="
```

```bash
# Cron: chạy lúc 3:00 AM mỗi ngày
0 3 * * * root /usr/local/bin/mirror-sync.sh
```

---

## Nginx config chi tiết

```nginx
server {
    listen 80;
    server_name mirror.internal.com;

    # aptly filesystem publish path mặc định
    root /srv/mirror/public/ubuntu;

    autoindex on;
    autoindex_exact_size off;
    autoindex_localtime on;

    # Không log cho metadata files — quá nhiều noise
    location ~* \.(gz|xz|bz2|diff|dsc)$ {
        access_log off;
        expires 1h;
    }

    # Cache lâu cho .deb — immutable content
    location ~* \.deb$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # InRelease và Release — cache ngắn
    location ~* (InRelease|Release|Release\.gpg)$ {
        expires 5m;
        add_header Cache-Control "public, must-revalidate";
    }

    # Limit connections để không bị hammer
    limit_conn_zone $binary_remote_addr zone=apt_conn:10m;
    limit_conn apt_conn 10;
    limit_rate 50m;   # 50MB/s per connection

    access_log /var/log/nginx/mirror-access.log;
    error_log  /var/log/nginx/mirror-error.log;
}
```

---

## Troubleshooting

```bash
# apt update báo lỗi GPG
# W: GPG error: http://mirror.internal ...
# → Import key cho client
curl http://mirror.internal/gpg.key | gpg --dearmor > \
  /etc/apt/trusted.gpg.d/internal.gpg

# apt update báo lỗi hash mismatch
# E: Failed to fetch ... Hash Sum mismatch
# → Mirror đang sync dở dang → đợi sync xong hoặc rollback snapshot
aptly publish switch jammy filesystem:ubuntu: jammy-full-<previous-date>

# 404 khi apt update
# → Kiểm tra nginx root path
nginx -T | grep root
ls -la /srv/mirror/public/ubuntu/dists/

# Mirror update fail do GPG
# → Re-import key
gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 871920D1991BC93C
gpg --export 871920D1991BC93C | aptly key add -

# Disk full giữa chừng
# → Check disk
df -h /srv/mirror
# → Cleanup rác
aptly db cleanup
bash /srv/mirror/var/clean.sh   # apt-mirror
```
