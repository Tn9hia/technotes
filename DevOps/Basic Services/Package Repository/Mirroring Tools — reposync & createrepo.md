---
title: Mirroring Tools — reposync & createrepo (RHEL)
tags:
  - rpm
  - dnf
  - reposync
  - createrepo
  - mirror
  - deep-dive
date: 2026-04-26
---

# Mirroring Tools — reposync & createrepo (RHEL/Rocky/AlmaLinux)

## Tổng quan tools

| Tool | Vai trò | Tương đương APT |
|------|---------|-----------------|
| `reposync` (dnf-utils) | Sync .rpm từ upstream repo | `apt-mirror` / `aptly mirror update` |
| `createrepo_c` | Tạo/update metadata (repodata/) | `apt-ftparchive` / `aptly publish` |
| `dnf` + `.repo` file | Client config | `apt` + `sources.list` |
| Nginx | Serve HTTP | Nginx (giống hệt) |

**Không có tool nào như aptly** (snapshot, rollback) trong RHEL ecosystem native. Nếu cần snapshot → dùng filesystem snapshot (LVM, Btrfs) hoặc build pipeline riêng.

---

## reposync

### Cài đặt

```bash
# RHEL/Rocky/Alma
dnf install dnf-utils
# hoặc
dnf install yum-utils   # legacy name, vẫn work

# Kiểm tra
reposync --version
```

### Workflow cơ bản

```
Cấu hình repo upstream trong .repo file
         ↓
    reposync --download
         ↓
   createrepo_c (tạo metadata)
         ↓
   nginx serve thư mục
```

### Config upstream repo (để reposync biết sync từ đâu)

```bash
# /etc/yum.repos.d/rocky-upstream.repo
[rocky9-baseos-upstream]
name=Rocky Linux 9 - BaseOS (upstream)
baseurl=https://download.rockylinux.org/pub/rocky/9/BaseOS/x86_64/os/
enabled=0        ← disabled để dnf install không dùng
gpgcheck=1
gpgkey=https://download.rockylinux.org/pub/rocky/RPM-GPG-KEY-Rocky-9

[rocky9-appstream-upstream]
name=Rocky Linux 9 - AppStream (upstream)
baseurl=https://download.rockylinux.org/pub/rocky/9/AppStream/x86_64/os/
enabled=0
gpgcheck=1
gpgkey=https://download.rockylinux.org/pub/rocky/RPM-GPG-KEY-Rocky-9

[rocky9-extras-upstream]
name=Rocky Linux 9 - Extras (upstream)
baseurl=https://download.rockylinux.org/pub/rocky/9/extras/x86_64/os/
enabled=0
gpgcheck=1
gpgkey=https://download.rockylinux.org/pub/rocky/RPM-GPG-KEY-Rocky-9
```

### reposync options quan trọng

```bash
reposync \
  --repoid=rocky9-baseos-upstream \     # repo ID trong .repo file
  --download-path=/srv/mirror/rocky/9/BaseOS/x86_64/os/ \
  --download-metadata \                 # download repodata luôn
  --delete \                            # xoá package không còn ở upstream
  --newest-only \                       # chỉ download version mới nhất (tiết kiệm disk)
  --gpgcheck \                          # verify GPG khi download
  --arch=x86_64                         # chỉ 1 arch
```

**Flag hay dùng:**
- `--newest-only`: tiết kiệm disk đáng kể — upstream thường giữ nhiều version cũ
- `--delete`: giữ mirror clean, không bị rác package cũ
- `--download-metadata`: bắt buộc nếu muốn serve repo mà không cần createrepo lại
- `--norepopath`: không tạo thư mục con theo tên repo (flat structure)

---

## createrepo_c

Sau khi `reposync --download-metadata`, metadata đã có. Nhưng nếu mày **thêm/xoá package thủ công** hoặc **tạo repo từ đầu**, phải chạy `createrepo_c`.

```bash
# Cài
dnf install createrepo_c

# Tạo metadata lần đầu
createrepo_c /srv/mirror/rocky/9/BaseOS/x86_64/os/

# Update sau khi thêm/xoá package
createrepo_c --update /srv/mirror/rocky/9/BaseOS/x86_64/os/

# Parallel workers (nhanh hơn với repo lớn)
createrepo_c --workers=8 /srv/mirror/rocky/9/BaseOS/x86_64/os/

# Với package groups (comps.xml)
createrepo_c \
  --groupfile=/srv/mirror/rocky/9/BaseOS/x86_64/os/repodata/comps.xml \
  /srv/mirror/rocky/9/BaseOS/x86_64/os/
```

---

## Production sync script

```bash
#!/bin/bash
# /usr/local/bin/rocky-mirror-sync.sh
set -euo pipefail

MIRROR_BASE=/srv/mirror/rocky
LOG=/var/log/rocky-mirror-sync.log
ARCH=x86_64
VER=9

REPOS=(
  "rocky9-baseos-upstream:${MIRROR_BASE}/${VER}/BaseOS/${ARCH}/os"
  "rocky9-appstream-upstream:${MIRROR_BASE}/${VER}/AppStream/${ARCH}/os"
  "rocky9-extras-upstream:${MIRROR_BASE}/${VER}/extras/${ARCH}/os"
)

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a $LOG; }

log "=== Rocky mirror sync started ==="

for entry in "${REPOS[@]}"; do
    repoid="${entry%%:*}"
    destdir="${entry##*:}"

    log "Syncing: $repoid → $destdir"
    mkdir -p "$destdir"

    reposync \
        --repoid="$repoid" \
        --download-path="$destdir" \
        --download-metadata \
        --delete \
        --newest-only \
        --gpgcheck \
        --arch="$ARCH" >> $LOG 2>&1

    log "Rebuilding metadata: $destdir"
    createrepo_c --update --workers=4 "$destdir" >> $LOG 2>&1
done

log "=== Sync completed ==="
```

```bash
# Cron: 2:00 AM hàng ngày
0 2 * * * root /usr/local/bin/rocky-mirror-sync.sh
```

---

## Nginx config (giống APT mirror)

```nginx
# /etc/nginx/conf.d/rpm-mirror.conf
server {
    listen 80;
    server_name mirror.internal.com;

    root /srv/mirror;
    autoindex on;
    autoindex_exact_size off;

    # Cache headers cho .rpm (immutable content)
    location ~* \.rpm$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Metadata — cache ngắn hơn
    location ~* repomd\.xml$ {
        expires 5m;
        add_header Cache-Control "public, must-revalidate";
    }

    location ~* \.(xml|xml\.gz|xml\.zck|sqlite\.bz2)$ {
        expires 1h;
        access_log off;
    }

    access_log /var/log/nginx/rpm-mirror-access.log;
    error_log  /var/log/nginx/rpm-mirror-error.log;
}
```

---

## Client config — trỏ về internal mirror

```bash
# Disable repo mặc định
dnf config-manager --disable baseos appstream extras

# Tạo file config nội bộ
cat > /etc/yum.repos.d/internal-mirror.repo << 'EOF'
[baseos]
name=Rocky Linux 9 - BaseOS (Internal Mirror)
baseurl=http://mirror.internal/rocky/9/BaseOS/x86_64/os/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9

[appstream]
name=Rocky Linux 9 - AppStream (Internal Mirror)
baseurl=http://mirror.internal/rocky/9/AppStream/x86_64/os/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9

[extras]
name=Rocky Linux 9 - Extras (Internal Mirror)
baseurl=http://mirror.internal/rocky/9/extras/x86_64/os/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9
EOF

# Import GPG key (nếu chưa có)
rpm --import https://download.rockylinux.org/pub/rocky/RPM-GPG-KEY-Rocky-9
# Hoặc từ mirror nội bộ
rpm --import http://mirror.internal/RPM-GPG-KEY-Rocky-9

# Test
dnf clean all
dnf makecache
dnf install nginx
```

---

## EPEL Mirror (Extra Packages for Enterprise Linux)

EPEL là repo community của Fedora, cung cấp package không có trong RHEL/Rocky chính thức.

```bash
# /etc/yum.repos.d/epel-upstream.repo
[epel9-upstream]
name=EPEL 9 (upstream)
baseurl=https://dl.fedoraproject.org/pub/epel/9/Everything/x86_64/
enabled=0
gpgcheck=1
gpgkey=https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-9

# Sync
reposync \
  --repoid=epel9-upstream \
  --download-path=/srv/mirror/epel/9/Everything/x86_64/ \
  --download-metadata \
  --delete \
  --newest-only

createrepo_c --update /srv/mirror/epel/9/Everything/x86_64/
```

---

## So sánh mirror strategy: APT vs RPM

| | APT (aptly) | RPM (reposync) |
|---|---|---|
| Snapshot | Native (aptly snapshot) | Manual (cp -r hoặc LVM snapshot) |
| Rollback | `aptly publish switch` | Đổi nginx root về snapshot cũ |
| Filter packages | Có (`-filter` flag) | Không native — filter thủ công |
| Disk dedup | Có (content-addressed store) | Không — mỗi sync là full copy |
| Metadata sign | `aptly publish` tự sign | Phải chạy `gpg` thủ công |
| Complexity | Cao hơn | Thấp hơn |

**Snapshot strategy cho RPM nếu cần:**

```bash
# Cách đơn giản: copy thư mục (dùng hardlink để tiết kiệm disk)
DATE=$(date +%Y%m%d)
cp -al /srv/mirror/rocky/ /srv/snapshots/rocky-${DATE}/

# Rollback: đổi nginx symlink
ln -sfn /srv/snapshots/rocky-20260101 /srv/mirror/rocky-current
# nginx root trỏ vào /srv/mirror/rocky-current
```

---

## Disk usage estimate (RPM)

| Distro | Components | Dung lượng xấp xỉ |
|--------|------------|-------------------|
| Rocky 9 BaseOS | x86_64, newest-only | ~4 GB |
| Rocky 9 AppStream | x86_64, newest-only | ~8 GB |
| Rocky 9 BaseOS + AppStream + Extras | x86_64 | ~15 GB |
| EPEL 9 | x86_64, newest-only | ~20 GB |
| Rocky 9 full (all history) | x86_64 | ~80 GB |

> RPM mirror **nhỏ hơn đáng kể** so với APT — Ubuntu main+universe ~200 GB vs Rocky 9 full ~15 GB với `--newest-only`.

---

## Troubleshooting

```bash
# reposync báo GPG error
# → Import GPG key
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9

# dnf báo metadata expired
dnf clean metadata
dnf makecache

# createrepo_c fail do lock file
rm -f /srv/mirror/rocky/.repodata/.lock

# dnf install báo "No such file or directory" cho .rpm
# → createrepo_c chưa chạy hoặc metadata cũ
createrepo_c --update /srv/mirror/rocky/9/BaseOS/x86_64/os/
dnf clean metadata && dnf makecache

# Check repo đang được dùng
dnf repolist enabled -v

# Test download từ mirror nội bộ
dnf install nginx --downloadonly --downloaddir=/tmp/test
```
