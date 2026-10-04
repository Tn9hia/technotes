# Image Pipeline — Build & Publish Node Image
Tags: #k8s-provider #image-pipeline #talos #image-factory #cloudstack #openstack
Related: [[Talos]], [[CAPC]], [[CAPO]], [[Fleet-Ops]], [[Provider-Security]]
Last updated: 2026-10-03

> ⚠️ **Cần verify lại**: Theo Talos Support Matrix, **CloudStack nằm ở Tier 3** (hỗ trợ nhưng ít test, dùng chung platform `nocloud` generic — Talos **không có** platform tên riêng "cloudstack"), còn **OpenStack ở Tier 2** (có platform `openstack` dedicated, metadata service native). Tier assignment đổi theo từng version Talos — verify lại tại `docs.siderolabs.com/talos/<version>/getting-started/support-matrix` trước khi cam kết SLA dựa trên tier này.

---

### **1. What — Nó là cái gì?**

Nhóm kiến thức về **build-time** (dùng Talos Image Factory để tạo image theo "schematic") và **distribution** (đăng ký image đó thành CloudStack Template / OpenStack Glance image) — tức là mọi việc xảy ra **trước khi** 1 VM node được tạo ra. Phạm vi note này **dừng lại** ở lúc image đã nằm sẵn trong CloudStack/OpenStack, chờ CAPC/CAPO tham chiếu tới; machine config runtime và partition layout bên trong node đã boot thuộc về [[Talos]], không lặp lại ở đây.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Đây **không phải** 1 lớp tiện ích tuỳ chọn như với distro thường — nó gần như **bắt buộc** vì bản chất immutable của Talos: Talos không cho cài package/extension ở runtime (không SSH, không package manager), nên mọi customization (driver, agent, extension) phải được "nướng" sẵn vào image ngay từ build-time qua Image Factory. Nếu không có pipeline này:
- Mỗi lần cần image mới (bump Talos version, thêm/đổi extension) phải tự tay build + upload lại — dễ version-drift giữa các "flavor" ClusterClass khác nhau, không ai chắc tenant A và tenant B đang chạy đúng cùng 1 image hay không.
- Không reproducible: build tay dễ quên 1 bước (convert format, set đúng OS type) → image lỗi phát hiện muộn, lúc node đã join cluster.

Pipeline hoá bước này (schematic lưu trong git + script build/publish) biến "có image Talos mới cho KVM/CloudStack/OpenStack" thành 1 quy trình lặp lại được, không phụ thuộc trí nhớ người vận hành.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng khi:** cần pin chính xác version Talos + system extension cho từng "flavor" cluster bán ra; cần build image trong môi trường airgap/không internet (self-host Image Factory offline); cần audit được "image nào đang chạy ở đâu" trên toàn fleet.

**KHÔNG cần tự dựng pipeline phức tạp khi:** lab/dev nhỏ, chấp nhận phụ thuộc internet và tải trực tiếp từ `factory.talos.dev` mỗi lần cần image mới — vẫn phải làm tối thiểu bước "đăng ký template/image" lên CloudStack/OpenStack (không skip được bước này dù dùng Image Factory public), nhưng không cần tự động hoá toàn bộ ngay từ đầu.

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
Talos Image Factory (factory.talos.dev, hoặc self-host offline)
 │  input: schematic.yaml (customization.systemExtensions.officialExtensions:
 │          siderolabs/qemu-guest-agent, ...) + platform + Talos version
 ▼
Schematic ID (hash content-addressable — cùng input luôn ra cùng ID)
 │
 ▼
Output image: ZSTD-compressed RAW disk, theo platform:
 ├─ platform=nocloud   → dùng cho CloudStack (không có platform riêng, Tier 3)
 └─ platform=openstack → dùng cho OpenStack (Tier 2, metadata service native)
      │
      ▼ (OpenStack cần convert thêm)
   qemu-img convert -f raw -O qcow2 openstack-amd64.raw talos-<ver>.qcow2
      │
      ▼
┌────────────────────────────┐        ┌───────────────────────────┐
│ CloudStack                  │        │ OpenStack                  │
│ registerTemplate            │        │ openstack image create     │
│  hypervisor=KVM              │        │  --disk-format qcow2        │
│  format=QCOW2|RAW            │        │  --container-format bare    │
│  ostypeid=<linux-generic>    │        │                             │
└──────────────┬───────────────┘        └──────────────┬──────────────┘
               ▼                                        ▼
 CloudStackMachineTemplate.spec                OpenStackMachineTemplate.spec
   .template.spec.template: <tên-template>       .template.spec.image.filter.name / .id
               │                                        │
               └────────────────────┬───────────────────┘
                                     ▼
                     CAPC/CAPO tạo VM từ image khi Cluster reconcile
                     → [[CAPC]] / [[CAPO]]
```

### **5. How — Cơ chế hoạt động**

- **Schematic** — YAML khai báo customization, chủ yếu `customization.systemExtensions.officialExtensions`. Với KVM guest, extension tối thiểu nên có là `siderolabs/qemu-guest-agent` (để hypervisor lấy IP qua agent, graceful shutdown thay vì ACPI thô). ⚠️ **Cần verify** có cần thêm extension nào khác cho networking/virtio trên môi trường lab cụ thể — virtio driver thường build sẵn trong kernel Talos nên nhiều khả năng không cần extension riêng, nhưng chưa xác nhận 100%.
- **Schematic ID** — hash content-addressable của chính nội dung schematic: build lại từ cùng schematic luôn ra cùng ID. Nên dùng ID này (không chỉ Talos version) để pin chính xác "đây là image nào" trong versioning — 2 schematic khác nhau ở cùng 1 Talos version vẫn là 2 image hoàn toàn khác.
- **Platform** — Talos **không có** platform tên "cloudstack" riêng; theo support matrix, CloudStack ở **Tier 3** (build qua platform `nocloud` generic, cloud-init/ConfigDrive), OpenStack ở **Tier 2** (platform `openstack` dedicated, có metadata service native tốt hơn). Khác biệt tier này ảnh hưởng tới độ "mượt" của tích hợp, không chỉ là nhãn hỗ trợ.
- **Output format** — cả 2 platform xuất ảnh **ZSTD-compressed RAW disk**. CloudStack (KVM) nhận được RAW hoặc QCOW2; OpenStack/Glance thường cần convert RAW → QCOW2 qua `qemu-img convert` trước khi upload.
- **Đăng ký image**: CloudStack qua API `registerTemplate` (tham số bắt buộc `hypervisor=KVM`, `format=QCOW2|RAW`, `ostypeid`); OpenStack qua `openstack image create --disk-format qcow2 --container-format bare`.
- **Reference từ CAPI provider** — CAPC dùng field `CloudStackMachineTemplate.spec.template.spec.template: <tên-template>` (khớp tên đã register); CAPO dùng `OpenStackMachineTemplate.spec.template.spec.image.filter.name` hoặc `.image.id` (khớp tên/UUID image trên Glance).
- **Installer image** (cho upgrade in-place, khác boot image lúc tạo mới) — dạng `factory.talos.dev/nocloud-installer/<schematic-id>:<version>`, dùng làm reference khi `talosctl upgrade`. Nghĩa là build/publish không chỉ chạy 1 lần lúc tạo cluster, mà lặp lại mỗi khi bump Talos version cho fleet đang chạy → liên hệ [[Fleet-Ops]].

### **6. Key Config — Cấu hình cần nhớ**

- `version:` Talos trong schematic **phải khớp** version khai báo ở `TalosControlPlane`/`TalosConfigTemplate` (CAPI) — lệch version giữa image đã publish và machine config gây lỗi join hoặc hành vi không xác định; **không có validation tự động** nào ở tầng CAPC/CAPO ngăn việc này.
- `ostypeid` sai khi register template trên CloudStack có thể khiến CloudStack áp sai driver/NIC mặc định dự kiến cho OS đó. ⚠️ **Cần verify** giá trị ostype phù hợp nhất cho Talos trên CloudStack — chưa tìm thấy khuyến nghị chính thức từ Sidero Labs cho tham số này, CloudStack cũng không có "Talos" sẵn trong danh sách OS type chuẩn.
- RAW vs QCOW2: RAW không hỗ trợ thin-provisioning/snapshot hiệu quả như QCOW2 trên một số storage backend của CloudStack (NFS/Ceph) — nên test thật trên storage pool cụ thể trước khi chọn format mặc định cho toàn fleet.
- Đặt tên template/image nên gồm **cả Talos version và schematic ID** (vd `talos-1.14.2-<schematicID-rút-gọn>`), không chỉ theo version — tránh nhầm giữa 2 image khác extension nhưng cùng version Talos.
- Không có cơ chế tự động dọn template/image cũ không còn `MachineTemplate` nào reference — phải tự script kiểm tra trước khi xoá, tránh xoá nhầm image đang được Cluster nào đó dùng.

### **7. Security Considerations**

- Image Factory public (`factory.talos.dev`) là dependency ngoài mạng — môi trường cần airgap/kiểm soát chặt nên self-host Image Factory bản offline. ⚠️ **Cần verify** chi tiết cách self-host tại thời điểm triển khai thật, chưa tra kỹ quy trình này.
- `qemu-guest-agent` extension mở 1 channel (virtio-serial) giữa KVM host và guest — về bản chất là bề mặt tin cậy 2 chiều. Trong môi trường multi-tenant, phải đảm bảo tầng KVM/CloudStack/OpenStack không để tenant khác lợi dụng channel này đọc chéo thông tin VM khác — đây là hardening ở tầng hypervisor, không phải tầng Talos. ⚠️ **Cần verify** isolation thực tế của channel này trong setup multi-tenant cụ thể.
- Credential dùng để `registerTemplate` (CloudStack) hoặc `openstack image create` (Glance) thường cần quyền admin-level trên IaaS — nên **tách biệt** khỏi credential runtime mà CAPC/CAPO giữ thường trực trong management cluster để tạo VM hàng ngày. Credential "build/publish image" chỉ cần cấp cho pipeline CI, giảm blast radius nếu Secret của CAPC/CAPO bị lộ → [[Provider-Security]].
- Image public (nếu để `is_public`/không giới hạn project-scope khi đăng ký) có thể bị tenant khác tự tạo VM từ image "nội bộ" ngoài ý muốn — luôn set đúng quyền truy cập khi publish.

### **8. Ops Runbook — Production Notes**

- **Health check pipeline**: verify schematic build ra đúng ID kỳ vọng (gọi lại Image Factory API với cùng schematic, so sánh ID) — đổi 1 ký tự trong schematic sẽ đổi hoàn toàn ID, dùng cách này để detect config drift của chính pipeline.
- **Log quan trọng**: log job CI build/publish (nếu tự dựng) — lỗi hay gặp ở bước `qemu-img convert` (hết dung lượng tạm) hoặc bước gọi API register template/Glance (quota/permission sai).
- **Rollout version mới**: build image mới → publish **song song** với image cũ (không xoá ngay) → cập nhật `ClusterClass`/`MachineTemplate` cho 1 flavor test trước → rollout dần qua cơ chế rolling của MachineDeployment (→ [[capi--clusterclass-topology]]) → chỉ xoá image cũ sau khi xác nhận không còn `MachineTemplate` nào reference.
- **Rollback**: giữ tối thiểu N version image gần nhất (N tuỳ dung lượng storage cho phép) để rollback `MachineTemplate` về image cũ nếu version mới phát hiện lỗi sau rollout.

### **9. Gotchas & Lessons Learned**

- Talos **không có platform "cloudstack" chính thức** — build chung qua `nocloud` (Tier 3). Đừng nhầm "CloudStack được liệt kê hỗ trợ" với "có integration sâu native" như OpenStack (Tier 2) — một số hành vi phụ thuộc platform-specific metadata có thể không đầy đủ trên CloudStack.
- Quên convert RAW → QCOW2 khi upload lên Glance là lỗi hay gặp — một số driver Glance "nhận" file RAW đặt tên `.qcow2` mà không báo lỗi ngay lúc upload, chỉ lộ ra khi boot fail/hành vi lạ về sau. ⚠️ **Cần verify** hành vi chính xác của driver Glance đang dùng.
- Schematic ID đổi bất cứ khi nào sửa schematic — nên lưu file schematic gốc trong git, đừng chỉ nhớ/ghi ID, vì ID không tự giải thích được nội dung gốc đã tạo ra nó.
- (Mục này để tự bổ sung tiếp sau khi build/publish image thật trên lab KVM + CloudStack/OpenStack — lesson learned thực chiến quan trọng hơn note lý thuyết)

### **10. Resources**

- Image Factory (official docs): https://docs.siderolabs.com/talos/v1.14/learn-more/image-factory
- Image Factory web UI: https://factory.talos.dev/
- Talos Support Matrix (platform tier — verify theo version đang dùng): https://docs.siderolabs.com/talos/v1.12/getting-started/support-matrix
- CAPC Custom Images: https://cluster-api-cloudstack.sigs.k8s.io/topics/custom-images
- CAPC Configuration (field `template`): https://cluster-api-cloudstack.sigs.k8s.io/clustercloudstack/configuration
- CAPO API reference (field `image`): https://cluster-api-openstack.sigs.k8s.io/api/v1beta1/api
- Apache CloudStack `registerTemplate` API: https://cloudstack.apache.org/api/apidocs-4.11/apis/registerTemplate.html
- **Lưu ý**: tên field CRD cụ thể (`template`, `image.filter.name`) lấy qua search engine, có thể không khớp 100% version CAPC/CAPO đang dùng thật — verify lại bằng `kubectl explain cloudstackmachinetemplate.spec.template.spec` / `kubectl explain openstackmachinetemplate.spec.template.spec` trên cluster thật.
