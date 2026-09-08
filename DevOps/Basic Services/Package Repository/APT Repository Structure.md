---
title: APT Repository Structure
tags:
  - apt
  - debian
  - mirror
  - deep-dive
date: 2026-04-26
---

# APT Repository Structure

## Tổng quan cấu trúc thư mục

```
/repo/ubuntu/
├── dists/                         ← metadata — apt đọc cái này trước
│   └── jammy/                     ← codename (jammy = 22.04, focal = 20.04)
│       ├── InRelease              ← metadata chính, GPG-signed inline
│       ├── Release                ← metadata không ký (legacy)
│       ├── Release.gpg            ← detached GPG signature của Release
│       ├── main/
│       │   ├── binary-amd64/
│       │   │   ├── Packages       ← index plain text
│       │   │   ├── Packages.gz    ← compressed
│       │   │   └── Packages.xz    ← compressed (apt ưu tiên xz)
│       │   ├── binary-arm64/
│       │   └── source/
│       │       └── Sources.gz
│       ├── restricted/
│       ├── universe/
│       └── multiverse/
└── pool/                          ← actual .deb files
    ├── main/
    │   ├── a/apt/
    │   │   └── apt_2.4.8_amd64.deb
    │   └── n/nginx/
    │       ├── nginx_1.18.0-6_amd64.deb
    │       └── nginx-common_1.18.0-6_all.deb
    ├── restricted/
    ├── universe/
    └── multiverse/
```

**Rule đặt file trong pool:** `pool/<component>/<first-letter>/<package-name>/`
- `nginx` → `pool/main/n/nginx/`
- `libc6` → `pool/main/l/libc6/`
- `lib*` package → thường dùng ký tự thứ 4: `libssl` → `pool/main/libs/libssl/`

---

## InRelease — File quan trọng nhất

`InRelease` = `Release` + inline GPG signature (clearsigned). Đây là file apt fetch đầu tiên.

```
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA512

Origin: Ubuntu
Label: Ubuntu
Suite: jammy
Version: 22.04
Codename: jammy
Date: Thu, 21 Apr 2022 17:16:08 UTC
Acquire-By-Hash: yes
Architectures: amd64 arm64 armhf ...
Components: main restricted universe multiverse
Description: Ubuntu Jammy 22.04
MD5Sum:
 abc123...  12345  main/binary-amd64/Packages
 ...
SHA256:
 deadbeef...  12345  main/binary-amd64/Packages
 ...
-----BEGIN PGP SIGNATURE-----
...
-----END PGP SIGNATURE-----
```

**Các field quan trọng:**
- `Suite` / `Codename`: apt dùng để match với sources.list
- `Components`: `main restricted universe multiverse`
- `SHA256`: hash của từng file trong `dists/jammy/` → tamper detection
- `Acquire-By-Hash`: cho phép download file theo hash thay vì tên → atomic update

---

## Packages Index

`Packages.xz` là flat-text database của tất cả packages trong component đó.

```
Package: nginx
Architecture: amd64
Version: 1.18.0-6ubuntu14.4
Priority: optional
Section: web
Maintainer: Ubuntu Developers
Installed-Size: 44
Depends: nginx-core (<< 1.18.0-6ubuntu14.4.1~) | nginx-full (<< ...
Filename: pool/main/n/nginx/nginx_1.18.0-6ubuntu14.4_amd64.deb
Size: 3732
MD5sum: abc...
SHA1: def...
SHA256: 1234abcd...
Homepage: https://nginx.net
Description: small, powerful, scalable web/proxy server
 ...

Package: nginx-common
Architecture: all
...
```

**apt lookup flow:**
1. Đọc `Packages.xz` → build in-memory index
2. User `apt install nginx` → tìm `Package: nginx` → lấy `Filename:` và `SHA256:`
3. Download `pool/main/n/nginx/nginx_1.18.0-...deb` → verify SHA256 → dpkg install

---

## GPG Signing & Trust Chain

```
Ubuntu Master Key (offline)
    │
    └── Ubuntu Archive Signing Key (871920D1991BC93C)
            │
            └── signs InRelease file
                    │
                    └── InRelease contains SHA256 of Packages files
                                │
                                └── Packages contains SHA256 of .deb files
```

Toàn bộ chain: trust vào 1 GPG key → trust vào toàn bộ content repo.

**Verify thủ công:**
```bash
# Download InRelease
curl -O http://archive.ubuntu.com/ubuntu/dists/jammy/InRelease

# Verify signature
gpg --verify InRelease

# Extract Release content
gpg --output Release --decrypt InRelease

# Verify hash của Packages file
sha256sum main/binary-amd64/Packages.xz
# So sánh với SHA256 field trong Release
```

**Khi tự làm mirror và cần sign:**
```bash
# Tạo GPG key riêng cho mirror
gpg --gen-key
# Key ID ví dụ: ABCDEF1234567890

# aptly tự sign khi publish
aptly publish snapshot ... \
  -gpg-key="ABCDEF1234567890" \
  -gpg-provider="gpg2"

# Client cần import public key
curl http://mirror.internal/gpg.key | gpg --dearmor > \
  /etc/apt/trusted.gpg.d/internal-mirror.gpg
```

---

## Sources.list syntax

```
deb [options] URI suite component1 component2 ...

# Ví dụ đầy đủ:
deb [arch=amd64 signed-by=/etc/apt/trusted.gpg.d/ubuntu.gpg] \
  http://archive.ubuntu.com/ubuntu \
  jammy \
  main restricted universe

# Các option trong []:
# arch=amd64,arm64       chỉ fetch cho architecture này
# signed-by=<path>       dùng key file cụ thể thay vì trusted.gpg.d/
# trusted=yes            skip GPG verification (NGUY HIỂM)
```

**New format (`.sources` file — DEB822 format):**
```
# /etc/apt/sources.list.d/ubuntu.sources
Types: deb
URIs: http://mirror.internal/ubuntu
Suites: jammy jammy-updates jammy-security
Components: main restricted universe
Architectures: amd64
Signed-By: /etc/apt/trusted.gpg.d/internal-mirror.gpg
```

---

## Apt cache & local paths

```bash
# Apt lưu downloaded .deb ở đây
ls /var/cache/apt/archives/

# Package lists (Packages.xz đã giải nén)
ls /var/lib/apt/lists/

# Xoá cache
apt-get clean        # xoá .deb trong archives/
apt-get autoclean    # xoá .deb của package đã uninstall
```

---

## Tạo repo đơn giản không cần aptly (nhỏ, custom package)

```bash
# Chuẩn bị thư mục
mkdir -p /srv/custom-repo/pool/main
cp my-app_1.0.0_amd64.deb /srv/custom-repo/pool/main/

# Tạo Packages index
cd /srv/custom-repo
dpkg-scanpackages pool/main /dev/null | gzip -9c > dists/stable/main/binary-amd64/Packages.gz

# Tạo Release file
apt-ftparchive release dists/stable > dists/stable/Release

# Sign
gpg --clearsign -o dists/stable/InRelease dists/stable/Release
gpg -abs -o dists/stable/Release.gpg dists/stable/Release

# Serve bằng nginx
# Client thêm sources.list:
# deb [signed-by=...] http://custom-repo.internal/ stable main
```

---

## Các lệnh apt debug hữu ích

```bash
# Xem full URL apt sẽ fetch
apt-get install -s nginx | grep "^Inst"

# Verbose apt update — thấy từng URL được fetch
apt-get update -o Debug::Acquire::http=true 2>&1 | grep "GET\|200\|404"

# Check package đến từ repo nào
apt-cache policy nginx
# Output:
# nginx:
#   Installed: 1.18.0-6ubuntu14
#   Candidate: 1.18.0-6ubuntu14
#   Version table:
#  *** 1.18.0-6ubuntu14 500
#         500 http://mirror.internal/ubuntu jammy/main amd64 Packages

# Verify integrity của installed package
debsums nginx
```
