# Cluster API Provider CloudStack (CAPC) — v0.6.x
Tags: #k8s-provider #capc #cloudstack #infra-provider
Related: [[CAPI]], [[CAPO]], [[Talos]], [[Networking]], [[Storage]], [[Provider-Security]]
Last updated: 2026-10-03

> ⚠️ **Version note — CẦN VERIFY LẠI**: Bản mới nhất tìm được qua search là **v0.6.1** (ghi "15 Jul", năm cụ thể chưa chắc chắn là 2026 — cần xem trực tiếp https://github.com/kubernetes-sigs/cluster-api-provider-cloudstack/releases). CAPC **vẫn pre-1.0** (0.x) — API đang trong quá trình promote `v1beta2` → `v1beta3` → `v1beta4` để bắt kịp CAPI core contract `v1beta2` (PR #493, "Upgrade to Cluster API v1.13 (v1beta2 contract) and promote API to v1beta4"). Nghĩa là field/CRD của CAPC **còn đổi nhanh hơn CAPI core** — đừng copy YAML example từ blog cũ (viết theo `v1alpha3`/`v1beta1`) mà không đối chiếu lại version provider đang cài.
>
> Maintainer/reviewer hiện tại (theo GitHub, chưa verify đầy đủ mức độ active của từng người): tổ hợp người từ **AWS, ShapeBlue (công ty dịch vụ CloudStack), và cộng đồng Apache CloudStack**, không phải 1 vendor lớn duy nhất đứng sau toàn thời gian như CAPO (SUSE/Red Hat). ⚠️ Đây là điểm cần tự đánh giá lại độ "bền" của project trước khi cam kết production dài hạn.

---

### **1. What — Nó là cái gì?**

CAPC là **Infrastructure Provider** của CAPI cho **Apache CloudStack** — dịch các object trừu tượng của CAPI (`Cluster`, `Machine`) thành resource thật trên CloudStack: VM instance, Isolated/Shared Network, Affinity Group. Sống dưới `kubernetes-sigs/cluster-api-provider-cloudstack`, cài qua `clusterctl init --infrastructure cloudstack`. Trong stack k8s-provider, CAPC là 1 trong 2 lựa chọn Infrastructure Provider (cùng [[CAPO]] cho OpenStack) để nối CAPI với tầng KVM qua CloudStack.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Nếu không có CAPC, muốn CAPI tạo VM trên CloudStack phải tự viết Infrastructure Provider riêng (implement đúng contract CRD + controller mà CAPI core gọi vào) — là 1 dự án kỹ thuật riêng biệt, không nhỏ. Thay vì vậy CAPC đã làm sẵn phần dịch "CAPI Machine → CloudStack VM" kèm theo:
- Quản lý network layer tự động (tạo Isolated Network nếu chưa có, gắn vào VPC nếu cần).
- Affinity Group — map vào khái niệm "host anti-affinity" của CloudStack, giúp control-plane node không bị đặt cùng 1 host vật lý.
- Failure Domain = CloudStack Zone — cho phép 1 `Cluster` trải nhiều zone nếu CloudStack setup có nhiều zone.

Đây chính là lớp "biết nói chuyện với CloudStack API" mà không cần tự viết, đổi lại phải chấp nhận giới hạn/API-maturity của 1 project pre-1.0 (xem mục 3/9).

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng khi:** hạ tầng KVM đã quản lý bằng **Apache CloudStack**, dùng **Advanced Zone** (không phải Basic Zone), và chấp nhận API provider còn thay đổi (pre-1.0) đổi lại được tích hợp CAPI chính thức trong org `kubernetes-sigs`.

**KHÔNG dùng khi:**
- CloudStack zone đang chạy ở **Basic Zone** hoặc **Advanced Zone có Security Groups** — theo doc chính thức, CAPC hiện **chỉ hỗ trợ Advanced Zone KHÔNG dùng Security Groups**. ⚠️ Nếu hạ tầng CloudStack hiện tại dùng Security Groups isolation (phổ biến ở setup nhỏ/KVM không hỗ trợ VLAN), phải đổi mô hình network CloudStack trước, không phải lỗi của CAPC mà là giới hạn thiết kế.
- Cần API ổn định lâu dài không đổi field — CAPC đang promote API version (`v1beta2→v1beta4`) để bắt kịp CAPI `v1beta2` contract, tức sắp có breaking change ở tầng CRD của chính CAPC, không chỉ CAPI core.
- Team ưu tiên provider có 1 vendor lớn bảo trì ổn định dài hạn hơn cộng đồng đa nguồn — nên so sánh thực tế mức độ active với [[CAPO]] tại thời điểm quyết định, đừng giả định "cả 2 đều do kubernetes-sigs host nên ngang hàng".

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
┌────────────────────────── Management Cluster ──────────────────────────┐
│                                                                          │
│   Cluster ──ref──▶ CloudStackCluster                                    │
│                      spec.failureDomains[]:                             │
│                        - zone.name        (= CloudStack Zone)           │
│                          zone.network.name (= Isolated/Shared Network)  │
│                          acsEndpoint.{name,namespace} (ref Secret)      │
│                                                                           │
│   Machine ──ref──▶ CloudStackMachine (qua CloudStackMachineTemplate)     │
│                      spec.offering.name  (= ServiceOffering CloudStack) │
│                      spec.template.name  (= VM Template/ISO)            │
│                      spec.networks[].name                               │
│                      spec.affinity       (host anti-affinity)           │
│                                                                           │
│   CAPC Controller (ns: capc-system)                                     │
│    - đọc Secret credential (api-url/api-key/secret-key)                  │
│    - gọi CloudStack API tạo VM, Isolated Network (nếu chưa có), AG       │
└──────────────────────────┬───────────────────────────────────────────────┘
                            ▼
              ┌──────────────────────────────┐
              │   Apache CloudStack API         │
              │   (Zone/Account/Domain/VPC)      │
              └───────────────┬───────────────────┘
                              ▼
              ┌──────────────────────────────┐
              │   KVM Host(s)  →  VM instance    │
              │   (control-plane / worker node,  │
              │    chạy Talos)                   │
              └──────────────────────────────┘
```

**Dependency**: CAPC cần CloudStack account có đủ quyền tạo VM/Network/AffinityGroup trong 1 hoặc nhiều Zone (xem mục 7 về least-privilege). CloudStack **Account/Domain** (đơn vị multi-tenancy sẵn có của CloudStack) không tự động map vào multi-tenancy ở tầng CAPI — đây là 2 lớp tách biệt, muốn liên kết (vd 1 tenant CloudStack Account = 1 nhóm Cluster) phải tự thiết kế ở tầng trên → [[Multi-Tenancy]].

### **5. How — Cơ chế hoạt động**

- **`CloudStackCluster`** — tương đương `InfrastructureCluster`, chứa `failureDomains[]` (mỗi failure domain = 1 CloudStack Zone + network + credential endpoint riêng, cho phép set up multi-zone).
- **`CloudStackMachine` / `CloudStackMachineTemplate`** — tương đương `InfrastructureMachine`, map `offering` (ServiceOffering = cấu hình CPU/RAM VM), `template` (VM Template/ISO — đây là nơi image Talos đã build qua Image Factory được đăng ký thành CloudStack Template → [[Image-Pipeline]]), `networks[]`, và `affinity` (anti-affinity group).
- **`CloudStackIsolatedNetwork`** — CRD riêng do CAPC tự quản lý khi dùng Isolated Network: tự tạo network nếu chưa tồn tại, có finalizer để tránh CAPI xoá nhầm trước khi cleanup xong ở CloudStack. Có thể gắn vào 1 **VPC** CloudStack sẵn có (`network.vpc.name`) nếu muốn nhiều Isolated Network chia sẻ 1 VPC.
- **Shared/Routed Network** — khác Isolated Network ở chỗ **cần kube-vip** để cấp VIP cho control-plane endpoint (CloudStack không tự cấp floating IP kiểu Isolated Network NAT) — dùng flavor `with-kube-vip` lúc `clusterctl generate cluster`. Chi tiết LB/endpoint → [[Networking]].
- **Credential** — 1 K8s Secret chứa `api-url`/`api-key`/`secret-key`/`verify-ssl`, base64-encode toàn bộ nội dung rồi gán vào biến môi trường `CLOUDSTACK_B64ENCODED_SECRET` lúc `clusterctl init`/`generate cluster`; `CloudStackCluster.spec.failureDomains[].acsEndpoint` reference tới Secret này theo `name`/`namespace`.

### **6. Key Config — Cấu hình cần nhớ**

- **Chỉ support Advanced Zone, không Security Groups** — đây là giới hạn cứng, không phải bug. Nếu CloudStack hiện tại dùng Basic Zone hoặc Security Groups, phải thiết kế lại zone trước khi dùng CAPC, không có workaround ở tầng CAPC.
- Biến môi trường kiểu `CLOUDSTACK_ZONE_NAME`, `CLOUDSTACK_NETWORK_NAME`, `CLOUDSTACK_CONTROL_PLANE_MACHINE_OFFERING` chỉ dùng lúc `clusterctl generate cluster` (template substitution) — không phải field runtime, dễ nhầm là config apply được sau khi cluster đã tạo.
- `CloudStackMachineTemplate` là **immutable** theo convention CAPI chung (sửa template không tự rollout Machine cũ) — đổi `offering`/`template` phải tạo `MachineDeployment` rollout mới, giống hành vi chuẩn CAPI, không phải đặc thù CAPC.
- Network Shared/Routed cần kube-vip chạy **thêm** như 1 static pod/addon trên control-plane — nếu quên cài, control-plane endpoint không có VIP, cluster tạo ra không truy cập được dù VM đã lên.
- API CRD đang promote version (`v1beta2`→`v1beta4`) — khi upgrade CAPC, **luôn đọc release note để biết field nào đổi tên/đổi cấu trúc**, đừng áp dụng lại YAML cũ không kiểm tra, khác hẳn mức ổn định mà CAPI core `v1beta1`/`v1beta2` đang giữ.

### **7. Security Considerations**

- **Attack surface chính** giống pattern chung của mọi infra provider (xem [[CAPI]] mục 7): Secret credential CloudStack (`api-key`/`secret-key`) nằm trong management cluster, namespace `capc-system`. Compromise namespace này = attacker có quyền gọi CloudStack API y hệt account đã cấp — nên **không** dùng account cloud-admin cho CAPC.
- CloudStack API key/secret theo thiết kế CloudStack gắn với 1 **Account** trong 1 **Domain** — nên tạo riêng 1 Account "service" cho CAPC, scope quyền chỉ trong Domain/Project cần, **không** dùng Account cấp Domain-admin/Root-admin trừ khi bắt buộc multi-zone cross-domain.
- ⚠️ **Cần verify**: CAPC có hỗ trợ CloudStack **Project** (đơn vị multi-tenancy mịn hơn Account) hay chỉ Account/Domain — chưa tra được chi tiết, quan trọng nếu muốn map "1 tenant KaaS = 1 Project CloudStack".
- Bootstrap data (machine config Talos) được CAPC truyền vào CloudStack dưới dạng **user-data** lúc tạo VM — user-data CloudStack mặc định **không mã hoá at-rest** theo hiểu biết chung về CloudStack (⚠️ cần verify lại trực tiếp với version CloudStack đang dùng), nên coi machine config chứa secret/cert là nhạy cảm tương đương credential, hạn chế ai đọc được DescribeInstances/user-data qua CloudStack UI/API.
- **Hardening tối thiểu**: (1) Account CAPC dùng riêng, least-privilege theo Domain/Zone cần; (2) rotate `api-key`/`secret-key` định kỳ qua CloudStack UI, update lại Secret trong management cluster đồng bộ; (3) hạn chế ai đọc được namespace `capc-system` (Secret + CloudStackMachine chứa thông tin network/IP nội bộ); (4) nếu CloudStack hỗ trợ, bật audit log phía CloudStack cho account CAPC để trace lại hành động tạo/xoá VM.

### **8. Ops Runbook — Production Notes**

- **Health check**: `kubectl get cloudstackcluster,cloudstackmachine -A`; `clusterctl describe cluster <name>` (xem [[CAPI]]) để thấy `CloudStackMachine` có `Ready` chưa — nếu stuck, thường do CloudStack API trả lỗi (quota Account hết, Network/Offering không tồn tại đúng tên).
- **Log quan trọng**: controller log namespace `capc-system` — lỗi phổ biến nhất là gọi CloudStack API fail do credential sai/expired hoặc tên Zone/Network/Offering không khớp (CAPC match theo **tên**, không phải ID, nên gõ sai tên = lỗi "not found" dễ gây nhầm là bug).
- **Metric**: ⚠️ **cần verify** — chưa xác nhận CAPC có expose metric riêng ngoài default controller-runtime `/metrics` hay không.
- **Troubleshooting network**: khi `CloudStackIsolatedNetwork` stuck, thường do finalizer chờ cleanup ở CloudStack chưa xong (vd còn VM đang dùng network) — không nên xoá finalizer bằng tay trừ khi chắc chắn đã dọn sạch ở CloudStack, tránh orphan resource.
- **CSI/Storage của volume VM** — không thuộc phạm vi CAPC (CAPC chỉ tạo VM, không quản lý volume data-plane của workload) → [[Storage]].

### **9. Gotchas & Lessons Learned**

- CAPC match resource CloudStack (Zone/Network/Offering/Template) theo **tên**, không phải UUID — đổi tên resource phía CloudStack sau khi cluster đã chạy có thể gây lệch reference âm thầm, nên tránh rename resource đang được CAPC tham chiếu.
- "Chỉ support Advanced Zone không Security Groups" là giới hạn dễ bị bỏ sót khi mới research CAPC — nhiều hướng dẫn CloudStack cơ bản trên mạng dùng Security Groups (phổ biến với KVM không có hạ tầng VLAN) nên cần xác nhận lại mô hình zone hiện có **trước khi** đầu tư thời gian setup CAPC.
- API provider (`v1beta2→v1beta4`) đổi nhanh hơn CAPI core — khác với Talos (CABPT/CACPPT) vốn theo kịp khá sát, CAPC có vẻ đang chạy đua bắt kịp CAPI `v1beta2` contract, nên pin version rõ và test kỹ mỗi lần upgrade, đừng auto-upgrade theo "latest".
- (Mục này để tự bổ sung tiếp sau khi thao tác trực tiếp với CloudStack thật — đặc biệt phần multi-zone và Project/Account mapping, lesson learned thực chiến quan trọng hơn note lý thuyết)

### **10. Resources**

- Official book: https://cluster-api-cloudstack.sigs.k8s.io/
- GitHub repo + releases: https://github.com/kubernetes-sigs/cluster-api-provider-cloudstack
- Configuration guide (nguồn field YAML cụ thể): https://github.com/kubernetes-sigs/cluster-api-provider-cloudstack/blob/main/docs/book/src/clustercloudstack/configuration.md
- CAPI v1beta2/v1beta4 migration PR: https://github.com/kubernetes-sigs/cluster-api-provider-cloudstack/pull/493
- Apache CloudStack docs (phía IaaS): https://cloudstack.apache.org/
- **Lưu ý khi dùng note này**: version (v0.6.1), danh sách maintainer, và tình trạng "pre-1.0 API đang đổi nhanh" lấy qua search engine + đọc trực tiếp configuration.md/PROJECT trên GitHub — **chưa** đọc trực tiếp CHANGELOG/release note đầy đủ của từng bản, và chưa verify CloudStack Project support. Đối chiếu lại tại https://github.com/kubernetes-sigs/cluster-api-provider-cloudstack/releases trước khi dùng cho quyết định production.
