---
tags:
  - openstack
  - operations
  - day2
  - maintenance
---

# Day 2 Operations

Các tác vụ vận hành hàng ngày sau khi OpenStack đã được deploy.

## Adding Compute Node

```bash
# 1. Chuẩn bị server mới (BIOS, OS, network)

# 2. Thêm vào Kolla inventory
vim /etc/kolla/inventory/multinode
# Thêm compute-new vào [compute] group

# 3. Bootstrap server mới
kolla-ansible -i inventory/multinode \
  --limit compute-new \
  bootstrap-servers

# 4. Deploy services lên node mới
kolla-ansible -i inventory/multinode \
  --limit compute-new \
  deploy

# 5. Verify
openstack compute service list | grep compute-new
openstack hypervisor show compute-new
```

## Removing Compute Node

```bash
# 1. Disable nova-compute trên node cần remove
openstack compute service set --disable compute-old nova-compute

# 2. Migrate tất cả VMs đang chạy
openstack server list --all-projects --host compute-old -f value -c ID | while read vm; do
  openstack server migrate --live-migration $vm
done

# 3. Đợi migration xong
watch "openstack server list --all-projects --host compute-old"

# 4. Xóa compute service
openstack compute service delete <service-id>

# 5. Xóa neutron agent
openstack network agent list | grep compute-old
openstack network agent delete <agent-id>

# 6. Undeploy với Kolla
kolla-ansible -i inventory/multinode \
  --limit compute-old \
  destroy
```

## Maintenance Mode

### Đặt compute node vào maintenance

```bash
# 1. Disable nova-compute
openstack compute service set --disable compute-01 nova-compute \
  --reason "Scheduled maintenance 2024-01-15"

# 2. Migrate VMs (live migration)
for vm in $(openstack server list --all-projects --host compute-01 -f value -c ID); do
  echo "Migrating $vm..."
  openstack server migrate --live-migration $vm
done

# 3. Thực hiện maintenance

# 4. Re-enable
openstack compute service set --enable compute-01 nova-compute
```

## Backup & Recovery

### Database Backup

```bash
#!/bin/bash
# Chạy trên controller, schedule daily

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR=/backup/openstack/db/$DATE
mkdir -p $BACKUP_DIR

# MariaDB backup với mariabackup (hot backup)
mariabackup --backup \
  --target-dir=$BACKUP_DIR \
  --user=root \
  --password=$DB_PASSWORD

# Compress
tar czf $BACKUP_DIR.tar.gz -C $(dirname $BACKUP_DIR) $(basename $BACKUP_DIR)
rm -rf $BACKUP_DIR

# Rotate (giữ 7 ngày)
find /backup/openstack/db -name "*.tar.gz" -mtime +7 -delete
```

### Keystone Fernet Key Backup

```bash
# Backup Fernet keys
tar czf /backup/fernet-keys-$(date +%Y%m%d).tar.gz \
  /etc/keystone/fernet-keys/

# Sync giữa controllers
rsync -avz /etc/keystone/fernet-keys/ ctrl2:/etc/keystone/fernet-keys/
rsync -avz /etc/keystone/fernet-keys/ ctrl3:/etc/keystone/fernet-keys/
```

### Config Backup

```bash
# Backup toàn bộ /etc/kolla và /etc/openstack
tar czf /backup/configs-$(date +%Y%m%d).tar.gz \
  /etc/kolla/ \
  /etc/openstack/ \
  /etc/ceph/
```

## Certificate Management

```bash
# Kiểm tra TLS cert hết hạn
openssl s_client -connect controller:5000 2>/dev/null | openssl x509 -noout -dates

# Renew cert (Kolla-Ansible tự generate self-signed)
kolla-ansible -i inventory/multinode certificates

# Deploy cert mới
kolla-ansible -i inventory/multinode deploy \
  --tags common
```

## Image Management

```bash
# Xem image catalog
openstack image list --format table --column Name --column Status --column Size

# Upload image mới
IMAGE_URL="https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img"
curl -L -o /tmp/ubuntu-22.04.img $IMAGE_URL

openstack image create "Ubuntu 22.04" \
  --file /tmp/ubuntu-22.04.img \
  --disk-format qcow2 \
  --container-format bare \
  --public

# Deprecate/retire old image
openstack image set --deactivate "Ubuntu 20.04"
openstack image delete "Ubuntu 20.04"

# Xem orphaned images (không có VM dùng)
# Cần custom script hoặc Ceph tools
rbd ls -p images  # images in Glance
```

## Quota Management

```bash
# Xem quota tất cả projects
for project in $(openstack project list -f value -c ID); do
  name=$(openstack project show $project -f value -c name)
  echo "=== $name ==="
  openstack quota show $project
done

# Set quota cho project mới
openstack quota set \
  --instances 20 \
  --cores 80 \
  --ram 163840 \
  --volumes 50 \
  --gigabytes 2000 \
  --floating-ips 5 \
  new-project
```

## Security: Rotate Passwords

```bash
# Với Kolla-Ansible
# Backup password cũ
cp /etc/kolla/passwords.yml /backup/passwords-$(date +%Y%m%d).yml

# Generate password mới cho service cụ thể
# (edit passwords.yml, set service password = rỗng, rồi genpwd)
kolla-ansible -i inventory/multinode \
  --tags nova \
  reconfigure

# Reconfigure tất cả services (cẩn thận - có downtime)
kolla-ansible -i inventory/multinode reconfigure
```

## Resource Cleanup

```bash
# Xóa VMs bị ERROR
openstack server list --all-projects --status ERROR -f value -c ID | \
  xargs -I {} openstack server delete {}

# Xóa floating IPs không được dùng
openstack floating ip list | grep "None" | awk '{print $2}' | \
  xargs -I {} openstack floating ip delete {}

# Xóa volumes không được attach
openstack volume list --all-projects --status available \
  -f value -c ID | \
  xargs -I {} openstack volume show {} | grep "attachments: \[\]"

# Xóa security group rules không dùng
openstack security group list

# Xóa orphaned ports
openstack port list --device-owner "" -f value -c ID | \
  xargs -I {} openstack port delete {}
```

## Capacity Planning

```bash
# Xem hypervisor capacity
openstack hypervisor stats show

# Per-host usage
openstack hypervisor list --long

# Tính overcommit ratio thực tế
python3 << 'EOF'
import subprocess, json

result = subprocess.run(
    ['openstack', 'hypervisor', 'list', '--long', '-f', 'json'],
    capture_output=True, text=True
)
hosts = json.loads(result.stdout)
for h in hosts:
    if h['State'] == 'up':
        cpu_ratio = h['vCPUs Used'] / h['vCPUs'] if h['vCPUs'] > 0 else 0
        ram_ratio = h['Memory MB Used'] / h['Memory MB'] if h['Memory MB'] > 0 else 0
        print(f"{h['Hypervisor Hostname']}: CPU {cpu_ratio:.1f}x, RAM {ram_ratio:.1f}x")
EOF
```

---
*Xem thêm: [[CLI Commands]] | [[Troubleshooting]] | [[Monitoring & Alerting]] | [[Upgrade Procedure]] | [[OpenStack]]*
