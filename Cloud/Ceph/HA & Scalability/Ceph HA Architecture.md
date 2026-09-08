---
tags:
  - ceph
  - ha
  - architecture
---

# Ceph HA Architecture

Ceph có tính sẵn sàng cao (HA) ở **nhiều tầng độc lập** — MON, MGR, OSD/data, MDS, RGW — mỗi tầng chịu lỗi theo cơ chế riêng, không phải một cơ chế HA duy nhất cho toàn hệ thống. Hiểu rõ "tầng nào chịu được gì" là kỹ năng quan trọng nhất khi phản ứng sự cố (incident response), vì nó quyết định bạn có cần hành động khẩn cấp ngay hay có thể xử lý bình thường trong giờ hành chính.

> [!tip] So với VMware vSAN
> vSAN gộp chung HA của control plane (CLOM/DOM) và data plane vào cùng 1 cluster object — bạn ít khi phải nghĩ tách bạch "quorum của cái gì". Ceph tách rõ: MON quorum là control plane (biết ai giữ dữ liệu gì), OSD/CRUSH là data plane (nơi dữ liệu thực sự nằm) — 2 thứ có thể fail độc lập với hệ quả rất khác nhau. Đây là điểm tư duy quan trọng nhất cần điều chỉnh khi chuyển từ vSAN sang Ceph.

## HA theo từng tầng

```
                    ┌─────────────────────────────────────┐
                    │         Control Plane (MON)          │
                    │   quorum floor((n-1)/2) chịu lỗi      │
                    └──────────────────┬────────────────────┘
                                        │ cluster map
                    ┌───────────────────┴────────────────────┐
                    │              MGR (active/standby)        │
                    │     dashboard, orchestrator, balancer     │
                    └───────────────────┬────────────────────┘
                                        │
        ┌───────────────┬───────────────┼───────────────┬───────────────┐
        │                │               │               │               │
   ┌────▼────┐      ┌────▼────┐    ┌────▼────┐    ┌─────▼─────┐   ┌─────▼─────┐
   │  OSD /   │      │  MDS     │    │  RGW     │    │  librbd /  │   │  librados  │
   │  CRUSH   │      │(CephFS)  │    │(S3/Swift)│    │  client    │   │  client    │
   │ replicate│      │ active + │    │ stateless│    │  tự retry, │   │  tự retry  │
   │ /EC theo │      │ standby- │    │ sau LB   │    │  reconnect │   │            │
   │ failure  │      │ replay   │    │(haproxy/ │    │  OSD mới   │   │            │
   │ domain   │      │          │    │keepalived)│   │  tự động   │   │            │
   └──────────┘      └──────────┘    └──────────┘    └────────────┘   └────────────┘
```

### MON — quorum

MON dùng thuật toán Paxos, cần **đa số (majority)** để hoạt động. Với n MON, cluster chịu được tối đa `floor((n-1)/2)` MON chết mà vẫn còn quorum:

| Số MON | Chịu được mất | Ghi chú |
|---|---|---|
| 1 | 0 | Không có HA — chỉ dùng cho lab/POC |
| 3 | 1 | Tối thiểu khuyến nghị cho production |
| 5 | 2 | Cluster lớn hoặc cần chịu lỗi cao hơn (ví dụ trải nhiều rack) |
| 4 (số chẵn — tránh) | 1 | Số chẵn không tăng tolerance so với n-1, chỉ tốn thêm tài nguyên — luôn dùng số lẻ |

### MGR — active/standby

Chỉ 1 MGR active tại một thời điểm, các MGR khác ở chế độ standby và tự nhận vai trò active khi MGR hiện tại chết. MGR **không giữ dữ liệu cluster map quan trọng** (đó là việc của MON) — MGR mất tạm thời không làm mất I/O, nhưng làm mất dashboard, một số module (balancer, Prometheus exporter) và khả năng chạy `ceph orch` cho tới khi standby lên thay.

### OSD/data — CRUSH failure domain

Redundancy thực sự của dữ liệu nằm ở replication/EC + CRUSH failure domain (thường là "host" — đảm bảo các bản sao của cùng 1 object không nằm trên cùng 1 host vật lý). Xem chi tiết ở [[CRUSH Algorithm & CRUSH Map]] và [[Pools, Replication & Erasure Coding|Replication & Erasure Coding]].

### MDS — active + standby-replay (CephFS)

Nếu dùng CephFS, MDS (Metadata Server) có thể chạy active + 1 hoặc nhiều standby, trong đó `standby-replay` liên tục theo dõi journal của active để failover nhanh hơn standby thường (không cần replay lại toàn bộ journal từ đầu lúc failover).

### RGW — stateless, cần load balancer bên ngoài

RGW không có cơ chế HA nội tại kiểu MON/MGR — nó là các instance **stateless** (không giữ state riêng, mọi state nằm trong RADOS). HA cho RGW là trách nhiệm của tầng phía trước nó:

```
Client (S3 API) ──> [haproxy/keepalived VIP hoặc DNS round-robin] ──> RGW-1, RGW-2, RGW-3
```

Ceph **không đi kèm load balancer** — bạn cần tự triển khai haproxy+keepalived (VIP failover) hoặc dùng DNS round-robin/LB bên ngoài (ví dụ LB của CloudStack/cloud provider) để phân phối traffic và loại bỏ instance chết khỏi pool.

### Client-side (librbd/librados)

Đây là điểm khác biệt lớn nhất so với SAN truyền thống: client Ceph (`librbd` mà CloudStack dùng qua KVM) **tự động** phát hiện OSD primary mới khi có failover PG, tự retry và reconnect — không cần cấu hình gì thêm ở tầng client. So với SAN cần multipath (`multipathd`, cấu hình path failover thủ công), Ceph client xử lý việc này hoàn toàn trong suốt nhờ luôn hỏi lại cluster map mới nhất khi cần.

## Kiểm tra nhanh trạng thái HA từng tầng

```bash
# MON quorum
ceph quorum_status --format json-pretty | grep -E '"quorum_names"|"name"'
ceph -s | grep mon

# MGR active/standby
ceph mgr stat
ceph -s | grep mgr

# MDS (nếu dùng CephFS)
ceph fs status
ceph mds stat

# RGW — kiểm tra qua orchestrator, health check thực tế nên làm ở tầng LB
ceph orch ps --daemon-type rgw

# OSD/host — daemon nào down, host nào ảnh hưởng
ceph osd tree | grep -i down
```

## Bảng "nếu X chết thì sao" — dùng cho incident response

| Sự kiện | Ảnh hưởng ngay lập tức | Cần hành động khẩn cấp? |
|---|---|---|
| 1 MON chết (còn quorum) | `HEALTH_WARN`, không ảnh hưởng I/O | Không khẩn cấp, nhưng cần thay thế trước khi mất thêm MON |
| Đa số MON chết (mất quorum) | **Toàn cluster ngừng nhận thay đổi map mới** — I/O đang chạy có thể tiếp tục một thời gian ngắn với map cũ nhưng mọi thao tác cần quorum (tạo pool, OSD up/down mới...) đều treo | **Có — sự cố nghiêm trọng nhất trong toàn bộ danh sách này** |
| Tất cả MGR chết | Mất dashboard, mất `ceph orch`, mất balancer/autoscaler — I/O client **không bị ảnh hưởng** | Không khẩn cấp cho I/O, nhưng cần xử lý sớm để lấy lại khả năng quản trị |
| 1 OSD chết | PG liên quan chuyển `degraded`, cluster tự backfill sang OSD khác theo CRUSH, I/O vẫn phục vụ bình thường (đọc/ghi qua bản sao còn lại) | Không khẩn cấp ngay, nhưng cần theo dõi và thay thế |
| 1 host OSD chết hoàn toàn | Nhiều OSD cùng lúc `down`, PG có thể vào `degraded` hoặc tạm thời `undersized` nếu failure domain=host và size=2 (không nên dùng size=2), backfill lớn bắt đầu | Cần xử lý sớm — theo dõi tải backfill ảnh hưởng client I/O |
| MDS chết | CephFS chuyển sang standby (hoặc standby-replay, nhanh hơn); nếu không có standby, CephFS "đứng hình" tới khi MDS mới lên | Có, nếu không có standby cấu hình sẵn |
| 1 RGW instance chết | Client bị timeout tới instance đó cho tới khi LB loại nó khỏi pool; các instance RGW còn lại vẫn phục vụ | Không khẩn cấp nếu có LB đúng cấu hình health check |

> [!warning] Lesson learned: nghĩ "Ceph tự self-heal nên không cần làm gì" — âm thầm tiêu hết redundancy budget
> Một đội vận hành thấy `HEALTH_OK` liên tục sau khi 1 OSD chết và cluster tự backfill xong (đúng như thiết kế), nên hình thành thói quen "cứ để Ceph tự lo, không cần thay ổ ngay". Vài tuần sau, một OSD khác trong **cùng failure domain** (cùng host, hoặc host khác nhưng cùng rack nếu failure domain=rack) tiếp tục chết — lúc này vì bản sao dữ liệu đã bị giảm sẵn từ lần trước chưa được bù đắp bằng phần cứng thật, PG rơi vào tình trạng thiếu bản sao nghiêm trọng hơn nhiều so với 1 sự cố đơn lẻ, có nguy cơ thực sự mất dữ liệu (không chỉ là cảnh báo). Bài học: `HEALTH_OK`/self-healing xử lý được **một** sự cố tại một thời điểm bằng cách dùng redundancy hiện có, nhưng nó không tự bổ sung lại phần cứng đã mất — mỗi OSD/host chết đi mà không được thay thế là một lần "rút bớt" ngân sách chịu lỗi của cluster cho lần sự cố **tiếp theo**. Luôn theo dõi và thay thế phần cứng hỏng kịp thời, đừng để self-healing khiến bạn chủ quan.

## RGW HA — mẫu haproxy/keepalived tối giản

```
# /etc/haproxy/haproxy.cfg (rút gọn)
frontend rgw_front
    bind 10.10.10.100:80
    default_backend rgw_back

backend rgw_back
    balance roundrobin
    option httpchk GET /
    server rgw1 10.10.10.11:8080 check
    server rgw2 10.10.10.12:8080 check
    server rgw3 10.10.10.13:8080 check
```

keepalived cung cấp VIP (`10.10.10.100`) trôi nổi giữa 2+ node haproxy để chính haproxy cũng không phải single point of failure — một tầng HA lồng trong tầng HA. Ceph không quản lý phần này; đây là hạ tầng bạn tự dựng và tự giám sát riêng, tách biệt hoàn toàn khỏi `ceph orch`.

---
*Xem thêm: [[Scaling the Cluster - Add-Remove Node & OSD]] | [[Recovery, Backfill & Self-healing]] | [[MON - Monitor]] | [[Ceph|Ceph]]*
