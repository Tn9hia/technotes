---
tags:
  - cloudstack
  - operations
  - upgrade
---

# CloudStack Upgrade Procedure

## Nguyên tắc chung

CloudStack upgrade thường yêu cầu **upgrade tuần tự qua từng minor/major version** (không nhảy cóc nhiều version cùng lúc), và luôn có **DB schema migration** chạy tự động khi Management Server mới khởi động lần đầu.

> [!warning] Luôn đọc kỹ Release Notes/Upgrade Notes chính thức cho ĐÚNG cặp version nguồn→đích
> Khác với việc "cứ apt upgrade là xong", CloudStack thường có các bước đặc thù theo từng version (thay đổi schema, đổi tên global setting, deprecate API...). Không có 1 quy trình chung áp dụng được cho mọi cặp version — luôn tra đúng tài liệu upgrade cho version đang chạy → version đích trước khi làm theo checklist chung dưới đây.

## Checklist chuẩn bị (bắt buộc)

- [ ] **Backup toàn bộ DB** (`cloud` + `cloud_usage`) — xem [[Database HA - MySQL Galera]]
- [ ] Backup file cấu hình: `server.properties`, `db.properties`, `agent.properties` trên tất cả MS/Host
- [ ] Đọc kỹ Upgrade Notes chính thức cho đúng version đang nhảy tới
- [ ] Kiểm tra tương thích hypervisor (một số version CloudStack yêu cầu version KVM/libvirt tối thiểu mới)
- [ ] Thông báo maintenance window cho các tenant/team liên quan
- [ ] Chuẩn bị môi trường test/staging giống production để dry-run trước (nếu có điều kiện)
- [ ] Xác nhận có đường lùi (rollback plan) rõ ràng nếu upgrade thất bại giữa chừng

## Quy trình tổng quát (Management Server, nhiều node)

```
1. Dừng toàn bộ Management Server (không dừng Host/Agent — VM vẫn chạy bình thường)
2. Backup DB đầy đủ (đảm bảo backup TRƯỚC khi bất kỳ MS nào chạm vào DB mới)
3. Nâng cấp gói cloudstack-management trên node ĐẦU TIÊN
   apt update && apt install cloudstack-management
   → Node này sẽ tự động chạy DB schema migration khi khởi động lần đầu
4. Khởi động node đầu tiên, theo dõi log kỹ:
   tail -f /var/log/cloudstack/management/management-server.log
5. Xác nhận UI/API hoạt động bình thường, kiểm tra vài thao tác cơ bản (list VM, list host)
6. Nâng cấp lần lượt các node MS còn lại (KHÔNG chạy schema migration lần 2 — chỉ node đầu tiên làm việc này)
7. Nâng cấp Agent trên từng KVM Host (rolling, dùng maintenance mode — xem [[CloudStack Day 2 Operations]])
8. Nâng cấp System VM Template nếu version mới yêu cầu (import template mới, restart network with cleanup dần theo kế hoạch)
```

> [!warning] Lesson learned nghiêm trọng: nhiều MS cùng chạy schema migration đồng thời = hỏng DB
> Nếu vô tình khởi động **nhiều Management Server đã upgrade cùng lúc** khi DB chưa từng được migrate, có nguy cơ **race condition trong quá trình chạy schema migration**, gây hỏng DB không thể phục hồi nếu không có backup. Luôn upgrade và khởi động **node đầu tiên một mình**, xác nhận migration hoàn tất và ổn định, rồi mới đụng tới các node còn lại.

## System VM Template — thường bị quên khi upgrade

Nhiều version CloudStack mới yêu cầu **System VM Template mới tương ứng** (SSVM/CPVM/VR chạy trên template cũ có thể không tương thích đầy đủ tính năng mới, hoặc bị CloudStack cảnh báo).

```bash
# Import system VM template mới (theo đúng version, đúng hypervisor)
/usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt \
  -m /export/secondary -u <url-template-moi> -h kvm -F

# Sau khi import xong, các System VM cũ cần được restart (theo kế hoạch, từng network một)
```

> [!warning] Đừng restart toàn bộ network cleanup=true ngay sau khi upgrade
> Việc ép toàn bộ VR/SSVM/CPVM tái tạo cùng lúc ngay sau khi upgrade dễ gây **gián đoạn diện rộng** nếu template mới có vấn đề chưa phát hiện. Nên rollout theo từng nhóm network không quan trọng trước, theo dõi ổn định, rồi mới mở rộng dần ra network production quan trọng — giống tư duy canary release.

## Rollback

CloudStack **không có cơ chế rollback tự động** cho DB schema đã migrate — kế hoạch rollback thực chất là **restore từ backup DB đã chụp trước khi upgrade**, kèm theo downgrade lại package Management Server về version cũ.

> [!warning] Vì sao backup trước upgrade là bắt buộc tuyệt đối, không phải "nên làm"
> Do schema migration là **một chiều** (không có migration ngược chính thức được hỗ trợ), nếu upgrade thất bại giữa chừng mà không có backup DB đầy đủ từ trước, khả năng cao là **không thể quay lại trạng thái cũ** một cách an toàn. Đây là điểm khác biệt lớn so với thói quen "Storage vMotion/snapshot rollback" quen thuộc bên VMware — ở đây, backup DB là tuyến phòng thủ duy nhất.

## Sau khi upgrade

- [ ] Verify version đúng qua UI (Infrastructure > About) hoặc `SELECT * FROM cloud.version ORDER BY id DESC LIMIT 1;`
- [ ] Test deploy 1 VM mới, tạo network mới, snapshot thử
- [ ] Theo dõi log/alert sát sao trong 24-48h đầu sau upgrade
- [ ] Cập nhật tài liệu vận hành nội bộ nếu có thay đổi hành vi/API đáng chú ý

---
*Xem thêm: [[Database HA - MySQL Galera]] | [[CloudStack Day 2 Operations]] | [[CloudStack Management Server]] | [[Cloudstack|CloudStack]]*
