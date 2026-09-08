---
tags:
  - cloudstack
  - identity
  - multi-tenancy
---

# Accounts, Domains & Projects (CloudStack)

CloudStack được thiết kế **multi-tenant từ gốc** — khác VMware vốn chỉ có khái niệm tổ chức tài nguyên qua Folder/Permission, không có mô hình tenant phân cấp sẵn có.

## Mô hình phân cấp

```
ROOT domain
 ├─ Domain "customer-a"
 │    ├─ Account "admin@customer-a"  (Domain Admin)
 │    │    └─ Project "web-app"      (chia sẻ resource giữa nhiều account)
 │    └─ Account "dev@customer-a"
 └─ Domain "customer-b"
      └─ Account "admin@customer-b"
```

| Khái niệm | Vai trò | Tương đương gần đúng |
|---|---|---|
| **Domain** | Đơn vị tổ chức phân cấp (có thể lồng domain con), thường = 1 khách hàng/phòng ban | Gần giống **Organization** trong vCloud Director, hoặc OU trong Active Directory |
| **Account** | Đơn vị sở hữu resource thực sự (VM, network, volume...), gắn với 1 hoặc nhiều user | Gần giống 1 **tenant/subscription** — KHÔNG giống 1 user đơn lẻ! |
| **User** | Một người dùng cụ thể, thuộc về 1 Account, có thể có nhiều user chung 1 account | Giống user trong AD nhưng luôn thuộc về đúng 1 Account |
| **Project** | Nhóm resource dùng chung bởi **nhiều Account khác nhau** trong cùng 1 Domain | Không có tương đương trực tiếp — gần giống 1 "shared resource group" xuyên tenant |

> [!warning] Nhầm lẫn cực kỳ phổ biến: Account ≠ User
> Dân mới hay nghĩ "Account" giống "User" (1 account = 1 người). Thực tế **Account là đơn vị sở hữu resource** (giống 1 tenant/công ty con), và **nhiều User có thể cùng thuộc 1 Account**, chia sẻ chung toàn bộ VM/network/quota của account đó. Xóa nhầm 1 Account (tưởng chỉ là xóa 1 user) sẽ **xóa toàn bộ VM và resource** account đó sở hữu — đây là 1 trong những thao tác nguy hiểm nhất cần cẩn trọng khi vận hành.

## Domain — cô lập & phân quyền quản trị theo tầng

```bash
cmk createDomain name=customer-a parentdomainid=<root-domain-id>

# Tạo account trong domain đó
cmk createAccount username=admin-a email=admin@customer-a.com \
  firstname=Admin lastname=A password=<pwd> \
  accounttype=2 domainid=<domain-id> account=customer-a-admin
```

- `accounttype`: `0` = Admin (Root Admin, thấy toàn hệ thống), `1` = Resource Domain Admin/User (tùy version), `2` = User thường (chỉ thấy resource trong scope của mình).
- Domain Admin chỉ quản lý được account **trong domain của mình và domain con**, không thấy domain khác — giống cách 1 OU Admin trong AD không quản được OU khác.

> [!tip] So với vCloud Director
> Domain + Account trong CloudStack gần đúng với **Organization + Org VDC** trong vCloud Director hơn là bất kỳ khái niệm nào trong vCenter thuần. Nếu công ty từng dùng vCD để cho thuê hạ tầng, mental model chuyển sang CloudStack sẽ nhanh hơn nhiều so với người chỉ quen vCenter nội bộ đơn thuần.

## Project — chia sẻ resource xuyên Account

```bash
cmk createProject name=web-app displaytext="Web App Team" domainid=<domain-id>

# Thêm account khác vào project (họ vẫn giữ account riêng, chỉ dùng chung resource project)
cmk addAccountToProject projectid=<project-id> account=dev@customer-a
```

> [!info] Khi nào dùng Project thay vì gộp chung 1 Account
> Dùng Project khi **nhiều team/account độc lập cần cộng tác trên cùng 1 tập VM/network** nhưng vẫn muốn giữ billing/quota riêng cho phần việc khác của họ ở account gốc. Nếu chỉ đơn giản là nhiều người dùng chung mọi thứ, thêm nhiều **User** vào chung 1 **Account** là đủ, không cần Project.

## Resource Limit (Quota) theo Domain/Account/Project

```bash
cmk updateResourceLimit account=customer-a-admin domainid=<domain-id> \
  resourcetype=0 max=20   # resourcetype 0 = số lượng VM instance

cmk listResourceLimits account=customer-a-admin
```

| resourcetype | Ý nghĩa |
|---|---|
| 0 | Instance (VM) |
| 1 | Public IP |
| 2 | Volume |
| 3 | Snapshot |
| 4 | Template |
| 6 | Network |
| 8 | CPU (core) |
| 9 | Memory (MB) |
| 10 | Primary Storage (GB) |
| 11 | Secondary Storage (GB) |

> [!warning] Lesson learned: quota set ở Domain không tự động giới hạn Account con chặt hơn domain
> Resource Limit có thể set ở cả cấp **Domain** và cấp **Account**, và chúng **không tự động cộng dồn/kiểm tra chéo hợp lý** như ta kỳ vọng — 1 Account có thể có quota riêng cao hơn cả domain cha nếu cấu hình không cẩn thận (tùy version, hành vi default resource limit type). Luôn kiểm tra **cả 2 cấp** khi debug "tại sao account này vẫn tạo được VM dù domain báo hết quota" hoặc ngược lại.

---
*Xem thêm: [[RBAC & Roles (CloudStack)]] | [[Service, Disk & Network Offerings]] | [[Cloudstack|CloudStack]]*
