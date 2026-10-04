# Multi-Tenancy — Isolation Model & Self-Service
Tags: #k8s-provider #multi-tenancy #clusterclass #kamaji #vcluster #quota
Related: [[CAPI]], [[capi--clusterclass-topology]], [[CAPC]], [[CAPO]], [[Fleet-Ops]], [[Provider-Security]]
Last updated: 2026-10-03

> ⚠️ **Version note — CẦN VERIFY LẠI**: `cluster-api-control-plane-provider-kamaji` được cập nhật gần nhất khoảng 28/09/2026 theo search engine (dự án `clastix/kamaji` vẫn active) — **chưa verify trực tiếp** compatibility matrix giữa Kamaji provider ↔ CAPI v1.14.x. vCluster "Private Nodes" (tier isolation mạnh nhất) là feature tương đối mới trong dòng sản phẩm — **cần verify** mức độ production-ready thật (so với Shared Nodes/Dedicated Nodes) tại thời điểm triển khai, không chỉ dựa vào trang marketing.

---

### **1. What — Nó là cái gì?**

Đây là nhóm quyết định **mô hình cấp 1 "cluster" cho 1 khách hàng** trong hệ thống KaaS — không phải 1 công cụ đơn lẻ, mà là lựa chọn giữa 2 hướng kiến trúc khác hẳn nhau: **(a) dedicated cluster** — mỗi tenant có 1 `Cluster` CAPI hoàn chỉnh, control-plane là VM thật (qua CACPPT Talos) chạy trên CloudStack/OpenStack; hoặc **(b) hosted/virtual control-plane** — control-plane của tenant chỉ là Pod (Kamaji) hoặc API server giả lập trong namespace (vCluster), chạy chung trên 1 "host cluster" lớn, chỉ worker node (nếu có) mới là VM riêng. Lựa chọn này quyết định toàn bộ bài toán cost, isolation, và vận hành phía sau — mọi group khác ([[CAPI]], [[CAPC]], [[CAPO]], [[Fleet-Ops]]) đều bị ảnh hưởng theo lựa chọn ở đây.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Nếu không nghĩ kỹ mô hình này trước, một KaaS provider build trên CAPI sẽ mặc định rơi vào **dedicated cluster cho mọi tenant** (vì đó là thứ CAPI "quick start" dạy), và gặp đúng vấn đề mà managed-K8s công khai từng gặp: mỗi cluster nhỏ (dev/test, hoặc tenant ít traffic) vẫn phải trả giá **≥3 VM control-plane** (HA etcd) dù chỉ chạy vài Pod — chi phí hạ tầng (VM, IP, LB) tăng tuyến tính theo số tenant, không theo tải thật. Ở phía ngược lại, nếu dùng chung 1 cluster K8s (chỉ namespace-level isolation) thì không đủ cứng cho khách hàng **bên ngoài** trả tiền (1 CVE K8s core hay 1 Pod escape ảnh hưởng toàn bộ khách).

Có 1 lớp giải pháp trung gian — **hosted control-plane** (Kamaji) hoặc **virtual cluster** (vCluster) — tách API server của tenant ra khỏi VM vật lý, chạy như Pod/process trong 1 cluster quản trị dùng chung, giữ được "mỗi tenant có 1 cluster object riêng, API riêng" mà không phải trả giá 3 VM/tenant. Đây chính là cách các managed-K8s thương mại lớn (kiểu "serverless control-plane") giảm chi phí vận hành hàng ngàn cluster nhỏ — và **Kamaji ship sẵn 1 CAPI Control Plane Provider chính thức**, nghĩa là có thể dùng ngay trong object model `Cluster` quen thuộc, không cần hệ thống quản lý riêng bên ngoài CAPI.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng dedicated cluster (CACPPT Talos, VM thật) khi:** khách hàng cần cluster lớn/traffic cao, cần SLA/isolation mạnh nhất (kể cả ở tầng kernel/hypervisor, không chỉ K8s API), hoặc khách yêu cầu hợp đồng "control-plane riêng vật lý" (compliance).

**Dùng hosted control-plane (Kamaji) khi:** số lượng tenant lớn, mỗi tenant nhỏ/vừa (dev/test, staging, hoặc production traffic thấp-trung) — tiết kiệm được nhiều nhất ở chính control-plane (thứ tốn 3 VM/tenant dù không có traffic); vẫn giữ được self-service qua CAPI `Cluster` object vì có provider chính thức.

**Dùng vCluster khi:** cần tenant tự quản (CRD riêng, RBAC riêng, thậm chí version K8s API riêng) nhưng chấp nhận **chia sẻ node vật lý** ở tier rẻ nhất (Shared Nodes) — hoặc đã lên tier "Private Nodes" nếu cần isolation mạnh hơn (network/storage riêng, không chia sẻ hardware).

**KHÔNG dùng hosted/virtual control-plane khi:**
- Khách hàng cần kiểm soát hoàn toàn control-plane (vd tự chọn version etcd, tự audit log ở tầng VM) — Kamaji/vCluster che giấu tầng này.
- Compliance yêu cầu "không share kernel/hypervisor" với tenant khác ở **bất kỳ layer nào** — dedicated cluster (hoặc ít nhất dedicated node) là lựa chọn an toàn hơn để chứng minh với auditor.
- Chưa có đủ năng lực vận hành 1 "host cluster" lớn chịu trách nhiệm cho **hàng trăm** control-plane Pod cùng lúc — single point of failure mới xuất hiện ở tầng host cluster, cần HA tốt hơn bất kỳ tenant cluster đơn lẻ (xem mục 7).

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
MÔ HÌNH A — Dedicated Cluster (CACPPT Talos, hiện tại đang dùng)
┌─────────────── Management Cluster (CAPI) ───────────────┐
│   Cluster "tenant-A"         Cluster "tenant-B"          │
│        │                            │                     │
│        ▼                            ▼                     │
│  ControlPlane = 3 VM Talos   ControlPlane = 3 VM Talos    │
│  (CACPPT, HA etcd riêng)     (CACPPT, HA etcd riêng)      │
│        │                            │                     │
│  Worker = N VM Talos         Worker = N VM Talos          │
└───────────────────────────────────────────────────────────┘
→ Isolation mạnh nhất (tới tầng VM/kernel), nhưng CHI PHÍ CỐ ĐỊNH
  tối thiểu 3 VM/tenant chỉ cho control-plane, bất kể traffic.

MÔ HÌNH B — Hosted Control Plane (Kamaji CAPI provider)
┌──────────────── Management/Host Cluster (CAPI + Kamaji) ─────────────┐
│  Cluster "tenant-A"                    Cluster "tenant-B"             │
│        │                                      │                       │
│        ▼                                      ▼                       │
│  ControlPlane = Pod (kube-apiserver,    ControlPlane = Pod (...)     │
│  controller-mgr, scheduler) trong        trong namespace "tenant-B"  │
│  namespace "tenant-A" — KHÔNG phải VM                                 │
│        │                                      │                       │
│  Worker = N VM Talos (CAPC/CAPO,        Worker = N VM Talos           │
│  vẫn là VM thật, riêng theo tenant)                                   │
└─────────────────────────────────────────────────────────────────────┘
→ Control-plane rẻ (chỉ Pod, share CPU/RAM host cluster theo request/
  limit), density cao (hàng trăm-ngàn tenant/host cluster theo tài
  liệu Kamaji) — nhưng etcd/API server của MỌI tenant giờ phụ thuộc
  vào uptime + bảo mật của 1 host cluster duy nhất.

MÔ HÌNH C — vCluster (namespace-based virtual cluster)
┌──────────── Host Cluster (bất kỳ, không cần CAPI) ──────────┐
│  Namespace "tenant-A"              Namespace "tenant-B"      │
│   vCluster API server (Pod)         vCluster API server (Pod)│
│   + "syncer" đồng bộ xuống Pod       + syncer                │
│   thật ở host cluster                                        │
│   [Shared Nodes | Dedicated Nodes | Private Nodes — 3 tier   │
│    isolation tăng dần, xem mục 3]                             │
└───────────────────────────────────────────────────────────────┘
→ Không đi qua CAPI object model — phù hợp tier rẻ nhất/dev-test
  hơn là "bán" như 1 sản phẩm managed K8s đầy đủ.
```

**Dependency quan trọng**: dù chọn mô hình nào, **quota ở tầng IaaS** (CloudStack Account/Domain limits hoặc OpenStack Nova/Neutron project quota) phải được map 1-1 với gói dịch vụ bán cho tenant — xem mục 6. Và dù hosted-control-plane giảm VM control-plane, **worker node vẫn là VM thật** qua [[CAPC]]/[[CAPO]] — tiết kiệm chỉ ở 1 nửa bài toán, không phải toàn bộ.

### **5. How — Cơ chế hoạt động**

- **Kamaji** biến 1 cluster K8s có sẵn thành "Management Cluster", chạy `kube-apiserver`/`kube-controller-manager`/`kube-scheduler` của **mỗi tenant** như Pod riêng (etcd có thể dùng chung 1 cluster etcd đa-tenant qua nhiều "datastore" logic, hoặc etcd riêng tuỳ cấu hình) — mô hình "hard multi-tenant", mỗi tenant là admin cluster của chính họ, không share gì với tenant khác ở tầng API/data.
- **Kamaji CAPI Control Plane Provider** (`cluster-api-control-plane-provider-kamaji`) cho phép `Cluster` object trong CAPI dùng Kamaji làm `controlPlaneRef` thay vì `TalosControlPlane` — nghĩa là có thể **kết hợp**: control-plane qua Kamaji (Pod, rẻ) + worker qua CAPC/CAPO + Talos bootstrap (VM thật, như hiện tại) — không cần đổi toàn bộ stack, chỉ đổi control-plane provider trong `ClusterClass`.
- **vCluster** hoạt động hoàn toàn khác tầng: không cần CAPI, chạy 1 API server "ảo" trong Pod + "syncer" đồng bộ object (Pod, Service...) từ virtual cluster xuống namespace thật ở host cluster. 3 tier: **Shared Nodes** (rẻ nhất, chỉ tách RBAC/CRD/API, Pod vẫn chạy chung node vật lý với tenant khác), **Dedicated Nodes** (node vật lý riêng theo tenant, vẫn chung network/storage layer), **Private Nodes** (riêng cả network + storage, gần tương đương dedicated cluster về isolation nhưng vẫn 1 control-plane ảo).
- **ClusterClass cho nhiều tier** — map trực tiếp 3 mô hình trên vào "flavor" bán cho khách: `ClusterClass` dùng Kamaji control-plane cho tier "starter/dev", `ClusterClass` dùng Talos CACPPT (dedicated) cho tier "enterprise" — khách chỉ thấy khác tên flavor, cơ chế implement phía sau khác hẳn. Chi tiết patch/variable → [[capi--clusterclass-topology]].
- **Quota mapping** — mỗi gói dịch vụ (tier) cần map sang limit cụ thể ở CloudStack (Account/Domain resource limit: CPU, RAM, Primary/Secondary storage, network rate) hoặc OpenStack (Nova quota: instances/cores/RAM; Neutron quota: floating IP, network, port) — việc này nằm **ngoài** CAPI, phải tự động hoá riêng (gọi API CloudStack/OpenStack set quota khi tạo tenant) nếu muốn self-service thật, CAPI không biết gì về "quota tiền" của khách.

### **6. Key Config — Cấu hình cần nhớ**

- **CloudStack limit**: áp theo Account hoặc Domain, cho CPU/Memory/Primary storage/Secondary storage/network rate (mbps). Nếu 1 operation chạm nhiều loại limit cùng lúc (vd Instance limit **và** CPU limit), CloudStack áp theo **limit nào chặt hơn** — dễ gây lỗi "deploy fail" mà log chỉ nói chung là "resource limit exceeded", phải tự kiểm tra từng loại limit riêng để biết tenant đang chạm giới hạn nào.
- **OpenStack default quota khá thấp cho production**: Nova mặc định 10 instance/10 floating IP/20 core per project; Neutron mặc định 50 floating IP per project (giá trị âm = unlimited). Nếu không chỉnh lại theo tier bán cho khách, tenant tier "enterprise" vẫn bị chặn bởi default quota như tier free.
- **Kamaji etcd datastore**: tuỳ chọn share 1 etcd cluster cho nhiều tenant (datastore logic riêng mỗi tenant) hay tách etcd vật lý riêng — ảnh hưởng trực tiếp tới "hard multi-tenant" thật tới mức nào; **cần verify** cấu hình datastore cụ thể (shared vs dedicated etcd) trước khi quảng cáo "tenant hoàn toàn cách ly" cho khách dùng chung datastore.
- Trộn `ClusterClass`: nếu 2 tier (Kamaji control-plane vs Talos CACPPT control-plane) dùng chung 1 `ClusterClass` qua biến điều kiện (`enabledIf`), patch sai điều kiện rất dễ khiến tenant tier rẻ vô tình được tạo control-plane VM tốn kém — nên tách `ClusterClass` riêng theo tier thay vì dồn hết logic vào 1 class để tránh nhầm.

### **7. Security Considerations**

- **Kamaji/hosted control-plane đổi hoàn toàn bản chất blast radius**: với dedicated cluster, compromise 1 tenant **không** tự động ảnh hưởng tenant khác (VM riêng, kernel riêng). Với Kamaji, API server của **mọi** tenant chạy trên cùng node vật lý của host cluster — 1 lỗ hổng container escape hoặc 1 misconfiguration ở host cluster (kernel CVE, containerd CVE) có thể ảnh hưởng **đồng thời nhiều tenant cùng lúc**, khác hẳn mô hình "1 VM lỗi chỉ ảnh hưởng 1 tenant". Đây là trade-off cost-vs-blast-radius phải nói rõ với khách nếu bán tier dùng Kamaji.
- vCluster **Shared Nodes** tier isolation yếu nhất: tenant khác nhau chạy Pod chung node — nếu không áp thêm NetworkPolicy/PodSecurity/resource quota chặt ở host cluster, 1 tenant vẫn có thể gây noisy-neighbor hoặc (tuỳ CVE container runtime) vượt ranh namespace. Không nên dùng tier này để bán cho khách ngoài trả tiền mà không có thêm lớp kiểm soát.
- Credential CAPC/CAPO (tạo worker VM) và credential/RBAC của Kamaji host cluster (tạo Pod control-plane) đều hội tụ về **cùng 1 management cluster** → attack surface management cluster mô tả ở [[CAPI]] mục 7 không giảm đi khi dùng hosted control-plane, mà **tăng thêm** 1 lớp (namespace isolation giữa tenant trong chính management/host cluster phải đúng, không chỉ RBAC namespace K8s thông thường mà còn network policy giữa các Pod control-plane của các tenant).
- **Hardening tối thiểu khi dùng Kamaji/vCluster**: (1) NetworkPolicy mặc định deny giữa namespace tenant trên host cluster; (2) resource quota + LimitRange bắt buộc theo namespace tenant, tránh 1 tenant chiếm hết CPU/RAM host cluster; (3) tách etcd/datastore theo tenant nếu compliance yêu cầu, đừng mặc định share; (4) audit riêng ai có quyền `kubectl exec`/debug vào Pod control-plane của Kamaji — đây tương đương quyền root trên "VM ảo" của tenant đó.

### **8. Ops Runbook — Production Notes**

- **Health check theo mô hình**: dedicated cluster check qua `kubectl get clusters -A` như [[CAPI]] mục 8; Kamaji cần thêm check riêng tình trạng Pod control-plane (`kubectl get pods -n <tenant-ns>` trên host cluster) — 1 Pod `kube-apiserver` crash loop ảnh hưởng y như control-plane VM chết, nhưng triệu chứng/log nằm ở chỗ khác (log Pod, không phải `talosctl logs`).
- **Capacity planning host cluster (Kamaji)**: vì density cao (nhiều tenant/node), phải theo dõi sát CPU/RAM request thực tế của toàn bộ Pod control-plane cộng lại — khác hẳn dedicated cluster (mỗi tenant tự lo capacity VM riêng của họ).
- **Quota drift**: nên có job định kỳ so khớp quota đã set ở CloudStack/OpenStack với gói dịch vụ tenant đang trả tiền (nâng/hạ tier mà quên update quota IaaS là lỗi vận hành phổ biến, không phải lỗi CAPI).
- Backup/DR cho Kamaji: etcd của host cluster giờ chứa **control-plane data của mọi tenant** gộp lại (nếu share datastore) — mất host cluster ảnh hưởng nặng hơn mất 1 management cluster CAPI thường, cần chiến lược backup riêng chặt hơn → [[Fleet-Ops]].

### **9. Gotchas & Lessons Learned**

- Đừng nghĩ "Kamaji = rẻ hơn nên luôn tốt hơn" — rẻ hơn ở control-plane, **không** rẻ hơn ở worker node (vẫn VM CAPC/CAPO như cũ), và đổi lại tăng blast radius nếu host cluster có sự cố. Quyết định đúng là theo tier khách hàng, không phải "chuyển hết sang Kamaji cho tiết kiệm".
- vCluster không đi qua CAPI object model — nếu đã đầu tư toàn bộ automation quanh `Cluster`/`ClusterClass`, thêm vCluster nghĩa là thêm 1 hệ thống quản lý **song song**, không tận dụng lại được tooling CAPI đã xây (GitOps, `clusterctl`...).
- OpenStack default quota rất thấp (10 instance/project) — nếu test lab thấy "tạo VM thứ 11 fail" đừng tưởng do CAPO lỗi, 90% là do quota chưa chỉnh.
- (Mục này để tự bổ sung tiếp sau khi thử nghiệm Kamaji provider thật trên management cluster — lesson learned thực chiến quan trọng hơn note lý thuyết)

### **10. Resources**

- Kamaji (hosted control plane manager): https://github.com/clastix/kamaji
- Kamaji + Cluster API: https://kamaji.clastix.io/cluster-api/
- vCluster — so sánh tenancy model: https://www.vcluster.com/guides/tenancy-models-with-vcluster
- vCluster vs Kamaji (control-plane scaling): https://www.vcluster.com/blog/kamaji-vs-vcluster-kubernetes-scaling
- CloudStack resource limits: https://cwiki.apache.org/confluence/display/CLOUDSTACK/Limit+Resources+to+domains+and+accounts
- OpenStack Nova quota: https://docs.openstack.org/nova/latest/admin/quotas.html
- OpenStack Neutron quota: https://docs.openstack.org/neutron/latest/admin/ops-quotas.html
- **Lưu ý khi dùng note này**: compatibility matrix Kamaji-provider ↔ CAPI version, và mức production-ready thật của vCluster Private Nodes, lấy qua search engine/trang sản phẩm — cần tự verify qua repo GitHub + test thật trước khi chọn cho production.
