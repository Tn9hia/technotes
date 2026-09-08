---
tags:
  - ceph
  - deployment
  - cephadm
---

# Ceph Deployment Models (cephadm, Rook, ceph-ansible)

Ceph không còn một cách cài đặt duy nhất — theo thời gian có 3 công cụ orchestration chính, mỗi cái ứng với một triết lý vận hành khác nhau. **`cephadm` là công cụ chuẩn hiện tại** (mặc định từ Ceph Octopus, hoàn thiện từ Pacific/Quincy trở đi, và là công cụ được dùng trong tài liệu chính thức của Squid/Tentacle) — nếu bạn nhận bàn giao một cluster mới triển khai gần đây, gần như chắc chắn đó là cephadm. `ceph-deploy` (công cụ cũ hơn cephadm) đã **bị loại bỏ hoàn toàn**, không còn được maintain — nếu thấy tài liệu/blog nào nhắc tới `ceph-deploy`, đó là tài liệu lỗi thời, bỏ qua.

> [!tip] So với VMware vSAN
> vSAN không có khái niệm "chọn công cụ triển khai" — nó luôn được bật/quản lý qua vCenter. Ceph thì ngược lại: RADOS là phần lõi giống nhau, nhưng cách bạn *bootstrap và quản lý vòng đời* các daemon (MON/MGR/OSD...) có thể khác hẳn tùy tổ chức chọn cephadm (tự quản lý, giống vSAN quản lý qua vCenter), Rook (nếu hạ tầng là Kubernetes, vòng đời daemon do k8s operator lo — không có tương đương trực tiếp bên VMware), hay ceph-ansible (config-management driven, gần giống việc dùng script/Ansible để cấu hình ESXi host thủ công thay vì để vCenter lo).

## Ba công cụ triển khai chính

| Công cụ | Mô hình orchestration | Daemon chạy như thế nào | Khi nào dùng | Vòng đời/lifecycle |
|---|---|---|---|---|
| **cephadm** (chuẩn hiện tại) | Ceph tự orchestrate chính nó, không cần tool bên ngoài | Container (podman/docker) do systemd quản lý trên từng host | Mặc định cho mọi triển khai mới — bare-metal hoặc VM | `ceph orch` lo add/remove/upgrade daemon, tự động, khai báo (declarative) |
| **Rook** | Kubernetes Operator | Container chạy như Pod trong k8s, Rook operator là CRD-driven | Tổ chức đã vận hành Kubernetes và muốn Ceph là 1 phần của hệ sinh thái k8s (ví dụ backend storage cho chính k8s) | k8s-native: `kubectl apply`, rolling upgrade qua CRD, Rook tự dịch sang thao tác Ceph |
| **ceph-ansible** (legacy) | Ansible playbook, config management truyền thống | Daemon chạy trực tiếp trên host (package) hoặc container tùy playbook | Cluster cũ đã triển khai từ trước (thường thấy trong Red Hat Ceph Storage bản cũ), đang trong lộ trình bị thay thế bởi cephadm | Thủ công qua re-run playbook, không có "orchestrator" runtime tích hợp |

> [!warning] ceph-deploy đã bị khai tử — đừng học nó như hiện tại
> `ceph-deploy` từng là công cụ CLI đơn giản phổ biến (`ceph-deploy new`, `ceph-deploy osd create`...) nhưng đã bị **loại bỏ khỏi các bản Ceph hiện đại**. Nếu tài liệu bàn giao, wiki nội bộ, hay runbook cũ của công ty còn nhắc tên này, đó là dấu hiệu tài liệu đã lỗi thời nghiêm trọng — cần rà soát lại toàn bộ, khả năng cao quy trình vận hành thực tế cũng đã đổi sang cephadm mà tài liệu chưa cập nhật theo.

## cephadm — công cụ chuẩn cần nắm vững

### Bootstrap cluster

```bash
# Chạy trên node đầu tiên — cephadm tự tải image container, khởi tạo MON/MGR đầu tiên
cephadm bootstrap --mon-ip <IP-node-dau-tien> \
  --initial-dashboard-user admin \
  --initial-dashboard-password '<password>'

# Kết quả: có ngay 1 MON, 1 MGR, dashboard bật sẵn, và file keyring admin tại
# /etc/ceph/ceph.client.admin.keyring trên node bootstrap
```

### Thêm host và quản lý daemon qua `ceph orch`

```bash
# Copy SSH key của cephadm (tạo tự động lúc bootstrap) sang host mới
ssh-copy-id -f -i /etc/ceph/ceph.pub root@<host-moi>

# Add host vào cluster, gắn label để orchestrator biết vai trò
ceph orch host add osd-node-04 <IP> --labels osd
ceph orch host ls

# Khai báo daemon placement (declarative — Ceph tự cân bằng số lượng)
ceph orch apply mon --placement="3 mon-node-01,mon-node-02,mon-node-03"
ceph orch apply mgr --placement="2"

# Tạo OSD tự động trên toàn bộ disk trống chưa dùng
ceph orch apply osd --all-available-devices

# Xem toàn bộ daemon đang chạy trên cluster
ceph orch ps
ceph orch device ls
```

### `cephadm shell` — cách chuẩn để chạy lệnh `ceph` trên cluster cephadm

Với cephadm, các daemon chạy trong container — **không có gói `ceph-common` cài sẵn trên mọi host** như kiểu triển khai package truyền thống. Cách chuẩn để chạy CLI admin là:

```bash
# Mở 1 container tạm thời, tự mount đúng keyring + ceph.conf từ host,
# xóa đi khi thoát — không để lại rác trên host
cephadm shell

# Bên trong shell này, mọi lệnh ceph/rbd/rados hoạt động bình thường
ceph -s
ceph orch ls

# Chạy 1 lệnh đơn lẻ mà không cần vào shell tương tác
cephadm shell -- ceph -s

# Cách thay thế: cài gói ceph-common + copy keyring thủ công lên host cụ thể
# (làm được, nhưng không phải cách "mặc định" của cephadm — chỉ nên làm
# trên node quản trị cố định, không làm tràn lan trên mọi host)
cephadm install ceph-common
```

## Rook (Kubernetes)

Nếu tổ chức của bạn cũng vận hành Kubernetes song song, Rook là lựa chọn đáng cân nhắc: Rook là một **Kubernetes Operator**, biến việc quản lý Ceph thành các Custom Resource (`CephCluster`, `CephBlockPool`, `CephObjectStore`...). Bạn không chạy `ceph orch` trực tiếp — thay vào đó `kubectl apply -f cluster.yaml`, Rook operator tự dịch sang thao tác RADOS bên dưới.

```yaml
# Ví dụ rút gọn — khai báo CephCluster qua Rook CRD
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v19
  mon:
    count: 3
  storage:
    useAllNodes: true
    useAllDevices: true
```

Với hạ tầng CloudStack/KVM thuần túy (không có k8s layer), Rook thường **không liên quan** — chỉ cần biết nó tồn tại để không nhầm lẫn khi đọc tài liệu Ceph tổng quát hoặc khi tổ chức có thêm cụm k8s dùng chung storage.

## ceph-ansible (legacy)

`ceph-ansible` là bộ playbook Ansible từng là chuẩn de-facto trước khi cephadm ra đời, đặc biệt phổ biến trong các bản Red Hat Ceph Storage cũ (RHCS 3/4). Nó dùng cách tiếp cận "config management" truyền thống: khai báo host/role trong inventory, chạy playbook để cài package/cấu hình daemon chạy trực tiếp trên host (không nhất thiết containerized).

```ini
# Ví dụ trích đoạn inventory ceph-ansible (chỉ mang tính minh họa lịch sử)
[mons]
mon-node-01
mon-node-02
mon-node-03

[osds]
osd-node-01
osd-node-02
```

Nếu bạn nhận bàn giao một cluster vẫn dùng ceph-ansible, cần biết: (1) không có `ceph orch` runtime — mọi thay đổi topology phải qua re-run playbook, (2) migrate sang cephadm là khả thi (Ceph có tài liệu chuyển đổi chính thức) nhưng là một dự án riêng cần lên kế hoạch, không nên coi nhẹ.

## So sánh nhanh khi cần quyết định

| Tiêu chí | cephadm | Rook | ceph-ansible |
|---|---|---|---|
| Cần hạ tầng ngoài? | Không (chỉ cần podman/docker + SSH) | Có (Kubernetes cluster) | Có (Ansible control node) |
| Daemon chạy dạng gì | Container, systemd quản lý | Container, k8s Pod quản lý | Thường process trực tiếp trên host |
| Thao tác day-2 chuẩn | `ceph orch ...` | `kubectl apply` (CRD) | Re-run Ansible playbook |
| Trạng thái hiện tại | **Chuẩn khuyến nghị** | Phù hợp khi có sẵn k8s | Legacy, đang bị phase-out |
| Độ khó học với người mới | Trung bình, tài liệu chính thức đầy đủ | Cần biết k8s trước | Cần biết Ansible + Ceph thủ công |

> [!warning] Lesson learned: chạy `ceph` command trên nhầm host, tưởng cluster bị "connection refused"
> Người mới quen với triển khai package-based cũ (kiểu ceph-ansible/RHCS) thường mặc định "cứ SSH vào bất kỳ node nào trong cluster rồi gõ `ceph -s` là chạy được" — vì gói `ceph-common` từng được cài trên mọi node. Với cephadm, **không phải host nào cũng có `ceph` CLI và keyring admin sẵn** — chỉ node bootstrap (hoặc host bạn chủ động cài `ceph-common` + copy keyring) mới chạy trực tiếp được. SSH vào một OSD host bất kỳ rồi gõ `ceph -s` thường ra lỗi kiểu "command not found" hoặc permission denied vì thiếu keyring, khiến người mới hoảng hốt tưởng cluster có sự cố. Cách xử lý: dùng `cephadm shell` (chạy được trên host có cephadm) hoặc xác định đúng "node quản trị" đã được cấu hình sẵn admin keyring trước khi kết luận cluster có vấn đề.

---
*Xem thêm: [[Ceph CLI Cheatsheet]] | [[Ceph Hardware & Network Design]] | [[Ceph|Ceph]]*
