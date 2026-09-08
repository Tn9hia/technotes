# Trivy — Kubernetes Scanning (`trivy k8s` CLI vs Trivy Operator)
Tier: 2
Parent: [[Trivy]]
Related: [[trivy--db-management]], [[trivy--scanners]]
Tags: #trivy #kubernetes #cks #operator

## What it does

Trivy có **2 cách hoàn toàn khác nhau** để quét Kubernetes, dễ nhầm lẫn nếu không phân biệt rõ:

| | `trivy k8s` (CLI) | Trivy Operator |
|---|---|---|
| Cách chạy | On-demand, 1 lần rồi thoát | Controller chạy thường trực trong cluster |
| Trigger | Người vận hành gõ lệnh | Tự động theo K8s event (Pod tạo mới...) |
| Kết quả | In ra table/JSON tại chỗ | Lưu thành **Custom Resource** (`VulnerabilityReport`, `ConfigAuditReport`, `ExposedSecretReport`, `RbacAssessmentReport`, `InfraAssessmentReport`, `ClusterComplianceReport`, `SbomReport`), query được qua `kubectl get` | 
| Status | Feature EXPERIMENTAL (tính năng đang thay đổi) | Dự án đang ở trạng thái **incubating** — API/CRD có thể thay đổi |
| Dùng khi | Audit thủ công, đưa vào script CI/cron | Cần dashboard/compliance liên tục, tích hợp GitOps/kubectl-native |

Khi scan cluster, Trivy phân biệt 3 tầng: **cluster infrastructure** (control plane/kubelet), **cluster configuration** (Role/ClusterRole, resource YAML), và **application workload** (image chạy trong Pod) — mỗi tầng được scan bằng cơ chế khác nhau (image scan riêng, resource definition scan riêng).

## Why it exists

`trivy k8s` phù hợp cho audit định kỳ/manual hoặc chạy trong CI trước khi apply manifest, nhưng không thể "biết" khi nào 1 Pod mới xuất hiện giữa 2 lần chạy — dẫn đến khoảng trống giám sát (drift). Trivy Operator giải quyết đúng vấn đề này bằng cách sống trong cluster như 1 controller, watch API server, tự trigger scan ngay khi có thay đổi, biến kết quả scan thành 1st-class Kubernetes object (CRD) để tích hợp GitOps/alerting/dashboard dễ dàng hơn nhiều so với parse output JSON từ CLI.

## How it works (flow/diagram)

```
trivy k8s [flags] [CONTEXT]
        │
        ├─ Không có CONTEXT → dùng cluster default trong kubeconfig
        ├─ RBAC: cần "list" trên core/apps/batch/networking.k8s.io/rbac.authorization.k8s.io
        ├─ (mặc định) tải image từng workload để scan vuln/secret
        │       └─ --skip-images để tắt (chỉ scan resource definition, nhanh hơn nhiều)
        ├─ (mặc định) chạy Node-Collector job trên mỗi node
        │       └─ thu thập config phục vụ CIS Benchmark / infra assessment
        │       └─ --disable-node-collector để tắt
        └─ Output: --report summary (mặc định, gọn) hoặc --report all (chi tiết từng resource)


Trivy Operator (trong cluster)
        │
        ├─ Watch Pod/Deployment/... events qua K8s API
        ├─ Tạo Scan Job (giống node-collector nhưng cho workload scan)
        ├─ Ghi kết quả thành CRD: VulnerabilityReport, ConfigAuditReport,
        │   ExposedSecretReport, RbacAssessmentReport, InfraAssessmentReport,
        │   ClusterComplianceReport (NSA/CISA, CIS Benchmark), SbomReport
        └─ kubectl get vulnerabilityreports -A   ← query như mọi K8s resource khác
```

## Config gotchas

- **RBAC thiếu là nguyên nhân lỗi phổ biến nhất khi mới setup `trivy k8s`** — role cần `list` trên `"*"` resource của API group core rỗng (`""`), cộng thêm `apps`, `batch`, `networking.k8s.io`, `rbac.authorization.k8s.io`. Nếu bật node-collector (mặc định bật), còn cần thêm quyền `get` trên `nodes/proxy`, `pods/log`, `watch` trên `events`, và quyền tạo/xoá/theo dõi Job (`create/delete/watch` trên `jobs`, `list/get` trên `jobs/cronjobs`) cùng quyền `create` namespace.
- **`--include-namespaces` và `--exclude-namespaces` không dùng chung được** — chỉ chọn 1 trong 2. Và `--exclude-namespaces` **chỉ hoạt động với ClusterRole** (vì Trivy cần liệt kê được toàn bộ namespace trước khi loại trừ).
- **Node-Collector chạy như 1 Job trên MỖI node** — nếu node bị taint, job sẽ không schedule được trừ khi thêm `--tolerations` tương ứng. Không thấy CIS Benchmark/infra assessment report từ 1 số node → khả năng cao là do taint chưa được toleration.
- **`--skip-images` không tắt hết mọi thứ** — nó chỉ tắt phần tải & scan image (vuln/secret trong container), **misconfig scan trên resource YAML vẫn chạy bình thường**. Dùng flag này để scan nhanh khi chỉ cần audit config, không cần vuln image.
- **JSON output khó tách theo từng container trong multi-container Pod** — K8s coi Pod là 1 object, nên `--report summary` gộp chung; cần `--report all` để tách chi tiết theo từng image trong Pod.
- **Trivy Operator là dự án "incubating"** (theo chính docs của project) — nghĩa là API/CRD schema **có thể đổi giữa các version**, cần kiểm tra kỹ CHANGELOG khi upgrade operator trong production, không nên coi CRD schema là ổn định tuyệt đối như core K8s API.

## Security notes

- **ServiceAccount chạy `trivy k8s`/Trivy Operator có quyền đọc gần như toàn bộ cluster** (list mọi resource ở nhiều API group quan trọng, bao gồm RBAC objects) — đây là target hấp dẫn nếu bị compromise, vì đọc được RoleBinding/ClusterRoleBinding lộ ra sơ đồ phân quyền toàn cluster. Không nên cấp quyền `list` rộng hơn mức cần thiết nếu chỉ audit 1 vài namespace.
- **Node-Collector job cần chạy trên node, đọc `/proc` và log kubelet** — về bản chất tương tự mức độ nhạy cảm như 1 privileged DaemonSet (giống Falco), cần kiểm soát ai được sửa/xoá job này để tránh bị lợi dụng làm vector đọc thông tin node.
- **CRD report (VulnerabilityReport, RbacAssessmentReport...) lưu ngay trong etcd của cluster** — bất kỳ ai có quyền `get`/`list` trên các CRD này (kể cả không có quyền trực tiếp trên workload gốc) đều thấy được toàn bộ finding bảo mật của cluster, kể cả danh sách secret bị lộ (`ExposedSecretReport`). Cần RBAC riêng hạn chế đọc các CRD report này, không mặc định cho mọi user cluster-viewer.

## Refs

- https://trivy.dev/latest/docs/target/kubernetes/
- https://aquasecurity.github.io/trivy-operator/latest/
- https://github.com/aquasecurity/trivy-operator
- CKS: Supply Chain Security domain (~20% đề thi) dùng `trivy k8s`/`trivy image` là chính, không yêu cầu Trivy Operator trong phạm vi thi (verify lại theo exam guide mới nhất trước khi thi vì curriculum có thể cập nhật).
