---
tags:
  - cloudstack
  - networking
  - virtual-router
---

# Virtual Router Deep Dive

Virtual Router (VR) là **trái tim của networking** trong CloudStack Advanced Networking — hầu hết mọi lỗi mạng của tenant cuối cùng đều dẫn về việc kiểm tra VR.

## VR là gì, thực chất

- Một **VM Debian-based nhỏ** (system VM), tự động tạo khi network đầu tiên trong đó được "implement" (VM đầu tiên deploy vào network), tự động xóa/tạo lại khi cần.
- Bên trong chạy: `dnsmasq` (DHCP/DNS), `iptables` (NAT/Firewall), `haproxy` (Load Balancer nhẹ), `strongswan`/`openswan` (Site-to-Site VPN), `keepalived` (nếu Redundant VR).
- Có tối đa **3 NIC** tùy vai trò: Guest (nội bộ), Public (ra ngoài), Link Local/Control (quản lý từ MS).

> [!tip] So với NSX Edge
> Về mặt chức năng, VR gần giống 1 **NSX Edge Node nhỏ, dùng chung 1 VM cho tất cả service** (routing + NAT + FW + LB + VPN) thay vì NSX tách riêng Tier-0/Tier-1/Edge Services. VR đơn giản hơn nhiều về mặt kiến trúc (không có distributed data plane, không ECMP phức tạp) nhưng cũng vì vậy **là single point of failure** cho cả network đó nếu không bật Redundant VR.

## Vòng đời VR

```
Network được tạo (chưa có VR)
        │
VM đầu tiên deploy vào network → CloudStack "implement" network
        │
VR được tạo tự động trên 1 host trong cluster phù hợp
        │
VR nhận IP Guest (.1), IP Public (nếu SNAT), IP Link Local
        │
VR chạy dnsmasq, cấu hình iptables NAT/FW theo rule hiện có
        │
Mọi thay đổi rule (port forward, FW, LB...) → MS đẩy lệnh cập nhật xuống VR qua SSH/API nội bộ
```

## Redundant Virtual Router (RVR)

- Chạy **2 VR** (Master/Backup) cho 1 network, dùng `keepalived`/VRRP để failover.
- Chỉ nên bật cho network **quan trọng** vì tốn gấp đôi tài nguyên và có độ phức tạp cao hơn khi troubleshoot (2 VR có thể split-brain nếu network quản lý giữa chúng bị lỗi).

```bash
# Bật khi tạo network offering
cmk createNetworkOffering ... redundantroutercapable=true
```

> [!warning] Lesson learned: RVR không phải "cứ bật là an toàn hơn"
> Redundant VR làm giảm rủi ro mất VR do lỗi host/hardware, nhưng **tăng bề mặt lỗi phần mềm** (split-brain giữa Master/Backup, đặc biệt khi mạng quản lý giữa 2 VR chập chờn). Với network không quá quan trọng, 1 VR + giám sát tốt + HA ở tầng host thường đủ và dễ debug hơn. Đừng bật RVR cho tất cả chỉ vì "nghe có vẻ an toàn hơn".

## Thao tác vận hành thường gặp

```bash
# Liệt kê VR
cmk list routers listall=true

# Xem VR đang chạy trên host nào, trạng thái
cmk list routers name=<router-name>

# Restart network (không cleanup) — chỉ đẩy lại config, không destroy VR
cmk restartNetwork id=<network-id> cleanup=false

# Restart network WITH cleanup — destroy & tạo lại VR hoàn toàn (mất kết nối tạm thời!)
cmk restartNetwork id=<network-id> cleanup=true

# SSH trực tiếp vào VR để debug (từ Management Server)
ssh -i /var/lib/cloudstack/management/.ssh/id_rsa -p 3922 <linklocal-ip-cua-VR>

# Bên trong VR — các lệnh hữu ích
cat /var/log/cloud.log
iptables -t nat -L -n -v
ip addr
cat /etc/dnsmasq.conf
```

> [!warning] `restartNetwork cleanup=true` = gây gián đoạn thật sự
> Đây là lệnh "chữa cháy" phổ biến nhất được truyền miệng, nhưng nó **destroy VR cũ và tạo VR mới hoàn toàn** — mọi kết nối NAT/VPN đang active bị cắt, DHCP lease cũ mất (VM có thể phải renew IP). Chỉ dùng khi chắc chắn VR đang lỗi không thể cứu bằng cách nhẹ hơn (`cleanup=false`, hoặc reboot router). Luôn thông báo trước cho tenant/downtime window nếu network đang có traffic production.

## Debug flow khi tenant báo "VM không ra internet được"

```
1. Xác nhận VM có IP đúng, gateway đúng (VM góc nhìn OS)
        │
2. cmk list routers → VR của network đó có "State=Running" không?
        │  Nếu không → reboot/recreate VR
3. SSH vào VR → kiểm tra iptables NAT có rule SNAT cho dải IP guest không
        │
4. Kiểm tra Public IP gắn vào VR còn active không (cmk list publicipaddresses)
        │
5. Kiểm tra Network ACL (nếu VPC tier) hoặc Security Group (nếu Basic/SG network) có block không
        │
6. Kiểm tra physical uplink (VLAN trunk, switch port) của Public traffic có OK không
```

## Cấu hình quan trọng liên quan VR

| Global Setting | Ý nghĩa |
|---|---|
| `router.template.kvm` / `.vmware` / `.xenserver` | Template dùng để tạo VR theo từng hypervisor |
| `network.dhcp.nondefaultnetwork.setgateway` | Có set gateway cho non-default NIC hay không |
| `router.aggregation.command.each.timeout` | Timeout khi MS đẩy loạt lệnh cấu hình xuống VR |
| `network.dns.basiczone.updates` | Cách VR xử lý update DNS trong zone Basic |

---
*Xem thêm: [[System VMs - SSVM, CPVM & Virtual Router]] | [[VPC & Isolated Networks]] | [[CloudStack Troubleshooting]] | [[Cloudstack|CloudStack]]*
