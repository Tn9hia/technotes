---
title: Ansible Inventory
tags:
  - ansible
  - inventory
  - deep-dive
date: 2026-04-26
---

# Inventory — Quản lý danh sách hosts

## Inventory là gì?

Inventory định nghĩa **danh sách host** Ansible sẽ quản lý, cách nhóm chúng, và variables gắn với từng host/group.

---

## Static Inventory

### INI format

```ini
# inventory/hosts

# Host đơn lẻ
192.168.1.10
web01.internal.com

# Group
[webservers]
web01.internal.com
web02.internal.com
192.168.1.11

[databases]
db01.internal.com  ansible_port=5522    # override port
db02.internal.com

# Group of groups
[backend:children]
webservers
databases

# Variables cho cả group
[webservers:vars]
http_port=80
nginx_worker_processes=4

# Host với inline variables
web03.internal.com ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/web03.pem
```

### YAML format (rõ ràng hơn, khuyến nghị)

```yaml
# inventory/hosts.yaml
all:
  children:
    webservers:
      hosts:
        web01.internal.com:
          http_port: 80
          nginx_worker_processes: 4
        web02.internal.com:
      vars:
        nginx_version: "1.24"       # variables chung cho group

    databases:
      hosts:
        db01.internal.com:
          ansible_port: 5522
        db02.internal.com:
      vars:
        pg_version: "15"

    backend:
      children:
        webservers:
        databases:

  vars:
    ansible_user: ubuntu            # áp dụng cho tất cả hosts
    ansible_ssh_private_key_file: ~/.ssh/infra.pem
```

### Inventory directory (multiple files)

```
inventory/
├── production/
│   ├── hosts.yaml          ← prod hosts
│   └── group_vars/
│       └── webservers.yaml ← prod-specific vars
└── staging/
    ├── hosts.yaml          ← staging hosts
    └── group_vars/
        └── webservers.yaml ← staging-specific vars
```

```bash
# Chạy với inventory directory
ansible-playbook site.yml -i inventory/production/
```

---

## Connection Variables quan trọng

```yaml
# Per-host hoặc trong group_vars
ansible_host: 10.0.1.50          # IP thực (nếu hostname khác)
ansible_port: 22                  # SSH port (default: 22)
ansible_user: ubuntu              # SSH user
ansible_password: secret          # SSH password (dùng vault!)
ansible_ssh_private_key_file: ~/.ssh/id_ed25519

# Privilege escalation
ansible_become: true
ansible_become_method: sudo       # sudo, su, pbrun, pfexec
ansible_become_user: root
ansible_become_password: secret   # dùng vault!

# Connection type
ansible_connection: ssh           # ssh, local, docker, winrm
ansible_python_interpreter: /usr/bin/python3   # path python trên host
```

---

## Host Patterns — Chọn host khi chạy

```bash
# Tất cả hosts
ansible all -m ping
ansible '*' -m ping

# Một group
ansible webservers -m ping

# Nhiều group (OR)
ansible webservers:databases -m ping

# Giao giữa 2 group (AND)
ansible "webservers:&staging" -m ping     # webservers VÀ staging

# Trừ host/group
ansible "all:!databases" -m ping          # tất cả trừ databases
ansible "webservers:!web01" -m ping

# Wildcard
ansible "web*" -m ping                    # web01, web02, webservers...

# Regex
ansible "~web[0-9]+" -m ping

# Range
ansible "web[01:05]" -m ping              # web01, web02, web03, web04, web05

# Limit trong playbook
ansible-playbook site.yml --limit "web01,web02"
ansible-playbook site.yml --limit @failed_hosts.txt  # retry file
```

---

## Dynamic Inventory

Khi infrastructure thay đổi liên tục (cloud), static file không thực tế → dynamic inventory **query API** để lấy danh sách host real-time.

### Cách hoạt động

```
ansible-playbook site.yml -i inventory/aws_ec2.yaml
              │
              ▼
      Ansible gọi inventory plugin
              │
              ▼
      Plugin query AWS EC2 API
              │
              ▼
      Trả về JSON {hosts, groups, variables}
              │
              ▼
      Ansible dùng như static inventory
```

### AWS EC2 Dynamic Inventory

```yaml
# inventory/aws_ec2.yaml
plugin: amazon.aws.aws_ec2
regions:
  - ap-southeast-1
  - us-east-1

# Filter chỉ lấy instance đang running
filters:
  instance-state-name: running
  tag:Environment: production     # chỉ lấy instance có tag Environment=production

# Nhóm host theo tag
keyed_groups:
  - key: tags.Role                # group = tag Role value
    prefix: role_
  - key: tags.Environment
    prefix: env_
  - key: placement.availability_zone
    prefix: az_

# Sử dụng private IP
hostnames:
  - private-ip-address

# Variables từ EC2 metadata
compose:
  ansible_host: private_ip_address
  ansible_user: ubuntu
```

```bash
# Test dynamic inventory
ansible-inventory -i inventory/aws_ec2.yaml --list
ansible-inventory -i inventory/aws_ec2.yaml --graph

# Output:
# @all:
#   |--@role_webserver:
#   |  |--10.0.1.10
#   |  |--10.0.1.11
#   |--@role_database:
#   |  |--10.0.2.10
```

### Custom Dynamic Inventory Script

Script bất kỳ có thể làm inventory nếu trả về JSON đúng format:

```python
#!/usr/bin/env python3
# inventory/custom_inventory.py
import json, sys, subprocess

def get_hosts():
    # Query database, CMDB, hoặc bất kỳ source nào
    return {
        "webservers": {
            "hosts": ["10.0.1.10", "10.0.1.11"],
            "vars": {"http_port": 80}
        },
        "databases": {
            "hosts": ["10.0.2.10"]
        },
        "_meta": {
            "hostvars": {
                "10.0.1.10": {"ansible_user": "ubuntu"},
                "10.0.1.11": {"ansible_user": "ubuntu"},
                "10.0.2.10": {"ansible_user": "postgres"}
            }
        }
    }

if __name__ == "__main__":
    if "--list" in sys.argv:
        print(json.dumps(get_hosts()))
    elif "--host" in sys.argv:
        print(json.dumps({}))  # host vars trong _meta là đủ
```

```bash
chmod +x inventory/custom_inventory.py
ansible-inventory -i inventory/custom_inventory.py --list
```

---

## group_vars và host_vars

```
inventory/
├── hosts.yaml
├── group_vars/
│   ├── all.yaml              ← áp dụng cho TẤT CẢ hosts
│   ├── all/                  ← có thể là thư mục (nhiều file)
│   │   ├── main.yaml
│   │   └── vault.yaml        ← encrypted vars
│   ├── webservers.yaml       ← chỉ group webservers
│   └── databases.yaml
└── host_vars/
    ├── web01.internal.com.yaml     ← chỉ host web01
    └── db01.internal.com/          ← có thể là thư mục
        ├── main.yaml
        └── vault.yaml
```

**Variable precedence (thấp → cao):**
```
role defaults < group_vars/all < group_vars/<group> < host_vars < play vars < task vars
```
→ Chi tiết: [[Variables & Facts#Precedence]]

---

## Inventory thực tế — Multiple environment

```
inventory/
├── production/
│   ├── hosts.yaml
│   └── group_vars/
│       ├── all.yaml          env: production, log_level: warning
│       └── webservers.yaml   replicas: 5, instance_type: c5.xlarge
├── staging/
│   ├── hosts.yaml
│   └── group_vars/
│       ├── all.yaml          env: staging, log_level: debug
│       └── webservers.yaml   replicas: 2, instance_type: t3.medium
└── dev/
    └── hosts.yaml
```

```bash
# Deploy lên staging
ansible-playbook site.yml -i inventory/staging/

# Deploy lên prod
ansible-playbook site.yml -i inventory/production/
```

---

## Gotchas

- **`ansible_host` vs hostname**: inventory key là tên Ansible dùng để identify host (không nhất thiết là IP/hostname thực). Set `ansible_host` nếu muốn kết nối đến IP khác.
- **INI group vars**: `[group:vars]` chỉ support string — nếu cần list/dict, dùng YAML format hoặc `group_vars/` file.
- **Dynamic inventory cần credentials**: AWS plugin cần AWS credentials. Set trong `~/.aws/credentials` hoặc environment variables — không hardcode trong inventory file.
- **`_meta` trong custom script**: luôn return `_meta.hostvars` thay vì handle `--host <hostname>` riêng lẻ — tránh N+1 API calls.
