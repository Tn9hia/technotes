# Networking — Tenant Cluster Networking (CNI + CCM + LB)
Tags: #k8s-provider #networking #cni #ccm #loadbalancer
Related: [[CAPC]], [[CAPO]], [[Talos]], [[Multi-Tenancy]], [[Storage]]
Last updated: 2026-10-03

> ⚠️ **Lưu ý tên project — dễ nhầm**: CCM cho CloudStack **không** tên là "cloud-provider-cloudstack". Project cũ `swisstxt/cloudstack-cloud-controller-manager` đã **ngừng maintain**, chính README của nó khuyến nghị chuyển sang `apache/cloudstack-kubernetes-provider` (bản chính thức, do Apache CloudStack maintain, release gần nhất v1.2.0 — **cần verify ngày release** khi dùng thật). Đừng clone nhầm bản cũ từ tutorial/blog cũ.
>
> CCM cho OpenStack thì rõ ràng hơn: `kubernetes/cloud-provider-openstack` — project chính thức dưới org `kubernetes`, không phải fork cộng đồng, mức độ maintain tốt hơn hẳn phía CloudStack.

---

### **1. What — Nó là cái gì?**

Nhóm kiến thức này bàn 2 việc: **(a) CNI** — networking *trong* 1 tenant cluster (Pod-to-Pod, Service ClusterIP, NetworkPolicy), và **(b) CCM + LB/control-plane endpoint** — networking *từ* cluster ra ngoài, tức cách Service `type=LoadBalancer`/Ingress và API server của cluster map xuống hạ tầng CloudStack/OpenStack thật (VM, security group, load balancer). **Không** bàn network isolation **giữa** các tenant khác nhau (vd mỗi khách 1 VPC/project riêng) — đó thuộc [[Multi-Tenancy]].

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Không có CNI, Pod trên node khác nhau không nói chuyện được — K8s tự nó không implement networking, chỉ định nghĩa interface (CNI spec) và giao cho plugin. Không có CCM, `Service type=LoadBalancer` sẽ **mãi `Pending`** — K8s core không biết cách gọi API của CloudStack/OpenStack để tạo LB thật, phải tự tạo tay mỗi lần có Service mới (không scale được cho mô hình bán cluster tự động). Không có giải pháp HA cho control-plane endpoint, API server của tenant cluster chỉ trỏ vào 1 IP control-plane node duy nhất — node đó chết là toàn cluster mất quyền quản trị (kể cả khi 2 control-plane khác vẫn sống), dù etcd vẫn còn quorum.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng khi:** cần tenant cluster tự expose Service ra ngoài qua LB thật (không phải NodePort tay); cần control-plane HA thật sự (sống sót khi mất 1 control-plane node); muốn CNI tối ưu cho Talos (immutable, không cài thêm binary ngoài container).

**KHÔNG dùng khi:**
- Lab/dev nhỏ, chấp nhận NodePort hoặc port-forward — cài CCM + LB thật cho 1 cluster thử nghiệm là overkill.
- Hạ tầng CloudStack/OpenStack chưa có LBaaS (Octavia chưa deploy, hoặc CloudStack chưa cấu hình Network Offering có LB) — CCM không tạo ra LB từ không khí, phải có LB service thật ở IaaS trước. **Cần verify** Octavia/CloudStack LB Network Offering đã sẵn sàng trước khi thiết kế phần này.

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
┌──────────────────────────── Tenant Cluster (Talos + K8s) ────────────────────────────┐
│                                                                                         │
│  ┌─────────────┐   Pod-to-Pod, Service ClusterIP, NetworkPolicy                        │
│  │  CNI (Cilium) │  eBPF, thường thay thế kube-proxy (xem mục 5/6)                      │
│  └─────────────┘                                                                       │
│                                                                                         │
│  kubelet mỗi node ──▶ KubePrism (localhost:7445, tính năng riêng của Talos)            │
│     dùng nội bộ để gọi API server mà KHÔNG cần qua LB ngoài                             │
│                                                                                         │
│  ┌───────────────────────────┐        ┌─────────────────────────────┐                 │
│  │ Cloud Controller Manager   │        │ kube-vip (static pod trên    │                 │
│  │ (CCM) — apache/cloudstack- │        │ mỗi control-plane node)      │                 │
│  │ kubernetes-provider (CAPC) │        │  HOẶC                        │                 │
│  │ hoặc cloud-provider-       │        │ LB có sẵn của IaaS trỏ thẳng  │                 │
│  │ openstack (CAPO)           │        │ vào IP control-plane node    │                 │
│  └─────────────┬───────────────┘        └──────────────┬────────────┘                 │
│                │ watch Service type=LB, Node            │ VIP/LB cho :6443             │
└────────────────┼─────────────────────────────────────────┼───────────────────────────┘
                 ▼                                         ▼
     ┌─────────────────────────────┐          ┌──────────────────────────┐
     │  CloudStack LB / Octavia      │          │  CloudStack LB / Octavia /  │
     │  (LBaaS tạo động theo Service)│          │  kube-vip VIP (L2 ARP/BGP)  │
     └─────────────────────────────┘          └──────────────────────────┘
```

### **5. How — Cơ chế hoạt động**

- **CNI trên Talos — Cilium là lựa chọn phổ biến nhất**: Talos docs chính thức có hướng dẫn riêng deploy Cilium, khuyến nghị dùng Helm chart chính thức, bật **kube-proxy replacement** (eBPF chặn traffic Service ngay trong kernel, không sinh iptables rule — nhanh hơn, có Service-level visibility qua Hubble). Cấu hình Talos cần đổi 2 field: `cluster.network.cni.name: none` (tắt CNI mặc định flannel) và `cluster.proxy.disabled: true` (tắt kube-proxy, để Cilium thay thế hoàn toàn).
- **KubePrism (tính năng riêng của Talos, không phải K8s chuẩn)** — mỗi node tự expose `localhost:7445`, cho phép kubelet/component nội bộ gọi API server **không cần qua LB ngoài**, tự failover giữa các control-plane. Giảm phụ thuộc vào control-plane LB cho traffic **nội bộ cluster**; LB/kube-vip vẫn cần cho traffic **từ ngoài vào** (`kubectl` từ máy quản trị, hoặc CABPT/CACPPT gọi vào từ management cluster).
- **CCM (Cloud Controller Manager)** — controller watch `Service type=LoadBalancer` + `Node`, gọi API IaaS để tạo/xoá LB thật và gắn label/taint node theo region/zone. Cho CloudStack: `apache/cloudstack-kubernetes-provider` (xem banner đầu file, đừng nhầm bản cũ). Cho OpenStack: `kubernetes/cloud-provider-openstack` (project chính thức SIG, tích hợp Octavia — LBaaS v2 là implementation mặc định cho `Service type=LoadBalancer`, có thêm `octavia-ingress-controller` riêng để dồn nhiều Ingress vào 1 LB, tiết kiệm chi phí LB hơn so với 1 LB/Service).
- **Control-plane endpoint HA** — 2 hướng: **kube-vip** chạy như **static pod** (quản lý trực tiếp bởi kubelet qua machine config, không phải DaemonSet) trên từng control-plane node, tự bầu leader + advertise VIP qua ARP (L2) hoặc BGP; phổ biến trong cộng đồng Talos vì tích hợp sạch với machine config. Hoặc **dùng LB có sẵn của IaaS** (CloudStack LB / Octavia) trỏ thẳng vào IP các control-plane node trên port 6443 — đơn giản hơn vận hành (không thêm 1 "ứng dụng" chạy trong cluster) nhưng phụ thuộc LBaaS của IaaS phải sẵn sàng từ **trước khi cluster có control-plane đầu tiên** (vấn đề con-gà-quả-trứng nếu LB cũng do CCM trong cluster đó tạo).

### **6. Key Config — Cấu hình cần nhớ**

- Dùng Cilium thay kube-proxy trên Talos: phải set cả 2 field `cluster.network.cni.name: none` **và** `cluster.proxy.disabled: true` trong machine config — thiếu 1 trong 2 dễ gây xung đột (2 CNI/2 lớp proxy cùng chạy).
- `octavia-ingress-controller` (OpenStack) dùng annotation riêng như `loadbalancer.openstack.org/lb-method` (giá trị `ROUND_ROBIN`/`LEAST_CONNECTIONS`/`SOURCE_IP`/`SOURCE_IP_PORT`) — không giống annotation chuẩn K8s, phải tra đúng docs của `cloud-provider-openstack`, không áp annotation của cloud provider khác.
- kube-vip static pod cần quyền `NET_ADMIN`/`NET_RAW` (để tạo/xoá VIP qua ARP/BGP) — xem mục 7 về việc đây là 1 trong số ít workload "privileged" chấp nhận được chạy trên control-plane node Talos.
- Credential CCM (API key CloudStack / `clouds.yaml` OpenStack) nằm trong Secret ở **tenant cluster**, khác với credential CAPC/CAPO nằm ở **management cluster** (xem [[CAPC]]/[[CAPO]]) — 2 bộ credential riêng biệt, dễ nhầm là 1.
- In-tree cloud provider (built-in `--cloud-provider=` cũ trong kube-controller-manager) đã bị **gỡ khỏi K8s** từ lâu — mọi hướng dẫn cũ dùng flag này đều lỗi thời, phải dùng external CCM (ngoài kube-controller-manager) như note này mô tả.

### **7. Security Considerations**

- LB cho control-plane endpoint (port 6443) thường bị expose ra ngoài phạm vi cluster để quản trị từ xa — cần giới hạn source CIDR (security group CloudStack / Octavia listener allowed_cidrs) chỉ cho management cluster + admin IP, không mở `0.0.0.0/0`.
- kube-vip static pod chạy với network capability cao (`NET_ADMIN`/`NET_RAW`) trên **control-plane node** — đây là 1 trong rất ít workload được chấp nhận chạy privileged trên node Talos; audit kỹ image/version kube-vip dùng, vì compromise nó = có khả năng ảnh hưởng network control-plane.
- Credential CCM trong tenant cluster (dùng để gọi API CloudStack/OpenStack tạo LB) nên scope hẹp hơn credential CAPC/CAPO ở management cluster — CCM chỉ cần quyền tạo/sửa/xoá LB + đọc thông tin instance, **không cần** quyền tạo/xoá VM như CAPC/CAPO.
- Cilium eBPF chạy với quyền kernel cao (cần `CAP_BPF`/privileged tuỳ version/kernel) — đây là trade-off đổi lấy performance, cần theo dõi security advisory riêng của Cilium (ngoài phạm vi note này).

### **8. Ops Runbook — Production Notes**

- **Health check CNI**: `cilium status` (qua `cilium` CLI hoặc exec vào pod agent) — kiểm tra `kube-proxy replacement: Strict` nếu đã cấu hình đúng, và `Cluster health: Ok`.
- **Service LoadBalancer Pending**: `kubectl describe svc <name>` xem Event — thường do CCM pod crash (sai credential, quota LB hết ở IaaS) hoặc chưa cài CCM. Check log CCM pod trong tenant cluster (namespace `kube-system`, tên pod theo provider, vd `cloudstack-cloud-controller-manager-*` hoặc `openstack-cloud-controller-manager-*`).
- **kube-vip**: `kubectl get pods -n kube-system -l app=kube-vip` + log để xem leader hiện tại; mất VIP (không ping được) thường do tất cả control-plane down hoặc lỗi ARP/BGP announce — kiểm tra network switch/router có chặn gratuitous ARP không (hay gặp khi hạ tầng mạng CloudStack/OpenStack có anti-spoofing).
- **Metric cần alert**: ⚠️ **cần verify** metric cụ thể CCM expose theo từng provider (không chuẩn hoá) — nhìn chung alert theo CCM pod restart count tăng bất thường + số Service `Pending` quá lâu (>5 phút).

### **9. Gotchas & Lessons Learned**

- Dễ dính "bẫy tên project" nhất của cả series này: Google "cloudstack cloud controller manager" ra rất nhiều tutorial cũ dùng `swisstxt/cloudstack-cloud-controller-manager` — project đã deprecated, tự khuyến nghị migrate sang `apache/cloudstack-kubernetes-provider`. Luôn kiểm tra README/badge "archived" trước khi `helm install`/clone theo blog cũ.
- In-tree cloud provider legacy (`--cloud-provider=openstack` built-in cũ) đã gỡ khỏi K8s từ lâu — vẫn thấy nhiều tài liệu/forum cũ tham chiếu flag này, không áp dụng được cho version K8s hiện tại.
- KubePrism dễ bị hiểu nhầm là "thay thế hoàn toàn" cho control-plane LB — thực ra nó chỉ giải quyết traffic **nội bộ** (kubelet, control-plane component gọi nhau), **không** giải quyết traffic từ ngoài vào (CABPT/CACPPT ở management cluster gọi vào tenant cluster, hoặc `kubectl` của admin) — LB/kube-vip vẫn cần cho traffic đó.
- (Mục này để tự bổ sung tiếp sau khi thao tác trực tiếp với CCM thật trên CloudStack/OpenStack lab — lesson learned thực chiến quan trọng hơn note lý thuyết)

### **10. Resources**

- Deploy Cilium trên Talos (official): https://www.talos.dev/v1.11/kubernetes-guides/network/deploying-cilium/
- Cilium + Talos (official Cilium docs): https://docs.cilium.io/en/stable/installation/k8s-install-talos-linux/
- CloudStack CCM chính thức: https://github.com/apache/cloudstack-kubernetes-provider
- CloudStack CCM bản cũ đã deprecated (chỉ để biết mà tránh): https://github.com/swisstxt/cloudstack-cloud-controller-manager
- OpenStack CCM chính thức (SIG): https://github.com/kubernetes/cloud-provider-openstack
- Expose Service LoadBalancer trên OpenStack: https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/openstack-cloud-controller-manager/expose-applications-using-loadbalancer-type-service.md
- Octavia Ingress Controller: https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/octavia-ingress-controller/using-octavia-ingress-controller.md
- kube-vip static pod docs: https://kube-vip.io/docs/installation/static/
- **Lưu ý khi dùng note này**: thông tin lấy qua search engine + docs chính thức đọc gián tiếp qua search snippet, chưa fetch full từng trang — đặc biệt phần metric/alert và ngày release cụ thể của `apache/cloudstack-kubernetes-provider` cần verify lại trực tiếp trước khi dùng cho production.
