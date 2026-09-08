---
title: Docker — Overview
tags:
  - docker
  - container
  - overview
date: 2026-04-26
---

# Docker — Overview

## 1. What is Docker?

Docker là platform container hóa — package ứng dụng + dependencies thành **image**, chạy như **container** isolated trên bất kỳ host nào có Docker Engine.

```
[App Code + Libs + Runtime + Config]
           │
      Docker Image  (read-only layers)
           │
    Docker Container  ──  Isolated process group
           │              (namespace + cgroup)
    Host OS Kernel  (shared — không có hypervisor)
```

**Container vs VM:**

| | Container | VM |
|---|---|---|
| Isolation | OS-level (namespace) | Hardware-level (hypervisor) |
| Boot time | ~100ms | ~30s |
| Overhead | Minimal (shared kernel) | Full guest OS |
| Density | 100s per host | 10–20 per host |
| Portability | Image là artifact duy nhất | Cần convert disk image |

---

## 2. Why Docker?

- **Consistency**: "Works on my machine" → image mang đủ dependencies
- **Speed**: Build → Ship → Run nhanh hơn VM nhiều lần
- **DevOps pipeline**: Image là standard artifact từ dev → staging → prod
- **Microservices**: Mỗi service isolated, scale/deploy/rollback độc lập
- **CI/CD**: Ephemeral build environment reproducible 100%
- **Air-gap deployment**: Ship image thay vì quản lý packages

---

## 3. When to Use

**Phù hợp:**
- Package ứng dụng + dependencies để deploy nhất quán
- Microservices — mỗi service là 1 container
- CI/CD build environment cần reproducible
- Local development: chạy DB/Redis/Kafka mà không cài trực tiếp
- Air-gap: toàn bộ stack trong registry nội bộ

**Không phù hợp / cân nhắc:**
- Ứng dụng cần kernel khác host → vẫn cần VM
- Hard real-time (cgroup latency overhead)
- Stateful legacy DB đã stable — thêm complexity, ít lợi
- Team chưa có Docker expertise — ops overhead ban đầu cao

---

## 4. Architecture

```
┌─────────────────────────────────────────────────────┐
│              Docker Client (CLI)                     │
│   docker build / run / pull / push / ps / logs ...  │
└──────────────────────┬──────────────────────────────┘
                       │ REST API
              /var/run/docker.sock
                       │
┌──────────────────────▼──────────────────────────────┐
│              dockerd  (Docker Daemon)                │
│  Image mgmt · Network mgmt · Volume mgmt · API      │
│                                                      │
│  ┌───────────────────────────────────────────────┐  │
│  │               containerd                       │  │
│  │     Container lifecycle supervisor             │  │
│  │                                               │  │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐   │  │
│  │   │  runc    │  │  runc    │  │  runc    │   │  │
│  │   │  (ctr 1) │  │  (ctr 2) │  │  (ctr 3) │   │  │
│  │   └──────────┘  └──────────┘  └──────────┘   │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
              │                     │
       OverlayFS                 Linux Networking
      (image layers)          (bridge/iptables/veth)
```

**Components:**
- **dockerd**: API server, quản lý images/networks/volumes
- **containerd**: Supervisor quản lý vòng đời container (pull image, start/stop)
- **runc**: OCI runtime — tạo container thật sự qua namespace + cgroup syscalls
- **Docker socket** (`/var/run/docker.sock`): Unix socket, ai access được socket = root trên host

**Isolation mechanisms:**
- **Namespaces**: PID (process isolation), NET (network stack riêng), MNT (filesystem view), UTS (hostname), IPC, USER
- **cgroups v2**: Giới hạn CPU/memory/I/O/network bandwidth per container
- **OverlayFS**: Union filesystem — stack read-only image layers + writable container layer

---

## 5. Container Lifecycle

```
              docker create
                   │
              [Created]  ← image pulled, container layer created
                   │ docker start
              [Running] ──── docker pause ──── [Paused]
                   │         (SIGSTOP)          │
                   │                      docker unpause
                   │ docker stop                │
                   │  (SIGTERM → wait → SIGKILL)◄┘
                   │ docker kill (SIGKILL)
              [Exited]  ← exit code stored
                   │
                   │ docker rm
              [Deleted]
```

```bash
# Run container (create + start + attach)
docker run -d --name web -p 80:80 --restart unless-stopped nginx:alpine

# Lifecycle
docker stop web          # SIGTERM, 10s timeout sau đó SIGKILL
docker start web
docker restart web
docker kill web          # SIGKILL ngay lập tức
docker rm web            # xoá container đã stopped
docker rm -f web         # force remove dù đang running

# Status
docker ps                # running containers
docker ps -a             # tất cả containers kể cả stopped
docker inspect web       # full JSON metadata
```

---

## 6. Networking

Docker networking dùng Linux bridge, iptables rules, và veth pairs. Chi tiết: [[DevOps/Container & Orchestration/Docker/Networking]].

**Network drivers:**

| Driver | Use case |
|---|---|
| **bridge** (default) | Single-host, container-to-container |
| **host** | Max performance, port không bị NAT |
| **overlay** | Multi-host Docker Swarm |
| **macvlan** | Container cần IP trong LAN thật |
| **none** | Hoàn toàn không có network |

```bash
# Custom bridge network — có built-in DNS (container name → IP)
docker network create --driver bridge mynet
docker run -d --name db --network mynet postgres:16
docker run -d --name app --network mynet -e DB_HOST=db myapp
# "app" resolve "db" bằng hostname → không cần hardcode IP

# Port mapping
docker run -p 8080:80 nginx              # 0.0.0.0:8080 → container:80
docker run -p 127.0.0.1:8080:80 nginx   # chỉ localhost
docker run -P nginx                      # auto-assign host ports từ EXPOSE
```

---

## 7. Storage

Chi tiết: [[DevOps/Container & Orchestration/Docker/Storage]].

| Type | Persistence | Managed by | Use case |
|---|---|---|---|
| **Volume** | Yes | Docker | Production data |
| **Bind mount** | Yes | Host OS | Dev, config injection |
| **tmpfs** | No (RAM) | Kernel | Sensitive data, temp cache |

```bash
# Named volume (recommended cho production)
docker volume create pgdata
docker run -v pgdata:/var/lib/postgresql/data postgres

# Bind mount (dev — live reload)
docker run -v $(pwd)/src:/app/src node:20-alpine

# Read-only bind mount (config injection)
docker run -v /etc/myapp/config.yml:/app/config.yml:ro myapp

# tmpfs (không persist sau khi container stop)
docker run --tmpfs /tmp:rw,size=100m myapp
```

---

## 8. Image & Dockerfile

Chi tiết: [[Dockerfile & Images]].

```
Image = stack of read-only layers (OverlayFS)
Container = image layers + thin writable layer on top

Layer 1: FROM ubuntu:22.04     (base)
Layer 2: RUN apt-get install   (deps)
Layer 3: COPY . /app           (source)
Layer 4: [writable]            (container runtime writes)
```

**Dockerfile cơ bản:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./          # copy package files trước → cache layer
RUN npm ci --only=production
COPY . .
EXPOSE 3000
USER node                      # non-root
CMD ["node", "server.js"]      # exec form — nhận SIGTERM trực tiếp
```

**Multi-stage build (reduce image size):**
```dockerfile
# Stage 1: build
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /app ./cmd/server

# Stage 2: runtime (chỉ binary, không có Go toolchain)
FROM gcr.io/distroless/static
COPY --from=builder /app /app
ENTRYPOINT ["/app"]
```

---

## 9. Security

Chi tiết: [[Docker Security Overview]].

**Quick checklist:**
- [ ] Non-root user trong Dockerfile (`USER 1000`)
- [ ] Drop tất cả capabilities, add chỉ những gì cần (`--cap-drop ALL --cap-add NET_BIND_SERVICE`)
- [ ] Read-only root filesystem (`--read-only`) + tmpfs cho /tmp
- [ ] Không dùng `--privileged` trong production
- [ ] Không mount Docker socket (`/var/run/docker.sock`) vào container
- [ ] Scan image với Trivy trước deploy
- [ ] Resource limits (`--memory 512m --cpus 0.5`)
- [ ] Dùng distroless/alpine base — giảm attack surface

```bash
docker run \
  --read-only \
  --tmpfs /tmp \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --memory 512m \
  --cpus 0.5 \
  --security-opt no-new-privileges \
  myapp:1.0.0
```

---

## 10. Ops Runbook

```bash
# --- Monitoring ---
docker stats                         # live CPU/mem/net/IO tất cả containers
docker stats --no-stream             # snapshot (1 lần)
docker inspect --format='{{.State.Health.Status}}' web

# --- Logs ---
docker logs web
docker logs -f --tail 100 web        # follow, last 100 lines
docker logs --since 1h web
docker logs --since "2026-04-26T08:00:00" web

# --- Debug ---
docker exec -it web sh               # shell vào container đang running
docker exec web ps aux               # chạy command không cần interactive
docker cp web:/etc/nginx/nginx.conf ./nginx.conf  # copy file từ container
docker diff web                      # xem files đã thay đổi so với image

# --- Network debug ---
docker network inspect mynet         # xem IPs, connected containers
docker exec web ping db              # test connectivity
docker exec web nslookup db          # test DNS

# --- Image management ---
docker images
docker image prune                   # xoá dangling images
docker image prune -a                # xoá tất cả unused images
docker pull nginx:1.26               # pull specific tag (không dùng :latest trong prod)

# --- Cleanup ---
docker system df                     # disk usage breakdown
docker system prune                  # stopped containers + dangling images + unused networks
docker system prune -a --volumes     # aggressive — kể cả unused images và volumes
docker volume prune                  # unused volumes (cẩn thận — mất data)
```

**Troubleshooting:**
```bash
# Container exit ngay lập tức
docker logs <container>
docker run -it --entrypoint sh myimage   # override entrypoint để debug

# OOMKilled
docker inspect <container> | grep -i oom
# → OOMKilled: true → tăng --memory

# Permission denied trên volume
docker exec <container> id              # xem UID đang chạy
# Fix trong Dockerfile: RUN chown -R 1000:1000 /data

# Network không kết nối được
docker network inspect bridge          # xem iptables rules
# Kiểm tra: ufw / firewalld có block không
```

---

## Gotchas

- **PID 1 signal handling**: `CMD node app.js` → shell là PID 1, không forward SIGTERM sang app → graceful shutdown không hoạt động. Dùng exec form `CMD ["node", "app.js"]` hoặc thêm `tini` làm init process.
- **Build context size**: `docker build .` gửi toàn bộ current dir lên daemon. Thiếu `.dockerignore` → chậm, leak secrets. Luôn có `.dockerignore` với `node_modules/`, `.git/`, `*.env`.
- **Layer cache invalidation**: `COPY . .` trước `RUN npm install` → mỗi thay đổi source code invalidate cache npm install. Pattern đúng: copy package files → install deps → copy source.
- **Volume permissions**: volume mount tạo thư mục với UID=root nếu empty. Container chạy non-root không write được. Fix: `RUN mkdir -p /data && chown 1000:1000 /data` trong Dockerfile.
- **docker stop timeout**: default 10s SIGTERM rồi SIGKILL. App cần graceful drain lâu hơn: `docker stop -t 60 web` hoặc `stop_grace_period: 60s` trong Compose.
- **ENV trong image**: `ENV DB_PASSWORD=xxx` bake vào image, thấy qua `docker inspect`. Không dùng cho secrets — dùng `--env-file` hoặc Docker Secrets (Swarm).
- **Dangling images**: mỗi `docker build` tạo layer mới. Layer cũ thành "dangling" nếu không có tag. Cron `docker image prune` hàng tuần để dọn disk.
