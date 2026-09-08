---
title: Ansible - Overview
tags:
  - ansible
  - iac
  - infra
  - GlobalTechJSC
date: 2026-04-26
status: in-progress
---

# Ansible — Overview

Tags: #ansible #iac #infra
Last updated: 2026-04-26

---

## 1. What — Ansible là gì?

**Ansible** là tool automation agentless — tự động hóa việc cấu hình server, deploy application, và quản lý infrastructure. Không cần cài agent trên target host, chỉ cần SSH (Linux) hoặc WinRM (Windows).

**3 thứ Ansible làm tốt:**
- **Configuration management**: cài package, sửa config file, đảm bảo state của server
- **Application deployment**: deploy code, restart service theo đúng thứ tự
- **Infrastructure orchestration**: provision VM, cấu hình network, phối hợp nhiều host

> Ansible là **imperative tool với declarative intent** — mày viết "state mày muốn" (install nginx), Ansible xác định cách đạt đến đó và đảm bảo idempotency.

---

## 2. Why — Tại sao Ansible, không phải shell script?

| | Shell Script | Ansible |
|---|---|---|
| Idempotency | Phải tự handle | Built-in (chạy 10 lần = 1 lần) |
| Error handling | `set -e`, trap | `failed_when`, `block/rescue` |
| Parallel execution | Phức tạp | Mặc định (configurable) |
| Inventory | Hardcode | Dynamic, group-based |
| Reusability | Copy-paste | Roles, Galaxy |
| Audit | Log thủ công | Verbose output, `--check` dry-run |
| Windows support | Không | WinRM |

**Idempotency là gì:**
```yaml
# Chạy 10 lần → kết quả như nhau, không lỗi
- name: Install nginx
  apt:
    name: nginx
    state: present   # "đảm bảo nginx installed"
                     # nếu đã có → skip, không install lại
```

---

## 3. Architecture — Agentless model

```
┌────────────────────────────────────────────┐
│           Control Node                     │
│   (máy chạy ansible-playbook command)      │
│                                            │
│  ┌──────────┐  ┌───────────┐  ┌─────────┐ │
│  │ Inventory│  │ Playbooks │  │  Roles  │ │
│  └──────────┘  └───────────┘  └─────────┘ │
│                                            │
│  ansible-playbook site.yml                 │
└───────┬────────────────────────────────────┘
        │  SSH (Linux) / WinRM (Windows)
        │  Copy Python modules → /tmp/ → execute → cleanup
        │
   ┌────▼────┐  ┌─────────┐  ┌─────────┐
   │ Host 1  │  │ Host 2  │  │ Host N  │
   │(no agent│  │(no agent│  │(no agent│
   │needed)  │  │needed)  │  │needed)  │
   └─────────┘  └─────────┘  └─────────┘
```

**Cách Ansible execute task:**
1. Kết nối SSH đến target host
2. Copy Python module lên `/tmp/ansible-xxx/`
3. Execute module
4. Đọc JSON output (changed/ok/failed)
5. Cleanup temp files

**Requirements trên target host:**
- SSH access (key-based preferred)
- Python 3.x (hầu hết distro đã có)
- `sudo` nếu cần privilege escalation

---

## 4. Core Components

```
ansible.cfg       ← global config (timeout, forks, inventory path...)
     │
     ├── inventory/    ← WHAT hosts to manage
     │     ├── hosts   (static: IP, hostname, groups)
     │     └── aws_ec2.yaml (dynamic: query AWS API)
     │
     ├── playbooks/    ← WHAT to do
     │     └── site.yml
     │           └── plays (target hosts + tasks)
     │                 └── tasks (gọi modules)
     │
     ├── roles/        ← reusable units of work
     │     └── nginx/
     │           ├── tasks/
     │           ├── handlers/
     │           ├── templates/
     │           └── defaults/
     │
     └── group_vars/   ← variables per group
           ├── all.yml
           └── webservers.yml
```

---

## 5. ansible.cfg — Config quan trọng

```ini
[defaults]
inventory          = ./inventory          # default inventory path
remote_user        = ubuntu               # SSH user mặc định
private_key_file   = ~/.ssh/id_ed25519    # SSH key
host_key_checking  = False                # tắt strict host key check (lab only)
forks              = 10                   # số host parallel (default: 5)
timeout            = 30                   # SSH connection timeout
retry_files_enabled = False               # không tạo .retry file
stdout_callback    = yaml                 # output format đẹp hơn
gathering          = smart                # cache facts, chỉ gather nếu chưa có

[privilege_escalation]
become             = True                 # dùng sudo mặc định
become_method      = sudo
become_user        = root
become_ask_pass    = False

[ssh_connection]
ssh_args           = -o ControlMaster=auto -o ControlPersist=60s
pipelining         = True                 # giảm SSH round-trips → nhanh hơn
```

**`pipelining = True`** là một trong những optimization quan trọng nhất — giảm đáng kể thời gian chạy playbook.

---

## 6. Ad-hoc commands — Chạy nhanh không cần playbook

```bash
# Ping all hosts
ansible all -m ping

# Chạy command
ansible webservers -m command -a "uptime"
ansible webservers -m shell -a "df -h | grep /dev/sda"

# Copy file
ansible all -m copy -a "src=./file.conf dest=/etc/file.conf"

# Install package
ansible all -m apt -a "name=nginx state=present" --become

# Restart service
ansible webservers -m service -a "name=nginx state=restarted" --become

# Gather facts
ansible web01 -m setup
ansible web01 -m setup -a "filter=ansible_distribution*"

# Check mode (dry-run)
ansible all -m apt -a "name=nginx state=latest" --check

# Limit đến subset hosts
ansible all -m ping --limit "web01,web02"
ansible all -m ping --limit "webservers:!db01"  # webservers trừ db01
```

---

## 7. Key Concepts liên kết

- [[Inventory]] — static, dynamic, groups, host patterns
- [[Playbook & Tasks]] — play structure, common modules, handlers
- [[Variables & Facts]] — precedence, group_vars, magic vars, facts
- [[Control Flow]] — when, loops, tags, error handling
- [[Roles]] — structure, dependencies, Galaxy
- [[Jinja2 Templating]] — filters, conditionals trong template
- [[Ansible Vault]] — encrypt secrets
- [[Dynamic Inventory]] — AWS/GCP/custom scripts
- [[Molecule — Testing Roles]] — unit test roles

---

## 8. Execution flow tổng thể

```
ansible-playbook site.yml -i inventory/ --tags "nginx" --limit "prod"
        │
        ├─ Parse ansible.cfg
        ├─ Load inventory → build host groups
        ├─ Filter: --limit + --tags
        │
        ├─ For each play in site.yml:
        │   ├─ Gather facts (nếu gather_facts: true)
        │   ├─ For each task (parallel across hosts, forks=10):
        │   │   ├─ Check conditions (when:)
        │   │   ├─ Execute module via SSH
        │   │   ├─ Collect result (changed/ok/failed/skipped)
        │   │   └─ Trigger handlers nếu changed
        │   └─ Run notified handlers (cuối play)
        │
        └─ Print PLAY RECAP
```

---

## 9. Gotchas

- **Không có state file**: Ansible không track "đã làm gì trước đây" như Terraform. Mỗi lần chạy đều re-check toàn bộ. → Idempotency phải đảm bảo ở module level.
- **`command` vs `shell`**: `command` không qua shell, không support pipe/redirect. `shell` support nhưng kém idempotent hơn. Prefer dùng module chuyên biệt (apt, copy, template...) thay vì shell.
- **`forks` và connection limit**: tăng forks nhưng SSH server có `MaxStartups` limit → có thể bị reject. Cân bằng forks với target server capacity.
- **fact caching**: mặc định gather facts mỗi lần → slow với nhiều host. Dùng `gathering = smart` + fact cache (jsonfile hoặc redis).

---

## 10. Resources

- [Ansible Docs](https://docs.ansible.com/)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html)
- [Ansible Galaxy](https://galaxy.ansible.com/)
- [Jeff Geerling's Ansible for DevOps](https://www.ansiblefordevops.com/)
