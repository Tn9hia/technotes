# Wazuh — Active Response
Tier: 2
Parent: [[wazuh]]
Related: [[wazuh--rules-decoders]], [[wazuh--agent-enrollment]]
Tags: #wazuh #response #automation

## What it does

Script tự động thực thi trên agent (hoặc manager) khi một rule đủ điều kiện (level/group nhất định) fire — vd block IP qua firewall local, disable user account, kill process.

## Why it exists

Phản ứng thủ công của SOC quá chậm với threat di chuyển nhanh (brute-force, ransomware bắt đầu drop file). Active response đóng vòng lặp detect → contain mà không cần người bấm nút, giảm dwell time.

## How it works (flow/diagram)

```
rule fire (level/group match cấu hình <active-response>)
     │
     ▼
manager gửi lệnh tới agent liên quan
     │
     ▼
agent chạy script local (vd firewall-drop.sh) với quyền thường là root/admin
     │
     ▼
action được log lại; có thể auto-revert sau <timeout>, hoặc escalate nếu
"repeated offenders" (vi phạm lặp lại) — block lâu hơn theo cấp số nhân
```

## Config gotchas

- Active response chỉ chạy được trên agent **đang connected và có script tương ứng bật** — log đến từ nguồn không có agent (vd syslog forward từ thiết bị network) thì rule fire nhưng **không có ai để thực thi block**.
- Default active response ship sẵn khá rộng (vd block IP trên nhiều rule auth-fail khác nhau) — false positive dễ khoá nhầm chính admin/service account đang thao tác hợp lệ (self-inflicted DoS).
- `timeout` và cơ chế "repeated offenders" cần tune theo thực tế — mặc định có thể quá dài (block admin cả giờ vì gõ sai password vài lần) hoặc quá ngắn (không đủ răn đe brute-force thật).
- Script chạy với quyền cao (thường root) trên agent — chất lượng/độ an toàn của script tự viết chính là attack surface mới.

## Security notes

- Nếu attacker có thể spoof source IP xuất hiện trong log (log injection), về lý thuyết có thể lợi dụng active-response như một DoS vector nhắm vào IP hợp lệ (vd IP của gateway/NAT chung) — không nên gắn active response cho rule dựa hoàn toàn vào field dễ giả mạo.
- Luôn thử nghiệm ở chế độ "report-only" (không gắn active response thật, chỉ alert) trong môi trường mới trước khi wiring sang hành động gây gián đoạn thật.
- Giới hạn active-response chỉ cho rule/group có độ tin cậy cao — không nên gắn tràn lan theo mọi rule có level cao, vì level cao không đồng nghĩa với confidence cao.

## Refs

- https://documentation.wazuh.com/current/user-manual/capabilities/active-response/index.html
