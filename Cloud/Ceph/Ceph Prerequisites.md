---
tags:
  - ceph
  - prerequisites
---

# Ceph Prerequisites

Kiến thức nền nên có trước khi nhận bàn giao và vận hành production **Ceph** — đặc biệt nếu xuất thân từ **VMware/vSAN** và Ceph cluster này đang phục vụ Primary Storage cho **Apache CloudStack (KVM)**.

## Kiến thức bắt buộc

| Mảng | Vì sao cần | Ghi chú cho dân VMware/vSAN |
|---|---|---|
| **Linux storage & block device cơ bản** | OSD daemon quản lý trực tiếp thiết bị block (`/dev/sdX`, LVM logical volume, hoặc raw disk); cần hiểu partition, filesystem, LVM để đọc đúng trạng thái OSD | vSAN ẩn hoàn toàn lớp block device dưới UI — bạn không cần biết `/dev/sdX` là gì để vận hành vSAN. Với Ceph, bạn **sẽ** phải nhìn thẳng vào `lsblk`, `ceph-volume lvm list`, xác định đúng ổ vật lý khi thay OSD |
| **Distributed systems cơ bản — quorum/consensus** | MON dùng thuật toán **Paxos** để đồng thuận trạng thái cluster; hiểu quorum (majority) giải thích vì sao luôn cần **số MON lẻ** và vì sao mất quá nửa MON = cluster "đóng băng" | vSAN admin đã quen khái niệm **witness** (vSAN 2-node) — đây là hình thức quorum đơn giản hoá. MON quorum là khái niệm tổng quát hơn (N node, chịu được ⌊N/2⌋ node chết) — cùng gốc tư duy nhưng cần hiểu sâu hơn vì áp dụng cho *cả cluster*, không chỉ 1 tình huống 2-node đặc biệt |
| **Networking cơ bản — VLAN, bonding, L2/L3** | Ceph khuyến nghị tách **public network** (client ↔ cluster) và **cluster network** (OSD ↔ OSD, replication/recovery traffic) — sai cấu hình mạng ảnh hưởng trực tiếp latency và khả năng chịu lỗi | Tương tự tư duy tách vSAN traffic ra VMkernel port group riêng (vsan vmknic), nhưng Ceph tách network **ở tầng cluster-wide config** (`ceph.conf`/`ceph config`), không phải per-host qua UI — sai một node là ảnh hưởng cách node đó tham gia cluster |
| **Nhận thức cơ bản về CloudStack/KVM/libvirt** | Ceph cluster này tồn tại để phục vụ CloudStack — hiểu QEMU/librbd gọi vào Ceph thế nào giúp bạn debug đúng lớp khi có sự cố "storage chậm" | Xem chi tiết ở [[Ceph with CloudStack]] — điểm mấu chốt: KVM host nói chuyện thẳng với Ceph qua `librbd`, Management Server **không** nằm trong đường I/O, khác hẳn mô hình vCenter/ESXi mà bạn quen |
| **Khái niệm replication/erasure coding cơ bản** | Quyết định pool dùng 3x-replica hay EC ảnh hưởng trực tiếp dung lượng khả dụng và hiệu năng | vSAN admin đã quen **FTT** (Failure To Tolerate) và RAID-1/5/6 storage policy — ánh xạ khá thẳng sang Replication size và Erasure Coding k+m của Ceph, nhưng Ceph áp cấu hình này **theo từng pool**, không theo từng VM/object như storage policy vSAN |

> [!tip] So với VMware vSAN
> Cái vSAN admin **đã có sẵn**: trực giác về failure domain, fault tolerance, và tư duy "storage phân tán không có 1 điểm chết". Cái **hoàn toàn mới**: mô hình tách daemon MON/MGR/OSD/MDS/RGW thành các tiến trình độc lập trên các node khác nhau (vSAN gộp mọi thứ vào ESXi kernel), và đặc biệt là mô hình **client-side CRUSH computation** — client (QEMU/librbd) tự tính toán object nằm ở OSD nào bằng thuật toán, không hỏi 1 bảng tra cứu tập trung. Đây là khác biệt kiến trúc gốc rễ nhất so với vSAN, nên dành thời gian đọc kỹ [[CRUSH Algorithm & CRUSH Map]] sớm.

## Checklist thông tin cần xin khi nhận bàn giao

> [!tip] Hỏi trước, đừng đoán
> Ceph là hệ thống mà một quyết định sai (VD: đổi replication size, đổi CRUSH rule) có thể ảnh hưởng dữ liệu **toàn cluster**, không phải 1 VM đơn lẻ. Xin đủ thông tin trước khi thao tác bất cứ gì.

- [ ] **Ceph version** chính xác (Squid 19.x / Tentacle 20.x / cũ hơn) + **deployment tool** đang dùng (cephadm, ceph-ansible, Rook, hoặc cài tay) — xem [[Ceph Deployment Models (cephadm, Rook, ceph-ansible)]]
- [ ] Số lượng node và **role phân bổ**: node nào chạy MON, MGR, OSD, RGW, MDS — có tách riêng hay colocate
- [ ] **`ceph -s` hiện tại**: HEALTH_OK hay có WARN/ERR tồn đọng — nếu có, tồn đọng từ bao giờ và vì sao chưa xử lý
- [ ] Danh sách **pool** và scheme của từng pool (replicated size mấy, hay erasure-coded k+m nào) — xem [[Pools, Replication & Erasure Coding]]
- [ ] **Capacity hiện tại**: `ceph df`, và các ngưỡng `nearfull_ratio`/`full_ratio`/`backfillfull_ratio` đang set là bao nhiêu
- [ ] **cephx key nào đang cấp cho CloudStack**, scope thế nào (`client.cloudstack` hay đang dùng `client.admin` — nếu là admin, đây là rủi ro cần xử lý sớm, xem [[Ceph with CloudStack]])
- [ ] **Network topology**: public network và cluster network là VLAN/subnet nào, băng thông thực tế (10G/25G/100G), có bonding không
- [ ] **Backup/DR strategy** hiện tại nếu có (RBD mirroring, snapshot schedule, backup ở tầng nào)
- [ ] Danh sách **sự cố/known issue tồn đọng** (OSD nào hay flap, disk nào sắp hỏng, PG nào từng stuck...)
- [ ] Ai là người có thể hỏi khi bí (đội hạ tầng Ceph, hoặc vendor support nếu có hợp đồng thương mại)

## Công cụ nên cài sẵn trên máy cá nhân / bastion host

```bash
# ceph-common — cung cấp CLI ceph, rbd, rados để thao tác/debug từ xa
apt install ceph-common      # Debian/Ubuntu
dnf install ceph-common      # RHEL/derivatives

# Copy keyring + config để CLI hoạt động (xin từ đội bàn giao, không tự generate)
# /etc/ceph/ceph.conf
# /etc/ceph/ceph.client.admin.keyring  (hoặc user riêng scoped đủ quyền đọc)

ceph -s   # test kết nối
```

> [!tip] Cluster dùng cephadm thì không cần cài gì cả
> Nếu cluster deploy bằng **cephadm** (khuyến nghị cho version hiện tại), bạn có thể thao tác ngay trên node MON mà không cần cài `ceph-common` cục bộ:
> ```bash
> cephadm shell   # mở container có sẵn đầy đủ ceph CLI, tự mount keyring/config đúng
> ```
> Tiện cho việc debug nhanh trên node, nhưng vẫn nên có `ceph-common` trên máy cá nhân/bastion để thao tác từ xa thường xuyên hơn.

## Phiên bản

Ghi chú trong nhóm bài Ceph này viết dựa trên dòng **Ceph Squid (19.x) / Tentacle (20.x)** — các bản hiện hành tại thời điểm viết, dùng `cephadm` làm công cụ triển khai mặc định. `ceph-deploy` đã **deprecated/obsolete** từ lâu, nếu bạn thấy nhắc tới trong tài liệu cũ hoặc runbook cũ của công ty, đó là dấu hiệu tài liệu đã lỗi thời — đừng làm theo nguyên văn.

```bash
ceph version
ceph mgr versions
ceph orch upgrade check   # nếu dùng cephadm — kiểm tra khả năng lên version mới
```

> [!warning] Trước khi chạm vào bất kỳ config nào trên hệ thống bàn giao
> Xác nhận đủ 5 điều sau trước khi thao tác bất cứ gì: (1) **Ceph version chính xác**, (2) **deployment tool** đang dùng (cephadm/ceph-ansible/Rook/thủ công), (3) **replication scheme** đang áp dụng trên từng pool (đặc biệt pool CloudStack đang dùng), (4) **network topology** (public/cluster network tách hay chung, băng thông thật), và (5) **ai là người có quyền admin thật sự** (ai giữ `client.admin` key, ai có quyền SSH root vào MON/OSD node). Năm thông tin này quyết định gần như toàn bộ mức độ an toàn khi bạn debug và thao tác — thiếu bất kỳ cái nào, một lệnh tưởng như vô hại (VD: đổi CRUSH rule, restart nhầm MON đang giữ quorum) có thể gây downtime hoặc mất dữ liệu diện rộng. Xem thêm [[Lessons Learned & Common Pitfalls (Ceph)]].

---
*Xem thêm: [[Ceph|Ceph]] | [[Lessons Learned & Common Pitfalls (Ceph)]]*
