---
title: Docker Swarm
tags:
  - docker
  - swarm
  - orchestration
  - deep-dive
date: 2026-04-26
---

# Docker Swarm

## What is Docker Swarm?

Docker Swarm là native container orchestrator built vào Docker — biến nhiều Docker hosts thành **cluster** với scheduling, scaling, load balancing, và rolling updates.

```
┌────────────────────────────────────────────────────────┐
│                    Docker Swarm Cluster                 │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  Manager 1  │  │  Manager 2  │  │  Manager 3  │    │
│  │  (Leader)   │  │  (Follower) │  │  (Follower) │    │
│  │  Raft quorum│◄─┤  Raft quorum├─►│  Raft quorum│    │
│  └──────┬──────┘  └─────────────┘  └─────────────┘    │
│         │ schedule tasks                               │
│  ┌──────▼──────┐  ┌─────────────┐  ┌─────────────┐   │
│  │  Worker 1   │  │  Worker 2   │  │  Worker 3   │   │
│  │ [task][task]│  │  [task]     │  │ [task][task]│   │
│  └─────────────┘  └─────────────┘  └─────────────┘   │
└────────────────────────────────────────────────────────┘
```

**Swarm vs Kubernetes:**

| | Docker Swarm | Kubernetes |
|---|---|---|
| Setup complexity | Low (5 phút) | High (hours) |
| Learning curve | Low | High |
| Scaling | Manual | Auto (HPA/VPA) |
| Ecosystem | Limited | Huge |
| Rolling updates | Native | Native |
| Storage | Volume drivers | PV/PVC/CSI |
| Use case | Simple stacks, small teams | Complex microservices |

---

## Architecture

### Manager Nodes

- Maintain cluster state (Raft consensus)
- Schedule tasks lên worker nodes
- Expose Docker API (swarm commands)
- **Odd number of managers** để có quorum: 1, 3, 5, 7

**Fault tolerance:**

| Managers | Quorum needed | Fault tolerance |
|---|---|---|
| 1 | 1 | 0 |
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

### Worker Nodes

- Chạy container tasks (được scheduler assign)
- Không tham gia Raft consensus
- Communicate với managers qua gossip protocol (TCP/UDP 7946)

### Raft Consensus

Managers dùng Raft để agree on cluster state:
- Leader nhận writes, propagate sang followers
- Nếu leader fail → election, follower có most up-to-date log trở thành leader
- Quorum (majority) cần để commit state changes
- Cluster state lưu encrypted trong `/var/lib/docker/swarm/`

---

## Setup Swarm Cluster

```bash
# ─── Node 1: Init swarm ───
docker swarm init --advertise-addr 192.168.1.10
# Output:
# Swarm initialized: current node (xxx) is now a manager.
# To add a worker: docker swarm join --token SWMTKN-1-xxx 192.168.1.10:2377

# Lấy join tokens
docker swarm join-token manager   # để add thêm manager
docker swarm join-token worker    # để add worker

# ─── Node 2, 3: Join as manager ───
docker swarm join \
  --token SWMTKN-1-<manager-token> \
  192.168.1.10:2377

# ─── Node 4+: Join as worker ───
docker swarm join \
  --token SWMTKN-1-<worker-token> \
  192.168.1.10:2377

# ─── Verify ───
docker node ls
# ID                  HOSTNAME    STATUS    AVAILABILITY   MANAGER STATUS
# xxx * (Leader)      manager1    Ready     Active         Leader
# yyy                 manager2    Ready     Active         Reachable
# zzz                 manager3    Ready     Active         Reachable
# aaa                 worker1     Ready     Active
```

**Ports cần mở giữa nodes:**
```
TCP  2377  cluster management (manager nodes)
TCP/UDP 7946  node discovery (gossip)
UDP  4789  overlay network VXLAN (data plane)
```

---

## Services

Service là đơn vị deploy trong Swarm — thay vì `docker run` (container đơn lẻ).

```bash
# Tạo service
docker service create \
  --name nginx-web \
  --replicas 3 \
  --publish published=80,target=80 \
  --network myoverlay \
  --constraint 'node.role == worker' \
  nginx:1.26-alpine

# List services
docker service ls

# List tasks (containers) của service
docker service ps nginx-web

# Service logs
docker service logs -f nginx-web

# Scale
docker service scale nginx-web=5

# Update (rolling)
docker service update \
  --image nginx:1.27-alpine \
  --update-parallelism 1 \
  --update-delay 30s \
  --update-order start-first \
  nginx-web

# Remove service
docker service rm nginx-web
```

---

## Stack — Compose file cho Swarm

Stack = Docker Compose file deployed vào Swarm cluster.

```yaml
# stack.yml
version: "3.9"

services:
  web:
    image: myapp:1.2.3              # phải là pre-built image (không dùng build:)
    networks:
      - frontend
    ports:
      - "80:3000"
    deploy:
      replicas: 3
      update_config:
        parallelism: 1              # update 1 container mỗi lần
        delay: 30s
        order: start-first          # start new → healthy → stop old (zero-downtime)
        failure_action: rollback    # tự rollback nếu update fail
        monitor: 60s                # monitor window sau mỗi update step
      rollback_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      placement:
        constraints:
          - node.role == worker
          - node.labels.region == us-east
        preferences:
          - spread: node.labels.zone  # phân bổ đều qua zones
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s

  db:
    image: postgres:16-alpine
    networks:
      - backend
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.storage == ssd

  nginx:
    image: nginx:1.26-alpine
    networks:
      - frontend
    ports:
      - "443:443"
    configs:
      - source: nginx_config
        target: /etc/nginx/nginx.conf
    secrets:
      - source: ssl_cert
        target: /etc/ssl/certs/app.crt
      - source: ssl_key
        target: /etc/ssl/private/app.key
    deploy:
      replicas: 2

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
    internal: true                  # không có outbound internet

volumes:
  pgdata:

secrets:
  db_password:
    external: true
  ssl_cert:
    external: true
  ssl_key:
    external: true

configs:
  nginx_config:
    external: true
```

```bash
# Deploy / update stack
docker stack deploy -c stack.yml myapp

# List stacks
docker stack ls

# List services trong stack
docker stack services myapp

# List tasks
docker stack ps myapp

# Remove stack
docker stack rm myapp
```

---

## Secrets & Configs

### Secrets

Encrypted ở rest (Raft log), chỉ decrypt trên node đang chạy task, mount vào `/run/secrets/`.

```bash
# Tạo secret
echo "supersecretpassword" | docker secret create db_password -
docker secret create ssl_cert ./certs/app.crt

# List / inspect (không xem được value)
docker secret ls
docker secret inspect db_password

# Remove
docker secret rm db_password
```

```yaml
# Trong service
services:
  app:
    secrets:
      - db_password               # mount tại /run/secrets/db_password
      - source: ssl_cert
        target: /etc/ssl/app.crt  # custom path
        uid: "1000"
        mode: 0400                # chỉ owner đọc được

secrets:
  db_password:
    external: true
```

### Configs

Plain text configs (nginx.conf, app.conf) — không encrypted, dùng cho config files.

```bash
docker config create nginx_config ./nginx.conf
docker config ls
docker config rm nginx_config
```

```yaml
services:
  nginx:
    configs:
      - source: nginx_config
        target: /etc/nginx/nginx.conf
        mode: 0444

configs:
  nginx_config:
    external: true
```

---

## Node Labels & Placement

```bash
# Add labels
docker node update --label-add region=us-east worker1
docker node update --label-add zone=us-east-1a worker1
docker node update --label-add storage=ssd worker2

# Xem labels
docker node inspect worker1 --format '{{json .Spec.Labels}}'

# Remove label
docker node update --label-rm zone worker1
```

```yaml
deploy:
  placement:
    constraints:
      - node.role == worker
      - node.hostname == worker1
      - node.labels.region == us-east
    preferences:
      - spread: node.labels.zone  # phân bổ đều qua zones (best effort)
```

---

## Rolling Updates & Rollback

```bash
# Update service image
docker service update \
  --image myapp:1.3.0 \
  --update-parallelism 2 \     # 2 tasks cùng lúc
  --update-delay 30s \         # đợi 30s giữa batches
  --update-order start-first \ # start mới → healthy → stop cũ
  myapp_web

# Monitor update progress
docker service ps myapp_web

# Rollback về version trước (automatic nếu failure_action: rollback)
docker service rollback myapp_web

# Rollback về image cụ thể
docker service update --image myapp:1.2.0 myapp_web
```

**Update order:**
- `start-first`: start new → health check pass → stop old → **zero-downtime**
- `stop-first` (default): stop old → start new → brief downtime

---

## Routing Mesh & Service Discovery

### Ingress Routing Mesh

```
External → any Swarm node :80 → Routing Mesh → Service VIP → Round-robin → Container
```

```bash
docker service create \
  --publish mode=ingress,published=80,target=3000 \  # routing mesh
  --replicas 3 \
  myapp:latest
# Request vào bất kỳ node nào → forward tới container đang chạy
```

### Host Mode (bypass routing mesh)

```bash
docker service create \
  --publish mode=host,published=80,target=3000 \  # direct port trên node
  myapp:latest
# Chỉ accessible qua node đang chạy container
# Không dùng VIP — cần external load balancer để distribute traffic
```

### DNS Service Discovery

```bash
# Trong container cùng overlay network
curl http://myapp_web:3000/health    # Service name → VIP → load balance
nslookup tasks.myapp_web             # Resolve tất cả task IPs (bypass VIP)
```

---

## Ops Runbook

```bash
# Cluster health
docker node ls
docker node inspect manager1 --pretty
docker service ls
docker stack ps myapp --filter "desired-state=running"

# Drain node trước maintenance
docker node update --availability drain worker1
# Tasks được rescheduled tự động

# Sau maintenance
docker node update --availability active worker1

# Force rebalance sau khi add node mới
docker service update --force myapp_web

# Promote / demote
docker node promote worker1          # worker → manager
docker node demote manager3          # manager → worker

# Remove node
docker swarm leave                   # chạy trên node muốn remove
docker node rm worker1               # chạy trên manager (sau khi node leave)

# Rotate join token (bảo mật)
docker swarm join-token --rotate worker
docker swarm join-token --rotate manager
```

---

## Gotchas

- **Odd number of managers**: chẵn managers (2, 4) không cải thiện fault tolerance, chỉ khó đạt quorum hơn. Luôn dùng 1, 3, 5, 7.
- **Stack không support `build:`**: `docker stack deploy` cần pre-built image trong registry. Không build on-the-fly như Compose.
- **Volumes không sync giữa nodes**: service `replicas: 3` với local volume → mỗi node có volume riêng, data không share. Dùng NFS driver hoặc shared storage cho stateful services.
- **Secrets chỉ add khi service create**: không thể update secret value mà không recreate service. Pattern: versioned secret names (`db_password_v2`) → update service với secret mới → remove old.
- **Quorum lost**: nếu mất majority managers, cluster freeze. Emergency recovery: `docker swarm init --force-new-cluster` trên node còn lại (có thể mất state).
- **`docker stack deploy` là idempotent**: chạy lại sẽ update services có thay đổi. Safe để dùng trong CI/CD.
- **start-first cần đủ resources**: `update_order: start-first` tạm thời cần resources cho cả old + new container. Đảm bảo nodes có đủ headroom.
