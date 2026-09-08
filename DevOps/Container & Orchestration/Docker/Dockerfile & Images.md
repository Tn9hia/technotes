---
title: Dockerfile & Images
tags:
  - docker
  - dockerfile
  - image
  - deep-dive
date: 2026-04-26
---

# Dockerfile & Images

## OverlayFS — Image Layer Model

```
docker build → tạo image với nhiều layers xếp chồng

FROM ubuntu:22.04          Layer 1: 77MB  (pulled từ registry)
RUN apt-get install curl   Layer 2: 12MB  (thêm vào)
COPY app /app              Layer 3: 5MB   (thêm vào)
RUN pip install -r req.txt Layer 4: 80MB  (thêm vào)
COPY . /app                Layer 5: 2MB   (thêm vào)
                           ─────────────────
                           Total:  176MB
```

**OverlayFS mechanics:**
```
Container A (running)
  └── [writable layer]   ← CoW writes go here
      [Layer 5] app/     ← read-only
      [Layer 4] /usr/lib/python  ← read-only
      [Layer 3] /app     ← read-only
      [Layer 2] /usr/bin/curl    ← read-only
      [Layer 1] /bin, /usr, ...  ← read-only (shared!)

Container B (running, same image)
  └── [writable layer]   ← separate writable layer
      [Layer 1-5] ← SHARED with Container A (single copy on disk)
```

Layers được cache và chia sẻ — 10 containers cùng image → không dùng 10x disk space.

```bash
docker history nginx:1.26         # xem layers và size
docker image inspect nginx:1.26   # full metadata
docker image ls                   # list images
docker image prune                # xoá dangling images
```

---

## Dockerfile Instructions

```dockerfile
# ─── Build arguments (available only during build) ───
ARG BASE_IMAGE=node:20-alpine
ARG BUILD_DATE

# ─── Base image ───
FROM ${BASE_IMAGE}

# ─── Metadata ───
LABEL maintainer="team@company.com"
LABEL version="1.0.0"
LABEL org.opencontainers.image.created=${BUILD_DATE}

# ─── Working directory ───
WORKDIR /app

# ─── Environment variables (baked into image, visible via inspect) ───
ENV NODE_ENV=production
ENV PORT=3000

# ─── Install deps TRƯỚC khi copy source (cache layer) ───
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# ─── Copy source ───
COPY --chown=node:node . .

# ─── Expose port (documentation only — không thực sự publish) ───
EXPOSE 3000

# ─── Non-root user ───
USER node

# ─── Health check ───
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# ─── Startup ───
ENTRYPOINT ["node"]         # không override được bằng CMD
CMD ["server.js"]           # override được bằng docker run <image> <cmd>
# Kết quả: node server.js
```

**ENTRYPOINT vs CMD:**
| | ENTRYPOINT | CMD |
|---|---|---|
| Override | `docker run --entrypoint` | `docker run <image> <cmd>` |
| Combine | exec form ENTRYPOINT + CMD = full command | Standalone nếu không có ENTRYPOINT |
| Use case | Fixed executable | Default arguments |

```dockerfile
# Pattern 1: Fixed command
ENTRYPOINT ["nginx", "-g", "daemon off;"]

# Pattern 2: Configurable args
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080"]       # default, override với: docker run img --port 9090

# Pattern 3: Shell script wrapper
COPY entrypoint.sh /
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

---

## Layer Caching — Instruction Order

Cache invalidate khi instruction thay đổi **hoặc khi layer trên thay đổi**.

```dockerfile
# BAD — thay đổi source code invalidate npm install cache
FROM node:20-alpine
COPY . .                          ← thay đổi bất kỳ file nào → cache miss
RUN npm ci                        ← chạy lại mỗi lần

# GOOD — tách deps và source
FROM node:20-alpine
COPY package*.json ./             ← chỉ cache miss khi package.json thay đổi
RUN npm ci
COPY . .                          ← cache miss → nhưng npm install không chạy lại

# Python
FROM python:3.12-slim
COPY requirements.txt .
RUN pip install -r requirements.txt  ← cache miss chỉ khi requirements.txt thay đổi
COPY . .

# Java/Maven
FROM maven:3.9-eclipse-temurin-21
COPY pom.xml .
RUN mvn dependency:go-offline        ← download deps trước
COPY src ./src
RUN mvn package -DskipTests
```

**Busting cache intentionally:**
```bash
docker build --no-cache .            # bypass tất cả cache
docker build --build-arg CACHE_BUST=$(date +%s) .  # bust từ ARG CACHE_BUST
```

---

## Multi-stage Build

Build image nhỏ bằng cách tách build environment khỏi runtime.

```dockerfile
# ─── Stage 1: Build ───
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                           # bao gồm devDependencies
COPY . .
RUN npm run build                    # compile TypeScript, bundle, etc.

# ─── Stage 2: Production runtime ───
FROM node:20-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production         # chỉ runtime deps
COPY --from=builder /app/dist ./dist # chỉ lấy build output
USER node
CMD ["node", "dist/server.js"]
# Result: không có TypeScript, không có devDeps, không có source .ts files
```

```dockerfile
# Go — static binary → distroless
FROM golang:1.22 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app ./cmd/server

FROM gcr.io/distroless/static-debian12
COPY --from=builder /app /app
ENTRYPOINT ["/app"]
# Result: ~5MB image (vs ~800MB golang image)
```

```dockerfile
# Java Maven → JRE only
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
USER 1000
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Truy cập artifacts từ stage cụ thể:**
```bash
docker build --target builder -t myapp:builder .   # build chỉ đến stage "builder"
docker build --target production -t myapp:latest . # full build
```

---

## Base Image Strategy

| Image | Size | Use case | Gotcha |
|---|---|---|---|
| `ubuntu:22.04` / `debian:12` | ~80MB | General purpose, cần apt | Lớn |
| `debian:12-slim` | ~30MB | Debian base, minimal | Thiếu tools debug |
| `alpine:3.19` | ~7MB | Minimal, scripting | musl libc (khác glibc) |
| `node:20-alpine` | ~60MB | Node apps | musl |
| `python:3.12-slim` | ~50MB | Python apps | |
| `gcr.io/distroless/nodejs20` | ~100MB | Node production | Không có shell |
| `gcr.io/distroless/static` | ~2MB | Go static binary | Không có gì ngoài binary |
| `scratch` | 0MB | Fully static binary | Không có shell, libs |

**Alpine gotchas:**
```dockerfile
# Alpine dùng musl libc → một số compiled binaries không chạy được
# apk thay vì apt-get
FROM alpine:3.19
RUN apk add --no-cache curl nginx

# Nếu app cần glibc (compiled trên glibc system):
FROM frolvlad/alpine-glibc  # alpine + glibc compatibility layer
```

---

## .dockerignore

Tương tự `.gitignore` — loại file/dir khỏi build context gửi lên daemon.

```dockerignore
# Version control
.git
.gitignore

# Dependencies (rebuild trong container)
node_modules
vendor/
__pycache__/
*.pyc
target/
*.class

# Development config
.env
.env.local
*.env.*
docker-compose*.yml
Makefile

# Test artifacts
coverage/
.pytest_cache/
*.test

# IDE
.vscode/
.idea/
*.swp

# Build artifacts
dist/
build/
*.o
*.a

# Docs
*.md
docs/

# Logs
*.log
logs/
```

**Tại sao quan trọng:**
1. **Performance**: nhỏ build context → gửi lên daemon nhanh hơn
2. **Security**: không leak `.env`, credentials, `.ssh/` vào image
3. **Cache**: tránh invalidate cache vô lý (`.git/` thay đổi mỗi commit)

---

## Image Tagging & Versioning

```bash
# Tagging
docker build -t myapp:1.2.3 .
docker build -t myapp:1.2.3 -t myapp:1.2 -t myapp:latest .

# Conventional tags
myapp:1.2.3                  # immutable — production (semver)
myapp:1.2                    # floating minor
myapp:latest                 # floating — KHÔNG dùng trong production manifests
myapp:sha-abc1234            # git SHA — traceable
myapp:20260426-1.2.3         # date + version

# Tag và push
docker tag myapp:1.2.3 registry.internal/team/myapp:1.2.3
docker push registry.internal/team/myapp:1.2.3

# Tạo multi-arch image
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t registry.internal/myapp:1.2.3 \
  --push .
```

---

## Optimize Image Size

```bash
# Xem size per layer
docker history myapp:latest

# Dive tool (third-party) — interactive layer explorer
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:latest myapp:latest
```

**Techniques:**
```dockerfile
# 1. Combine RUN commands (reduce layers)
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      curl \
      nginx && \
    rm -rf /var/lib/apt/lists/*    # cleanup apt cache trong cùng layer

# 2. Multi-stage (biggest impact)
# → xem phần trên

# 3. Specific package versions (reproducible + no unwanted extras)
RUN apt-get install -y nginx=1.24.0-1~jammy

# 4. --no-install-recommends
RUN apt-get install -y --no-install-recommends nginx

# 5. Remove dev tools sau khi dùng
RUN apt-get install -y gcc python3-dev && \
    pip install -r requirements.txt && \
    apt-get remove -y gcc python3-dev && \
    apt-get autoremove -y
```

---

## Image Scanning

```bash
# Trivy — scan vulnerabilities
trivy image myapp:1.2.3
trivy image --severity HIGH,CRITICAL myapp:1.2.3
trivy image --exit-code 1 --severity CRITICAL myapp:1.2.3   # fail CI nếu có CRITICAL

# Grype
grype myapp:1.2.3
grype myapp:1.2.3 --fail-on critical

# Integrate vào CI
# .github/workflows/build.yml
# - name: Scan image
#   run: trivy image --exit-code 1 --severity CRITICAL ${{ env.IMAGE }}
```

---

## Gotchas

- **`latest` tag là antipattern cho production**: `latest` thay đổi mỗi lần push → không biết đang chạy version nào, khó rollback. Luôn dùng immutable tag (semver hoặc git SHA).
- **ADD vs COPY**: `ADD` có hidden features (untar archives tự động, download URLs). Luôn dùng `COPY` trừ khi cần untar.
- **ENV secrets**: `ENV DB_PASSWORD=secret` bake vào image history → `docker history` thấy. Dùng runtime env vars, không build-time.
- **WORKDIR tự tạo**: `WORKDIR /app` tạo directory nếu không tồn tại. Không cần `RUN mkdir -p /app`.
- **USER instruction và volumes**: nếu dùng `USER 1000` và có volume mount, directory phải chown 1000 trước trong image: `RUN chown 1000:1000 /data` trước `USER 1000`.
- **Multi-stage và BuildKit**: BuildKit (default từ Docker 23) chạy stages song song nếu không có dependency → nhanh hơn. Enable: `DOCKER_BUILDKIT=1 docker build .`
- **HEALTHCHECK exit code**: `HEALTHCHECK` dùng exit code của command (0=healthy, 1=unhealthy). `|| exit 1` sau curl là pattern chuẩn. Container tiếp tục chạy dù unhealthy — Docker chỉ report status.
