---
type: moc
aliases: [cloud-init index, cloud-init MOC]
tags: [moc, cloud-init]
updated: 2026-10-05
---

# cloud-init — Index (MOC)

Topic này gom toàn bộ kiến thức về cloud-init: cơ chế boot, datasource, network config, user-data/merging,
CLI/debug, và runbook vận hành. Note áp dụng cho **cloud-init 26.2**.

## Thứ tự đọc gợi ý

1. [[cloud-init]] — bắt đầu ở đây, có glossary tra nhanh.
2. [[cloud-init--boot-stages]] — khung xương 5 stage + kiến trúc single-process.
3. [[cloud-init--datasources]] — nơi data tới từ đâu.
4. [[cloud-init--user-data-merging]] — định dạng user-data, merge, vendor-data.
5. [[cloud-init--network-config]] — cách network config được render.
6. [[cloud-init--cli-status-debug]] — công cụ hỏi "xong chưa, lỗi gì".

## Theo chủ đề

**Boot & lifecycle:**
- [[cloud-init--boot-stages]]

**Nguồn data:**
- [[cloud-init--datasources]]
- [[cloud-init--user-data-merging]]

**Network:**
- [[cloud-init--network-config]]

**Vận hành & debug:**
- [[cloud-init--cli-status-debug]]
- [[cloud-init--Runbook]]

## Network matrix tổng hợp

### cloud-init

![[cloud-init#^ports]]

## Ops quick links

- **[[cloud-init--Runbook]] — đọc đầu tiên khi có sự cố, checklist hằng ngày**
- [[cloud-init#9. Ops Runbook — Production Notes|cloud-init — Ops]]
- [[cloud-init#10. Gotchas & Lessons Learned|cloud-init — Gotchas]]
