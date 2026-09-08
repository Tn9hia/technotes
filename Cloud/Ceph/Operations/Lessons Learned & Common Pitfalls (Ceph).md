---
tags:
  - ceph
  - operations
  - lessons-learned
---

# Lessons Learned & Common Pitfalls (Ceph)

Tổng hợp lại toàn bộ "lesson learned" rải rác trong các note khác vào 1 chỗ — dùng làm checklist tra cứu nhanh trước khi thao tác trên hệ thống production, đặc biệt hữu ích trong giai đoạn đầu mới nhận bàn giao cluster Ceph phục vụ CloudStack.

## Top nhầm lẫn khái niệm (do quen tư duy VMware)

| Nhầm lẫn | Sự thật | Chi tiết |
|---|---|---|
| Nghĩ CRUSH giống vSAN DOM — có 1 nơi tập trung tra lookup vị trí data | CRUSH là thuật toán **client tự tính toán**, không có bảng lookup tập trung nào cả | [[CRUSH Algorithm & CRUSH Map]] |
| Nhầm PG với OSD — tưởng PG là 1 khái niệm vật lý | PG (Placement Group) là **đơn vị sharding logic**, không phải ổ đĩa hay daemon vật lý | [[Placement Groups (PG)]] |
| Nghĩ `size=2 min_size=1` an toàn tương đương FTT=1 bên vSAN | Cấu hình này có rủi ro **split-brain/mất dữ liệu thật sự** khi chỉ còn 1 replica sống mà vẫn cho phép ghi | [[Pools, Replication & Erasure Coding]] |
| Tưởng MON down giống vCenter down — chỉ mất quản trị, VM vẫn chạy bình thường vô hạn | I/O vẫn chạy tạm thời, nhưng **mất quorum MON chặn mọi thay đổi cluster map**, và kéo dài đủ lâu sẽ ảnh hưởng cả I/O | [[MON - Monitor]] |
| Nhầm trạng thái `down` với `out` của OSD | `down` = daemon không phản hồi (có thể tạm thời), `out` = đã bị loại khỏi CRUSH map, data đã bắt đầu remap | [[OSD - Object Storage Daemon]] |
| Tưởng Ceph tự heal nên không cần làm gì khi hỏng ổ | Self-healing chỉ lo phần **data redundancy**, phần cứng hỏng vẫn cần thay thế kịp thời — để lâu giảm dần độ dự phòng còn lại | [[Ceph HA Architecture]] |

## Top sự cố vận hành cần cảnh giác

1. **Quên tắt `noout` sau bảo trì** → OSD tưởng đã maintenance xong nhưng flag vẫn còn, cluster không tự rebalance khi có OSD thật sự down sau đó. ([[Ceph Day 2 Operations]])
2. **Cluster chạm `full ratio`** → toàn bộ write trên cluster bị khóa ngay lập tức, không riêng pool nào bị đầy. ([[Ceph Sizing & Capacity Planning]])
3. **RGW bucket không được shard** → hotspot metadata trên 1 shard duy nhất khi bucket tăng trưởng lớn, degrade hiệu năng toàn bộ bucket đó. ([[RGW - Object Storage Gateway]])
4. **Xoay vòng cephx key mà không đồng bộ lại libvirt secret trên KVM host** → CloudStack fail âm thầm khi thao tác storage (tạo volume, snapshot), VM đang chạy vẫn bình thường nên khó phát hiện ngay. ([[Ceph with CloudStack]])
5. **Tắt scrub để "đỡ tốn tài nguyên"** → im lặng bỏ qua khả năng phát hiện corruption âm thầm (silent data corruption), chỉ phát hiện khi đã lan rộng. ([[Recovery, Backfill & Self-healing]])
6. **Thêm hàng loạt OSD/node cùng lúc không throttle** → rebalance storm chiếm băng thông, ảnh hưởng latency I/O production đang chạy. ([[Scaling the Cluster - Add-Remove Node & OSD]])
7. **Chọn sai failure domain trong CRUSH rule** (VD: `osd` thay vì `host` hoặc `rack`) → cluster tưởng đã an toàn nhưng thực chất nhiều replica cùng object nằm chung 1 host, mất host đó có thể mất luôn data. ([[CRUSH Algorithm & CRUSH Map]])
8. **MON store phình to bất thường do PG kẹt lâu ngày** (`stuck`/`inconsistent` không được xử lý) → MON DB tăng dung lượng liên tục, ảnh hưởng hiệu năng và thời gian recovery khi MON restart. ([[MON - Monitor]])
9. **Benchmark chỉ bằng sequential I/O lớn** → số liệu đẹp nhưng không phản ánh workload VM thật (random 4K-64K), gây ngỡ ngàng khi production chậm. ([[Ceph Performance Tuning]])
10. **Chạy `ceph pg repair` mù mà không xác định nguyên nhân gốc** → có thể "sửa" bằng cách ghi đè bản sao sai lên bản đúng nếu chọn nhầm primary, thay vì thực sự khắc phục corruption. ([[Ceph Troubleshooting]])
11. **Bỏ qua yêu cầu HEALTH_OK trước khi upgrade** → tiến trình `ceph orch upgrade` bị kẹt giữa chừng chờ PG lành mà PG không bao giờ tự lành trong lúc mixed-version. ([[Ceph Upgrade Procedure]])
12. **Clock skew giữa các MON** → quorum bất ổn định, cảnh báo `clock skew detected`, có thể dẫn tới mất quorum nếu lệch quá ngưỡng cho phép. ([[MON - Monitor]])
13. **Dùng `client.admin` key cho ứng dụng/CloudStack thay vì key least-privilege riêng** → 1 thành phần bị lộ key kéo theo toàn quyền quản trị cluster. ([[Ceph Security Considerations]])
14. **Không tách public/cluster network** → traffic replication/recovery cạnh tranh băng thông trực tiếp với traffic client, gây nghẽn lan rộng khi có sự cố backfill lớn. ([[Ceph Hardware & Network Design]])

## Nguyên tắc vận hành an toàn

> [!tip] Nguyên tắc vận hành an toàn
> 1. **Verify bằng lệnh, đừng tin giả định** — "chắc HEALTH_OK", "chắc key đã đồng bộ" đều cần xác nhận bằng `ceph -s`, `ceph health detail` cụ thể trước khi tin tưởng.
> 2. **Luôn xác nhận HEALTH_OK trước mọi thao tác rủi ro** — upgrade, thêm/xóa OSD hàng loạt, sửa CRUSH rule — không thao tác khi cluster đang WARN/ERR.
> 3. **Throttle thay vì vội vàng** — recovery/backfill/rebalance nên được kiểm soát tốc độ, đặc biệt trong giờ hành chính khi VM production đang chạy I/O.
> 4. **Hiểu rõ blast radius trước khi đụng CRUSH hoặc pool setting** — một thay đổi failure domain hay `size`/`min_size` sai có thể ảnh hưởng toàn bộ data trong pool đó, không chỉ 1 phần.
> 5. **Giữ public network và cluster network tách biệt** — đây là nền tảng để mọi throttle/QoS phía trên có ý nghĩa thực tế.

## Câu hỏi nên tự đặt ra định kỳ

- Nếu một host chết ngay bây giờ, tôi biết chính xác cluster mất bao lâu để phục hồi lại đủ redundancy không?
- Tôi có biết chính xác failure domain hiện tại là gì (`host`, `rack`...) và cluster chịu được tối đa bao nhiêu failure cùng lúc không?
- Lần cuối tôi thực sự test disaster recovery / restore từ backup là khi nào — hay chỉ tin "job backup đang chạy"?
- Toàn bộ daemon trong cluster đã cùng ở version vá lỗi mới nhất chưa, hay còn sót daemon cũ từ lần upgrade dở dang?
- Có flag `noout`/maintenance nào còn sót lại từ một maintenance window cũ mà không ai nhớ tắt không?

---
*Xem thêm: [[Ceph Prerequisites]] | [[Ceph Troubleshooting]] | [[Ceph HA Architecture]] | [[Ceph|Ceph]]*
