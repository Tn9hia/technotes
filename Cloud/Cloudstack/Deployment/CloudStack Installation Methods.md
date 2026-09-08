---
tags:
  - cloudstack
  - deployment
  - installation
---

# CloudStack Installation Methods

## Các cách cài đặt phổ biến

| Phương pháp | Mô tả | Khi nào dùng |
|---|---|---|
| **Package-based (apt/yum) thủ công** | Cài `cloudstack-management`, `cloudstack-agent` từng gói theo tài liệu chính thức | Hiểu sâu từng bước, phù hợp học/production nhỏ được kiểm soát chặt |
| **Terraform / Ansible tự viết** | Automate hoá lại quy trình thủ công | Phổ biến ở production nghiêm túc cần reproducible infra |
| **CloudStack Kubernetes Provider / CAPC** | Triển khai Kubernetes cluster lên trên CloudStack | Khi cần K8s-as-a-Service trên nền CloudStack |
| **Marvin (test framework)** | Framework Python để dựng & test môi trường CloudStack tự động | Chủ yếu dùng bởi contributor/CI, ít dùng cho production thật |

> [!info] CloudStack không có "installer all-in-one chính thức" mạnh như Kolla-Ansible/TripleO bên OpenStack
> Khác OpenStack (có nhiều dự án con chuyên installer như Kolla-Ansible), CloudStack truyền thống được cài qua **package theo tài liệu**, khá thủ công so với chuẩn hiện đại. Nhiều tổ chức tự viết Ansible role riêng để chuẩn hoá — nếu hệ thống bàn giao có sẵn automation nội bộ, đây là tài sản quan trọng cần được bàn giao đầy đủ, đừng để thất lạc.

## Quy trình cài đặt tổng quát (KVM, package-based)

```bash
# --- Trên Management Server (Ubuntu/RHEL) ---
# 1. Cài MySQL/MariaDB, setup DB
apt install mariadb-server
cloudstack-setup-databases cloud:<password>@localhost --deploy-as=root:<rootpass>

# 2. Cài management server
apt install cloudstack-management
cloudstack-setup-management

# 3. Import systemvm template (BẮT BUỘC trước khi tạo Zone)
/usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt \
  -m /export/secondary -u http://download.cloudstack.org/systemvm/.../systemvmtemplate-kvm.qcow2.bz2 \
  -h kvm -F

# --- Trên mỗi KVM Host ---
apt install cloudstack-agent qemu-kvm libvirt-daemon-system
# Cấu hình bridge mạng TRƯỚC (cloudbr0...) rồi mới add host qua UI/API
```

> [!warning] Lesson learned: quên import System VM Template = setup Zone xong nhưng không tạo được VM nào
> Đây là bước **rất hay bị bỏ sót** khi làm theo tài liệu tóm tắt trên mạng — thiếu system VM template khiến SSVM/CPVM/VR không bao giờ khởi tạo được, và triệu chứng chỉ lộ ra **sau khi** đã cấu hình xong Zone/Pod/Cluster/Host (tưởng đã xong nhưng vẫn chưa deploy VM được). Luôn kiểm tra `cmk listTemplates templatefilter=all | grep -i systemvm` trước khi coi setup zone là hoàn tất.

## Thứ tự setup 1 Zone mới (checklist)

```
1. Cài & cấu hình Management Server + DB
2. Import System VM Template đúng hypervisor
3. Tạo Zone (chọn Basic/Advanced networking — KHÔNG đổi được sau!)
4. Tạo Physical Network(s), cấu hình traffic label, isolation method
5. Tạo Pod (dải IP management/system VM)
6. Tạo Cluster (chọn đúng hypervisor)
7. Add Host vào Cluster (bridge mạng phải sẵn sàng trước)
8. Add Primary Storage vào Cluster
9. Add Secondary Storage vào Zone
10. Enable Zone
11. Import/registerTemplate cho các OS template người dùng sẽ dùng
12. Tạo Service/Disk/Network Offering cơ bản
13. Test deploy 1 VM thử trước khi bàn giao cho user
```

> [!tip] Luôn "enable Zone" sau cùng
> CloudStack cho phép cấu hình đầy đủ hạ tầng trong khi Zone ở trạng thái **disabled**, tránh user vô tình deploy VM vào hạ tầng chưa hoàn chỉnh. Chỉ `updateZone allocationstate=Enabled` khi đã test xong toàn bộ luồng (deploy thử VM, network, snapshot) — thói quen tốt cần giữ khi mở rộng thêm Zone mới trong tương lai.

## Infrastructure as Code

CloudStack có **Terraform Provider chính thức** (`cloudstack/cloudstack`) và **Ansible modules** (`ngine_io.cloudstack` hoặc modules cộng đồng) — nên tận dụng nếu hệ thống cần thêm Zone/Offering thường xuyên, thay vì thao tác tay qua UI mỗi lần.

```hcl
provider "cloudstack" {
  api_url    = "https://cloudstack.company.local/client/api"
  api_key    = var.cs_api_key
  secret_key = var.cs_secret_key
}

resource "cloudstack_instance" "web" {
  name             = "web-01"
  service_offering = "Standard-4C8G"
  network_id       = cloudstack_network.web_tier.id
  template         = "ubuntu-22.04"
  zone             = "zone-a"
}
```

---
*Xem thêm: [[CloudStack Deployment Models]] | [[CloudStack Management Server]] | [[Key Configuration Reference]] | [[Cloudstack|CloudStack]]*
