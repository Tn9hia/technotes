---
tags:
  - cloudstack
  - identity
  - rbac
---

# RBAC & Roles (CloudStack)

## Role mặc định

| Role | Phạm vi | Tương đương VMware |
|---|---|---|
| **Root Admin** | Toàn quyền toàn hệ thống, mọi Domain | Giống `Administrator@vsphere.local` |
| **Domain Admin** | Toàn quyền trong 1 Domain (và domain con) | Gần giống phân quyền theo Folder/OU trong vCenter, nhưng có sẵn scope rõ ràng hơn |
| **Resource Admin** (tùy version) | Quản lý resource nhưng giới hạn hơn Domain Admin | — |
| **User** | Chỉ thao tác resource của chính Account mình | Giống 1 role custom "self-service" hạn chế trong vCenter |

## Dynamic Roles (từ CloudStack 4.11+) — quan trọng với các bản mới

Các bản mới của CloudStack hỗ trợ **Dynamic Role-Based Access Control**: tạo role tùy chỉnh, gán/chặn từng API call cụ thể — linh hoạt hơn nhiều so với 4 role tĩnh truyền thống.

```bash
# Tạo role tùy chỉnh
cmk createRole name=network-operator type=User description="Chỉ quản lý network, không đụng VM"

# Gán rule cho phép/từ chối từng API
cmk createRolePermission roleid=<role-id> rule="listNetworks" permission=allow
cmk createRolePermission roleid=<role-id> rule="createNetwork" permission=allow
cmk createRolePermission roleid=<role-id> rule="deployVirtualMachine" permission=deny

# Gán role cho account
cmk updateAccount account=netops-user domainid=<domain-id> roleid=<role-id>
```

> [!tip] So với VMware Custom Roles
> Cơ chế tương tự **Custom Roles trong vCenter** (chọn từng privilege cụ thể), nhưng CloudStack thao tác ở granularity **theo API call** (VD: `deployVirtualMachine`, `listNetworks`) thay vì theo object/privilege category như vCenter. Muốn biết chính xác 1 role được phép làm gì, phải liệt kê hết `RolePermission` gắn với nó — không có UI phân nhóm privilege trực quan mạnh như vCenter.

> [!warning] Lesson learned: rule càng cụ thể càng phải để đúng thứ tự & default deny
> Giống Network ACL, `RolePermission` cũng có khái niệm rule cụ thể hơn nên đặt trước, và role tùy chỉnh nên có **rule mặc định deny ở cuối** rồi mở dần từng API cần thiết (nguyên tắc least-privilege), thay vì bắt đầu từ allow-all rồi deny bớt — dễ sót lỗ hổng nếu làm ngược.

## API Key / Secret Key — xác thực cho automation

Mỗi User có 1 cặp **API Key/Secret Key** riêng (khác với password đăng nhập UI), dùng để ký HMAC-SHA1 cho request API.

```bash
cmk listApiKeys account=customer-a-admin
# Sinh lại key mới (revoke key cũ ngay lập tức!)
cmk registerUserKeys id=<user-id>
```

> [!warning] Lesson learned: regenerate API key làm gãy mọi automation đang dùng key cũ
> Vì `registerUserKeys` sinh cặp key **mới và vô hiệu hóa key cũ ngay lập tức**, mọi script/Terraform provider/Ansible playbook đang dùng key cũ sẽ **fail authentication ngay** không báo trước cho người vận hành script đó. Trước khi regenerate key của 1 account đang được dùng cho automation, phải xác nhận và cập nhật đồng bộ ở mọi nơi consume key đó — đây là nguyên nhân phổ biến của các sự cố "tự nhiên pipeline CI/CD gãy" không rõ lý do.

## LDAP / SAML Integration

CloudStack hỗ trợ tích hợp **LDAP** (Active Directory) và **SAML SSO** — hữu ích khi công ty đã có AD sẵn từ hạ tầng VMware/Windows.

```bash
cmk ldapConfig hostname=ldap.company.local port=389 \
  basedn="DC=company,DC=local" searchbase="DC=company,DC=local"

cmk importLdapUsers domainid=<domain-id> accounttype=2
```

> [!tip] Không tự động đồng bộ liên tục
> Khác kỳ vọng "tích hợp AD thì mọi thay đổi group/user tự cập nhật realtime" — import LDAP user vào CloudStack thường là thao tác **theo lô (batch import)**, không phải sync liên tục hai chiều. User bị xóa/disable bên AD **không tự động** bị khóa trong CloudStack trừ khi có job đồng bộ định kỳ hoặc script riêng — cần lưu ý về khoảng trễ này khi thiết kế quy trình offboarding nhân sự.

---
*Xem thêm: [[Accounts, Domains & Projects (CloudStack)]] | [[CloudStack Security Considerations]] | [[Cloudstack|CloudStack]]*
