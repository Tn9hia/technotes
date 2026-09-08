---
title: DoH & DoT — Encrypted DNS
tags:
  - dns
  - doh
  - dot
  - security
  - deep-dive
date: 2026-04-26
---
 
# DoH & DoT — DNS-over-HTTPS & DNS-over-TLS

## Vấn đề DNS truyền thống

DNS query gửi plain text qua UDP port 53:
- **ISP có thể đọc**: biết mày đang truy cập site nào
- **MITM**: attacker trên same network thay đổi DNS response
- **Censorship**: ISP/firewall block domain bằng cách drop DNS response
- **DNS leakage**: VPN user vẫn bị lộ DNS query nếu không cẩn thận

DNSSEC giải quyết authentication nhưng không encrypt. DoT/DoH giải quyết cả hai.

---

## So sánh DoT vs DoH

| | DoT (DNS-over-TLS) | DoH (DNS-over-HTTPS) |
|---|---|---|
| **Port** | 853 (TCP) | 443 (TCP) |
| **Protocol** | TLS + DNS wire format | HTTPS + DNS trong HTTP body |
| **RFC** | RFC 7858 | RFC 8484 |
| **Nhận diện** | Dễ — port 853 riêng biệt | Khó — trộn với HTTPS traffic |
| **Block** | Dễ — block port 853 | Khó — phải DPI |
| **Overhead** | TLS handshake | TLS + HTTP overhead |
| **Latency** | Thấp hơn sau handshake | Cao hơn chút |
| **Client support** | Android 9+, Linux (systemd-resolved) | Browser (Chrome, Firefox), curl |
| **Enterprise control** | Dễ kiểm soát | Khó — browser bypass system DNS |

**DoQ (DNS-over-QUIC)** — RFC 9250 — mới hơn, dùng QUIC thay TCP, latency thấp nhất. Chưa phổ biến.

---

## DNS wire format trong DoH

DoH gửi DNS query dưới dạng binary (DNS wire format) encode base64url trong HTTP GET, hoặc binary trong HTTP POST:

```http
# GET request
GET /dns-query?dns=AAABAAABAAAAAAAAA3d3dwdleGFtcGxlA2NvbQAAAQAB HTTP/2
Host: dns.cloudflare.com
Accept: application/dns-message

# POST request
POST /dns-query HTTP/2
Host: dns.cloudflare.com
Content-Type: application/dns-message
Content-Length: 33

<binary DNS query>
```

Response:
```http
HTTP/2 200 OK
Content-Type: application/dns-message
Cache-Control: max-age=300

<binary DNS response>
```

---

## dnsdist — DoH/DoT Termination

dnsdist đứng trước làm TLS termination, forward về backend bằng plain DNS UDP/TCP.

```
[Client DoT :853]  ──┐
[Client DoH :443]  ──┤  dnsdist  ──→  [pdns-recursor :5300 UDP]
[Client DNS :53]   ──┘
```

### Config DoT

```lua
-- /etc/dnsdist/dnsdist.conf

-- Plain DNS
setLocal("0.0.0.0:53")

-- DoT — port 853
addTLSLocal("0.0.0.0:853",
  "/etc/ssl/certs/dns.example.com.pem",
  "/etc/ssl/private/dns.example.com.key",
  {
    minTLSVersion="tls1.2",
    ciphers="ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256",
    numberOfTicketsKeys=5   -- TLS session tickets
  }
)

-- Backend
newServer({address="127.0.0.1:5300", name="recursor"})

-- ACL
setACL({"0.0.0.0/0", "::/0"})   -- public resolver: allow all
-- Hoặc restrict cho internal
-- setACL({"10.0.0.0/8", "172.16.0.0/12"})
```

### Config DoH

```lua
-- DoH — port 443
-- Cần thêm package: dnsdist-lua-records hoặc build với --enable-dns-over-https

addDOHLocal("0.0.0.0:443",
  "/etc/ssl/certs/dns.example.com.pem",
  "/etc/ssl/private/dns.example.com.key",
  "/dns-query",     -- URL path
  {
    minTLSVersion="tls1.2",
    -- HTTP/2 tự động nếu OpenSSL hỗ trợ
    customResponseHeaders={
      ["access-control-allow-origin"]="*",
      ["x-content-type-options"]="nosniff"
    }
  }
)
```

### Config kết hợp cả 3

```lua
-- dnsdist.conf đầy đủ

-- Plain DNS (internal)
setLocal("0.0.0.0:53")

-- DoT (public hoặc internal)
addTLSLocal("0.0.0.0:853",
  "/etc/letsencrypt/live/dns.example.com/fullchain.pem",
  "/etc/letsencrypt/live/dns.example.com/privkey.pem"
)

-- DoH (public)
addDOHLocal("0.0.0.0:443",
  "/etc/letsencrypt/live/dns.example.com/fullchain.pem",
  "/etc/letsencrypt/live/dns.example.com/privkey.pem",
  "/dns-query"
)

-- Backends
newServer({address="127.0.0.1:5300", name="recursor-1"})
newServer({address="127.0.0.1:5301", name="recursor-2"})

-- Policy
setServerPolicy(roundrobin)

-- Rate limiting
addAction(MaxQPSIPRule(100), DropAction())

-- ACL cho plain DNS (internal only)
-- DoH/DoT thường allow all (public resolver)
setACL({"0.0.0.0/0", "::/0"})
```

---

## Client config

### Linux — systemd-resolved (DoT)

```ini
# /etc/systemd/resolved.conf
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com
DNS=8.8.8.8#dns.google
DNSOverTLS=yes
# opportunistic = thử DoT, fallback plain nếu fail
# yes = bắt buộc DoT, fail nếu không được
DNSSEC=yes
```

```bash
systemctl restart systemd-resolved
resolvectl status
# Protocol: DNS, DNSOverTLS
```

### Linux — resolv.conf + stubby (DoT daemon)

```yaml
# /etc/stubby/stubby.yml
resolution_type: GETDNS_RESOLUTION_STUB
dns_transport_list:
  - GETDNS_TRANSPORT_TLS
tls_authentication: GETDNS_AUTHENTICATION_REQUIRED
tls_query_padding_blocksize: 128
idle_timeout: 10000
listen_addresses:
  - 127.0.0.1@53000
upstream_recursive_servers:
  - address_data: 1.1.1.1
    tls_auth_name: "cloudflare-dns.com"
  - address_data: 8.8.8.8
    tls_auth_name: "dns.google"
```

### curl — DoH

```bash
# curl hỗ trợ DoH native từ 7.62.0
curl --doh-url https://cloudflare-dns.com/dns-query https://example.com

# Hoặc dùng internal DoH server
curl --doh-url https://dns.internal.com/dns-query https://app.internal.com
```

### Firefox — DoH

```
about:preferences → Network Settings → Enable DNS over HTTPS
Provider: Custom → https://dns.internal.com/dns-query
```

### Chrome / Chromium — DoH

```
chrome://settings/security → Use secure DNS
Custom: https://dns.internal.com/dns-query
```

**Vấn đề với DoH trong browser:** Browser bypass system DNS → enterprise không kiểm soát được DNS filtering. Giải pháp: block DoH providers (1.1.1.1:443, 8.8.8.8:443) bằng firewall và cung cấp internal DoH endpoint.

---

## Internal DoH/DoT — Use case On-Premise

**Scenario:** Môi trường corporate, muốn:
1. Encrypt DNS traffic từ workstation → DNS server
2. Vẫn kiểm soát được DNS filtering
3. Ngăn browser dùng DoH external (Cloudflare, Google)

```
[Workstation]
    │  DoH: https://dns.internal.com/dns-query
    ▼
[dnsdist :443 — TLS termination]
    │  plain UDP :5300
    ▼
[pdns-recursor]
    │  forward internal zones → auth
    │  external zones → upstream (có thể filter ở đây)
    ▼
[Internet hoặc upstream resolver]
```

**Block external DoH providers:**
```bash
# Firewall rule: block workstation kết nối đến DoH providers
iptables -I FORWARD -d 1.1.1.1 -p tcp --dport 443 -j REJECT
iptables -I FORWARD -d 8.8.8.8 -p tcp --dport 443 -j REJECT
# + DNS-based block: trả NXDOMAIN cho cloudflare-dns.com, dns.google
```

---

## Cert management cho DoT/DoH

### Let's Encrypt (public resolver)

```bash
# Certbot
certbot certonly --standalone -d dns.example.com

# Config dnsdist dùng cert
addTLSLocal("0.0.0.0:853",
  "/etc/letsencrypt/live/dns.example.com/fullchain.pem",
  "/etc/letsencrypt/live/dns.example.com/privkey.pem"
)

# Auto reload khi cert renew
# /etc/letsencrypt/renewal-hooks/deploy/dnsdist-reload.sh
#!/bin/bash
echo "server.reloadTLSCertificates()" | dnsdist --client
```

### Internal CA (private resolver)

```bash
# Tạo internal cert (dùng easy-rsa hoặc openssl)
openssl req -x509 -newkey ecdsa -pkeyopt ec_paramgen_curve:P-256 \
  -keyout dns.internal.com.key \
  -out dns.internal.com.crt \
  -days 365 \
  -subj "/CN=dns.internal.com"

# Client phải trust internal CA cert
# Linux: copy vào /usr/local/share/ca-certificates/ → update-ca-certificates
# Windows: import vào Trusted Root CA store
```

---

## Verify DoT/DoH hoạt động

```bash
# Test DoT
kdig -d @dns.example.com +tls example.com A
# hoặc
openssl s_client -connect dns.example.com:853 -servername dns.example.com

# Test DoH bằng curl
curl -H "accept: application/dns-json" \
  "https://cloudflare-dns.com/dns-query?name=example.com&type=A"

# Test DoH với binary format
echo -n '\x00\x00\x01\x00\x00\x01\x00\x00\x00\x00\x00\x00' | \
  curl -s -X POST https://dns.internal.com/dns-query \
  -H "Content-Type: application/dns-message" \
  --data-binary @- | xxd | head

# dnsdist stats: xem encrypted vs plain traffic
dnsdist --client
> showServers()
> showTLSContexts()
```

---

## Gotchas

- **DoH và HTTP/2**: dnsdist cần build với `--enable-dns-over-https` và OpenSSL hỗ trợ ALPN. Package distro thường đã có sẵn.
- **TLS cert cho internal resolver**: client cần trust CA → deploy CA cert via Ansible/GPO trước khi bật DoT/DoH bắt buộc
- **DoH bypass enterprise policy**: Firefox `about:config` → `network.trr.mode=0` để disable DoH theo network policy (`canary domain` trick)
- **Session resumption**: bật TLS session tickets (default trong dnsdist) → giảm latency sau kết nối đầu tiên
- **Port 443 conflict**: nếu server đang chạy nginx/webserver trên 443 → conflict. Giải pháp: dnsdist dùng port khác (8443) hoặc dùng SNI routing để tách
