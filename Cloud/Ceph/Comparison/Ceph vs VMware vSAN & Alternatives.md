---
tags:
  - ceph
  - vsan
  - comparison
  - storage
---

# Ceph vs VMware vSAN & Alternatives

Trang tổng hợp so sánh dành riêng cho người có nền tảng **vSAN** chuyển sang vận hành **Ceph**. Mục tiêu không phải "cái nào tốt hơn" mà là **map đúng khái niệm** để rút ngắn đường cong học tập, và đưa ra góc nhìn thực tế — kể cả những điểm Ceph **thua** vSAN — để bạn không bị sốc khi vận hành thật.

> [!tip] So với VMware vSAN
> Khác biệt triết lý lớn nhất: vSAN là **HCI tích hợp sẵn trong vSphere** — 1 sản phẩm, 1 vendor, 1 UX (vCenter). Ceph là **hệ phân tán độc lập hoàn toàn với hypervisor** — bạn tự ráp cluster, tự chọn deployment tool, tự tích hợp với lớp compute (CloudStack/OpenStack/KVM trần/Kubernetes...). Đây vừa là sức mạnh (không khoá vendor) vừa là gánh nặng (không ai "lo hộ" bạn) của Ceph.

## Bản đồ khái niệm: Ceph ↔ vSAN

| Ceph | Gần nhất bên vSAN | Khác biệt cốt lõi |
|---|---|---|
| [[MON - Monitor]] | Không có tương đương trực tiếp — gần nhất là vai trò "nguồn sự thật" của vCenter | MON là **cụm quorum-based** (Paxos) độc lập, tự nó là 1 hệ phân tán nhỏ có thể chịu mất thiểu số node; vCenter là 1 điểm quản lý (dù có vCenter HA) không tham gia trực tiếp vào I/O path |
| [[MGR - Manager]] | vSAN Health Service / một phần vROps | MGR chạy module (dashboard, Prometheus exporter, balancer...) như 1 nền tảng plugin; vSAN Health là tính năng đóng gói sẵn, không mở rộng được |
| [[OSD - Object Storage Daemon]] | Ổ đĩa trong 1 disk group | Ceph: **1 daemon quản lý 1 disk** (thường), tách biệt hoàn toàn — hạt mịn hơn nhiều so với disk group (thường gồm 1 cache + nhiều capacity disk dùng chung) |
| [[CRUSH Algorithm & CRUSH Map]] | CLOM (Cluster Level Object Manager) + DOM (Distributed Object Manager) | CRUSH là thuật toán **client-side, tính toán trực tiếp** vị trí object (không tra bảng); CLOM/DOM của vSAN quản lý placement tập trung hơn qua metadata |
| [[Pools, Replication & Erasure Coding|Pools]] + [[Placement Groups (PG)]] | Storage Policy (FTT, RAID-1/5/6, stripe width) | Storage Policy áp per-VM/per-object; Pool+PG là lớp hạ tầng logic **không có tương đương** bên vSAN — PG là khái niệm hoàn toàn mới cần học |
| [[RBD - Block Storage]] | VMDK trên VMFS/vSAN datastore | Tương đồng nhất về vai trò (block device cho VM), nhưng RBD expose thẳng qua `librbd`/kernel module, không qua filesystem cluster như VMFS |
| [[CephFS - File Storage]] | vSAN File Service | CephFS trưởng thành hơn nhiều, hỗ trợ POSIX đầy đủ, multi-MDS scale-out; vSAN File Service ra sau, giới hạn hơn về scale và tính năng |
| [[RGW - Object Storage Gateway]] | Không có tương đương | vSAN không có storage object/S3 built-in; đây là khả năng độc quyền của Ceph trong nhóm HCI/SDS truyền thống |
| [[Pools, Replication & Erasure Coding|Replication & Erasure Coding]] | FTT (Failure To Tolerate) dạng RAID-5/6 | Cùng ý tưởng erasure coding, nhưng Ceph cho phép **tuỳ biến profile k+m** linh hoạt hơn, áp theo từng pool riêng biệt |
| cephx | Không có tương đương trực tiếp | vSAN thừa hưởng authentication của vSphere/vCenter (SSO); Ceph có **hệ thống auth riêng** (cephx) độc lập với lớp compute — cần quản lý user/key riêng |
| [[Ceph Deployment Models (cephadm, Rook, ceph-ansible)]] | vLCM (vSphere Lifecycle Manager) + vai trò deploy của vCenter | vLCM quản lý lifecycle tích hợp sẵn trong vCenter; cephadm/Rook là công cụ **riêng biệt**, không tích hợp vào lớp compute |

## So sánh ưu/nhược điểm thẳng thắn

| Tiêu chí | Ceph | VMware vSAN |
|---|---|---|
| **Khoá hypervisor** | Hypervisor-agnostic — dùng được với KVM (CloudStack/OpenStack), bare-metal, Kubernetes (Rook) | Chỉ dùng được với vSphere/ESXi — khoá chặt vào hệ sinh thái VMware/Broadcom |
| **Đa dạng giao thức** | 1 cluster cấp cả **block (RBD) + file (CephFS) + object (RGW)** | Chủ yếu block (VMDK); File Service có nhưng mới hơn, giới hạn hơn về scale/tính năng; không có object/S3 native |
| **Độ phức tạp vận hành** | Cao — nhiều daemon, nhiều khái niệm mới (CRUSH, PG, cephx...), cần tự tune nhiều | Thấp hơn nhiều — tích hợp chặt vào vCenter UI, phần lớn tự động hoá, ít bề mặt cấu hình tay |
| **Mô hình chi phí** | Mã nguồn mở, không license theo node/CPU (chi phí chính là phần cứng + nhân sự vận hành) | License theo CPU/node (vSAN), cộng thêm vSphere license — chi phí tăng tuyến tính theo cluster |
| **Mô hình hỗ trợ** | Cộng đồng miễn phí + thương mại tuỳ chọn (Red Hat/IBM Ceph Storage, Croit, 45Drives, SUSE...) | Enterprise support trực tiếp từ VMware/Broadcom, SLA rõ ràng |
| **Khả năng mở rộng quy mô** | Scale tới hàng nghìn OSD, chi phí mở rộng khá tuyến tính (thêm node/disk) | Cluster vSAN thường giới hạn thực tế nhỏ hơn nhiều (khuyến nghị/giới hạn kỹ thuật theo host count), chi phí license tăng theo |
| **Tính linh hoạt phần cứng** | Chạy trên phần cứng commodity rộng rãi, không yêu cầu HCL nghiêm ngặt (dù vẫn nên kiểm tra tương thích) | Ràng buộc bởi vSAN ReadyNode / VMware HCL — chọn sai phần cứng ngoài HCL rủi ro không được hỗ trợ |
| **Tính linh hoạt failure domain** | CRUSH map cho phép định nghĩa hierarchy **tuỳ ý** (host/rack/row/datacenter/region...) | Fault domain của vSAN đơn giản hơn, chủ yếu theo host/rack, ít tuỳ biến hierarchy sâu |

> [!tip] So với VMware vSAN
> Nếu bạn quen đánh giá storage qua "IOPS/latency per VM" kiểu vSAN dashboard, hãy chuẩn bị tinh thần Ceph đòi hỏi bạn tự ráp bức tranh đó từ nhiều nguồn (`ceph osd perf`, Prometheus exporter, `rbd perf image iostat`...) — không có 1 dashboard "ra lò sẵn" toàn diện như vSAN Performance Service, trừ khi bạn tự dựng Grafana/Ceph Dashboard đầy đủ.

## So với các lựa chọn khác (ngoài vSAN)

| Lựa chọn | Vị trí trong bức tranh | Khi nào hợp lý |
|---|---|---|
| **NFS/iSCSI SAN truyền thống** | Đơn giản nhất để setup với CloudStack/OpenStack (xem [[Primary Storage Backends]]) | Team nhỏ, không có nhân sự storage ops chuyên trách, đã có sẵn SAN từ hạ tầng cũ; HA/scale phụ thuộc hoàn toàn vào chính SAN đó (thường là 1 điểm phụ thuộc trừ khi SAN tự cluster) |
| **Local Storage** | Nhanh nhất về raw IOPS, không có HA/migration | Đã bàn kỹ ở [[Primary Storage Backends]] (vault CloudStack) — chỉ dùng cho workload chấp nhận mất dữ liệu khi mất host, hoặc có replication ở tầng ứng dụng |
| **StorPool** | SDS thương mại, tối ưu rất mạnh cho hiệu năng/latency thấp, có official CloudStack plugin | Khi cần hiệu năng cực cao và sẵn sàng trả phí license + hỗ trợ chuyên biệt, đổi lại giảm gánh vận hành so với tự quản Ceph |
| **VxFlex OS / PowerFlex (Dell)** | SDS thương mại quy mô lớn, mạnh về consistency và enterprise support | Doanh nghiệp lớn đã trong hệ sinh thái Dell, ưu tiên support enterprise hơn chi phí license |
| **LINSTOR/DRBD** | SDS nhẹ hơn, phù hợp cluster nhỏ-vừa, mô hình replication đơn giản hơn Ceph | Cluster nhỏ cần block storage HA mà không muốn gánh độ phức tạp vận hành của Ceph |

Trong nhóm SDS mã nguồn mở, **Ceph vẫn là lựa chọn phổ biến nhất cho CloudStack/OpenStack/KVM** — không hẳn vì công nghệ vượt trội tuyệt đối, mà vì hệ sinh thái tích hợp **first-class**: driver RBD trưởng thành trong cả CloudStack lẫn OpenStack (Cinder/Nova), cộng đồng lớn, tài liệu phong phú, và khả năng dùng chung 1 cluster cho cả block/file/object giúp giảm số lượng hệ thống storage phải vận hành song song.

## Khi nào nên chọn Ceph, khi nào nên cân nhắc khác

**Ceph phù hợp khi:** bạn cần storage **hypervisor-agnostic** (đặc biệt nếu dùng KVM qua CloudStack/OpenStack, hoặc kết hợp cả VM lẫn Kubernetes/Rook), muốn tránh khoá vendor và chi phí license theo node, cần **cả 3 giao thức** (block+file+object) từ 1 hạ tầng, và đang vận hành ở quy mô đủ lớn để lợi ích kỹ thuật (linh hoạt CRUSH, scale gần tuyến tính) vượt qua chi phí vận hành ban đầu.

**Nên cân nhắc phương án khác khi:** cluster nhỏ (dưới ~5-6 node vật lý dành cho storage), hoặc team **không có nhân sự chuyên trách storage ops** — lúc đó độ phức tạp vận hành của Ceph (MON quorum, CRUSH tuning, PG autoscaler, xử lý recovery/backfill...) có thể trở thành rủi ro vận hành lớn hơn lợi ích nó mang lại. Một NFS/SAN đơn giản, hoặc 1 SDS thương mại "appliance-like" hơn (StorPool, PowerFlex) với support đi kèm, thường **giảm rủi ro vận hành thực tế** hơn là tự vận hành Ceph mà không đủ năng lực đội ngũ.

> [!warning] Lesson learned: đánh giá thấp chi phí nhân sự khi rời vSAN sang Ceph
> Nhiều team chuyển từ VMware/vSAN sang Ceph (thường vì lý do license/chi phí sau các đợt tăng giá của Broadcom) chỉ so sánh **chi phí license phần mềm**, mà bỏ qua chi phí **năng lực vận hành**. vSAN gần như "appliance" — cắm ổ, join cluster qua vCenter, hầu hết phần còn lại tự động. Ceph đòi hỏi đội ngũ hiểu CRUSH, PG, quy trình add/remove OSD an toàn, tuning recovery throttle, xử lý `HEALTH_WARN` đúng cách — những kỹ năng **không tự nhiên có sẵn** ở một team quen thao tác qua vCenter GUI. Kết quả thực tế ở không ít nơi: cluster Ceph chạy ổn giai đoạn đầu (ít tải, ít sự cố), rồi lúng túng nghiêm trọng ở sự cố **đầu tiên** thật sự (OSD hỏng hàng loạt, MON mất quorum...) vì đội vận hành chưa từng thực hành runbook trong điều kiện thực. Nếu quyết định chuyển sang Ceph, hãy **đầu tư training + chạy thử kịch bản sự cố (game day) trước khi go-live**, đừng đợi sự cố thật dạy bạn — xem [[Ceph HA Architecture]] và [[Lessons Learned & Common Pitfalls (Ceph)]].

---
*Xem thêm: [[Ceph Prerequisites]] | [[Ceph HA Architecture]] | [[CloudStack vs VMware vs OpenStack]] | [[Primary Storage Backends]] | [[Ceph|Ceph]]*
