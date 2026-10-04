# Fleet Ops — Vận Hành Nhiều Tenant Cluster
Tags: #k8s-provider #fleet-ops #gitops #autoscaler #backup-dr
Related: [[CAPI]], [[Talos]], [[Management-Plane]], [[Multi-Tenancy]], [[Provider-Security]]
Last updated: 2026-10-03

> ⚠️ **Lưu ý nguồn**: Phần GitOps-cho-CAPI và annotation Cluster Autoscaler lấy từ docs chính thức (`kubernetes/autoscaler` README, Cluster API Book) nên tương đối tin được; phần "case study GitOps thực tế áp dụng cho CAPI ở quy mô production" **chưa tìm được 1 case study cụ thể đáng tin** (chỉ có blog/so sánh tool chung, không phải write-up vận hành thật) — coi phần đó là pattern lý thuyết, cần tự verify khi làm thật.

---

### **1. What — Nó là cái gì?**

Fleet Ops là nhóm kiến thức về vận hành **nhiều** tenant cluster cùng lúc — khác với [[CAPI]] (cơ chế object để tạo 1 cluster) hay [[Talos]] (OS của 1 node), ở đây là lớp "quản lý vĩ mô" cả fleet: dùng GitOps để quản lý hàng loạt `Cluster`/`ClusterClass` CR trong management cluster, để `MachineHealthCheck` + Cluster Autoscaler tự co giãn/tự chữa theo từng cluster, và tách 2 lớp backup/restore (management cluster vs từng tenant cluster) cho disaster recovery.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Nếu không có lớp Fleet Ops, vận hành 10-100 tenant cluster bằng CAPI "trần" (chỉ có object + controller) sẽ gặp các vấn đề:
- **Thao tác tay từng cluster** — muốn đổi version K8s hay thêm 1 field cho toàn bộ fleet phải tự viết script loop qua từng `Cluster` CR, không có audit trail "ai đổi gì lúc nào" rõ ràng như Git history.
- **Không tự scale theo tải** — CAPI tự nó chỉ giữ đúng số replica đã khai trong `MachineDeployment`; muốn co giãn theo tải thực tế (nhiều tenant cùng lúc tăng workload) phải tự viết logic ngoài, hoặc dùng Cluster Autoscaler (nhưng phải tự tích hợp, không tự động có sẵn).
- **Backup "cả cục"** — nếu chỉ nghĩ "backup management cluster là đủ" sẽ sai: mất management cluster không làm chết tenant cluster đang chạy, nhưng etcd của **từng tenant cluster** vẫn cần backup riêng (crash etcd trong 1 tenant không liên quan gì tới management cluster).

Fleet Ops lấp khoảng trống: biến "sửa fleet" thành 1 commit Git (GitOps), biến "co giãn theo tải" thành tự động (MachineHealthCheck + Autoscaler), và tách rõ 2 lớp backup để disaster recovery không bị hiểu lầm là "backup 1 chỗ là xong".

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng khi:** số lượng tenant cluster đủ lớn để thao tác tay không còn scale (dù chỉ vài chục cluster là đã đáng áp GitOps); cần audit trail "ai đổi flavor/version lúc nào" cho compliance; cần tự động co giãn worker node theo tải thực tế của từng tenant mà không cần người can thiệp.

**KHÔNG dùng khi:**
- Mới có 1-2 cluster test/lab — thêm Flux/Argo CD + Cluster Autoscaler vào giai đoạn này là overhead vận hành không cần thiết, áp dụng tay qua `kubectl`/`clusterctl` còn dễ debug hơn.
- Tenant có pattern tải rất ổn định, biết trước (không cần autoscale) — chạy cố định `replicas` trong `MachineDeployment` đơn giản và dễ dự đoán chi phí hơn autoscaler.
- Chưa có hạ tầng object storage (S3-compatible) để chứa Velero backup — nên giải quyết hạ tầng này trước, đừng triển khai Velero mà không có nơi lưu backup đáng tin.

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
┌─────────────────────────────┐
│   Git Repo (nguồn chân lý)    │   chứa Cluster / ClusterClass / MachineDeployment YAML
│   1 thư mục / branch per      │   cho từng tenant hoặc từng "flavor"
│   tenant hoặc per-environment  │
└───────────────┬───────────────┘
                │  reconcile (pull-based, định kỳ hoặc qua webhook)
                ▼
┌─────────────────────────────────────────────────────────────┐
│                    Management Cluster                          │
│   Flux/ArgoCD Controller  ──apply──▶  CAPI Core + CABPT/CACPPT  │
│                                        + CAPC/CAPO (xem [[CAPI]])│
│                                                                  │
│   MachineHealthCheck (per Cluster) ──remediate──▶ xoá/tạo lại   │
│   Machine khi Node unhealthy quá threshold                      │
│                                                                  │
│   Cluster Autoscaler (cloudprovider=clusterapi)                 │
│     đọc annotation min/max-size trên MachineDeployment          │
│     ──scale──▶ sửa .spec.replicas của MachineDeployment          │
└───────────────┬───────────────────────────────┬─────────────────┘
                │ tạo/quản lý                     │ backup (Velero)
                ▼                                 ▼
   ┌─────────────────────────┐        ┌───────────────────────────┐
   │  Tenant Cluster #1..N     │        │ Object Storage (S3-compat) │
   │  (control-plane + worker  │        │  - Velero: backup CRD +    │
   │   chạy Talos + K8s)       │        │    Secret của mgmt cluster  │
   │                           │        │  - talosctl etcd snapshot: │
   │  etcd riêng mỗi cluster ──┼────────▶   backup etcd TỪNG tenant  │
   └─────────────────────────┘        └───────────────────────────┘
```

**2 lớp backup độc lập, đừng nhầm làm 1**: (1) Velero backup **management cluster** (toàn bộ `Cluster`/`Machine`/`MachineDeployment`/Secret kubeconfig+credential IaaS) — mất cái này mất khả năng *quản lý* fleet; (2) `talosctl etcd snapshot` backup **etcd của từng tenant cluster** — mất cái này mất dữ liệu workload của riêng tenant đó. 2 việc không thay thế được cho nhau.

### **5. How — Cơ chế hoạt động**

- **GitOps cho CAPI** — pattern phổ biến: Flux hoặc Argo CD chạy trong management cluster, theo dõi 1 Git repo chứa `Cluster`/`ClusterClass`/`MachineDeployment` YAML (thường tổ chức 1 thư mục/branch per tenant hoặc per-flavor), tự `kubectl apply` khi có commit mới. Flux thiên về "mỗi cluster tự kéo config của mình" (không có control-plane trung tâm riêng), Argo CD thiên về "GitOps-as-a-service" có UI/console tập trung — phù hợp hơn nếu platform team cần giao diện cho người không phải cluster-admin xem trạng thái fleet.
- **`MachineHealthCheck`** — monitor Node condition theo điều kiện khai báo (vd `Ready=False` quá X phút), field quan trọng: `nodeStartupTimeout` (default **10 phút** — Machine mới tạo không join cluster kịp trong thời gian này bị coi là failed; set `0` để tắt check này) và `maxUnhealthy` (ngưỡng — nếu số Machine unhealthy vượt ngưỡng, **dừng remediation** để tránh cascading failure, chờ can thiệp tay). Remediation mặc định là xoá + tạo lại Machine; có thể cắm `remediationTemplate` (external remediation, vd `SelfNodeRemediationTemplate`) nếu cần logic phức tạp hơn.
- **Cluster Autoscaler (cloudprovider `clusterapi`)** — đọc annotation trên `MachineDeployment`: `cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size` và `cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size`, tự sửa `.spec.replicas` trong giới hạn đó theo tải Pod thực tế (pending Pod không schedule được → scale up; node rảnh lâu → scale down). Scale-from-zero (MachineDeployment bắt đầu từ 0 replica) cần enable riêng, không phải default.
- **Rolling upgrade theo fleet** — không có cơ chế "upgrade tất cả cluster 1 lần" tích hợp sẵn; thực hành phổ biến là tự phân nhóm tenant (canary nhóm nhỏ trước, rồi mở rộng dần) bằng cách commit thay đổi `spec.topology.version` theo thứ tự trong Git, dựa trên cơ chế rollout machine-by-machine đã có ở tầng [[capi--clusterclass-topology]] cho **1** cluster — Fleet Ops chỉ là lớp điều phối **thứ tự** áp dụng qua nhiều cluster, bản thân CAPI không biết gì về "fleet".
- **Backup/restore 2 lớp** — xem diagram mục 4.

### **6. Key Config — Cấu hình cần nhớ**

- `nodeStartupTimeout` default **10 phút** — nếu VM trên CloudStack/OpenStack thường boot chậm hơn mức này (vd do image lớn hoặc IaaS đang tải cao), Machine mới sẽ bị remediate (xoá & tạo lại) ngay khi chưa kịp join, gây loop tạo-xoá vô ích. Nên đo thời gian boot thật trước khi set threshold.
- `maxUnhealthy` nên set theo tỉ lệ phần trăm hoặc số tuyệt đối tuỳ ngữ cảnh — set quá cao (vd 100%) làm MachineHealthCheck vô dụng (không bao giờ dừng remediation dù cả cluster sập), set quá thấp (vd 1) dễ false-positive dừng remediation chỉ vì 1 node tạm thời flap.
- ⚠️ Annotation Cluster Autoscaler tìm thấy **không hoàn toàn đồng nhất giữa các nguồn** (1 nguồn ghi prefix `cluster.x-k8s.io/`, 1 nguồn khác ghi `cluster.k8s.io/` cho max-size) — **cần verify đúng prefix tại version `kubernetes/autoscaler` đang dùng** (xem `cloudprovider/clusterapi/README.md` trên GitHub) trước khi áp dụng, gõ sai prefix annotation sẽ khiến autoscaler coi như MachineDeployment không có giới hạn cấu hình, im lặng không scale.
- Flux/Argo CD cần RBAC riêng trên management cluster để tạo/sửa `Cluster`/`MachineDeployment` — không nên dùng chung ServiceAccount với provider controller (CAPC/CAPO), xem thêm [[Provider-Security]].
- Velero backup Secret (kubeconfig + credential IaaS) nghĩa là **bucket lưu backup cũng trở thành tài sản nhạy cảm tương đương management cluster** — phải bảo vệ access control bucket S3-compatible ở mức tương đương.

### **7. Security Considerations**

- Pipeline GitOps (Flux/Argo CD) có quyền apply trực tiếp vào management cluster — nếu Git repo hoặc CI/CD credential push vào repo đó bị compromise, attacker gián tiếp có quyền tạo/sửa `Cluster` trên toàn fleet (vd tạo cluster trỏ credential IaaS khác, hoặc sửa `ClusterClass` ảnh hưởng hàng loạt tenant — xem thêm rủi ro ClusterClass ở [[capi--clusterclass-topology]]). Nên bảo vệ branch chính (require review/PR) cho repo chứa `Cluster`/`ClusterClass` giống mức độ bảo vệ 1 production deploy pipeline thật, không coi nhẹ vì "chỉ là YAML".
- Cluster Autoscaler cần quyền sửa `MachineDeployment` — ServiceAccount của nó nên giới hạn đúng resource này, không cấp thừa quyền quản trị `Cluster`/Secret.
- Backup Velero (chứa Secret nhạy cảm của mgmt cluster) nên **encrypt at rest** ở phía object storage + giới hạn ai có quyền đọc bucket, vì về bản chất đây là 1 bản sao đầy đủ của "root of trust" đã nói ở [[CAPI]] mục 7.
- `talosctl etcd snapshot` của từng tenant cluster chứa dữ liệu workload thật của khách hàng — nếu nhiều tenant chia sẻ cùng 1 bucket backup, cần tách quyền đọc theo tenant (namespace/prefix riêng + IAM scope riêng trên object storage), tránh 1 tenant đọc được backup của tenant khác.

### **8. Ops Runbook — Production Notes**

- **Health check GitOps**: `flux get kustomizations -A` hoặc Argo CD UI/`argocd app list` — xem app/kustomization nào đang `OutOfSync`/lỗi reconcile, thường là dấu hiệu đầu tiên khi 1 thay đổi fleet-wide bị kẹt.
- **Health check autoscaler**: log của pod `cluster-autoscaler` (namespace tuỳ cách deploy) — khi thấy pending Pod không được scale up dù dưới `max-size`, thường do annotation sai tên/giá trị (xem mục 6) hoặc quota IaaS (CloudStack/OpenStack) đã hết.
- **Metric cần alert**: số `MachineHealthCheck` đang ở trạng thái "remediation paused" (do vượt `maxUnhealthy`) — đây là dấu hiệu sự cố lớn hơn bình thường (nhiều node cùng fail), cần người can thiệp tay ngay, không để tự động xử lý tiếp.
- **Backup verify**: định kỳ test-restore Velero backup vào 1 management cluster tạm (không phải chỉ tin tưởng backup job "chạy thành công" là đủ — object resource restore được không đồng nghĩa controller sẽ reconcile đúng, nhất là với giới hạn "không restore Status" đã nói ở [[capi--clusterctl]]). Tương tự, định kỳ test-restore `talosctl etcd snapshot` của ít nhất 1 tenant cluster mẫu.
- **Runbook rolling upgrade fleet**: luôn upgrade theo nhóm nhỏ (canary) trước khi áp toàn fleet, theo dõi `MachineHealthCheck`/alert trong suốt quá trình nhóm canary rollout xong mới mở rộng — tự xây quy trình này vì CAPI/Fleet Ops không có cơ chế "pause toàn fleet nếu 1 cluster lỗi" tích hợp sẵn.

### **9. Gotchas & Lessons Learned**

- Đừng tưởng "Velero backup management cluster" = "đã an toàn cho toàn bộ dữ liệu" — dữ liệu thật của tenant (etcd, PV) nằm ở tầng tenant cluster, phải backup riêng từng cluster, xem mục 4.
- `nodeStartupTimeout` default 10 phút là con số tham khảo từ docs chung, **không có gì đảm bảo đúng với tốc độ boot VM thật trên CloudStack/OpenStack** — luôn đo thực tế trước khi tin vào default.
- Annotation Cluster Autoscaler prefix (`cluster.x-k8s.io/` vs `cluster.k8s.io/`) không nhất quán giữa các nguồn tài liệu tìm được — đây là loại lỗi "âm thầm không báo" (autoscaler không log lỗi rõ ràng, chỉ đơn giản không scale), nên test kỹ bằng cách quan sát hành vi thật, không chỉ đọc 1 nguồn rồi tin.
- Chưa tìm được 1 case study GitOps-cho-CAPI ở quy mô production thật (chỉ có bài so sánh tool chung) — các pattern viết trong note này (branch/folder per tenant, Flux pull-based) là suy luận hợp lý từ docs chính thức của Flux/CAPI riêng lẻ, **chưa phải đã được ai đó xác nhận chạy tốt ở quy mô lớn**. Tự validate bằng lab trước khi tin tưởng 100%.
- (Mục này để tự bổ sung tiếp sau khi vận hành fleet thật — lesson learned thực chiến quan trọng hơn note lý thuyết)

### **10. Resources**

- Cluster Autoscaler + CAPI provider (nguồn chính thức cho annotation): https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/clusterapi/README.md
- Autoscaling — Cluster API Book: https://cluster-api.sigs.k8s.io/tasks/automated-machine-management/autoscaling
- Configure a MachineHealthCheck — Cluster API Book: https://cluster-api.sigs.k8s.io/tasks/healthcheck (bản mới nhất; tham khảo thêm bản theo version nếu field khác)
- Velero docs: https://velero.io/docs/
- Flux docs: https://fluxcd.io/flux/ | Argo CD docs: https://argo-cd.readthedocs.io/
- **Lưu ý khi dùng note này**: phần GitOps-cho-CAPI ở mức pattern/best-practice suy luận từ docs riêng lẻ, chưa có case study production xác nhận; annotation autoscaler cần đối chiếu đúng version `kubernetes/autoscaler` đang dùng trước khi áp dụng.
