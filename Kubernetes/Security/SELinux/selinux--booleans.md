# SELinux — Booleans
Tier: 2
Parent: [[selinux]]
Related: [[selinux--troubleshooting]], [[selinux--contexts-and-labeling]]
Tags: #selinux #booleans

## What it does

Boolean là công tắc bật/tắt runtime cho một **cụm rule đã biên dịch sẵn** trong policy — không cần compile lại policy để đổi hành vi. Mỗi boolean thường bật/tắt nhiều dòng `allow` rule cùng lúc, không phải 1 rule đơn lẻ.

## Why it exists

Nhiều hành vi "có thể cần, có thể không" tuỳ theo cách admin dùng service (VD: httpd có cần nối NFS không? có cần mở network connection ra ngoài không?) — nếu compile cứng vào policy thì mỗi lần đổi ý phải build lại module. Boolean cho phép đổi ngay lập tức, an toàn hơn viết custom policy module tay.

## How it works (flow/diagram)

```bash
getsebool -a                        # liệt kê tất cả boolean + trạng thái hiện tại
getsebool -a | grep httpd           # lọc theo service
semanage boolean -l                 # giống trên nhưng kèm mô tả ý nghĩa từng boolean
semanage boolean -l | grep httpd

setsebool httpd_use_nfs on          # đổi TẠM THỜI (mất khi reboot)
setsebool -P httpd_use_nfs on       # đổi PERSISTENT (ghi vào policy store, sống qua reboot)
```

`-P` phải build lại phần policy liên quan nên **chậm hơn** lệnh không có `-P` — bình thường, không phải lỗi treo máy.

## Config gotchas

- Quên `-P` → tưởng đã fix permanent, reboot xong lỗi denial quay lại y hệt cũ. Đây là gotcha phổ biến nhất với boolean.
- Boolean off theo default **luôn có lý do bảo mật** — bật tất cả boolean liên quan tới một service theo kiểu "thử cho hết lỗi" (anti-pattern hay gặp khi vội) sẽ mở rộng attack surface không cần thiết. Chỉ bật đúng boolean mà `audit2why`/log denial chỉ đích danh.
- Tên boolean không phải lúc nào cũng trực quan — luôn `semanage boolean -l | grep <keyword>` đọc mô tả trước khi bật, đừng đoán theo tên.

## Security notes

- Boolean đáng chú ý khi audit hardening: `httpd_can_network_connect` (cho phép httpd chủ động nối ra ngoài — nếu app không cần gọi API ngoài, giữ off), `httpd_enable_homedirs`, `selinuxuser_execmod` (cho phép exec vùng nhớ writable — liên quan CVE class W^X, giữ off trừ khi bắt buộc), `httpd_unified` (gộp nhiều type content thành 1 — tiện nhưng giảm phân tách).
- Review định kỳ boolean nào đang ON khác default — đó là danh sách "nới lỏng đã áp dụng", nên có lý do ghi lại (ai bật, tại sao, ticket nào).

## Refs

- [Chapter 1 — Getting started with SELinux (RHEL 9), phần booleans](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/using_selinux/index)
