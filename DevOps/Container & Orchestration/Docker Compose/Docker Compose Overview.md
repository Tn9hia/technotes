---
title: Docker Compose
tags:
  - docker
  - compose
  - orchestration
  - deep-dive
date: 2026-04-26
---

# Docker Compose

## What is Docker Compose?

Docker Compose là tool định nghĩa và chạy **multi-container applications** bằng YAML file. Thay vì chạy nhiều `docker run` commands, describe toàn bộ stack trong `compose.yml`.

```
compose.yml → docker compose up → tạo network + volumes + containers → stack running
```

**Compose file naming** (ưu tiên từ trên xuống):
1. `compose.yml` (v2+ spec)
2. `compose.yaml`
3. `docker-compose.yml` (legacy)
4. `docker-compose.yaml`

---

## Compose File Structure

```yaml
# compose.yml
name: myapp                        # project name (default: dirname)

services:
  web:                             # service name → container name: <project>-web-1
    image: nginx:1.26-alpine
    # hoặc build từ Dockerfile:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        BUILD_DATE: "2026-04-26"
      target: production           # multi-stage target
    container_name: web            # override default naming (fixed name)
    ports:
      - "80:80"
      - "127.0.0.1:443:443"        # bind chỉ localhost
    environment:
      NODE_ENV: production
      PORT: "3000"
    env_file:
      - .env                       # load từ file
      - .env.production
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - static_files:/var/www/html
    networks:
      - frontend
      - backend
    depends_on:
      db:
        condition: service_healthy  # chờ db healthy trước khi start
      redis:
        condition: service_started  # chỉ chờ started (default)
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s            # grace period khi container mới start
    restart: unless-stopped        # no | always | on-failure | unless-stopped
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    labels:
      com.company.team: platform
      traefik.enable: "true"

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - backend

volumes:
  pgdata:
  redis_data:
  static_files:
    driver: local                  # explicit driver

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true                 # không có outbound internet access

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

---

## Environment Variables

### .env file

```bash
# .env (không commit — add to .gitignore)
POSTGRES_PASSWORD=supersecret
REDIS_PASSWORD=redispass
APP_VERSION=1.2.3
DOMAIN=myapp.internal
```

```yaml
# compose.yml — auto-load .env (không cần khai báo)
services:
  web:
    image: myapp:${APP_VERSION}   # sử dụng từ .env
    environment:
      VIRTUAL_HOST: ${DOMAIN}
```

```bash
# Override env khi chạy
APP_VERSION=1.3.0 docker compose up -d

# Xem resolved config
docker compose config             # show full resolved compose file
docker compose config --quiet     # validate only
```

### .env.example (commit được)
```bash
# .env.example — template cho team
POSTGRES_PASSWORD=changeme
REDIS_PASSWORD=changeme
APP_VERSION=latest
DOMAIN=localhost
```

### Secrets (không qua env vars)

```yaml
services:
  app:
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password   # app đọc file
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt  # local file
    # hoặc external (Docker Swarm secret):
    # external: true
```

---

## depends_on & Health Check

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy     # chờ db healthy
        restart: true                  # restart app nếu db restart
      migration:
        condition: service_completed_successfully  # chờ job hoàn thành

  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  migration:
    image: myapp:latest
    command: ["python", "manage.py", "migrate"]
    depends_on:
      db:
        condition: service_healthy
```

**Health check conditions:**
- `service_started`: container started (không check health)
- `service_healthy`: healthcheck passed
- `service_completed_successfully`: container exited với code 0

---

## Override File

```yaml
# compose.yml — base (production-like defaults)
services:
  web:
    image: myapp:${VERSION:-latest}
    restart: unless-stopped
    ports:
      - "80:3000"

# compose.override.yml — auto-loaded in development
# (merge với compose.yml khi chạy docker compose up)
services:
  web:
    build: .                       # build local thay vì pull image
    volumes:
      - .:/app:cached              # live code reload
      - /app/node_modules          # giữ node_modules từ image
    environment:
      NODE_ENV: development
      DEBUG: "true"
    ports:
      - "3001:3000"                # different port cho dev
```

```bash
# Auto merge: compose.yml + compose.override.yml
docker compose up

# Production — không load override
docker compose -f compose.yml up

# Staging
docker compose -f compose.yml -f compose.staging.yml up

# CI
docker compose -f compose.yml -f compose.test.yml run test
```

---

## CLI Commands

```bash
# Lifecycle
docker compose up                  # create + start (foreground)
docker compose up -d               # detached (background)
docker compose up --build          # force rebuild images
docker compose up --force-recreate # recreate containers dù image không đổi
docker compose up web db           # chỉ start specific services

docker compose down                # stop + remove containers + networks
docker compose down -v             # cũng remove volumes
docker compose down --rmi all      # cũng remove images

docker compose start               # start existing containers (không create)
docker compose stop                # stop containers (không remove)
docker compose restart web         # restart specific service

# Status & Logs
docker compose ps                  # list containers
docker compose ps --status running # filter
docker compose logs                # tất cả services
docker compose logs -f web         # follow specific service
docker compose logs --tail 50 db   # last 50 lines

# Exec & Run
docker compose exec web sh         # exec vào running container
docker compose run --rm web npm test  # one-off command (tạo container mới)
docker compose run --rm --no-deps db psql -U postgres  # skip depends_on

# Build
docker compose build               # build tất cả services có `build:`
docker compose build --no-cache web  # no cache

# Config
docker compose config              # show merged + resolved config
docker compose convert             # normalize to canonical form

# Scale
docker compose up -d --scale web=3  # chạy 3 instances của web

# Cleanup
docker compose rm                  # remove stopped containers
```

---

## Production Patterns

### Resource limits
```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 1G
        reservations:
          cpus: "0.5"
          memory: 512M
```

### Restart policy
```yaml
restart: unless-stopped    # production default
restart: always            # restart kể cả docker daemon restart
restart: on-failure        # chỉ khi exit code != 0
restart: "on-failure:3"    # max 3 retries
restart: "no"              # không restart (CI/batch jobs)
```

### Logging
```yaml
logging:
  driver: json-file     # default
  options:
    max-size: "10m"      # rotate sau 10MB
    max-file: "5"        # giữ 5 files

# Gửi tới Loki
logging:
  driver: loki
  options:
    loki-url: "http://loki:3100/loki/api/v1/push"
    loki-labels: "job=myapp,env=production"

# Gửi tới syslog
logging:
  driver: syslog
  options:
    syslog-address: "tcp://log-server:514"
    tag: "myapp/{{.Name}}"
```

### Zero-downtime với external load balancer
```yaml
services:
  web:
    image: myapp:${VERSION}
    scale: 2
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
```

```bash
# Rolling update
export VERSION=1.3.0
docker compose up -d --no-deps --scale web=4 web   # scale up
docker compose up -d --no-deps --scale web=2 web   # scale down (remove old)
# Load balancer health check sẽ drain old containers
```

---

## Docker Compose vs Kubernetes

| | Docker Compose | Kubernetes |
|---|---|---|
| **Scope** | Single host | Multi-host cluster |
| **Complexity** | Low | High |
| **Scaling** | Manual (`--scale`) | Auto (HPA) |
| **Self-healing** | Restart policy | Deployment controller |
| **Networking** | Bridge/overlay (single host) | CNI (cluster-wide) |
| **Storage** | Volume drivers | PV/PVC/StorageClass |
| **Config/Secrets** | env_file/secrets (file-based) | ConfigMap/Secret |
| **Service discovery** | Container DNS | CoreDNS + Service |
| **Rolling updates** | Manual | Native (maxSurge/maxUnavailable) |
| **Health-based routing** | No (restart only) | Yes (readiness probe) |
| **Use case** | Dev env, small deployments | Production microservices |

**Compose → K8s migration**: `kompose convert` tạo K8s manifests từ compose file (rough conversion, cần review).

---

## Profiles — Conditional Services

```yaml
services:
  app:
    image: myapp:latest

  db:
    image: postgres:16

  adminer:
    image: adminer          # UI để quản lý DB
    profiles: ["tools"]    # chỉ start khi profile "tools" active

  prometheus:
    image: prom/prometheus
    profiles: ["monitoring"]

  grafana:
    image: grafana/grafana
    profiles: ["monitoring"]
```

```bash
docker compose up                      # chỉ app + db
docker compose --profile tools up      # app + db + adminer
docker compose --profile monitoring up # app + db + prometheus + grafana
```

---

## Gotchas

- **compose.yml vs docker-compose.yml**: Docker Compose v2 (plugin, `docker compose`) ưu tiên `compose.yml`. v1 (standalone, `docker-compose`) dùng `docker-compose.yml`. Dùng v2 plugin trong production.
- **depends_on không đảm bảo app ready**: `condition: service_started` chỉ chờ container start, không chờ app trong container sẵn sàng. Cần healthcheck + `condition: service_healthy`.
- **Environment variable precedence**: shell env > .env file > compose.yml defaults. `docker compose config` để xem giá trị cuối cùng.
- **Volume naming**: Nếu không có `name: myapp` ở project level, volume tên là `<dirname>_pgdata`. Khi chạy ở path khác → tạo volume mới, mất data cũ. Dùng `name:` explicit trong volumes section.
- **`docker compose run` vs `exec`**: `run` tạo container mới (không vào container đang chạy), `exec` vào container existing. `run --rm` để cleanup sau khi xong.
- **Scaling và port conflicts**: không thể scale service có fixed host port mapping. Dùng port range (`"80-82:80"`) hoặc không map port (dùng reverse proxy).
- **Override merge strategy**: list values (ports, volumes, environment) được **merge** không replace. Nếu muốn replace hoàn toàn: không được — dùng multiple -f files và khai báo lại đầy đủ.
