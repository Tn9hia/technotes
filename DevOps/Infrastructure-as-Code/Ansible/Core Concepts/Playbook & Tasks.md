---
title: Playbook & Tasks
tags:
  - ansible
  - playbook
  - modules
  - deep-dive
date: 2026-04-26
---

# Playbook & Tasks

## Playbook Structure

```yaml
# site.yml
---
- name: Configure webservers          # ← Play 1
  hosts: webservers                   # target group từ inventory
  become: true                        # sudo
  gather_facts: true                  # collect host info (OS, IP, ...)
  vars:
    http_port: 80
  vars_files:
    - vars/nginx_vars.yaml

  pre_tasks:                          # chạy trước roles
    - name: Update apt cache
      apt:
        update_cache: true
        cache_valid_time: 3600

  roles:                              # apply roles (theo thứ tự)
    - common
    - nginx

  tasks:                              # tasks thêm sau roles
    - name: Verify nginx running
      service:
        name: nginx
        state: started

  post_tasks:                         # chạy sau tất cả
    - name: Send notification
      debug:
        msg: "Webservers configured"

  handlers:                           # chỉ chạy khi được notify
    - name: restart nginx
      service:
        name: nginx
        state: restarted

- name: Configure databases           # ← Play 2
  hosts: databases
  become: true
  roles:
    - postgresql
```

---

## Task Structure

```yaml
- name: Install nginx                  # mô tả (hiển thị khi chạy)
  apt:                                 # module name
    name: nginx                        # module params
    state: present
  become: true                         # override become per-task
  when: ansible_os_family == "Debian"  # conditional
  register: install_result             # lưu output vào variable
  notify: restart nginx                # trigger handler nếu changed
  tags:
    - nginx
    - install
  ignore_errors: true                  # tiếp tục dù fail
  failed_when: install_result.rc != 0  # custom fail condition
  changed_when: false                  # không report changed
  retries: 3                           # retry nếu fail
  delay: 5                             # giây giữa các retry
  timeout: 60                          # task timeout
  no_log: true                         # ẩn output (dùng cho secret)
```

---

## Handlers

Handler chỉ chạy **khi được notify** và **chỉ chạy 1 lần** dù được notify nhiều lần, vào **cuối play**.

```yaml
tasks:
  - name: Copy nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: restart nginx          # notify handler

  - name: Copy nginx site config
    template:
      src: site.conf.j2
      dest: /etc/nginx/sites-enabled/default
    notify: restart nginx          # notify cùng handler → vẫn chỉ chạy 1 lần

handlers:
  - name: restart nginx
    service:
      name: nginx
      state: restarted

  - name: reload nginx             # reload nhẹ hơn restart
    service:
      name: nginx
      state: reloaded
```

**Force handler chạy giữa chừng:**
```yaml
tasks:
  - name: Some task
    command: echo "hello"
    notify: my handler

  - name: Flush handlers now
    meta: flush_handlers            # chạy handlers ngay tại đây

  - name: Continue with fresh state
    command: echo "after handler"
```

---

## Modules thường dùng

### File & Directory

```yaml
# copy — copy file từ control node lên target
- name: Copy config file
  copy:
    src: files/app.conf         # relative to role/playbook
    dest: /etc/app/app.conf
    owner: root
    group: root
    mode: '0644'
    backup: true                # backup file cũ

# template — render Jinja2 template
- name: Render config
  template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: '0644'
  notify: reload nginx

# file — manage file/dir/symlink/permissions
- name: Create directory
  file:
    path: /opt/myapp/logs
    state: directory            # directory / file / touch / absent / link
    owner: appuser
    group: appuser
    mode: '0755'
    recurse: true               # recursive chmod (chỉ với directory)

- name: Create symlink
  file:
    src: /opt/myapp/current
    dest: /opt/myapp/active
    state: link

- name: Remove file
  file:
    path: /tmp/tempfile
    state: absent

# fetch — pull file từ target về control node
- name: Fetch log file
  fetch:
    src: /var/log/app.log
    dest: ./fetched_logs/       # lưu tại: ./fetched_logs/<hostname>/var/log/app.log
    flat: false
```

### Package Management

```yaml
# apt (Debian/Ubuntu)
- name: Install packages
  apt:
    name:
      - nginx
      - curl
      - git
    state: present              # present / latest / absent
    update_cache: true
    cache_valid_time: 3600      # chỉ update cache nếu cũ hơn 1h

- name: Install specific version
  apt:
    name: nginx=1.24.0-1
    state: present

- name: Remove package
  apt:
    name: apache2
    state: absent
    purge: true                 # xoá cả config files

# dnf/yum (RHEL/CentOS/Rocky)
- name: Install packages
  dnf:
    name:
      - nginx
      - firewalld
    state: present

# package — generic, detect OS tự động
- name: Install package (cross-platform)
  package:
    name: curl
    state: present
```

### Service

```yaml
- name: Start and enable nginx
  service:
    name: nginx
    state: started              # started / stopped / restarted / reloaded
    enabled: true               # enable on boot

# systemd — dùng thay service khi cần systemd-specific
- name: Reload systemd daemon
  systemd:
    daemon_reload: true

- name: Enable and start service
  systemd:
    name: myapp
    state: started
    enabled: true
    masked: false
```

### Command Execution

```yaml
# command — không qua shell, không support pipe/redirect/glob
- name: Run command
  command: /usr/bin/myapp --init
  args:
    chdir: /opt/myapp            # working directory
    creates: /opt/myapp/.initialized   # skip nếu file này tồn tại (idempotency)

# shell — qua bash, support pipe/redirect
- name: Run shell command
  shell: |
    find /tmp -name "*.log" -mtime +7 | xargs rm -f
  args:
    executable: /bin/bash

# raw — SSH trực tiếp, không cần Python (dùng khi bootstrap)
- name: Install python on fresh server
  raw: apt-get install -y python3
  changed_when: true

# script — copy script lên target và chạy
- name: Run local script
  script: scripts/setup.sh arg1 arg2
  args:
    creates: /opt/.setup_done
```

### User & Group

```yaml
- name: Create group
  group:
    name: appgroup
    gid: 1500
    state: present

- name: Create user
  user:
    name: appuser
    uid: 1500
    group: appgroup
    groups:
      - sudo
      - docker
    append: true               # thêm vào groups, không replace
    shell: /bin/bash
    home: /home/appuser
    create_home: true
    password: "{{ vault_user_password }}"
    password_lock: false
    state: present

- name: Add SSH key for user
  authorized_key:
    user: appuser
    key: "{{ lookup('file', 'files/appuser.pub') }}"
    state: present
    exclusive: false           # giữ keys cũ
```

### Cron

```yaml
- name: Add cron job
  cron:
    name: "backup database"    # unique identifier
    minute: "0"
    hour: "2"
    day: "*"
    month: "*"
    weekday: "*"
    job: "/usr/local/bin/backup.sh >> /var/log/backup.log 2>&1"
    user: root
    state: present

- name: Remove cron job
  cron:
    name: "backup database"
    state: absent
```

### Network & URI

```yaml
# get_url — download file
- name: Download binary
  get_url:
    url: https://github.com/release/v1.0/binary
    dest: /usr/local/bin/binary
    mode: '0755'
    checksum: sha256:abc123...  # verify integrity
    timeout: 30

# uri — HTTP request
- name: Wait for API to be ready
  uri:
    url: http://localhost:8080/health
    status_code: 200
    timeout: 10
  register: health_check
  until: health_check.status == 200
  retries: 30
  delay: 10

- name: POST to API
  uri:
    url: http://api.internal/register
    method: POST
    body_format: json
    body:
      name: "{{ inventory_hostname }}"
      env: "{{ environment }}"
    headers:
      Authorization: "Bearer {{ api_token }}"
    status_code: [200, 201]
```

### Debug & Assert

```yaml
- name: Print variable
  debug:
    var: ansible_default_ipv4.address

- name: Print message
  debug:
    msg: "Server {{ inventory_hostname }} is running {{ ansible_distribution }} {{ ansible_distribution_version }}"

- name: Print only when verbose
  debug:
    msg: "Detailed info: {{ result }}"
    verbosity: 2    # chỉ hiện khi chạy với -vv

- name: Assert condition
  assert:
    that:
      - ansible_memtotal_mb >= 2048
      - ansible_distribution in ['Ubuntu', 'Debian']
    fail_msg: "Server không đủ RAM hoặc không phải Debian-based"
    success_msg: "System requirements OK"
```

### Git & Archive

```yaml
- name: Clone repo
  git:
    repo: https://github.com/myorg/myapp.git
    dest: /opt/myapp
    version: v1.2.3             # branch, tag, hoặc commit SHA
    force: false                # không overwrite local changes
    depth: 1                    # shallow clone

- name: Extract archive
  unarchive:
    src: files/app.tar.gz       # local file
    # src: https://example.com/app.tar.gz  # hoặc URL
    dest: /opt/app
    remote_src: false           # true nếu src trên target host
    creates: /opt/app/bin/app   # skip nếu file này tồn tại
```

### File Content Manipulation

```yaml
# lineinfile — đảm bảo 1 dòng tồn tại/không tồn tại trong file
- name: Set max open files in sysctl.conf
  lineinfile:
    path: /etc/security/limits.conf
    regexp: '^appuser\s+soft\s+nofile'   # regex tìm dòng cần thay
    line: 'appuser soft nofile 65536'    # nội dung thay thế
    state: present                       # present / absent
    create: true                         # tạo file nếu chưa có
    backup: true                         # backup trước khi sửa

- name: Remove a config line
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^PermitRootLogin'
    state: absent

# blockinfile — insert/update/remove cả một block văn bản
- name: Add nginx upstream block
  blockinfile:
    path: /etc/nginx/nginx.conf
    marker: "# {mark} ANSIBLE MANAGED — upstream myapp"   # BEGIN/END marker
    insertafter: "http {"
    block: |
      upstream myapp {
          server 127.0.0.1:3000;
          server 127.0.0.1:3001;
      }
    state: present

# replace — regex find-and-replace toàn file
- name: Replace old domain in config
  replace:
    path: /etc/app/config.ini
    regexp: 'old\.domain\.com'
    replace: 'new.domain.com'
    backup: true

# ini_file — manage INI-style config files (sections & keys)
- name: Set database config
  ini_file:
    path: /etc/app/app.ini
    section: database
    option: host
    value: db.internal
    state: present

- name: Remove deprecated option
  ini_file:
    path: /etc/app/app.ini
    section: cache
    option: old_key
    state: absent
```

### File Discovery & Info

```yaml
# stat — lấy thông tin file/dir (thường dùng để check trước khi làm gì đó)
- name: Check if config exists
  stat:
    path: /etc/app/config.yaml
  register: config_stat

- name: Deploy config only if not present
  template:
    src: config.yaml.j2
    dest: /etc/app/config.yaml
  when: not config_stat.stat.exists

- name: Verify file checksum
  stat:
    path: /usr/local/bin/myapp
    checksum_algorithm: sha256
  register: binary_stat

# find — tìm files thỏa điều kiện (kết quả dùng với loop)
- name: Find old log files
  find:
    paths: /var/log/myapp
    patterns: "*.log"
    age: "30d"              # cũ hơn 30 ngày
    recurse: true
  register: old_logs

- name: Delete old logs
  file:
    path: "{{ item.path }}"
    state: absent
  loop: "{{ old_logs.files }}"

# slurp — đọc nội dung file từ target về control node (base64)
- name: Read remote config
  slurp:
    src: /etc/app/secret.conf
  register: remote_config

- name: Print decoded content
  debug:
    msg: "{{ remote_config.content | b64decode }}"
```

### System Configuration

```yaml
# hostname — đặt hostname cho host
- name: Set hostname
  hostname:
    name: "{{ inventory_hostname }}"
    use: systemd              # systemd / debian / redhat / generic

# sysctl — quản lý kernel parameters (/etc/sysctl.conf)
- name: Enable IP forwarding
  sysctl:
    name: net.ipv4.ip_forward
    value: '1'
    sysctl_set: true          # apply ngay lập tức
    reload: true              # reload sysctl sau khi set
    state: present

- name: Tune network buffers
  sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    state: present
  loop:
    - { name: net.core.rmem_max, value: '16777216' }
    - { name: net.core.wmem_max, value: '16777216' }

# mount — quản lý filesystem mounts (/etc/fstab)
- name: Mount NFS share
  mount:
    path: /mnt/data
    src: nas.internal:/export/data
    fstype: nfs
    opts: rw,relatime,vers=4.1
    state: mounted            # mounted / unmounted / present / absent

- name: Ensure swap is disabled
  mount:
    path: none
    src: /dev/sda3
    fstype: swap
    state: absent
```

### Flow Control & Testing

```yaml
# ping — test kết nối Python (không phải ICMP ping)
- name: Test connectivity
  ping:
  # trả về pong nếu ok, thường dùng ở đầu play để verify inventory

# fail — fail play với thông báo tùy chỉnh
- name: Fail if wrong OS
  fail:
    msg: "Playbook này chỉ support Ubuntu 20.04+, hiện tại: {{ ansible_distribution }} {{ ansible_distribution_version }}"
  when: ansible_distribution != 'Ubuntu' or ansible_distribution_major_version | int < 20

# setup — thu thập facts (thường tự động, nhưng có thể gọi thủ công)
- name: Re-gather facts after OS change
  setup:
    gather_subset:
      - network
      - hardware
      - virtual
  # gather_subset: min / all / network / hardware / virtual / ohai / facter

# add_host — thêm host động vào inventory trong runtime
- name: Register new VM into inventory
  add_host:
    name: "{{ new_vm_ip }}"
    groups: newly_created
    ansible_user: ubuntu

# group_by — phân nhóm hosts theo fact
- name: Group by OS
  group_by:
    key: "os_{{ ansible_distribution | lower }}"
  # tạo group os_ubuntu, os_centos, ... dùng được trong play sau
```

---

## register & when kết hợp

```yaml
- name: Check if service exists
  command: systemctl status myapp
  register: myapp_status
  failed_when: false            # không fail nếu service chưa có
  changed_when: false

- name: Install service only if not present
  copy:
    src: myapp.service
    dest: /etc/systemd/system/
  when: myapp_status.rc != 0   # chỉ install nếu service chưa có

- name: Print output of previous task
  debug:
    var: myapp_status.stdout_lines
  when: myapp_status.rc == 0
```

---

## Useful patterns

### Include / Import

```yaml
# import_tasks — static, load tại parse time
- name: Import setup tasks
  import_tasks: tasks/setup.yaml

# include_tasks — dynamic, load tại runtime (support with_items)
- name: Include tasks dynamically
  include_tasks: "tasks/{{ ansible_os_family }}.yaml"

# import_playbook
- import_playbook: playbooks/webservers.yaml
- import_playbook: playbooks/databases.yaml
```

### Set_fact — tạo variable trong runtime

```yaml
- name: Set computed variable
  set_fact:
    app_url: "https://{{ ansible_default_ipv4.address }}:{{ app_port }}"
    is_primary: "{{ inventory_hostname == groups['databases'][0] }}"
    cacheable: true             # persist fact qua tasks
```

### Pause & Wait

```yaml
- name: Wait for service to start
  wait_for:
    port: 8080
    host: localhost
    delay: 5                    # chờ 5s trước khi bắt đầu check
    timeout: 60                 # timeout sau 60s
    state: started              # started / stopped / drained / absent

- name: Pause for manual verification
  pause:
    prompt: "Verify deployment at http://{{ ansible_host }}, then press Enter"
    # hoặc: minutes: 2
```

---

## Gotchas

- **`command` vs `shell`**: `command` không expand `~`, không support `|`, `>`, `&&`. Dùng `shell` khi cần những thứ này, nhưng nhớ thêm `creates:` hoặc `changed_when:` để đảm bảo idempotency.
- **Handler chỉ chạy khi changed**: nếu task `ok` (không changed) → handler không được notify. Nếu muốn force: `changed_when: true`.
- **`notify` tên phải match chính xác**: `notify: restart nginx` phải match `name: restart nginx` trong handlers, kể cả case.
- **`register` và failed task**: nếu task fail và không có `ignore_errors`, variable được register nhưng play dừng. Dùng `failed_when: false` để tiếp tục và check `rc` sau.
- **Template và `dest` là directory**: nếu `dest` là thư mục, Ansible tự append tên file từ `src`. Nên specify đường dẫn đầy đủ.
