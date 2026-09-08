# Wazuh — Rules & Decoders
Tier: 2
Parent: [[wazuh]]
Related: [[wazuh--active-response]], [[wazuh--agent-enrollment]]
Tags: #wazuh #detection-engineering

## What it does

**Decoder**: bóc log thô (unstructured text/JSON) thành field có cấu trúc (regex/pcre2). **Rule**: chấm điều kiện trên các field đã decode, nếu match thì sinh alert với một `level` (0–16) và group tag.

## Why it exists

Log thô từ 50 loại nguồn khác nhau (sshd, nginx, Windows Event Log, app JSON log...) không thể so sánh/correlate trực tiếp — cần một tầng chuẩn hoá (decoder) trước khi rule engine so khớp logic chung ("5 lần login fail trong 60s" phải hiểu được field `srcip`, `user` bất kể log gốc trông thế nào).

## How it works (flow/diagram)

```
raw log line
     │
     ▼
decoder cha (xác định "hình dạng" log, vd prefix "sshd:")
     │
     ▼
decoder con (trích thêm field: srcip, user, port...)
     │
     ▼
rule engine — match field/regex, có thể chain qua if_sid
  (rule B chỉ fire nếu rule A vừa fire trong X giây → correlation)
     │
     ▼
alert (level 0–16, level 0 = never alert / silence)
```

## Config gotchas

- Custom rule/decoder phải để ở `/var/ossec/etc/rules/local_rules.xml` và `/var/ossec/etc/decoders/local_decoder.xml` — **không** sửa trực tiếp file ruleset vendor, vì bị ghi đè mỗi lần `update_ruleset`/upgrade.
- ID rule custom phải **≥ 100000** (range dành riêng cho local) — dùng ID trùng/thấp hơn vendor sẽ bị conflict hoặc mất khi update.
- Rule level 0 hay bị lạm dụng để "im lặng" noise — nếu regex quá rộng, vô tình silence luôn case thật (blind spot mà không ai biết vì... nó im lặng).
- Order load file theo prefix số trong tên file quyết định thứ tự evaluate — quan trọng khi có `if_sid` phụ thuộc rule ở file khác.
- Regex tệ (đặc biệt pattern có nested quantifier) gây catastrophic backtracking → `analysisd` treo CPU, ảnh hưởng toàn bộ pipeline alert (không chỉ log liên quan tới rule đó).

## Security notes

- Đây là detection logic cốt lõi — audit định kỳ danh sách rule level 0, vì nó chính là nơi hunter/analyst có thể vô tình (hoặc bị dụ) tạo blind spot.
- Luôn test rule/decoder mới bằng `wazuh-logtest` (hoặc `/var/ossec/bin/wazuh-logtest`) trước khi deploy — một decoder sai có thể làm sập khả năng xử lý alert của toàn manager, tức là về bản chất đây là một attack surface availability nếu quy trình review rule không chặt (ai cũng commit rule thẳng vào prod).
- Rule chain dài (`if_sid` nhiều tầng) khó audit — nên giữ document riêng mapping "rule nào phụ thuộc rule nào" nếu ruleset custom phức tạp.

## Refs

- https://documentation.wazuh.com/current/user-manual/ruleset/index.html
- https://github.com/wazuh/wazuh-ruleset
