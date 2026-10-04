# Provider Security — Hardening cho Nền Tảng KaaS
Tags: #k8s-provider #security #hardening #secrets #iam
Related: [[CAPI]], [[CAPC]], [[CAPO]], [[Talos]], [[Multi-Tenancy]], [[Management-Plane]]
Last updated: 2026-10-03

> 📌 **Phạm vi**: Note này khác với note `Security` generic ở top-level vault (vốn bàn tool bảo mật *trong* 1 cluster đơn như Kyverno/Falco/gVisor). Ở đây bàn attack surface đặc thù của **provider** — tức tầng *phía trên* mọi tenant cluster: credential IaaS, management plane, secret của toàn fleet. Chi tiết CVE/hardening riêng của CAPI core và Talos OS đã có ở [[CAPI]] (mục 7) và [[Talos]] (mục 7) — note này **tổng hợp góc nhìn toàn provider**, không lặp lại danh sách CVE, chỉ cross-reference.
>
> ⚠️ Phần "best practice IAM cho CAPC/CAPO" dưới đây dựa trên khuyến nghị chính thức của OpenStack (Application Credential) và tài liệu CloudStack (scoped sub-account) — đã verify là khuyến nghị chính thức, **không phải** suy luận cá nhân. Phần rotation tự động cho credential CAPC/CAPO cụ thể thì **chưa verify** — xem mục 6.

---

### **1. What — Nó là cái gì?**

Provider Security là tập hợp các biện pháp hardening cho riêng lớp "nhà cung cấp dịch vụ" trong mô hình KaaS — khác về bản chất so với bảo mật 1 cluster đơn, vì ở đây có **1 management plane duy nhất kiểm soát credential + lifecycle của nhiều cluster thuộc nhiều khách hàng khác nhau**. 3 mảng chính: (1) IAM/credential cho infra provider (CAPC/CAPO) truy cập CloudStack/OpenStack, (2) secret của Talos (machine config, disk encryption key), (3) RBAC/network policy tách tenant ở tầng CAPI.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Nếu chỉ áp dụng security checklist kiểu "hardening 1 cluster" (đã đủ dùng cho 1 công ty tự vận hành 1 cluster nội bộ) cho mô hình KaaS, sẽ bỏ sót đúng phần nguy hiểm nhất: **shared management plane**. Khác biệt cốt lõi:
- Vận hành 1 cluster: compromise cluster A chỉ ảnh hưởng chủ của cluster A.
- Vận hành KaaS: compromise **management cluster** (nơi CAPI + CAPC/CAPO + credential IaaS chạy) = attacker có đường tới kubeconfig admin của **mọi** tenant **cùng lúc** + credential dùng để tạo/xoá VM trên toàn hạ tầng CloudStack/OpenStack. Đây không còn là "incident của 1 khách hàng" mà là sự cố ở cấp *nhà cung cấp dịch vụ* — mức độ trách nhiệm pháp lý/hợp đồng khác hẳn.

Nếu không chủ động thiết kế IAM scoped + tách RBAC theo tenant từ đầu, sẽ phải vá lại kiến trúc quyền hạn sau khi đã có khách hàng thật — rủi ro và chi phí cao hơn nhiều so với thiết kế đúng từ lúc build management plane.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Áp dụng khi:** đã quyết định build management plane dùng chung cho nhiều tenant (chính là mô hình KaaS đang theo đuổi) — không có lựa chọn "không áp dụng" một khi đã chọn kiến trúc shared control plane.

**Cân nhắc khác đi khi:**
- Nếu mô hình thực tế là "1 management cluster riêng cho mỗi khách hàng lớn" (dedicated management plane) thay vì shared — thì phần threat model "compromise 1 điểm = mất toàn bộ khách hàng" không còn đúng theo nghĩa *toàn hệ thống*, nhưng vẫn đúng theo nghĩa *toàn bộ cluster của riêng khách đó*. Quyết định dedicated vs shared management plane nên làm trước khi áp checklist này → xem [[Multi-Tenancy]].
- Giai đoạn lab/PoC nội bộ (chưa có khách hàng thật, chưa multi-tenant) — áp đủ checklist IAM scoped vẫn nên làm (tập thói quen đúng từ đầu), nhưng RBAC tách namespace-per-tenant có thể làm sau khi có nhu cầu thật.

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
                         ┌─────────────────────────────────────────┐
                         │   Credential IaaS (root of trust #1)       │
                         │   - CloudStack: API key/secret sub-account │
                         │   - OpenStack: Application Credential      │
                         └──────────────────┬──────────────────────┘
                                            │ lưu dưới dạng K8s Secret
                                            ▼
┌───────────────────────────── Management Cluster ("root of trust #2") ─────────────────────┐
│                                                                                              │
│   CAPI core + CAPC/CAPO (ns capc-system/capo-system) + CABPT/CACPPT (Talos)                │
│        │                                      │                                             │
│        │ RBAC theo namespace-per-tenant        │ gọi API IaaS bằng credential trên           │
│        ▼ (xem Multi-Tenancy)                   ▼                                             │
│   Namespace tenant-A    Namespace tenant-B    ...  →  mỗi Cluster CR + Secret kubeconfig     │
│   (chỉ chứa Cluster/     (chỉ chứa Cluster/                riêng của từng tenant             │
│    Machine của A)         Machine của B)                                                     │
└───────────────────────┬──────────────────────────────┬──────────────────────────────────────┘
                         ▼                               ▼
              Tenant Cluster A (Talos)          Tenant Cluster B (Talos)
              - STATE partition (secrets)        - STATE partition (secrets)
              - EPHEMERAL (encrypt qua KMS?)      - EPHEMERAL (encrypt qua KMS?)
```

**Điểm mấu chốt**: có **2 "root of trust"** xếp lớp — credential IaaS gốc (ai tạo được account/application-credential mới) và management cluster (nơi credential đó được dùng hàng ngày). Compromise **một trong hai** đều dẫn tới blast radius toàn provider. Chi tiết attack surface riêng của management cluster (RBAC provider controller, bootstrap data) → [[CAPI]] mục 7; chi tiết attack surface riêng của node Talos (apid, role-based cert) → [[Talos]] mục 7.

### **5. How — Cơ chế hoạt động**

- **Credential IaaS scoped** — không dùng account cloud-admin cho CAPC/CAPO. OpenStack khuyến nghị chính thức dùng **Application Credential** (tạo qua `openstack application credential create`, scope theo đúng 1 project, revoke độc lập không ảnh hưởng user gốc) thay vì username/password tĩnh. CloudStack khuyến nghị tạo **sub-account riêng** (không dùng account domain-admin cấp API key) rồi gán **dynamic role** chỉ đủ quyền tạo/xoá VM+network+LB trong scope cần — root admin có thể tắt API-key access toàn cục và chỉ bật cho account cụ thể.
- **Secret lưu trong management cluster** — credential trên được CAPC/CAPO đọc từ K8s Secret (namespace riêng của provider). Đây là lý do RBAC đọc Secret ở namespace `capc-system`/`capo-system` phải hẹp nhất có thể (xem [[CAPI]] mục 6/7).
- **RBAC/namespace per tenant** — mỗi tenant có 1 namespace riêng trong management cluster, chỉ chứa `Cluster`/`Machine`/`MachineDeployment` của tenant đó; RoleBinding namespace-scoped cho tenant (nếu cho self-service), `ClusterRoleBinding` chỉ dành cho "break-glass admin" của platform. Giới hạn thật: namespace-level isolation **không** cô lập được control-plane/API-server của management cluster — mọi tenant vẫn dùng chung CRD, chung scheduler, chung webhook của CAPI core, nên 1 CRD conflict hoặc 1 bug ở core vẫn ảnh hưởng chéo được giữa tenant. Mức isolation mạnh hơn (dedicated management plane/hosted control plane) → [[Multi-Tenancy]].
- **Talos secret** — machine config (chứa join token, cert) nằm trong `STATE` partition của từng node; disk encryption (KMS hoặc static key) bảo vệ lớp này ở mức OS — chi tiết đầy đủ → [[Talos]] mục 6/7.

### **6. Key Config — Cấu hình cần nhớ**

- **Credential rotation cho CAPC/CAPO**: CAPI/CAPC/CAPO **không có cơ chế rotation tự động built-in** cho Secret chứa credential IaaS — rotate vẫn phải tự làm (tạo credential mới → update Secret → xoá credential cũ) hoặc dùng công cụ ngoài như **External Secrets Operator (ESO)** để sync từ Vault/cloud secret manager vào K8s Secret theo interval. ⚠️ **Cần verify**: CAPC/CAPO có tự động reload credential khi Secret đổi hay phải restart pod controller — hành vi "hot reload" khi Secret cập nhật tuỳ theo cách provider đọc Secret (watch vs đọc 1 lần lúc start), chưa kiểm tra kỹ cho từng provider.
- OpenStack Application Credential hỗ trợ **rotation không downtime**: tạo credential mới, rollout, xoá credential cũ sau — do 2 credential có thể cùng active 1 lúc. CloudStack API key/secret thì **không có khái niệm 2 key cùng active** theo mặc định — đổi key nghĩa là key cũ mất hiệu lực ngay, cần verify kỹ quy trình rolling update Secret trong K8s để tránh gây downtime cho CAPC.
- `clouds.yaml` (OpenStack) và CloudStack API key/secret đều là **plaintext trong Secret** (chỉ base64, không mã hoá) trừ khi cluster bật K8s Secret encryption-at-rest (EncryptionConfiguration + KMS provider) — mặc định etcd của management cluster **không** mã hoá Secret.
- Nếu cho tenant self-service tạo `Cluster` trực tiếp (không qua lớp API riêng), phải validate kỹ `spec.topology.variables` (nếu dùng ClusterClass) để tenant không tự inject giá trị ngoài dự kiến ảnh hưởng tới provider — xem thêm `capi--clusterclass-topology` (file con của [[CAPI]]).

### **7. Security Considerations**

- **Tổng hợp attack surface toàn provider** (không lặp lại chi tiết, chỉ liệt kê theo lớp — chi tiết từng lớp đã có ở note con):
  1. Credential IaaS gốc (ai tạo account/application-credential) — nằm ngoài phạm vi K8s, thuộc IAM của CloudStack/OpenStack.
  2. Management cluster (CAPI + CAPC/CAPO + Secret credential + kubeconfig mọi tenant) → chi tiết [[CAPI]] mục 7.
  3. Node Talos (apid API, machine config secret, disk encryption) → chi tiết [[Talos]] mục 7.
  4. Ranh giới RBAC giữa tenant trong management cluster (namespace isolation **không đầy đủ**, như nêu ở mục 5).
- **Misconfiguration nguy hiểm nhất riêng ở tầng provider**: dùng 1 account cloud-admin duy nhất cho CAPC/CAPO (tiện lúc setup, nhưng credential này rò ra = attacker điều khiển được *toàn bộ* CloudStack/OpenStack, không chỉ phần dành cho K8s). Đây là lỗi thường gặp nhất khi mới lab vì "account admin luôn chạy được", nhưng phải sửa trước khi production.
- **Hardening checklist tối thiểu (bổ sung, không lặp CAPI/Talos)**:
  1. Tạo Application Credential (OpenStack) / sub-account + dynamic role scoped (CloudStack) riêng cho CAPC/CAPO — không dùng credential admin.
  2. Bật Secret encryption-at-rest cho management cluster (etcd encryption provider) — vì mặc định Secret chỉ base64.
  3. Nếu có nhiều tenant thật, đánh giá nghiêm túc giữa namespace-isolation (rẻ, nhưng không cô lập control-plane) vs dedicated/hosted control-plane per tenant (đắt hơn, cô lập thật) → [[Multi-Tenancy]].
  4. Thiết lập quy trình rotation credential IaaS định kỳ (thủ công hoặc qua ESO) — đừng để credential sống vĩnh viễn từ lúc setup ban đầu.
  5. Audit riêng ai có quyền đọc Secret ở namespace `capc-system`/`capo-system`/`capi-system` — đây là nhóm người có khả năng *gián tiếp* lấy được credential IaaS dù không phải cloud-admin.

### **8. Ops Runbook — Production Notes**

- **Health check định kỳ (bổ sung an ninh)**: rà soát Secret trong `capc-system`/`capo-system` xem credential còn hạn/còn hợp lệ (đặc biệt Application Credential OpenStack có thể set thời hạn); rà soát RoleBinding/ClusterRoleBinding trong management cluster định kỳ để phát hiện quyền dư thừa tích tụ theo thời gian (privilege creep).
- **Khi nghi ngờ credential IaaS bị rò**: (1) revoke/xoá credential đó ngay ở CloudStack/OpenStack (Application Credential revoke độc lập không cần đổi password user, sub-account CloudStack disable API key được); (2) tạo credential mới, update Secret trong management cluster; (3) audit log API IaaS xem có hoạt động tạo VM/network bất thường trong khoảng thời gian nghi ngờ không.
- **Khi nghi ngờ management cluster bị compromise**: coi như **toàn bộ** credential IaaS + kubeconfig mọi tenant đã lộ — quy trình ứng cứu phải ở mức "incident toàn provider", không phải incident 1 cluster: revoke toàn bộ credential IaaS, rotate toàn bộ kubeconfig tenant, rebuild management cluster từ backup đã biết sạch → liên quan trực tiếp tới quy trình DR ở [[Fleet-Ops]].
- Không có metric/alert chuẩn hoá sẵn cho riêng lớp "provider security" này — cần tự dựng (vd alert khi Secret trong `*-system` namespace bị đọc bởi ServiceAccount lạ, qua audit log K8s).

### **9. Gotchas & Lessons Learned**

- Dễ nhầm rằng "đã hardening CAPI + đã hardening Talos" = đã an toàn — nhưng khoảng trống thật nằm ở **tầng nối giữa 2 thứ đó với IAM của IaaS**, thứ không thuộc phạm vi tài liệu CAPI/Talos (2 project đó không biết gì về cách CloudStack/OpenStack cấp quyền).
- Namespace-per-tenant trong management cluster tạo cảm giác "đã multi-tenant an toàn", nhưng như nêu ở mục 5, đây **không phải** isolation thật ở tầng control-plane — nếu hợp đồng/compliance với khách hàng yêu cầu isolation mức cao, namespace không đủ, phải xem [[Multi-Tenancy]] cho mô hình dedicated/hosted control-plane.
- ⚠️ Phần "CAPC/CAPO tự reload credential khi Secret đổi" chưa verify — nếu thực tế là đọc 1 lần lúc start, thì "rotate Secret" tưởng đã xong nhưng controller vẫn dùng credential cũ trong memory tới khi restart, tạo ra false sense of security. Phải tự test hành vi này trên hệ thống thật trước khi tin vào quy trình rotation.
- (Mục này để tự bổ sung tiếp sau khi thao tác trực tiếp — lesson learned thực chiến quan trọng hơn note lý thuyết)

### **10. Resources**

- OpenStack Application Credentials (spec gốc): https://specs.openstack.org/openstack/keystone-specs/specs/keystone/queens/application-credentials.html
- OpenStack Application Credentials — Identity API v3: https://docs.openstack.org/api-ref/identity/v3/index.html?expanded=authenticating-with-an-application-credential-detail
- CloudStack Roles, Accounts, Users, and Domains: https://docs.cloudstack.apache.org/en/latest/adminguide/accounts.html
- CAPI Security Guidelines (chung cho infra provider): https://cluster-api.sigs.k8s.io/developer/providers/security-guidelines
- External Secrets Operator: https://external-secrets.io/
- Parent/Related: [[CAPI]], [[Talos]], [[CAPC]], [[CAPO]], [[Multi-Tenancy]]
- **Lưu ý khi dùng note này**: phần IAM scoped (Application Credential, CloudStack sub-account) đã verify là khuyến nghị chính thức từ tài liệu OpenStack/CloudStack. Phần rotation tự động/hot-reload credential của riêng CAPC/CAPO **chưa verify trực tiếp từ source code/docs provider** — xem mục 6 và 9 trước khi thiết kế quy trình rotation thật.
