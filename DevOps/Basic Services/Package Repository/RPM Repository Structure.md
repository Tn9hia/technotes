---
title: RPM Repository Structure
tags:
  - rpm
  - dnf
  - yum
  - rhel
  - deep-dive
date: 2026-04-26
---

# RPM Repository Structure

## Debian vs Red Hat — so sánh nhanh

| | Debian/Ubuntu | Red Hat/CentOS/Rocky |
|---|---|---|
| Package format | `.deb` | `.rpm` |
| Package manager | `apt` / `dpkg` | `dnf` / `yum` / `rpm` |
| Repo metadata | `dists/` + `Packages.gz` | `repodata/` + `repomd.xml` |
| Metadata tool | `apt-ftparchive` | `createrepo_c` |
| Mirror tool | `apt-mirror`, `aptly` | `reposync`, `dnf reposync` |
| Repo config | `/etc/apt/sources.list.d/` | `/etc/yum.repos.d/*.repo` |
| Signing | GPG clearsign InRelease | GPG detached `.asc` / RPM header |
| Package index | `Packages.xz` (flat text) | `primary.xml.gz` (XML) |

---

## Cấu trúc thư mục RPM repo

```
/srv/mirror/rocky/9/
├── BaseOS/
│   └── x86_64/
│       ├── os/
│       │   ├── repodata/                  ← metadata — dnf đọc cái này trước
│       │   │   ├── repomd.xml             ← index của metadata (entry point)
│       │   │   ├── repomd.xml.asc         ← GPG signature của repomd.xml
│       │   │   ├── primary.xml.gz         ← danh sách packages + dependencies
│       │   │   ├── filelists.xml.gz       ← danh sách files trong mỗi package
│       │   │   ├── other.xml.gz           ← changelog
│       │   │   └── <hash>-comps.xml.gz    ← package groups (minimal, server, etc.)
│       │   └── Packages/                  ← actual .rpm files
│       │       ├── b/
│       │       │   └── bash-5.1.8-6.el9.x86_64.rpm
│       │       └── n/
│       │           └── nginx-1.22.1-1.el9.ngx.x86_64.rpm
│       └── debug/                         ← debuginfo packages (riêng)
├── AppStream/
│   └── x86_64/
│       └── os/
│           ├── repodata/
│           └── Packages/
└── extras/
```

**Khác biệt quan trọng so với APT:**
- APT: `dists/` chứa metadata, `pool/` chứa packages — **tách biệt hoàn toàn**
- RPM: `repodata/` và `Packages/` nằm **cùng cấp trong một thư mục repo**
- RPM có thể có nhiều repo độc lập (BaseOS, AppStream, extras) — mỗi cái có `repodata/` riêng

---

## repomd.xml — Entry point

`repomd.xml` = tương đương `InRelease` của APT. dnf fetch cái này đầu tiên.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<repomd xmlns="http://linux.duke.edu/metadata/repo"
        xmlns:rpm="http://linux.duke.edu/metadata/rpm">
  <revision>1698765432</revision>

  <data type="primary">
    <checksum type="sha256">abc123...</checksum>
    <open-checksum type="sha256">def456...</open-checksum>
    <location href="repodata/abc123-primary.xml.gz"/>
    <timestamp>1698765432</timestamp>
    <size>1234567</size>
    <open-size>9876543</open-size>
  </data>

  <data type="filelists">
    <checksum type="sha256">...</checksum>
    <location href="repodata/xxx-filelists.xml.gz"/>
    ...
  </data>

  <data type="other">...</data>
  <data type="comps">...</data>
</repomd>
```

**Flow:** `repomd.xml` → lấy path của `primary.xml.gz` → download + verify checksum → parse package list

---

## primary.xml — Package database

Tương đương `Packages.xz` của APT, nhưng ở dạng XML.

```xml
<package type="rpm">
  <name>nginx</name>
  <arch>x86_64</arch>
  <version epoch="1" ver="1.22.1" rel="1.el9.ngx"/>
  <checksum type="sha256" pkgid="YES">deadbeef...</checksum>
  <summary>A high performance web server</summary>
  <description>...</description>
  <packager>NGINX Packaging</packager>
  <url>https://nginx.org</url>
  <time file="1698000000" build="1697000000"/>
  <size package="823456" installed="2345678" archive="2345000"/>
  <location href="Packages/n/nginx-1.22.1-1.el9.ngx.x86_64.rpm"/>
  <format>
    <rpm:provides>
      <rpm:entry name="nginx" flags="EQ" epoch="1" ver="1.22.1" rel="1.el9.ngx"/>
      <rpm:entry name="webserver"/>
    </rpm:provides>
    <rpm:requires>
      <rpm:entry name="openssl-libs"/>
      <rpm:entry name="pcre2"/>
      ...
    </rpm:requires>
    <rpm:conflicts/>
    <rpm:obsoletes/>
    <rpm:files>/usr/sbin/nginx</rpm:files>
  </format>
</package>
```

---

## RPM file naming convention

```
nginx  -  1.22.1  -  1.el9.ngx  .  x86_64  .rpm
  │          │            │            │
  │          │            │            └── Architecture: x86_64, aarch64, noarch
  │          │            └── Release: build revision + distro tag
  │          └── Version: upstream version
  └── Name

# Distro tags:
# el9   = RHEL/CentOS/Rocky 9
# el8   = RHEL/CentOS/Rocky 8
# fc39  = Fedora 39
# amzn2 = Amazon Linux 2
# ngx   = built by NGINX team (third-party)

# noarch = platform-independent (scripts, configs, docs)
# src    = source RPM
```

**Epoch:** tiebreaker khi version string không đủ để so sánh. `1:1.22.1 > 2.0.0` vì epoch 1 > epoch 0.

---

## GPG Signing trong RPM ecosystem

Khác với APT sign metadata, RPM **sign từng package file**:

```
GPG Key
  └── ký vào RPM header của từng .rpm file
        └── dnf verify khi install

+ repomd.xml.asc  ← sign metadata riêng
```

```bash
# Import GPG key của Rocky Linux
rpm --import https://download.rockylinux.org/pub/rocky/RPM-GPG-KEY-Rocky-9

# Verify key đã import
rpm -q gpg-pubkey --qf '%{name}-%{version}-%{release} --> %{summary}\n'

# Verify RPM package thủ công
rpm --checksig nginx-1.22.1-1.el9.ngx.x86_64.rpm
# Output: nginx-...: digests signatures OK

# Verify package đã install
rpm -V nginx
# Không có output = OK
# S = size changed, M = mode changed, 5 = MD5/hash fail
```

**Khi làm mirror nội bộ:**
```bash
# Sign repo metadata bằng key riêng
gpg --detach-sign --armor repodata/repomd.xml
# → tạo repomd.xml.asc

# Hoặc sign khi createrepo
createrepo_c --repo-gpg-sign /path/to/repo

# Client import key
rpm --import http://mirror.internal/RPM-GPG-KEY-internal
```

---

## `.repo` file — Client config

```ini
# /etc/yum.repos.d/rocky.repo
[baseos]
name=Rocky Linux $releasever - BaseOS
baseurl=http://mirror.internal/rocky/$releasever/BaseOS/$basearch/os/
#mirrorlist=https://mirrors.rockylinux.org/mirrorlist?arch=$basearch&repo=BaseOS-$releasever
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9
countme=1

[appstream]
name=Rocky Linux $releasever - AppStream
baseurl=http://mirror.internal/rocky/$releasever/AppStream/$basearch/os/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9

[extras]
name=Rocky Linux $releasever - Extras
baseurl=http://mirror.internal/rocky/$releasever/extras/$basearch/os/
enabled=0   ← disabled by default
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9
```

**Biến tự động:**
- `$releasever` → `9` (từ `/etc/os-release`)
- `$basearch` → `x86_64` (từ `uname -m`)
- `$arch` → full arch string

**Tắt/bật repo:**
```bash
dnf config-manager --disable extras
dnf config-manager --enable extras
dnf install nginx --enablerepo=extras
```

---

## DNF cache & local paths

```bash
# Cache metadata
ls /var/cache/dnf/

# Downloaded RPMs (trước khi install)
ls /var/cache/dnf/*/packages/

# Xoá cache
dnf clean all          # xoá tất cả cache
dnf clean metadata     # chỉ xoá metadata
dnf clean packages     # chỉ xoá .rpm đã download

# Package database
ls /var/lib/rpm/        # RPM database (Berkeley DB hoặc SQLite)
rpm --rebuilddb         # rebuild nếu corrupt
```

---

## DNF debug commands

```bash
# Xem package đến từ repo nào
dnf info nginx
# Repository  : appstream

# Xem tất cả version available
dnf --showduplicates list nginx

# Check update khả dụng
dnf check-update

# Xem dependency
dnf deplist nginx
dnf repoquery --requires nginx

# Search package chứa file cụ thể
dnf provides /usr/sbin/nginx
# → nginx-1.22.1-1.el9.ngx.x86_64

# Verbose để debug repo issue
dnf install nginx -v 2>&1 | grep -E "repo|mirror|baseurl"

# List repos đang active
dnf repolist
dnf repolist all    # kể cả disabled

# Verify integrity package đã install
rpm -V nginx
```

---

## Tạo repo nội bộ (custom packages)

```bash
# Cài createrepo_c
dnf install createrepo_c

# Chuẩn bị thư mục
mkdir -p /srv/custom-repo
cp my-app-1.0.0-1.el9.x86_64.rpm /srv/custom-repo/

# Tạo metadata
createrepo_c /srv/custom-repo/
# → tạo /srv/custom-repo/repodata/

# Update sau khi thêm package mới
createrepo_c --update /srv/custom-repo/

# Sign metadata (optional nhưng nên làm)
gpg --detach-sign --armor /srv/custom-repo/repodata/repomd.xml

# Client config
cat > /etc/yum.repos.d/custom.repo << 'EOF'
[custom]
name=Custom Internal Repo
baseurl=http://mirror.internal/custom-repo/
enabled=1
gpgcheck=0   # hoặc gpgcheck=1 + gpgkey=...
EOF

dnf install my-app
```
