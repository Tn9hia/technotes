# SELinux — MLS/MCS & sVirt (Isolation Layer)
Tier: 2
Parent: [[selinux]]
Related: [[selinux--containers-kubernetes]], [[selinux--systemd-service-confinement]]
Tags: #selinux #mcs #mls #svirt #isolation

## What it does

Thêm một trục kiểm soát **độc lập với Type Enforcement**: dùng "level" (`sX` cho MLS) hoặc "category" (`cX` cho MCS) để phân biệt các instance **cùng type** nhưng cần cô lập lẫn nhau. Đây chính là cơ chế đứng sau việc 2 VM hoặc 2 container trên cùng máy — dù cùng chạy domain `svirt_t`/`container_t` — vẫn không đọc được dữ liệu của nhau.

## Why it exists

TE trả lời "domain này được làm gì với type kia" nhưng không phân biệt được "instance nào trong nhiều instance cùng type". Nếu 10 container đều chạy `container_t`, TE một mình không đủ để chặn container A đọc volume của container B (vì type giống hệt nhau). MCS thêm nhãn category riêng cho từng instance (`s0:c1,c2` cho container A, `s0:c3,c4` cho container B) → rule ngầm định "phải trùng category mới được truy cập" tách chúng ra.

## How it works (flow/diagram)

**MLS (Multi-Level Security)** — mô hình phân cấp (s0 < s1 < s2...), dùng cho yêu cầu kiểu quân sự/chính phủ (Bell-LaPadula: "no read up, no write down"). RHEL 9 có `selinux-policy-mls` riêng (`SELINUXTYPE=mls`), ít dùng trong doanh nghiệp thông thường vì phức tạp vận hành.

**MCS (Multi-Category Security)** — không phân cấp, chỉ là tập hợp "nhãn" độc lập (giống tag), dùng cho targeted policy (default trên RHEL 9). Đây là cơ chế thực tế bạn sẽ gặp hàng ngày với containers/VM.

```bash
semanage user -l                          # xem SELinux user, role, MLS/MCS range đang cấu hình
chcat -l -- +category1,-category2 user1   # thêm/bớt category cho 1 Linux user (lưu ý dấu -- bắt buộc)
```

**sVirt** — lớp tích hợp libvirt/container runtime với MCS: mỗi VM (qua libvirt) hoặc container (qua Podman/CRI-O) khi khởi chạy được **tự động cấp một cặp category ngẫu nhiên duy nhất** (VD: `s0:c123,c456`), gắn lên cả process lẫn volume nó mount — đảm bảo cô lập automatic mà admin không cần tự tay quản lý category cho từng workload.

## Config gotchas

- Category là tài nguyên hữu hạn theo policy (thường 0-1023 cho targeted) — hệ thống chạy **rất nhiều** container/VM đồng thời hiếm khi cạn nhưng cần biết giới hạn này tồn tại khi thiết kế hệ thống multi-tenant cực lớn.
- Nhầm lẫn phổ biến: tưởng đổi type là đủ để cô lập 2 workload, nhưng nếu chúng vô tình được gán **cùng category** (VD: set `--security-opt label=level:s0:c1,c2` giống nhau cho 2 container thủ công) thì cô lập MCS bị vô hiệu hoá dù type giống nhau vẫn đúng thiết kế.
- MLS và MCS **không dùng chung** trên một hệ thống theo kiểu bật cả hai độc lập — `SELINUXTYPE` chỉ chọn 1 trong `targeted` (có MCS), `mls` (có cả MLS+MCS), hoặc `minimum`.

## Security notes

- Đây là lớp phòng thủ chính chống lại **container/VM escape ngang hàng** (lateral movement giữa các workload trên cùng host) — namespace/cgroup cô lập process view và resource, còn MCS cô lập ở tầng access control nếu kẻ tấn công thoát được namespace.
- Tắt sVirt tự động cấp category (VD: chạy container với `--security-opt label=disable`) đồng nghĩa loại bỏ hẳn lớp cô lập này cho container đó — chỉ nên làm khi thật sự cần thiết và có kiểm soát bù đắp khác.

## Refs

- [Chapter 6 — Using Multi-Level Security (MLS) (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/using-multi-level-security-mls_using-selinux)
- [Chapter 7 — Using Multi-Category Security (MCS) for data confidentiality (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/assembly_using-multi-category-security-mcs-for-data-confidentiality_using-selinux)
