## Transparent Proxy — Full Flow

```
[Client]──────►[Router/Gateway]──────►[Squid]──────►[Internet]
10.0.0.10      10.0.0.1               10.0.0.2
               (iptables redirect)    port 3128/3129
```

Traffic flow thực tế:

```
1. Client gửi request đến google.com:80
2. Packet đến Gateway (default route của client)
3. iptables PREROUTING intercept → redirect sang Squid:3128
4. Squid forward ra internet
5. Response về Squid → về Client
   (Client nghĩ đang nói chuyện với google.com)
```

### Phần 1 — Squid config (trên Squid VM)

```squid
# /etc/squid/squid.conf

# Transparent mode — KHÔNG dùng http_port thường
http_port 3128 intercept
https_port 3129 intercept ssl-bump \
    cert=/etc/squid/ssl/ca.crt \
    key=/etc/squid/ssl/ca.key

# ACL cơ bản
acl localnet src 10.0.0.0/24
http_access allow localnet
http_access deny all

# SSL bump
ssl_bump bump all
```

> ⚠️ Keyword `intercept` là bắt buộc — thiếu cái này Squid từ chối nhận redirected traffic.

### Phần 2 — Router/Gateway config (iptables)

Đây là phần Nghia hỏi. Có **2 scenario**:
##### Scenario A: Squid chạy TRÊN CHÍNH Router/Gateway

```bash
# Squid và Gateway là 1 máy
# Redirect traffic từ clients sang Squid local

iptables -t nat -A PREROUTING \
  -i eth0 \                        # interface nhận traffic từ LAN
  -p tcp --dport 80 \
  -j REDIRECT --to-port 3128       # redirect local port

iptables -t nat -A PREROUTING \
  -i eth0 \
  -p tcp --dport 443 \
  -j REDIRECT --to-port 3129
```

```
[Client] ──eth0──► [Gateway+Squid] ──eth1──► [Internet]
                   iptables REDIRECT
                   (loop lại local)
```

##### Scenario B: Squid là máy riêng biệt ⭐ (production-like hơn)

```bash
# Trên Gateway — DNAT traffic sang Squid VM
iptables -t nat -A PREROUTING \
  -i eth0 \
  -p tcp --dport 80 \
  -j DNAT --to-destination 10.0.0.2:3128    # IP của Squid VM

iptables -t nat -A PREROUTING \
  -i eth0 \
  -p tcp --dport 443 \
  -j DNAT --to-destination 10.0.0.2:3129

# Cho phép forward packet
iptables -A FORWARD -d 10.0.0.2 -p tcp --dport 3128 -j ACCEPT
iptables -A FORWARD -d 10.0.0.2 -p tcp --dport 3129 -j ACCEPT

# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward
# Permanent:
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
```

---

#### Phần 3 — Squid VM cần thêm 1 rule

Khi dùng Scenario B, Squid nhận packet có **destination IP là google.com** (không phải IP của nó) → cần NAT masquerade để response về đúng:

bash

```bash
# Trên Squid VM
iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
```

---

### Routing Table — Không cần thêm gì đặc biệt

```
Client default route: 10.0.0.1 (Gateway)  ← giữ nguyên
Gateway routing table:
  10.0.0.0/24 → local (LAN)
  0.0.0.0/0   → ISP   (WAN)

Không cần thêm static route vì iptables xử lý
trước khi routing decision được đưa ra (PREROUTING chain)
```

```
Packet lifecycle:
PREROUTING → ROUTING DECISION → FORWARD/INPUT → POSTROUTING
    ▲
    └── iptables intercept ở đây, trước khi route
```

---

### Tóm lại

| |Scenario A|Scenario B|
|---|---|---|
|Squid location|Trên Gateway|VM riêng|
|iptables rule|`REDIRECT`|`DNAT`|
|Production fit|Dev/lab|✅ Production|
|Complexity|Thấp|Medium|