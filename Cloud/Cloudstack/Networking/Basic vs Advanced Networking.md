---
tags:
  - cloudstack
  - networking
---

# Basic vs Advanced Networking

Đây là **quyết định thiết kế nền tảng nhất** của một Zone CloudStack — chọn sai (hoặc không biết Zone hiện tại đang chạy mode nào) sẽ khiến mọi thao tác network sau này đi sai hướng hoàn toàn. **Không thể đổi qua lại giữa 2 mode sau khi Zone đã tạo** (phải tạo Zone mới).

## So sánh tổng quan

| Tiêu chí | Basic Networking | Advanced Networking |
|---|---|---|
| Mô hình mạng | Flat, mọi VM chung 1 dải L2 lớn (giống 1 "security group cloud" kiểu EC2-Classic cũ) | Mỗi tenant/network có VLAN/VXLAN riêng, cô lập L2/L3 thật sự |
| Cô lập giữa các VM/tenant | Bằng **Security Group** (rule theo IP/port, không cô lập L2) | Bằng **VLAN + Virtual Router** (cô lập L2 lẫn L3) |
| Virtual Router | Không có VR riêng cho từng mạng (chỉ có VR chung cấp DHCP) | Mỗi Isolated Network / VPC tier có VR riêng |
| Hỗ trợ VPC | Không | Có |
| Static NAT / Port Forwarding / Site-to-Site VPN | Hạn chế | Đầy đủ |
| Độ phức tạp vận hành | Thấp | Cao hơn, nhưng linh hoạt hơn nhiều |
| Ví dụ tương tự | Giống **EC2-Classic** (AWS thời kỳ đầu) | Giống **VPC** hiện đại (AWS VPC / Azure VNet) |

> [!tip] So với thế giới VMware/NSX
> **Basic Networking** gần giống một **flat port group** dùng chung, cô lập bằng **Distributed Firewall theo tag/group** (không có router riêng cho từng nhóm máy) — tức concept gần với NSX Distributed Firewall (micro-segmentation bằng rule) hơn là routing thật.
> **Advanced Networking** gần giống **NSX-T với T1 Gateway riêng cho từng tenant/network** — có router logic riêng, NAT riêng, có thể làm multi-tier network như một VPC thực thụ.

## Basic Networking — chi tiết

- Toàn bộ VM trong zone nằm trên **1 dải IP lớn duy nhất** (thường là IP "gần public" hoặc NAT 1-1 tùy thiết kế), CloudStack gán trực tiếp Public IP hoặc IP từ pool cho VM.
- Cô lập bằng **Security Group**: VM chỉ được các VM khác truy cập nếu rule security group cho phép — tương tự tư duy AWS EC2-Classic Security Groups.
- **Không có VPC, không có Isolated Network, không có multi-tier**.
- Đơn giản, phù hợp: hosting đơn giản, ít yêu cầu cô lập mạng phức tạp, hoặc hạ tầng nhỏ.

```bash
# Kiểm tra network mode của 1 zone
cmk list zones | grep -i networktype
```

## Advanced Networking — chi tiết

- Mỗi tenant có thể tạo **Isolated Network** riêng (VLAN/VXLAN riêng), có **Virtual Router riêng**, dải IP nội bộ riêng (RFC1918), NAT ra ngoài qua Public IP.
- Hỗ trợ **VPC**: nhiều tier (network) trong 1 VPC, routing giữa các tier qua VPC Virtual Router, ACL theo tier — xem [[VPC & Isolated Networks]].
- Hỗ trợ Site-to-Site VPN, Remote Access VPN, Load Balancing, Static NAT, Port Forwarding đầy đủ.
- Phức tạp hơn nhưng là **lựa chọn bắt buộc** nếu cần multi-tenancy thật sự cô lập (nhiều khách hàng/dự án dùng chung hạ tầng nhưng không được thấy mạng của nhau).

> [!warning] Lesson learned: xác định sai mode khi debug = tốn hàng giờ vô ích
> Rất nhiều lỗi network "kỳ lạ" hóa ra chỉ vì kỹ sư mới quen assumption sai về mode đang chạy. Ví dụ: cố tìm Virtual Router cho 1 VM trong Zone **Basic Networking** — sẽ **không có VR riêng** để tìm (chỉ có VR chung cấp DHCP toàn zone), khiến việc debug đi vào ngõ cụt. **Việc đầu tiên khi nhận bàn giao**: xác nhận rõ zone đang Basic hay Advanced trước khi đọc bất kỳ note networking nào khác trong bộ note này.

## Đa số production hiện đại dùng gì?

Phần lớn triển khai CloudStack nghiêm túc, đa tenant, cần cô lập mạng chặt (điều mà một công ty chuyển từ VMware/NSX sang thường kỳ vọng) sẽ dùng **Advanced Networking + VPC**. Basic Networking ngày càng ít gặp trừ các zone legacy tạo từ rất lâu.

---
*Xem thêm: [[CloudStack Network Architecture Overview]] | [[VPC & Isolated Networks]] | [[Security Groups & Network ACLs]] | [[Cloudstack|CloudStack]]*
