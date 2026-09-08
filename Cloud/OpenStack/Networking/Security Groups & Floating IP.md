---
tags:
  - openstack
  - neutron
  - security-group
  - floating-ip
---

# Security Groups & Floating IP

## Security Groups

Security Group là **stateful firewall** được implement bằng iptables (OVS) hoặc OVN ACL ở cấp độ VM's port.

### Default behavior
- **Inbound**: DROP tất cả (trừ traffic từ cùng security group)
- **Outbound**: ALLOW tất cả (theo mặc định)
- **Stateful**: reply traffic của established connections được tự động allow

### Create và manage

```bash
# Tạo security group
openstack security group create web-sg \
  --description "Web server security group"

# Rules cho web server
openstack security group rule create web-sg \
  --protocol tcp --dst-port 22 --remote-ip 10.0.0.0/8  # SSH chỉ từ internal

openstack security group rule create web-sg \
  --protocol tcp --dst-port 80 --remote-ip 0.0.0.0/0  # HTTP public

openstack security group rule create web-sg \
  --protocol tcp --dst-port 443 --remote-ip 0.0.0.0/0  # HTTPS public

openstack security group rule create web-sg \
  --protocol icmp --remote-ip 10.0.0.0/8  # Ping từ internal

# Allow từ security group khác (ví dụ: LB → App)
openstack security group rule create app-sg \
  --protocol tcp --dst-port 8080 \
  --remote-group lb-sg  # chỉ cho phép từ sg lb-sg
```

### Security Group Reference (sg-to-sg)

```bash
# Tạo SG cho database, chỉ cho phép từ app-sg
openstack security group create db-sg
openstack security group rule create db-sg \
  --protocol tcp --dst-port 3306 \
  --remote-group app-sg
```

> [!tip] Dùng SG reference thay vì IP ranges
> Khi scale, VMs mới trong app-sg tự động được phép truy cập db-sg mà không cần update rules.

### Xem rules hiện tại

```bash
openstack security group rule list web-sg
openstack security group list

# Xem SG của VM
openstack server show <vm-id> | grep security_groups
```

### Egress rules

```bash
# Mặc định egress ALLOW ALL. Nếu muốn restrict:
# Xóa default egress rules
openstack security group rule delete <default-egress-rule-id>

# Chỉ allow DNS và HTTP ra ngoài
openstack security group rule create web-sg \
  --direction egress \
  --protocol udp --dst-port 53

openstack security group rule create web-sg \
  --direction egress \
  --protocol tcp --dst-port 443
```

## Floating IP

### Khái niệm

```
VM Private IP: 10.0.1.10 (trong tenant network)
Floating IP:   203.0.113.50 (từ provider network pool)

Khi packet đến 203.0.113.50:
  iptables DNAT → 10.0.1.10

Khi VM gửi packet ra:
  iptables SNAT → 203.0.113.50
```

### Lifecycle

```bash
# 1. Tạo floating IP
FIP=$(openstack floating ip create provider-vlan100 -f value -c floating_ip_address)
echo "Floating IP: $FIP"

# 2. Gán cho VM
openstack server add floating ip <vm-id> $FIP

# 3. Xem association
openstack floating ip list
openstack floating ip show $FIP

# 4. Gỡ khỏi VM
openstack server remove floating ip <vm-id> $FIP

# 5. Gán cho VM khác (failover thủ công)
openstack server add floating ip <vm2-id> $FIP

# 6. Xóa hoàn toàn
openstack floating ip delete $FIP
```

### Port floating IP

```bash
# Floating IP gán vào port cụ thể (hữu ích khi VM có nhiều NICs)
PORT_ID=$(openstack port list --server <vm-id> -f value -c ID | head -1)
openstack floating ip set --port $PORT_ID $FIP
```

## Allowed Address Pairs

Cho phép VM nhận traffic đến **nhiều IPs** (VIP/VRRP):

```bash
# VM chạy keepalived với VIP 10.0.1.200
openstack port set \
  --allowed-address-pair ip-address=10.0.1.200 \
  <port-id>

# Wildcard (nguy hiểm, dùng cẩn thận)
openstack port set \
  --allowed-address-pair ip-address=0.0.0.0/0 \
  <port-id>
```

## Port Security

```bash
# Disable port security cho một port (ví dụ: network appliance, router VM)
openstack port set --disable-port-security <port-id>

# Hoặc khi tạo port
openstack port create \
  --network my-net \
  --no-security-group \
  --disable-port-security \
  router-port
```

## Troubleshooting Security Groups

```bash
# Trên compute node: xem iptables rules cho VM
iptables -L -n -v | grep neutron
iptables -L neutron-openvswitch-FORWARD -n -v

# OVS: xem flow rules security group
ovs-ofctl dump-flows br-int | grep ct_state  # connection tracking

# Packet trace (OVS 2.8+)
ovs-appctl ofproto/trace br-int \
  in_port=<port>,tcp,nw_src=10.0.1.1,nw_dst=10.0.1.10,tcp_dst=80
```

---
*Xem thêm: [[Neutron Architecture]] | [[Provider & Tenant Networks]] | [[OVS & OVN]]*
