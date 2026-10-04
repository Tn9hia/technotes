# CAPI — clusterctl
Tier: 2
Parent: [[CAPI]]
Related: [[capi--clusterclass-topology]], [[Management-Plane]], [[Fleet-Ops]]
Tags: #capi #clusterctl #cli #operations

## What it does

`clusterctl` là CLI chính thức để quản lý lifecycle của chính CAPI/provider trên **management cluster** (không phải để quản lý workload bên trong tenant cluster) — khởi tạo management cluster (`init`), sinh YAML tạo workload cluster (`generate cluster`), di chuyển toàn bộ object CAPI giữa 2 management cluster (`move`), và upgrade version CAPI/provider đang chạy (`upgrade plan`/`upgrade apply`).

## Why it exists

Nếu không có `clusterctl`, cài CAPI nghĩa là tự `kubectl apply` đúng thứ tự CRD + controller deployment + RBAC + webhook cert cho **từng provider** (core, bootstrap, control-plane, infrastructure) — rất dễ sai thứ tự hoặc version lệch nhau. Muốn đổi management cluster (vd từ bootstrap cluster tạm trên laptop sang cluster HA thật) thì phải tự "export" toàn bộ `Cluster`/`Machine`/`Secret` liên quan và apply đúng thứ tự dependency sang cluster mới — dễ sót object hoặc tạo ra orphan reference. `clusterctl` chuẩn hoá cả 2 việc này thành 1 lệnh, dựa trên "provider contract" mà mọi provider phải tuân theo nên CLI này hoạt động được với **bất kỳ** infra provider (CAPC, CAPO, CAPA...) miễn tuân thủ contract.

## How it works (flow/diagram)

```
clusterctl init --bootstrap talos --control-plane talos --infrastructure capc
   │
   ├─ đọc clusterctl.yaml (provider repository config — version nào, lấy component YAML từ đâu)
   ├─ cài CAPI core + cert-manager (nếu chưa có) + từng provider theo thứ tự:
   │     core → bootstrap → control-plane → infrastructure
   └─ management cluster sẵn sàng nhận Cluster object

clusterctl generate cluster <name> --flavor <ten-flavor> --kubernetes-version vX > cluster.yaml
   │  (thay biến môi trường/flag vào template provider cung cấp sẵn)
   ▼
kubectl apply -f cluster.yaml   → tạo workload cluster thật

clusterctl move --to-kubeconfig <target>
   │
   ├─ set Cluster.spec.paused = true (dừng reconcile ở source) trên mọi Cluster liên quan
   ├─ chờ annotation clusterctl.cluster.x-k8s.io/block-move biến mất khỏi mọi object liên quan
   ├─ export toàn bộ object CAPI + dependency (Secret, ConfigMap reference...) theo contract
   ├─ apply sang management cluster đích
   └─ resume reconcile ngay khi object đã nằm ở cluster đích

clusterctl upgrade plan                        → liệt kê version provider hiện tại + version đề xuất
clusterctl upgrade apply --contract v1beta2     → upgrade theo contract version chỉ định
```

## Config gotchas

- `clusterctl init` cần cert-manager đã cài sẵn (hoặc tự cài kèm) — thiếu cert-manager thì webhook của core/provider không có cert, controller crash loop ngay khi khởi động, dễ nhầm là lỗi provider.
- Provider version lấy theo `clusterctl.yaml` (mặc định trỏ GitHub release của từng provider) — nếu không pin version rõ trong file config, chạy `init` ở 2 thời điểm khác nhau có thể ra 2 version provider khác nhau (vì mặc định lấy "latest"). Nên luôn pin version rõ khi setup production, đừng để default tự chọn.
- `clusterctl move`: **không** restore `Status` subresource (kể cả `status.conditions`) — controller ở cluster đích phải tự rebuild status từ spec. Theo doc chính thức, `move`/`move --to-directory` **không** được thiết kế cho production backup/restore — chỉ dùng cho mục đích pivot/di chuyển, muốn backup thật nên dùng Velero.
- Trong lúc `move` đang chạy, **không** được upgrade/scale/remediate Cluster liên quan — tài liệu chính thức cảnh báo có thể gây race condition chưa được xử lý.
- `clusterctl generate cluster` cần đúng `--flavor` mà provider publish (mỗi infra provider tự định nghĩa flavor riêng, ví dụ flavor khác cho external cloud-provider hay khác CNI) — flavor sai tên sẽ lấy nhầm template hoặc lỗi "flavor not found". ⚠️ **Cần verify**: danh sách flavor cụ thể mà CAPC/CAPO publish khi dùng thật, chưa tra trực tiếp.

## Security notes

- `clusterctl init` thường chạy với `kubectl` context có quyền cluster-admin trên management cluster (để tạo CRD, ClusterRole, webhook) — chỉ nên chạy bởi người có quyền vận hành platform, không giao cho tenant/self-service.
- `clusterctl move` di chuyển **cả Secret** (kubeconfig, credential provider) sang management cluster đích — nếu cluster đích chưa hardening tương đương cluster nguồn (RBAC, network policy), move có thể vô tình hạ thấp mức bảo vệ của credential nhạy cảm. Nên audit RBAC/network policy cluster đích **trước** khi move, không phải sau.
- Pin version provider trong `clusterctl.yaml`/config override, tránh trỏ thẳng "latest" trong môi trường production — vừa để ổn định, vừa tránh vô tình kéo về 1 version provider có lỗ hổng mới công bố mà chưa kịp đánh giá.

## Refs

- clusterctl Commands overview: https://cluster-api.sigs.k8s.io/clusterctl/commands/commands
- `clusterctl move`: https://cluster-api.sigs.k8s.io/clusterctl/commands/move
- `clusterctl init`: https://cluster-api.sigs.k8s.io/clusterctl/commands/init
- Quick Start (ví dụ `generate cluster`): https://cluster-api.sigs.k8s.io/user/quick-start
- Parent note: [[CAPI]]
