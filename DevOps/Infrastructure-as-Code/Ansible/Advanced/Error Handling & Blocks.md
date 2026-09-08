---
title: Error Handling & Blocks
tags:
  - ansible
  - error-handling
  - blocks
  - deep-dive
date: 2026-04-26
---

# Error Handling & Blocks

## Default behavior khi lỗi

```
Task fail
    │
    ├─ Ansible dừng toàn bộ play trên host đó
    ├─ Các host khác vẫn tiếp tục (trừ khi any_errors_fatal)
    └─ Host được thêm vào failed_hosts list
```

---

## `ignore_errors`

```yaml
- name: Check if legacy config exists
  command: cat /etc/oldapp/config
  register: old_config
  ignore_errors: true       # tiếp tục dù task fail

- name: Migrate config if exists
  template:
    src: config.j2
    dest: /etc/newapp/config
  when: old_config.rc == 0
```

**Dùng khi**: biết task có thể fail và đó là behavior mong đợi.

---

## `failed_when` — Custom fail condition

```yaml
# Task không fail dù rc != 0 (grep trả về 1 khi không tìm thấy)
- name: Check if process running
  command: pgrep myapp
  register: pgrep_result
  failed_when: false                     # không bao giờ fail

- name: Fail only on specific condition
  command: /opt/health_check.sh
  register: health
  failed_when:
    - health.rc != 0
    - '"CRITICAL" in health.stdout'      # fail chỉ khi có CRITICAL

# Fail khi output không chứa expected string
- name: Verify deployment
  uri:
    url: http://localhost:8080/health
  register: response
  failed_when: '"status":"ok"' not in response.content
```

---

## `changed_when` — Control changed status

```yaml
# Script luôn exit 0 → Ansible báo "ok" dù có thay đổi thật
# Fix: parse output để biết có changed không
- name: Run idempotent script
  command: /opt/setup.sh
  register: setup_result
  changed_when: '"already configured" not in setup_result.stdout'

# Task không bao giờ thay đổi (chỉ check)
- name: Check service status
  command: systemctl status nginx
  changed_when: false
  failed_when: false

# Custom change detection
- name: Update configuration
  command: /opt/update_config.sh
  register: config_result
  changed_when: config_result.stdout_lines | length > 0
```

---

## `block / rescue / always` — Try/Catch/Finally

```yaml
- name: Deploy application
  block:                           # TRY — thực thi bình thường
    - name: Pull docker image
      docker_image:
        name: registry.internal/myapp:{{ version }}
        source: pull

    - name: Stop old container
      docker_container:
        name: myapp
        state: stopped

    - name: Start new container
      docker_container:
        name: myapp
        image: registry.internal/myapp:{{ version }}
        state: started

  rescue:                          # CATCH — chạy khi bất kỳ task nào trong block fail
    - name: Log failure
      debug:
        msg: "Deployment failed: {{ ansible_failed_task.name }}"

    - name: Rollback to previous version
      docker_container:
        name: myapp
        image: registry.internal/myapp:{{ previous_version }}
        state: started

    - name: Alert on-call
      uri:
        url: https://alerts.internal/deploy-failed
        method: POST
        body_format: json
        body:
          app: myapp
          version: "{{ version }}"
          error: "{{ ansible_failed_result.msg }}"

  always:                          # FINALLY — luôn chạy
    - name: Send deployment notification
      slack:
        token: "{{ slack_token }}"
        msg: "Deployment of {{ version }} {{ 'succeeded' if not ansible_failed_task is defined else 'FAILED' }}"
```

### Variables trong rescue

```yaml
rescue:
  - debug:
      var: ansible_failed_task       # task object bị fail
      # .name, .action, .args

  - debug:
      var: ansible_failed_result     # result của task fail
      # .msg, .rc, .stdout, .stderr
```

---

## `any_errors_fatal`

```yaml
- hosts: webservers
  any_errors_fatal: true    # nếu 1 host fail → dừng tất cả hosts
  tasks:
    - name: Critical task
      command: /opt/critical.sh
```

**Dùng khi**: task có tính chất all-or-nothing (migrate DB, enable maintenance mode).

---

## `max_fail_percentage`

```yaml
- hosts: webservers
  max_fail_percentage: 20    # cho phép tối đa 20% hosts fail
  tasks:
    - name: Rolling update
      apt:
        name: myapp
        state: latest
```

---

## Retry với `until`

```yaml
- name: Wait for database to be ready
  command: pg_isready -h {{ db_host }}
  register: db_ready
  until: db_ready.rc == 0
  retries: 30
  delay: 10
  changed_when: false
```

---

## Xử lý lỗi thực tế — Deployment pattern

```yaml
- name: Zero-downtime deployment
  hosts: webservers
  serial: 1                     # từng server một
  max_fail_percentage: 0        # dừng ngay nếu có server fail

  pre_tasks:
    - name: Remove from load balancer
      uri:
        url: "http://lb.internal/api/server/{{ inventory_hostname }}/disable"
        method: POST
      delegate_to: localhost

    - name: Wait for connections to drain
      wait_for:
        timeout: 30

  tasks:
    - block:
        - name: Deploy new version
          apt:
            name: "myapp={{ new_version }}"
            state: present

        - name: Restart service
          service:
            name: myapp
            state: restarted

        - name: Health check
          uri:
            url: "http://{{ ansible_host }}:8080/health"
            status_code: 200
          retries: 10
          delay: 5
          register: health
          until: health.status == 200

      rescue:
        - name: Rollback on failure
          apt:
            name: "myapp={{ current_version }}"
            state: present
          notify: restart myapp

        - name: Fail play after rollback
          fail:
            msg: "Deployment failed on {{ inventory_hostname }}, rolled back"

  post_tasks:
    - name: Re-add to load balancer
      uri:
        url: "http://lb.internal/api/server/{{ inventory_hostname }}/enable"
        method: POST
      delegate_to: localhost
```

---

## Gotchas

- **`rescue` không "clear" fail status tự động**: nếu rescue thành công nhưng mày muốn play tiếp tục bình thường (không báo fail), dùng `meta: clear_host_errors` cuối rescue block.
- **`always` chạy kể cả khi `rescue` fail**: nếu rescue cũng fail → always vẫn chạy. Đảm bảo always block không phụ thuộc vào trạng thái của rescue.
- **`ignore_errors` vs `failed_when: false`**: `ignore_errors` vẫn ghi vào error summary nhưng play tiếp tục. `failed_when: false` nói "task này không bao giờ fail" — sạch hơn về semantics.
- **`block` không loop được**: không thể dùng `loop:` trên `block:`. Wrap block trong `include_tasks` nếu cần loop qua nhiều block.
