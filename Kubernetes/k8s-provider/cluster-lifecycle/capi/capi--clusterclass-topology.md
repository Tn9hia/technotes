# CAPI — ClusterClass & Topology
Tier: 2
Parent: [[CAPI]]
Related: [[capi--clusterctl]], [[Multi-Tenancy]], [[Fleet-Ops]]
Tags: #capi #clusterclass #topology #templating

## What it does

`ClusterClass` là 1 CRD đóng vai trò "class/template" cho `Cluster` — định nghĩa sẵn infrastructure template, control-plane template, bootstrap template, và danh sách MachineDeployment "class" (worker pool type), cộng 1 schema `variables` (OpenAPI v3) để tham số hoá. Khi tạo 1 `Cluster` có `spec.topology.class: <ten-class>`, Cluster Topology Controller tự "dệt" (compose) ra `InfrastructureCluster`/`ControlPlane`/`MachineDeployment` thật từ template + patches + giá trị variables cung cấp — chỉ cần viết 1 đoạn YAML ngắn (class + version + variables) cho mỗi `Cluster` mới, thay vì copy-paste toàn bộ template dài mỗi lần.

## Why it exists

Trước `ClusterClass`, mỗi `Cluster` phải có riêng 1 bộ `InfrastructureClusterTemplate`/`ControlPlaneTemplate`/`MachineDeploymentTemplate` đầy đủ — tạo 100 cluster giống nhau (chỉ khác size/region/version) nghĩa là 100 bộ YAML gần như giống nhau, sửa 1 chỗ (vd bump base image) phải sửa lại tay ở toàn bộ. Nếu không có `ClusterClass`, muốn "bán" nhiều loại cluster chuẩn hoá cho khách phải tự dựng tầng template hoá riêng (Helm/kustomize/generator tự viết) nằm **ngoài** CAPI, mất đi lợi ích "mọi thứ qua 1 K8s API". `ClusterClass` đưa việc template hoá này vào ngay object model của CAPI: sửa 1 `ClusterClass` → mọi `Cluster` reference nó tự rollout theo, và RBAC cho "ai được sửa flavor" vs "ai chỉ được dùng flavor" tách biệt tự nhiên theo 2 loại object khác nhau — đúng là thứ cần cho mô hình bán nhiều "size" cluster (small/medium/large) cho nhiều tenant.

## How it works (flow/diagram)

```
ClusterClass (viết 1 lần)
 ├─ infrastructure.ref         → InfrastructureClusterTemplate (CAPC/CAPO)
 ├─ controlPlane.ref            → ControlPlaneTemplate (Talos CACPPT)
 ├─ workers.machineDeployments[] → mỗi "class" worker pool (vd "default", "gpu")
 └─ variables: []                → OpenAPI v3 schema (vd `controlPlaneReplicas`, `machineImage`)
      │
      │  patches: []  (JSON Patch, có enabledIf điều kiện theo variable)
      ▼
Cluster (viết mỗi lần tạo cluster mới — ngắn)
 spec.topology:
   class: <ten-class>
   version: v1.32.x
   controlPlane.replicas: 3
   workers.machineDeployments: [{class: default, replicas: 5}]
   variables: [{name: machineImage, value: "talos-1.9.0"}]
      │
      ▼
Topology Controller (core CAPI, namespace capi-system)
 - đọc ClusterClass + Cluster.spec.topology
 - áp patches (inline JSONPatch hoặc external qua Runtime SDK hook GeneratePatches)
 - compose ra InfrastructureCluster / ControlPlane / MachineDeployment THẬT
 - khi version/variable trong Cluster.spec.topology đổi → rollout machine-by-machine,
   xoá Machine cũ chỉ sau khi Machine mới healthy (không cho nhảy quá +2 minor K8s version 1 lần)
```

2 loại patch:
- **Inline patch** — viết thẳng trong `ClusterClass`, JSON Patch (add/remove/replace) nhắm vào 1 `selector` (match template nào) + `enabledIf` (Go-template condition, bật/tắt patch theo giá trị variable). Chỉ patch được `/spec`, và patch mảng (array) chỉ append/prepend — không patch được 1 phần tử ở index tuỳ ý.
- **External patch** — implement qua Runtime SDK, core gọi ra 1 RuntimeExtension (webhook ngoài) qua 3 hook: `DiscoverVariables`, `GeneratePatches`, `ValidateTopology`. Dùng khi logic patch phức tạp hơn JSON Patch tĩnh có thể diễn tả (vd tính giá trị dựa trên gọi API ngoài).

## Config gotchas

- Patch chỉ sửa được `/spec`, không sửa `/status` hay field ngoài spec — nếu cần thay đổi field khác, phải nghĩ lại thiết kế template, không ép patch.
- `enabledIf` sai cú pháp Go-template hoặc variable chưa có default trong schema → `Cluster` derive từ class fail ngay ở bước topology reconcile, message lỗi thường trỏ vào `ClusterClass` chứ không rõ `Cluster` nào gây ra — khó debug hơn lỗi YAML thường.
- **Upgrade CAPI core version có CRD apiVersion mới**: phải upgrade CAPI core **trước**, rồi mới sửa `apiVersion` reference trong `ClusterClass` **sau** — vì topology controller luôn dùng đúng version CRD được `ClusterClass` reference, đảo ngược thứ tự sẽ gây lỗi reconcile. Khi bump apiVersion cũng phải rà lại patch `selector`/JSONPatch `path` cho khớp schema mới.
- Giới hạn rollout: không cho nhảy **quá +2 minor Kubernetes version** trong 1 lần đổi `spec.topology.version` — đổi version cách xa phải đi qua version trung gian.
- Tránh đổi control-plane và worker cùng lúc trong 1 lần apply — dễ gây rollout chồng chéo, turnover hạ tầng (VM tạo/xoá) lớn hơn cần thiết, hoặc nghẽn do quá nhiều Machine rolling cùng lúc.
- Đổi label/annotation của Machine template trong `ClusterClass` cũng **trigger rollout** (tạo Machine mới) — dễ bất ngờ nếu nghĩ "chỉ là metadata, không ảnh hưởng gì".

## Security notes

- `ClusterClass` nên coi là object **có quyền hạn cao hơn** `Cluster` thường: ai sửa được `ClusterClass` có thể ảnh hưởng **mọi** `Cluster` đang reference nó (đổi image, đổi patch) — RBAC nên tách: team platform được sửa `ClusterClass`, tenant/self-service chỉ được tạo `Cluster` (chỉ set variables trong khuôn schema cho phép).
- External patch (Runtime SDK) chạy như 1 webhook ngoài, core CAPI gọi ra và **tin tưởng** response để build spec thật của `InfrastructureCluster`/`ControlPlane` — nếu RuntimeExtension bị compromise hoặc endpoint bị giả mạo (không có mTLS/cert pinning đúng cách), attacker có thể tiêm patch tuỳ ý vào mọi cluster derive từ class đó. ⚠️ **Cần verify**: cấu hình bảo mật RuntimeExtension (xác thực, network policy) cụ thể khi áp dụng thật — chưa kiểm tra chi tiết phần này.
- Variable schema nên validate chặt (OpenAPI v3 `enum`/`pattern`/giới hạn) nếu expose cho self-service, tránh tenant set variable ra giá trị ngoài dự kiến (vd tự chọn image lạ, resource quá lớn).

## Refs

- Proposal gốc: https://github.com/kubernetes-sigs/cluster-api/blob/main/docs/proposals/20210526-cluster-class-and-managed-topologies.md
- Operating a managed Cluster: https://cluster-api.sigs.k8s.io/tasks/experimental-features/cluster-class/operate-cluster
- Topology Mutation Hook (external patch): https://cluster-api.sigs.k8s.io/tasks/experimental-features/runtime-sdk/implement-topology-mutation-hook
- Proposal topology-mutation-hook: https://github.com/kubernetes-sigs/cluster-api/blob/main/docs/proposals/20220330-topology-mutation-hook.md
- Parent note: [[CAPI]]
