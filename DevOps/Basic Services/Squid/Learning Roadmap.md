### Squid Forward Proxy — Learning Roadmap

```
Level 1 → Level 2 → Level 3 → Level 4
[Core]   [Access]  [HTTPS]   [Production]
```

---

### 🗺️ Roadmap Chi Tiết

#### Level 1 — Core Concepts (nắm chắc trước)

- [ ]  Squid hoạt động như thế nào (request flow)
- [ ]  Cài đặt, cấu trúc file config `/etc/squid/squid.conf`
- [ ]  HTTP vs HTTPS proxy khác nhau chỗ nào
- [ ]  Các port cơ bản: `3128` (http), `3129` (ssl-bump)
- [ ]  Cache hoạt động ra sao: `cache_dir`, `cache_mem`
- [ ]  Log format: `access.log`, `cache.log`

bash

```bash
# Verify squid chạy đúng
squid -k parse        # check config syntax
squid -NCd1           # debug mode
tail -f /var/log/squid/access.log
```

---

#### Level 2 — ACL & Access Control ⭐ (quan trọng nhất)

ACL là trái tim của Squid. Phải nắm chắc.

squid

```squid
# ACL types cần biết
acl localnet src 10.0.0.0/8          # by source IP
acl blocked_domains dstdomain .facebook.com .tiktok.com
acl work_hours time MTWHF 08:00-18:00
acl SSL_ports port 443
acl managers src 192.168.1.10        # specific user IP

# Logic rules
http_access deny blocked_domains
http_access allow managers           # managers bypass giờ hành chính
http_access allow localnet work_hours
http_access deny all                 # default deny — LUÔN có dòng này
```

> ⚠️ **Rule order matters** — Squid đọc từ trên xuống, match đầu tiên thắng.

---

#### Level 3 — HTTPS Interception (SSL Bump)

Đây là phần tricky nhất, nhiều người bỏ qua rồi production fail.

```
Normal HTTPS:
Client ──CONNECT──► Squid ──tunnel──► Server
       (Squid blind, chỉ tunnel, không thấy nội dung)

SSL Bump:
Client ──CONNECT──► Squid ──decrypt──► inspect ──► re-encrypt──► Server
       (Squid thấy hết, MitM hợp lệ với CA nội bộ)
```

Cần nắm:

- [ ]  Tạo CA cert nội bộ, deploy lên clients
- [ ]  Config `ssl_bump` modes: `peek`, `stare`, `bump`, `splice`
- [ ]  `bump` vs `splice` — khi nào bump, khi nào để nguyên (banking sites, etc.)
- [ ]  Certificate error handling

squid

```squid
# ssl-bump config cơ bản
http_port 3128 ssl-bump \
  cert=/etc/squid/ssl/ca.crt \
  key=/etc/squid/ssl/ca.key

acl no_bump dstdomain .banking.com .gov.vn
ssl_bump splice no_bump       # không inspect banking
ssl_bump bump all             # inspect phần còn lại
```

---

#### Level 4 — Production Hardening

- [ ]  **Authentication**: Basic Auth, LDAP/AD integration
- [ ]  **Transparent proxy**: redirect traffic bằng `iptables` — không cần config client
- [ ]  **Cache tuning**: `maximum_object_size`, `cache_replacement_policy`
- [ ]  **Monitoring**: parse access.log → gửi vào Grafana/ELK
- [ ]  **High availability**: 2 Squid node + keepalived VIP
- [ ]  **Performance**: worker processes, `workers N`

squid

```squid
# Transparent proxy — client không cần biết proxy tồn tại
http_port 3128 intercept
https_port 3129 intercept ssl-bump cert=... key=...
```

bash

```bash
# iptables redirect traffic về Squid
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 3128
iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port 3129
```

---

### Priority Order

```
Week 1: Level 1 + Level 2 (ACL)
Week 2: Level 3 (SSL Bump) — lab kỹ, dễ break
Week 3: Level 4 (Production stuff)
```

---

### Lab Setup Gợi Ý

```
[Client VM] ──► [Squid VM] ──► [Internet]
 10.0.0.10       10.0.0.1
```