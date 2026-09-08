---
tags:
  - cloudstack
  - comparison
  - vmware
  - openstack
---

# CloudStack vs VMware vs OpenStack

## Bảng so sánh tổng quan

| Tiêu chí | Apache CloudStack | VMware (vSphere + vCD/NSX) | OpenStack |
|---|---|---|---|
| Giấy phép | Mã nguồn mở, miễn phí (Apache 2.0) | Thương mại, license theo core/CPU | Mã nguồn mở, miễn phí |
| Độ phức tạp triển khai | Trung bình — ít service rời rạc hơn OpenStack | Thấp (nếu dùng đúng sản phẩm VMware chuẩn, có vendor support) | Cao — nhiều service (Nova, Neutron, Cinder...) phải tích hợp với nhau |
| Đa hypervisor | Có (KVM, VMware, XenServer/XCP-ng, Hyper-V) | Chỉ ESXi | Chủ yếu KVM (hỗ trợ khác nhưng ít phổ biến) |
| Multi-tenancy sẵn có | Có sẵn từ gốc (Domain/Account/Project) | Cần thêm vCloud Director/Aria Automation | Có sẵn (Project/Domain trong Keystone) |
| Networking nâng cao (SDN, micro-segmentation) | Cơ bản-trung bình (VR, VPC, Security Group) | Rất mạnh nếu có NSX (distributed FW, overlay phức tạp) | Mạnh, linh hoạt (Neutron + OVN/OVS), nhưng phức tạp vận hành |
| Cộng đồng & hệ sinh thái | Nhỏ hơn OpenStack, ổn định, ít thay đổi kiến trúc lớn | Hệ sinh thái vendor lớn, support thương mại mạnh | Cộng đồng lớn nhất trong nhóm mã nguồn mở, nhiều vendor tham gia (Red Hat, Canonical...) |
| Tốc độ cập nhật tính năng mới | Chậm, thận trọng | Nhanh, theo roadmap thương mại rõ ràng | Nhanh nhưng phân mảnh, mỗi dự án con tốc độ khác nhau |
| Chi phí vận hành | Thấp license, cần đội ngũ kỹ thuật tự vận hành | Cao license, nhưng giảm chi phí đội ngũ (vendor support tốt) | Thấp license, cần đội ngũ kỹ thuật rất mạnh, chi phí ẩn cao |
| Độ trưởng thành HA/Live Migration | Tốt, đơn giản, ổn định | Rất trưởng thành (vSphere HA/DRS/vMotion hàng chục năm phát triển) | Tốt nhưng cấu hình phức tạp hơn (Pacemaker, Galera...) |
| Phù hợp quy mô | Vừa đến lớn, đặc biệt hosting/service provider | Doanh nghiệp mọi quy mô, đặc biệt khi cần support thương mại | Rất lớn, đặc biệt telco/public cloud tự vận hành |

## Ưu điểm CloudStack

- **Đơn giản hơn OpenStack đáng kể**: ít service rời rạc, kiến trúc gói gọn trong 1 Management Server + Agent — thời gian học và vận hành nhanh hơn.
- **Đa hypervisor thật sự**: cho phép chiến lược di dân dần từ VMware sang KVM mà không cần big-bang migration (chạy song song 2 loại cluster trong cùng zone).
- **Multi-tenancy & self-service có sẵn từ đầu**: không cần mua thêm sản phẩm riêng như vCD để có khái niệm tenant/project.
- **Chi phí license = 0**, phù hợp khi ngân sách hạn chế nhưng vẫn cần tính năng cloud tự phục vụ.
- **API đơn giản, nhất quán**: 1 kiểu xác thực (HMAC), 1 kiểu async job cho toàn bộ hệ thống — dễ tự động hóa hơn việc phải học nhiều API con như OpenStack.

## Nhược điểm CloudStack

- **Hệ sinh thái/cộng đồng nhỏ hơn** OpenStack và VMware — ít tài liệu bên thứ 3, ít công cụ giám sát/tích hợp sẵn có (phải tự xây nhiều thứ, VD: Prometheus exporter không chính thức).
- **SDN/networking kém linh hoạt hơn NSX hoặc Neutron+OVN**: Virtual Router đơn giản, không có distributed routing/firewall mạnh, khó đáp ứng yêu cầu network phức tạp cấp doanh nghiệp lớn.
- **Không có vendor support thương mại mạnh như VMware** (dù có vài công ty cung cấp support CloudStack, quy mô nhỏ hơn nhiều so với hệ sinh thái Broadcom/VMware).
- **Tốc độ phát triển tính năng chậm hơn**, một số tính năng hiện đại (backup framework thống nhất, K8s-as-a-Service) đến muộn hơn so với OpenStack/VMware Tanzu.
- **Ít lựa chọn storage plugin thương mại** hơn OpenStack Cinder (Cinder có rất nhiều driver vendor chính thức).

## Khi nào chọn CloudStack thay vì VMware

> [!tip] Trường hợp điển hình
> - Muốn thoát dần khỏi chi phí license VMware (đặc biệt sau các đợt tăng giá license) nhưng chưa sẵn sàng/không đủ nhân lực cho độ phức tạp của OpenStack.
> - Cần cung cấp dịch vụ tự phục vụ (self-service) đa khách hàng/đa phòng ban mà không muốn đầu tư thêm vCloud Director/Aria Automation.
> - Đội ngũ đã quen Linux/KVM, không ngại tự vận hành hạ tầng thay vì phụ thuộc hoàn toàn vào vendor support.

## Khi nào VMware vẫn là lựa chọn hợp lý hơn

> [!tip] Trường hợp điển hình
> - Cần độ ổn định/tính năng đã được kiểm chứng qua hàng chục năm (DRS, vMotion, vSAN, NSX) cho khối lượng công việc cực kỳ quan trọng (mission-critical), rủi ro downtime là không chấp nhận được.
> - Có ngân sách cho license và ưu tiên **support thương mại nhanh, SLA rõ ràng** hơn là tự chủ hoàn toàn về công nghệ.
> - Đội ngũ vận hành quen hệ sinh thái VMware sâu, chi phí đào tạo lại đội ngũ sang Linux/KVM cao hơn giá trị tiết kiệm được từ license.

## Khi nào OpenStack phù hợp hơn CloudStack

> [!tip] Trường hợp điển hình
> - Quy mô cực lớn (hàng nghìn+ host), cần networking SDN sâu (multi-region, complex routing, telco-grade NFV) — Neutron/OVN linh hoạt hơn nhiều so với Virtual Router của CloudStack.
> - Cần hệ sinh thái dịch vụ đa dạng có sẵn (Object Storage/Swift, Bare Metal/Ironic, Orchestration/Heat, Load Balancer/Octavia...) mà không muốn tự tích hợp bên thứ 3.
> - Có đội ngũ đủ lớn và đủ chuyên sâu để chấp nhận độ phức tạp vận hành cao hơn, đổi lại được sự linh hoạt và hệ sinh thái vendor phong phú (Red Hat OpenStack, Canonical...).

## Tóm tắt 1 câu cho mỗi nền tảng

> [!info] Ghi nhớ nhanh
> - **VMware**: "Trả tiền để mọi thứ ổn định, trưởng thành, có người support" — tối ưu cho độ tin cậy và giảm rủi ro vận hành.
> - **CloudStack**: "Đơn giản, đủ dùng, miễn phí, đa hypervisor" — tối ưu cho chi phí và tốc độ triển khai self-service mà không cần đội ngũ khổng lồ.
> - **OpenStack**: "Linh hoạt tối đa, hệ sinh thái đầy đủ, đổi lại độ phức tạp vận hành cao nhất" — tối ưu cho quy mô cực lớn và tùy biến sâu.

---
*Xem thêm: [[Hypervisor Support - KVM, VMware & Others]] | [[Basic vs Advanced Networking]] | [[Cloudstack|CloudStack]] | [[Cloud/OpenStack/OpenStack|OpenStack]]*
