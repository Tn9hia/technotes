---
title: Ansible Roles
tags:
  - ansible
  - roles
  - galaxy
  - deep-dive
date: 2026-04-26
---

# Roles — Reusable Units of Work

## Role là gì?

Role là cách đóng gói tasks, handlers, templates, variables thành một unit có thể tái sử dụng. Thay vì copy-paste tasks giữa các playbook, viết 1 role và dùng ở nhiều nơi.

```
Không dùng role:              Dùng role:
playbooks/
├── webserver.yml             roles/
│   ├── install nginx tasks   └── nginx/
│   ├── config tasks              ├── tasks/
│   └── handler                   ├── handlers/
├── staging.yml               ├── templates/
│   ├── install nginx tasks   ├── defaults/
│   └── (copy-paste)          └── vars/
```

---

## Role Structure — Đầy đủ

```
roles/nginx/
├── tasks/
│   ├── main.yml          ← entry point (luôn được load)
│   ├── install.yml       ← import từ main.yml
│   └── configure.yml
├── handlers/
│   └── main.yml          ← handlers (notify từ bất kỳ task nào trong role)
├── templates/
│   ├── nginx.conf.j2
│   └── vhost.conf.j2
├── files/
│   ├── ssl/
│   │   └── dhparam.pem   ← static files (copy nguyên bản)
│   └── mime.types
├── vars/
│   └── main.yml          ← vars HIGH priority (không nên override)
├── defaults/
│   └── main.yml          ← vars LOW priority (designed to be overridden)
├── meta/
│   └── main.yml          ← role metadata, dependencies
└── README.md
```

### `tasks/main.yml`

```yaml
# roles/nginx/tasks/main.yml
---
- name: Import install tasks
  import_tasks: install.yml
  tags: [nginx, install]

- name: Import configure tasks
  import_tasks: configure.yml
  tags: [nginx, config]
```

```yaml
# roles/nginx/tasks/install.yml
---
- name: Install nginx
  package:
    name: nginx
    state: present
  notify: restart nginx

- name: Ensure nginx directories exist
  file:
    path: "{{ item }}"
    state: directory
    owner: root
    group: root
    mode: '0755'
  loop:
    - /etc/nginx/sites-available
    - /etc/nginx/sites-enabled
    - /var/log/nginx
```

### `handlers/main.yml`

```yaml
# roles/nginx/handlers/main.yml
---
- name: restart nginx
  service:
    name: nginx
    state: restarted

- name: reload nginx
  service:
    name: nginx
    state: reloaded

- name: validate nginx config
  command: nginx -t
  changed_when: false
```

### `defaults/main.yml` — Override được

```yaml
# roles/nginx/defaults/main.yml
# Mọi variable ở đây đều có thể bị override
nginx_port: 80
nginx_server_name: "_"
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_client_max_body_size: 10m
nginx_keepalive_timeout: 65
nginx_access_log: /var/log/nginx/access.log
nginx_error_log: /var/log/nginx/error.log
nginx_gzip: true
nginx_ssl_enabled: false
nginx_ssl_certificate: ""
nginx_ssl_certificate_key: ""
```

### `vars/main.yml` — Không nên override

```yaml
# roles/nginx/vars/main.yml
# Internal vars — không expose để override
_nginx_config_path: /etc/nginx/nginx.conf
_nginx_sites_available: /etc/nginx/sites-available
_nginx_sites_enabled: /etc/nginx/sites-enabled
```

### `meta/main.yml` — Dependencies

```yaml
# roles/nginx/meta/main.yml
galaxy_info:
  author: myorg
  description: Install and configure Nginx
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: Ubuntu
      versions: [22.04, 24.04]
    - name: Debian
      versions: [11, 12]

dependencies:
  - role: common              # install common packages trước
  - role: ssl_certs           # cần SSL certs nếu SSL enabled
    when: nginx_ssl_enabled   # conditional dependency
```

---

## Sử dụng Role trong Playbook

```yaml
# playbook đơn giản
- hosts: webservers
  roles:
    - common
    - nginx
    - myapp

# Với variables override
- hosts: webservers
  roles:
    - role: nginx
      vars:
        nginx_port: 8080
        nginx_worker_processes: 4

# include_role — dynamic (runtime)
- hosts: webservers
  tasks:
    - name: Apply nginx role
      include_role:
        name: nginx
      vars:
        nginx_port: "{{ custom_port }}"

# import_role — static (parse time)
- hosts: webservers
  tasks:
    - import_role:
        name: nginx
      vars:
        nginx_port: 8080
```

---

## Path resolution trong Role

```yaml
# Trong tasks — relative paths tự động resolve đến role
- copy:
    src: ssl/dhparam.pem         # → roles/nginx/files/ssl/dhparam.pem
    dest: /etc/nginx/ssl/

- template:
    src: nginx.conf.j2           # → roles/nginx/templates/nginx.conf.j2
    dest: /etc/nginx/nginx.conf

# Không cần prefix đường dẫn → cleaner code
```

---

## Ansible Galaxy — Community Roles

### Tìm và cài role

```bash
# Tìm role
ansible-galaxy search nginx
ansible-galaxy search --author geerlingguy nginx

# Xem info
ansible-galaxy info geerlingguy.nginx

# Cài role
ansible-galaxy install geerlingguy.nginx
ansible-galaxy install geerlingguy.nginx,v3.2.0   # specific version

# Cài vào thư mục cụ thể
ansible-galaxy install geerlingguy.nginx -p ./roles/
```

### `requirements.yml` — Quản lý dependencies

```yaml
# requirements.yml
---
roles:
  # Từ Galaxy
  - name: geerlingguy.nginx
    version: "3.2.0"

  - name: geerlingguy.docker
    version: "7.0.0"

  # Từ GitHub
  - name: myrole
    src: https://github.com/myorg/ansible-role-myrole
    version: main
    scm: git

  # Từ Git tag
  - name: ssl_certs
    src: git+https://github.com/myorg/ansible-role-ssl.git
    version: v1.2.0

collections:
  - name: amazon.aws
    version: ">=6.0.0"
  - name: community.general
    version: ">=7.0.0"
```

```bash
# Cài tất cả dependencies
ansible-galaxy install -r requirements.yml -p ./roles/
ansible-galaxy collection install -r requirements.yml

# CI/CD: cài vào thư mục project (không cần sudo)
ansible-galaxy install -r requirements.yml --roles-path ./roles/
```

---

## Role best practices

### 1. Role chỉ làm 1 việc

```
# Tốt
roles/nginx/          ← chỉ install + configure nginx
roles/ssl_certs/      ← chỉ manage SSL certificates
roles/firewall/       ← chỉ manage firewall rules

# Không tốt
roles/webserver/      ← install nginx + SSL + firewall + logrotate (quá nhiều)
```

### 2. Tất cả biến configurable nên có defaults

```yaml
# defaults/main.yml — luôn có giá trị mặc định hợp lý
nginx_port: 80         # sensible default
nginx_ssl: false       # safe default (opt-in, không opt-out)
```

### 3. Idempotent tasks — chạy nhiều lần không thay đổi kết quả

```yaml
# Tốt — idempotent
- file:
    path: /etc/nginx/ssl
    state: directory

# Không tốt — chạy lại sẽ fail nếu dir đã có
- command: mkdir /etc/nginx/ssl
```

### 4. Handlers đặt tên rõ ràng

```yaml
# Tốt
- name: restart nginx
- name: reload nginx
- name: validate nginx config

# Không tốt
- name: handler1
- name: do thing
```

### 5. Prefix biến với tên role

```yaml
# Tránh collision với biến từ role khác
nginx_port: 80
nginx_worker_processes: 4

# Không phải
port: 80              # conflict với role khác cũng dùng 'port'
```

### 6. README.md đầy đủ

```markdown
# Role: nginx

## Variables

| Variable | Default | Description |
|---|---|---|
| nginx_port | 80 | HTTP listen port |
| nginx_ssl | false | Enable HTTPS |

## Example Playbook

```yaml
- hosts: webservers
  roles:
    - role: nginx
      nginx_port: 8080
```

## Dependencies
- role: common
```

---

## Testing role với Molecule

→ Chi tiết: [[Molecule — Testing Roles]]

---

## Gotchas

- **`defaults` vs `vars`**: biến trong `vars/` có priority cao hơn inventory/playbook vars → khó override. Nếu muốn user tùy chỉnh được → dùng `defaults/`. Chỉ dùng `vars/` cho internal vars role cần nhưng không muốn ai override.
- **Role dependency chạy 1 lần**: nếu nhiều role cùng depend vào `common`, `common` chỉ chạy 1 lần. Nếu muốn chạy lại → dùng `allow_duplicates: true` trong `meta/main.yml`.
- **`include_role` và tags**: tags trong playbook không propagate vào `include_role` (dynamic). Dùng `import_role` nếu cần tags hoạt động đúng.
- **`ansible-galaxy install` overwrite**: mặc định không overwrite role đã cài. Thêm `--force` để update, hoặc xoá thư mục role cũ trước.
