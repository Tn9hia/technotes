# Redis — 7.x
Tags: #redis #cache #database #in-memory #message-broker
Last updated: 2026-04-30

---

## 1. What — Nó là cái gì?

Redis (Remote Dictionary Server) là in-memory data store — lưu tất cả data trong RAM và expose qua một tập lệnh đơn giản. Nó không phải chỉ là cache: Redis hỗ trợ nhiều data structures (List, Hash, Sorted Set, Stream…) giúp nó kiêm luôn vai trò message broker, session store, leaderboard engine, và distributed lock provider.

---

## 2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?

Nếu không có Redis, mỗi request đều phải hit PostgreSQL — dù là lấy config tĩnh hay session của user. Với hệ thống microservices, đó là hàng nghìn DB connections đồng thời, latency leo thang, và DB trở thành bottleneck duy nhất.

Redis giải quyết ba vấn đề cụ thể:
- **Latency**: RAM lookup < 1ms vs disk I/O ~5-10ms. Cho những data đọc nhiều, thay đổi ít (user session, config, access token) — cache Redis trước, DB chỉ là fallback.
- **Thundering herd**: Khi nhiều service cùng cần một piece of data (ví dụ product catalog sau cache miss) → chỉ 1 request được phép rebuild cache (distributed lock), còn lại chờ. Không có Redis, tất cả đổ vào DB cùng lúc.
- **Cross-service communication**: Pub/Sub và Streams cho phép services giao tiếp async mà không cần deploy Kafka cho những use cases đơn giản.

---

## 3. When — Dùng khi nào / KHÔNG dùng khi nào?

**Dùng khi:**
- Cache kết quả query đắt tiền (product catalog, user profile, config)
- Session store cho stateless services (JWT không cần lưu, nhưng refresh token và session data thì cần)
- Rate limiting (INCR + EXPIRE là atomic, không cần transaction)
- Distributed lock (SET NX EX)
- Leaderboard / ranking (Sorted Set với ZADD/ZREVRANK)
- Job queue nhỏ đến vừa (List BLPOP hoặc Stream)
- Real-time pub/sub (chat, notification — nếu không cần persistence)

**KHÔNG dùng khi:**
- Data cần persist bền vững và không thể mất — Redis AOF vẫn có thể mất ~1s data. Dùng PostgreSQL.
- Data lớn hơn RAM — Redis scale bằng cách thêm RAM/node, không phải disk. Lưu blob, file, log dài hạn vào object storage.
- Complex query với joins, aggregations — dùng PostgreSQL. Redis không có query language.
- Kafka replacement khi cần consumer group offset tracking lâu dài, retention days, replay — Streams có thể làm nhưng không phải thế mạnh.
- Primary database — Redis là second layer, không phải source of truth.

---

## 4. Where — Architecture — Nó nằm ở đâu trong hệ thống?

```
                    ┌─────────────────────────────────────────┐
                    │           Microservices Layer            │
                    │  Service A   Service B   Service C       │
                    └──────┬────────────┬───────────┬─────────┘
                           │            │           │
                    ┌──────▼────────────▼───────────▼─────────┐
                    │              Redis Layer                  │
                    │                                          │
                    │   ┌──────────┐      ┌─────────────┐     │
                    │   │  Cache   │      │   Pub/Sub   │     │
                    │   │ Sessions │      │   Streams   │     │
                    │   │  Locks   │      │   Queues    │     │
                    │   └──────────┘      └─────────────┘     │
                    │                                          │
                    │   Master ──────► Replica 1               │
                    │      │      ──► Replica 2               │
                    │      │                                   │
                    │   Sentinel 1, 2, 3 (monitor + failover) │
                    └──────────────────────────────────────────┘
                           │
                    ┌──────▼─────────────────────────────────┐
                    │           PostgreSQL (source of truth)  │
                    └────────────────────────────────────────┘
```

**Dependency:**
- Services → Redis qua TCP port 6379 (hoặc 26379 cho Sentinel)
- Redis → không cần DB nào khác (standalone in-memory)
- Monitoring: redis_exporter → Prometheus → Grafana
- Kubernetes: Bitnami Helm chart (Sentinel mode), PVC cho RDB/AOF persistence

---

## 5. How — Cơ chế hoạt động

**Single-threaded event loop:**
Redis xử lý tất cả commands trong 1 thread — không có lock, không có race condition. Đây là lý do latency thấp và predictable. Redis 6.0+ thêm I/O threads cho network read/write nhưng command execution vẫn single-threaded.

**In-memory với optional persistence:**
Data sống trong RAM. Persistence là optional add-on:
- **RDB** (snapshot): Fork process, dump memory ra file. Nhanh khi restart, có thể mất vài phút data.
- **AOF** (append-only log): Ghi mọi write command. Mất tối đa ~1s data (everysec mode). Restart chậm hơn.
- **Hybrid** (recommended): AOF file bắt đầu bằng RDB snapshot, append phần còn lại.

**Eviction khi đầy RAM:**
Khi `used_memory >= maxmemory`, Redis chạy eviction policy. `allkeys-lru` là phổ biến nhất cho cache thuần — evict key ít được dùng gần đây nhất. Nếu không set `maxmemory` → Redis dùng hết RAM server → OOM killer.

**Replication — async:**
Master ghi vào bộ nhớ và trả lời client ngay lập tức, sau đó replication stream chạy async tới replicas. Replica lag thường < 1s nhưng **không phải zero** — đọc từ replica có thể nhận stale data.

**Sentinel — automatic failover:**
3 Sentinel process giám sát master. Khi master không respond > `down-after-milliseconds`, Sentinels vote (cần quorum >= 2/3). Sentinel thắng vote chọn replica tốt nhất (least replication lag) → promote → redirect các replicas còn lại. Toàn bộ quá trình mất ~10-30s.

---

## 6. Key Config — Cấu hình cần nhớ

```bash
# QUAN TRỌNG NHẤT — luôn set trong production
maxmemory 4gb                    # default = 0 = không limit → OOM
maxmemory-policy allkeys-lru     # default = noeviction → error khi đầy

# Persistence (recommended: hybrid)
aof-use-rdb-preamble yes
appendonly yes
appendfsync everysec             # default = everysec (OK). "always" quá chậm, "no" quá rủi ro

# Eviction async (giảm latency spike khi xóa)
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes

# Sentinel quorum — phải >= (n/2 + 1)
sentinel monitor mymaster <ip> 6379 2    # với 3 sentinels → quorum=2

# Memory limits trong K8s phải > maxmemory
# limits.memory = maxmemory × 1.2 (cho overhead)
```

**Default values nguy hiểm:**
- `maxmemory 0` — không có limit. Redis sẽ fill hết RAM.
- `maxmemory-policy noeviction` — trả error khi đầy thay vì evict. App phải handle OOM error.
- `bind 0.0.0.0` — Redis lắng nghe tất cả interfaces. Nếu không có firewall → exposed.
- `protected-mode yes` — tắt nếu không có auth → Redis accessible without password khi bound to all interfaces.

---

## 7. Security Considerations

**Attack surface:**
- Port 6379 exposed ra ngoài: Redis không có built-in TLS trước 6.0, không có ACL trước 6.0. Exposed Redis = full data access.
- `FLUSHALL`, `CONFIG`, `DEBUG`, `SLAVEOF` commands: attacker có thể xóa data hoặc pivot sang server khác.
- Lua scripts (`EVAL`): Lua chạy trong Redis process. Malicious Lua script có thể đọc toàn bộ keyspace.

**Hardening checklist tối thiểu:**
```bash
# 1. Require auth
requirepass <strong-random-password-32-chars>

# 2. Bind chỉ internal interface
bind 127.0.0.1 10.0.0.10    # không bind 0.0.0.0

# 3. Disable/rename dangerous commands
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command DEBUG ""
rename-command CONFIG "CONFIG_SECRET_STRING"   # disable hoàn toàn ảnh hưởng redis_exporter

# 4. Enable TLS (Redis 6.0+)
tls-port 6380
tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key
tls-ca-cert-file /etc/redis/tls/ca.crt

# 5. ACL (Redis 6.0+) — least privilege
ACL SETUSER appuser on ><password> ~user:* ~session:* +GET +SET +DEL +EXPIRE
ACL SETUSER readonly on ><password> ~cache:* +GET +MGET +EXISTS +TTL
```

**Kubernetes-specific:**
- Dùng NetworkPolicy để chỉ allow pods có label `redis-client: "true"` connect tới port 6379
- Lưu password trong Kubernetes Secret, không hardcode trong values.yaml
- Redis PVC: dùng `reclaimPolicy: Retain` — tránh mất data khi Helm uninstall

---

## 8. Ops Runbook — Production Notes

**Health check:**
```bash
redis-cli -a $PASS PING              # PONG = healthy
redis-cli -a $PASS INFO replication  # kiểm tra master/replica status
redis-cli -p 26379 SENTINEL masters  # kiểm tra Sentinel
```

**Metrics cần alert:**
| Metric | Threshold | Ý nghĩa |
|--------|-----------|---------|
| `redis_memory_used_bytes / redis_memory_max_bytes` | > 85% | Sắp đầy → tăng maxmemory hoặc scale |
| `rate(redis_evicted_keys_total[5m])` | > 10/s | Đang mất data |
| `redis_connected_clients / redis_config_maxclients` | > 90% | Connection exhaustion |
| `redis_connected_slaves` | < 1 | Mất replica → không có HA |
| `redis_up` | == 0 | Instance down |

**Log quan trọng:**
```bash
# Redis log levels: debug, verbose, notice, warning
loglevel notice    # production default

# Các dòng cần chú ý:
# WARNING: 32 bit instance detected...    ← sai platform
# MASTER <-> REPLICA sync started         ← replication event
# Connection with replica lost            ← replica disconnected
# Can't save in background: fork: Cannot allocate memory ← thiếu RAM cho BGSAVE
# WARNING overcommit_memory is set to 0   ← OS config cần fix
```

**Restart/rollback procedure:**
```bash
# Graceful restart (giữ data nếu persistence enabled)
redis-cli -a $PASS BGSAVE && redis-cli -a $PASS SHUTDOWN SAVE

# Kubernetes: rolling restart StatefulSet
kubectl rollout restart statefulset/redis-master -n redis

# Nếu data corrupt — restore từ RDB:
# 1. Stop Redis
# 2. Replace dump.rdb
# 3. Start Redis (load RDB on startup)

# Force Sentinel failover (maintenance):
redis-cli -p 26379 SENTINEL failover mymaster
```

---

## 9. Gotchas & Lessons Learned

- **`KEYS *` trong production**: Block event loop. Với 5M keys → hang ~500ms. Mọi command khác bị queue lại. Dùng `SCAN` với cursor.
- **maxmemory không set**: Redis crash server vì OOM killer. Luôn set maxmemory = 75-80% RAM instance. Phần còn lại cho AOF rewrite buffer, connection overhead, fragmentation.
- **K8s: limits.memory ≤ maxmemory**: Redis bị OOMKilled trước khi eviction kịp chạy. Rule: `limits.memory = maxmemory × 1.2`.
- **Sentinel và client library không support**: Một số ORM/framework chỉ support standalone Redis. Trước khi deploy Sentinel, verify client library support. Không phải tất cả `redis-py`, `ioredis`, `Lettuce` đều handle failover transparent.
- **Cache stampede**: Nhiều keys expire cùng lúc (ví dụ batch-set với cùng TTL) → cache miss đồng loạt → thundering herd vào DB. Fix: jitter TTL (`base_ttl + random(0, base_ttl * 0.1)`).
- **Replica read và stale data**: Replication async → replica có thể lag vài trăm ms. Đọc user's own write từ replica → stale. Pattern: write xong → đọc từ master trong cùng request, sau đó mới dùng replica cho reads tiếp theo.

---

## 10. Resources

- [Redis Documentation](https://redis.io/docs/) — official, section "Commands" rất đầy đủ
- [Redis in Action (book)](https://www.manning.com/books/redis-in-action) — use cases thực tế
- [Bitnami Redis Helm Chart](https://github.com/bitnami/charts/tree/main/bitnami/redis) — K8s deployment
- [oliver006/redis_exporter](https://github.com/oliver006/redis_exporter) — Prometheus metrics
- [Redis University](https://university.redis.com/) — free courses, RU101 (Data Structures) là điểm bắt đầu tốt
