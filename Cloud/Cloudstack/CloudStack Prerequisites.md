---
tags:
  - cloudstack
  - prerequisites
---

# CloudStack Prerequisites

Kiến thức nền nên có trước khi nhận bàn giao và vận hành production CloudStack — đặc biệt nếu xuất thân từ VMware.

## Kiến thức bắt buộc

| Mảng | Vì sao cần | Ghi chú cho dân VMware |
|---|---|---|
| **Linux quản trị cơ bản** | Management Server, System VM, và (thường) Host đều chạy Linux | ESXi là hypervisor đóng gói sẵn, không cần biết Linux sâu; KVM thì host **là** một máy Linux đầy đủ — bạn cần biết systemd, networking (bridge/OVS), package manager |
| **KVM/libvirt** | Nếu hypervisor là KVM (phổ biến nhất) | Không có "ESXi" ở giữa — `libvirtd` + `qemu-kvm` đóng vai trò tương tự VMkernel, và bạn thao tác trực tiếp bằng `virsh` khi cần |
| **MySQL/MariaDB** | CloudStack DB lưu toàn bộ state — mất DB = mất control plane | vCenter cũng dùng DB (PostgreSQL embedded/Oracle) nhưng ít khi phải đụng tay trực tiếp; CloudStack DB thường cần thao tác tay nhiều hơn khi debug |
| **Networking L2/L3 cơ bản** | VLAN, bridge, NAT, routing — Virtual Router làm việc này | Tương tự tư duy NSX Edge/T1-T0 nhưng đơn giản hơn nhiều |
| **NFS / Ceph / iSCSI** | Phần lớn Primary/Secondary Storage dùng các giao thức này | Tương tự việc biết VMFS/NFS datastore, nhưng CloudStack thường "trần trụi" hơn — ít abstraction, dễ thấy tận gốc vấn đề |
| **REST API cơ bản** | Toàn bộ UI/CLI đều gọi qua CloudStack API (HMAC-signed) | Giống PowerCLI/vSphere API nhưng CloudStack UI **là** một API client thuần, không có gì "ẩn" thêm |

## Checklist thông tin cần xin khi nhận bàn giao

> [!tip] Hỏi trước, đừng đoán
> Khi nhận bàn giao, hãy chủ động xin đủ các thông tin sau — thiếu bất kỳ cái nào cũng khiến bạn debug mù trong lần sự cố đầu tiên.

- [ ] Version CloudStack chính xác + hypervisor + version hypervisor (KVM: `qemu-kvm`, `libvirt` version)
- [ ] Số lượng Management Server, có LB phía trước không, có HA DB (Galera) không
- [ ] Networking mode: Basic hay Advanced (có VPC không), Isolation method (VLAN/VXLAN)
- [ ] Danh sách Zone/Pod/Cluster/Host và mapping vật lý (rack, switch, uplink)
- [ ] Primary Storage backend (NFS server nào, Ceph cluster nào, dung lượng còn lại)
- [ ] Secondary Storage backend + nơi backup lưu trữ
- [ ] Danh sách Account/Domain/Project quan trọng và ai đang dùng
- [ ] Global Settings đã tùy chỉnh khác default (rất hay bị quên khi bàn giao miệng)
- [ ] SSH key / root access vào Management Server và Host
- [ ] Quy trình backup DB hiện tại, tần suất, vị trí lưu
- [ ] Có đang có sự cố/known issue nào tồn đọng không (VM stuck, VR lỗi, storage gần đầy...)
- [ ] Ai là người có thể hỏi khi bí (Slack/email của người bàn giao)

## Công cụ nên cài sẵn trên máy cá nhân

```bash
# CloudMonkey — CLI chính thức
pip install cloudmonkey
# hoặc tải binary từ Apache CloudStack releases

# Test API bằng curl cũng nên biết cách ký HMAC (xem CLI & API - CloudMonkey)
```

---
*Xem thêm: [[Cloudstack|CloudStack]] | [[Lessons Learned & Common Pitfalls]]*
