# Cluster API Provider OpenStack (CAPO) — v0.15.x
Tags: #k8s-provider #capo #openstack #infra-provider
Related: [[CAPI]], [[CAPC]], [[Talos]], [[Networking]], [[Storage]], [[Provider-Security]]
Last updated: 2026-10-03

> ⚠️ **Version note — CẦN VERIFY LẠI**: Bản mới nhất tìm được qua search là **v0.15.0** (17/09/2026) — release này có ~184 commit, **12 breaking change**, và đưa thêm API `v1beta2` chạy **song song** `v1beta1` cũ. Số liệu lấy qua aggregator bên thứ 3 (newreleases.io), chưa đối chiếu trực tiếp CHANGELOG gốc. Verify lại bằng https://github.com/kubernetes-sigs/cluster-api-provider-openstack/releases trước khi dùng để quyết định production.
>
> Khác với CAPC, CAPO nằm **trực tiếp trong org `kubernetes-sigs`** (không phải repo community độc lập) — tín hiệu tốt về mức độ "chính thức", nhưng **không đồng nghĩa luôn release đồng bộ với CAPI core**, vẫn phải tự tra compatibility matrix (xem mục 9).

---

### **1. What — Nó là cái gì?**

CAPO (Cluster API Provider OpenStack) là Infrastructure Provider của CAPI cho OpenStack — dịch các object trừu tượng (`Cluster`, `Machine`) thành resource thật trên OpenStack: Nova instance, Neutron network/port/security group, Glance image, Octavia load balancer. Trong stack k8s-provider, CAPO là lựa chọn "infra provider" khi hạ tầng KVM được quản trị bằng **OpenStack** thay vì CloudStack → xem [[CAPC]] cho lựa chọn tương đương bên CloudStack, và [[CAPI]] cho vai trò "lõi điều phối" chung.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Nếu không có CAPO, để tạo VM trên OpenStack theo mô hình CAPI phải tự viết actuator/infra provider riêng — phá vỡ hoàn toàn lý do ban đầu dùng CAPI (chuẩn hoá vòng đời cluster qua 1 K8s API chung). Terraform + OpenStack provider tạo VM tốt, nhưng không có khái niệm "K8s Machine" tự remediate hay rolling-upgrade qua `MachineDeployment`. CAPO lấp khoảng trống: expose đúng CRD theo provider contract của CAPI, map 1-1 khái niệm Nova/Neutron/Octavia vào field CRD. Quan trọng hơn với lựa chọn "OpenStack cho KaaS": OpenStack đã có sẵn mô hình multi-tenant native ở tầng IaaS (Keystone Project/Domain) — CAPO cho phép tận dụng trực tiếp mô hình này (mỗi tenant = 1 Project + 1 credential riêng) thay vì phải tự dựng tenant isolation từ đầu.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng khi:** hạ tầng KVM đã chạy OpenStack (Nova/Neutron/Keystone/Glance, có Octavia nếu cần HA control-plane endpoint tự động); muốn multi-tenancy ở tầng IaaS ánh xạ trực tiếp theo OpenStack Project, kết hợp với tầng CAPI/ClusterClass ở trên.

**KHÔNG dùng khi:**
- OpenStack đang chạy **chưa có Octavia** và không có kế hoạch triển khai — vẫn tạo cluster được (`apiServerLoadBalancer.enabled: false`, dùng floating IP + kube-vip) nhưng mất HA endpoint tự động quản lý bởi CAPO, cần đánh giá kỹ trade-off.
- Cần feature Neutron/Nova rất mới hoặc plugin đặc thù chưa phổ biến — luôn kiểm tra CAPO/CCM đã support chưa trước khi cam kết.
- Team chưa quen vận hành OpenStack (Keystone domain/project, Neutron network model, Octavia) — learning curve này tách biệt hoàn toàn khỏi phần CAPI/Talos đã học.

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
┌────────────────────────── Management Cluster ──────────────────────────┐
│  Cluster + OpenStackCluster + OpenStackMachineTemplate (CRD của CAPO)    │
│                     │                                                   │
│                     ▼                                                   │
│   CAPO Controller (namespace: capo-system)                              │
│     đọc Secret qua identityRef → clouds.yaml (OpenStack credential)     │
│     gọi OpenStack API: Nova / Neutron / Glance / Octavia / Cinder       │
└───────────────┬──────────────────────────────────────────────────────┘
                │
   ┌────────────┼──────────────────────────────────────┬────────────────┐
   ▼            ▼                                        ▼                │
 Nova         Neutron                                  Octavia            │
 (instance,   (network, subnet, port,                  (LB cho control-   │
  flavor,      security group, router,                 plane endpoint —  │
  dùng image   floating IP)                             amphora/ovn,      │
  từ Glance)                                             apiServerLoadBal-│
   │                                                      ancer)          │
   ▼                                                                      │
 Tenant / Workload Cluster (Talos + Kubernetes trên VM do Nova quản lý)   │
   │                                                                      │
   ├─ openstack-cloud-controller-manager (CCM) → Service type=LoadBalancer, dùng Octavia
   └─ cinder-csi-plugin (CSI) → PersistentVolume, dùng Cinder            → [[Storage]]
```

Image Talos dùng trong `OpenStackMachineTemplate.spec.image` phải được publish sẵn vào Glance **trước** khi CAPO tạo Machine — bước build/publish này nằm ở [[Image-Pipeline]], ngoài phạm vi CAPO.

### **5. How — Cơ chế hoạt động**

- **`OpenStackCluster`** — InfrastructureCluster: chứa network config (`network`, `externalNetwork`/router), `identityRef` (credential), và `apiServerLoadBalancer` config.
- **`OpenStackMachine` / `OpenStackMachineTemplate`** — InfrastructureMachine: map tới Nova `flavor`, `image` (theo tên/ID Glance — image Talos từ [[Image-Pipeline]]), `ports` (Neutron port spec, security group gán kèm).
- **`identityRef` (`OpenStackIdentityReference`)** — 2 kiểu: `Secret` (trỏ trực tiếp 1 Secret chứa `clouds.yaml`, cùng namespace với `OpenStackCluster`) hoặc `ClusterIdentity` (trỏ tới `OpenStackClusterIdentity` — resource cluster-scoped, cho phép nhiều `OpenStackCluster` **chia sẻ** 1 credential mà không copy Secret). `cloudName` chọn entry nào trong `clouds.yaml` (1 file có thể chứa nhiều cloud/project). Từ `v1beta1`, API server CAPO **bắt buộc** `OpenStackCluster` phải có `identityRef` — không còn optional như bản cũ.
- **`apiServerLoadBalancer`** — nếu `enabled: true` + `managed: true`, CAPO tự tạo Octavia LB (chọn provider `amphora` hoặc `ovn`), listener port 6443, health monitor, member pool gồm toàn bộ control-plane node, hỗ trợ `allowedCIDRs` giới hạn ai gọi được API server. Không dùng Octavia thì phải tự lo phương án HA endpoint khác (floating IP cố định + kube-vip, hoặc LB ngoài CAPO quản lý).
- **2 add-on chạy TRONG tenant cluster, KHÔNG phải do CAPO cài** — dễ nhầm là cùng 1 thứ: `openstack-cloud-controller-manager` (CCM, cho `Service type=LoadBalancer` của workload dùng Octavia) và `cinder-csi-plugin` (CSI, PersistentVolume dùng Cinder). CAPO chỉ dựng hạ tầng ban đầu (VM + network + LB control-plane); 2 add-on này phải cài riêng sau khi cluster up → [[Networking]], [[Storage]].

### **6. Key Config — Cấu hình cần nhớ**

- `clouds.yaml` Secret phải đúng format OpenStack chuẩn (`clouds.<cloudName>.auth...`), và **key trong Secret phải đặt đúng tên `clouds.yaml`** (chữ thường) — sai tên key là lỗi phổ biến nhất lúc mới setup `identityRef` kiểu `Secret`.
- Nên dùng **Application Credential** (Keystone) cho `clouds.yaml` của CAPO, thay vì user/password tài khoản gốc — least-privilege hơn, revoke độc lập được mà không ảnh hưởng account chính. ⚠️ **Cần verify**: CAPO v0.15.x có hỗ trợ đầy đủ auth type này tuỳ bản gophercloud lib mà CAPO đang dùng, chưa test thực tế.
- `cloudName` **đã đổi vị trí qua các version**: từ nằm trực tiếp ở `OpenStackMachine`/`OpenStackCluster` → chuyển vào trong `identityRef`. Copy YAML mẫu cũ (trước `v1beta1`, vd từ blog/tutorial) dễ apply fail vì field không còn tồn tại ở vị trí cũ.
- `externalNetwork`/network filter — nếu project OpenStack có nhiều external network hoặc network trùng tên, phải filter rõ bằng **ID** (không chỉ tên) để tránh CAPO chọn sai network khi tạo port/floating IP.
- Octavia `provider` (`amphora` vs `ovn`) ảnh hưởng performance + feature của LB — `amphora` (VM-based) tốn resource riêng mỗi LB, `ovn` (nếu OpenStack đã chạy OVN) nhẹ hơn nhưng ⚠️ **cần verify** tính năng có tương đương `amphora` hay thiếu field nào (vd health monitor) tuỳ bản Octavia đang chạy thật.

### **7. Security Considerations**

- `clouds.yaml` lưu trong Secret ở management cluster (namespace chứa `OpenStackCluster`/`capo-system`) — đây là credential OpenStack thật (Keystone), kế thừa toàn bộ rủi ro đã nêu ở [[CAPI]] mục 7 (compromise management cluster = lộ credential IaaS). Nên tạo riêng 1 OpenStack Project + Application Credential **chỉ đủ quyền** tạo/xoá instance+network+LB trong scope cần, **không** dùng credential admin domain cho CAPO.
- Không tìm thấy security advisory công khai riêng cho CAPO tại thời điểm research (⚠️ **cần verify trực tiếp** `github.com/kubernetes-sigs/cluster-api-provider-openstack/security/advisories`). Nhưng OpenStack platform (nền mà CAPO gọi vào) có CVE ảnh hưởng gián tiếp — ví dụ `CVE-2026-43001` (Keystone thiếu validate `project_id` khi tạo EC2-type credential, cho phép tạo credential nhắm sang project khác → lateral movement cross-project). Nhắc rằng mô hình multi-tenancy của OpenStack chỉ an toàn nếu Keystone **đã patch đầy đủ**, không nên coi "OpenStack có multi-tenancy sẵn" là tự động an toàn.
- `OpenStackClusterIdentity` (chia sẻ 1 credential cho nhiều `OpenStackCluster`) tiện nhưng tăng blast radius: 1 credential rò = ảnh hưởng mọi cluster dùng chung identity đó. Nếu multi-tenancy thiết kế mỗi tenant = 1 Cluster riêng, nên ưu tiên **mỗi tenant 1 OpenStack Project + 1 credential riêng** thay vì share 1 `OpenStackClusterIdentity` cho tất cả, để giữ đúng boundary tenant tới tận tầng IaaS → xem thêm [[Provider-Security]].
- `allowedCIDRs` trên `apiServerLoadBalancer` nên luôn set rõ (không để trống/mở `0.0.0.0/0`) — ⚠️ **cần verify** hành vi default chính xác khi field này bỏ trống ở version đang dùng.

### **8. Ops Runbook — Production Notes**

- **Health check**: `kubectl get openstackcluster,openstackmachine -A`; log controller ở namespace `capo-system` khi `Machine` stuck ở `Provisioning` — thường do lỗi gọi Nova/Neutron (quota hết, network/flavor sai ID, Application Credential hết hạn).
- **Debug chéo OpenStack-side**: `openstack port list`, `openstack loadbalancer show <id>` (khi dùng Octavia) để đối chiếu trực tiếp với trạng thái `OpenStackCluster` — lỗi nhiều khi nằm ở phía Neutron/Octavia, không phải logic CAPO.
- Octavia LB tạo bởi CAPO (`managed: true`) sẽ **bị xoá theo khi xoá `Cluster`** — nếu cần giữ lại LB/floating IP cố định (vd đã trỏ DNS cho khách), nên đánh giá dùng `managed: false` + tự quản LB riêng. ⚠️ **Cần verify** chính xác hành vi cleanup ở version đang dùng trước khi dựa vào.
- Upgrade CAPO theo `clusterctl upgrade plan`/`apply` như mọi provider khác (xem [[capi--clusterctl]]) — nhưng do v0.15.x vừa đổi sang chạy song song `v1beta1` + `v1beta2`, **cần verify kỹ compatibility matrix với CAPI core** trước khi upgrade management cluster đang chạy thật, không upgrade theo "latest" mù.

### **9. Gotchas & Lessons Learned**

- CAPO nằm trong org `kubernetes-sigs` (vị trí "chính thức" hơn CAPC), nhưng **không đồng nghĩa luôn đồng bộ version với CAPI core** — v0.15.0 vẫn có 12 breaking change riêng của chính nó, vẫn phải tự tra compatibility, đừng suy luận "chính thức thì chắc tương thích luôn".
- `cloudName`/`identityRef` đổi vị trí field qua các version (`v1alpha7` → `v1beta1` → `v1beta2` đang song song) — copy YAML mẫu từ tutorial/blog cũ mà không để ý version annotation dễ bị webhook validation reject ngay khi apply.
- Octavia **bắt buộc thay cho Neutron-LBaaS** (đã deprecated từ lâu phía OpenStack, từ cycle Queens) — nếu hạ tầng OpenStack đang dùng còn cũ, chưa có Octavia, phải triển khai Octavia trước khi dùng `apiServerLoadBalancer` hoặc CCM `Service type=LoadBalancer`, không có đường tắt nào khác.
- (Mục này để tự bổ sung tiếp sau khi thao tác trực tiếp với OpenStack thật — lesson learned thực chiến quan trọng hơn note lý thuyết)

### **10. Resources**

- Official book: https://cluster-api-openstack.sigs.k8s.io/
- GitHub repo & releases: https://github.com/kubernetes-sigs/cluster-api-provider-openstack/releases
- `OpenStackClusterIdentity`: https://cluster-api-openstack.sigs.k8s.io/topics/openstack-cluster-identity
- Migration v1alpha7 → v1beta1: https://cluster-api-openstack.sigs.k8s.io/topics/crd-changes/v1alpha7-to-v1beta1
- openstack-cloud-controller-manager: https://github.com/kubernetes/cloud-provider-openstack
- Cinder CSI plugin: https://openstack-cloud-controller-manager.readthedocs.io/en/latest/using-cinder-csi-plugin/
- **Lưu ý khi dùng note này**: version v0.15.0 (17/09/2026) và chi tiết breaking-change lấy qua aggregator bên thứ 3 (newreleases.io), chưa đối chiếu trực tiếp GitHub Releases/CHANGELOG gốc. Không tìm thấy security advisory riêng cho CAPO tại thời điểm research — verify trực tiếp trang `security/advisories` của repo trước khi kết luận "chưa từng có CVE".
