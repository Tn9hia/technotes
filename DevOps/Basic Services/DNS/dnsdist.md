---
title: dnsdist — DNS Load Balancer & Firewall
tags:
  - dns
  - dnsdist
  - deep-dive
date: 2026-04-26
---

# dnsdist — DNS Load Balancer & Firewall

## dnsdist là gì và tại sao cần

dnsdist là **DNS traffic director** — đứng trước tất cả DNS backend, xử lý:
- **Load balancing**: phân phối query qua nhiều Recursor/Authoritative
- **ACL & Firewall**: drop/block query theo source IP, query type, domain
- **Rate limiting**: chống DDoS, DNS amplification
- **DoH/DoT termination**: TLS endpoint cho encrypted DNS
- **Query routing**: forward query khác nhau đến backend khác nhau
- **Observability**: metrics, logging, tracing

> dnsdist không resolve DNS — nó chỉ route và policy. Backend thực sự resolve là pdns-recursor hoặc authoritative.

---

## Architecture

```
                    ┌─────────────────────────────────┐
DNS :53 ────────────►                                 │
DoT :853 ───────────►        dnsdist                  ├──► recursor-1 :5300
DoH :443 ───────────►    (Lua rules engine)           ├──► recursor-2 :5300
                    │                                 ├──► auth :5301
                    └─────────────────────────────────┘
                              │
                    metrics, console, API
```

---

## Lua config — ngôn ngữ cấu hình

dnsdist dùng **Lua** cho toàn bộ config. Không phải config file truyền thống.

**Cấu trúc logic:**

```lua
-- 1. Setup listeners
setLocal("0.0.0.0:53")

-- 2. Define backends
newServer({...})

-- 3. Add rules (top-down, first match wins)
addAction(SomeRule(), SomeAction())
addAction(SomeRule(), SomeAction())

-- 4. Default: pass to backend (nếu không rule nào match)
```

---

## Listeners

```lua
-- Plain DNS UDP+TCP
setLocal("0.0.0.0:53")
setLocal("0.0.0.0:53", {reusePort=true})  -- SO_REUSEPORT cho multi-thread

-- Chỉ TCP
addLocal("0.0.0.0:53", {tcpFastOpen=true})

-- DoT
addTLSLocal("0.0.0.0:853",
  "/etc/ssl/dns.crt",
  "/etc/ssl/dns.key",
  {
    minTLSVersion="tls1.2",
    numberOfTicketsKeys=5,
    sessionTimeout=300
  }
)

-- DoH
addDOHLocal("0.0.0.0:443",
  "/etc/ssl/dns.crt",
  "/etc/ssl/dns.key",
  "/dns-query",
  {minTLSVersion="tls1.2"}
)
```

---

## Backends (Servers)

```lua
-- Backend đơn giản
newServer("127.0.0.1:5300")

-- Backend với options
newServer({
  address="127.0.0.1:5300",
  name="recursor-1",        -- tên hiển thị trong stats
  qps=1000,                 -- max queries/sec gửi đến backend này
  order=1,                  -- priority (thấp hơn = ưu tiên hơn)
  weight=10,                -- weight cho load balancing
  checkName=".",            -- health check: query "." A
  checkType="A",
  maxCheckFailures=3,       -- số lần fail trước khi mark down
  rise=2,                   -- số lần pass để mark up lại
  tcpConnectTimeout=2,
  tcpSendTimeout=2,
  tcpRecvTimeout=2,
  useClientSubnet=true,     -- pass EDNS client subnet
})

-- Multiple backends
newServer({address="10.0.1.10:53", name="recursor-dc1", pool="recursive"})
newServer({address="10.0.1.11:53", name="recursor-dc2", pool="recursive"})
newServer({address="10.0.2.10:53", name="auth-dc1",     pool="authoritative"})
```

---

## Load Balancing Policies

```lua
-- Round Robin (default)
setServerPolicy(roundrobin)

-- Least Outstanding Queries (tốt nhất cho production)
setServerPolicy(leastOutstanding)

-- Chased Round Robin (weighted round-robin)
setServerPolicy(wrandom)

-- Fastest response time
setServerPolicy(firstAvailable)

-- Per-pool policy
setPoolServerPolicy(leastOutstanding, "recursive")
setPoolServerPolicy(roundrobin, "authoritative")
```

---

## Rules & Actions — Tim của dnsdist

**Rule** = điều kiện (match hay không). **Action** = làm gì nếu match.

### Rules phổ biến

```lua
-- Match theo source IP
NetmaskGroupRule({"10.0.0.0/8", "192.168.0.0/16"})
NotRule(NetmaskGroupRule({"10.0.0.0/8"}))  -- NOT

-- Match theo query name
QNameRule("blocked.example.com.")
QNameSuffixRule("example.com.")   -- match cả subdomain

-- Match theo query type
QTypeRule(DNSQType.A)
QTypeRule(DNSQType.AAAA)
QTypeRule(DNSQType.ANY)

-- Match theo OPCODE
OpcodeRule(DNSOpcode.Query)

-- Rate limiting
MaxQPSIPRule(100)           -- > 100 qps từ cùng IP
MaxQPSRule(10000)           -- > 10000 qps tổng cộng

-- Kết hợp rules (AND)
AndRule({
  NetmaskGroupRule({"10.0.0.0/8"}),
  QTypeRule(DNSQType.ANY)
})
```

### Actions phổ biến

```lua
-- Drop (không trả lời gì)
DropAction()

-- Trả REFUSED
RCodeAction(DNSRCode.REFUSED)

-- Trả NXDOMAIN
RCodeAction(DNSRCode.NXDOMAIN)

-- Trả NOERROR với empty answer (lie)
NoRecurseAction()

-- Forward đến pool cụ thể
PoolAction("authoritative")

-- Forward đến server cụ thể
ToPoolAction("internal-auth")

-- Truncate (force TCP retry)
TCAction()

-- Log query
LogAction("/var/log/dnsdist/queries.log")

-- Thêm ECS (EDNS Client Subnet)
ECSAction(24, 56)   -- prefix length IPv4, IPv6

-- Trả SERVFAIL
SpoofAction("0.0.0.0")  -- trả IP giả
```

### Ví dụ thực tế

```lua
-- 1. Block known malware domains
local malwareDomains = newDNSNameSet()
malwareDomains:add("malware.example.com.")
malwareDomains:add("phishing.bad.com.")
addAction(QNameSetRule(malwareDomains), RCodeAction(DNSRCode.NXDOMAIN))

-- 2. Rate limit: chống DDoS
addAction(MaxQPSIPRule(100), DropAction())
addAction(MaxQPSRule(50000), DropAction())

-- 3. Block ANY queries (amplification vector)
addAction(QTypeRule(DNSQType.ANY), TCAction())  -- force TCP

-- 4. Internal queries → internal pool
addAction(
  AndRule({
    NetmaskGroupRule({"10.0.0.0/8"}),
    QNameSuffixRule("internal.com.")
  }),
  PoolAction("authoritative")
)

-- 5. External queries → recursive pool
addAction(AllRule(), PoolAction("recursive"))
```

---

## ACL

```lua
-- Chỉ accept query từ internal networks
setACL({"10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16", "127.0.0.0/8"})

-- Public resolver: accept tất cả
setACL({"0.0.0.0/0", "::/0"})

-- Xem ACL hiện tại trong console
> showACL()

-- Thêm IP vào ACL runtime (không lưu)
> addACL("203.0.113.0/24")
```

---

## Health Check

```lua
newServer({
  address="127.0.0.1:5300",
  name="recursor",

  -- Query health check
  checkName=".",           -- query name để check
  checkType="A",           -- query type
  checkInterval=2,         -- check mỗi 2 giây
  maxCheckFailures=3,      -- mark down sau 3 lần fail
  rise=2,                  -- cần 2 lần pass để mark up
  checkTimeout=1000,       -- timeout 1000ms

  -- TCP health check
  tcpConnectTimeout=2,
  tcpSendTimeout=2,
  tcpRecvTimeout=2,
})
```

```lua
-- Xem trạng thái backend
> showServers()
-- Output:
-- #   Name            Address         State  Qps  Qlim  Ord  Wt  Queries  Drops
-- 0   recursor-1      127.0.0.1:5300  up     123  0     1    1   456789   0
-- 1   recursor-2      127.0.0.1:5301  up     98   0     1    1   345678   0
```

---

## Console

dnsdist có interactive Lua console để debug và quản lý runtime.

```bash
# Kết nối console
dnsdist --client

# Hoặc qua TCP (nếu config controlSocket)
# dnsdist.conf:
# controlSocket("127.0.0.1:5199")
# setKey(makeKey())  -- generate key: dnsdist --gen-key

dnsdist --client 127.0.0.1:5199
```

**Lệnh hay dùng trong console:**

```lua
-- Stats tổng quan
> showServers()
> showRules()
> showACL()

-- Top clients theo query count
> topClients(10)

-- Top domains được query nhiều nhất
> topQueries(20)

-- Top SERVFAIL
> topServFails(10)

-- Cache stats
> showCaches()

-- Xem queries real-time
> grepq("example.com")     -- filter theo domain
> grepq("192.168.1.1")     -- filter theo source IP
> grepq("", 1)              -- 1 giây vừa qua

-- Reload TLS cert không restart
> server.reloadTLSCertificates()

-- Thêm rule runtime
> addAction(QNameRule("block.me."), DropAction())

-- Xem metrics
> getStatisticsCounters()
```

---

## Metrics & Monitoring

```lua
-- dnsdist.conf: expose webserver cho metrics
webserver("0.0.0.0:8083")
setWebserverConfig({
  password="secret",
  apiKey="api-secret",
  acl="127.0.0.1/8, 10.0.0.0/8"
})
```

```bash
# JSON stats endpoint
curl -u secret: http://localhost:8083/

# Prometheus metrics endpoint
curl http://localhost:8083/metrics
```

**Metrics quan trọng:**

| Metric | Ý nghĩa | Alert khi |
|--------|---------|-----------|
| `queries` | Tổng query received | - |
| `servfail-responses` | SERVFAIL trả về | > 1% |
| `downstream-timeouts` | Backend timeout | > 0.1% |
| `rule-drop` | Query bị drop bởi rule | Tăng đột biến |
| `latency-avg100` | Avg latency 100 queries | > 50ms |
| `cache-hits` | Cache hit (nếu dùng cache) | < 30% |

---

## Query Logging & Tracing

```lua
-- Log tất cả queries
addAction(AllRule(), LogAction("/var/log/dnsdist/all.log", false, true))
-- false = không log early response, true = append

-- Log chỉ queries bị drop
addAction(MaxQPSIPRule(100), LogAction("/var/log/dnsdist/ratelimited.log"))
addAction(MaxQPSIPRule(100), DropAction())

-- Protobuf logging (cho analytics)
rl = newRemoteLogger("127.0.0.1:4242")
addAction(AllRule(), RemoteLogAction(rl))
```

---

## Lua scripting nâng cao

```lua
-- Custom function để block dynamically
local blockedIPs = newNetmaskGroup()

function blockIP(ip)
  blockedIPs:addMask(ip .. "/32")
end

addAction(NetmaskGroupRule(blockedIPs), DropAction())

-- Block IP động từ console:
-- > blockIP("1.2.3.4")

-- Custom response dựa trên query
addAction(QNameRule("time.internal.com."), SpoofAction("10.0.0.1"))

-- Lua function làm action
function customAction(dq)
  if dq.qname:toString() == "test.internal.com." then
    return DNSAction.Spoof, "1.2.3.4"
  end
  return DNSAction.None
end
addAction(AllRule(), LuaAction(customAction))
```

---

## Full config ví dụ (production)

```lua
-- /etc/dnsdist/dnsdist.conf

-- Listeners
setLocal("0.0.0.0:53", {reusePort=true})
addTLSLocal("0.0.0.0:853",
  "/etc/letsencrypt/live/dns.example.com/fullchain.pem",
  "/etc/letsencrypt/live/dns.example.com/privkey.pem",
  {minTLSVersion="tls1.2"}
)
addDOHLocal("0.0.0.0:443",
  "/etc/letsencrypt/live/dns.example.com/fullchain.pem",
  "/etc/letsencrypt/live/dns.example.com/privkey.pem",
  "/dns-query", {minTLSVersion="tls1.2"}
)

-- ACL
setACL({"10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16", "127.0.0.0/8"})

-- Backends
newServer({address="127.0.0.1:5300", name="recursor-1",
  pool="recursive", checkName=".", maxCheckFailures=3})
newServer({address="10.0.0.2:5300", name="recursor-2",
  pool="recursive", checkName=".", maxCheckFailures=3})
newServer({address="127.0.0.1:5301", name="auth",
  pool="authoritative"})

setPoolServerPolicy(leastOutstanding, "recursive")

-- Security rules
addAction(MaxQPSIPRule(200), DropAction())
addAction(QTypeRule(DNSQType.ANY), TCAction())

-- Internal zones → authoritative
addAction(
  QNameSuffixRule("internal.com."),
  PoolAction("authoritative")
)
addAction(
  QNameSuffixRule("10.in-addr.arpa."),
  PoolAction("authoritative")
)

-- Default → recursive
addAction(AllRule(), PoolAction("recursive"))

-- Monitoring
webserver("127.0.0.1:8083")
setWebserverConfig({password="changeme", apiKey="changeme"})
controlSocket("127.0.0.1:5199")
```

---

## Gotchas

- **Rule order**: first match wins — đặt specific rule trước general rule
- **Backend down**: nếu tất cả backend trong pool down → dnsdist trả SERVFAIL, không fallback sang pool khác (trừ khi config `useClientSubnet` + fallback policy)
- **Thread count**: mặc định 1 thread — tăng `setMaxTCPClientThreads()` và `setNumWorkerThreads()` cho production
- **DoH và HTTP/2**: cần OpenSSL 1.1+ và build flag `--enable-dns-over-https`
- **Console key**: phải generate key (`makeKey()`) và sync giữa config và client nếu dùng TCP console
- **Cache**: dnsdist có built-in DNS cache (`addCacheHitResponseAction`) nhưng thường để recursor cache — tránh double-cache
