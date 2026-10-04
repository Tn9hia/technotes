# Cluster API (CAPI) — v1.14.x
Tags: #k8s-provider #capi #cluster-lifecycle #cluster-api
Related: [[CAPC]], [[CAPO]], [[Talos]], [[Management-Plane]], [[Fleet-Ops]]
Last updated: 2026-10-03

> ⚠️ **Version note — CẦN VERIFY LẠI**: Tại thời điểm viết, bản mới nhất tìm được qua search là **v1.14.2** (v1.14.0 ra khoảng cuối tháng 7/2026, v1.14.2 khoảng đầu tháng 9/2026). Số liệu này lấy từ search engine + PR renovate-bot bên thứ 3, **không phải đọc trực tiếp từ CHANGELOG/Releases chính thức**. Trước khi dùng số version này để quyết định gì, chạy `clusterctl version` trên hệ thống thật hoặc xem trực tiếp https://github.com/kubernetes-sigs/cluster-api/releases.
>
> Fact quan trọng đã verify được nguồn tương đối rõ: **API `v1beta1` (core CAPI, CABPK, KCP) đang trên lộ trình unserved ở CAPI v1.16** — tức object cũ dùng `apiVersion: cluster.x-k8s.io/v1beta1` sẽ không còn được apiserver của CAPI serve nữa. Timeline cụ thể (có nguồn blog bên thứ 3 ghi "trước 04/2027") **cần verify lại ở trang chính thức** https://cluster-api.sigs.k8s.io/reference/versions trước khi lập kế hoạch migrate thật.

---

### **1. What — Nó là cái gì?**

Cluster API (CAPI) là project của Kubernetes SIG Cluster Lifecycle, cung cấp declarative API (CRD + controller) để tạo/scale/upgrade/xoá **Kubernetes cluster** — giống cách `Deployment` quản lý `Pod`, nhưng ở tầng quản lý *cluster* thay vì quản lý *workload*. CAPI core tự nó **không biết tạo VM hay cài K8s** — nó chỉ định nghĩa object model chung (`Cluster`, `Machine`...) và giao việc thực thi cho các provider pluggable (infrastructure, bootstrap, control-plane). Trong stack k8s-provider của mình, CAPI đóng vai trò "lõi điều phối" — mọi group khác (CAPC/CAPO, Talos, Fleet Ops) đều cắm vào đây.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Nếu không có CAPI, để tự động hoá vòng đời cluster phải chọn 1 trong các hướng đều tệ hơn:
- **Script tay (kubeadm + Ansible/Terraform)** — chạy được, nhưng không có reconciliation loop: node chết không tự được phát hiện/thay thế, cluster drift khỏi desired state không ai tự sửa.
- **Terraform/Pulumi thuần** — tạo VM tốt, nhưng không có khái niệm "K8s cluster" ở tầng API: không có `MachineHealthCheck` tự remediate, rolling-upgrade K8s version phải tự điều phối ngoài K8s API, khó áp RBAC của K8s để tách quyền theo tenant.
- **Managed K8s (EKS/GKE/AKS)** — không áp dụng được vì đây là KaaS tự build trên private cloud (KVM + CloudStack/OpenStack), không phải public cloud.

CAPI lấp khoảng trống: "tạo cluster" trở thành `kubectl apply` một `Cluster` + machine template vào **management cluster**, controller tự reconcile — tạo VM thiếu, join node mới, rolling-replace node unhealthy/outdated. Đây chính là nền tảng để biến "cấp 1 cluster K8s" thành 1 service tự động hoá được, thay vì 1 tác vụ vận hành thủ công mỗi lần có khách mới.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng khi:** cần tự động hoá vòng đời **nhiều** K8s cluster trên hạ tầng tự quản lý (private cloud); cần self-service cluster cho nhiều tenant; cần chuẩn hoá "flavor" cluster (size/version/add-on) qua `ClusterClass` để tạo hàng loạt cluster giống nhau.

**KHÔNG dùng khi:**
- Chỉ cần vận hành 1 cluster duy nhất, ít thay đổi — overhead chạy thêm 1 management cluster (CAPI cần nơi để chạy controller) không đáng, `kubeadm` thủ công hoặc kOps đơn giản hơn nhiều.
- Cần quản lý resource **trong** cluster (Pod/Deployment/Service) — đó là phạm vi K8s thông thường, CAPI chỉ quản lý hạ tầng + control-plane của cluster, không đụng tới app bên trong.
- Infra provider cho IaaS của mình (CAPC cho CloudStack, CAPO cho OpenStack) chưa đủ mature/được maintain tốt tại thời điểm triển khai thật — đây là project community-driven, mức độ active khác nhau theo thời gian, **cần verify tình trạng maintain hiện tại** trước khi cam kết production (xem thêm [[CAPC]] / [[CAPO]]).

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
┌───────────────────────────── Management Cluster ─────────────────────────────┐
│                                                                                 │
│  kubectl/clusterctl apply (Cluster + ClusterClass + MachineDeployment)         │
│                        │                                                      │
│                        ▼                                                      │
│   ┌───────────────────────────────┐                                           │
│   │   CAPI Core Controllers        │  reconcile Cluster / Machine /           │
│   │   (ns: capi-system)            │  MachineSet / MachineDeployment /        │
│   │                                 │  MachineHealthCheck / ClusterClass       │
│   └───────────────┬─────────────────┘  + Topology controller                   │
│                    │ gọi provider qua "contract" (CRD + status convention)     │
│      ┌─────────────┼───────────────────────┬────────────────────┐             │
│      ▼             ▼                       ▼                    ▼             │
│ ┌───────────┐ ┌────────────────┐   ┌──────────────────┐  ┌───────────────┐    │
│ │ Bootstrap  │ │ Control Plane   │   │ Infrastructure    │  │ Addon (tuỳ    │    │
│ │ Provider   │ │ Provider        │   │ Provider          │  │ chọn — CNI/   │    │
│ │ CABPT      │ │ CACPPT          │   │ CAPC / CAPO        │  │ CSI qua       │    │
│ │ (Talos)    │ │ (Talos)         │   │                    │  │ ClusterResour-│    │
│ └─────┬──────┘ └────────┬────────┘   └─────────┬──────────┘  │ ceSet)        │    │
│       │ sinh            │ quản lý control-plane  │ gọi API IaaS thật          │    │
│       │ machine config  │ lifecycle              │ (tạo VM/network/LB)       │    │
└───────┼─────────────────┼───────────────────────┼────────────────────────────┘
        │                 │                       │
        └─────────────────┴───────────┬───────────┘
                                       ▼
                     ┌───────────────────────────────┐
                     │   CloudStack / OpenStack (KVM)   │
                     │   → VM instance, network, LB,    │
                     │     template/image                │
                     └───────────────┬────────────────┘
                                      ▼
                     ┌───────────────────────────────┐
                     │      Tenant / Workload Cluster    │
                     │  (control-plane + worker node     │
                     │   chạy Talos + Kubernetes)          │
                     └───────────────────────────────┘
```

**Dependency quan trọng**: management cluster phải tồn tại **trước** (1 K8s cluster "mồi" — kind/minikube cho dev, hoặc 1 cluster HA riêng cho production, xem [[Management-Plane]]). Mất management cluster + etcd của nó ≈ mất khả năng *quản lý* (scale/upgrade/xoá) toàn bộ fleet — nhưng tenant cluster đang chạy **không chết theo** vì đã independent sau khi provisioning xong. Backup etcd management cluster là việc critical nhất ở tầng này → [[Fleet-Ops]].

### **5. How — Cơ chế hoạt động**

- **`Cluster`** — object đại diện 1 workload cluster, reference tới `InfrastructureCluster` (object specific theo CAPC/CAPO) + `ControlPlane` object.
- **`Machine`** — đại diện 1 node (control-plane hoặc worker), reference tới `InfrastructureMachine` (VM thật trên IaaS) + bootstrap data (machine config Talos). Xoá `Machine` = xoá VM + drain node, không phải chỉ xoá K8s object.
- **`MachineSet` / `MachineDeployment`** — tương tự `ReplicaSet`/`Deployment` của K8s nhưng cho `Machine`. `MachineDeployment` cho phép rolling upgrade: đổi K8s version hoặc Talos image → tạo `Machine` mới, drain & xoá `Machine` cũ theo `maxSurge`/`maxUnavailable`.
- **`MachineHealthCheck`** — giám sát Node condition, tự trigger remediation (xoá & tạo lại `Machine`) khi node unhealthy quá threshold — nền cho self-healing fleet, chi tiết vận hành thực tế → [[Fleet-Ops]].
- **`ClusterClass` + Cluster Topology** — template hoá 1 "flavor" cluster (số control-plane, provider nào, variables gì) để tạo nhiều `Cluster` giống nhau chỉ bằng vài field — đây chính là cơ chế để "bán" nhiều loại cluster (small/medium/large, có/không HA) cho nhiều khách mà không viết lại YAML từ đầu mỗi lần. Chi tiết variables/patches/rollout → [[capi--clusterclass-topology]].
- **Provider contract** — mỗi provider (bootstrap/control-plane/infrastructure/addon/IPAM) là 1 bộ CRD + controller riêng, cài qua `clusterctl init`, giao tiếp với core qua convention (status field như `ready`/`initialized`, `ownerReferences`) — **không** qua gRPC/API call trực tiếp. Hệ quả: version skew giữa core và từng provider rất quan trọng, không có gì tự enforce compatibility ngoài metadata provider tự khai.
- **`clusterctl`** — CLI quản lý lifecycle CAPI/provider trên management cluster (`init`, `generate cluster`, `move`, `upgrade`). `clusterctl move` dùng để di chuyển toàn bộ object CAPI từ management cluster A sang B (ví dụ pivot từ bootstrap cluster tạm sang management cluster HA thật) — chi tiết lệnh/gotcha → [[capi--clusterctl]].

### **6. Key Config — Cấu hình cần nhớ**

- `clusterctl init --bootstrap talos --control-plane talos --infrastructure <capc|capo>` — quên `--infrastructure` hoặc chọn version không tương thích là lỗi phổ biến nhất lúc mới cài.
- **Version skew 3 chiều**: core CAPI ↔ bootstrap/control-plane provider Talos (CABPT/CACPPT, do Sidero Labs maintain **riêng**, không cùng release cycle với CAPI core) ↔ infra provider (CAPC/CAPO). Không có gì đảm bảo 3 bên release đồng bộ — áp sai version → controller crash loop hoặc webhook reject im lặng. **Cần verify compatibility matrix thật** (`metadata.yaml` của từng provider) tại thời điểm lab, đừng giả định "bản mới nhất của tất cả luôn tương thích nhau".
- `v1beta1` → `v1beta2`: object cũ sẽ **unserved ở CAPI v1.16** (ngày cụ thể cần verify, xem banner đầu file). Nếu Helm chart/GitOps/script khác đang hardcode `apiVersion: cluster.x-k8s.io/v1beta1`, sẽ fail silently sau khi management cluster upgrade CAPI — phải migrate field trước.
- `ClusterClass.variables` + `patches` (JSONPatch) — nơi dễ misconfig nhất: patch sai target path hoặc variable thiếu default → `Cluster` tạo từ class lỗi ở tầng topology controller, message lỗi khó đọc hơn lỗi YAML thông thường.
- RBAC của management cluster: ServiceAccount của provider controller thường có quyền đọc/ghi rộng trong toàn management cluster (kể cả Secret chứa kubeconfig + credential IaaS) — xem mục 7.

### **7. Security Considerations**

- **Attack surface chính**: management cluster là **root of trust của toàn bộ fleet**. Compromise management cluster (hoặc chỉ cần namespace của 1 provider controller) = attacker đọc được Secret chứa kubeconfig admin của **mọi** tenant cluster + credential CloudStack/OpenStack (dùng để tạo VM) → leo thang ra toàn bộ hạ tầng IaaS, không chỉ riêng 1 cluster. Đây là khác biệt lớn nhất so với vận hành 1 cluster đơn: blast radius ở đây là cả provider, không phải 1 tenant.
- Credential IaaS (CloudStack API key/secret, OpenStack `clouds.yaml`) được CAPC/CAPO lưu dưới dạng K8s Secret trong management cluster. Theo security guideline chính thức của CAPI (`cluster-api.sigs.k8s.io/developer/providers/security-guidelines`): credential này phải **least-privileged** (không dùng account cloud-admin cho CAPC/CAPO) và namespace chứa nó phải hạn chế quyền đọc chỉ cho cloud-admin thật.
- Bootstrap data (machine config Talos — chứa secret/cert để node join cluster) đi qua `BootstrapConfig`, được infra provider đóng gói để đưa vào VM lúc tạo (thường qua user-data/cloud-init). **Cần verify** CAPC/CAPO cụ thể có hỗ trợ encrypt-at-rest cho bootstrap data hay chỉ base64-encode (base64 không phải mã hoá, chỉ là encoding).
- **Hardening tối thiểu**: (1) management cluster tách biệt hoàn toàn, không chạy chung workload nào khác; (2) RBAC tối thiểu, tách namespace/tenant nếu cho self-service tạo `Cluster` trực tiếp; (3) rotate credential IaaS định kỳ theo least-privilege; (4) backup etcd management cluster riêng + mã hoá at-rest (K8s secret encryption provider), vì nó chứa toàn bộ credential IaaS + kubeconfig của cả fleet.

### **8. Ops Runbook — Production Notes**

- **Health check**: `kubectl get clusters -A` (cột `PHASE` phải là `Provisioned`); `clusterctl describe cluster <name> -n <ns>` — cho view dạng cây Cluster → Machine → conditions, cách nhanh nhất để thấy resource nào đang stuck.
- **Log quan trọng**: controller log trong namespace `capi-system`, `capi-bootstrap-system`, `capi-controlplane-system`, và namespace riêng của infra provider (`capc-system`/`capo-system`). Khi `Cluster` stuck ở `Provisioning`, thường do infra provider controller log lỗi gọi API IaaS (quota hết, credential sai/expired).
- **Metric cần alert**: chưa có metric chuẩn hoá chung toàn bộ provider (mỗi provider expose riêng qua controller-runtime default `/metrics`) — **cần verify** metric cụ thể của CAPC/CAPO/CABPT khi setup thật. Nhìn chung nên alert theo `controller_runtime_reconcile_errors_total` tăng bất thường.
- **Backup/restore**: backup toàn bộ custom resource CAPI (`Cluster`, `Machine`, `MachineDeployment`, Secret kubeconfig...) bằng Velero hoặc `clusterctl move --to-directory` (xuất ra file, không cần cluster đích sẵn). Mất management cluster không xoá tenant cluster đang chạy, nhưng mất khả năng *quản lý* tới khi restore được state.
- `clusterctl move` dùng để pivot management cluster (ví dụ từ bootstrap cluster tạm trên laptop sang management cluster HA thật) — cần pause reconciliation đúng thứ tự trước khi move để tránh 2 management cluster cùng reconcile 1 lúc.

### **9. Gotchas & Lessons Learned**

- v1beta1 unserved ở v1.16 — timeline chính xác **chưa verify từ nguồn chính thức** (chỉ có blog bên thứ 3 ghi mốc "trước 04/2027"). Phải tra lại https://cluster-api.sigs.k8s.io/reference/versions trước khi lên kế hoạch migrate thật.
- Đừng giả định CABPT/CACPPT (Talos, do Sidero Labs maintain độc lập) luôn theo kịp release của CAPI core — luôn tra `metadata.yaml`/compatibility matrix của provider trước khi `clusterctl init` hoặc upgrade.
- CAPC và CAPO là 2 project độc lập, mức độ actively maintained khác nhau theo thời điểm — **cần verify thực tế** (số maintainer, issue mở, ngày release gần nhất) trước khi chọn 1 trong 2 cho production, đừng chỉ dựa vào việc "CAPI hỗ trợ cả hai" để coi ngang hàng.
- (Mục này để tự bổ sung tiếp sau khi thao tác trực tiếp với management cluster thật — lesson learned thực chiến quan trọng hơn note lý thuyết)

### **10. Resources**

- Official book: https://cluster-api.sigs.k8s.io/
- Version support policy: https://cluster-api.sigs.k8s.io/reference/versions
- GitHub releases (nguồn version/changelog đáng tin nhất): https://github.com/kubernetes-sigs/cluster-api/releases
- Security guidelines cho infra provider: https://cluster-api.sigs.k8s.io/developer/providers/security-guidelines
- Talos bootstrap provider (CABPT): https://github.com/siderolabs/cluster-api-bootstrap-provider-talos
- Talos control-plane provider (CACPPT): https://github.com/siderolabs/cluster-api-control-plane-provider-talos
- **Lưu ý khi dùng note này**: số version cụ thể (v1.14.2, mốc v1beta1 unserved) lấy qua search engine/blog bên thứ 3, chưa đối chiếu trực tiếp CHANGELOG gốc. Verify lại bằng `clusterctl version` + GitHub Releases trước khi dùng để quyết định production.
