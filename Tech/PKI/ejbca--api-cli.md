# EJBCA — REST API, CLI & Enrollment Protocol Endpoints
Tier: 2
Parent: [[EJBCA]]
Related: [[pki--enrollment-protocols]], [[ejbca--end-entities-ra]], [[ejbca--architecture-components]]
Tags: #ejbca #api #cli #automation

## What it does

Các cách tự động hoá tương tác với EJBCA thay vì thao tác tay qua Admin/RA Web:
- **REST API** — quản lý cert/End Entity qua HTTP+JSON (issue, revoke, search, download CRL...).
- **CLI (`ejbca.sh`/`ejbca.bat`)** — command-line chạy trực tiếp trên node EJBCA, dùng cho scripting/admin nội bộ.
- **Protocol endpoints** — EJBCA đóng vai trò server cho các giao thức enrollment chuẩn: **ACME, SCEP, EST, CMP** (xem [[pki--enrollment-protocols]] để hiểu bản chất từng giao thức).
- **Peer connector** — kênh giao tiếp giữa các node EJBCA phân tán (RA node ↔ CA node), không dành cho client bên ngoài.

## Why it exists

Thao tác thủ công qua Web UI không scale khi có hàng trăm/nghìn service cần enrollment/renew tự động, hoặc khi cần tích hợp EJBCA vào pipeline nội bộ (vd hệ thống provisioning tự tạo End Entity + issue cert khi 1 server mới được khởi tạo). API/CLI/Protocol là 3 cách khác nhau để đạt cùng mục tiêu: loại bỏ con người khỏi vòng lặp cho các thao tác lặp lại.

## How it works (flow/diagram)

```
                         ┌─────────────────────────┐
   Script/Pipeline nội bộ│      REST API             │  → issue/revoke/search
   (CI/CD, provisioning) │  (JSON qua HTTPS, auth     │     cert theo API call
   ─────────────────────►│   bằng client cert hoặc    │     trực tiếp
                          │   API key tuỳ cấu hình)    │
                          └─────────────────────────┘

   Admin script nội bộ    ┌─────────────────────────┐
   ─────────────────────►│      CLI (ejbca.sh)        │  → chạy trực tiếp trên
                          │  (chạy trên chính node,    │     server, quyền hạn
                          │   quyền OS-level)          │     theo OS user, không
                          └─────────────────────────┘     qua Role/Access Rule

   Thiết bị/service tự    ┌─────────────────────────┐
   động enroll            │  ACME / SCEP / EST / CMP  │  → client tự enroll/renew
   ─────────────────────►│  endpoints                │     theo đúng chuẩn giao
                          │  (mỗi protocol có alias    │     thức, không cần biết
                          │   endpoint riêng, cấu      │     chi tiết nội bộ EJBCA
                          │   hình theo CA/Profile)    │
                          └─────────────────────────┘

   RA node ←──── Peer connector (mTLS nội bộ) ────→ CA node
   (không dùng cho client/service bên ngoài, chỉ giữa các node EJBCA)
```

## Config gotchas

- **REST API bật mặc định nhưng không có nghĩa là an toàn để expose public** — luôn giới hạn network access + xác thực chặt (client cert/mTLS được khuyến nghị hơn API key thuần).
- **CLI chạy trực tiếp trên node = bỏ qua toàn bộ Role/Access Rule của Admin Web** — quyền hạn CLI phụ thuộc OS user chạy nó, cần kiểm soát ai có SSH access tới EJBCA node ngang mức kiểm soát Super Administrator.
- **Mỗi protocol endpoint (ACME/SCEP/EST/CMP) cấu hình alias riêng, trỏ tới 1 CA + Certificate Profile + End Entity Profile cụ thể** — dễ nhầm alias khi có nhiều CA, dẫn tới thiết bị enroll nhầm CA không mong muốn.
- **SCEP endpoint dùng challenge password tĩnh theo alias** (không theo từng thiết bị) là điểm yếu nếu không rotate định kỳ — xem thêm gotcha ở [[pki--enrollment-protocols]].

## Security notes

- Coi mọi endpoint tự động hoá (REST/protocol) như 1 cổng issue cert **không có con người review từng request** — kiểm soát ở tầng End Entity Profile + Certificate Profile phải đủ chặt vì đây là lớp phòng thủ chính, không phải "sẽ có người check lại sau".
- Log đầy đủ mọi request qua các endpoint tự động — khi có sự cố, đây là nguồn điều tra chính (ai/cái gì đã enroll, lúc nào, từ đâu).

## Refs

- [[pki--enrollment-protocols]] — chi tiết cơ chế từng giao thức (ACME/SCEP/EST/CMP).
- [[ejbca--end-entities-ra]] — End Entity vẫn là đơn vị nền tảng, kể cả khi enrollment diễn ra qua API/protocol thay vì Web UI.
- Official REST API docs: docs.keyfactor.com (mục EJBCA REST Interface).
