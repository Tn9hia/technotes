---
title: Squid Proxy
tags:
  - proxy
  - infra
  - basic-services
  - GlobalTechJSC
date: 2026-04-19
status: in-progress
---

# Squid Proxy — 7.x

Tags: #proxy #infra #basic-services
Last updated: 2026-04-19

---

## 1. What — Nó là cái gì?

Squid là một **caching proxy server** mã nguồn mở, hỗ trợ HTTP, HTTPS, FTP. Nó đứng giữa client và internet (hoặc upstream server), có thể cache response, kiểm soát access, và log traffic.

> Squid không phải web server, không phải load balancer thuần túy — nó là proxy với khả năng cache và access control mạnh.

---

## 2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?

Nếu không có Squid (hoặc proxy tương tự):

- Không có điểm kiểm soát tập trung cho outbound traffic của toàn bộ hệ thống
- Không cache → mỗi request đều hit upstream → tốn bandwidth, tăng latency
- Không audit được "ai đang gọi cái gì ra ngoài internet"
- Không thể whitelist/blacklist domain ở tầng infra

**Use case thực tế trong On-Premise:**

- Internal clients cần ra internet nhưng phải đi qua proxy (compliance requirement)
- Cache package repository (apt, yum) để tiết kiệm bandwidth
- Transparent proxy để intercept traffic mà không cần config từng client
- Upstream pool + health check thay thế một phần chức năng của load balancer cho outbound

---

## 3. When — Dùng khi nào / KHÔNG dùng khi nào?

**Dùng khi:**
- Cần kiểm soát và audit outbound HTTP/HTTPS traffic
- Muốn cache content (repo mirror, static assets)
- Môi trường air-gap / restricted internet access
- Cần forward proxy cho internal services gọi external API

**KHÔNG dùng khi:**
- Cần reverse proxy thuần túy cho inbound traffic → dùng Nginx/HAProxy
- Cần load balancing phức tạp với health check, session persistence → dùng HAProxy
- Traffic là TCP non-HTTP (SSH, database) → Squid không handle
- Cần TLS termination cho inbound → Squid không phải tool đúng

> [!warning] Squid vs Reverse Proxy
> Squid **có thể** làm reverse proxy nhưng đây không phải điểm mạnh. Dùng Nginx/Traefik nếu inbound reverse proxy là use case chính.

---

## 4. Architecture — Nó nằm ở đâu trong hệ thống?

### Forward Proxy (use case chính)

```
[Internal Clients]
       |
       | HTTP CONNECT / plain HTTP
       v
  [Squid Proxy :3128]
       |
       | → cache hit: trả về ngay
       | → cache miss: forward request
       v
  [Internet / Upstream]
```

### Transparent Proxy

```
[Internal Clients]
       |
       | (không biết có proxy — iptables redirect)
       v
  [iptables REDIRECT → :3128]
       v
  [Squid Proxy]
       v
  [Internet]
```

### Reverse Proxy (ít dùng)

```
[External Clients]
       |
       v
  [Squid :80/:443]
       |
       v
  [Internal Backend Servers]
```

### Upstream Pool (cache_peer)

```
[Squid]
   |
   |── cache_peer primary.upstream.com (parent, port 3128)
   |── cache_peer secondary.upstream.com (parent, port 3128)
   |
   └── never_direct allow all
```

---

## 5. How — Cơ chế hoạt động

### 5.1 Request Flow (Forward Proxy)

1. Client gửi request đến Squid (configured `http_proxy=squid:3128`)
2. Squid kiểm tra **ACL** → allow/deny
3. Nếu allowed → kiểm tra **cache**:
   - Cache hit: trả về từ disk/memory
   - Cache miss: forward lên upstream, lưu cache nếu cacheable
4. Log vào access.log

### 5.2 HTTPS / CONNECT Method

Client gửi `CONNECT target.com:443 HTTP/1.1` → Squid mở TCP tunnel đến target → Squid không thấy nội dung (encrypted). Để inspect HTTPS phải dùng **SSL Bump** (man-in-the-middle) — cần cẩn thận về legal/privacy.

### 5.3 ACL System

ACL là core của Squid. ACL định nghĩa **điều kiện**, `http_access` quyết định **allow/deny**:

```
acl localnet src 10.0.0.0/8          # điều kiện
http_access allow localnet           # action
http_access deny all                 # default deny cuối cùng
```

Thứ tự rule: **first match wins** — giống iptables.

### 5.4 Cache Mechanism

- **Memory cache** (`cache_mem`): hot objects
- **Disk cache** (`cache_dir`): persistent, survive restart
- Cache key = URL (+ Vary header nếu có)
- `cache_peer` để chain proxy hoặc tạo upstream pool

### 5.5 Transparent Proxy

Squid không thấy destination address bằng cách thông thường → cần kernel hỗ trợ qua `iptables` + `REDIRECT` hoặc `TPROXY`:

```bash
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 3128
```

Squid phải dùng `http_port 3128 intercept` mode.

---

## 6. Key Config — Cấu hình cần nhớ

### squid.conf cơ bản

```nginx
# Port lắng nghe
http_port 3128

# ACL network nội bộ
acl localnet src 10.0.0.0/8
acl localnet src 172.16.0.0/12
acl localnet src 192.168.0.0/16

# ACL port được phép
acl SSL_ports port 443
acl Safe_ports port 80 443 8080 21 70 210 1025-65535
acl CONNECT method CONNECT

# Chặn port nguy hiểm
http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports

# Allow internal
http_access allow localnet
http_access deny all   # LUÔN có dòng này ở cuối

# Cache
cache_mem 256 MB
cache_dir ufs /var/spool/squid 10000 16 256
maximum_object_size 100 MB

# Log
access_log /var/log/squid/access.log squid
cache_log /var/log/squid/cache.log

# DNS
dns_nameservers 8.8.8.8 1.1.1.1
```

### Upstream Pool (cache_peer)

```nginx
# Thêm upstream peer
cache_peer proxy1.internal.com parent 3128 0 no-query default
cache_peer proxy2.internal.com parent 3128 0 no-query

# Không kết nối trực tiếp, phải qua peer
never_direct allow all

# Load balance: round-robin hoặc weighted
```

### Transparent Proxy mode

```nginx
http_port 3128 intercept
# Kết hợp với iptables REDIRECT
```

### ACL nâng cao

```nginx
# Whitelist domain
acl whitelist_domains dstdomain .github.com .npmjs.com .debian.org
http_access allow localnet whitelist_domains
http_access deny localnet   # block tất cả ngoài whitelist

# Blacklist
acl blocked_sites dstdomain .facebook.com .tiktok.com
http_access deny blocked_sites

# Time-based ACL
acl working_hours time MTWHF 08:00-18:00
http_access allow localnet working_hours
```

### Logging format

```nginx
# Default format đủ dùng
access_log /var/log/squid/access.log squid

# Custom format nếu cần
logformat custom %tl %>a %Ss/%03>Hs %<st %rm %ru %[un %Sh/%<a %mt
```

> [!tip] Config hay bị sai
> - Quên `http_access deny all` ở cuối → **open proxy**
> - Không có ACL cho `CONNECT` method → client không HTTPS được
> - `cache_dir` không đủ permission cho squid user → squid crash khi start

### Client configuration to use squid
#### Cách 1: Per-session (tạm thời)

```bash
export http_proxy="http://proxy-ip:3128"
export https_proxy="http://proxy-ip:3128"
export no_proxy="localhost,127.0.0.1,10.0.0.0/8"
```

Mất khi close terminal.

---

#### Cách 2: Persistent cho user

```bash
# ~/.bashrc hoặc ~/.bash_profile
export http_proxy="http://proxy-ip:3128"
export https_proxy="http://proxy-ip:3128"
export no_proxy="localhost,127.0.0.1"
```

Then run
```bash
source ~/.bashrc
```

---

#### Cách 3: System-wide (recommended)

```bash
# /etc/environment
http_proxy="http://proxy-ip:3128"
https_proxy="http://proxy-ip:3128"
no_proxy="localhost,127.0.0.1,10.0.0.0/8"
```

Hoặc file riêng cho profile:

```bash
# /etc/profile.d/proxy.sh
export http_proxy="http://proxy-ip:3128"
export https_proxy="http://proxy-ip:3128"
export no_proxy="localhost,127.0.0.1"
```

---

#### DNF/YUM cần config riêng 


```bash
# /etc/dnf/dnf.conf — thêm vào cuối
proxy=http://proxy-ip:3128
```

Vì DNF không đọc environment variable như các tool khác.

---

#### Có auth không?

```bash
http_proxy="http://username:password@proxy-ip:3128"
```

---

## 7. Security Considerations

### Attack surface

- **Open proxy**: nếu thiếu ACL → bất kỳ ai cũng dùng được proxy của bạn để ẩn danh/spam
- **SSRF via proxy**: internal services bị exposed nếu ACL không chặn `localhost`, `169.254.x.x`
- **Cache poisoning**: attacker inject malicious content vào cache
- **SSL Bump**: nếu bật → bạn đang MITM traffic của user, cần policy rõ ràng

### Hardening checklist tối thiểu

- [ ] `http_access deny all` là rule cuối cùng — bắt buộc
- [ ] Chặn access đến localhost và link-local từ proxy:
  ```nginx
  acl to_localhost dst 127.0.0.0/8 ::1
  acl to_linklocal dst 169.254.0.0/16
  http_access deny to_localhost
  http_access deny to_linklocal
  ```
- [ ] Không expose port 3128 ra internet
- [ ] Dùng `via off` + `forwarded_for delete` để ẩn internal IP:
  ```nginx
  via off
  forwarded_for delete
  request_header_access X-Forwarded-For deny all
  ```
- [ ] Giới hạn `cache_peer` chỉ trusted upstream
- [ ] Rotate log định kỳ, không để log đầy disk

---

## 8. Ops Runbook — Production Notes

### Health check

```bash
# Check squid process
systemctl status squid

# Test proxy hoạt động
curl -x http://squid-host:3128 http://example.com -I

# Check cache stats
squidclient -h localhost mgr:info
squidclient -h localhost mgr:counters

# Check active connections
squidclient -h localhost mgr:active_requests
```

### Log quan trọng cần monitor

| File | Nội dung | Alert khi |
|------|----------|-----------|
| `access.log` | Mọi request | Error rate tăng đột biến |
| `cache.log` | Cache events, errors | `FATAL` / `ERROR` xuất hiện |

**Parse access.log:**
```bash
# Top domains được access nhiều nhất
awk '{print $7}' /var/log/squid/access.log | cut -d/ -f3 | sort | uniq -c | sort -rn | head 20

# Request bị deny
grep "TCP_DENIED" /var/log/squid/access.log | tail -50

# Cache hit ratio
awk '{print $4}' /var/log/squid/access.log | sort | uniq -c
```

### Metrics cần alert

- Cache hit ratio < 30% (nếu dùng để cache)
- `5xx` response rate tăng
- Disk usage của `cache_dir` > 85%
- Memory usage bất thường (leak)

### Restart / Reload

```bash
# Reload config không restart (graceful)
squid -k reconfigure

# Full restart
systemctl restart squid

# Kiểm tra config trước khi apply
squid -k parse

# Clear cache (cẩn thận trên production)
squid -k shutdown && rm -rf /var/spool/squid/* && squid -z && systemctl start squid
```

---

## 9. Gotchas & Lessons Learned

> Phần này điền thêm khi có kinh nghiệm thực tế.

- **`dns_nameservers` phải set rõ** — mặc định Squid dùng `/etc/resolv.conf` của host, có thể bị ảnh hưởng nếu host DNS bị lỗi
- **Transparent proxy với HTTPS**: `CONNECT` tunnel đi qua fine, nhưng không inspect được nội dung — nếu cần filter HTTPS phải dùng SSL Bump + deploy CA cert cho toàn bộ client
- **`cache_peer` với `no-query`**: bỏ ICP query (thường không cần) — nếu upstream không hỗ trợ ICP mà thiếu flag này sẽ có lỗi
- **Worker mode** (`workers N`): Squid SMP mode cải thiện performance trên multi-core nhưng cần test kỹ vì một số feature không tương thích hoàn toàn

---

## 10. Resources

- [Squid Official Docs](http://www.squid-cache.org/Doc/)
- [Squid GitHub](https://github.com/squid-cache/squid)
- [squid.conf directives reference](http://www.squid-cache.org/Doc/config/)
- [Squid ACL guide (thực tế)](https://wiki.squid-cache.org/SquidFaq/SquidAcl)
- [Transparent Proxy setup](https://wiki.squid-cache.org/Features/Tproxy4)
