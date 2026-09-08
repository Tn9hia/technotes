---
tags:
  - cloudstack
  - ha
  - scalability
---

# CloudStack HA Architecture

CloudStack không có 1 cơ chế HA duy nhất — mà là **tổng hợp HA riêng lẻ ở từng tầng**. Cần hiểu rõ từng tầng để biết chỗ nào hệ thống đang bàn giao có HA thật, chỗ nào chỉ là "tưởng có".

## Bản đồ HA theo tầng

| Tầng | Cơ chế HA | Tự động hay cần cấu hình thêm? |
|---|---|---|
| **Management Server** | Nhiều instance + Load Balancer | Cần chủ động triển khai (không tự có) |
| **Database** | MySQL/MariaDB Galera Cluster | Cần chủ động triển khai — xem [[Database HA - MySQL Galera]] |
| **Host (KVM)** | CloudStack HA: phát hiện host chết (mất heartbeat), tự khởi động lại VM có `offerha=true` trên host khác | Cần bật đúng Service Offering + fencing đáng tin cậy |
| **VM** | HA ở cấp Service Offering (`offerha`) | Không bật mặc định cho mọi offering |
| **System VM (SSVM/CPVM/VR)** | Tự động — CloudStack tự giám sát & recreate | Có sẵn, không cần cấu hình thêm |
| **Network (VR)** | Redundant Virtual Router (tùy chọn) | Cần bật riêng trên Network Offering |
| **Storage** | Phụ thuộc hoàn toàn vào backend (Ceph replication, NFS HA cluster, SAN HA) | CloudStack không tự tạo HA cho storage — trách nhiệm của hạ tầng storage |

> [!warning] Lesson learned: "Hệ thống có HA" là câu nói mơ hồ nguy hiểm
> Khi nhận bàn giao, câu "hệ thống đã có HA" gần như vô nghĩa nếu không hỏi rõ **HA ở tầng nào**. Rất nhiều triển khai có HA cho VM (`offerha=true`) nhưng **chỉ 1 Management Server duy nhất, DB không cluster, NFS Secondary Storage là 1 điểm chết đơn** — nghĩa là VM sống sót khi host chết, nhưng cả hệ thống "mù" (không quản trị được) nếu MS/DB chết. Luôn kiểm tra từng dòng trong bảng trên với hệ thống thực tế thay vì tin vào mô tả chung chung.

## Host HA — cơ chế fencing

Khi 1 KVM Host mất kết nối/heartbeat, CloudStack phải **chắc chắn host đó thực sự chết** (không phải chỉ mất mạng quản lý trong khi VM vẫn chạy) trước khi khởi động lại VM ở nơi khác — nếu không sẽ gây **split-brain: cùng 1 VM chạy 2 nơi, ghi đè dữ liệu lẫn nhau**.

```properties
# Global settings liên quan
ha.tag=                     # host tag dành riêng cho HA (nếu dùng dedicated HA host)
kvm.ha.fence.mode           # cơ chế fencing (tùy plugin/version: IPMI, SSH-based check...)
```

> [!warning] Đây là rủi ro nghiêm trọng nhất trong toàn bộ HA stack
> Nếu fencing không đáng tin cậy (VD: chỉ dựa vào ping ICMP, không có IPMI/out-of-band thật sự), một sự cố mất mạng tạm thời (không phải host chết) có thể khiến CloudStack **khởi động lại VM ở host khác trong khi VM gốc vẫn đang chạy** → 2 VM cùng ghi vào chung 1 volume (nếu storage cho phép) → **hỏng dữ liệu nghiêm trọng**. Đây là khác biệt lớn so với vSphere HA (đã có cơ chế network partition detection khá trưởng thành qua nhiều năm) — với CloudStack, **phải xác nhận rõ cơ chế fencing đang dùng là gì và có đáng tin không** trước khi tin tưởng Host HA sẽ an toàn tuyệt đối.

## Bật HA cho 1 VM

```bash
# Service Offering có offerha=true → VM deploy từ offering đó tự có HA
cmk createServiceOffering name=ha-4c8g offerha=true cpunumber=4 memory=8192

# Kiểm tra VM hiện tại có HA hay không
cmk listVirtualMachines id=<vm-id> | grep -i haenable
```

## Redundant Virtual Router

Xem chi tiết cơ chế và cảnh báo ở [[Virtual Router Deep Dive]] — bật per Network Offering (`redundantroutercapable=true`).

## Checklist đánh giá "độ HA thật" của 1 hệ thống khi nhận bàn giao

- [ ] Có ≥ 2 Management Server sau LB không? LB health-check đúng port chưa?
- [ ] DB có Galera cluster ≥ 3 node không, hay chỉ 1 MySQL instance?
- [ ] Secondary Storage (NFS) có phải điểm chết đơn (single NFS server) không?
- [ ] Cơ chế fencing cho Host HA là gì — có IPMI/BMC thật hay chỉ dựa vào network reachability?
- [ ] Service Offering đang dùng cho VM production có `offerha=true` không?
- [ ] Network quan trọng có Redundant VR không, hay chỉ 1 VR duy nhất?
- [ ] Có test failover thật (tắt thử 1 MS, 1 DB node, 1 host) hay chỉ tin vào thiết kế trên giấy?

---
*Xem thêm: [[Database HA - MySQL Galera]] | [[Scaling the Infrastructure]] | [[CloudStack Troubleshooting]] | [[Cloudstack|CloudStack]]*
