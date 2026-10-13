---
tags:
  - os-template
  - checklist
  - linux
  - ubuntu
  - almalinux
  - debian
---

# OS Template - Checklist Linux (Ubuntu / AlmaLinux / Debian)

Parent: [[OS-Template--Checklist-Chung]]
Related: [[OS-Template--Index]]

> [!NOTE]
> File này chỉ liệt kê **điểm khác biệt/đặc thù riêng** so với [[OS-Template--Checklist-Chung]]. Phải tick đủ checklist chung trước, file này bổ sung thêm phần lệch giữa 3 distro.

## Điểm khác biệt chính

| Hạng mục | Ubuntu (cloud image) | AlmaLinux (cloud image) | Debian (cloud/genericcloud image) |
| --- | --- | --- | --- |
| Package manager | apt | dnf | apt |
| Network renderer mặc định | netplan (backend systemd-networkd hoặc NetworkManager) | NetworkManager (+ `nm-cloud-setup` cho môi trường cloud) | ifupdown (bản cũ) hoặc systemd-networkd (bookworm trở lên) — kiểm tra version cụ thể |
| cloud-init có sẵn | Có sẵn trong official cloud image | Có sẵn trong official cloud image | Có sẵn trong variant **genericcloud**, KHÔNG có trong variant **generic** thường — phải chọn đúng image |
| Grow root partition | `cloud-initramfs-growroot` hoặc `growpart` (gói `cloud-guest-utils`) | `cloud-utils-growpart` | gói `cloud-guest-utils` |
| Firewall mặc định | ufw (thường disable sẵn trong cloud image) | firewalld (enable sẵn — cần mở port cần thiết trước khi seal) | không có firewall enable mặc định |
| SELinux/AppArmor | AppArmor (thường enabled) | SELinux (enforcing mặc định — cần đúng context khi cài thêm service) | AppArmor (tuỳ cấu hình, thường không bật mặc định) |
| Serial console config | GRUB có sẵn `console=ttyS0` trong cloud image | cần xác nhận GRUB config, cloud image RHEL-like thường đã có sẵn | tương tự Ubuntu, kiểm tra lại `GRUB_TERMINAL` |
| Unattended upgrade | `unattended-upgrades` — quyết định bật/tắt (khuyến nghị tắt trong template, quản lý patch qua day-2 ops) | `dnf-automatic` — tương tự, quyết định bật/tắt | `unattended-upgrades` tương tự Ubuntu |

## Checklist riêng từng distro

### Ubuntu

- [ ] Dùng đúng cloud image chính thức từ `cloud-images.ubuntu.com`, không dùng desktop ISO convert tay
- [ ] Netplan config không hardcode tên interface cứng — dùng `match: {name: en*}` thay vì tên cố định, tránh lệch khi clone sang host khác
- [ ] `cloud-init clean --logs --seed` trước khi seal (đã có ở checklist chung, nhắc lại vì hay bị quên riêng với Ubuntu do netplan cache)

### AlmaLinux

- [ ] Dùng đúng cloud image chính thức từ `repo.almalinux.org/almalinux/<version>/cloud`
- [ ] Kiểm tra SELinux context sau khi cài thêm package/service: `restorecon -Rv /path` nếu cần — tránh service fail do context sai mà log không rõ nguyên nhân
- [ ] Enable service `nm-cloud-setup` để NetworkManager tự nhận biết môi trường cloud (gán MTU, routing đúng theo platform)
- [ ] firewalld: mở đúng port cần thiết trước khi seal, không disable hoàn toàn trừ khi security group ở platform đã là lớp bảo vệ chính thức

### Debian

- [ ] Bắt buộc dùng variant **genericcloud** hoặc tối thiểu **generic**, không dùng netinst ISO install tay rồi convert thủ công — thiếu cloud-init/growpart mặc định
- [ ] Debian 12 (bookworm) trở lên dùng cloud-init + systemd-networkd mặc định; bản cũ hơn có thể vẫn dùng ifupdown — xác nhận version trước khi áp dụng bước cấu hình mạng
- [ ] Kiểm tra `/etc/apt/sources.list` trỏ đúng mirror nội bộ nếu tổ chức có mirror riêng, tránh pull từ mirror public khi build hàng loạt

## Reference

- [Ubuntu Cloud Images](https://cloud-images.ubuntu.com/)
- [AlmaLinux Cloud Images](https://almalinux.org/get-almalinux/)
- [Debian Cloud Images](https://cloud.debian.org/images/cloud/)
