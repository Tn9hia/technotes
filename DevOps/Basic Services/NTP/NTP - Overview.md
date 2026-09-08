---
title: NTP - NTPsec & Chrony
tags:
  - ntp
  - infra
  - basic-services
  - GlobalTechJSC
date: 2026-04-19
status: in-progress
---

# NTP — NTPsec & Chrony

Tags: #ntp #infra #basic-services
Last updated: 2026-04-19

---

## 1. What — Nó là cái gì?

**NTP (Network Time Protocol)** là giao thức đồng bộ thời gian giữa các máy tính qua mạng. Mục tiêu: đảm bảo tất cả host trong hệ thống có cùng đồng hồ, sai lệch dưới mức chấp nhận được (thường < 1ms trong LAN).

Hai implementation phổ biến:
- **Chrony** (`chronyd`): lightweight, converge nhanh, phù hợp cho VM / container, **khuyến nghị dùng hiện nay**
- **NTPsec**: fork bảo mật của ntpd gốc, codebase nhỏ hơn, audit kỹ hơn — dùng khi cần NTP server public hoặc môi trường high-security

> Trên hầu hết Linux hiện đại (Ubuntu 20.04+, RHEL 8+), `chronyd` đã thay thế `ntpd` làm default. NTPsec là lựa chọn khi cần NTP server thực sự (stratum thấp, nhiều client).

---

## 2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?

Nếu đồng hồ các máy lệch nhau:

- **TLS/SSL cert validation fail**: cert có `notBefore` trong tương lai → connection bị reject
- **Log correlation bất khả thi**: sự kiện trên host A lúc `10:00:05` và host B lúc `10:00:00` thực ra xảy ra đồng thời, nhưng log thì không thể ghép
- **Kerberos authentication fail**: Kerberos yêu cầu lệch < 5 phút — nếu quá sẽ báo lỗi
- **Distributed system sai**: database replication conflict, distributed lock, event ordering đều phụ thuộc vào clock
- **JWT / session token expired sớm hoặc không expired**: server và client dùng clock khác nhau

---

## 3. When — Dùng khi nào / KHÔNG dùng khi nào?

**Chrony — dùng khi:**
- Client NTP trên server/VM (đồng bộ giờ từ upstream)
- NTP server nội bộ phục vụ LAN (vài trăm đến vài nghìn client)
- Môi trường VM thường bị suspend/resume (chrony converge nhanh hơn ntpd)
- Laptop / máy thường bị sleep

**NTPsec — dùng khi:**
- Cần NTP server độc lập, stratum thấp (stratum 1-2)
- Cần compatibility với ntpd ecosystem cũ
- Môi trường high-security cần codebase đã audit

**KHÔNG cần NTP daemon khi:**
- Cloud VM (AWS, GCP, Azure): thường có hardware clock sync riêng (PTP/VMware Tools) — NTP vẫn nên chạy nhưng không phải lo nhiều
- Container: kế thừa clock từ host, không cần chạy NTP trong container

---

## 4. Architecture — Nó nằm ở đâu trong hệ thống?

### Stratum Hierarchy

```
[Stratum 0 — Hardware Clock]
   GPS receiver, atomic clock, PPS signal
   (không phải NTP device, không giao tiếp qua network)
          |
          | reference
          v
[Stratum 1 — NTP Server]
   pool.ntp.org, time.google.com, time.cloudflare.com
   Kết nối trực tiếp với stratum 0
          |
          | NTP (UDP :123)
          v
[Stratum 2 — Internal NTP Server]  ← đây là cái mình triển khai
   ntp.internal.com (chrony hoặc NTPsec)
   Sync từ stratum 1, serve cho internal clients
          |
          | NTP (UDP :123)
          v
[Stratum 3 — Internal Clients]
   Tất cả server, VM, container trong hạ tầng
```

### Deployment Pattern (On-Premise)

```
[Internet]
    |
    | pool.ntp.org (stratum 1)
    v
[Internal NTP Server — chrony/NTPsec]
ntp1.internal.com  ntp2.internal.com  ← HA, 2 server
    |
    | broadcast hoặc unicast UDP :123
    v
[All Internal Hosts]
  server01, server02, ..., serverN
    |
    v
[VMs, Containers]
  kế thừa từ hypervisor host (VMware Tools / KVM)
```

---

## 5. How — Cơ chế hoạt động

### 5.1 Các khái niệm cốt lõi

**Stratum:**
- Số đo "khoảng cách" từ nguồn thời gian chuẩn
- Stratum 0: đồng hồ nguyên tử / GPS (không phải NTP)
- Stratum 1: sync trực tiếp từ stratum 0
- Stratum N: sync từ stratum N-1
- Stratum 16: unsynchronized (giá trị đặc biệt, nghĩa là "không tin được")

**Clock drift:**
- Đồng hồ phần cứng (CMOS clock) không chính xác tuyệt đối — tự lệch dần theo thời gian
- Chrony/NTPsec đo drift rate và **điều chỉnh dần** (`slew`) thay vì nhảy cóc (`step`)
- **Slew**: điều chỉnh từ từ (±0.5ms/s) → không bao giờ có thời gian đi lùi
- **Step**: nhảy ngay lập tức → an toàn khi lệch lớn (> 1s), nhưng nguy hiểm cho log

**Leap second:**
- Mỗi vài năm, UTC được thêm hoặc bớt 1 giây để bù Earth rotation
- NTP server broadcast thông báo trước
- Chrony handle tự động bằng `leapsectz right/UTC` hoặc smear (phân tán 1s lệch đều trong vài giờ)

**Offset:** sai lệch hiện tại giữa local clock và NTP source (mục tiêu → 0)

**Jitter:** độ biến động của offset qua các lần đo (thấp = tốt)

### 5.2 NTP Packet Exchange

```
Client                          Server
  |                               |
  |── T1: send request ──────────>|
  |                         T2: receive
  |                         T3: send response
  |<── T4: receive response ──────|

Offset = ((T2-T1) + (T3-T4)) / 2
Round-trip delay = (T4-T1) - (T3-T2)
```

Chrony chọn source có **jitter thấp nhất** và **stratum thấp nhất** làm reference chính.

### 5.3 Selection Algorithm

Chrony dùng thuật toán **Marzullo's algorithm** để chọn best source từ nhiều NTP server:
1. Loại bỏ outlier (falseticker)
2. Chọn intersection của các khoảng tin cậy
3. Prefer source có stratum thấp, jitter thấp, delay thấp

---

## 6. Key Config — Cấu hình cần nhớ

### 6.1 Chrony — Client config (`/etc/chrony.conf`)

```ini
# Upstream NTP servers (pool = multiple IPs)
pool pool.ntp.org iburst
server time.google.com iburst prefer
server time.cloudflare.com iburst

# Hoặc trỏ về internal NTP server
server ntp1.internal.com iburst prefer
server ntp2.internal.com iburst

# iburst: gửi 8 packet burst khi bắt đầu → đồng bộ nhanh hơn

# Cho phép step nếu lệch > 1s (chỉ trong 3 lần update đầu)
makestep 1.0 3

# Lưu drift để khởi động nhanh hơn
driftfile /var/lib/chrony/drift

# RTC (hardware clock) sync
rtcsync

# Log
logdir /var/log/chrony
log measurements statistics tracking
```

### 6.2 Chrony — NTP Server config (serve cho internal clients)

```ini
# Upstream
pool pool.ntp.org iburst
server time.google.com iburst

# Serve cho internal subnet
allow 10.0.0.0/8
allow 172.16.0.0/12
allow 192.168.0.0/16

# Local stratum (fallback nếu mất kết nối upstream)
# Tự coi mình là stratum 10, vẫn serve cho client
local stratum 10

makestep 1.0 3
driftfile /var/lib/chrony/drift
rtcsync

logdir /var/log/chrony
log measurements statistics tracking
```

### 6.3 NTPsec — Server config (`/etc/ntp.conf`)

```ini
# Upstream
pool pool.ntp.org iburst
server time.google.com iburst prefer

# Serve cho internal
restrict default kod limited nomodify notrap nopeer noquery
restrict 127.0.0.1
restrict ::1
restrict 10.0.0.0 mask 255.0.0.0 nomodify notrap

# === Clients trong internal subnet ===
# nomodify  : client không được thay đổi config server
# notrap    : disable remote event logging (attack surface)
# nopeer    : không tự động peer
# noquery   : block ntpq/ntpdc query từ subnet này (thêm nếu paranoid)
# limited   : Enforce rate limiting
# kod      : Kiss-o'-Death packet — rate limit clients vi phạm

# Local clock fallback
server 127.127.1.0
fudge 127.127.1.0 stratum 10

# Drift file
driftfile /var/lib/ntp/ntp.drift

# Logging
logfile /var/log/ntp.log
statsdir /var/log/ntpstats/
statistics loopstats peerstats clockstats
```

> [!tip] Config hay bị sai
> - Chrony: quên `allow` → server chạy nhưng client không query được
> - NTPsec: `restrict default ... noquery` → block cả monitoring tools như `ntpq`
> - `local stratum` quá thấp (vd: stratum 1) → client tin tưởng server nội bộ hơn cả GPS source — nguy hiểm
> - Quên `makestep` → khi VM bị suspend lâu, chrony slew rất chậm để bắt kịp → hàng giờ mới sync được

---

## 7. Security Considerations

### Attack surface

- **NTP amplification**: attacker gửi request nhỏ với source IP giả (victim) → server trả response lớn về victim → DDoS amplification factor ~10x
- **Time attack**: attacker làm lệch clock để bypass cert expiry hoặc Kerberos
- **monlist exploit** (ntpd cũ): command `monlist` trả về 600 client gần nhất → amplification factor 200x (đã fix trong NTPsec)

### Hardening checklist

- [ ] Chrony: `allow` chỉ internal subnet, không để `allow 0.0.0.0/0`
- [ ] NTPsec: `restrict default ... noquery nopeer` — chặn monlist và peer management từ ngoài
- [ ] Firewall: UDP 123 chỉ mở từ internal, không expose ra internet
- [ ] Dùng **NTP Pool** (`pool`) thay vì hardcode 1 server → tránh single point of failure
- [ ] Kiểm tra `chronyc tracking` thường xuyên — offset bất thường có thể là dấu hiệu bị attack
- [ ] Với NTPsec: bật **NTS (Network Time Security)** nếu upstream hỗ trợ (time.cloudflare.com hỗ trợ NTS):
  ```ini
  server time.cloudflare.com iburst nts
  ```

---

## 8. Ops Runbook — Production Notes

### Health check

```bash
# Chrony — trạng thái tổng quan
chronyc tracking

# Output quan trọng:
# Reference ID   : nguồn đang dùng
# Stratum        : stratum hiện tại
# System time    : offset so với NTP source
# RMS offset     : jitter
# Frequency      : drift rate (ppm)
# Leap status    : Normal

# Xem tất cả sources và chất lượng
chronyc sources -v

# Xem statistics từng source
chronyc sourcestats

# NTPsec
ntpq -pn          # list peers
ntpq -c rv        # system variables (offset, jitter)
ntpstat           # trạng thái nhanh

# Check port đang lắng nghe
ss -ulnp | grep 123
```

**Đọc output `chronyc sources`:**

```
MS Name/IP         Stratum Poll Reach LastRx Last sample
===============================================================================
^* time.google.com       1   6   377    42    -0.5ms[+0.1ms] +/-   12ms
^+ pool.ntp.org          2   7   377   108    +1.2ms[+1.8ms] +/-   25ms
^- ntp2.internal.com     2   6   377    15    +5.1ms[+5.1ms] +/-   30ms

M = mode: 
* = current sync source
+ = acceptable
- = rejected
? = unreachable

S = source: 
^ = server
= = peer
# = local reference
```

### Log & Metrics cần monitor

```bash
# Chrony log (nếu bật log)
tail -f /var/log/chrony/tracking.log

# Check systemd log
journalctl -u chronyd -f

# Offset hiện tại (script để monitor)
chronyc tracking | grep "System time" | awk '{print $4}'
```

| Metric | Normal | Alert |
|--------|--------|-------|
| System time offset | < 10ms | > 100ms |
| RMS offset | < 5ms | > 50ms |
| Stratum | 2-3 | ≥ 10 (local fallback) |
| Reach | 377 (octal) | < 177 (mất sync) |

### Reload / Restart

```bash
# Chrony — reload config
systemctl reload chronyd
# hoặc
chronyc reload sources

# Force sync ngay (cẩn thận: có thể step clock)
chronyc makestep

# Restart
systemctl restart chronyd

# NTPsec
systemctl restart ntpsec
ntpq -c "exit"   # test connection
```

### Troubleshoot đồng hồ lệch sau VM resume

```bash
# Check offset hiện tại
chronyc tracking | grep "System time"

# Force step nếu lệch lớn (> 1s)
chronyc makestep

# Nếu chrony không sync được (reach = 0)
chronyc sources        # kiểm tra upstream có reachable không
systemctl restart chronyd
```

---

## 9. Gotchas & Lessons Learned

> Phần này điền thêm khi có kinh nghiệm thực tế.

- **VM suspend/resume**: sau khi VM resume từ snapshot, clock bị lệch lớn. `makestep 1.0 3` chỉ áp dụng trong 3 lần update đầu sau start — nếu suspend sau đó, chrony sẽ **slew** chậm rãi. Giải pháp: thêm `makestep 0.1 -1` để step bất cứ khi nào lệch > 0.1s (không giới hạn lần).
- **`local stratum` và client trust**: nếu set `local stratum 1`, client có thể prefer internal server hơn pool.ntp.org stratum 1 thực sự — set ít nhất stratum 5-10.
- **Firewall block UDP 123**: hay bị quên trong môi trường có firewall nghiêm ngặt — chrony sẽ show `reach = 0` cho tất cả sources.
- **NTS và self-signed cert**: NTS cần cert hợp lệ. Không dùng self-signed cho NTS production.
- **Container và NTP**: không chạy NTP daemon trong container — container dùng clock của host. Đảm bảo host sync đúng là đủ.

---

## 10. Resources

- [Chrony Official Docs](https://chrony-project.org/documentation.html)
- [chrony.conf man page](https://chrony-project.org/doc/4.5/chrony.conf.html)
- [NTPsec Docs](https://www.ntpsec.org/documentation.html)
- [NTP Pool Project](https://www.ntppool.org/)
- [Cloudflare NTS Server](https://developers.cloudflare.com/time-services/nts/)
- [RFC 5905 — NTPv4](https://datatracker.ietf.org/doc/html/rfc5905)
- [Julia Evans — How NTP works](https://jvns.ca/blog/2016/11/21/how-do-you-set-up-a-new-server/)
