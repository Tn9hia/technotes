---
tags:
  - cloudstack
  - networking
---

# CloudStack Network Architecture Overview

## Traffic Types

CloudStack chia lưu lượng mạng thành **4 loại traffic**, mỗi loại có thể map vào 1 physical network riêng (hoặc gộp chung nếu hạ tầng nhỏ):

| Traffic Type | Mục đích | Tương đương bên VMware |
|---|---|---|
| **Management** | Giao tiếp giữa Management Server ↔ Host Agent, và giữa các System VM với nhau | vMotion/Management network của ESXi |
| **Public** | IP public, nơi Virtual Router SNAT/DNAT ra internet | Uplink ra ngoài qua NSX Edge/physical router |
| **Guest** | Traffic của VM tenant (thường qua VLAN/VXLAN cô lập) | Port group của VM Network |
| **Storage** (tùy chọn) | Traffic NFS/iSCSI tới Primary/Secondary Storage, tách riêng nếu cần băng thông lớn | VMkernel port cho storage (NFS/iSCSI) |

> [!warning] Nhầm lẫn thường gặp: gộp hết traffic vào 1 NIC rồi không hiểu vì sao chậm
> Nhiều triển khai nhỏ gộp cả 4 loại traffic vào chung 1 physical NIC/bridge để tiết kiệm cổng mạng. Việc này **chạy được** nhưng khi migrate VM lớn hoặc copy template, Storage traffic sẽ cạnh tranh băng thông với Guest traffic của VM đang chạy, gây độ trễ network cho toàn bộ tenant. Bên VMware, tách VMkernel storage riêng là thực hành chuẩn từ lâu — áp dụng tương tự cho CloudStack, tách **ít nhất** Storage và Public ra khỏi Guest nếu có thể.

## Physical Network & Isolation Method

Mỗi Zone có 1 hoặc nhiều **Physical Network** — đại diện cho 1 hạ tầng mạng vật lý cụ thể (1 hoặc nhiều NIC/bond). Mỗi Physical Network chọn 1 **Isolation Method** cho Guest traffic:

| Isolation Method | Cơ chế | Khi nào dùng |
|---|---|---|
| **VLAN** | Mỗi Guest Network = 1 VLAN ID riêng | Phổ biến nhất, đơn giản, giới hạn 4094 VLAN |
| **VXLAN** | Overlay, vượt giới hạn VLAN, cần VTEP | Datacenter lớn, cần > 4094 network |
| **Security Groups only** | Không cô lập L2 bằng VLAN, cô lập bằng firewall rule (Basic Networking) | Zone dùng Basic Networking |
| **GRE / Netris / SDN controllers khác** | Advanced, cần plugin SDN | Triển khai SDN chuyên sâu |

> [!tip] So với VMware NSX
> VLAN isolation method giống hệt việc gán từng port group vào 1 VLAN ID trên vDS — không có gì lạ. VXLAN trong CloudStack tương tự khái niệm VXLAN trong NSX-T nhưng đơn giản hơn nhiều (không có full SDN control plane như NSX Manager, không có distributed routing/firewall tinh vi).

## Network Service Provider

Mỗi network service (DHCP, DNS, Firewall, LB, VPN, Source NAT...) được cung cấp bởi 1 **Network Service Provider**, mặc định là **Virtual Router**, nhưng có thể thay bằng thiết bị vật lý/appliance thật (VD: NetScaler cho LB, Juniper SRX cho firewall, tuỳ config). Đây là cơ chế cho phép CloudStack **tích hợp hardware appliance có sẵn** thay vì luôn dùng VR ảo.

```bash
cmk list networkserviceproviders zoneid=<zone-id>
```

## Kiến trúc Guest Network (mức cao)

```
                    Internet / Public Network
                              │
                    Public IP (gán vào Virtual Router)
                              │
                    ┌─────────────────┐
                    │  Virtual Router │  (Source NAT, Firewall, LB, VPN)
                    └────────┬────────┘
                             │  Guest Network (VLAN/VXLAN riêng)
              ┌──────────────┼──────────────┐
              │              │              │
           VM web-01      VM web-02      VM db-01
```

Có 2 mô hình networking nền tảng khác nhau hoàn toàn ở tầng thiết kế: xem [[Basic vs Advanced Networking]] để chọn đúng mental model trước khi debug bất kỳ vấn đề mạng nào.

## DNS & DHCP

- DHCP cho Guest VM do **Virtual Router** (dnsmasq bên trong VR) cấp phát — không phải DHCP server ngoài, trừ khi bạn cấu hình "External DHCP".
- Mỗi Guest Network có 1 dải IP nội bộ riêng (CIDR khai báo lúc tạo network), VR luôn giữ IP `.1` làm gateway (giống thói quen gateway `.1` phổ biến, nhưng đây là do CloudStack quy ước cứng).

---
*Xem thêm: [[Basic vs Advanced Networking]] | [[Virtual Router Deep Dive]] | [[VPC & Isolated Networks]] | [[Cloudstack|CloudStack]]*
