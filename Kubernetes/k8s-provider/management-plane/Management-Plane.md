# Management Plane — Cluster Quản Trị CAPI
Tags: #k8s-provider #management-plane #ha #observability #pivot
Related: [[CAPI]], [[capi--clusterctl]], [[Fleet-Ops]], [[Provider-Security]], [[Talos]]
Last updated: 2026-10-03

> ⚠️ **Lưu ý về nguồn**: Resource sizing cụ thể và case study "self-hosted management cluster" trong note này lấy từ search engine + 1 PR thực tế (Deckhouse) + CAPI book — không phải benchmark tự làm trên hạ tầng KVM/CloudStack/OpenStack của mình. Coi đây là điểm khởi đầu, không phải số liệu production-ready.

---

### **1. What — Nó là cái gì?**

Management Plane là **chính cluster K8s chạy CAPI controller** (core + CABPT/CACPPT + CAPC/CAPO) — khác với "tenant cluster" mà nó tạo ra. Đây không phải 1 tool riêng, mà là **vai trò vận hành** của 1 K8s cluster: nó không chạy workload khách hàng, chỉ chạy controller để quản lý vòng đời của các cluster khác. Trong [[CAPI]] đã nói management cluster là "root of trust của toàn bộ fleet" — note này đi sâu vào cách build/vận hành/giám sát chính cluster đó cho đúng, không lặp lại định nghĩa CAPI.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Nếu không tách riêng "management plane" như 1 khái niệm vận hành độc lập, dễ rơi vào 2 sai lầm:
- **Coi management cluster như 1 cluster "phụ", không đầu tư HA/backup đúng mức** — vì nó không chạy workload khách nên trông "không quan trọng". Nhưng mất nó = mất khả năng *quản lý* toàn bộ fleet (scale, upgrade, xoá), và nếu chưa backup đúng etcd của nó, có thể mất luôn cả khả năng biết "tenant cluster nào đang tồn tại, cấu hình gì" — dù tenant cluster vẫn chạy.
- **Chạy CAPI "lỏng" ngay trên 1 kind cluster tạm trên laptop** rồi quên pivot sang nơi bền — hợp lý cho dev/test, nhưng không ai định nghĩa rõ "khi nào phải pivot sang cluster thật" thì dễ vô tình chạy production dựa trên 1 cluster có thể biến mất khi tắt laptop.

Tách riêng "management plane" thành 1 group kiến thức buộc phải trả lời rõ: cluster này HA ở mức nào, backup ra sao, ai giám sát nó, và quy trình "bootstrap → pivot → cluster HA thật" cụ thể là gì — thay vì để nó trôi theo quán tính "tiện đâu chạy đó".

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Áp dụng khi:** đã có từ 1 tenant cluster "thật" (không phải demo) do CAPI quản lý — tức ngay khi CAPI rời khỏi giai đoạn lab, cần nghĩ tới management plane như 1 hệ thống tier-0 cần SLA riêng.

**KHÔNG cần đầu tư nặng khi:**
- Đang ở giai đoạn thử nghiệm/học CAPI — 1 kind/k3d cluster tạm trên máy cá nhân là đủ, chưa cần HA/backup chính quy cho management cluster.
- Số lượng tenant cluster rất nhỏ (1-2) và có thể chấp nhận downtime quản lý vài giờ khi cần — overhead vận hành 1 management cluster HA 3-node có thể không đáng so với quy mô thật.

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
Giai đoạn 1 — Bootstrap (tạm, trên máy cá nhân/CI runner)
┌─────────────────────────┐
│  kind/k3d cluster tạm     │   clusterctl init (core+CABPT+CACPPT+CAPC/CAPO)
│  (KHÔNG dùng production)  │   clusterctl generate cluster <mgmt-cluster-flavor>
└────────────┬─────────────┘
             │ tạo ra 1 Cluster object MỚI = chính management cluster HA thật
             ▼
Giai đoạn 2 — Target management cluster (HA, chạy trên KVM/CloudStack/OpenStack)
┌───────────────────────────────────────────────────────────────┐
│  3 control-plane node (Talos) ──── etcd quorum riêng của        │
│                                     management cluster            │
│  Chạy: CAPI core + CABPT + CACPPT + CAPC/CAPO controller          │
└───────────────────────────────┬────────────────────────────────┘
             clusterctl move    │  (pause reconcile ở bootstrap cluster,
             (pivot)            │   export toàn bộ Cluster/Machine/Secret,
                                 │   apply + resume ở cluster đích)
             ▼
Giai đoạn 3 — Steady state
┌───────────────────────────────────────────────────────────────┐
│       Management Cluster (HA, bền, giám sát riêng)                │
│  quản lý N tenant cluster  →  xem [[CAPI]], [[Fleet-Ops]]           │
└───────────────────────────────────────────────────────────────┘
       sau khi pivot xong → xoá bootstrap cluster tạm (đã hết nhiệm vụ)
```

**Dependency ngược (chicken-and-egg)**: controller tạo cluster phải chạy *trên* 1 cluster — nên management cluster thật phải được tạo ra **từ** 1 cluster tạm khác (bootstrap cluster), không thể tự tạo ra chính nó từ đầu. Nếu chọn mô hình **self-hosted** (management cluster tự quản lý chính nó bằng CAPI, tức có luôn `Cluster` object đại diện cho bản thân nó), cần hiểu rõ rủi ro ở mục 9.

### **5. How — Cơ chế hoạt động**

- **Bootstrap & Pivot** — quy trình chuẩn theo CAPI book: tạo bootstrap cluster tạm (kind/k3d) → `clusterctl init` → dùng nó tạo management cluster thật trên hạ tầng production → `clusterctl move` di chuyển toàn bộ object CAPI sang cluster đích → xoá bootstrap cluster. Chi tiết lệnh/gotcha của `move` → [[capi--clusterctl]].
- **Self-hosted management cluster** — management cluster có 1 `Cluster` object đại diện cho chính nó (tự quản lý bằng CAPI). Cho phép dùng chính CAPI để scale/upgrade control-plane của management cluster, nhưng tạo ra vòng phụ thuộc: nếu controller bên trong nó gặp lỗi trong lúc tự reconcile chính mình, không có "cluster ngoài" nào cứu — xem gotcha mục 9.
- **HA sizing** — theo khuyến nghị etcd chuẨn (không riêng CAPI): tối thiểu 3 control-plane node cho quorum chịu được 1 node chết. Số lượng **provider controller** chạy đồng thời trên management cluster (core + CABPT + CACPPT + CAPC + CAPO) là điểm khác biệt so với 1 K8s cluster "trống" bình thường — càng nhiều provider, request tới kube-apiserver + lượng reconcile loop càng lớn, cần tính vào capacity, không chỉ tính theo số node quản lý.
- **Tuning controller-runtime** — mỗi provider controller có flag riêng `--kube-api-qps`/`--kube-api-burst` để điều chỉnh tốc độ gọi API server; tăng quá tay có thể làm nghẽn apiserver thay vì giúp nhanh hơn — xem CAPI book "Tuning controllers".
- **Observability** — ⚠️ **chưa tìm được dashboard Grafana chính thức do CAPI project publish** cho riêng management plane. Hướng thực tế: scrape `/metrics` default của controller-runtime (`controller_runtime_reconcile_errors_total`, `controller_runtime_reconcile_time_seconds`) cho từng provider, cộng với theo dõi `kubectl get clusters -A` / `clusterctl describe` ở tầng ứng dụng → chi tiết vận hành/backup fleet → [[Fleet-Ops]].

### **6. Key Config — Cấu hình cần nhớ**

- **Resource request/limit của provider controller dễ set quá thấp.** Có case thực tế (Deckhouse, qua PR công khai): `capi-controller-manager` mặc định request ~50Mi memory, nhưng VPA cap ở 70Mi từng gây **memory pressure → controller không trả lời health probe → kubelet tự restart liên tục**; phải tăng cap VPA lên 256Mi để ổn định. Bài học: đừng copy default resource request/limit mà không theo dõi thực tế sau khi chạy nhiều provider cùng lúc.
- Chạy **nhiều provider cùng lúc** (core + CABPT + CACPPT + CAPC + CAPO) trên cùng management cluster nghĩa là nhiều controller cùng gọi kube-apiserver — nếu apiserver của management cluster sizing theo "cluster nhỏ, ít node" mà không tính thêm tải từ chính các provider, dễ bị throttle/API server chậm dù số *node vật lý* quản lý không nhiều.
- ⚠️ **Chưa có khuyến nghị CPU/RAM chính thức cho toàn bộ management cluster** (tổng hợp core + mọi provider) ở quy mô fleet cụ thể của mình (bao nhiêu tenant cluster) — cần tự benchmark bằng cách tăng dần số `Cluster` test và quan sát resource controller, không suy từ tài liệu generic.
- RBAC & namespace tách biệt cho từng provider (`capi-system`, `capi-bootstrap-system`, `capi-controlplane-system`, `capc-system`/`capo-system`) nên giữ đúng theo convention cài đặt của `clusterctl init`, tránh gộp chung namespace "cho gọn" — vì RBAC least-privilege của từng provider dựa trên ranh giới namespace này.

### **7. Security Considerations**

- Management cluster là nơi tập trung **toàn bộ credential nhạy cảm của fleet** (đã nói ở [[CAPI]] mục 7) — ở góc độ vận hành plane này, điều cần thêm là: **ai có quyền `kubectl`/SSH-equivalent (ở đây là `talosctl`) vào chính management cluster** phải là nhóm hẹp nhất có thể, tách biệt hoàn toàn khỏi nhóm vận hành tenant cluster thông thường.
- Nếu dùng self-hosted (management cluster tự quản lý chính nó), namespace/ServiceAccount của provider controller vừa có quyền ghi vào **chính hạ tầng nó đang chạy trên** — một lỗi logic hoặc bug trong reconcile loop lý thuyết có thể ảnh hưởng ngược lại chính node đang chạy controller đó. Nên đánh giá kỹ trước khi chọn mô hình này cho production, non-self-hosted (quản lý từ 1 management cluster tách biệt, không tự quản lý mình) đơn giản hơn để reason về blast radius.
- Backup etcd của management cluster (xem mục 8) phải mã hoá at-rest và hạn chế quyền đọc tương đương mức bảo vệ của chính management cluster — backup là 1 bản sao đầy đủ của mọi Secret/credential, không kém nhạy cảm hơn bản chạy thật.

### **8. Ops Runbook — Production Notes**

- **Health check tổng quan**: ngoài `kubectl get clusters -A` (ở tầng CAPI, xem [[CAPI]]), cần check riêng health của chính management cluster như 1 K8s cluster bình thường (etcd quorum, control-plane component) — dùng `talosctl health` nếu control-plane chạy Talos, xem [[Talos]].
- **Metric cần alert riêng cho management plane**: `controller_runtime_reconcile_errors_total` tăng bất thường ở **bất kỳ** provider nào (không chỉ CAPI core); resource usage (memory) của từng provider controller — vì đã có tiền lệ thực tế memory limit quá thấp gây crash loop (xem mục 6).
- **Backup/restore**: etcd snapshot của **chính management cluster** (khác hoàn toàn etcd của từng tenant cluster) là bản backup quan trọng nhất trong toàn hệ thống — mất nó mà không có backup đúng kỷ luật (không chỉ dựa vào `clusterctl move --to-directory`, vì nó không restore `Status`) đồng nghĩa phải "khám phá lại" tenant cluster nào đang tồn tại bằng cách quét trực tiếp CloudStack/OpenStack. Theo tài liệu/case thực tế: `clusterctl move` **không phải** công cụ backup/restore — cluster mất etcd không "hồi phục" bằng cách chạy lại `move`, cần kỷ luật backup riêng (Velero + etcd snapshot) như với bất kỳ datastore tier-0 nào.
- **Quy trình pivot (bootstrap → production)**: luôn giữ bootstrap cluster sống cho tới khi xác nhận `clusterctl move` thành công và management cluster đích đã start reconcile đúng (`kubectl get clusters -A` ở cluster đích show đúng trạng thái) — chỉ xoá bootstrap cluster sau bước xác nhận này, không xoá ngay sau khi lệnh `move` chạy xong không lỗi.

### **9. Gotchas & Lessons Learned**

- **Self-hosted = chicken-and-egg thật, không chỉ lý thuyết**: nếu management cluster tự quản lý chính nó (có `Cluster` object đại diện cho bản thân), và control-plane của nó gặp sự cố nặng (vd mất quorum etcd) ngay giữa lúc đang tự reconcile — không có "management cluster khác" đứng ngoài để cứu nó. Cần chuẩn bị phương án fallback (1 bootstrap cluster dự phòng có thể khởi tạo lại rất nhanh) nếu chọn mô hình này.
- `clusterctl move`/`move --to-directory` **không được thiết kế làm backup/restore tool** — dễ nhầm "đã move ra file rồi, coi như có backup". Thực tế: mất etcd của management cluster không phục hồi được chỉ bằng chạy lại `move`, phải có kỷ luật backup riêng (Velero + etcd snapshot định kỳ) từ đầu.
- Resource limit copy từ default/tutorial dễ gây crash loop khi chạy **nhiều provider cùng lúc** (đã có case thực tế ở mục 6) — luôn theo dõi resource usage thật trong vài ngày đầu sau khi thêm provider mới (vd thêm CAPO sau khi đã chạy CAPC), đừng coi default là đủ.
- Chưa có Grafana dashboard "chính thức" cho CAPI management plane — nếu tìm thấy dashboard template tốt khi triển khai thật, nên lưu lại ở đây.
- (Mục này để tự bổ sung tiếp sau khi thao tác trực tiếp với management cluster thật trên KVM — lesson learned thực chiến quan trọng hơn note lý thuyết)

### **10. Resources**

- `clusterctl move` (nguồn chính thức): https://cluster-api.sigs.k8s.io/clusterctl/commands/move.html
- Tuning controllers (QPS/burst, resource): https://main.cluster-api.sigs.k8s.io/developer/core/tuning
- Metal³ Pivoting guide (cùng khái niệm pivot, áp dụng tương tự ngoài bare-metal): https://book.metal3.io/capm3/pivoting
- Case study resource limit thực tế (Deckhouse PR): https://github.com/deckhouse/deckhouse/pull/23184
- Parent/related: [[CAPI]], [[capi--clusterctl]], [[Fleet-Ops]]
- **Lưu ý**: chưa tìm được tài liệu chính thức nào nêu số liệu sizing CPU/RAM cụ thể cho toàn bộ management cluster theo quy mô fleet — phần này trong note đang dựa trên suy luận + 1 case thực tế lẻ, cần tự benchmark khi triển khai thật.
