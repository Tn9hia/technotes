---
tags:
  - ceph
  - operations
  - day2
---

# Ceph Day 2 Operations

**Vận hành thường nhật** một cluster Ceph khác khá nhiều so với triển khai ban đầu — phần lớn công việc là kiểm tra sức khỏe định kỳ, bảo trì host an toàn, và quản lý các "flag" tạm thời trong lúc thao tác. Phần lớn sự cố production trên Ceph không đến từ bug phần mềm mà từ **quy trình bảo trì không đầy đủ** — đặc biệt là quên dọn dẹp trạng thái sau khi bảo trì xong.

> [!tip] So với VMware vSAN
> vCenter tự động hóa gần như toàn bộ vòng đời bảo trì (DRS di dời VM, vSAN tự "resync" khi host vào maintenance mode và tự dừng khi thoát). Ceph với cephadm cũng có khái niệm maintenance mode tương tự (`ceph orch host maintenance enter/exit`), nhưng **ở mức thấp hơn** — nó không tự động re-check định kỳ, và nếu bạn quên bước "exit" hoặc quên unset 1 flag thủ công đã set trước đó, sẽ không có cảnh báo mạnh nào nhắc bạn ngay lập tức. Kỷ luật vận hành ở đây quan trọng hơn ở vSAN.

## Checklist kiểm tra sức khỏe hằng ngày

- [ ] `ceph -s` — trạng thái tổng quan, có đang `HEALTH_OK` không
- [ ] `ceph health detail` — nếu WARN/ERR, đọc rõ từng dòng, đừng bỏ qua
- [ ] `ceph orch ps` — có daemon nào không phải `running` (stopped, error, unknown)
- [ ] `ceph df` — theo dõi xu hướng dung lượng theo pool, không chỉ số hiện tại
- [ ] `ceph osd df` — kiểm tra có OSD nào lệch %USE bất thường (dấu hiệu CRUSH weight lệch hoặc PG phân bố không đều)
- [ ] `ceph crash ls` — có crash report mới nào chưa xử lý/archive
- [ ] Kiểm tra các flag đang bật có hợp lý không: `ceph osd dump | grep flags` (không nên có flag "mồ côi" từ lần bảo trì trước)
- [ ] `ceph time-sync-status` — clock skew giữa các MON (xem thêm ở [[Ceph Monitoring & Alerting]])

```bash
# Script nhanh cho routine check buổi sáng
ceph -s
ceph health detail
ceph orch ps --refresh
ceph df
ceph osd df | tail -5
ceph crash ls
```

## Maintenance Mode cho 1 Host (trước khi patch/reboot)

```bash
ceph orch host maintenance enter <hostname>
# → cephadm tự động: dừng daemon trên host, set 'noout' phạm vi liên quan
#   để Ceph không coi các OSD trên host này là "mất vĩnh viễn" và bắt đầu backfill dữ liệu đi nơi khác

# ... thực hiện patch / reboot host ...

ceph orch host maintenance exit <hostname>
# → khởi động lại daemon, gỡ trạng thái maintenance
```

| Bước | Việc cần làm | Vì sao |
|---|---|---|
| 1 | `ceph -s` — xác nhận `HEALTH_OK` trước khi bắt đầu | Không bao giờ bắt đầu bảo trì khi cluster đang degraded |
| 2 | `ceph orch host maintenance enter <host>` | Set các flag cần thiết + dừng daemon an toàn |
| 3 | Patch OS / reboot | — |
| 4 | `ceph orch host maintenance exit <host>` | Khởi động lại daemon, gỡ maintenance state |
| 5 | `ceph -s` — chờ về `HEALTH_OK` hoàn toàn | Xác nhận host đã tái hòa nhập đầy đủ trước khi làm host tiếp theo |

> [!tip] Vì sao maintenance mode quan trọng hơn vẻ ngoài
> Nếu reboot 1 host **mà không** vào maintenance mode trước, Ceph sẽ coi các OSD trên host đó là "down" — sau khoảng `mon_osd_down_out_interval` (mặc định 600 giây / 10 phút), chúng bị đánh dấu `out` và cluster **bắt đầu backfill** để tái tạo đủ bản sao dữ liệu ở nơi khác. Với 1 lần reboot 5-10 phút, việc backfill này hoàn toàn không cần thiết — vừa tốn băng thông/IO, vừa có thể tự hủy giữa chừng khi host quay lại rồi lại phải backfill ngược. Maintenance mode set `noout` đúng phạm vi để tránh việc này.

## Patch/Reboot nhiều node — nguyên tắc tuần tự

```
Với mỗi node Ceph (làm TUẦN TỰ, không song song trong cùng failure domain):
1. ceph -s → xác nhận HEALTH_OK
2. ceph orch host maintenance enter <host>
3. Patch OS, reboot nếu cần
4. Đợi host lên lại, daemon tự khởi động (hoặc restart thủ công nếu cần)
5. ceph orch host maintenance exit <host>
6. ceph -s → đợi về HEALTH_OK hoàn toàn (không chỉ "no more WARN mới")
7. Chỉ sau khi bước 6 xong mới chuyển sang host tiếp theo
```

> [!warning] Lesson learned: patch nhiều host cùng failure domain cùng lúc
> Nếu cluster dùng CRUSH rule theo `host` (replication factor 3, mỗi bản sao 1 host khác nhau) mà bạn patch 2 host cùng lúc, có PG nào đó có thể **mất 2/3 bản sao tạm thời** — nếu bản sao còn lại gặp sự cố trong đúng lúc đó (hiếm nhưng không phải zero), dữ liệu PG đó mất thật. Ngay cả khi không mất dữ liệu, pool có `min_size 2` sẽ có PG rơi vào trạng thái không writable nếu 2 trong 3 OSD cùng down. Luôn patch **từng host một** theo đúng failure domain của CRUSH rule đang dùng, y hệt nguyên tắc rolling maintenance ở tầng compute KVM.

## Config Changes — Centralized Config vs ceph.conf cũ

```bash
# Cách chuẩn hiện nay (từ Ceph Octopus trở đi, mặc định với cephadm)
ceph config set osd osd_max_backfills 4
ceph config set global mon_allow_pool_delete true
ceph config get mon.a mon_allow_pool_delete

# Xem toàn bộ override đang áp dụng
ceph config dump
```

| | Centralized config (`ceph config set`) | `ceph.conf` file cũ |
|---|---|---|
| Lưu ở đâu | Trong MON database (config KV store) | File text trên từng host, phải tự đồng bộ |
| Áp dụng khi nào | Runtime, hầu hết không cần restart daemon | Thường cần restart daemon để đọc lại file |
| Đồng bộ giữa các node | Tự động (MON là nguồn chân lý duy nhất) | Thủ công / cần công cụ config management (Ansible...) |
| Khuyến nghị hiện tại | **Chuẩn mặc định** với cephadm | Chỉ dùng cho vài tham số bootstrap ban đầu (VD: `mon_host`) |

> [!tip] Vì sao centralized config là chuẩn
> Với kiến trúc cephadm, daemon chạy trong container và không có `ceph.conf` "tiện" để sửa tay trên từng host như trước — sửa sai 1 file trên 1 node dễ gây cluster config lệch nhau (drift) mà rất khó phát hiện. `ceph config set` ghi thẳng vào MON, áp dụng nhất quán cho toàn cluster (hoặc đúng phạm vi daemon/host bạn chỉ định), và `ceph config dump` luôn cho bạn 1 nguồn sự thật duy nhất để audit.

## Kiểm tra Capacity & PG định kỳ

```bash
ceph df detail                          # xu hướng theo pool, không chỉ % tổng
ceph osd pool autoscale-status          # PG autoscaler đang đề xuất gì cho từng pool
ceph pg dump_stuck                      # PG bị stuck bất thường — nên = 0 hàng ngày
```

Nên theo dõi **xu hướng** (trend qua Prometheus/Grafana — xem [[Ceph Monitoring & Alerting]]) thay vì chỉ nhìn số hiện tại, để dự báo trước khi chạm ngưỡng `nearfull`/`full` — tham khảo thêm ở [[Ceph Sizing & Capacity Planning]].

## Dọn dẹp Crash Directory

```bash
ceph crash ls                            # xem crash report còn pending
ceph crash info <id>                     # đọc chi tiết trước khi archive (đừng archive mù)
ceph crash archive-all                   # archive toàn bộ sau khi đã review
```

Crash report tồn đọng lâu (`ceph crash stat` > 0 kéo dài) làm nhiễu `ceph health` và khiến việc phát hiện crash **mới** khó hơn giữa một đống report cũ đã biết nguyên nhân.

## Safe vs Unsafe Flags — Dùng đúng lúc bảo trì có kế hoạch

| Flag | Tác dụng | Dùng khi | Rủi ro nếu quên unset |
|---|---|---|---|
| `noout` | OSD down không tự động bị đánh dấu `out` (không kích hoạt backfill) | Reboot/bảo trì host ngắn hạn | Cluster không tự "chữa lành" khi có OSD chết thật sau này — mất redundancy âm thầm |
| `noin` | OSD mới thêm/restart không tự động vào `in` | Thêm OSD hàng loạt, muốn kiểm soát thời điểm bắt đầu backfill | OSD mới không bao giờ nhận dữ liệu, tưởng đã cân bằng nhưng chưa |
| `nobackfill` | Chặn hoàn toàn backfill (tái tạo dữ liệu quy mô lớn) | Bảo trì network/storage ảnh hưởng băng thông | Cluster không phục hồi redundancy sau sự cố thật |
| `norecover` | Chặn recovery (tương tự backfill, phạm vi hẹp hơn) | Tình huống khẩn cấp cần dừng ngay mọi I/O phục hồi | Tương tự `nobackfill` |
| `norebalance` | Chặn di chuyển dữ liệu do CRUSH map thay đổi (VD: thêm OSD) | Đang thêm OSD hàng loạt, muốn kiểm soát tốc độ rebalance | Cluster không tối ưu lại phân bố dữ liệu, có OSD quá tải kéo dài |
| `noscrub` | Tạm dừng scrub thường | Giờ cao điểm cần toàn bộ IO cho VM, hoặc bảo trì | Bỏ lỡ phát hiện sớm lỗi dữ liệu — nếu quên unset lâu, rủi ro corrupt âm thầm tăng |
| `nodeep-scrub` | Tạm dừng deep-scrub (tốn IO hơn scrub thường) | Tương tự trên, ít gấp hơn | Tương tự trên |

```bash
# Set/unset — luôn đi theo cặp, luôn có kế hoạch unset rõ ràng
ceph osd set noout
ceph osd unset noout

# Kiểm tra flag nào đang bật trên cluster
ceph osd dump | grep flags
ceph -s          # cũng hiển thị flag đang bật ngay trong phần đầu output
```

> [!warning] Lesson learned: quên unset `noout`/maintenance flag — nguyên nhân sự cố production phổ biến nhất
> Đây là bẫy vận hành Ceph kinh điển nhất: set `noout` (hoặc vào maintenance mode) trước 1 đợt bảo trì, xong việc rồi **quên unset**. Cluster vẫn chạy `HEALTH_OK` hoặc chỉ WARN nhẹ — không có gì "nổ" ngay lập tức, nên không ai để ý. Vài tuần hoặc vài tháng sau, một OSD/disk khác chết **thật sự** (không phải do bảo trì) — nhưng vì `noout` vẫn đang bật, Ceph **không tự động backfill** để khôi phục đủ số bản sao. Cluster âm thầm chạy với dữ liệu **kém redundant hơn mức thiết kế** trong suốt thời gian đó, và nếu thêm 1 sự cố nữa xảy ra trước khi có người phát hiện, có thể dẫn tới **PG mất hoàn toàn** (`min_size` không đạt) hoặc mất dữ liệu thật. Điều nguy hiểm nhất là **không có tín hiệu rõ ràng** nào báo "bạn đang quên 1 flag" — nó nằm im trong output `ceph -s`/`ceph osd dump` mà nhiều người không nhìn kỹ mỗi ngày.
>
> **Cách phòng tránh:** (1) luôn dùng `ceph orch host maintenance enter/exit` thay vì set flag thủ công khi có thể — nó tự quản lý phạm vi và có `ceph orch host ls` cho biết host nào đang ở trạng thái maintenance; (2) đưa "kiểm tra flag đang bật" vào checklist hằng ngày (`ceph osd dump | grep flags`); (3) cấu hình alert riêng cho tình huống "flag bảo trì bật kéo dài quá X giờ" (xem [[Ceph Monitoring & Alerting]]); (4) ghi log/ticket rõ ràng mỗi lần set flag thủ công, kèm thời điểm dự kiến unset.

## Runbook xử lý sự cố tối thiểu

```
1. Xác định phạm vi: 1 OSD? 1 host? 1 pool? toàn cluster?
2. ceph -s + ceph health detail trước tiên — luôn là bước đầu
3. Nếu liên quan OSD/disk → xem [[OSD - Object Storage Daemon]] và [[Ceph Troubleshooting]]
4. Nếu liên quan PG/dữ liệu → xem [[Placement Groups (PG)]] và [[Recovery, Backfill & Self-healing]]
5. Nếu liên quan MON/quorum → xử lý cực kỳ thận trọng, xem [[Ceph Troubleshooting]]
6. Ghi lại timeline + root cause sau khi xử lý xong
7. Cập nhật vào [[Lessons Learned & Common Pitfalls (Ceph)]] nếu là bài học mới
```

---
*Xem thêm: [[Ceph CLI Cheatsheet]] | [[Ceph Troubleshooting]] | [[Ceph Monitoring & Alerting]] | [[Ceph|Ceph]]*
