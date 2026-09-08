---
title: Docker Security
tags:
  - docker
  - security
  - deep-dive
date: 2026-04-26
---

# Docker Security

## Threat Model

```
Attack surfaces của Docker:

1. Image         ← vulnerable packages, malicious layers, secrets baked in
2. Dockerfile    ← root user, overprivileged, secrets in ENV/ARG
3. Registry      ← man-in-the-middle, tampered images
4. Runtime       ← privileged containers, exposed socket, escape to host
5. Network       ← container-to-container lateral movement
6. Host kernel   ← kernel exploits từ container (shared kernel)
```

---

## Non-root User

Container process mặc định chạy với UID 0 (root trong container). Nếu container escape xảy ra → root trên host.

```dockerfile
# Cách 1: dùng existing system user
FROM node:20-alpine
WORKDIR /app
COPY --chown=node:node . .
RUN npm ci --only=production
USER node                         # UID 1000 trong node image
CMD ["node", "server.js"]

# Cách 2: tạo user mới
FROM python:3.12-slim
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
WORKDIR /app
COPY --chown=appuser:appgroup . .
RUN pip install -r requirements.txt
USER appuser
CMD ["python", "app.py"]

# Cách 3: numeric UID (không cần user exist trong image)
USER 1000:1000
```

```bash
# Verify tại runtime
docker inspect --format='{{.Config.User}}' myapp
docker run --rm myapp id          # xem UID đang chạy

# Override nếu cần (debug only)
docker run --user root myapp sh
```

---

## Capabilities

Linux capabilities chia nhỏ quyền root thành các permission riêng lẻ. Container mặc định có ~14 capabilities.

**Default capabilities Docker cho container:**
```
CHOWN, DAC_OVERRIDE, FSETID, FOWNER, MKNOD, NET_RAW, SETGID,
SETUID, SETFCAP, SETPCAP, NET_BIND_SERVICE, SYS_CHROOT,
KILL, AUDIT_WRITE
```

```bash
# Production pattern: drop ALL, add lại chỉ những gì cần
docker run \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \    # bind port < 1024
  --cap-add CHOWN \               # chown files
  nginx:alpine

# Xem capabilities của process trong container
docker exec myapp capsh --print
```

**Common capabilities cần biết:**

| Capability | Cho phép |
|---|---|
| `NET_BIND_SERVICE` | Bind port < 1024 |
| `NET_ADMIN` | Network config (iptables, interfaces) |
| `NET_RAW` | Raw sockets (ping) |
| `SYS_ADMIN` | Mount, sysctl, nhiều syscalls privileged |
| `SYS_PTRACE` | ptrace (debugging) |
| `CHOWN` | chown bất kỳ file |
| `SETUID` / `SETGID` | Thay đổi UID/GID |
| `DAC_OVERRIDE` | Bypass file permission checks |

```dockerfile
# Trong Dockerfile — drop capabilities tại image level (không phụ thuộc runtime)
# Dùng seccomp thay thế (xem phần dưới)
```

---

## Read-only Filesystem

```bash
# Root filesystem read-only
docker run \
  --read-only \
  --tmpfs /tmp:rw,size=50m \       # app cần write /tmp
  --tmpfs /var/cache/nginx:rw \    # nginx cần write cache
  --tmpfs /var/run:rw \            # pid files
  nginx:alpine

# Mount specific volumes writable
docker run \
  --read-only \
  -v /var/log/myapp:/var/log/myapp \   # log dir writable
  myapp:1.0.0
```

```yaml
# docker-compose.yml
services:
  app:
    read_only: true
    tmpfs:
      - /tmp:size=50m
      - /var/run:size=1m
```

---

## no-new-privileges

Ngăn container process escalate privileges (dùng setuid binaries, sudo).

```bash
docker run --security-opt no-new-privileges myapp

# Verify
docker inspect --format='{{.HostConfig.SecurityOpt}}' myapp
```

```yaml
security_opt:
  - no-new-privileges:true
```

---

## Seccomp Profile

Seccomp (Secure Computing Mode) lọc syscalls container được phép gọi.

Docker có default seccomp profile block ~44 syscalls nguy hiểm (kexec_load, mount, reboot, ...).

```bash
# Xem default profile
curl -s https://raw.githubusercontent.com/moby/moby/master/profiles/seccomp/default.json | jq .

# Dùng default (tự động nếu kernel support)
docker run --security-opt seccomp=default myapp

# Disable seccomp (không dùng production)
docker run --security-opt seccomp=unconfined myapp

# Custom profile
docker run --security-opt seccomp=/path/to/seccomp.json myapp
```

**Custom seccomp profile (ví dụ: chỉ cho phép syscalls cần thiết):**
```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": [
        "read", "write", "open", "close", "stat",
        "fstat", "lstat", "poll", "lseek", "mmap",
        "mprotect", "munmap", "brk", "rt_sigaction",
        "rt_sigprocmask", "ioctl", "pread64", "pwrite64",
        "readv", "writev", "access", "pipe", "select",
        "sched_yield", "mremap", "msync", "mincore",
        "madvise", "dup", "dup2", "nanosleep",
        "getitimer", "alarm", "setitimer", "getpid",
        "sendfile", "socket", "connect", "accept",
        "sendto", "recvfrom", "sendmsg", "recvmsg",
        "shutdown", "bind", "listen", "getsockname",
        "getpeername", "socketpair", "setsockopt",
        "getsockopt", "clone", "fork", "vfork", "execve",
        "exit", "wait4", "kill", "uname", "fcntl",
        "flock", "fsync", "fdatasync", "truncate",
        "ftruncate", "getdents", "getcwd", "chdir",
        "fchdir", "rename", "mkdir", "rmdir", "creat",
        "link", "unlink", "symlink", "readlink",
        "chmod", "fchmod", "chown", "fchown", "lchown",
        "umask", "gettimeofday", "getrlimit", "getrusage",
        "sysinfo", "times", "getuid", "syslog", "getgid",
        "setuid", "setgid", "geteuid", "getegid",
        "getppid", "getpgrp", "setsid", "getgroups",
        "sigaltstack", "mknod", "statfs", "fstatfs",
        "getpriority", "setpriority", "prctl",
        "arch_prctl", "setrlimit", "sync", "gettid",
        "futex", "sched_setaffinity", "sched_getaffinity",
        "set_thread_area", "get_thread_area",
        "exit_group", "set_tid_address",
        "clock_gettime", "clock_getres", "clock_nanosleep",
        "tgkill", "mbind", "get_mempolicy", "set_mempolicy",
        "waitid", "openat", "mkdirat", "mknodat",
        "fchownat", "unlinkat", "renameat", "linkat",
        "symlinkat", "readlinkat", "fchmodat", "faccessat",
        "pselect6", "ppoll", "splice", "tee",
        "epoll_create", "epoll_ctl", "epoll_wait",
        "epoll_pwait", "sendmmsg", "recvmmsg",
        "accept4", "getrandom", "memfd_create",
        "statx", "epoll_create1", "pipe2",
        "dup3", "inotify_init1", "inotify_add_watch",
        "inotify_rm_watch", "eventfd2",
        "signalfd4", "timerfd_create",
        "timerfd_settime", "timerfd_gettime",
        "rt_sigreturn", "set_robust_list", "get_robust_list",
        "prlimit64", "seccomp", "sched_getparam",
        "sched_getscheduler", "sched_setscheduler",
        "socket"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

---

## AppArmor Profile

AppArmor giới hạn file access, network access, capability của process.

```bash
# Load custom profile
apparmor_parser -r -W /etc/apparmor.d/docker-nginx

# Dùng profile
docker run \
  --security-opt apparmor=docker-nginx \
  nginx:alpine

# Dùng default docker profile
docker run --security-opt apparmor=docker-default myapp

# Disable (không dùng production)
docker run --security-opt apparmor=unconfined myapp
```

**Ví dụ AppArmor profile đơn giản:**
```
#include <tunables/global>

profile docker-nginx flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  network inet tcp,
  network inet udp,
  network inet icmp,

  deny network raw,

  /etc/nginx/** r,
  /var/log/nginx/** w,
  /var/cache/nginx/** rw,
  /run/nginx.pid rw,

  deny /proc/sys/kernel/** w,
  deny /sys/** w,
}
```

---

## Docker Socket Security

`/var/run/docker.sock` = root access trên host. Container mount socket → có thể escape ra host.

```bash
# NGUY HIỂM — container có thể làm bất cứ điều gì trên host
docker run -v /var/run/docker.sock:/var/run/docker.sock myapp

# Attack example từ container:
docker run --rm -v /:/hostfs --privileged busybox chroot /hostfs
```

**Alternatives:**
1. **Docker Socket Proxy** (Tecnativa): expose subset của Docker API
2. **Rootless Docker**: daemon chạy với user UID
3. **Podman**: daemonless, không có socket vấn đề

```yaml
# Socket proxy — chỉ expose endpoints cần thiết
services:
  dockerproxy:
    image: tecnativa/docker-socket-proxy
    environment:
      CONTAINERS: 1    # allow read containers
      SERVICES: 1      # allow read services
      TASKS: 1
      NETWORKS: 1
      VOLUMES: 0       # deny volumes API
      POST: 0          # deny all POST (no create/delete)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - proxy_net

  traefik:
    image: traefik:v3
    environment:
      - DOCKER_HOST=tcp://dockerproxy:2375   # dùng proxy thay vì socket
    networks:
      - proxy_net
```

---

## Rootless Docker

Chạy Docker daemon với non-root user — container escape không có root trên host.

```bash
# Cài rootless mode
dockerd-rootless-setuptool.sh install

# Chạy rootless daemon
systemctl --user start docker

# Dùng
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
docker run hello-world

# Rootless limitations:
# - overlay network không dùng được (cần VXLAN)
# - port mapping < 1024 cần thêm config
# - Một số mount options không available
```

---

## Image Scanning

### Trivy

```bash
# Scan image (vulnerabilities)
trivy image nginx:1.26
trivy image --severity HIGH,CRITICAL nginx:1.26

# Scan với exit code cho CI
trivy image --exit-code 1 --severity CRITICAL myapp:1.2.3

# Scan local tarball
docker save myapp:latest | trivy image --input -

# Scan Dockerfile (misconfig)
trivy config ./Dockerfile

# Scan IaC (compose files, k8s manifests)
trivy config ./compose.yml

# Scan với ignore file
cat .trivyignore
# CVE-2023-12345   # false positive, patched in our config
trivy image --ignorefile .trivyignore myapp:latest

# Output formats
trivy image --format json --output report.json myapp:latest
trivy image --format sarif --output trivy.sarif myapp:latest  # GitHub Security tab
```

### Grype

```bash
grype myapp:1.2.3
grype myapp:1.2.3 --fail-on critical
grype myapp:1.2.3 -o json > grype-report.json
```

### Integrate vào CI/CD

```yaml
# .github/workflows/security.yml
- name: Build image
  run: docker build -t myapp:${{ github.sha }} .

- name: Run Trivy scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: sarif
    output: trivy-results.sarif
    severity: HIGH,CRITICAL
    exit-code: 1

- name: Upload Trivy results to GitHub Security
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: trivy-results.sarif
```

---

## Image Signing — Cosign

Ký image để đảm bảo chỉ deploy image đã được verify. Xem thêm [[GitOps Security]].

```bash
# Install cosign
brew install cosign

# Generate key pair
cosign generate-key-pair
# → cosign.key (private), cosign.pub (public)

# Sign image sau khi push
cosign sign --key cosign.key registry.internal/myapp:1.2.3

# Verify
cosign verify --key cosign.pub registry.internal/myapp:1.2.3

# Keyless signing (Sigstore — dùng OIDC identity)
cosign sign registry.internal/myapp:1.2.3
# Mở browser → verify GitHub/Google identity → sign với ephemeral key
```

---

## Content Trust (DCT)

Docker Content Trust dùng Notary để sign và verify images.

```bash
# Enable DCT (tất cả pull/push phải signed)
export DOCKER_CONTENT_TRUST=1

# Sign khi push
docker push registry.internal/myapp:1.2.3
# → prompt nhập passphrase để ký

# Verify khi pull
docker pull registry.internal/myapp:1.2.3
# → verify signature, fail nếu không có/invalid
```

---

## Security Checklist

```
Image build:
  [ ] Non-root USER trong Dockerfile
  [ ] Multi-stage build — loại bỏ build tools khỏi runtime image
  [ ] .dockerignore — không leak secrets, .git, node_modules
  [ ] Không dùng ENV/ARG cho secrets
  [ ] Pin base image với digest: FROM ubuntu@sha256:xxx
  [ ] Trivy scan trong CI — block on CRITICAL

Runtime:
  [ ] --cap-drop ALL --cap-add <chỉ cần thiết>
  [ ] --read-only + --tmpfs cho /tmp
  [ ] --security-opt no-new-privileges
  [ ] Resource limits (--memory, --cpus)
  [ ] Không --privileged
  [ ] Không mount /var/run/docker.sock (hoặc dùng socket proxy)
  [ ] Network: internal networks cho backend services

Registry:
  [ ] Image signing (Cosign/Notary)
  [ ] Scan on push (Harbor Trivy integration)
  [ ] Retention policy — xoá old tags
  [ ] RBAC — không dùng admin account cho CI/CD

Host:
  [ ] Docker daemon không expose TCP (hoặc dùng TLS client certs)
  [ ] Rootless Docker nếu có thể
  [ ] Audit Docker socket access
  [ ] Keep Docker Engine updated
```

---

## Gotchas

- **`--privileged` = root trên host**: container có `--privileged` có thể mount host filesystem, load kernel modules, thay đổi iptables. Không bao giờ dùng trong production.
- **UID 0 trong container ≈ root trên host (legacy)**: với user namespace remapping OFF (default), UID 0 trong container map sang UID 0 trên host → nếu escape → root. Enable user namespace (`userns-remap`) để map sang non-root UID trên host.
- **`SYS_ADMIN` = basically root**: capability này bao gồm hàng chục quyền nguy hiểm (mount, sysctl, setns, ...). Nếu app "cần" `SYS_ADMIN`, refactor app thay vì cấp.
- **Env vars và secrets**: `docker inspect` thấy tất cả env vars. `docker history` thấy `ENV` instructions. Secrets trong env vars → không an toàn cho compliance. Dùng file-based secrets (`/run/secrets/`).
- **Base image tags thay đổi**: `FROM ubuntu:22.04` hôm nay khác ngày mai (patches, updates). Nếu cần reproducible: pin digest `FROM ubuntu:22.04@sha256:abc123`. Trade-off: không nhận security patches tự động.
- **`--network host` bypass network isolation**: container thấy toàn bộ host network, có thể sniff traffic của containers khác trên default bridge, connect tới host services trên localhost.
