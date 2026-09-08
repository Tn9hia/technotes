---
tags:
  - ceph
  - troubleshooting
  - operations
  - debug
---

# Ceph Troubleshooting

**Debug theo triệu chứng** — vì cephadm chạy mọi daemon dưới dạng container, việc tra log không còn đơn giản như đọc 1 file text cố định trên host; cần biết cả `cephadm logs`, `journalctl`, lẫn đường dẫn log trên filesystem host.

> [!tip] So với VMware vSAN
> vSAN gom hầu hết log debug vào `vsanmgmtd`/`vmkernel` trên từng ESXi host, xem qua `esxcli vsan` hoặc vCenter Support Bundle. Ceph phân tán log theo **từng daemon riêng biệt** (mỗi OSD, mỗi MON có log riêng) — cephadm ghi log vào cả journald (qua container) **và** file trên host, nên có 2 cách tra cứu song song. Quen với việc "log theo daemon" thay vì "log theo host" là chuyển đổi tư duy quan trọng nhất khi debug Ceph.

## Nguyên tắc debug chung

```
1. Luôn bắt đầu bằng: ceph -s → ceph health detail
2. Xác định daemon/PG/OSD cụ thể liên quan (đừng debug mù trên toàn cluster)
3. Tra log đúng daemon đó — ưu tiên cephadm logs / journalctl trước khi vào file trực tiếp
4. Với vấn đề PG/dữ liệu: ceph pg <id> query luôn là bước đầu, cho biết state chi tiết + acting set
5. Với vấn đề performance: ceph osd perf trước, rồi mới soi historic ops của OSD nghi vấn
```

## Log Files Reference

| Thành phần | Log path (trên host, cephadm) | Cách xem khác |
|---|---|---|
| Mọi daemon (tổng quát) | `/var/log/ceph/<fsid>/ceph-<daemon>.log` | `cephadm logs --name <daemon>` |
| Mọi daemon (qua journald) | — | `journalctl -u ceph-<fsid>@<daemon>.service` |
| MON | `/var/log/ceph/<fsid>/ceph-mon.<id>.log` | `journalctl -u ceph-<fsid>@mon.<id>.service` |
| MGR | `/var/log/ceph/<fsid>/ceph-mgr.<id>.log` | `journalctl -u ceph-<fsid>@mgr.<id>.service` |
| OSD | `/var/log/ceph/<fsid>/ceph-osd.<id>.log` | `journalctl -u ceph-<fsid>@osd.<id>.service` |
| MDS (CephFS) | `/var/log/ceph/<fsid>/ceph-mds.<id>.log` | `journalctl -u ceph-<fsid>@mds.<id>.service` |
| RGW | `/var/log/ceph/<fsid>/ceph-rgw.<id>.log` | `journalctl -u ceph-<fsid>@rgw.<id>.service` |
| Cephadm chính nó | `/var/log/ceph/cephadm.log` | — |
| Audit log (cluster commands) | `/var/log/ceph/<fsid>/ceph.audit.log` | — |

```bash
# Cách tra log chuẩn với cephadm — không cần biết fsid thuộc lòng
ceph orch ps                                    # xem danh sách daemon + tên chính xác
cephadm logs --name osd.12                      # xem log 1 daemon cụ thể (đọc từ journald)
cephadm logs --name osd.12 -- -f                 # follow realtime (thêm flag journalctl sau --)
journalctl -u ceph-$(ceph fsid)@osd.12.service --since "1 hour ago"

# Lấy fsid nếu cần dùng trực tiếp
ceph fsid
```

## HEALTH_WARN: "X pgs not deep-scrubbed in time"

Thường **không nghiêm trọng** — là backlog scrub tích lũy do lịch scrub bị nghẽn (tải I/O cao, cửa sổ scrub bị giới hạn giờ), nhưng không nên phớt lờ lâu vì deep-scrub là cơ chế chính phát hiện silent data corruption.

```bash
ceph health detail                              # liệt kê chính xác PG nào bị trễ
ceph pg dump | grep -i "not deep-scrubbed"       # xem chi tiết last scrub timestamp
ceph config get osd osd_scrub_begin_hour         # kiểm tra cửa sổ giờ scrub đang cấu hình
ceph config get osd osd_scrub_end_hour
ceph config get osd osd_max_scrubs               # số scrub đồng thời cho phép mỗi OSD

# Ép scrub thủ công 1 PG cụ thể nếu cần gấp
ceph pg deep-scrub <pgid>
```

Nếu backlog liên tục tăng thay vì giảm dần, cân nhắc nới `osd_max_scrubs` hoặc mở rộng cửa sổ giờ scrub — nhưng đánh đổi là tăng tải I/O trong giờ đó.

## HEALTH_ERR: "pgs inconsistent"

Nghĩa là scrub/deep-scrub đã **phát hiện thật sự** dữ liệu không khớp giữa các bản sao của 1 PG — cần xử lý cẩn thận, không phải lỗi tạm thời.

```bash
ceph health detail                              # xác định chính xác PG nào inconsistent
ceph pg <pgid> query                            # xem acting set, recovery state chi tiết
rados list-inconsistent-obj <pgid> --format=json-pretty     # object nào lỗi, OSD nào có lỗi (đọc kỹ trước khi repair)
rados list-inconsistent-snapset <pgid>          # nếu liên quan snapshot metadata

ceph pg repair <pgid>                           # yêu cầu Ceph tự sửa
```

> [!warning] Lesson learned: `ceph pg repair` không phải nút "sửa an toàn" tuyệt đối
> `ceph pg repair` mặc định lấy dữ liệu từ **bản sao primary** của PG và ghi đè lên các bản sao khác đang khác biệt — nó **giả định primary luôn đúng**. Trong đa số trường hợp (bit rot ngẫu nhiên trên 1 OSD không phải primary) điều này đúng và an toàn. Nhưng trong trường hợp hiếm — chính OSD primary mới là nơi bị corrupt (do lỗi firmware disk, lỗi RAM không ECC trên node đó, hoặc bad sector không được phát hiện đúng lúc) — chạy `repair` phản xạ ngay khi thấy `HEALTH_ERR` sẽ **lấy bản dữ liệu sai làm chuẩn** và ghi đè lên các bản sao còn đúng, biến corrupt cục bộ thành corrupt vĩnh viễn trên toàn bộ PG.
>
> **Quy trình đúng trước khi repair (trừ trường hợp quá hiển nhiên là lỗi tầm thường):** (1) chạy `rados list-inconsistent-obj <pgid> --format=json-pretty` và đọc kỹ phần `errors` của từng shard — nó cho biết OSD nào có `read_error`, `data_digest_mismatch`, `size_mismatch`; (2) nếu chỉ 1 OSD báo lỗi rõ ràng (VD: `read_error` — disk thật sự lỗi khi đọc) và OSD đó **không phải** primary, `repair` an toàn; (3) nếu nghi ngờ chính primary có vấn đề (disk đó có lịch sử SMART lỗi, hoặc host đó gần đây có sự cố phần cứng khác), nên xem xét set lại primary tạm thời (`ceph osd pg-temp` hoặc điều chỉnh primary affinity) hoặc tham khảo kỹ trước khi repair; (4) với dữ liệu cực kỳ quan trọng, cân nhắc backup thủ công object nghi vấn trước khi repair. Đừng biến `ceph pg repair` thành phản xạ tự động mỗi khi thấy `HEALTH_ERR` — nó là công cụ có chủ đích, không phải nút "fix it".

## OSD Flapping (liên tục up/down)

```bash
ceph -w                                          # theo dõi realtime khi đang xảy ra
ceph health detail                               # thường có dòng "X osds down" hoặc OSD_FLAPPING
ceph osd tree                                    # OSD nào flapping, thuộc host nào

# Kiểm tra log OSD nghi vấn — tìm "wrongly marked me down" hoặc heartbeat timeout
cephadm logs --name osd.<id> | grep -i "wrongly marked\|heartbeat"

# Kiểm tra network giữa các node — nguyên nhân phổ biến nhất
ping -c 20 <osd-host-ip>
ethtool <interface> | grep Speed                # xác nhận link speed đúng kỳ vọng, không bị auto-negotiate sai

# Kiểm tra disk — SMART, dmesg lỗi I/O
smartctl -a /dev/<device>
dmesg -T | grep -i "error\|reset" | tail -50
```

| Nguyên nhân | Dấu hiệu | Hướng xử lý |
|---|---|---|
| Network chập chờn (packet loss, MTU mismatch trên cluster network) | Nhiều OSD trên nhiều host khác nhau cùng flap | Kiểm tra switch, cable, MTU jumbo frame nhất quán 2 đầu |
| Disk sắp hỏng | Chỉ 1-2 OSD cụ thể flap, log có I/O error | `smartctl`, cân nhắc thay disk, đánh dấu `out` trước khi hỏng hẳn |
| `osd_heartbeat_grace`/timeout quá chặt so với tải thực tế | Flap tăng vọt đúng lúc cluster tải cao (backfill/recovery) | Cân nhắc tăng `osd_heartbeat_grace` tạm thời trong đợt tải cao, không nên tăng vĩnh viễn mù quáng |
| Node quá tải CPU/RAM khiến OSD process không kịp gửi heartbeat | Flap trùng thời điểm CPU/RAM host cao | Kiểm tra `top`/`node_exporter` metrics đúng lúc flap |

## Slow Requests / Slow Ops

```bash
ceph health detail                               # "X slow ops, oldest one blocked for Y sec"
ceph daemon osd.<id> dump_historic_ops            # chạy trên chính host của OSD đó, hoặc qua cephadm shell
ceph daemon osd.<id> dump_historic_ops_by_duration
ceph daemon osd.<id> ops                          # ops đang treo ngay lúc này

# Xác định op đang chờ ở bước nào trong pipeline (queue, network, disk, replica ack...)
```

| Nguyên nhân thường gặp | Cách xác nhận |
|---|---|
| Disk quá tải/chậm bất thường | `ceph osd perf` — `apply_latency`/`commit_latency` cao rõ rệt ở 1 OSD |
| Network chậm/lag giữa OSD | So sánh latency giữa các OSD cùng host vs khác host |
| PG bị stuck (recovery/backfill kẹt) | `ceph pg dump_stuck`, `ceph pg <id> query` |
| 1 OSD cụ thể là "outlier" chậm liên tục | `dump_historic_ops` cho thấy op luôn chờ lâu ở đúng OSD đó — cân nhắc `ceph osd out` để loại tạm |

## MON Quorum Lost

Tình huống nghiêm trọng nhất — cần cực kỳ thận trọng, vì thao tác sai trên MON có thể làm mất luôn cluster map.

```bash
ceph -s                                          # nếu vẫn còn quorum tối thiểu, sẽ thấy rõ MON nào thiếu
ceph orch ps --daemon_type mon                    # kiểm tra MON daemon còn chạy không
ceph mon stat
ceph quorum_status --format=json-pretty          # chi tiết quorum hiện tại (chạy trên MON còn sống)
```

```
Nguyên tắc khi mất quorum (còn > 0 MON sống nhưng không đủ đa số):
1. TUYỆT ĐỐI không xóa/reinit MON store vội — đó là bản sao cluster map, mất hết là mất cluster
2. Ưu tiên khôi phục MON đã chết (network/disk/service) thay vì rebuild từ đầu
3. Nếu chắc chắn cần "phẫu thuật" (VD: remove MON chết vĩnh viễn để hạ số lượng cần cho quorum),
   dùng ceph-mon --extract-monmap / --inject-monmap CHỈ SAU KHI đã backup toàn bộ mon data dir
4. Nếu chỉ còn 1 MON sống và cần tự nó thành quorum tạm thời để cứu cluster,
   đây là thao tác rủi ro cao — nên tham khảo kỹ tài liệu chính thức cho đúng phiên bản
   trước khi thực hiện, không làm theo trí nhớ
```

> [!warning] Không có "undo" cho thao tác sai trên MON store
> Khác với OSD (mất 1 OSD có bản sao khác bù đắp), MON lưu **toàn bộ cluster map** — mất quá nửa số MON đồng thời hoặc corrupt MON store trong lúc "sửa chữa" có thể khiến cluster không còn cách nào phục hồi ngoài rebuild từ backup/OSD (rất phức tạp, có thể mất dữ liệu). Luôn backup mon data dir trước bất kỳ thao tác monmap thủ công nào, và ưu tiên tuyệt đối việc khôi phục MON hiện có hơn là "sửa" quorum.

## Cluster Stuck ở `nearfull`/`full`

```bash
ceph df detail                                   # pool nào gần đầy nhất
ceph osd df | sort -k8 -n -r | head               # OSD nào %USE cao nhất (cột USE% tùy version)
ceph health detail                                # OSD_NEARFULL / OSD_FULL, liệt kê chính xác OSD nào
```

Xử lý khẩn cấp khi chạm `full` (cluster ngừng nhận write):

| Hành động | Khi nào dùng | Lưu ý |
|---|---|---|
| `ceph osd reweight <id> 0.9` cho OSD đầy nhất | Cần giải phóng ngay, tạm thời | Chỉ là giảm tải PG mới, không xóa dữ liệu — vẫn cần thêm capacity thật sớm sau đó |
| Xóa snapshot/object không cần thiết | Có dữ liệu rác xác định được | An toàn hơn reweight nhưng cần biết chắc dữ liệu không cần |
| Thêm OSD/host mới | Giải pháp gốc rễ, không phải chữa cháy | Cần thời gian backfill, không tức thời |
| Tăng tạm `full_ratio`/`nearfull_ratio` | CHỈ khi chắc chắn có kế hoạch giải phóng dung lượng ngay sau | Rủi ro cao nếu quên hạ lại — xem [[Ceph Sizing & Capacity Planning]] |

Xem chiến lược capacity planning dài hạn (đặt threshold sớm, dự báo trend) ở [[Ceph Sizing & Capacity Planning]] — xử lý khẩn cấp ở trên chỉ là giải pháp tạm thời, không thay thế việc lập kế hoạch dung lượng đúng.

## Client (RBD/CephFS/RGW) Không Kết Nối Được

```bash
# 1. Kiểm tra MON connectivity từ phía client trước tiên
ceph -s --id <client-user>                       # test bằng đúng user/keyring client đang dùng
telnet <mon-ip> 6789                             # hoặc: nc -zv <mon-ip> 3300 (msgr v2)

# 2. Kiểm tra cephx cap của user
ceph auth get client.<user>
ceph auth caps client.<user> mon 'allow r' osd 'allow rwx pool=<pool>'   # sửa cap nếu thiếu quyền

# 3. Với RBD cụ thể — kiểm tra image có bị lock bởi client khác không
rbd status <pool>/<image>

# 4. Với CephFS — kiểm tra MDS đang active
ceph fs status

# 5. Với RGW — kiểm tra service, không phải lỗi tầng RADOS
ceph orch ps --daemon_type rgw
radosgw-admin user info --uid=<uid>
```

| Nguyên nhân | Dấu hiệu |
|---|---|
| Sai/thiếu cephx cap | `Permission denied` khi mount/connect |
| Firewall chặn port MON (6789 v1 / 3300 v2) hoặc OSD (6800-7300 range) | Timeout, không phải permission error |
| Keyring sai hoặc hết hạn | `unable to authenticate` |
| RBD image đang bị lock bởi client khác (watcher cũ chưa release) | `rbd map` treo hoặc lỗi lock |
| MDS không active (CephFS) | Mount CephFS treo vô thời hạn |

## Bảng lỗi thường gặp

| Lỗi/Cảnh báo | Nguyên nhân | Hướng xử lý |
|---|---|---|
| `HEALTH_WARN: X pgs not deep-scrubbed in time` | Backlog scrub do tải cao hoặc cửa sổ giờ hẹp | Thường benign, theo dõi trend; nới `osd_max_scrubs` nếu tăng liên tục |
| `HEALTH_ERR: pgs inconsistent` | Scrub phát hiện dữ liệu lệch giữa các bản sao | Kiểm tra `list-inconsistent-obj` trước, `ceph pg repair` sau khi xác nhận primary đúng |
| `OSD_FULL` / `OSD_NEARFULL` | 1+ OSD chạm ngưỡng dung lượng | Reweight khẩn cấp, thêm capacity, xem Sizing note |
| `MON_CLOCK_SKEW` | NTP/chrony không đồng bộ đúng giữa các MON | Kiểm tra `ceph time-sync-status`, sửa chrony |
| `PG_AVAILABILITY` (PG not active) | Không đủ OSD trong acting set để phục vụ I/O | `ceph pg <id> query`, kiểm tra OSD liên quan có down không |
| `PG_DEGRADED` | PG thiếu bản sao so với `size` cấu hình | Bình thường trong lúc recovery; kéo dài bất thường thì điều tra |
| `REQUEST_SLOW` / slow ops | Disk/network chậm, hoặc PG kẹt | `dump_historic_ops`, `ceph osd perf` |
| `MON_DOWN` | 1+ MON daemon không chạy | `ceph orch ps`, kiểm tra host/network MON đó |
| `Permission denied` (client) | Thiếu cephx cap | `ceph auth caps` sửa lại quyền |
| `CEPHADM_FAILED_DAEMON` | Container daemon crash loop | `cephadm logs --name <daemon>`, kiểm tra config/resource host |

---
*Xem thêm: [[Ceph Day 2 Operations]] | [[Ceph Monitoring & Alerting]] | [[Ceph CLI Cheatsheet]] | [[Ceph|Ceph]]*
