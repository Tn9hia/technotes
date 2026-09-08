---
tags:
  - cloudstack
  - operations
  - lessons-learned
---

# Lessons Learned & Common Pitfalls

Tổng hợp lại toàn bộ "lesson learned" rải rác trong các note khác vào 1 chỗ — dùng làm checklist tra cứu nhanh trước khi thao tác trên hệ thống production, đặc biệt hữu ích trong giai đoạn đầu mới nhận bàn giao.

## Top nhầm lẫn khái niệm (do quen tư duy VMware)

| Nhầm lẫn | Sự thật | Chi tiết |
|---|---|---|
| Nghĩ CloudStack thay thế vCenter khi dùng VMware hypervisor | CloudStack chỉ orchestrate qua vCenter API, **vCenter vẫn phải tồn tại và chạy** | [[Hypervisor Support - KVM, VMware & Others]] |
| Nhầm Pod với Cluster | Pod = ranh giới mạng quản lý, Cluster = ranh giới storage + migration | [[Zones, Pods, Clusters & Hosts]] |
| Nhầm Account với User | Account là đơn vị sở hữu resource (như 1 tenant), nhiều User có thể chung 1 Account | [[Accounts, Domains & Projects (CloudStack)]] |
| Nhầm Volume Snapshot với VM Snapshot | Volume Snapshot chỉ chụp 1 disk, không đồng bộ multi-disk như VM Snapshot | [[Secondary Storage, Snapshots & Backups]] |
| Nhầm Security Group với Network ACL | 2 cơ chế độc lập, dùng ở 2 network mode khác nhau, không trộn lẫn | [[Security Groups & Network ACLs]] |
| Tưởng Maintenance Mode tự động di chuyển VM như DRS | Chỉ cố live-migrate trong cluster, có thể treo nếu thiếu capacity hoặc dùng local storage | [[Zones, Pods, Clusters & Hosts]] |

## Top sự cố vận hành cần cảnh giác

1. **`cluster.node.IP` sai khi thêm Management Server mới** → job kẹt hàng loạt không rõ nguyên nhân. ([[CloudStack Management Server]])
2. **Bootstrap Galera từ node không phải seqno cao nhất** sau khi toàn cụm DB cùng down → mất dữ liệu mới nhất vĩnh viễn. ([[Database HA - MySQL Galera]])
3. **Xoay vòng Ceph cephx key mà không đồng bộ libvirt secret trên host** → thao tác storage mới fail âm thầm dù VM cũ vẫn chạy bình thường. ([[Primary Storage Backends]])
4. **`restartNetwork cleanup=true` dùng như thói quen "chữa cháy"** → destroy & tạo lại VR hoàn toàn, cắt kết nối NAT/VPN đang active, không phải thao tác nhẹ nhàng. ([[Virtual Router Deep Dive]])
5. **Fencing không đáng tin cậy cho Host HA** → nguy cơ split-brain, 1 VM chạy 2 nơi cùng lúc, hỏng dữ liệu. ([[CloudStack HA Architecture]])
6. **`forced=true` khi stop VM** chỉ sửa state ở DB, không đảm bảo hypervisor đã dừng thật → nguy cơ tương tự split-brain nếu start lại VM ở host khác. ([[CloudStack Day 2 Operations]])
7. **Nhiều Management Server cùng chạy DB schema migration đồng thời khi upgrade** → nguy cơ hỏng DB không phục hồi được nếu thiếu backup. ([[CloudStack Upgrade Procedure]])
8. **NFS export thiếu quyền cho toàn dải IP cluster** (chỉ mở cho IP host đầu tiên) → cluster báo Alert hàng loạt khi host thứ 2/3 join. ([[Primary Storage Backends]])
9. **Regenerate API key của account đang dùng cho automation** → gãy pipeline/Terraform/Ansible ngay lập tức không báo trước. ([[RBAC & Roles (CloudStack)]])
10. **Quên import System VM Template trước khi setup Zone** → cấu hình xong Zone/Pod/Cluster nhưng không deploy được VM nào, SSVM/CPVM/VR không bao giờ tạo được. ([[CloudStack Installation Methods]])
11. **Thiết kế dải IP Pod quá chật ngay từ đầu** → không mở rộng thêm host được sau này. ([[Scaling the Infrastructure]])
12. **Trộn CPU đời khác nhau trong cùng Cluster mà không set baseline CPU model** → live migration crash VM ngẫu nhiên giữa 1 số cặp host. ([[Scaling the Infrastructure]])
13. **Tier VPC mới tạo mặc định deny-all ACL** → VM mới deploy "không lý do" không ping/SSH được. ([[VPC & Isolated Networks]])
14. **Storage Tag/Host Tag không khớp** → lỗi "No suitable host found" dù cluster còn dư tài nguyên rất nhiều. ([[CloudStack Troubleshooting]])
15. **Usage Server (`cloudstack-usage`) chết âm thầm** → không ảnh hưởng VM, nhưng ảnh hưởng billing, chỉ phát hiện cuối kỳ. ([[CloudStack Monitoring & Alerting]])

## Nguyên tắc vận hành an toàn (rút ra chung)

> [!tip] 5 nguyên tắc nên khắc cốt ghi tâm
> 1. **Đừng sửa tay bên trong System VM** — mọi thay đổi có thể biến mất khi CloudStack tự tái tạo VM đó.
> 2. **Luôn backup DB trước bất kỳ thao tác lớn nào** (upgrade, đổi Global Setting quan trọng, bootstrap Galera) — DB là "não" của hệ thống, không có cơ chế rollback tự động.
> 3. **Luôn thao tác tuần tự (rolling), không đồng loạt** — patch host, upgrade MS, restart network đều nên làm từng cái một, theo dõi ổn định rồi mới tiếp tục.
> 4. **Phân biệt rõ layer trước khi debug** — control plane (MS/DB/API) và data plane (Host/VR/Storage) độc lập với nhau, VM sống hay chết không phụ thuộc MS.
> 5. **Verify bằng lệnh, đừng tin mô tả miệng** — "hệ thống có HA", "đã backup", "chắc port đó mở rồi" đều cần được xác nhận bằng lệnh cụ thể (`cmk`, `virsh`, `mysql`, `ceph`) trước khi tin tưởng.

## Câu hỏi nên tự đặt ra định kỳ (không chỉ lúc nhận bàn giao)

- Nếu Management Server chết ngay bây giờ, tôi có thể phục hồi trong bao lâu?
- Nếu 1 Galera node chết, tôi có biết chính xác cần bootstrap lại như thế nào không, và từ node nào?
- Nếu Secondary Storage đầy vào lúc 2h sáng, ai/cái gì sẽ báo cho tôi biết trước khi user phát hiện?
- Global Settings hiện tại có bao nhiêu cái khác default, và tôi có hiểu hết lý do vì sao không?
- Lần cuối cùng tôi (hoặc đội cũ) thử restore DB từ backup thật sự là khi nào?

---
*Xem thêm: [[CloudStack Prerequisites]] | [[CloudStack Troubleshooting]] | [[CloudStack HA Architecture]] | [[Cloudstack|CloudStack]]*
