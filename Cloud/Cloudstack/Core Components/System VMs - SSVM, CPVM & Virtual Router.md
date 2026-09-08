---
tags:
  - cloudstack
  - core-components
  - system-vm
---

# System VMs — SSVM, CPVM & Virtual Router

System VM là các **VM đặc biệt do chính CloudStack tạo, quản lý và tự "hồi sinh"** khi cần — không phải VM của người dùng. Đây là khái niệm **hoàn toàn không có bên VMware thuần** (vCenter/ESXi không cần các VM phụ trợ này vì chức năng tương đương đã nằm sẵn trong vCenter/ESXi services).

> [!tip] Vì sao CloudStack cần System VM còn VMware thì không?
> vCenter chạy như 1 service tập trung có sẵn kết nối trực tiếp tới mọi ESXi host qua vSphere API để làm mọi việc (upload ISO, mở console, browse datastore...). Với KVM, CloudStack **không có "vCenter" ở giữa** để làm những việc đó — nên nó tự triển khai các VM nhỏ, đặc biệt, đóng vai trò "cánh tay nối dài" ngay trong hạ tầng KVM để thực hiện các tác vụ này.

## 3 loại System VM chính

| System VM | Vai trò | Số lượng | Tương đương gần đúng bên VMware |
|---|---|---|---|
| **SSVM** (Secondary Storage VM) | Xử lý template/ISO/snapshot: download, copy, convert format giữa các Secondary Storage và Primary Storage | 1 / Zone (có thể nhiều nếu cần HA) | Không có tương đương trực tiếp; gần giống tiến trình ovftool/Content Library sync chạy nền |
| **CPVM** (Console Proxy VM) | Cung cấp console (VNC) truy cập VM qua trình duyệt trong UI | 1 / Zone (scale theo tải) | Gần giống chức năng **VMRC/HTML5 console** nhưng ESXi tự phục vụ trực tiếp, không cần VM trung gian |
| **Virtual Router (VR)** | Router/firewall/DHCP/LB/VPN cho từng network cô lập | 1 / network (Isolated Network hoặc VPC tier), redundant thì x2 | Gần giống **NSX Edge / NSX-T Tier-1 Gateway** thu nhỏ, chạy dạng VM nhỏ (Debian-based) thay vì appliance chuyên dụng |

## Đặc điểm chung của System VM

- Chạy trên **Debian-based template** riêng (`systemvm template`) do CloudStack tự động download khi setup Zone.
- Có IP trên **3 loại network** tùy vai trò: Management (Link Local/Control), Public, Guest.
- Tự động được **CloudStack giám sát và tái tạo (destroy + recreate)** nếu phát hiện lỗi — đừng cố "sửa tay" bên trong rồi kỳ vọng thay đổi tồn tại lâu dài.
- Truy cập được qua SSH bằng **private key mặc định** của CloudStack (dùng để debug), không dùng password.

```bash
# Liệt kê system VM
cmk list systemvms

# SSH vào system VM (từ Management Server, dùng script có sẵn)
/usr/share/cloudstack-common/scripts/vm/systemvm/ssh-privatekey <systemvm-ip>
# hoặc chuẩn hơn:
ssh -i /var/lib/cloudstack/management/.ssh/id_rsa -p 3922 <linklocal-ip>
```

> [!warning] Lesson learned: đừng sửa tay bên trong System VM
> System VM có thể bị CloudStack **destroy & recreate bất cứ lúc nào** (VD: sau khi restart management server, sau khi bạn bấm "Restart Network with cleanup", hoặc khi health check fail). Mọi thay đổi thủ công bên trong (sửa iptables, thêm route tay) sẽ **biến mất không báo trước**. Nếu cần customize hành vi VR lâu dài, phải làm qua Network Offering / router template / hoặc CloudStack config chính thức — không sửa trực tiếp trong VM.

## SSVM — Secondary Storage VM

- Mount cả Secondary Storage (NFS/S3) và có đường truy cập tới Primary Storage để copy template/snapshot qua lại.
- Là điểm nghẽn thường gặp khi **upload template lớn** hoặc **restore snapshot hàng loạt** — vì mặc định chỉ có 1 SSVM/zone xử lý tuần tự.

```bash
# Xem trạng thái SSVM
cmk list systemvms systemvmtype=secondarystoragevm

# Restart SSVM (destroy & recreate) khi bị treo
cmk destroySystemVm id=<ssvm-id>   # CloudStack sẽ tự tạo lại
```

> [!tip] Debug SSVM stuck ở "Starting"
> Nguyên nhân phổ biến nhất: SSVM không mount được NFS Secondary Storage (network/permission issue), hoặc host được chọn để chạy SSVM không kết nối được Secondary Storage qua network đó. Kiểm tra log agent trên host được chọn, và thử `showmount -e <nfs-server>` từ chính host đó.

## CPVM — Console Proxy VM

- Khi user bấm "View Console" trên UI, request được **proxy qua CPVM** (không kết nối thẳng tới hypervisor) — lý do bảo mật, tránh expose VNC port của hypervisor ra ngoài.
- Nếu CPVM chết → toàn bộ tính năng console trên UI mất tạm thời, nhưng **VM vẫn chạy bình thường** (đây không phải outage nghiêm trọng, dễ gây hoảng loạn nhầm).

## Virtual Router — quan trọng nhất trong 3 loại

VR đảm nhiệm gần như toàn bộ network service cho 1 Isolated Network / VPC tier: DHCP, DNS relay, Source NAT, Static NAT, Port Forwarding, Load Balancing (nhẹ), Site-to-Site VPN, Firewall.

Do độ quan trọng và độ phức tạp, VR có note riêng: [[Virtual Router Deep Dive]].

## Bảng lệnh xử lý sự cố nhanh

| Tình huống | Lệnh |
|---|---|
| VR/SSVM/CPVM bị treo, không phản hồi | `cmk rebootRouter id=<id>` hoặc `cmk destroyRouter id=<id>` (tự tạo lại) |
| Cần recreate toàn bộ system VM của 1 network | UI: Network > chọn network > Restart Network **with cleanup** |
| Xem log bên trong VR | SSH vào VR, xem `/var/log/cloud.log`, `/var/log/routerServiceMonitor.log` |
| System VM template thiếu/lỗi khi setup zone mới | Global setting `router.template.kvm` / `secstorage.ssvm.min.cpu.cores`... cần trỏ đúng template đã import |

---
*Xem thêm: [[Virtual Router Deep Dive]] | [[Secondary Storage, Snapshots & Backups]] | [[CloudStack Troubleshooting]] | [[Cloudstack|CloudStack]]*
