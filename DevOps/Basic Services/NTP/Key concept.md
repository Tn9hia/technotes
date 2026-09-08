## Stratum
![[Pasted image 20260426165122.png]]
**Key rules về Stratum:**

- Stratum càng thấp = càng gần nguồn thời gian thật = càng chính xác
- Mỗi hop (stratum++) thêm ~vài ms jitter
- **Cloud**: AWS/GCP/Azure đều cung cấp Stratum 2–3 endpoint internal. Mày point chrony vào đó, đừng ra public pool.
- Stratum 16 = node không sync được → coi là broken, alerting phải bắt cái này


## Clock drift
![[Pasted image 20260426164028.png]]


### Clock drift là gì?
Oscillator trong CPU/mainboard chạy không hoàn toàn chính xác. Sau 24h không sync, có thể lệch **vài giây đến vài phút** tùy hardware.

### Tại sao Step clock nguy hiểm?
Log timestamp bị lộn xộn, TLS cert validation fail, distributed locks bị race condition, Kafka/Cassandra replication lỗi.

### Slew vs Step
`Slew` — điều chỉnh chậm, không giật clock. Chrony dùng mặc định khi offset < 1s.  
`Step` — nhảy ngay lập tức. Chỉ dùng khi offset quá lớn (makestep config).

### Cloud VM đặc biệt hơn
VM bị suspend/resume → clock đứng yên trong khi wall-clock chạy → drift cực lớn khi resume. Chrony xử lý cái này tốt hơn ntpd nhiều.


## Leap Second 
Leap second xảy ra vì Trái Đất quay không đều → UTC lệch khỏi astronomical time → IERS thêm 1 giây vào cuối ngày (thường 30/6 hoặc 31/12)

![[Pasted image 20260426164800.png]]

```
Cloud Provider NTP (Stratum 2)
  │  e.g. 169.254.169.123 (AWS), metadata.google.internal (GCP)
  ▼
chrony (Stratum 3) — chạy trên mỗi VM/node
  │  leapsectz right/UTC  ← dùng leap smear từ provider
  │  makestep 1.0 3        ← chỉ step 3 lần đầu khởi động
  │  maxdistance 1.5       ← reject source quá xa
  ▼
App / Kubernetes pods (inherit từ host)
```


## Clock Offset
Khoảng lệch giữa local clock và NTP server. Chrony dùng cái này để quyết định slew hay step.
```shell
chronyc tracking
# → RMS offset: 0.000123 s
```

Nếu offset > 1s liên tục → xem lại upstream hoặc VM bị suspend

## Jitter
Độ dao động của offset qua nhiều lần đo. Jitter cao = network không ổn định hoặc server upstream bị load.

```shell
chronyc sourcestats
# → Std Dev (jitter) column
```

Jitter > 10ms trên LAN → nghi ngờ network congestion hoặc NTP server overload

## Root Dispersion
Sai số tích lũy tối đa từ stratum 0 xuống đến node hiện tại. Càng nhiều hop → dispersion càng lớn.

```shell
chronyc tracking
# → Root dispersion: 0.001s
```
Root dispersion + Root delay = System Error bound thực tế

## Clock Selection & Falseticker
NTP dùng thuật toán **Intersection Algorithm** (Marzullo's algo) để chọn tập server "đồng thuận". Server nào nằm ngoài intersection bị gọi là **falseticker** — bị loại.
```shell
chronyc sources
# * = selected  + = acceptable
# ? = falseticker (bị loại)
# x = không dùng
```
**Dùng ít nhất 4 server** để algo phát hiện được falseticker. 3 server → nếu 1 sai, không biết thằng nào đúng.

## Best Source Selection (Peer Distance)

Sau khi loại falseticker, chrony chọn source có **peer distance** nhỏ nhất = (root delay / 2) + root dispersion. Không phải chỉ chọn stratum thấp nhất.

```shell
chronyc sourcestats -v
# Score column = peer distance
# Thấp hơn = được ưu tiên hơn
```

2 server cùng stratum → thằng nào latency thấp hơn sẽ được chọn

## NTP Amplification DDoS

Attacker spoof IP nạn nhân → gửi monlist request đến NTP server → server trả response lớn hơn 100× về nạn nhân.
```shell
# Disable monlist (ntpd):
restrict default noquery
# Chrony: monlist disabled by default ✓
```
**Không expose NTP server ra public** nếu không cần thiết

## NTP Spoofing / MITM

Attacker inject gói UDP giả → shift clock của victim → làm hỏng TLS cert validation, token expiry, log timestamp.
```shell
# Dùng NTS (NTP over TLS):
server time.cloudflare.com iburst nts
# Hoặc symmetric key:
keyfile /etc/chrony.keys
```

Môi trường nhạy cảm → bật **NTS** (RFC 8915), chrony 4.0+ hỗ trợ

## NTS — NTP over TLS

Network Time Security — xác thực NTP packet bằng TLS 1.3. Ngăn spoofing và MITM hoàn toàn. Cloudflare & Google đều support.
```shell
server time.cloudflare.com \
  iburst nts
# Verify:
chronyc ntssources
```

NTS = zero-trust NTP. Dùng nếu infra mày có security requirement cao

## Poll Interval & Burst Mode

**Poll interval**: tần suất chrony hỏi upstream (mặc định 64s–1024s, tự điều chỉnh). **iburst**: gửi 4–8 packet liên tiếp khi mới kết nối để sync nhanh hơn.

```shell
server x.x.x.x iburst
minpoll 4   # 16s minimum
maxpoll 10  # 1024s maximum
```

Dùng iburst cho tất cả server. Không set minpoll quá thấp → spam upstream

## makestep & initstepslew

**makestep**: cho phép step clock (nhảy ngay) trong N lần đầu khởi động nếu offset vượt ngưỡng. Sau đó chỉ slew. Quan trọng khi VM boot lần đầu.

```shell
makestep 1.0 3
# Step nếu offset > 1s
# Chỉ trong 3 lần sync đầu
```

Thiếu makestep → VM boot với clock sai hàng phút, TLS fail ngay lập tức

## Local Clock Fallback

Khi mất tất cả upstream, chrony có thể fallback về local hardware clock và vẫn serve time cho client (với stratum cao hơn). Dùng cho isolated network.

```shell
local stratum 10
# Serve time ngay cả khi
# không có upstream
# Stratum 10 = "degraded"
```

Không bật cái này trừ khi mày intentionally muốn island mode

## Container / K8s gotcha

Container **không có hardware clock riêng** — dùng chung kernel clock của host. Đừng chạy NTP daemon trong container, đừng mount /dev/rtc vào pod. Chỉ sync ở host level.

```shell
# SAI: chạy chrony trong pod
# ĐÚNG: chrony trên K8s node
# Pod tự inherit clock từ host
# Check: date trong pod = host
```

Chạy NTP trong container → conflict với host → clock chaos

## systemd-timesyncd — Conflict hay gặp

`systemd-timesyncd` là NTP client nhẹ built-in vào systemd, **chạy mặc định** trên Ubuntu/Debian. Khi cài chrony mà không disable `timesyncd` → 2 daemon cùng điều chỉnh clock → conflict, drift không ổn định.

```shell
# Kiểm tra timesyncd có đang chạy không
systemctl status systemd-timesyncd

# Chrony package thường tự disable timesyncd khi install
# Nhưng nên verify thủ công:
timedatectl show | grep NTP
# NTPSynchronized=yes
# NTP=yes → đang dùng timesyncd hoặc chrony

# Disable timesyncd trước khi chạy chrony
systemctl disable --now systemd-timesyncd

# Verify chrony đang giữ clock
timedatectl show-timesync
chronyc tracking
```

**Dấu hiệu conflict:** `chronyc sources` hiển thị sources bình thường nhưng offset dao động lạ, hoặc `timedatectl` báo synced nhưng chrony báo unsynchronized.

Nếu chỉ cần NTP client đơn giản trên desktop/VM không phải server → `systemd-timesyncd` đủ dùng, không cần cài chrony.

## PTP — Precision Time Protocol (IEEE 1588)

NTP đạt độ chính xác ~1ms trên LAN. **PTP (IEEE 1588)** đạt sub-microsecond (~100ns) nhờ hardware timestamping ở NIC.

```
Stratum 0 (GPS/atomic)
    │
    ▼
PTP Grandmaster Clock  ← hardware timestamp
    │  PTP (UDP :319/:320)
    ▼
PTP Boundary Clock (switch hỗ trợ PTP)
    │
    ▼
PTP Ordinary Clock (server)  ← hardware timestamp tại NIC
```

**Liên hệ với NTP/Chrony:**

Cloud provider (AWS, GCP, VMware) dùng PTP internally. Chrony có thể làm **PTP client** qua `refclock PHC` — đọc PTP hardware clock từ NIC thay vì dùng NTP packet:

```shell
# /etc/chrony.conf — dùng PTP hardware clock làm reference
refclock PHC /dev/ptp0 poll 0 dpoll -2 offset 0

# AWS: dùng PTP endpoint (chrony-friendly)
server 169.254.169.123 prefer iburst
# AWS cũng cung cấp PTP qua /dev/ptp0 trên Nitro instances
```

```shell
# Kiểm tra NIC có hỗ trợ hardware timestamping không
ethtool -T eth0
# Cần: hardware-transmit, hardware-receive, hardware-raw-clock

# Xem PTP clock device
ls /dev/ptp*
```

**Khi nào cần PTP thay NTP:**
- Financial trading, HFT — yêu cầu < 1µs
- Telecom (5G), industrial control systems
- VMware vSphere: ESXi host dùng PTP → VM inherit qua VMware Tools

**Trong scope On-Premise thông thường:** NTP + chrony là đủ. PTP chỉ cần khi có hardware hỗ trợ và yêu cầu độ chính xác cao hơn 1ms.

## chronyc waitsync & one-shot sync

### `chronyc waitsync` — Block đến khi sync xong

Dùng trong boot script hoặc Ansible để đảm bảo service chỉ start sau khi clock đã sync.

```shell
# Cú pháp: chronyc waitsync [max-tries] [max-correction] [max-skew] [interval]
chronyc waitsync 30 0.1 0.1 1
# Thử tối đa 30 lần
# Mỗi lần cách nhau 1 giây
# Dừng khi correction < 0.1s và skew < 0.1 ppm

# Đơn giản nhất: đợi tối đa 60s
chronyc waitsync 60

# Exit code: 0 = synced, 1 = timeout
```

**Dùng trong systemd service:**
```ini
# /etc/systemd/system/myapp.service
[Unit]
After=chronyd.service
Wants=chronyd.service

[Service]
ExecStartPre=chronyc waitsync 30 0.1 0.1 1
ExecStart=/usr/bin/myapp
```

**Dùng trong Ansible:**
```yaml
- name: Wait for NTP sync before proceeding
  command: chronyc waitsync 30 0.1 0.1 1
  changed_when: false
```

### `chronyd -q` — One-shot sync (không chạy daemon)

Sync một lần rồi exit, không chạy background. Dùng trong cron hoặc script init.

```shell
# Sync một lần từ server cụ thể rồi exit
chronyd -q 'server pool.ntp.org iburst'

# Với config file
chronyd -q -f /etc/chrony.conf

# Chỉ step nếu offset > 0.1s (không slew)
chronyd -q -t 10 'server pool.ntp.org iburst'
# -t 10 = timeout 10 giây
```

**`chronyd -Q`** — Dry run: tính offset nhưng không thay đổi clock:
```shell
chronyd -Q 'server pool.ntp.org iburst'
# Output: System clock wrong by 0.123456 seconds (step)
# Dùng để check offset mà không động đến clock
```

**So sánh `chronyc makestep` vs `chronyd -q`:**

| | `chronyc makestep` | `chronyd -q` |
|---|---|---|
| Daemon | Phải đang chạy | Không cần |
| Dùng khi | Clock lệch sau suspend | Script khởi tạo, cron |
| Sau khi chạy | Daemon tiếp tục | Process exit |