# Ceph Cluster - Triển khai Primary và Secondary Storage cho CloudStack

- **Bối cảnh và vấn đề**: CloudStack cần Primary Storage (nơi lưu volume/disk của VM, đòi hỏi performance và low-latency) và Secondary Storage (nơi lưu template, ISO, snapshot). Dựng hai storage system riêng biệt cho hai mục đích này tốn gấp đôi hạ tầng, trong khi ở quy mô nhỏ một cụm Ceph production-grade có thể phục vụ tốt cả hai vai trò nếu tách pool/service và giới hạn quyền truy cập đúng cách.
- **Cách giải quyết**: Dùng `cephadm` triển khai một cụm Ceph 3 node (converged: mon + mgr + osd + mds trên cùng node) trên Ubuntu 24.04. Tạo pool RBD riêng (`cloudstack-primary`) làm Primary Storage cho KVM/CloudStack qua giao thức RBD. Triển khai CephFS + NFS-Ganesha (Ceph `nfs` module, có ingress VIP để HA) export ra làm Secondary Storage qua NFS. Toàn bộ cụm được hardening: cephx caps tối thiểu theo từng client, mã hoá dữ liệu tại chỗ (OSD dmcrypt), mã hoá đường truyền nội bộ (msgr v2 secure mode), tách network Public/Cluster/Management, và Dashboard chạy HTTPS với RBAC riêng.
- **Kết quả sau khi hoàn thành**: CloudStack có một Primary Storage Pool chạy trên RBD và một Secondary Storage Pool chạy trên NFS, cùng share hạ tầng vật lý của một cụm Ceph 3 node. Admin thao tác OSD/pool qua `ceph orch`, không cần quản lý daemon thủ công. Người đọc nắm được lý do đằng sau từng tham số hardening để tự điều chỉnh khi lên production thật.

> [!NOTE]
> Lab này là một phần của series dựng cụm CloudStack production hoàn chỉnh — xem [[CloudStack Production Cluster - Lab Series Overview]] để biết thứ tự triển khai đầy đủ cùng các lab Control Plane, Compute Node, Tungsten Fabric SDN, Advanced Zone.

> [!NOTE]
> Lab này chọn kiến trúc **converged** (mon+mgr+osd+mds+nfs+rgw dùng chung 3 node) để tối ưu chi phí phần cứng cho một cụm nhỏ. Production lớn hơn nên tách riêng node cho gateway (RGW/NFS-Ganesha) khỏi node OSD để tránh cạnh tranh CPU/RAM lúc rebalance. Phần [Reference](#reference) có link tới kiến trúc scale-out khi cụm lớn hơn 3 node.

> [!NOTE]
> Về lựa chọn NFS hay S3 cho Secondary Storage: lab này triển khai **CephFS + NFS-Ganesha (native NFS)** làm phương án chính vì tương thích rộng với mọi phiên bản CloudStack (KVM host mount NFS trực tiếp, không cần lớp dịch nào ở giữa) và đơn giản hơn khi vận hành với một cụm nhỏ. Bước [Bước 8 - Triển khai RGW](#bước-8---tuỳ-chọn-triển-khai-rgw-làm-secondary-storage-dạng-s3) trình bày phương án S3 gốc (không qua NFS staging) như một lựa chọn thay thế để scale-out về sau. Phương án "S3 qua NFS staging" (kiểu cũ, CloudStack cache template ra NFS trước rồi đẩy lên S3) không được khuyến nghị vì cộng dồn độ phức tạp vận hành của cả NFS lẫn S3 mà không có lợi ích tương xứng ở quy mô nhỏ.

## Prerequisites

- **Hạ tầng**: CloudStack Management Server và ít nhất 1 KVM Cluster/Host đã cài đặt, add vào zone, đang ở trạng thái Up. DNS/NTP nội bộ hoạt động, các node Ceph resolve được lẫn nhau (qua DNS nội bộ hoặc `/etc/hosts`).
- **Máy chủ / VM**: 3 node vật lý hoặc VM riêng biệt (không đặt chung host với hypervisor KVM đang chạy workload, tránh vòng lặp phụ thuộc storage). Cấu hình ví dụ dùng trong lab — điều chỉnh lại theo capacity thực tế:

  | Node | CPU | RAM | OS disk | OSD disk |
  | --- | --- | --- | --- | --- |
  | ceph-node01/02/03 | 16 vCPU | 64 GB | 2x 480GB SSD (RAID1) | 4x 3.84TB SSD/NVMe (bluestore) |

- **Tài khoản và quyền**: user sudo trên cả 3 node để cài đặt và cho `cephadm` SSH vào orchestrate; nếu dùng internal CA, cần quyền request certificate cho Dashboard/RGW/NFS.
- **Mạng**: dải Public network, Cluster network, Management network tách biệt (VLAN riêng) đã xin từ team Network — xem chi tiết placeholder ở bảng Planning bên dưới. Switch hỗ trợ Jumbo Frame nếu muốn bật MTU 9000 cho Cluster network.
- **Kiến thức nền**: runbook này giả định người đọc đã biết Linux administration cơ bản, khái niệm TCP/IP, khái niệm Ceph (OSD/MON/MGR/PG/CRUSH) và khái niệm Zone/Pod/Cluster/Primary-Secondary Storage trong CloudStack — không giải thích lại từ đầu.

> [!NOTE]
> Lab giả định đây là Primary/Secondary Storage **mới hoàn toàn** được add thêm vào zone, không phải migrate dữ liệu từ storage hiện có. Việc di chuyển volume/template từ storage cũ sang Ceph nằm ngoài phạm vi runbook này.

## Thông tin Planning liên quan

<!-- Network để trống theo yêu cầu — điền lại sau khi có kết quả planning với team Network. -->

| Thành phần | Giá trị | Ghi chú |
| --- | --- | --- |
| ceph-node01 hostname | `<TBD>` | mon + mgr + osd + mds, label thêm `_admin` (host bootstrap) |
| ceph-node02 hostname | `<TBD>` | mon + mgr + osd + mds |
| ceph-node03 hostname | `<TBD>` | mon + mgr + osd + mds |
| Management network CIDR | `<TBD>` | SSH, cephadm orchestration, Dashboard HTTPS |
| Public network CIDR | `<TBD>` | Client traffic: RBD (CloudStack/KVM), NFS, RGW |
| Cluster network CIDR | `<TBD>` | Riêng biệt, không route ra ngoài — OSD replication/heartbeat |
| MTU Cluster network | `9000` | Jumbo frame, giảm CPU overhead khi replicate giữa các OSD |
| VIP NFS-Ganesha ingress | `<TBD>` | VIP HA cho Secondary Storage (NFS) |
| VIP RGW ingress (tuỳ chọn) | `<TBD>` | VIP HA cho RGW nếu dùng phương án S3 |
| Ceph release | `<xác nhận bản LTS mới nhất tại docs.ceph.com/en/latest/releases>` | Pin version cụ thể trước khi bootstrap, không dùng bản dev/rc cho production |
| cephx client Primary Storage | `client.cloudstack-rbd` | caps giới hạn trong pool `cloudstack-primary` |
| Pool Primary Storage | `cloudstack-primary` | replicated x3, min_size 2 |
| CephFS volume Secondary Storage | `cloudstack-secondary` | data + metadata pool, replicated x3 |
| NFS export path | `/cloudstack-secondary` | pseudo path export cho CloudStack Secondary Storage VM (SSVM) |
| Secondary Storage client CIDR | `<TBD>` | Dải IP của SSVM + KVM host, dùng để giới hạn NFS export |
| Dashboard admin user | `<TBD>` | Role `administrator`, không dùng chung tài khoản `admin` mặc định |
| SSH orchestration user | `ceph-adm` | User riêng cho `cephadm` SSH, không dùng `root` |

## Diagram

```mermaid
flowchart TD
    MGMT[CloudStack Management Server] -- "1. RBD + cephx" --> PUB["Ceph Public Network<br/>&lt;public-cidr&gt;"]
    KVM[KVM Hypervisor Hosts] -- "librbd" --> PUB
    SSVM[Secondary Storage VM] -- "2. NFSv4.1" --> VIP["NFS Ingress VIP<br/>&lt;nfs-vip&gt;"]

    PUB --> N1[ceph-node01<br/>mon+mgr+osd+mds]
    PUB --> N2[ceph-node02<br/>mon+mgr+osd+mds]
    PUB --> N3[ceph-node03<br/>mon+mgr+osd+mds]

    VIP --> NFS1[nfs-ganesha<br/>node01/node02]
    NFS1 --> CFS[CephFS pool<br/>cloudstack-secondary]

    N1 --> RBD[Pool cloudstack-primary<br/>replicated x3]

    N1 -- "3. Cluster network<br/>replication/heartbeat" --> N2
    N2 -- "3. Cluster network" --> N3
    N1 -- "3. Cluster network" --> N3
```

---

## Installation

### Bước 1 - Chuẩn bị hệ điều hành Ubuntu 24.04 trên 3 node

Milestone này đưa 3 node về cùng baseline trước khi cephadm bắt đầu quản lý — sai NTP hoặc thiếu resolve hostname là hai nguyên nhân phổ biến nhất khiến bootstrap hoặc cephx auth thất bại.

- Đặt hostname đúng theo Planning table (thực hiện trên từng node):

```bash
sudo hostnamectl set-hostname <ceph-nodeXX-hostname>
```

- Khai báo resolve giữa các node. Chỉnh sửa file `/etc/hosts` trên cả 3 node, thêm IP Public network của cả 3:

```text
<public-ip-node01>  ceph-node01
<public-ip-node02>  ceph-node02
<public-ip-node03>  ceph-node03
```

- Cài đặt và đồng bộ NTP bằng `chrony`. Cephx dùng timestamp để chống replay attack — lệch giờ quá `mon_clock_drift_allowed` (mặc định 50ms) sẽ khiến node bị đá khỏi quorum:

```bash
sudo apt update
sudo apt install -y chrony
sudo systemctl enable chrony --now
chronyc tracking
```

- Tạo user riêng cho `cephadm` orchestrate qua SSH, không dùng `root`:

```bash
sudo adduser --disabled-password --gecos "" ceph-adm
sudo usermod -aG sudo ceph-adm
echo "ceph-adm ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ceph-adm
```

> [!NOTE]
> `NOPASSWD:ALL` ở đây để `cephadm` tự động chạy lệnh qua SSH không cần nhập password tương tác. Nếu tổ chức yêu cầu chặt hơn, có thể giới hạn danh sách lệnh cụ thể trong sudoers, nhưng cần test kỹ vì `cephadm` gọi khá nhiều binary khác nhau (`systemctl`, container runtime, `mkdir`...).

- Sinh SSH keypair trên node bootstrap (ceph-node01) và copy public key sang cả 3 node cho user `ceph-adm`:

```bash
sudo -u ceph-adm ssh-keygen -t ed25519 -f /home/ceph-adm/.ssh/ceph-adm-key -N ""
sudo -u ceph-adm ssh-copy-id -i /home/ceph-adm/.ssh/ceph-adm-key.pub ceph-adm@ceph-node01
sudo -u ceph-adm ssh-copy-id -i /home/ceph-adm/.ssh/ceph-adm-key.pub ceph-adm@ceph-node02
sudo -u ceph-adm ssh-copy-id -i /home/ceph-adm/.ssh/ceph-adm-key.pub ceph-adm@ceph-node03
```

- Cài đặt Podman làm container runtime cho `cephadm`. Ubuntu 24.04 có sẵn Podman trong repo chính thức nên không cần thêm repo bên thứ ba như khi cài Docker CE:

```bash
sudo apt install -y podman
```

- Thiết lập firewall baseline bằng `ufw`, mặc định deny toàn bộ inbound rồi mở dần theo từng service ở các bước sau:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from <management-cidr> to any port 22 proto tcp
sudo ufw enable
```

> [!WARNING]
> Chạy `ufw enable` qua kết nối SSH từ xa có thể tự khoá bản thân nếu rule allow port 22 sai dải mạng. Luôn kiểm tra lại rule `allow ... port 22` trỏ đúng `<management-cidr>` trước khi enable.

- Kiểm tra kết quả bước này trên cả 3 node:

```bash
chronyc tracking | grep "Leap status"
ssh ceph-adm@ceph-node02 hostname
```

Kết quả mong đợi: `Leap status: Normal` và lệnh SSH từ node01 sang node02 chạy được không hỏi password.

### Bước 2 - Cài đặt cephadm và Bootstrap Ceph Cluster

Bootstrap khởi tạo mon/mgr đầu tiên và sinh ra cephx admin keyring — đây là node sẽ giữ label `_admin`.

- Cài đặt `cephadm` trên ceph-node01, dùng đúng release đã xác nhận ở Planning table:

```bash
CEPH_RELEASE=<ceph-release-đã-xác-nhận-tại-docs.ceph.com>
curl --silent --remote-name --location https://raw.githubusercontent.com/ceph/ceph/$CEPH_RELEASE/src/cephadm/cephadm
chmod +x cephadm
sudo ./cephadm add-repo --release $CEPH_RELEASE
sudo ./cephadm install
```

- Bootstrap cluster với network và ssh-user đã chuẩn bị:

```bash
sudo cephadm bootstrap \
  --mon-ip <public-ip-node01> \
  --cluster-network <cluster-network-cidr> \
  --ssh-user ceph-adm \
  --ssh-private-key /home/ceph-adm/.ssh/ceph-adm-key \
  --ssh-public-key /home/ceph-adm/.ssh/ceph-adm-key.pub \
  --initial-dashboard-user <dashboard-admin-user> \
  --initial-dashboard-password "$(cat /root/ceph-dashboard.pass)"
```

> [!WARNING]
> Không gõ password trực tiếp trên command line — nó sẽ lưu lại trong `~/.bash_history` và trong output của `ps aux` lúc lệnh đang chạy. Tạo password trước bằng `openssl rand -base64 20 | sudo tee /root/ceph-dashboard.pass && sudo chmod 600 /root/ceph-dashboard.pass` rồi đọc lại qua `$(cat ...)` như trên, và xoá file này ngay sau khi bootstrap xong.

- Cấu hình biến môi trường để dùng `ceph` CLI trực tiếp trên node bootstrap:

```bash
sudo cp /etc/ceph/ceph.client.admin.keyring /etc/ceph/ceph.client.admin.keyring.bak
sudo chmod 600 /etc/ceph/ceph.client.admin.keyring
```

> [!WARNING]
> `/etc/ceph/ceph.client.admin.keyring` cấp full quyền lên toàn cụm. Chỉ giữ file này trên node có label `_admin` (mặc định là node bootstrap), không copy sang node khác hay sang máy CloudStack Management Server.

- Kiểm tra kết quả bước này:

```bash
sudo ceph -s
```

Kết quả mong đợi: `health: HEALTH_OK` (hoặc `HEALTH_WARN` với cảnh báo "1 mon" / "no OSDs" — bình thường ở giai đoạn này vì chưa thêm node và OSD).

### Bước 3 - Thêm node vào cluster và gán label

Label quyết định `cephadm` sẽ deploy daemon gì lên node nào ở các bước `orch apply` sau này.

- Copy public key của cụm (được `cephadm` tự sinh lúc bootstrap) sang 2 node còn lại:

```bash
sudo ssh-copy-id -f -i /etc/ceph/ceph.pub ceph-adm@ceph-node02
sudo ssh-copy-id -f -i /etc/ceph/ceph.pub ceph-adm@ceph-node03
```

- Thêm node02 và node03 vào cluster, gán label `mon,mgr,osd,mds`:

```bash
sudo ceph orch host add ceph-node02 <public-ip-node02> --labels mon,mgr,osd,mds
sudo ceph orch host add ceph-node03 <public-ip-node03> --labels mon,mgr,osd,mds
```

- Gán label cho node01 (đã có sẵn label `_admin` từ lúc bootstrap):

```bash
sudo ceph orch host label add ceph-node01 mon
sudo ceph orch host label add ceph-node01 mgr
sudo ceph orch host label add ceph-node01 osd
sudo ceph orch host label add ceph-node01 mds
```

- Gán thêm label `nfs` cho node01 và node02 (2 node cho HA của NFS-Ganesha, dùng ở Bước 7):

```bash
sudo ceph orch host label add ceph-node01 nfs
sudo ceph orch host label add ceph-node02 nfs
```

- Áp dụng placement cho mon/mgr theo label, đảm bảo đúng 3 mon để có quorum chịu được 1 node down:

```bash
sudo ceph orch apply mon --placement="label:mon"
sudo ceph orch apply mgr --placement="label:mgr"
```

- Kiểm tra kết quả bước này:

```bash
sudo ceph orch host ls
sudo ceph -s
```

Kết quả mong đợi: 3 host hiển thị đủ label, `ceph -s` báo `3 mons, quorum ceph-node01,ceph-node02,ceph-node03`.

### Bước 4 - Triển khai OSD với mã hoá dữ liệu tại chỗ (dmcrypt)

Milestone này đưa toàn bộ disk flash trống vào cụm dưới dạng OSD bluestore, mã hoá bằng LUKS ngay từ lúc tạo — mất chi phí CPU không đáng kể nhưng bảo vệ dữ liệu nếu disk vật lý bị tháo trộm khỏi node.

- Tạo file spec `osd-spec.yaml` trên node01:

```yaml
service_type: osd
service_id: cloudstack-osds
placement:
  label: "osd"
spec:
  data_devices:
    rotational: 0
  encrypted: true
```

> [!NOTE]
> `rotational: 0` chọn mọi device flash (SSD/NVMe) đang ở trạng thái "available" (chưa có filesystem/partition) trên host có label `osd`. OS disk không bị chọn nhầm vì nó đã có filesystem từ lúc cài Ubuntu. `encrypted: true` bật dmcrypt full-disk — key LUKS được `cephadm`/`ceph-volume` tự sinh và lưu trong Monitor config-key store, không cần quản lý key thủ công.

- Áp dụng spec:

```bash
sudo ceph orch apply -i osd-spec.yaml
```

> [!WARNING]
> Lệnh này format mọi disk "available" khớp filter — chạy `ceph orch device ls` trước để xác nhận đúng danh sách disk dự kiến, tránh format nhầm disk đang chứa dữ liệu khác.

- Kiểm tra kết quả bước này:

```bash
sudo ceph orch device ls
sudo ceph osd tree
```

Kết quả mong đợi: mỗi node hiển thị 4 OSD `up`, tổng 12 OSD trên cả cụm, `ceph -s` chuyển dần về `HEALTH_OK` sau khi PG rebalance xong.

### Bước 5 - Hardening authentication và network ở tầng cluster

Các cấu hình này nên bật ngay sau khi cụm healthy, trước khi tạo pool phục vụ production traffic.

- Bật mã hoá đường truyền nội bộ giữa các daemon (msgr v2 secure mode):

```bash
sudo ceph config set global ms_cluster_mode secure
sudo ceph config set global ms_service_mode secure
sudo ceph config set global ms_client_mode secure
```

> [!NOTE]
> Mặc định msgr v2 chỉ bật `crc` mode (kiểm tra toàn vẹn, không mã hoá). `secure` mode mã hoá toàn bộ traffic giữa mon/osd/mds/client bằng AES-GCM — cần thiết nếu Cluster network hoặc Public network đi qua switch/segment không hoàn toàn tin cậy.

- Vô hiệu hoá cơ chế `insecure global id reclaim` (liên quan CVE-2021-20288/CVE-2023-…, cho phép client giả mạo global_id sau khi mất kết nối mon):

```bash
sudo ceph health detail | grep -i "insecure global id"
sudo ceph config set mon auth_allow_insecure_global_id_reclaim false
```

> [!WARNING]
> Chỉ set `false` sau khi xác nhận không còn client nào (kể cả KVM host, CloudStack Management Server) đang dùng thư viện `librados`/`librbd` cũ chưa hỗ trợ reclaim an toàn — nếu còn, client đó sẽ mất kết nối tới cụm. Kiểm tra bằng lệnh `ceph health detail` ở trên trước khi set.

- Chặn thao tác xoá pool ngoài ý muốn — một `rm pool` sai tay có thể xoá sạch dữ liệu Primary/Secondary Storage đang phục vụ CloudStack:

```bash
sudo ceph config set mon mon_allow_pool_delete false
```

- Mở firewall cho các port cần thiết của Ceph daemon trên Public network (thực hiện trên cả 3 node):

```bash
sudo ufw allow from <public-network-cidr> to any port 3300,6789 proto tcp comment 'ceph mon'
sudo ufw allow from <public-network-cidr> to any port 6800:7300 proto tcp comment 'ceph osd/mds'
sudo ufw allow from <management-network-cidr> to any port 8443 proto tcp comment 'ceph dashboard'
sudo ufw allow from <management-network-cidr> to any port 9283 proto tcp comment 'ceph prometheus exporter'
```

> [!NOTE]
> Port 6789 (msgr v1) vẫn cần mở song song với 3300 (msgr v2) vì một số client librbd cũ (tuỳ version QEMU trên KVM host) vẫn kết nối qua v1. Chỉ tắt bind v1 (`ms_bind_msgr1 false`) sau khi xác nhận toàn bộ KVM host đã dùng QEMU/librbd bản hỗ trợ msgr v2.

- Kiểm tra kết quả bước này:

```bash
sudo ceph config get global ms_cluster_mode
sudo ceph health detail
```

Kết quả mong đợi: `ms_cluster_mode` trả về `secure`, `ceph health detail` không còn cảnh báo `AUTH_INSECURE_GLOBAL_ID_RECLAIM`.

### Bước 6 - Tạo Pool RBD và cephx key cho CloudStack Primary Storage

- Tạo pool riêng cho Primary Storage, không dùng chung pool với dữ liệu khác để giới hạn blast radius nếu cephx key của CloudStack bị lộ:

```bash
sudo ceph osd pool create cloudstack-primary 128 128 replicated
sudo ceph osd pool set cloudstack-primary size 3
sudo ceph osd pool set cloudstack-primary min_size 2
sudo ceph osd pool application enable cloudstack-primary rbd
sudo rbd pool init cloudstack-primary
```

> [!NOTE]
> `min_size 2` nghĩa là pool vẫn nhận write khi 1 trong 3 replica down (ví dụ đang bảo trì 1 node), nhưng sẽ block write nếu chỉ còn 1 bản sao — tránh write vào trạng thái không đủ redundancy. Số PG 128 phù hợp cho 12 OSD ở quy mô lab này; công thức và PG calculator tham khảo ở phần Reference khi scale cụm lớn hơn.

- Tạo cephx client riêng cho CloudStack, dùng `profile rbd` — caps chuẩn được Ceph khuyến nghị cho tích hợp RBD với hypervisor, chỉ cho phép thao tác trong đúng pool `cloudstack-primary`:

```bash
sudo ceph auth get-or-create client.cloudstack-rbd \
  mon 'profile rbd' \
  osd 'profile rbd pool=cloudstack-primary'
```

> [!WARNING]
> Không dùng `client.admin` cho CloudStack. Nếu key `client.cloudstack-rbd` bị lộ, kẻ tấn công chỉ thao tác được trong pool `cloudstack-primary`, không đọc/xoá được pool khác hay thay đổi cấu hình cụm.

- Lấy secret key để khai báo vào CloudStack ở Bước 9 (không in ra terminal log tồn tại lâu dài):

```bash
sudo ceph auth print-key client.cloudstack-rbd
```

- Kiểm tra kết quả bước này:

```bash
sudo ceph auth get client.cloudstack-rbd
sudo rbd -p cloudstack-primary --id cloudstack-rbd ls
```

Kết quả mong đợi: lệnh `rbd ls` chạy được (trả về danh sách rỗng vì pool mới tạo) mà không bị lỗi permission denied — xác nhận cephx caps hoạt động đúng.

### Bước 7 - Triển khai CephFS + NFS-Ganesha cho CloudStack Secondary Storage

- Tạo CephFS volume, `cephadm` tự tạo data pool + metadata pool và deploy MDS theo label:

```bash
sudo ceph fs volume create cloudstack-secondary --placement="label:mds"
```

- Đặt replication cho 2 pool vừa tạo (mặc định kế thừa `osd_pool_default_size`, khai báo tường minh cho rõ ràng):

```bash
sudo ceph osd pool set cephfs.cloudstack-secondary.data size 3
sudo ceph osd pool set cephfs.cloudstack-secondary.data min_size 2
sudo ceph osd pool set cephfs.cloudstack-secondary.meta size 3
sudo ceph osd pool set cephfs.cloudstack-secondary.meta min_size 2
```

- Tạo NFS cluster (Ceph `nfs` module quản lý NFS-Ganesha như một service của orchestrator):

```bash
sudo ceph nfs cluster create cloudstack-nfs --placement="label:nfs"
```

- Tạo export, giới hạn client theo CIDR của SSVM/KVM host thay vì mở cho toàn bộ Public network:

```bash
sudo ceph nfs export create cephfs \
  --cluster-id cloudstack-nfs \
  --pseudo-path /cloudstack-secondary \
  --fsname cloudstack-secondary \
  --path / \
  --client_addr <secondary-storage-client-cidr>
```

> [!NOTE]
> Cú pháp `ceph nfs export create` có thể thay đổi nhẹ giữa các minor release — chạy `ceph nfs export create cephfs --help` để xác nhận tham số đúng với version đang cài trước khi apply.

> [!WARNING]
> Nếu bỏ trống `--client_addr`, export mặc định mở cho mọi client trên Public network đọc/ghi được toàn bộ Secondary Storage — luôn giới hạn rõ dải IP của SSVM và KVM host.

- Triển khai ingress (haproxy + keepalived do `cephadm` quản lý) để có 1 VIP HA cho NFS thay vì trỏ thẳng vào IP của 1 node:

```yaml
service_type: ingress
service_id: nfs.cloudstack-nfs
placement:
  count: 2
spec:
  backend_service: nfs.cloudstack-nfs
  frontend_port: 2049
  monitor_port: 9049
  virtual_ip: <nfs-vip>/<prefix>
  virtual_interface_networks:
    - <public-network-cidr>
```

```bash
sudo ceph orch apply -i ingress-nfs.yaml
```

- Mở firewall cho NFS trên Public network, chỉ cho phép từ dải client Secondary Storage:

```bash
sudo ufw allow from <secondary-storage-client-cidr> to any port 2049 proto tcp comment 'nfs-ganesha'
```

- Kiểm tra kết quả bước này (thực hiện từ một máy nằm trong `<secondary-storage-client-cidr>`, ví dụ SSVM hoặc KVM host):

```bash
showmount -e <nfs-vip>
sudo mount -t nfs4 -o vers=4.1 <nfs-vip>:/cloudstack-secondary /mnt
touch /mnt/test-file && ls -la /mnt/test-file
sudo umount /mnt
```

Kết quả mong đợi: mount thành công, tạo/xoá file test không lỗi permission.

### Bước 8 - (Tuỳ chọn) Triển khai RGW làm Secondary Storage dạng S3

<!-- Milestone tham khảo, không bắt buộc cho lab chính — dùng khi muốn scale-out Secondary Storage bằng object storage thay vì NFS. -->

- Triển khai RGW service theo label:

```yaml
service_type: rgw
service_id: cloudstack-s3
placement:
  label: rgw
spec:
  rgw_frontend_port: 8080
  rgw_realm: default
  rgw_zone: default
```

```bash
sudo ceph orch host label add ceph-node01 rgw
sudo ceph orch host label add ceph-node02 rgw
sudo ceph orch apply -i rgw-spec.yaml
```

- Triển khai ingress cho RGW, terminate TLS ngay tại haproxy bằng certificate từ internal CA (không dùng self-signed cho production):

```yaml
service_type: ingress
service_id: rgw.cloudstack-s3
placement:
  count: 2
spec:
  backend_service: rgw.cloudstack-s3
  frontend_port: 443
  monitor_port: 9443
  virtual_ip: <rgw-vip>/<prefix>
  ssl_cert: |
    <nội dung cert + key dạng PEM, nối liền nhau — lấy từ internal CA>
```

```bash
sudo ceph orch apply -i ingress-rgw.yaml
```

- Tạo S3 user riêng cho CloudStack:

```bash
sudo radosgw-admin user create --uid=cloudstack-s3 --display-name="CloudStack Secondary Storage"
```

> [!NOTE]
> Ghi lại `access_key`/`secret_key` trả về vào secret store — dùng để khai báo Secondary Storage trong CloudStack với `Protocol: S3`, `Endpoint: https://<rgw-vip>`, có TLS end-to-end vì ingress đã terminate HTTPS.

- Kiểm tra kết quả bước này:

```bash
s3cmd --host=<rgw-vip> --host-bucket="%(bucket)s.<rgw-vip>" --access_key=<access-key> --secret_key=<secret-key> mb s3://cloudstack-test
```

Kết quả mong đợi: bucket tạo thành công, không lỗi TLS/certificate.

### Bước 9 - Bảo mật Ceph Dashboard

- Cài certificate TLS thật từ internal CA thay vì self-signed mặc định:

```bash
sudo ceph dashboard set-ssl-certificate -i dashboard.crt
sudo ceph dashboard set-ssl-certificate-key -i dashboard.key
```

- Tạo tài khoản Dashboard theo đúng role, tách biệt tài khoản admin toàn quyền và tài khoản chỉ xem (audit):

```bash
openssl rand -base64 20 | sudo tee /root/dashboard-admin.pass
sudo ceph dashboard ac-user-create <dashboard-admin-user> -i /root/dashboard-admin.pass administrator

openssl rand -base64 20 | sudo tee /root/dashboard-readonly.pass
sudo ceph dashboard ac-user-create <dashboard-readonly-user> -i /root/dashboard-readonly.pass read-only
```

- Xoá tài khoản `admin` mặc định được sinh lúc bootstrap sau khi đã có tài khoản administrator riêng:

```bash
sudo ceph dashboard ac-user-delete admin
```

> [!WARNING]
> Chỉ xoá `admin` sau khi xác nhận tài khoản administrator mới đăng nhập được — nếu xoá nhầm trước khi có tài khoản thay thế, sẽ mất quyền truy cập Dashboard qua UI và phải xử lý lại qua CLI.

- Giảm thời gian session timeout để giảm rủi ro session bị chiếm dụng khi quên logout:

```bash
sudo ceph config set mgr mgr/dashboard/session-expire 900
```

- Kiểm tra kết quả bước này:

```bash
sudo ceph dashboard ac-user-show
curl -Iv https://<node01-ip>:8443 2>&1 | grep -i "subject\|issuer"
```

Kết quả mong đợi: danh sách user gồm tài khoản administrator + read-only, không còn `admin` mặc định; certificate hiển thị đúng issuer là internal CA thay vì self-signed.

### Khai báo thông tin nhạy cảm

- Các giá trị nhạy cảm trong lab này gồm: Dashboard admin password, cephx secret key của `client.cloudstack-rbd`, RGW access/secret key (nếu dùng Bước 8). Toàn bộ được sinh bằng `openssl rand`, lưu tạm ở `/root/*.pass` với quyền `600`, và phải được chuyển vào secret store của tổ chức (Vault, hoặc biến bí mật trong hệ thống CI/CD nếu về sau tự động hoá bằng Ansible/Terraform) rồi xoá file tạm:

```bash
sudo shred -u /root/ceph-dashboard.pass /root/dashboard-admin.pass /root/dashboard-readonly.pass
```

- cephx keyring `client.admin` (`/etc/ceph/ceph.client.admin.keyring`) chỉ tồn tại trên node có label `_admin`, quyền `600`, không đồng bộ ra ngoài cụm. Việc rotate key định kỳ nằm ngoài phạm vi lab này, tham khảo thêm ở [Reference](#reference).

## Kiểm tra kết quả

- Trạng thái tổng thể cụm phải `HEALTH_OK` trước khi tích hợp CloudStack:

```bash
sudo ceph -s
```

### Tích hợp Primary Storage vào CloudStack

- Trong CloudStack UI: **Infrastructure → Primary Storage → Add Primary Storage**, khai báo:

  | Trường | Giá trị |
  | --- | --- |
  | Protocol | `RBD` |
  | Server | `<public-ip-node01>,<public-ip-node02>,<public-ip-node03>` |
  | Port | `6789` |
  | Path (pool) | `cloudstack-primary` |
  | CephX username | `cloudstack-rbd` |
  | CephX secret | `<secret-key-lấy-ở-bước-6>` |

> [!NOTE]
> CloudStack Agent trên KVM host tự động tạo libvirt secret từ username/secret khai báo ở trên (`virsh secret-define`) — admin không cần thao tác `virsh` thủ công.

- Tạo thử một volume trên Primary Storage vừa add (qua UI: Storage → Volumes → Create Volume, chọn pool vừa thêm), sau đó xác nhận volume xuất hiện dưới dạng RBD image:

```bash
sudo rbd -p cloudstack-primary --id cloudstack-rbd ls
```

Kết quả mong đợi: thấy image tương ứng với volume vừa tạo trên UI.

### Tích hợp Secondary Storage vào CloudStack

- Trong CloudStack UI: **Infrastructure → Secondary Storage → Add Secondary Storage**, khai báo:

  | Trường | Giá trị |
  | --- | --- |
  | Provider | `NFS` |
  | Server | `<nfs-vip>` |
  | Path | `/cloudstack-secondary` |

- Upload thử 1 template hoặc ISO nhỏ, xác nhận SSVM ghi được vào NFS export:

```bash
ssh <ssvm-ip> "df -h | grep cloudstack-secondary"
```

  | Hạng mục cần kiểm tra | Cách kiểm tra | Kết quả đúng |
  | --- | --- | --- |
  | Cụm Ceph healthy | `ceph -s` | `HEALTH_OK` |
  | Primary Storage nhận volume | `rbd -p cloudstack-primary --id cloudstack-rbd ls` | Thấy RBD image mới tạo từ UI |
  | Secondary Storage mount trên SSVM | `df -h` trên SSVM | Thấy mount point trỏ `<nfs-vip>:/cloudstack-secondary` |
  | Upload template thành công | UI CloudStack → Templates | Trạng thái `Ready` |

## Troubleshooting

Không áp dụng - lab dựng mới theo hướng dẫn triển khai chuẩn, chưa có log lỗi thực tế phát sinh trong quá trình build để ghi nhận.

## Rollback

- Gỡ Secondary/Primary Storage khỏi CloudStack trước (UI → Delete Secondary Storage / Primary Storage), tránh để CloudStack còn tham chiếu tới pool sắp xoá.
- Gỡ export và NFS cluster:

```bash
sudo ceph nfs export delete cloudstack-nfs /cloudstack-secondary
sudo ceph orch rm nfs.cloudstack-nfs
sudo ceph orch rm ingress.nfs.cloudstack-nfs
```

- Gỡ RGW/ingress nếu đã triển khai Bước 8:

```bash
sudo ceph orch rm rgw.cloudstack-s3
sudo ceph orch rm ingress.rgw.cloudstack-s3
```

- Xoá CephFS volume (đồng thời xoá data + metadata pool):

```bash
sudo ceph fs volume rm cloudstack-secondary --yes-i-really-mean-it
```

- Xoá pool Primary Storage (đã set `mon_allow_pool_delete false` ở Bước 5 nên cần bật lại trước khi xoá):

```bash
sudo ceph config set mon mon_allow_pool_delete true
sudo ceph osd pool rm cloudstack-primary cloudstack-primary --yes-i-really-really-mean-it
sudo ceph config set mon mon_allow_pool_delete false
```

> [!CAUTION]
> Hai lệnh xoá pool và xoá CephFS volume ở trên **không thể hoàn tác** — toàn bộ volume/template/ISO đang lưu trong pool bị mất vĩnh viễn. Chỉ chạy sau khi đã xác nhận CloudStack không còn tham chiếu và dữ liệu đã được backup nếu cần giữ lại.

- Nếu cần gỡ toàn bộ cụm (chỉ dùng khi phá bỏ lab, không áp dụng khi chỉ muốn xoá một phần):

```bash
sudo cephadm rm-cluster --fsid <fsid> --force --zap-osds
```

> [!CAUTION]
> `--zap-osds` xoá sạch dữ liệu trên toàn bộ disk OSD của cụm, không thể khôi phục. Không chạy lệnh này nếu chỉ muốn rollback một phần (ví dụ chỉ gỡ Secondary Storage).

## Reference

- [Cephadm - Deploying a new Ceph cluster](https://docs.ceph.com/en/latest/cephadm/install/)
- [Ceph - OSD Service Specification (encrypted, drive groups)](https://docs.ceph.com/en/latest/cephadm/services/osd/)
- [Ceph - CephX authentication and capabilities](https://docs.ceph.com/en/latest/rados/operations/user-management/)
- [Ceph - Messenger v2 protocol (secure mode)](https://docs.ceph.com/en/latest/rados/configuration/msgr2/)
- [Ceph - NFS module / NFS-Ganesha via cephadm](https://docs.ceph.com/en/latest/cephadm/services/nfs/)
- [Ceph - RGW service via cephadm](https://docs.ceph.com/en/latest/cephadm/services/rgw/)
- [Ceph - Ingress service (HA VIP for RGW/NFS)](https://docs.ceph.com/en/latest/cephadm/services/ingress/)
- [Ceph Dashboard - Security and hardening](https://docs.ceph.com/en/latest/mgr/dashboard/)
- [Ceph - Placement Group (PG) count calculator](https://docs.ceph.com/en/latest/rados/operations/placement-groups/)
- [Apache CloudStack - KVM Hypervisor Host Installation](https://docs.cloudstack.apache.org/en/latest/installguide/hypervisor/kvm.html)
- [Apache CloudStack - Add Primary Storage](https://docs.cloudstack.apache.org/en/latest/adminguide/storage.html)
- [Apache CloudStack - Add Secondary Storage](https://docs.cloudstack.apache.org/en/latest/adminguide/storage.html)
