# Storage — CSI cho Tenant Cluster
Tags: #k8s-provider #storage #csi #cloudstack #openstack #talos
Related: [[Talos]], [[CAPC]], [[CAPO]], [[Image-Pipeline]]
Last updated: 2026-10-03

> ⚠️ **Version note — CẦN VERIFY LẠI**: Thông tin CSI driver CloudStack trong note này tổng hợp từ search engine (chưa tự clone repo/đọc source), đặc biệt phần "driver nào đang active nhất" — tại thời điểm viết có **nhiều fork cùng tồn tại** (Apalia gốc, Leaseweb, CloudStack-community), chưa xác nhận fork nào nên dùng cho production. Cinder CSI (OpenStack) thì tin cậy hơn vì là project chính thức `kubernetes/cloud-provider-openstack`, nhưng version pin cụ thể vẫn cần verify theo K8s version thật sẽ dùng.

---

### **1. What — Nó là cái gì?**

Nhóm kiến thức về **CSI (Container Storage Interface) driver** cho volume của tenant cluster — lớp dịch `PersistentVolumeClaim` của K8s thành volume thật trên IaaS: **cloudstack-csi-driver** (CloudStack, qua primary storage CloudStack quản lý) và **cinder-csi-plugin** (OpenStack, qua Cinder block storage). Đi kèm là lưu ý bắt buộc khi chạy CSI trên node **Talos** — vì Talos immutable/không package manager, driver cần kernel module như iSCSI/multipath phải được khai báo qua **system extension** (build-time, Image Factory) + machine config, không cài runtime như distro thường.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Nếu không có CSI driver tương ứng, `PersistentVolumeClaim` trong tenant cluster không có cách nào map xuống storage thật của CloudStack/OpenStack — workload cần persistent data (database, registry nội bộ...) phải tự xoay bằng `hostPath` (mất data khi node bị xoá/replace — mâu thuẫn trực tiếp với mô hình "node là cattle" của CAPI+Talos) hoặc tự mount NFS/iSCSI tay (không có lifecycle tự động, không resize/snapshot qua `kubectl`). CSI driver lấp khoảng trống: biến "tạo volume, attach vào đúng VM, resize, snapshot" thành toàn bộ qua K8s API — khớp với toàn bộ triết lý "mọi thứ qua CAPI/K8s API" của stack này, không có lớp thao tác tay nào lọt ra ngoài.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng khi:** tenant cần `StorageClass` dynamic provisioning thật (database, registry, bất kỳ workload cần data sống sót qua việc Machine bị xoá/replace bởi CAPI); cần snapshot/resize volume qua API chuẩn.

**KHÔNG dùng khi:**
- Toàn bộ workload tenant là stateless (web app thuần, worker queue không cần persist local) — không cần CSI, tránh thêm 1 tầng phụ thuộc (node plugin DaemonSet chạy privileged) không cần thiết.
- Hạ tầng CloudStack chưa có primary storage pool phù hợp (vd chỉ có local storage không hỗ trợ attach cross-host) — CSI driver không tự vượt qua giới hạn vật lý của IaaS, cần xác nhận storage backend trước.
- Cần performance/latency storage cực cao mà CSI qua API layer (provisioning call, attach call) có overhead không chấp nhận được — một số workload đặc biệt vẫn cần local NVMe pass-through riêng, nằm ngoài phạm vi CSI chuẩn.

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
Tenant Cluster
┌────────────────────────────────────────────────────────────────────┐
│  PersistentVolumeClaim (do workload tạo)                             │
│         │                                                            │
│         ▼                                                            │
│  CSI Controller (Deployment: external-provisioner, external-attacher,│
│  external-resizer, external-snapshotter sidecar + driver container)  │
│         │  gọi API IaaS để tạo volume thật                           │
│         ▼                                                            │
│  CSI Node Plugin (DaemonSet, privileged, host mount) ── 1 Pod/node    │
│         │  format + mount volume vào đúng node đang chạy Pod         │
└─────────┼──────────────────────────────────────────────────────────┘
          │
          ▼ (API call)
┌─────────────────────────┐         ┌─────────────────────────┐
│  CloudStack               │   hoặc  │  OpenStack Cinder          │
│  (cloudstack-csi-driver)  │         │  (cinder-csi-plugin,       │
│  → primary storage pool   │         │   kubernetes/cloud-        │
│    tạo volume + attach    │         │   provider-openstack)      │
│    vào VM instance         │         │  → block storage volume    │
└─────────────────────────┘         │    tạo + attach qua iSCSI/  │
                                      │    RBD/khác tuỳ backend     │
                                      └─────────────────────────┘
          ▲
          │ nếu backend cần iSCSI/multipath ở tầng OS node
          │
   Talos Node — CẦN system extension build-time (Image Factory):
   `iscsi-tools` (ghcr.io/siderolabs/iscsi-tools) + `multipath-tools`
   → xem [[Image-Pipeline]] cho cách build image kèm extension này
```

### **5. How — Cơ chế hoạt động**

- **CSI Controller** — chạy như Deployment trong tenant cluster, gồm driver container + sidecar chuẩn CSI (`external-provisioner` nghe `PersistentVolumeClaim` mới → gọi `CreateVolume` tới driver; `external-attacher` gọi `ControllerPublishVolume`; `external-resizer`/`external-snapshotter` tương tự cho resize/snapshot). Driver container là phần duy nhất thật sự gọi API CloudStack/Cinder, sidecar đều là code generic CSI dùng chung cho mọi driver.
- **CSI Node Plugin** — DaemonSet, 1 Pod mỗi node, chạy **privileged** (cần quyền mount/format filesystem trực tiếp trên host) — đây là điểm cần chú ý trên Talos: dù OS layer đã giảm attack surface tối đa, Node Plugin vẫn là 1 Pod có quyền gần-root trên node, xem thêm [[Provider-Security]].
- **cloudstack-csi-driver** — dịch `CreateVolume`/`ControllerPublishVolume` thành gọi CloudStack API tạo Data Volume trên primary storage pool + attach vào VM instance tương ứng (map theo `instanceID` CloudStack, không phải tên K8s node). ⚠️ **Cần verify**: project có **nhiều fork** (gốc Apalia SAS → Leaseweb maintain tiếp → bản dưới tổ chức `cloudstack` trên GitHub, tự nhận "community-led, không phải ASF project", mục tiêu mở rộng hỗ trợ đa hypervisor KVM/VMware/XenServer + domains/projects/CKS/CAPC/snapshot) — cần chốt dùng fork nào trước khi lab thật, đừng giả định "CloudStack official docs có trang riêng cho nó" nghĩa là chỉ có 1 bản duy nhất.
- **cinder-csi-plugin** — project chính thức `kubernetes/cloud-provider-openstack`, version release theo kèm K8s minor version (Helm chart `app version` ~ K8s minor, vd chart 2.29.x ↔ K8s 1.29) — mature hơn, ít rủi ro "chọn sai fork" hơn hẳn CloudStack.
- **Talos + iSCSI/multipath** — Cinder (và một số backend CloudStack dùng iSCSI) cần `iscsiadm`/kernel module không có sẵn trên Talos mặc định. Phải thêm **system extension** lúc build image qua Image Factory: `iscsi-tools` (cung cấp `iscsiadm`), `multipath-tools` (đọc `/etc/multipath.conf` trực tiếp từ Talos host, không qua `ExtensionServiceConfig` như version cũ) — chi tiết build image kèm extension → [[Image-Pipeline]].

### **6. Key Config — Cấu hình cần nhớ**

- `StorageClass` tenant cluster phải khai đúng `provisioner` (vd `csi.cloudstack.apache.org` hoặc tên provisioner thật của fork đang dùng / `cinder.csi.openstack.org`) — sai tên provisioner thì PVC stuck `Pending` vô thời hạn, không có lỗi rõ ràng nếu không xem event.
- Node Plugin DaemonSet cần `hostPath`/`privileged` + thường cần chạy trước khi app Pod có thể mount — nếu MachineHealthCheck/autoscaler xoá-tạo node quá nhanh (xem [[Fleet-Ops]]), có thể có khoảng trễ Node Plugin chưa kịp Ready trên node mới, PVC mount fail tạm thời.
- iSCSI trên Talos: biến môi trường driver cần set đúng host strategy (`ISCSIADM_HOST_STRATEGY`/`ISCSIADM_HOST_PATH` theo tài liệu CSI driver dùng chung mẫu `csi-driver-iscsi`/democratic-csi) để driver gọi đúng `iscsiadm` nằm trong extension, không phải trong container — cấu hình sai hướng path này là nguồn lỗi phổ biến khi mới tích hợp Talos + iSCSI.
- Resize/snapshot: phải verify driver (CloudStack fork đang chọn) **có** support `external-resizer`/`external-snapshotter` hay chỉ hỗ trợ `CreateVolume`/`ControllerPublishVolume` cơ bản — không phải fork nào cũng implement đủ toàn bộ CSI spec optional feature.

### **7. Security Considerations**

- Node Plugin DaemonSet chạy privileged trên **mọi** node — compromise 1 Pod app cộng với lỗ hổng escape container có thể chạm tới quyền gần-root của node qua Node Plugin, dù Talos đã giảm attack surface OS-level. Không nên coi "Talos immutable" là lý do bỏ qua việc audit riêng CSI Node Plugin.
- Credential CSI driver gọi API CloudStack/Cinder (API key/secret hoặc `clouds.yaml`) là **credential riêng**, khác với credential CAPC/CAPO ở management cluster — nên scope quyền tối thiểu (chỉ đủ tạo/attach/xoá volume, không cần quyền tạo VM/network) nếu IaaS hỗ trợ tách role chi tiết đến mức đó.
- Volume data không tự mã hoá at-rest chỉ vì đi qua CSI — encryption (nếu cần) phải cấu hình ở tầng storage backend (CloudStack primary storage encryption / Cinder volume encryption), CSI chỉ là lớp provisioning, không phải lớp bảo mật dữ liệu.

### **8. Ops Runbook — Production Notes**

- **Health check**: `kubectl get pods -n <csi-namespace>` (cả Controller Deployment + Node Plugin DaemonSet phải Running trên **mọi** node); `kubectl describe pvc <name>` khi PVC stuck — event log thường chỉ rõ bước nào fail (provision/attach/mount).
- **Log quan trọng**: log của driver container (không phải sidecar) khi debug lỗi gọi API IaaS; trên node Talos dùng `talosctl logs` cho container runtime, không có `/var/log` qua SSH (xem [[Talos]]).
- **Lỗi đã biết trên Talos**: GitHub issue `siderolabs/talos#9134` — "iSCSI system extension not working / reported as uninstalled by applications" — driver phía K8s đôi khi không detect đúng extension đã cài trên Talos, cần verify lại theo version Talos + version extension cụ thể đang dùng trước khi kết luận "extension lỗi" vs "driver đọc path sai".
- **Metric cần alert**: số PVC `Pending` kéo dài, số volume orphan trên IaaS (tạo nhưng K8s object đã xoá — xảy ra nếu Controller crash giữa lúc `DeleteVolume`) — nên có job định kỳ reconcile chéo giữa volume trên IaaS và PV còn tồn tại trong cluster.

### **9. Gotchas & Lessons Learned**

- Đừng chọn fork `cloudstack-csi-driver` chỉ vì nó "được CloudStack docs chính thức dẫn link" — trang docs dẫn tới 1 bản, nhưng cộng đồng đang có nhiều fork song song với mục tiêu khác nhau (1 số hướng tới hỗ trợ CAPC/đa hypervisor) — đọc kỹ CHANGELOG/release gần nhất của fork cụ thể trước khi chốt.
- `multipath-tools` extension đổi cách đọc config giữa các version Talos (từ `ExtensionServiceConfig` sang đọc trực tiếp `/etc/multipath.conf` trên host) — nếu theo tutorial cũ dễ cấu hình sai cách cho version Talos mới.
- (Mục này để tự bổ sung tiếp sau khi thao tác trực tiếp với CSI driver thật trên CloudStack/OpenStack lab — lesson learned thực chiến quan trọng hơn note lý thuyết)

### **10. Resources**

- CloudStack CSI Driver (docs chính thức Apache CloudStack, dẫn tới 1 fork cụ thể): https://docs.cloudstack.apache.org/en/latest/plugins/cloudstack-csi-driver.html
- CloudStack-community fork (đa hypervisor, hướng CAPC): https://github.com/cloudstack/cloudstack-csi-driver
- Leaseweb fork: https://github.com/Leaseweb/cloudstack-csi-driver
- Cinder CSI Plugin (chính thức, kubernetes-sigs): https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/cinder-csi-plugin/using-cinder-csi-plugin.md
- Cinder CSI features matrix: https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/cinder-csi-plugin/features.md
- Talos CSI/Storage guide: https://docs.siderolabs.com/kubernetes-guides/csi/storage
- Talos System Extensions (iscsi-tools, multipath-tools): https://github.com/siderolabs/extensions
- Talos iSCSI extension issue đã biết: https://github.com/siderolabs/talos/issues/9134
- **Lưu ý**: fork CloudStack CSI nào là "chuẩn" cho production chưa được xác nhận trực tiếp từ nguồn chính thống duy nhất — verify lại khi chọn trước khi dùng cho hệ thống thật.
