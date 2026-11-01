---
type: moc
aliases: [Monitor index, Monitor MOC]
tags: [moc, monitor]
updated: 2026-11-01
---

# Monitor — Index (MOC)

Topic này tập hợp note về các công cụ monitoring và runbook vận hành kèm theo. Hiện tại gồm **Prometheus 3.15.1** + **Alertmanager 0.34.2** (thu thập + lưu trữ + alerting), **Grafana 13.3.0** (visualization + dashboarding), và **VictoriaMetrics 1.154.1** (long-term storage hiệu quả cao, tương thích PromQL) — tool khác (Loki...) sẽ bổ sung sau.

## Thứ tự đọc gợi ý

1. [[Prometheus]] — bắt đầu ở đây, có glossary tra nhanh và architecture diagram tổng.
2. [[prometheus--promql-recording-rules]] — PromQL cơ bản, cần nắm trước khi đọc các note khác vì mọi nơi đều dùng tới.
3. [[prometheus--service-discovery-relabeling]] — cách Prometheus tìm target.
4. [[prometheus--storage-tsdb]] — cách Prometheus lưu dữ liệu local + remote write.
5. [[prometheus--alertmanager]] — xử lý alert sau khi Prometheus evaluate.
6. [[prometheus--ha-federation-scaling]] — khi 1 Prometheus không còn đủ.
7. [[prometheus--exporters-cardinality]] — hệ sinh thái exporter + quản lý cardinality, nên đọc cùng lúc với #3.
8. [[Grafana]] — sau khi đã có data source (Prometheus), đọc tiếp để biết cách visualize + alerting trên Grafana.
9. [[grafana--datasources]] → [[grafana--dashboards-panels-variables]] — cách Grafana nối vào Prometheus và build dashboard.
10. [[grafana--alerting]] — hệ alerting riêng của Grafana, khác Alertmanager của Prometheus.
11. [[grafana--provisioning]] — quản lý Grafana theo GitOps.
12. [[grafana--plugins]], [[grafana--rbac-teams-orgs]], [[grafana--ha-database]] — vận hành sâu hơn, đọc khi cần.
13. [[VictoriaMetrics]] — khi Prometheus local storage không còn đủ (xem [[prometheus--ha-federation-scaling]] trước), đọc tiếp để biết cách dùng VictoriaMetrics làm long-term storage.
14. [[victoriametrics--architecture-cluster]] → [[victoriametrics--ingestion-dedup]] — kiến trúc và cách nhận data từ Prometheus/vmagent.
15. [[victoriametrics--metricsql]] — khác biệt so với PromQL cần biết trước khi port query/alert.
16. [[victoriametrics--vmalert]], [[victoriametrics--retention-downsampling]], [[victoriametrics--backup-restore]], [[victoriametrics--security-vmauth]] — vận hành sâu hơn, đọc khi cần.

## Theo chủ đề

**Core — thu thập & lưu trữ:**
- [[Prometheus]]
- [[prometheus--service-discovery-relabeling]]
- [[prometheus--storage-tsdb]]
- [[prometheus--exporters-cardinality]]

**Query & rule:**
- [[prometheus--promql-recording-rules]]

**Alerting:**
- [[prometheus--alertmanager]]

**Scale & HA:**
- [[prometheus--ha-federation-scaling]]

**Visualization & dashboarding (Grafana):**
- [[Grafana]]
- [[grafana--dashboards-panels-variables]]
- [[grafana--datasources]]

**Grafana alerting & vận hành:**
- [[grafana--alerting]]
- [[grafana--provisioning]]
- [[grafana--plugins]]
- [[grafana--rbac-teams-orgs]]
- [[grafana--ha-database]]

**Long-term storage (VictoriaMetrics):**
- [[VictoriaMetrics]]
- [[victoriametrics--architecture-cluster]]
- [[victoriametrics--metricsql]]
- [[victoriametrics--ingestion-dedup]]

**VictoriaMetrics vận hành:**
- [[victoriametrics--vmalert]]
- [[victoriametrics--retention-downsampling]]
- [[victoriametrics--backup-restore]]
- [[victoriametrics--security-vmauth]]

## Network matrix tổng hợp

### Prometheus

![[Prometheus#^ports]]

### Alertmanager

Port cluster/API của Alertmanager nằm trong bảng **Network** riêng của concept note: [[prometheus--alertmanager#Network]].

### Grafana

![[Grafana#^ports]]

### VictoriaMetrics

![[VictoriaMetrics#^ports]]

## Ops quick links

- **[[Monitor--Runbook]] — đọc đầu tiên khi có sự cố, checklist hằng ngày**
- [[Prometheus#9. Ops Runbook — Production Notes|Prometheus — Ops]]
- [[Prometheus#10. Gotchas & Lessons Learned|Prometheus — Gotchas]]
- [[prometheus--alertmanager#Ops notes|Alertmanager — Ops]]
- [[prometheus--exporters-cardinality#Bẫy vận hành|Cardinality — Bẫy vận hành]]
- [[Grafana#9. Ops Runbook — Production Notes|Grafana — Ops]]
- [[Grafana#10. Gotchas & Lessons Learned|Grafana — Gotchas]]
- [[VictoriaMetrics#9. Ops Runbook — Production Notes|VictoriaMetrics — Ops]]
- [[VictoriaMetrics#10. Gotchas & Lessons Learned|VictoriaMetrics — Gotchas]]

## Note chưa viết

- `prometheus--hardening.md` — nếu sau này cần tách riêng phần security/hardening sâu hơn 8 item hiện có ở [[Prometheus#8. Security Considerations]].
