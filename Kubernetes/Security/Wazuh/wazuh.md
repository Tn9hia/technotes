# Wazuh — 4.x
Tags: #siem #xdr #security #infra #open-source
Last updated: 2026-08-20

---

### **1. What — Nó là cái gì?**

Wazuh là platform SIEM/XDR open source, hậu duệ tinh thần của OSSEC (fork từ 2015 khi OSSEC ngừng phát triển tích cực). Nó thu log/telemetry từ agent cài trên endpoint (hoặc agentless qua syslog/API), phân tích qua rule engine, và đẩy alert vào một stack search/dashboard (OpenSearch) để hunt và report.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Nếu không có nó: mỗi server tự log ra file riêng, không ai correlate được "SSH fail 20 lần từ 1 IP" với "user đó login thành công 2 phút sau" trừ khi ngồi grep tay từng máy. Wazuh giải quyết 3 bài toán cùng lúc mà nếu tách riêng phải mua/dựng nhiều tool: (1) log aggregation + correlation rule-based, (2) host-based monitoring (File Integrity Monitoring, rootkit/malware detection, config compliance/CIS benchmark), (3) response tự động (active response) — tất cả không tốn license như Splunk/QRadar/Sentinel.

Đánh đổi: bạn tự vận hành một cluster (manager + OpenSearch), tức là tự gánh phần ops mà SaaS SIEM đã làm sẵn cho bạn.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng khi:**
- Cần SIEM/host monitoring không muốn trả license theo GB ingest (Splunk) hay theo seat.
- Cần FIM, SCA (Security Config Assessment), vulnerability detection dựa trên CPE/CVE của package cài trên host.
- Quy mô vừa (vài chục → vài nghìn agent) và team có năng lực tự vận hành Linux + OpenSearch.

**KHÔNG dùng khi:**
- Team không có kinh nghiệm vận hành Elasticsearch/OpenSearch — indexer là phần dễ chết nhất trong stack (disk watermark, heap OOM, shard quá nhiều), không có ai giữ nó sẽ thành nợ kỹ thuật.
- Cần network-level deep packet inspection / IDS thật sự — Wazuh không phân tích traffic network sâu như Suricata/Zeek, chỉ đọc log. Muốn NIDS phải pair thêm Suricata và feed log vào Wazuh.
- Volume log cực lớn (nhiều TB/ngày) mà không có budget tune indexer nghiêm túc — sẽ tụt hậu (lag) giữa log sinh ra và alert xuất hiện.
- Cần compliance "managed" kiểu SaaS có SLA support 24/7 — Wazuh community support qua Slack/forum, không có SLA trừ khi mua Wazuh Cloud/support enterprise.

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
┌──────────────┐      1514/tcp (encrypted)      ┌────────────────────────────┐
│ Wazuh Agent  │ ─────────────────────────────▶ │        Wazuh Manager        │
│ (endpoint)   │ ◀───────────────────────────── │  remoted → analysisd →      │
│ Linux/Win/Mac│      1515/tcp (enrollment)      │  (decoder+rule engine)      │
└──────────────┘                                 │  → alerts.json / archives   │
                                                  └──────────────┬─────────────┘
                                                                 │ Filebeat ships
                                                                 ▼
                                                  ┌────────────────────────────┐
                                                  │      Wazuh Indexer          │
                                                  │  (OpenSearch fork, 9200)    │
                                                  └──────────────┬─────────────┘
                                                                 │
                                                                 ▼
                                                  ┌────────────────────────────┐
                                                  │     Wazuh Dashboard         │
                                                  │ (OpenSearch Dashboards, 443)│
                                                  └────────────────────────────┘

                    Wazuh API (55000/tcp) ── quản lý agent, rule, cluster qua REST
```

- **Vị trí trong hệ thống:** ngồi ngoài band, không inline traffic — agent đọc log/local state, gửi metadata về manager, không chặn traffic (trừ active response chủ động thực thi lệnh trên host).
- **Component chính:** Agent (thu thập) → Manager (remoted nhận kết nối, analysisd giải mã + chấm rule) → Filebeat (shipper) → Indexer (lưu trữ, search) → Dashboard (UI).
- **Data flow:** raw log → decoder (structure hoá) → rule engine (match + level) → alert → filebeat → index theo ngày trong indexer → query qua dashboard.
- **Dependency:** OpenSearch (bắt buộc cho indexer/dashboard), Filebeat (cầu nối manager↔indexer, nếu chết thì alert dồn ứ trên manager disk).

### **5. How — Cơ chế hoạt động**

- **remoted**: daemon trên manager nhận kết nối từ agent (1514), giải mã payload bằng key riêng của từng agent.
- **analysisd**: pipeline chính — chạy log qua decoder (regex/pcre2 trích field), rồi chạy rule engine match theo field đã trích, gán rule level (0–16, 0 = never alert). Đây là bottleneck CPU chính khi throughput cao.
- **Ruleset = decoders.xml + rules.xml**: decoder xác định "hình dạng" log (vd sshd, json), rule chấm điều kiện trên field đã decode. Rule có thể chain qua `if_sid` để correlate nhiều event (vd 5 lần failed-login trong 60s → 1 alert brute-force).
- **FIM (syscheck)**: agent hash file định kỳ/real-time (inotify trên Linux), báo thay đổi về manager.
- **SCA / Rootcheck**: agent tự chấm điểm config host theo policy (CIS benchmark) hoặc quét rootkit signature.
- **Active response**: rule level/group đủ điều kiện → manager lệnh agent chạy script (block IP, disable user...). Xem [[wazuh--active-response]].
- Backlinks: [[wazuh--agent-enrollment]] · [[wazuh--rules-decoders]] · [[wazuh--active-response]]

### **6. Key Config — Cấu hình cần nhớ**

- `ossec.conf` (agent lẫn manager) — mọi thứ nằm đây, không có config UI đầy đủ, phải sửa XML tay hoặc qua API.
- `registration_password` (authd, enrollment 1515) — **default rỗng**, nếu port này expose ra ngoài mà không set password = ai cũng enroll được agent giả. Xem chi tiết [[wazuh--agent-enrollment]].
- Local rule/decoder ID phải **≥ 100000** — vendor ruleset dùng ID thấp hơn và bị ghi đè mỗi lần update, để lẫn ID là mất rule khi upgrade. Xem [[wazuh--rules-decoders]].
- `logall`/`logall_json` (archives) — mặc định **tắt**. Nghĩa là log không match rule nào thì bị *drop*, không lưu lại để hunt sau. Bật lên thì tốn disk nhanh (archives ghi mọi thứ, không chỉ alert).
- Indexer JVM heap — theo chuẩn OpenSearch/ES: set ~50% RAM node, **không bao giờ vượt 32GB** (mất compressed oops, GC tệ hơn). Đây là gotcha kinh điển bị copy-paste sai khi mới setup.
- Wazuh API (55000) và Dashboard/Indexer admin — **default credentials** (`admin`/`admin` kiểu) phải đổi ngay khi cài, nhiều lần bị scan/exploit tự động vì để mặc định.

### **7. Security Considerations**

- **Attack surface:** 1514 (agent↔manager, encrypted nhưng enrollment yếu = agent giả); 1515 (authd enrollment, điểm yếu nhất nếu không đặt password); 55000 (API, RBAC + default creds); Dashboard 443 (nếu expose internet mà không giới hạn network/VPN + MFA).
- **Misconfig dẫn tới breach:**
  - authd mở ra 0.0.0.0 không password → agent giả enroll, có thể inject alert giả để che dấu attack thật (log injection để đánh lạc hướng SOC).
  - Indexer/Dashboard giữ credential mặc định — pattern rất phổ biến trong các báo cáo compromise ELK/OpenSearch expose ra internet.
  - `client.keys` của agent bị lộ (endpoint compromise) = attacker impersonate agent, gửi log giả hoặc dùng kết nối đó làm pivot.
- **Hardening checklist tối thiểu:**
  1. Đổi toàn bộ default password (indexer admin, dashboard, API user).
  2. Set `registration_password` mạnh cho authd, giới hạn 1515 chỉ trong mạng nội bộ/VPN.
  3. Giới hạn 1514/55000 bằng firewall/security group, không public.
  4. Bật TLS end-to-end (agent-manager đã mặc định encrypted, nhưng indexer↔dashboard↔filebeat cần verify cert, không tắt cert check).
  5. RBAC trên API — không dùng 1 tài khoản full quyền cho mọi tool tích hợp.
  6. Review định kỳ rule level 0 (silence rule) — dễ bị lạm dụng để che noise nhưng vô tình che luôn alert thật.

### **8. Ops Runbook — Production Notes**

- **Health check:**
  - `systemctl status wazuh-manager wazuh-indexer wazuh-dashboard filebeat`
  - Manager: `/var/ossec/bin/wazuh-control status` (agent) hoặc Dashboard → Agents (active/disconnected/never-connected).
  - Cluster: API `/manager/status`, `/cluster/healthcheck`.
- **Log quan trọng cần monitor:**
  - `/var/ossec/logs/ossec.log` — lỗi manager, rule load fail, decoder parse error.
  - `/var/ossec/logs/cluster.log` — nếu chạy cluster mode.
  - Filebeat log — nếu filebeat không ship được, alert **dồn ứ trên đĩa manager**, không mất ngay nhưng dashboard sẽ "trống" dù manager vẫn alert bình thường (dễ gây hoang mang khi troubleshoot).
  - Indexer log (`/var/log/wazuh-indexer`) — theo dõi GC pause, circuit breaker.
- **Metric cần alert:**
  - Số agent disconnected tăng đột biến (có thể là outage thật hoặc network issue chứ không phải agent chết hết).
  - Disk watermark của indexer (OpenSearch mặc định flood-stage ~95% → index chuyển read-only, **mất khả năng ghi alert mới**, đây là failure mode âm thầm và nguy hiểm nhất).
  - Tỷ lệ event bị drop ở analysisd queue (dấu hiệu manager quá tải CPU).
  - JVM heap usage indexer node.
- **Restart order:** Indexer → Manager → Dashboard (filebeat đi kèm/khởi động cùng manager). Restart sai thứ tự dễ khiến dashboard báo "cannot connect to indexer" dù indexer thật ra chưa kịp up.
- **Upgrade/rollback:** snapshot index trước khi upgrade major version; backup `/var/ossec/etc/rules` và `/var/ossec/etc/decoders` (custom rule dễ bị format thay đổi hoặc reset khi upgrade ruleset vendor).

### **9. Gotchas & Lessons Learned**

> Phần dưới là kiến thức chung từ community/docs — cần bạn xác nhận/bổ sung khi đã vận hành thực tế trong môi trường của mình.

- Index của indexer rotate theo ngày (`wazuh-alerts-*`) — nếu không có ISM (Index State Management) policy để rollover/delete index cũ, disk sẽ đầy dần mà không ai để ý cho tới khi chạm flood-stage watermark.
- Ruleset mặc định khá "noisy" ở môi trường thật — kỳ vọng phải tune vài tuần đầu (thêm rule level 0 để silence false positive, viết custom rule cho log app riêng) trước khi SOC hết bị alert fatigue.
- FIM scan đồng loạt nhiều agent cùng giờ (theo default schedule) có thể gây spike CPU/IO trên endpoint lẫn network về manager — nên stagger/randomize `frequency`.
- Custom decoder với regex tệ (catastrophic backtracking) có thể làm `analysisd` treo CPU 100% — luôn test bằng `wazuh-logtest` trước khi deploy prod.

### **10. Resources**

- Official docs: https://documentation.wazuh.com
- Source/repo: https://github.com/wazuh/wazuh
- Ruleset repo (community rule đóng góp, tham khảo pattern viết rule thật): https://github.com/wazuh/wazuh-ruleset
