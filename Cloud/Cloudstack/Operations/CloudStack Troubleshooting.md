---
tags:
  - cloudstack
  - troubleshooting
  - operations
  - debug
---

# CloudStack Troubleshooting

## Nguyên tắc debug chung

```
1. Xác định layer nghi vấn: Control plane (MS/DB) hay Data plane (Host/Network/Storage)?
   → VM đang chạy vẫn sống khi MS chết, nên "VM chạy chậm/lỗi" hiếm khi do MS,
     còn "không tạo/sửa/xóa được gì" gần như chắc chắn do MS/DB/Agent.
2. Xác định đúng hypervisor (KVM/VMware/khác) — cách debug khác nhau hoàn toàn
3. Lấy đúng ID liên quan (vm-id, network-id, job-id) trước khi tra log
4. Trace theo thứ tự: Management log → Agent log (nếu KVM) → libvirt/qemu log → System VM log (nếu liên quan network)
```

## Log Files Reference

| Thành phần | Log path |
|---|---|
| Management Server | `/var/log/cloudstack/management/management-server.log` |
| Usage Server | `/var/log/cloudstack/usage/usage.log` |
| KVM Agent (trên host) | `/var/log/cloudstack/agent/agent.log` |
| Libvirt (trên host) | `/var/log/libvirt/libvirtd.log` |
| QEMU (log riêng từng VM, trên host) | `/var/log/libvirt/qemu/i-<account>-<vm-id>-<vm-name>.log` |
| Virtual Router / SSVM / CPVM (bên trong system VM) | `/var/log/cloud.log`, `/var/log/routerServiceMonitor.log` |
| MySQL/Galera | `/var/log/mysql/error.log` |

## VM không deploy được ("No suitable host found" / lỗi tương tự)

```bash
# 1. Xem chi tiết job lỗi
cmk query asyncjobresult jobid=<job-id>

# 2. Kiểm tra capacity cluster có đủ CPU/RAM theo Service Offering không
cmk list capacity clusterid=<cluster-id>

# 3. Kiểm tra Storage Tag của Disk Offering có khớp Primary Storage nào không
cmk list storagepools clusterid=<cluster-id>

# 4. Xem log Management Server tìm chính xác lý do allocator từ chối host
grep "<vm-id>\|No suitable" /var/log/cloudstack/management/management-server.log
```

> [!tip] "No suitable host" thường do storage tag/host tag, không phải thiếu tài nguyên
> Rất nhiều ca báo lỗi này dù cluster còn dư CPU/RAM rất nhiều — nguyên nhân thật thường là **Storage Tag** trên Disk Offering không khớp bất kỳ Primary Storage nào trong cluster, hoặc **Host Tag** trên Service Offering không khớp host nào. Kiểm tra tag trước khi nghi ngờ thiếu tài nguyên vật lý.

## VM Stuck ở "Starting" / "Migrating" quá lâu

```bash
# Trên host được chọn — kiểm tra libvirt trực tiếp
virsh list --all | grep i-
virsh domstate <instance-name>

# Xem qemu log của chính VM đó
tail -100 /var/log/libvirt/qemu/<instance-name>.log

# Kiểm tra agent trên host có đang kết nối MS bình thường không
systemctl status cloudstack-agent
tail -f /var/log/cloudstack/agent/agent.log
```

## Network Issues

### VM không lấy được IP (DHCP)

```bash
# 1. Kiểm tra VR của network đó có đang Running không
cmk list routers listall=true

# 2. SSH vào VR, kiểm tra dnsmasq
ssh -p 3922 -i /var/lib/cloudstack/management/.ssh/id_rsa <vr-linklocal-ip>
ps aux | grep dnsmasq
cat /etc/dnsmasq.conf
tail -f /var/log/cloud.log
```

### VM không ra internet được

Theo đúng flow debug chi tiết ở [[Virtual Router Deep Dive]] — kiểm tra VR state → iptables SNAT → Public IP → ACL/Security Group/Firewall Rule → physical uplink.

### Floating/Public IP không hoạt động (Static NAT/Port Forwarding)

```bash
cmk list publicipaddresses ipaddress=<ip>
# SSH vào VR, kiểm tra iptables DNAT
ssh ... <vr-ip>
iptables -t nat -L -n -v | grep <private-ip>
```

## Storage Issues

### Volume stuck "Creating"/"Migrating"

```bash
# Kiểm tra SSVM có Running không (nếu liên quan copy từ secondary storage)
cmk list systemvms systemvmtype=secondarystoragevm

# Xem log SSVM
ssh ... <ssvm-ip>
tail -100 /var/log/cloud.log

# Nếu Ceph backend — kiểm tra tầng Ceph
ceph health detail
rbd ls -p <pool-name>
```

### Host báo "Alert" liên quan Primary Storage

```bash
# Kiểm tra host có mount được NFS/Ceph pool không
mount | grep nfs   # với NFS
rbd -p <pool> ls    # với Ceph, chạy trên host

# Xem agent log để biết lỗi kết nối cụ thể
tail -f /var/log/cloudstack/agent/agent.log
```

## Management Server / API Issues

### 401 / 531 Unable to verify user credentials

Xem chi tiết cách ký signature ở [[CLI & API - CloudMonkey]] — 90% nguyên nhân là lỗi thứ tự sort/lowercase/encode khi tự viết integration, không phải sai key.

### Job bị kẹt "pending" hàng loạt

```bash
# Kiểm tra bảng mshost — MS node đã chết còn giữ "ownership" của job không
mysql -u cloud -p cloud -e "SELECT id, service_ip, state, last_update FROM mshost;"

# Kiểm tra async job đang treo
cmk list asyncjobs listall=true

# Nếu xác nhận MS node đã chết vĩnh viễn, cần dọn bảng mshost (thận trọng, backup DB trước!)
```

> [!warning] Xem thêm nguyên nhân gốc thường gặp ở [[CloudStack Management Server]]
> `cluster.node.IP` cấu hình sai giữa các MS là nguyên nhân điển hình gây hiện tượng job kẹt hàng loạt không rõ lý do.

## Bảng lỗi thường gặp

| Lỗi | Nguyên nhân | Hướng xử lý |
|---|---|---|
| `No suitable host found` | Storage/host tag không khớp, hoặc thật sự hết capacity | Kiểm tra tag trước, capacity sau |
| `Unable to verify user credentials` (531) | Lỗi ký HMAC signature hoặc key sai | Xem lại thuật toán ký ở [[CLI & API - CloudMonkey]] |
| `Resource limit exceeded` | Vượt quota Account/Domain | `cmk listResourceLimits`, tăng quota nếu hợp lệ |
| `Insufficient capacity` | Cluster/Pod hết CPU/RAM/IP | Kiểm tra `cmk list capacity`, cân nhắc thêm host/mở rộng Pod IP |
| VM `Error` state sau khi start fail | Thường do lỗi hypervisor/libvirt cụ thể | Xem qemu log trên host, `virsh dominfo` |
| Network "Alert" liên tục | VR không phản hồi health check | `cmk rebootRouter` hoặc `restartNetwork cleanup=true` (cân nhắc downtime) |

---
*Xem thêm: [[CloudStack Day 2 Operations]] | [[CloudStack Monitoring & Alerting]] | [[Lessons Learned & Common Pitfalls]] | [[Cloudstack|CloudStack]]*
