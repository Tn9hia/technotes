# SELinux — Port & Network Labeling
Tier: 2
Parent: [[selinux]]
Related: [[selinux--troubleshooting]], [[selinux--systemd-service-confinement]]
Tags: #selinux #network #port

## What it does

Gắn type lên **port number**, tương tự như file có type. Process muốn `bind()` vào port phải có domain được policy cho phép nối vào type gắn với port đó — không liên quan gì tới firewall (`firewalld`/`iptables`), đây là lớp kiểm soát hoàn toàn khác.

## Why it exists

Nếu không kiểm soát ở mức port, một service bị chiếm quyền có thể tự mở bind lên port bất kỳ (kể cả port của service khác, hoặc port nhạy cảm) miễn DAC cho phép. Gắn type theo port giới hạn: domain nào chỉ được bind đúng những port đã khai cho nó.

## How it works (flow/diagram)

**Xem port đã gắn type gì:**
```bash
semanage port -l | grep http
# http_port_t    tcp   80, 81, 443, 488, 8008, 8009, 8443, 9000 ...
```

**Thêm port mới cho 1 type có sẵn** (VD: chạy httpd ở port lạ 3131):
```bash
semanage port -a -t http_port_t -p tcp 3131
```

**Gỡ:**
```bash
semanage port -d -t http_port_t -p tcp 3131
```

## Config gotchas

- **Triệu chứng kinh điển:** đổi config service sang chạy port khác (VD: nginx từ 80 sang 8090), service log lỗi `bind: Permission denied` dù chạy bằng root và firewall đã mở đúng port. Nguyên nhân gần như luôn là thiếu bước `semanage port -a` — không phải bug ứng dụng, không phải firewall.
- 1 port chỉ được gán cho **đúng 1 type** tại một thời điểm — nếu port đó đã được type khác "chiếm" từ trước (VD: 8080 đã gán `http_cache_port_t` do Squid), phải add đúng port cho type đang cần hoặc xoá gán cũ trước, không thể có 2 type cùng lúc trên cùng port/protocol.
- Đổi port trong `semanage` không tự đồng bộ với firewall — đây là 2 lớp độc lập, phải cấu hình cả hai (`firewalld`/`iptables` mở port + `semanage port` gán type) khi đổi port service.

## Security notes

- Đây là điểm kiểm tra hữu ích khi audit: `semanage port -l` cho biết chính xác domain nào có thể nghe ở đâu, hữu ích hơn nhìn `netstat` (netstat chỉ cho biết port đang mở thực tế, không cho biết port đó có được **phép theo policy** hay không).
- Container runtime (CRI-O/containerd) và Kubernetes NodePort/hostPort cũng đi qua lớp này ở tầng OS nếu container chạy không dùng cơ chế cô lập namespace mạng riêng biệt hoàn toàn — liên quan tới [[selinux--containers-kubernetes]].

## Refs

- [Chapter 4 — Configuring SELinux for applications and services (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/configuring-selinux-for-applications-and-services-with-non-standard-configurations_using-selinux)
