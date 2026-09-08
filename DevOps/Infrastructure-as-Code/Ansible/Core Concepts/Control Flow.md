---
title: Control Flow — Conditionals, Loops, Tags
tags:
  - ansible
  - conditionals
  - loops
  - tags
  - deep-dive
date: 2026-04-26
---

# Control Flow — Conditionals, Loops, Tags

## Conditionals — `when`

`when` nhận Jinja2 expression, **không cần** dấu `{{ }}`.

```yaml
# So sánh
- when: ansible_distribution == "Ubuntu"
- when: ansible_distribution_major_version | int >= 20
- when: http_port == 80

# Boolean
- when: debug_mode
- when: not debug_mode
- when: debug_mode is true

# Defined / undefined
- when: my_var is defined
- when: my_var is undefined
- when: my_var | default('') != ''

# String contains
- when: "'nginx' in installed_packages"
- when: ansible_hostname is search("web")     # regex match

# List contains
- when: "'databases' in group_names"          # host thuộc group databases

# AND (tất cả phải đúng)
- when:
    - ansible_os_family == "Debian"
    - ansible_distribution_version is version('20.04', '>=')

# OR
- when: ansible_distribution == "Ubuntu" or ansible_distribution == "Debian"
- when: >
    ansible_distribution == "Ubuntu" or
    ansible_distribution == "Debian"

# Kết hợp
- when: >
    (ansible_distribution == "Ubuntu" and
     ansible_distribution_major_version | int >= 20) or
    ansible_distribution == "Debian"
```

### `when` với `register`

```yaml
- name: Check if file exists
  stat:
    path: /opt/myapp/installed
  register: app_installed

- name: Install app only if not present
  command: ./install.sh
  when: not app_installed.stat.exists

- name: Check command result
  command: grep "ready" /var/log/app.log
  register: grep_result
  failed_when: false

- name: Act on result
  debug:
    msg: "App is ready"
  when: grep_result.rc == 0
```

### `failed_when` & `changed_when`

```yaml
# Custom fail condition
- name: Run script
  command: /opt/check.sh
  register: result
  failed_when:
    - result.rc != 0
    - '"error" in result.stderr'

# Luôn report changed (task thường không report)
- name: Send webhook
  uri:
    url: https://hooks.example.com/deploy
    method: POST
  changed_when: true

# Không bao giờ report changed
- name: Check status
  command: systemctl status nginx
  changed_when: false
  failed_when: false
```

---

## Loops

### `loop` — cơ bản

```yaml
# Loop qua list đơn giản
- name: Install packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - curl
    - git

# Hoặc từ variable
- name: Install packages
  apt:
    name: "{{ item }}"
    state: present
  loop: "{{ required_packages }}"

# Loop qua dict
- name: Create users
  user:
    name: "{{ item.name }}"
    uid: "{{ item.uid }}"
    shell: "{{ item.shell | default('/bin/bash') }}"
  loop:
    - { name: alice, uid: 1001 }
    - { name: bob,   uid: 1002, shell: /bin/sh }
    - { name: carol, uid: 1003 }
```

### `loop` với `loop_control`

```yaml
- name: Process items
  debug:
    msg: "Processing {{ item.name }}"
  loop: "{{ users }}"
  loop_control:
    label: "{{ item.name }}"     # chỉ hiện name thay vì full item dict
    index_var: idx               # biến đếm index (0-based)
    loop_var: user               # đổi tên 'item' thành 'user' (tránh conflict nested loop)
    pause: 1                     # chờ 1s giữa các iteration
```

### Nested loops

```yaml
- name: Create directories for each user
  file:
    path: "/home/{{ item.0 }}/{{ item.1 }}"
    state: directory
  with_nested:
    - [alice, bob]
    - [documents, downloads, music]
  # → tạo: alice/documents, alice/downloads, alice/music, bob/documents, ...
```

### Loop với dict

```yaml
# with_dict (legacy)
- name: Set multiple sysctl values
  sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
  with_dict:
    net.ipv4.ip_forward: 1
    net.ipv6.conf.all.disable_ipv6: 0

# dict2items (modern)
- name: Set sysctl
  sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
  loop: "{{ sysctl_settings | dict2items }}"
  vars:
    sysctl_settings:
      net.ipv4.ip_forward: 1
      net.ipv6.conf.all.disable_ipv6: 0
```

### Loop với `until` — retry

```yaml
- name: Wait for service to be ready
  uri:
    url: http://localhost:8080/health
    status_code: 200
  register: result
  until: result.status == 200
  retries: 30           # thử tối đa 30 lần
  delay: 10             # chờ 10s giữa mỗi lần
```

### with_fileglob & with_items

```yaml
# Copy nhiều files
- name: Copy all config files
  copy:
    src: "{{ item }}"
    dest: /etc/myapp/
  with_fileglob:
    - "files/config/*.conf"

# with_items (legacy — dùng loop thay)
- name: Old style loop
  debug:
    msg: "{{ item }}"
  with_items:
    - one
    - two
    - three
```

---

## Tags — Chạy subset tasks

### Gán tags

```yaml
- name: Install nginx
  apt:
    name: nginx
    state: present
  tags:
    - nginx
    - install
    - packages

- name: Configure nginx
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  tags:
    - nginx
    - config

- name: Start nginx
  service:
    name: nginx
    state: started
  tags:
    - nginx
    - service
```

### Tags trên play / role / block

```yaml
- hosts: webservers
  tags: webservers          # tag cả play

  roles:
    - role: nginx
      tags: [nginx, web]    # tag cả role

  tasks:
    - block:
        - name: task 1
        - name: task 2
      tags: myblock          # tag cả block
```

### Chạy với tags

```bash
# Chỉ chạy tasks có tag nginx
ansible-playbook site.yml --tags nginx

# Chạy nhiều tags (OR)
ansible-playbook site.yml --tags "nginx,config"

# Bỏ qua tags
ansible-playbook site.yml --skip-tags "install,packages"

# Special tags
ansible-playbook site.yml --tags all        # tất cả (default)
ansible-playbook site.yml --tags tagged     # chỉ tasks có ít nhất 1 tag
ansible-playbook site.yml --tags untagged   # chỉ tasks không có tag

# List tasks sẽ chạy (không execute)
ansible-playbook site.yml --tags nginx --list-tasks
```

### Tags best practice

```yaml
# Mỗi role nên có tag riêng
# Phân loại theo action
tags:
  - nginx          # component name
  - install        # action: install / config / service / deploy
  - production     # environment (ít dùng)

# Trong CI/CD:
# Deploy toàn bộ:  --tags all
# Chỉ update config: --tags config
# Chỉ restart service: --tags service
```

---

## `any_errors_fatal` & `max_fail_percentage`

```yaml
- hosts: webservers
  any_errors_fatal: true    # dừng TẤT CẢ hosts nếu 1 host fail

- hosts: webservers
  max_fail_percentage: 20   # cho phép tối đa 20% hosts fail
                             # nếu vượt quá → dừng toàn bộ
```

---

## Serial — Rolling deployment

```yaml
- hosts: webservers
  serial: 1              # deploy từng host một (zero-downtime)
  # serial: 2            # 2 hosts cùng lúc
  # serial: "30%"        # 30% hosts mỗi batch
  # serial: [1, 5, 10%]  # batch 1, rồi 5, rồi 10%

  tasks:
    - name: Remove from load balancer
      # ...
    - name: Deploy new version
      # ...
    - name: Add back to load balancer
      # ...
```

---

## run_once & delegate_to

```yaml
# Chỉ chạy 1 lần, trên 1 host đại diện
- name: Run DB migration
  command: python manage.py migrate
  run_once: true           # chạy trên host đầu tiên của play
  delegate_to: "{{ groups['databases'][0] }}"   # delegate đến host khác

# delegate_to: localhost — chạy trên control node
- name: Send Slack notification
  uri:
    url: https://hooks.slack.com/...
    method: POST
  delegate_to: localhost
  run_once: true
```

---

## Gotchas

- **`loop` với `apt` module**: cách hiệu quả hơn là pass list trực tiếp vào `name:` thay vì loop — apt install tất cả trong 1 lần gọi.
  ```yaml
  # Tốt hơn
  - apt:
      name: [nginx, curl, git]
      state: present
  # Thay vì loop từng cái
  ```
- **`when` trên loop**: `when` được evaluate **cho từng iteration**. Nếu muốn skip toàn bộ loop, đặt `when` ở ngoài task.
- **Tags không inherit xuống include_tasks**: `import_tasks` inherit tags, `include_tasks` thì không. Phải tag explicit trong included file nếu cần.
- **`serial` và handlers**: handlers chạy cuối **mỗi batch**, không phải cuối toàn bộ play. Thường là behavior mong muốn (restart nginx sau mỗi batch) nhưng cần lưu ý.
