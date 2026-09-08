---
tags:
  - cloudstack
  - networking
  - security
---

# Security Groups & Network ACLs

Đây là **2 cơ chế firewall hoàn toàn khác nhau** trong CloudStack, dùng ở 2 ngữ cảnh khác nhau — nhầm lẫn giữa chúng là lỗi phổ biến nhất khi mới tiếp cận networking CloudStack.

## Bảng phân biệt

| | Security Group | Network ACL |
|---|---|---|
| Áp dụng cho | VM (theo nhóm/tag), hoạt động ở mức **instance** | Tier/Network trong VPC hoặc Isolated Network, hoạt động ở mức **network** |
| Chỉ dùng được khi | **Basic Networking**, hoặc Isolated Network dùng SG-enabled offering | **Advanced Networking** (Isolated Network / VPC) |
| Cơ chế | Giống AWS EC2 Security Group cổ điển — allow rule theo source CIDR/SG, ingress/egress | Giống AWS VPC Network ACL / NSX Tier Gateway firewall — allow/deny theo CIDR, có traffictype riêng |
| Stateful? | Có (return traffic tự động allow) | Tùy rule (thường cấu hình 2 chiều rõ ràng hơn) |
| Có Deny rule tường minh không | Không — chỉ có Allow, mặc định Deny những gì không match | Có — có thể set explicit Deny |
| Tương đương VMware/NSX | Gần giống **NSX Distributed Firewall theo Security Group/Tag** | Gần giống **firewall rule trên NSX Tier-1 Gateway / AWS Network ACL** |

> [!warning] Không thể dùng cả 2 cùng lúc trên cùng 1 network
> Một Network Offering chỉ chọn **1 trong 2** làm cơ chế bảo mật (`SecurityGroup` service HOẶC networking VPC/Isolated với ACL) — không trộn lẫn. Khi thấy tài liệu/community hướng dẫn dùng Security Group mà zone của bạn đang Advanced Networking + VPC, hướng dẫn đó **không áp dụng được**, phải chuyển sang tư duy Network ACL.

## Security Group — chi tiết

```bash
# Tạo security group
cmk createSecurityGroup name=web-sg

# Thêm rule (cho phép 443 từ bất kỳ đâu)
cmk authorizeSecurityGroupIngress securitygroupname=web-sg \
  protocol=tcp startport=443 endport=443 cidrlist=0.0.0.0/0

# Cho phép SG khác truy cập (thay vì CIDR) — giống "reference security group" của AWS
cmk authorizeSecurityGroupIngress securitygroupname=db-sg \
  protocol=tcp startport=3306 endport=3306 \
  usersecuritygrouplist=web-sg
```

> [!tip] Ưu điểm Security Group ít ai để ý
> Vì hoạt động ở host-level (thường implement bằng iptables/ebtables ngay trên hypervisor, không qua VR), Security Group **không tạo điểm nghẽn qua VR** và scale tốt hơn cho hosting mật độ cao, đơn giản (VD: shared hosting, VPS provider). Đánh đổi là mất khả năng multi-tier/VPC phức tạp.

## Network ACL — chi tiết

Xem ví dụ lệnh chi tiết ở [[VPC & Isolated Networks]]. Điểm cần nhớ thêm:

```bash
# Danh sách ACL đang gắn vào 1 tier
cmk listNetworkACLLists vpcid=<vpc-id>

# Rule trong 1 ACL list được xử lý theo THỨ TỰ (number), giống access-list truyền thống trên router/firewall vật lý
cmk createNetworkACL aclid=<acl-id> number=100 protocol=TCP startport=22 endport=22 cidrlist=10.0.0.0/8 action=Allow
cmk createNetworkACL aclid=<acl-id> number=32767 protocol=all cidrlist=0.0.0.0/0 action=Deny
```

> [!warning] Lesson learned: thứ tự rule (number) quyết định kết quả, không phải "match rule cụ thể nhất"
> Khác với Security Group (mọi rule Allow match đều có hiệu lực, không quan tâm thứ tự), Network ACL xử lý **tuần tự theo `number` tăng dần**, rule đầu tiên match sẽ quyết định — giống ACL trên router Cisco/Juniper truyền thống. Đặt nhầm thứ tự (VD: rule Deny all đặt số nhỏ hơn rule Allow cụ thể) sẽ khiến rule Allow phía sau **không bao giờ được xét tới**.

## Firewall Rule (Isolated Network không VPC) — loại thứ 3 cần phân biệt

Ngoài Security Group và Network ACL, còn có **Firewall Rule** cấp Isolated Network (không nằm trong VPC) — áp dụng cho traffic đi qua Public IP của VR (SNAT/Static NAT), khác cả 2 loại trên.

```bash
cmk createFirewallRule ipaddressid=<public-ip-id> protocol=tcp \
  startport=443 endport=443 cidrlist=0.0.0.0/0
```

> [!tip] Cách nhớ nhanh 3 cơ chế
> - **Security Group** → chặn ở mức **VM**, chỉ Basic Networking hoặc SG-enabled network.
> - **Network ACL** → chặn ở mức **tier/network**, chỉ VPC/Advanced.
> - **Firewall Rule** → chặn traffic đi qua **Public IP của VR** trên Isolated Network (không VPC).
> Ba cái này **độc lập nhau**, một hệ thống Advanced Networking không-VPC thường dùng kết hợp Firewall Rule (ở VR) — còn Security Group gần như không xuất hiện ở đó.

---
*Xem thêm: [[Basic vs Advanced Networking]] | [[VPC & Isolated Networks]] | [[Virtual Router Deep Dive]] | [[CloudStack Security Considerations]] | [[Cloudstack|CloudStack]]*
