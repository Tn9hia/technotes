---
title: Variables & Facts
tags:
  - ansible
  - variables
  - facts
  - deep-dive
date: 2026-04-26
---

# Variables & Facts

## Variable Precedence — Thứ tự ưu tiên (thấp → cao)

```
1.  role defaults          (roles/myrole/defaults/main.yml)
2.  inventory file vars    ([group:vars] trong INI)
3.  inventory group_vars/all
4.  playbook group_vars/all
5.  inventory group_vars/*
6.  playbook group_vars/*
7.  inventory host_vars/*
8.  playbook host_vars/*
9.  host facts / cached set_facts
10. play vars
11. play vars_prompt
12. play vars_files
13. role vars              (roles/myrole/vars/main.yml)
14. block vars
15. task vars
16. include_vars
17. set_facts / registered vars
18. role (and include_role) params
19. include params
20. extra vars             (-e "key=value")   ← HIGHEST
```

**Rule thực tế:**
- `defaults/` = "nếu không ai override, dùng cái này"
- `vars/` = "role cần cái này, không nên override"
- `-e` flag = "luôn thắng tất cả" — dùng để override khi troubleshoot

---

## Khai báo Variables

### Trong playbook

```yaml
- hosts: webservers
  vars:
    http_port: 80
    app_name: myapp
    config:              # nested dict
      timeout: 30
      retries: 3
    packages:            # list
      - nginx
      - curl
```

### vars_files

```yaml
- hosts: webservers
  vars_files:
    - vars/common.yaml
    - vars/{{ env }}.yaml    # dynamic file load
```

```yaml
# vars/common.yaml
http_port: 80
app_version: "1.2.3"
```

### group_vars & host_vars

```yaml
# group_vars/webservers.yaml
nginx_worker_processes: 4
nginx_worker_connections: 1024

# group_vars/all.yaml  (áp dụng cho tất cả)
ntp_servers:
  - 0.pool.ntp.org
  - 1.pool.ntp.org

# host_vars/web01.internal.com.yaml
ansible_host: 10.0.1.10
custom_port: 8080
```

### Extra vars (command line)

```bash
# Highest priority — override tất cả
ansible-playbook site.yml -e "env=production version=v1.2.3"
ansible-playbook site.yml -e @vars/extra.yaml    # từ file
ansible-playbook site.yml -e '{"env":"production","version":"v1.2.3"}'
```

---

## Variable Types & Syntax

```yaml
# String
app_name: "myapp"
app_name: myapp        # quotes không bắt buộc (trừ khi có ký tự đặc biệt)

# Number
port: 8080
timeout: 30.5

# Boolean
debug_mode: true
debug_mode: yes        # yaml boolean aliases
debug_mode: on

# List
packages:
  - nginx
  - curl
  - git
packages: [nginx, curl, git]   # inline

# Dictionary
database:
  host: db01.internal.com
  port: 5432
  name: myapp_prod

# Access nested vars
"{{ database.host }}"           # dot notation
"{{ database['host'] }}"        # bracket notation (safer với key có dấu -)
```

### Variable trong string

```yaml
# Interpolation
app_url: "https://{{ server_name }}/{{ app_path }}"

# Toàn bộ là expression → không cần quotes
port_number: "{{ base_port + 1000 }}"    # arithmetic

# Conditional expression
log_level: "{{ 'debug' if env == 'dev' else 'warning' }}"
```

---

## Facts — Thông tin host tự động

Facts là variables được Ansible tự động thu thập từ host qua `setup` module.

```bash
# Xem tất cả facts của 1 host
ansible web01 -m setup

# Filter facts
ansible web01 -m setup -a "filter=ansible_distribution*"
ansible web01 -m setup -a "filter=ansible_memory_mb"
ansible web01 -m setup -a "filter=ansible_interfaces"
```

### Facts hay dùng

```yaml
# OS
ansible_distribution           # Ubuntu, CentOS, Debian...
ansible_distribution_version   # 22.04, 8.5...
ansible_distribution_major_version  # 22, 8...
ansible_os_family              # Debian, RedHat, Arch...

# Network
ansible_default_ipv4.address   # primary IP
ansible_default_ipv4.interface # primary interface (eth0, ens3...)
ansible_all_ipv4_addresses     # list of all IPs
ansible_hostname               # hostname
ansible_fqdn                   # fully qualified domain name

# Hardware
ansible_processor_count        # số CPU core
ansible_memtotal_mb            # tổng RAM (MB)
ansible_memfree_mb             # RAM còn trống (MB)
ansible_devices                # dict of block devices

# System
ansible_date_time.iso8601      # current time
ansible_env                    # environment variables dict
ansible_user_id                # current user
ansible_python_version         # python version trên host
```

### Custom Facts

Đặt script/file trong `/etc/ansible/facts.d/` trên target host → Ansible tự đọc vào `ansible_local.*`

```bash
# /etc/ansible/facts.d/app.fact
[app]
version=1.2.3
environment=production
deploy_date=2026-04-26
```

```yaml
# Truy cập custom facts
"{{ ansible_local.app.app.version }}"   # ansible_local.<filename>.<section>.<key>
```

```python
#!/usr/bin/env python3
# /etc/ansible/facts.d/myapp.fact
# Script executable → phải output JSON
import json
print(json.dumps({
    "version": open("/opt/myapp/VERSION").read().strip(),
    "pid": open("/var/run/myapp.pid").read().strip()
}))
```

---

## Magic Variables

Variables đặc biệt Ansible tự inject, không cần define:

```yaml
# Host & inventory
inventory_hostname       # tên host như trong inventory (e.g. "web01")
inventory_hostname_short # phần trước dấu chấm đầu tiên
ansible_play_hosts       # list hosts đang active trong play
ansible_play_hosts_all   # list tất cả hosts (kể cả failed)
groups                   # dict của tất cả groups
groups['webservers']     # list hosts trong group webservers
group_names              # list groups mà host hiện tại thuộc về
hostvars                 # dict: hostvars['web01']['ansible_default_ipv4']

# Play & role
ansible_play_name        # tên play hiện tại
role_path                # absolute path của role đang chạy
playbook_dir             # thư mục chứa playbook

# Connection
ansible_host             # IP/hostname kết nối thực tế
ansible_user             # SSH user
ansible_port             # SSH port
```

**Ví dụ hay dùng:**

```yaml
# Kiểm tra host có trong group không
- when: "'databases' in group_names"

# Lấy IP của host khác
- debug:
    msg: "DB host: {{ hostvars['db01']['ansible_default_ipv4']['address'] }}"

# Chạy task chỉ trên host đầu tiên của group
- when: inventory_hostname == groups['webservers'][0]

# Build danh sách tất cả IP trong group
- set_fact:
    all_web_ips: "{{ groups['webservers'] | map('extract', hostvars, ['ansible_default_ipv4', 'address']) | list }}"
```

---

## include_vars — Load vars động

```yaml
# Load vars file theo OS
- name: Load OS-specific vars
  include_vars: "vars/{{ ansible_os_family }}.yaml"

# Load từ thư mục (load tất cả .yaml files)
- name: Load all vars
  include_vars:
    dir: vars/
    extensions: [yaml, yml]
    ignore_unknown_extensions: true

# Load với prefix
- name: Load vars with prefix
  include_vars:
    file: vars/database.yaml
    name: db                  # tất cả vars trong db.* namespace
```

---

## Vault Variables — Secret Management

```yaml
# group_vars/all/vault.yaml (encrypted)
vault_db_password: "supersecret"
vault_api_key: "abc123"

# group_vars/all/main.yaml (plain)
db_password: "{{ vault_db_password }}"   # reference vault var
```

→ Chi tiết: [[Ansible Vault]]

---

## Gotchas

- **Variable không tồn tại → lỗi**: `{{ undefined_var }}` → fail. Dùng `{{ undefined_var | default('fallback') }}` hoặc kiểm tra với `is defined`.
- **Boolean trap**: `"true"` (string) ≠ `true` (boolean). Trong YAML: `enabled: true` (boolean), `enabled: "true"` (string). Conditional `when: enabled` hoạt động khác nhau.
- **Dict merge**: Ansible **không deep-merge** dict variables theo mặc định. `hash_behaviour = merge` trong `ansible.cfg` thay đổi điều này nhưng có side effects. Prefer flat variables hơn nested dict khi cần override.
- **Facts và `gather_facts: false`**: khi tắt gather_facts để tăng tốc → tất cả `ansible_*` facts đều undefined. Nếu vẫn cần 1 số facts, dùng `setup` module với `filter`.
- **`hostvars` chỉ có dữ liệu của host đã gather facts**: nếu host B chưa được gather facts → `hostvars['hostB']` sẽ thiếu facts. Thứ tự play quan trọng.
