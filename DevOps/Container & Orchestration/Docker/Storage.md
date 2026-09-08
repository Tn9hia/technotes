---
title: Docker Storage
tags:
  - docker
  - storage
  - deep-dive
date: 2026-04-26
---

# Docker Storage

## Storage Types

```
┌─────────────────────────────────────────────────────────┐
│                      Container                          │
│                                                         │
│  /data  ←──── Volume    (docker-managed, /var/lib/docker/volumes/)
│  /config ←─── Bind mount (host path mapped directly)    │
│  /tmp   ←──── tmpfs      (RAM, không persist)           │
│                                                         │
│  /app   ←──── Image layers (read-only, OverlayFS)       │
│  [writes] ──► Container layer (writable, OverlayFS)     │
└─────────────────────────────────────────────────────────┘
```

| Type | Persistence | Managed by | Performance | Use case |
|---|---|---|---|---|
| **Volume** | Yes | Docker | Good | Production data |
| **Bind mount** | Yes (host path) | Host OS | Good | Dev, config |
| **tmpfs** | No (RAM) | Kernel | Excellent | Secrets, cache |
| **Container layer** | No (lost on rm) | OverlayFS | Worse (CoW) | Temp writes |

---

## Volume

Named volumes được Docker quản lý tại `/var/lib/docker/volumes/<name>/_data`.

```bash
# Tạo volume
docker volume create pgdata

# Mount vào container
docker run -d \
  --name postgres \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# Anonymous volume (tự generate tên)
docker run -v /var/lib/postgresql/data postgres:16

# Inspect volume
docker volume inspect pgdata
# → Mountpoint: /var/lib/docker/volumes/pgdata/_data

# List và cleanup
docker volume ls
docker volume rm pgdata
docker volume prune          # xoá tất cả unused volumes
```

**Volume trong Dockerfile:**
```dockerfile
VOLUME /var/lib/postgresql/data
# Docker tự tạo anonymous volume khi container start
# Tốt hơn: đừng dùng VOLUME trong Dockerfile — để caller quyết định
```

**Volume share giữa containers:**
```bash
# Container 1 tạo data
docker run -d --name writer -v shared:/data ubuntu

# Container 2 đọc cùng data
docker run -d --name reader -v shared:/data:ro ubuntu
```

---

## Bind Mount

Map một đường dẫn của host vào container.

```bash
# Cú pháp -v
docker run -v /host/absolute/path:/container/path myapp
docker run -v $(pwd)/config:/app/config:ro nginx    # read-only

# Cú pháp --mount (explicit, khuyến nghị)
docker run --mount type=bind,source=/host/path,target=/container/path myapp
docker run --mount type=bind,source=$(pwd),target=/app,readonly myapp
```

**Dev workflow (live reload):**
```bash
docker run -d \
  --name dev \
  -v $(pwd)/src:/app/src \       # source code live sync
  -p 3000:3000 \
  node:20-alpine \
  npm run dev
```

**Config injection (production):**
```bash
docker run -d \
  -v /etc/myapp/config.yml:/app/config.yml:ro \
  -v /etc/ssl/certs/myapp.crt:/certs/app.crt:ro \
  -v /etc/ssl/private/myapp.key:/certs/app.key:ro \
  myapp:1.0.0
```

**Chú ý permissions:**
```bash
# File/dir trên host cần accessible bởi UID của process trong container
# Container chạy UID 1000, file trên host cần readable bởi 1000
chown -R 1000:1000 /host/data
# Hoặc chmod o+r nếu không muốn chown
```

---

## tmpfs

Mount vào RAM — không persist sau khi container stop/rm.

```bash
# Basic tmpfs
docker run --tmpfs /tmp myapp

# Với options
docker run --tmpfs /tmp:rw,size=100m,uid=1000 myapp

# Mount syntax
docker run --mount type=tmpfs,target=/tmp,tmpfs-size=100m myapp
```

**Use cases:**
- `/tmp` cho temporary files
- Session data không cần persist
- Secrets in-memory (không ghi ra disk)
- Cache tăng performance

---

## OverlayFS — Image Layers

```
Image: nginx:1.26
  ├── Layer 1: FROM debian:12-slim        (100MB)
  ├── Layer 2: RUN apt-get install nginx  (25MB)
  ├── Layer 3: COPY nginx.conf /etc/...   (2KB)
  └── Layer 4: COPY html/ /var/www/...    (5MB)

Container: nginx (running)
  ├── [Layer 1-4 read-only] ← shared với tất cả containers dùng image này
  └── [Writable layer] ← changes tại runtime (log files, pid, etc.)
```

**Copy-on-Write (CoW):**
- Container đọc file từ image layer → trực tiếp từ layer (không copy)
- Container ghi/sửa file từ image layer → **copy lên writable layer trước**, sau đó sửa
- Implication: file lớn bị modify → double disk usage tạm thời

```bash
# Xem layers của image
docker history nginx:1.26
docker image inspect nginx:1.26 | jq '.[0].RootFS.Layers'

# Xem thay đổi của container so với image
docker diff mycontainer
# C = changed, A = added, D = deleted
```

**Tại sao writable container layer là bad cho persistent data:**
- Mất khi `docker rm`
- Performance kém do CoW overhead
- Khó backup/migrate
→ Luôn dùng Volume hoặc Bind mount cho persistent data

---

## Volume Drivers

Plugin cho phép volume dùng external storage.

```bash
# NFS volume
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=nfs-server.internal,rw,nfsvers=4 \
  --opt device=:/exports/data \
  nfs-data

docker run -v nfs-data:/data myapp

# CIFS/SMB
docker volume create \
  --driver local \
  --opt type=cifs \
  --opt device=//smb-server/share \
  --opt o=username=user,password=pass \
  smb-data
```

**Third-party drivers:**
- `rexray` — Dell EMC storage
- `convoy` — NFS, EBS, GlusterFS
- `local-persist` — persist named volumes beyond pruning

---

## Data Persistence Patterns

### Pattern 1: Named volume (production stateful service)
```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

volumes:
  pgdata:
    driver: local
```

### Pattern 2: Bind mount (configuration)
```bash
# Config files managed outside Docker lifecycle
docker run -d \
  --name nginx \
  -v /etc/nginx/conf.d:/etc/nginx/conf.d:ro \
  -v /etc/ssl:/etc/ssl:ro \
  nginx:1.26
```

### Pattern 3: Init container populate volume
```yaml
services:
  init-data:
    image: myapp:1.0.0
    command: ["cp", "-r", "/app/default-data/.", "/data/"]
    volumes:
      - appdata:/data
    # Runs once, exits

  app:
    image: myapp:1.0.0
    volumes:
      - appdata:/data
    depends_on:
      init-data:
        condition: service_completed_successfully

volumes:
  appdata:
```

### Pattern 4: Backup volume
```bash
# Backup
docker run --rm \
  -v pgdata:/data:ro \
  -v $(pwd)/backup:/backup \
  ubuntu \
  tar czf /backup/pgdata-$(date +%Y%m%d).tar.gz -C /data .

# Restore
docker run --rm \
  -v pgdata:/data \
  -v $(pwd)/backup:/backup:ro \
  ubuntu \
  tar xzf /backup/pgdata-20260426.tar.gz -C /data
```

---

## Gotchas

- **Anonymous volumes và `docker rm`**: `docker rm` không xoá anonymous volumes. Dùng `docker rm -v` hoặc `docker volume prune` để dọn. Named volumes an toàn hơn vì explicit.
- **VOLUME in Dockerfile**: khi Dockerfile có `VOLUME /data`, mọi `docker run` tự tạo anonymous volume → khó kiểm soát. Tốt hơn là để user quyết định mount hay không.
- **Bind mount và selinux**: trên RHEL/CentOS có SELinux, bind mount cần `:z` (shared) hoặc `:Z` (private) label: `-v /host:/container:Z`. Thiếu → permission denied.
- **NFS và UID mapping**: file trên NFS có UID từ NFS server. Container UID có thể không match → permission denied. Sync UIDs hoặc dùng `no_root_squash` (risky).
- **tmpfs và container restart**: `--restart always` + `--tmpfs /tmp` → tmpfs content bị xoá mỗi lần container restart. Đây là behavior mong đợi nhưng dễ gây nhầm.
- **Layer cache và COPY order**: `COPY requirements.txt .` → `RUN pip install` → `COPY . .` giúp tận dụng build cache. Đảo ngược thứ tự → pip install chạy lại mỗi lần thay đổi source code.
