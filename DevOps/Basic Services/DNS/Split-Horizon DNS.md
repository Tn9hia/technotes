---
title: Split-Horizon DNS
tags:
  - dns
  - split-horizon
  - deep-dive
date: 2026-04-26
---

# Split-Horizon DNS (Split-View DNS)

## Khái niệm

**Split-horizon DNS** = cùng một domain name nhưng trả lời khác nhau tùy theo nguồn query đến từ đâu.

```
app.example.com query từ internal  →  10.0.1.50   (private IP)
app.example.com query từ internet  →  203.0.113.10 (public IP)
```

**Tại sao cần:**
- Internal client kết nối qua LAN → dùng private IP → nhanh hơn, không tốn NAT
- External client kết nối qua internet → dùng public IP
- Không lộ internal IP ra internet
- Security: internal service không cần public DNS record

---

## Các pattern implement

### Pattern 1: Recursor + forward-zones (PowerDNS — đơn giản nhất)

Recursor forward internal zone về authoritative nội bộ, external zone recursive ra ngoài.

```
[Client Internal]
      │
      ▼
[pdns-recursor]
      │
      ├── query "app.internal.com"? → forward-zones → [auth internal] → 10.0.1.50
      │
      └── query "google.com"? → recursive lookup từ root → 142.250.x.x
```

**Config recursor:**

```ini
# /etc/pdns-recursor/recursor.conf

# Internal zones → authoritative nội bộ
forward-zones=internal.com=127.0.0.1:5301
forward-zones+=corp.example.com=127.0.0.1:5301
forward-zones+=10.in-addr.arpa=127.0.0.1:5301
forward-zones+=168.192.in-addr.arpa=127.0.0.1:5301

# External: tự recursive từ root (không set gì thêm)
# Hoặc forward ra upstream:
# forward-zones-recurse=.=8.8.8.8;1.1.1.1
```

**Đây là cách PowerDNS recommend** — đơn giản, rõ ràng, dễ maintain.

---

### Pattern 2: Hai authoritative — internal và external

Hai server authoritative riêng biệt, mỗi cái phục vụ một "view" khác nhau.

```
[Internal Client] → [Internal Recursor] → [Auth Internal]
                                           app.example.com = 10.0.1.50

[External Client] → [Public DNS] → [Auth External / Public NS]
                                    app.example.com = 203.0.113.10
```

**Setup:**
- Auth internal: PowerDNS với zone `example.com` chứa private IPs
- Auth external: PowerDNS (hoặc Cloudflare, Route53) với zone `example.com` chứa public IPs
- Internal recursor: `forward-zones=example.com=<auth-internal-IP>`
- Client internal: DNS trỏ về internal recursor
- Client external: DNS trỏ về public NS

**Ưu điểm:** Tách biệt hoàn toàn, dễ quản lý khi zone lớn.
**Nhược điểm:** Phải maintain 2 zone song song → dễ out-of-sync.

---

### Pattern 3: BIND9 Views (nếu dùng BIND)

BIND9 có native support với `view`:

```bind
// named.conf
acl "internal" { 10.0.0.0/8; 172.16.0.0/12; };

view "internal" {
    match-clients { internal; };
    zone "example.com" {
        type master;
        file "/etc/bind/internal/example.com.zone";
    };
};

view "external" {
    match-clients { any; };
    zone "example.com" {
        type master;
        file "/etc/bind/external/example.com.zone";
    };
};
```

> PowerDNS không có Views như BIND9. PowerDNS dùng approach khác: multiple backends hoặc Lua scripting.

---

### Pattern 4: PowerDNS Lua records (dynamic response)

PowerDNS Authoritative có thể trả lời dynamic dựa trên source IP của query (khi recursor pass ECS):

```lua
-- /etc/powerdns/zones/example.com.lua

-- Trả IP khác nhau tùy source
function app_record(dq)
  local src = dq.remoteaddr:toStringWithPort()
  
  -- Internal range
  if newNetmaskGroup():addMask("10.0.0.0/8") and 
     newNetmaskGroup():match(dq.remoteaddr) then
    return {{"10.0.1.50", 300}}
  end
  
  -- Default: public IP
  return {{"203.0.113.10", 300}}
end

-- Zone config trong pdns.conf:
-- launch=lua2
-- lua2-script=/etc/powerdns/zones/example.com.lua
```

**Thực tế ít dùng pattern này** — phức tạp, khó debug.

---

## Setup thực tế — Pattern 1 (recommended)

### Scenario

```
Domain: example.com
Internal zone: corp.example.com (private)
External zone: example.com (public, quản lý bởi Cloudflare)

Internal hosts:
  app.corp.example.com → 10.0.1.50
  db.corp.example.com  → 10.0.2.10
  
Public:
  www.example.com → 203.0.113.10
```

### Step 1: PowerDNS Authoritative cho internal zone

```ini
# /etc/powerdns/pdns.conf
local-port=5301
local-address=127.0.0.1
launch=gmysql
# ... db config ...
```

```bash
# Tạo zone internal
pdnsutil create-zone corp.example.com ns1.corp.example.com
pdnsutil add-record corp.example.com @ NS ns1.corp.example.com
pdnsutil add-record corp.example.com ns1 A 10.0.0.1
pdnsutil add-record corp.example.com app A 10.0.1.50
pdnsutil add-record corp.example.com db A 10.0.2.10
```

### Step 2: Recursor với forward-zones

```ini
# /etc/pdns-recursor/recursor.conf
local-port=5300
local-address=127.0.0.1

# Forward internal zone → auth nội bộ
forward-zones=corp.example.com=127.0.0.1:5301
forward-zones+=10.in-addr.arpa=127.0.0.1:5301

# External zones: recursive từ root (mặc định)
# Không cần config thêm

allow-from=10.0.0.0/8, 127.0.0.0/8
```

### Step 3: dnsdist routing

```lua
-- /etc/dnsdist/dnsdist.conf
setLocal("0.0.0.0:53")
setACL({"10.0.0.0/8", "127.0.0.0/8"})

-- Backends
newServer({address="127.0.0.1:5300", name="recursor", pool="default"})

-- Tất cả query → recursor (recursor tự biết forward internal)
addAction(AllRule(), PoolAction("default"))
```

### Step 4: Client config

```bash
# Tất cả internal host trỏ DNS về dnsdist
# /etc/resolv.conf hoặc DHCP option 6
nameserver 10.0.0.1   # dnsdist IP

# Test
dig app.corp.example.com @10.0.0.1
# → 10.0.1.50 (từ internal auth)

dig www.example.com @10.0.0.1
# → 203.0.113.10 (resolved từ Cloudflare qua internet)
```

---

## Edge cases phức tạp

### 1. Cùng domain, khác subdomain: public + private

```
example.com → public (Cloudflare quản lý)
internal.example.com → private (PowerDNS nội bộ)
```

Config recursor:
```ini
# Chỉ forward subdomain nội bộ, không phải toàn bộ example.com
forward-zones=internal.example.com=127.0.0.1:5301
# example.com vẫn recursive ra Cloudflare
```

### 2. Overriding public record cho internal client

Scenario: `api.example.com` tồn tại trên Cloudflare (public: `203.0.113.20`), nhưng internal client phải dùng private IP `10.0.1.20`.

```ini
# Recursor: forward TOÀN BỘ example.com về auth nội bộ
forward-zones=example.com=127.0.0.1:5301
```

```bash
# Auth nội bộ có zone example.com với:
api.example.com → 10.0.1.20  (override)
www.example.com → 203.0.113.10 (giống public)
# ... những record khác cũng phải có đầy đủ
```

> [!warning] Maintenance burden
> Khi dùng approach này, mọi thay đổi DNS public phải được sync thủ công về internal zone. Dễ bị out-of-sync. Chỉ dùng khi thực sự cần override.

### 3. Split-horizon với Docker/Kubernetes

Container/Pod thường dùng CoreDNS, không phải host DNS. Cần cấu hình CoreDNS forward internal zones.

```yaml
# CoreDNS ConfigMap trong K8s
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        
        # Forward internal zones về PowerDNS nội bộ
        corp.example.com:53 {
            forward . 10.0.0.1:53
            cache 30
        }
        
        # Forward internal.com về PowerDNS
        internal.com:53 {
            forward . 10.0.0.1:53
            cache 30
        }
        
        # Tất cả còn lại
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

### 4. Split-horizon với VPN

Client kết nối VPN → nhận DNS server nội bộ → resolve internal domain.
Client không VPN → dùng public DNS → chỉ thấy public records.

```
VPN client:
  DNS pushed by VPN: 10.0.0.1 (dnsdist nội bộ)
  app.corp.example.com → 10.0.1.50 ✓

Non-VPN client:
  DNS: 8.8.8.8
  app.corp.example.com → NXDOMAIN (không có trên public DNS)
```

Config VPN (ví dụ WireGuard):
```ini
[Interface]
DNS = 10.0.0.1  # push DNS về client

[Peer]
# ...
```

---

## Debugging split-horizon

```bash
# Test từ internal (phải ra IP private)
dig app.corp.example.com @10.0.0.1
# Expected: 10.0.1.50

# Test từ external (phải ra IP public hoặc NXDOMAIN)
dig app.corp.example.com @8.8.8.8
# Expected: NXDOMAIN hoặc public IP nếu có

# Trace full path từ recursor
dig +trace app.corp.example.com @10.0.0.1

# Xem recursor đang forward zone nào
rec_control get-all | grep forward

# Flush cache recursor để test
rec_control wipe-cache corp.example.com
dig app.corp.example.com @10.0.0.1

# Kiểm tra zone trên authoritative
dig app.corp.example.com @127.0.0.1 -p 5301
```

---

## Common mistakes

- **Quên reverse zone**: `10.in-addr.arpa` cũng phải forward về internal auth → không thì PTR lookup fail
- **TTL quá cao**: nếu client cache IP public của `app.example.com` lâu, sau khi connect VPN vẫn resolve ra public IP → kết nối fail hoặc đi ra ngoài internet thay vì LAN
- **forward-zones vs forward-zones-recurse**: 
  - `forward-zones`: forward và **trust** response (không validate DNSSEC)  
  - `forward-zones-recurse`: forward nhưng vẫn validate DNSSEC từ upstream
  - Với internal zone không có DNSSEC → dùng `forward-zones`
- **CoreDNS trong K8s**: Pod DNS mặc định là CoreDNS cluster → phải config CoreDNS forward, không phải chỉ config node DNS
