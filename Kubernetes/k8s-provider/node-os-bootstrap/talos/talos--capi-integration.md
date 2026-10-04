# Talos — Tích hợp với CAPI (CABPT/CACPPT)
Tier: 2
Parent: [[Talos]]
Related: [[CAPI]], [[capi--clusterclass-topology]], [[capi--clusterctl]]
Tags: #talos #capi #cabpt #cacppt #machine-config

## What it does

Mô tả cách 2 provider do Sidero Labs maintain — **CABPT** (bootstrap provider) và **CACPPT** (control-plane provider) — biến object CAPI trừu tượng (`Machine`, `TalosControlPlane`) thành machine config Talos thật, tự động sinh secret/cert cần cho cluster (CA, bootstrap token), và cách customize qua `configPatches`/`strategicPatches`.

## Why it exists

Nếu không có CABPT/CACPPT, dùng Talos với CAPI sẽ phải tự `talosctl config generate` machine config cho từng node, tự quản lý CA/cert/bootstrap token ngoài K8s API, rồi tự nhồi data đó vào field bootstrap data của `Machine` — mất hết lợi ích "mọi thứ qua CAPI reconcile loop". CABPT/CACPPT lấp khoảng trống: core CAPI tạo `Machine` → provider tự sinh machine config tương ứng (`worker.yaml`/`controlplane.yaml`) + tự quản lý Talos CA/PKI/bootstrap token hoàn toàn trong vòng đời CAPI, không cần bước tay nào ở giữa.

## How it works (flow/diagram)

```
Cluster (CAPI core)
 └─ controlPlaneRef → TalosControlPlane (CACPPT)
      spec.version / replicas / infrastructureTemplate / controlPlaneConfig:
        generateType: controlplane   (auto-gen — khuyến nghị)
          hoặc: none + data: <machine config viết tay qua `talosctl config generate`>
        controlPlaneConfig.strategicPatches: [...]

 └─ bootstrap.configRef (worker) → TalosConfigTemplate (CABPT)
      spec.template.spec:
        generateType: worker
        configPatches: [{op: add, path: /machine/network/..., value: ...}]

Machine (CAPI core, do MachineDeployment tạo)
 └─ bootstrap.dataSecretName  ◀── CABPT render ra Secret chứa machine config YAML cuối cùng
 └─ infrastructureRef          ◀── CAPC/CAPO tạo VM thật, đưa machine config vào qua user-data

Lần đầu bootstrap cluster:
 CACPPT tự sinh Talos CA + PKI cluster (etcd CA, K8s CA, bootstrap token), lưu trong Secret của
 management cluster (<cluster-name>-ca, <cluster-name>-talosconfig...) — provider tự quản lý,
 không phải thao tác tay như `talosctl gen secrets` kiểu standalone.
```

## Config gotchas

- 2 cách sinh config: `generateType: controlplane/worker` (auto, provider tự render — **khuyến nghị**, giữ được khả năng provider quản lý lifecycle đầy đủ) vs `generateType: none` + field `data` chứa **toàn bộ machine config viết tay**. Dùng `none` + `data` sẽ **mất khả năng** CACPPT tự sinh `talosconfig` sau này — chỉ nên dùng khi có lý do đặc biệt (migrate cluster có sẵn), không nên chọn làm default.
- `configPatches` (CABPT, JSON Patch kiểu cũ) vs `strategicPatches` (CACPPT, patch kiểu mới hơn, gần giống strategic-merge-patch) — 2 provider dùng 2 cơ chế patch **không giống nhau hoàn toàn**. ⚠️ **Cần verify** lại cú pháp chính xác từng loại tại version đang dùng trước khi viết patch thật — dễ nhầm cú pháp giữa 2 bên nếu quen 1 loại.
- CABPT/CACPPT là 2 project riêng do Sidero Labs maintain **ngoài** CAPI core, update không đồng bộ với CAPI core hay CAPC/CAPO — áp dụng y nguyên gotcha version-skew đã note ở [[CAPI]] cho cặp provider này.
- Tài liệu chính thức mô tả quan hệ `TalosControlPlane` ↔ `TalosConfigTemplate` ↔ Secret sinh ra hiện khá sơ sài. ⚠️ Phần lifecycle secret/cert ở trên tổng hợp từ nhiều nguồn blog + mô tả repo, **cần verify lại bằng cách đọc source/CRD thật** (`kubectl explain taloscontrolplane` trên cluster thật) trước khi dùng để thiết kế production.

## Security notes

- Secret CA/PKI do CACPPT tự sinh (etcd CA, K8s CA, bootstrap token) nằm trong management cluster — đây chính là phần được nhắc ở mục Security của [[CAPI]] ("compromise management cluster = lộ toàn bộ credential fleet"), áp dụng trực tiếp: ai đọc được Secret này có thể mạo danh control-plane của **bất kỳ** tenant cluster nào do CACPPT quản lý.
- `generateType: none` + machine config viết tay trong field `data` — nếu secret/cert bị nhúng thẳng vào đó dưới dạng plaintext trong YAML (thay vì để provider tự quản lý qua Secret riêng), dễ vô tình commit secret vào Git nếu quản lý `TalosControlPlane` theo GitOps → xem [[Fleet-Ops]] về rủi ro GitOps + secret.
- Kết hợp với các CVE về role API của Talos đã nêu ở [[Talos]] (`GHSA-rjwj-368c-f82r` và nhóm advisory 18/09/2026): `talosconfig` do CACPPT sinh ra thường có role cao (để quản lý control-plane) — audit kỹ ai/thành phần nào trong management cluster có quyền đọc Secret `talosconfig` này.

## Refs

- CACPPT repo: https://github.com/siderolabs/cluster-api-control-plane-provider-talos
- CABPT repo: https://github.com/siderolabs/cluster-api-bootstrap-provider-talos
- Parent note: [[Talos]]
- Liên quan: [[CAPI]], [[capi--clusterclass-topology]]
