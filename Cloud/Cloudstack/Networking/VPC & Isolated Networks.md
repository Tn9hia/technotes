---
tags:
  - cloudstack
  - networking
  - vpc
---

# VPC & Isolated Networks

Chỉ tồn tại trong Zone dùng **Advanced Networking** (xem [[Basic vs Advanced Networking]]).

## Isolated Network (không nằm trong VPC)

- Là 1 mạng L2/L3 riêng biệt, có **1 Virtual Router** phục vụ DHCP, SNAT, Firewall, Port Forwarding, Static NAT, LB cho chính nó.
- Đơn giản nhất: 1 network = 1 VR = 1 dải CIDR nội bộ = 1 (hoặc nhiều) Public IP gắn vào.
- Phù hợp khi **không cần nhiều tier** giao tiếp với nhau qua routing nội bộ có kiểm soát ACL riêng từng tier.

## VPC (Virtual Private Cloud)

VPC cho phép nhiều **Tier** (mỗi tier = 1 network con) cùng tồn tại trong 1 boundary logic, routing với nhau qua **VPC Virtual Router** dùng chung, mỗi tier có ACL riêng.

```
                    Public Network
                          │
                   Public IP (Static NAT / Port Forward / LB)
                          │
                 ┌────────────────┐
                 │  VPC Virtual   │
                 │     Router     │  ← 1 VR dùng chung cho cả VPC
                 └───┬───────┬────┘
                     │       │
          Tier "web" │       │ Tier "db"
          10.0.1.0/24│       │10.0.2.0/24
                     │       │
              VM web-01   VM db-01
              VM web-02   VM db-02
```

| Đặc điểm | Isolated Network | VPC |
|---|---|---|
| Số tier | 1 | Nhiều (tối đa theo giới hạn cấu hình, mặc định thường vài chục) |
| Routing giữa các tier | N/A | Qua VPC VR, có thể chặn/cho phép bằng **Network ACL List** |
| ACL scope | Theo network | Theo từng tier riêng biệt |
| Site-to-Site VPN | Có | Có, dùng chung VPC VR |
| Private Gateway (kết nối trực tiếp ra mạng nội bộ ngoài CloudStack, không qua Public IP) | Không | Có — quan trọng khi cần nối vào mạng doanh nghiệp sẵn có |

> [!tip] So với VMware/NSX & AWS
> VPC trong CloudStack **gần như đúng nghĩa đen "VPC" của AWS**: nhiều subnet (tier), 1 router logic dùng chung, route table qua ACL. Nếu bạn từng làm AWS VPC, mental model gần như y hệt. So với NSX-T, VPC CloudStack tương đương 1 **Tier-1 Gateway với nhiều downlink segment**, còn Private Gateway giống khái niệm **T0 uplink kết nối trực tiếp vào physical/underlay network**.

## Network ACL (khác Security Group!)

- **Network ACL** áp dụng ở cấp **Tier** trong VPC — stateless hoặc stateful tùy rule, kiểm soát traffic ra/vào của cả tier, tương tự **Network ACL của AWS VPC** hoặc **firewall rule trên NSX Tier-1 Gateway**.
- Khác với **Security Group** (áp dụng theo VM/nhóm VM, chỉ có ở Basic Networking hoặc Isolated Network không VPC dùng SG-supported network offering).
- **Không dùng chung 1 khái niệm** — đây là nguồn nhầm lẫn rất lớn, xem thêm [[Security Groups & Network ACLs]].

```bash
# Tạo VPC
cmk createVPC name=my-vpc displaycontext=my-vpc cidr=10.0.0.0/16 vpcofferingid=<offering-id> zoneid=<zone-id>

# Tạo tier (network) trong VPC
cmk createNetwork name=web-tier networkofferingid=<offering-id> \
  zoneid=<zone-id> vpcid=<vpc-id> gateway=10.0.1.1 netmask=255.255.255.0

# Tạo ACL list và rule
cmk createNetworkACLList name=web-acl vpcid=<vpc-id>
cmk createNetworkACL aclid=<acl-id> protocol=TCP startport=443 endport=443 cidrlist=0.0.0.0/0 action=Allow traffictype=Ingress
cmk replaceNetworkACLList aclid=<acl-id> networkid=<tier-id>
```

> [!warning] Lesson learned: tier mới tạo mặc định **deny all** ACL
> Khác thói quen "mặc định cho phép rồi block dần" — 1 tier VPC mới tạo thường gắn vào ACL mặc định **deny toàn bộ traffic** (tùy ACL list default được chọn). Rất nhiều ca "VM mới deploy trong VPC không ping được, không SSH được" chỉ vì quên gán đúng ACL list cho tier, không phải do lỗi hạ tầng.

## Private Gateway — cầu nối ra mạng doanh nghiệp sẵn có

Khi công ty đã có mạng nội bộ (on-prem) và muốn VPC CloudStack route thẳng vào đó (không qua Public IP/NAT), dùng **Private Gateway**: gắn 1 NIC của VPC VR vào 1 VLAN nội bộ cụ thể, khai báo static route hai chiều.

```bash
cmk createPrivateGateway vpcid=<vpc-id> physicalnetworkid=<physnet-id> \
  vlan=<vlan-id> ipaddress=10.99.0.2 gateway=10.99.0.1 netmask=255.255.255.0
cmk createStaticRoute gatewayid=<privategw-id> cidr=192.168.0.0/16
```

---
*Xem thêm: [[Basic vs Advanced Networking]] | [[Virtual Router Deep Dive]] | [[Security Groups & Network ACLs]] | [[Cloudstack|CloudStack]]*
