# EJBCA — Architecture & Components
Tier: 2
Parent: [[EJBCA]]
Related: [[pki--hsm-key-protection]], [[ejbca--profiles]], [[ejbca--api-cli]]
Tags: #ejbca #architecture

## What it does

Mô tả các thành phần vật lý/logic tạo nên 1 hệ thống EJBCA đang chạy: app server, database, crypto token (HSM/soft), và (ở bản Enterprise) khả năng tách node RA riêng khỏi node CA qua cơ chế **Peer connector**.

## Why it exists

EJBCA cần tách rõ "nơi lưu dữ liệu" (DB), "nơi giữ key" (Crypto Token), và "nơi xử lý logic" (app server) để có thể: (1) bảo vệ key độc lập với việc bảo vệ dữ liệu thường, (2) scale/HA từng phần riêng biệt, (3) cho phép mô hình triển khai mà node RA (tiếp xúc nhiều với bên ngoài, rủi ro cao hơn) tách biệt vật lý khỏi node CA (giữ key, cần bảo vệ nghiêm ngặt nhất).

## How it works (flow/diagram)

```
┌─────────────────────────────────────────────────────────┐
│  EJBCA Node                                                │
│  - Chạy trên WildFly/JBoss (Java EE app server)             │
│  - Hoặc container image (EJBCA Enterprise cung cấp          │
│    Docker/Helm chart chính thức)                            │
└───────────────┬──────────────────────┬─────────────────────┘
                │                      │
      ┌─────────▼─────────┐   ┌────────▼─────────┐
      │  Database          │   │  Crypto Token      │
      │  (MariaDB/         │   │  - SOFT: keystore   │
      │  PostgreSQL/       │   │    file mã hoá        │
      │  Oracle...)        │   │    password, lưu       │
      │  Lưu: cert, CA      │   │    trong DB/filesystem │
      │  metadata, End      │   │  - PKCS#11: HSM vật    │
      │  Entity, audit log, │   │    lý hoặc network HSM │
      │  Profile config     │   │    (Thales, Utimaco,   │
      │                     │   │    AWS CloudHSM...)     │
      └────────────────────┘   └─────────────────────────┘
```

**Mô hình phân tán (Enterprise, cluster lớn):**
```
     [RA Node 1]  [RA Node 2]   ← xử lý enrollment, tiếp xúc
          │             │          client/internet nhiều hơn
          └──────┬──────┘
                 │ Peer connector (kênh riêng, xác thực
                 │ bằng mTLS giữa các node EJBCA)
          ┌──────▼──────┐
          │  CA Node     │   ← giữ Crypto Token/HSM, không cần
          │  (offline    │      expose trực tiếp ra ngoài,
          │  hơn, ít mở  │      chỉ nhận request qua Peer
          │  port ra     │      connector từ RA node đã xác thực
          │  ngoài)      │
          └──────────────┘
```

Mô hình này cho phép RA node (rủi ro tấn công cao hơn vì tiếp xúc bên ngoài) không cần quyền truy cập trực tiếp vào Crypto Token — mọi yêu cầu ký phải đi qua Peer connector đã xác thực, giảm attack surface tới CA node thực sự giữ key.

## Config gotchas

- **1 CA luôn gắn với đúng 1 Crypto Token** — đổi/di chuyển Crypto Token cho 1 CA đang hoạt động là thao tác nhạy cảm, cần lên kế hoạch kỹ (downtime issue trong lúc chuyển).
- **Soft Crypto Token password** — nếu dùng soft keystore (không HSM) cho CA production, password mở khoá token phải quản lý tách biệt khỏi DB backup (backup DB + password cùng chỗ = vô hiệu hoá luôn lớp bảo vệ).
- **Kết nối PKCS#11 tới HSM external** cần cấu hình đúng thư viện driver (`.so`/`.dll` của vendor HSM) khớp version — sai driver là nguyên nhân phổ biến khiến CA "start được nhưng không ký được".
- **App server (WildFly) tuning** — heap size, connection pool tới DB cần tune theo tải enrollment thực tế; mặc định cho môi trường nhỏ có thể không đủ khi có batch issue lớn.

## Security notes

- Node giữ Crypto Token/HSM nên hạn chế network exposure tối đa (không cần public-facing) — chỉ RA/App tier mới cần tiếp xúc bên ngoài.
- Audit log lưu trong DB cùng dữ liệu nghiệp vụ — nên export định kỳ ra hệ thống log tập trung (SIEM) để tránh trường hợp attacker chiếm được DB cũng xoá sạch được audit trail.

## Refs

- [[pki--hsm-key-protection]] — lý thuyết nền về HSM.
- [[ejbca--api-cli]] — Peer connector cũng là 1 dạng giao tiếp giữa các node, liên quan tới REST/CLI khi tự động hoá.
