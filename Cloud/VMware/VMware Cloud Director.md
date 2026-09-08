## Thuật ngữ
### 1. Provider
**Ai đây?**  
Là bên cung cấp tài nguyên vật lý – tụi mình gọi là **cloud operator**, kiểu Viettel IDC ấy. Là người quản lý hạ tầng vật lý/thực tế như vSphere, ESXi, NSX, storage backend...
**Quyền lực:** Full quyền. Tạo, cấu hình mọi thứ, bao gồm cả các **Organization**, **Provider VDC**, network, catalog,...
### 2. Organization (Org)
**Ai đây?**  
Là "doanh nghiệp khách hàng" – giống như một tenant lớn chứa toàn bộ user, VM, network... của 1 công ty.
**Phân quyền:**
- Mỗi Org có thể có nhiều user riêng biệt.
- Mỗi user có thể có quyền **Org Admin**, **Catalog Author**, **vApp User**, v.v.

### 3. Tenant
**Thực ra là gì?**  
Trong bối cảnh VCD, **tenant = Organization**. Hai từ này hay dùng thay nhau, nhưng cốt lõi là **tenant là khách thuê dịch vụ cloud**, được đại diện bởi **Organization**.
> **Note:** Nếu tài liệu hoặc dev nào nói “tenant”, cứ hiểu là “Organization” cho dễ sống.

### 4. Organization Administrator

**Quyền gì?**  
Là người quản lý toàn bộ trong 1 Org cụ thể (tenant cụ thể).  
Không quản lý được mấy thứ bên ngoài Org đó (ví dụ: không đụng vào hạ tầng vSphere, NSX Provider...)

### 5. Global Roles / Rights Bundles
**Khái niệm quan trọng!**  
VMware VCD sử dụng 2 kiểu để kiểm soát quyền:
- **Global Roles**: Những role được định nghĩa sẵn (Admin, Console Access Only, vApp Author,...)
- **Rights Bundles**: Là bộ quyền chi tiết (granular) do Provider định nghĩa. Nó giúp kiểm soát sâu hơn (ví dụ: cho phép user A được thao tác vApp nhưng không được tạo Org VDC).
Cậu có thể clone, custom những rights bundles này, gán vào Role rồi gán cho user.
### 6. Provider VDC (PVDC)
Cái này thuộc **Provider**, nó là một abstract layer tập hợp compute + storage từ vSphere.
###  7. Organization VDC (OrgVDC)
Là tài nguyên cloud được cấp phát cho một Organization (tenant). Org dùng nó để tạo VM, vApp, network, etc.
> OrgVDC là chỗ user thao tác nhiều nhất.
### 8. vApp
**vApp** là một tổ chức logic bao gồm một hoặc nhiều VM – có thể gán policy, startup order, network riêng,...

---
### 9. User Roles (trong Org) – một vài ví dụ:

- **Organization Administrator** – toàn quyền trong Org đó.
- **Catalog Author** – tạo/sửa/xóa catalog.
- **vApp Author** – tạo/sửa vApp.
- **Console Access Only** – chỉ được mở VM console, không quyền thao tác.
- **Read-Only User** – khỏi nói 
## Allocation model
| Tính năng             | Allocation Pool    | Reservation Pool               | Pay-As-You-Go                 |
| --------------------- | ------------------ | ------------------------------ | ----------------------------- |
| Oversubscribe được?   | ✅ Có               | ❌ Không                        | ✅ Có                          |
| Commit bao nhiêu?     | Một phần           | Full                           | Khi dùng mới commit           |
| Quản lý resource pool | Cần theo dõi       | Dễ predict                     | Khó kiểm soát nếu không limit |
| Hiệu năng             | Trung bình         | Cao                            | Biến động                     |
| Phù hợp với           | SMBs, public cloud | Enterprise, critical workloads | Dev, trial, dynamic workloads |

Note the different network types available for OVDC networks: 
- *Routed* - creates a segment in NSX-T and routes through an existing (pre-configured) Edge gateway 
- *Isolated* - standalone segment created in NSX-T
- *Direct* - directly connects to pre-configured External Network (can be a vSphere Port Group or an NSX-T segment) 
- *Imported* - uses an existing (pre-configured) vDS Port Group or NSX-T Segment