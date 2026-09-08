---
title: Dynamic Inventory
tags:
  - ansible
  - inventory
  - dynamic
  - deep-dive
date: 2026-04-26
---

# Dynamic Inventory — Query Infrastructure APIs

## Khi nào cần Dynamic Inventory?

Static inventory file phù hợp khi infrastructure tĩnh (bare metal, VM cố định). Khi infrastructure thay đổi liên tục (cloud, autoscaling), static file không thực tế:

```
Static:  web01.internal, web02.internal, web03.internal  ← hardcode
Dynamic: query AWS EC2 API → lấy tất cả EC2 đang running với tag Role=webserver
```

---

## Cách hoạt động

```
ansible-playbook site.yml -i inventory/aws_ec2.yaml
              │
              ▼
    Ansible detect file là inventory plugin
    (có field `plugin:` ở đầu file)
              │
              ▼
    Plugin query AWS API / GCP API / ...
              │
              ▼
    Trả về structure giống static inventory:
    {hosts, groups, hostvars}
              │
              ▼
    Ansible proceed bình thường
```

---

## AWS EC2 Plugin

### Cài collection

```bash
ansible-galaxy collection install amazon.aws
pip install boto3 botocore
```

### Config file

```yaml
# inventory/aws_ec2.yaml
plugin: amazon.aws.aws_ec2

# Regions cần query
regions:
  - ap-southeast-1
  - ap-northeast-1

# Credentials (ưu tiên: file này > env vars > ~/.aws/credentials)
# aws_access_key: "{{ lookup('env', 'AWS_ACCESS_KEY_ID') }}"
# aws_secret_key: "{{ lookup('env', 'AWS_SECRET_ACCESS_KEY') }}"
# Tốt hơn: dùng IAM Role hoặc env vars, không hardcode

# Filter instances
filters:
  instance-state-name: running    # chỉ instance đang chạy
  "tag:Environment": production   # chỉ production
  # "tag:Managed": ansible       # optional: chỉ instance có tag Managed=ansible

# Nhóm host theo tag, AZ, ...
keyed_groups:
  - key: tags.Role
    prefix: role
    separator: "_"
    # → group: role_webserver, role_database

  - key: tags.Environment
    prefix: env
    # → group: env_production, env_staging

  - key: placement.availability_zone
    prefix: az
    # → group: az_ap-southeast-1a

  - key: instance_type
    prefix: type
    # → group: type_t3_medium

# Thêm host vào group dựa trên condition
groups:
  webservers: "'webserver' in tags.get('Role', '')"
  databases: "'database' in tags.get('Role', '')"
  large_instances: "instance_type.startswith('c5') or instance_type.startswith('m5')"

# Hostname: dùng private IP thay vì instance ID
hostnames:
  - private-ip-address
  # - tag:Name        # hoặc theo tag Name
  # - dns-name        # public DNS

# Variables tự động gán cho mỗi host
compose:
  ansible_host: private_ip_address
  ansible_user: >-
    'ec2-user' if platform == 'Red Hat Enterprise Linux'
    else 'ubuntu'
  env: tags.Environment | lower
  role: tags.Role | lower
```

### Test

```bash
# Xem toàn bộ inventory
ansible-inventory -i inventory/aws_ec2.yaml --list

# Xem dạng tree
ansible-inventory -i inventory/aws_ec2.yaml --graph

# Output:
# @all:
#   |--@role_webserver:
#   |  |--10.0.1.10
#   |  |--10.0.1.11
#   |--@env_production:
#   |  |--10.0.1.10
#   |  |--10.0.2.10

# Xem vars của 1 host
ansible-inventory -i inventory/aws_ec2.yaml --host 10.0.1.10

# Chạy playbook
ansible-playbook site.yml -i inventory/aws_ec2.yaml
ansible-playbook site.yml -i inventory/aws_ec2.yaml --limit role_webserver
```

---

## GCP Compute Engine Plugin

```bash
ansible-galaxy collection install google.cloud
pip install requests google-auth
```

```yaml
# inventory/gcp_compute.yaml
plugin: google.cloud.gcp_compute

projects:
  - my-gcp-project-id

zones:
  - asia-southeast1-a
  - asia-southeast1-b

filters:
  - status = RUNNING
  - labels.environment = production

keyed_groups:
  - key: labels.role
    prefix: role
  - key: zone
    prefix: zone

hostnames:
  - networkInterfaces[0].networkIP    # private IP

compose:
  ansible_host: networkInterfaces[0].networkIP
  ansible_user: "'ubuntu'"

auth_kind: serviceaccount
service_account_file: /etc/ansible/gcp-credentials.json
```

---

## Custom Inventory Script

Khi không có plugin sẵn (CMDB nội bộ, VMware vSphere, custom API):

```python
#!/usr/bin/env python3
# inventory/cmdb_inventory.py

import argparse
import json
import sys
import requests  # pip install requests


CMDB_URL = "https://cmdb.internal.com/api"
API_TOKEN = "your-token"


def get_inventory():
    """Fetch all hosts from CMDB and build Ansible inventory structure."""
    headers = {"Authorization": f"Bearer {API_TOKEN}"}

    # Query CMDB
    response = requests.get(f"{CMDB_URL}/servers?status=active", headers=headers)
    servers = response.json()

    inventory = {
        "_meta": {"hostvars": {}},
        "all": {"children": []}
    }

    groups = {}

    for server in servers:
        hostname = server["hostname"]
        role = server.get("role", "ungrouped")
        env = server.get("environment", "unknown")

        # Build group names
        role_group = f"role_{role}"
        env_group = f"env_{env}"

        # Add to groups
        for group in [role_group, env_group]:
            if group not in groups:
                groups[group] = {"hosts": []}
            groups[group]["hosts"].append(hostname)

        # Host variables
        inventory["_meta"]["hostvars"][hostname] = {
            "ansible_host": server["ip_address"],
            "ansible_user": server.get("ssh_user", "ubuntu"),
            "server_id": server["id"],
            "datacenter": server.get("datacenter", ""),
            "role": role,
            "environment": env,
        }

    # Merge groups into inventory
    inventory.update(groups)

    # Build "all" children list
    inventory["all"]["children"] = list(groups.keys())

    return inventory


def get_host(hostname):
    """Return variables for a specific host."""
    headers = {"Authorization": f"Bearer {API_TOKEN}"}
    response = requests.get(f"{CMDB_URL}/servers/{hostname}", headers=headers)
    server = response.json()
    return {
        "ansible_host": server["ip_address"],
        "ansible_user": server.get("ssh_user", "ubuntu"),
    }


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--list", action="store_true")
    parser.add_argument("--host", type=str)
    args = parser.parse_args()

    if args.list:
        print(json.dumps(get_inventory(), indent=2))
    elif args.host:
        print(json.dumps(get_host(args.host), indent=2))
    else:
        print(json.dumps({}))
```

```bash
chmod +x inventory/cmdb_inventory.py

# Test
./inventory/cmdb_inventory.py --list | python3 -m json.tool

# Dùng với ansible
ansible-inventory -i inventory/cmdb_inventory.py --graph
ansible-playbook site.yml -i inventory/cmdb_inventory.py
```

---

## Combine static + dynamic

```bash
# Inventory directory — Ansible merge tất cả files
inventory/
├── static_hosts.yaml      ← servers không trên cloud
├── aws_ec2.yaml           ← AWS instances
└── group_vars/
    └── all.yaml

ansible-playbook site.yml -i inventory/
```

---

## Cache Dynamic Inventory

Query API mỗi lần chạy rất chậm → cache kết quả:

```ini
# ansible.cfg
[inventory]
cache = true
cache_plugin = jsonfile
cache_connection = /tmp/ansible_inventory_cache
cache_timeout = 3600        # cache valid 1 giờ
```

```bash
# Force refresh cache
ansible-inventory -i inventory/aws_ec2.yaml --refresh-cache --list
```

---

## Gotchas

- **Credentials cho cloud plugins**: không hardcode trong inventory file. Dùng env vars (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) hoặc IAM Role nếu chạy trên EC2.
- **`_meta.hostvars` quan trọng**: trong custom script, return tất cả host vars trong `_meta.hostvars` thay vì implement `--host` riêng lẻ — tránh N+1 API calls (N hosts = N requests).
- **`keyed_groups` và special characters**: tag `App-Name` tạo group `role_App-Name` — dấu `-` trong group name gây lỗi. Dùng `separator: "_"` và chú ý tag naming convention.
- **Cache stale**: nếu có instance mới nhưng cache chưa expire → không thấy trong inventory. Cân bằng `cache_timeout` với tần suất thay đổi infrastructure.
- **Private IP vs Public IP**: trong môi trường production, luôn dùng private IP. Set `hostnames: [private-ip-address]` và `ansible_host: private_ip_address`.
