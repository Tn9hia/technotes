---
tags:
  - networking
  - ovs
---

# Open vSwitch Integration

**Open vSwitch (OVS)** là software switch thay thế Linux bridge khi cần tính năng nâng cao mà bridge thuần không có: VLAN trunking linh hoạt, VXLAN/GRE tunnel, OpenFlow (SDN programmable), QoS per-port, mirroring. KVM/libvirt tích hợp OVS bằng cách khai báo `virtualport type='openvswitch'` trong interface XML — VM vẫn dùng TAP device như thường, chỉ khác TAP đó được add vào OVS bridge thay vì Linux bridge.

> [!tip] So với vSphere Distributed Switch (VDS)
> OVS gần với VDS hơn Linux bridge thuần về mặt tính năng — cả 2 đều hỗ trợ port group/VLAN policy tập trung, có thể trải rộng nhiều host (OVS qua tunnel VXLAN, VDS qua vCenter). Sự khác biệt: VDS được quản lý hoàn toàn qua vCenter GUI; OVS quản lý qua CLI (`ovs-vsctl`) hoặc controller SDN riêng (OpenDaylight, OVN, hoặc chính CloudStack/OpenStack Neutron dùng OVS làm backend).

## When — dùng OVS khi nào, không cần khi nào

- **Cần OVS khi**: multi-tenant network isolation phức tạp (nhiều VLAN/VXLAN động theo tenant — đây là lý do CloudStack Advanced Zone và OpenStack Neutron thường chọn OVS làm backend), cần SDN/OpenFlow, cần port mirroring cho monitoring/IDS.
- **Không cần khi**: setup đơn giản, 1-2 VLAN cố định, không cần tunnel — Linux bridge thuần (xem [[Linux Bridge & NAT Networking]]) đủ dùng và ít phức tạp vận hành hơn.

## How — cấu hình cơ bản

```bash
# Tạo OVS bridge
ovs-vsctl add-br ovsbr0
ovs-vsctl add-port ovsbr0 eth0        # gắn NIC vật lý vào OVS bridge

# Xem topology hiện tại
ovs-vsctl show
ovs-vsctl list-ports ovsbr0
```

```xml
<interface type='bridge'>
  <source bridge='ovsbr0'/>
  <virtualport type='openvswitch'>
    <parameters interfaceid='9db3be8a-XXXX-XXXX-XXXX-XXXXXXXXXXXX'/>
  </virtualport>
  <vlan>
    <tag id='100'/>
  </vlan>
  <model type='virtio'/>
</interface>
```

> [!info] `virtualport type='openvswitch'` là điểm khác biệt duy nhất so với Linux bridge trong domain XML
> Về phía QEMU/virtio, không có gì khác — vẫn là TAP device + virtio-net driver như thường. libvirt chỉ thêm bước gọi `ovs-vsctl` để add/remove port vào OVS bridge tương ứng khi VM start/stop, thay vì gọi `brctl`/`ip link` cho Linux bridge. Đây là lý do đổi từ Linux bridge sang OVS **không ảnh hưởng hiệu năng I/O nội bộ VM** — khác biệt chỉ ở tầng switching logic.

## Key Config — VLAN tagging qua OVS

```bash
# Set VLAN access mode cho 1 port cụ thể (thường libvirt tự làm qua <vlan><tag> trong XML)
ovs-vsctl set port <tap-interface> tag=100

# Trunk nhiều VLAN qua 1 port (uplink NIC vật lý)
ovs-vsctl set port eth0 trunks=100,200,300
```

## Ops Runbook

```bash
ovs-vsctl show                          # topology tổng quan
ovs-ofctl dump-flows ovsbr0             # xem flow table (nếu có OpenFlow rule custom)
ovs-appctl fdb/show ovsbr0              # MAC learning table
```

## Gotchas & Lessons Learned

> [!warning] Lesson learned: OVS bridge không tương thích trực tiếp với `brctl`/`ip link` để debug
> Vì OVS quản lý bridge qua kernel module + datapath riêng của nó (`ovs-vswitchd`), các lệnh Linux networking thường (`brctl show`, đôi khi cả `bridge link`) **không hiển thị đúng** thông tin OVS bridge hoặc hiển thị thiếu. Luôn dùng `ovs-vsctl`/`ovs-ofctl`/`ovs-appctl` để debug OVS, đừng áp dụng thói quen debug Linux bridge thường sang OVS rồi kết luận sai "port không tồn tại".

> [!tip] Nếu đang dùng CloudStack Advanced Zone hoặc OpenStack Neutron — OVS thường đã được orchestrator tự quản lý
> Tương tự lưu ý ở [[Libvirt Storage Pools & Volumes]] về storage pool, network OVS trong môi trường có orchestrator thường do orchestrator tự tạo/xóa port theo lifecycle VM — tự tay `ovs-vsctl add-port`/`del-port` trên host đang bị CloudStack/OpenStack quản lý dễ gây lệch state giữa OVS thật và database orchestrator.

## Resources

- Open vSwitch documentation: https://docs.openvswitch.org/
- `man ovs-vsctl`, `man ovs-ofctl`

---
*Xem thêm: [[Linux Bridge & NAT Networking]] | [[Macvtap & SR-IOV Passthrough]] | [[Kvm-virtualization|KVM Virtualization]]*
