---
title: DNS - PowerDNS
tags:
  - dns
  - infra
  - basic-services
  - GlobalTechJSC
date: 2026-04-19
status: in-progress
---

# DNS — PowerDNS (Authoritative + Recursor + dnsdist)

Tags: #dns #infra #basic-services
Last updated: 2026-04-19

---

## 1. What — Nó là cái gì?

**DNS (Domain Name System)** là hệ thống phân giải tên miền thành địa chỉ IP — "danh bạ điện thoại" của internet. Không có DNS, mọi kết nối phải dùng IP thủ công.

**PowerDNS** là bộ DNS software gồm 3 component độc lập:
- **Authoritative Server** (`pdns`): trả lời authoritative answer cho zone mình quản lý
- **Recursor** (`pdns-recursor`): recursive resolver — đi hỏi từ root đến authoritative để trả lời client
- **dnsdist**: DNS load balancer / firewall / policy engine — đứng trước Recursor hoặc Authoritative

> PowerDNS khác BIND9 ở chỗ: lưu zone data trong **database** (MySQL, PostgreSQL, SQLite) thay vì flat file → dễ quản lý programmatically, dễ scale.

---

## 2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?

Nếu không có internal DNS server:
- Mọi hostname nội bộ phải dùng IP tĩnh → brittle, khó maintain
- Không có split-horizon → internal service bị resolve ra IP public (sai)
- Không kiểm soát được DNS query của nội bộ (ai đang query gì)
- Phụ thuộc hoàn toàn vào DNS public (8.8.8.8) → single point of failure

**PowerDNS so với BIND9:**

| | BIND9 | PowerDNS |
|---|---|---|
| Zone storage | Flat file | Database (MySQL/PG/SQLite) |
| API | Không có native | REST API built-in |
| Split-horizon | Views (phức tạp) | Multiple backends |
| Scale | Khó | Dễ (DB replication) |
| Performance | Tốt | Tốt hơn ở high query rate |

---

## 3. When — Dùng khi nào / KHÔNG dùng khi nào?

**Dùng khi:**
- Cần internal DNS cho infrastructure (service discovery, hostname resolution)
- Cần split-horizon DNS (internal vs external view khác nhau)
- Quản lý nhiều zone, cần API hoặc automation
- Cần DNS load balancing / rate limiting (dnsdist)
- Air-gap / on-premise environment

**KHÔNG dùng khi:**
- Chỉ cần DNS đơn giản cho 1-2 host → `/etc/hosts` đủ rồi
- Public authoritative DNS với traffic rất lớn → xem xét thêm Anycast routing
- Cần DNSSEC phức tạp mà team chưa có kinh nghiệm → complexity cao

---

## 4. Architecture — Nó nằm ở đâu trong hệ thống?

### DNS Resolution Flow

```
[Client]
   |
   | 1. Query: "what is app.internal.com?"
   v
[dnsdist :53]  ← DNS firewall / load balancer
   |
   | 2. Forward to recursor
   v
[pdns-recursor :5300]
   |
   |── 3a. Internal zone? → forward to Authoritative
   |── 3b. External? → recursive lookup từ root
   v
[pdns Authoritative :5301]  ← quản lý internal zones
   |
   | 4. Đọc từ database
   v
[MySQL / PostgreSQL]
```

### Split-horizon DNS

```
[Internal Client]                [External Client]
       |                                |
       v                                v
[dnsdist]                        [Public DNS]
       |                                |
       v                                v
[Recursor]                      [Authoritative Public]
       |                           app.example.com → 203.x.x.x
       |── internal zone
       v
[Authoritative Internal]
   app.example.com → 10.0.1.50   ← internal IP
```

### Component Roles

```
dnsdist          → Rate limiting, ACL, DoH/DoT termination, routing
pdns-recursor    → Recursive resolution (như 8.8.8.8)
pdns (auth)      → Authoritative cho zone mình quản lý (như ns1.example.com)
```

---

## 5. How — Cơ chế hoạt động

### 5.1 DNS Resolution Flow

**Recursive (client góc nhìn):**
1. Client hỏi Recursor: `app.internal.com?`
2. Recursor check cache → miss
3. Recursor hỏi Root NS: `.com` ở đâu?
4. Root → `.com` NS
5. `.com` NS → `internal.com` NS (authoritative)
6. Authoritative → `10.0.1.50`
7. Recursor cache + trả về client

**Iterative (server tự đi hỏi từng bước, không nhờ server khác đi thay):** Đây là cách Recursor thực sự hoạt động — nó tự đi qua từng bước thay cho client.

### 5.2 Record Types

| Record | Mục đích | Ví dụ |
|--------|----------|-------|
| **A** | Domain → IPv4 | `app.internal.com → 10.0.1.50` |
| **AAAA** | Domain → IPv6 | `app → 2001:db8::1` |
| **CNAME** | Alias đến domain khác | `www → app.internal.com` |
| **MX** | Mail server | `@ → mail.internal.com` (priority 10) |
| **TXT** | Arbitrary text | SPF, DKIM, domain verification |
| **PTR** | IP → Domain (reverse) | `50.1.0.10.in-addr.arpa → app.internal.com` |
| **NS** | Name server cho zone | `internal.com → ns1.internal.com` |
| **SRV** | Service location | `_http._tcp → host:port` |
| **SOA** | Zone authority info | Serial, refresh, retry, expire |

> [!tip] CNAME gotcha
> CNAME không thể đứng ở **apex** của zone (root domain). `example.com CNAME other.com` là **invalid**. Dùng `ALIAS` record (PowerDNS hỗ trợ) hoặc flatten CNAME.

### 5.3 TTL & Caching Strategy

- **TTL** = thời gian client/resolver được cache record
- TTL thấp (60-300s): thay đổi nhanh nhưng tốn query, tăng load
- TTL cao (3600-86400s): ít query, nhưng propagation chậm khi thay đổi

**Rule of thumb:**
- Static infra records: TTL 3600 (1h)
- Services có thể thay IP: TTL 300 (5m)
- Trước khi migrate: hạ TTL xuống 60s trước 24h

### 5.4 PowerDNS Authoritative — Zone từ Database

PowerDNS đọc zone data từ DB thay vì file. Schema mặc định có bảng `domains` và `records`:

```sql
-- Tạo zone
INSERT INTO domains (name, type) VALUES ('internal.com', 'NATIVE');

-- Thêm record
INSERT INTO records (domain_id, name, type, content, ttl)
VALUES (1, 'app.internal.com', 'A', '10.0.1.50', 300);
```

Hoặc dùng **PowerDNS API** (REST) để quản lý programmatically.

### 5.5 DoH / DoT

- **DoT (DNS-over-TLS)**: DNS query wrapped trong TLS, port 853
- **DoH (DNS-over-HTTPS)**: DNS query qua HTTPS, port 443 — khó block hơn DoT

dnsdist xử lý DoH/DoT termination, forward về Recursor qua UDP/TCP thông thường:

```
[Client DoH] → [dnsdist :443 TLS termination] → [Recursor :5300 UDP]
```

---

## 6. Key Config — Cấu hình cần nhớ

### 6.1 dnsdist (`/etc/dnsdist/dnsdist.conf`)

```lua
-- Lắng nghe trên port 53
setLocal("0.0.0.0:53")

-- Backend: forward đến recursor
newServer({address="127.0.0.1:5300", name="recursor"})

-- ACL: chỉ allow internal subnet
setACL({"10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16", "127.0.0.1/8"})

-- Rate limiting: chống DNS amplification
addAction(MaxQPSIPRule(100), DropAction())

-- DoH endpoint (cần cert)
addDOHLocal("0.0.0.0:443", "/etc/ssl/cert.pem", "/etc/ssl/key.pem", "/dns-query")

-- Logging
addAction(AllRule(), LogAction("/var/log/dnsdist/queries.log", false, true))
```

### 6.2 pdns-recursor (`/etc/pdns-recursor/recursor.conf`)

```ini
# Port lắng nghe (tránh conflict với dnsdist)
local-port=5300
local-address=127.0.0.1

# Forward internal zones đến authoritative
forward-zones=internal.com=127.0.0.1:5301
forward-zones+=10.in-addr.arpa=127.0.0.1:5301

# Upstream resolvers cho external
# (để trống = tự recursive từ root, hoặc set forward-zones-recurse)
forward-zones-recurse=.=8.8.8.8;1.1.1.1

# Cache
max-cache-entries=1000000
max-cache-ttl=86400

# Security
dnssec=validate
allow-from=127.0.0.1/8, 10.0.0.0/8

# Logging
log-common-errors=yes
loglevel=3
```

### 6.3 PowerDNS Authoritative (`/etc/powerdns/pdns.conf`)

```ini
# Port (tránh conflict)
local-port=5301
local-address=127.0.0.1

# Backend: MySQL
launch=gmysql
gmysql-host=127.0.0.1
gmysql-dbname=pdns
gmysql-user=pdns
gmysql-password=secret

# API (để manage qua REST)
api=yes
api-key=your-secret-api-key
webserver=yes
webserver-address=127.0.0.1
webserver-port=8081

# DNSSEC
default-ksk-algorithm=ecdsa256

# Logging
log-dns-queries=yes
```

### 6.4 Zone file (nếu dùng BIND backend)

```zone
; internal.com zone
$ORIGIN internal.com.
$TTL 300

@   IN  SOA  ns1.internal.com. admin.internal.com. (
            2026041901  ; Serial (YYYYMMDDnn)
            3600        ; Refresh
            900         ; Retry
            604800      ; Expire
            300 )       ; Negative TTL

    IN  NS   ns1.internal.com.
    IN  NS   ns2.internal.com.

ns1 IN  A    10.0.0.1
ns2 IN  A    10.0.0.2
app IN  A    10.0.1.50
db  IN  A    10.0.2.10
www IN  CNAME app.internal.com.
```

### 6.5 DNS Tools

```bash
# dig — query cụ thể
dig app.internal.com A @127.0.0.1
dig -x 10.0.1.50 @127.0.0.1              # reverse lookup
dig internal.com NS @127.0.0.1           # query NS record
dig +trace app.internal.com              # trace full resolution path
dig +short app.internal.com              # chỉ lấy answer

# nslookup
nslookup app.internal.com 127.0.0.1

# PowerDNS API
curl -H "X-API-Key: your-secret-api-key" http://127.0.0.1:8081/api/v1/servers/localhost/zones

# Tạo zone qua API
curl -X POST -H "X-API-Key: secret" \
  -H "Content-Type: application/json" \
  -d '{"name":"internal.com.","kind":"Native","nameservers":["ns1.internal.com."]}' \
  http://127.0.0.1:8081/api/v1/servers/localhost/zones
```

> [!tip] Config hay bị sai
> - `forward-zones` trong recursor phải có dấu chấm cuối tên zone: `internal.com.` không phải `internal.com`
> - Serial trong SOA **phải tăng** mỗi khi thay đổi zone, nếu không slave không nhận update
> - `allow-from` trong recursor bị bỏ sót → open resolver → bị dùng cho DNS amplification attack

---

## 7. Security Considerations

### Attack surface

- **Open resolver**: Recursor không giới hạn `allow-from` → bị dùng cho DDoS amplification (query nhỏ, response lớn)
- **DNS cache poisoning**: attacker inject fake records vào cache (giảm thiểu bằng DNSSEC + randomize source port)
- **Zone transfer**: AXFR không giới hạn → lộ toàn bộ internal records
- **DNS tunneling**: exfiltrate data qua DNS query — khó detect

### Hardening checklist

- [ ] `allow-from` chỉ cho internal subnet — **bắt buộc**
- [ ] Tắt AXFR hoặc chỉ allow từ slave IP:
  ```ini
  # pdns.conf
  allow-axfr-ips=10.0.0.2/32   # chỉ slave
  ```
- [ ] Rate limiting qua dnsdist (`MaxQPSIPRule`)
- [ ] Bật DNSSEC validation ở Recursor (`dnssec=validate`)
- [ ] Không expose Authoritative ra internet trực tiếp (đứng sau dnsdist)
- [ ] API key đủ mạnh, webserver chỉ bind `127.0.0.1`
- [ ] Monitor bất thường: query rate đột biến, nhiều NXDOMAIN (subdomain brute-force)

---

## 8. Ops Runbook — Production Notes

### Health check

```bash
# Test resolution hoạt động
dig +short app.internal.com @127.0.0.1

# Check dnsdist stats
dnsdist -c   # console interactive
> showServers()
> showACL()

# Check recursor stats
rec_control get-all

# Check authoritative
pdns_control ping
pdns_control list

# Check zone loaded
pdns_control list-zones
```

### Log quan trọng

| Component | Log file | Alert khi |
|-----------|----------|-----------|
| dnsdist | `/var/log/dnsdist/` | Drop rate tăng |
| Recursor | syslog / `/var/log/pdns-recursor.log` | Timeout, SERVFAIL tăng |
| Authoritative | `/var/log/pdns/` | DB connection error |

```bash
# SERVFAIL nhiều → upstream hoặc DNSSEC issue
journalctl -u pdns-recursor | grep SERVFAIL

# Query rate per client
dnsdist -c
> topClients(10)

# Cache hit ratio của recursor
rec_control get cache-hits
rec_control get cache-misses
```

### Metrics cần alert

- SERVFAIL rate > 1%
- Query latency p95 > 100ms
- Cache hit ratio < 70% (nếu caching recursor)
- DB connection errors (Authoritative)

### Reload / Restart

```bash
# Reload zone không restart Authoritative
pdns_control reload

# Reload config Recursor
rec_control reload-lua-config

# Restart từng component
systemctl restart dnsdist
systemctl restart pdns-recursor
systemctl restart pdns

# Flush cache Recursor
rec_control wipe-cache .
```

---

## 9. Gotchas & Lessons Learned

> Phần này điền thêm khi có kinh nghiệm thực tế.

- **Thứ tự forward-zones trong Recursor**: match longest suffix first — `app.internal.com` match trước `internal.com`. Không cần worry về overlap.
- **DNSSEC + forward-zones**: khi dùng `forward-zones` mà upstream không support DNSSEC → validation fail → SERVFAIL. Phải dùng `forward-zones-recurse` hoặc tắt validation cho zone đó.
- **Port conflict**: cả 3 component mặc định đều muốn bind `:53`. Phải lên kế hoạch port trước: dnsdist `:53`, recursor `:5300`, auth `:5301`.
- **Serial SOA và automation**: nếu dùng API để update record mà quên bump serial → slave không sync. PowerDNS API **tự động bump serial** khi dùng `PATCH /zones/:id` — đây là lý do nên dùng API thay vì sửa DB trực tiếp.
- **Split-horizon với Docker/K8s**: container DNS thường point đến CoreDNS, cần cấu hình CoreDNS forward internal zones về PowerDNS Recursor.

---

## 10. Resources

- [PowerDNS Authoritative Docs](https://doc.powerdns.com/authoritative/)
- [PowerDNS Recursor Docs](https://doc.powerdns.com/recursor/)
- [dnsdist Docs](https://dnsdist.org/)
- [PowerDNS Admin (Web UI)](https://github.com/PowerDNS-Admin/PowerDNS-Admin)
- [DNS RFC cơ bản: RFC 1034, RFC 1035](https://datatracker.ietf.org/doc/html/rfc1034)
- [DNS-over-HTTPS: RFC 8484](https://datatracker.ietf.org/doc/html/rfc8484)
- [DNS tools cheatsheet — `dig` deep dive](https://jvns.ca/blog/2021/12/04/how-to-use-dig/)
