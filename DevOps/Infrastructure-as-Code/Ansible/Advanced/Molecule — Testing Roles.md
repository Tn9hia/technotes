---
title: Molecule — Testing Roles
tags:
  - ansible
  - molecule
  - testing
  - deep-dive
date: 2026-04-26
---

# Molecule — Test Ansible Roles

## Molecule là gì?

Molecule là framework test cho Ansible roles. Nó tự động:
1. Tạo môi trường test (Docker container hoặc VM)
2. Apply role vào môi trường đó
3. Chạy test assertions (dùng Testinfra hoặc Ansible verify)
4. Destroy môi trường sau khi test

```
Role code → Molecule → Docker container → Apply role → Run tests → Pass/Fail → Destroy
```

---

## Cài đặt

```bash
pip install molecule molecule-plugins[docker]
# Hoặc với đầy đủ dependencies
pip install molecule molecule-plugins[docker] pytest-testinfra docker

# Kiểm tra
molecule --version
```

---

## Init Molecule vào role

```bash
# Init scenario mặc định (trong thư mục role)
cd roles/nginx
molecule init scenario

# Init với driver cụ thể
molecule init scenario --driver-name docker
molecule init scenario --driver-name vagrant

# Init scenario bổ sung
molecule init scenario --scenario-name debian   # test trên Debian
```

---

## Cấu trúc sau khi init

```
roles/nginx/
├── tasks/
├── handlers/
├── templates/
├── defaults/
└── molecule/
    └── default/                    ← scenario "default"
        ├── molecule.yml            ← cấu hình chính
        ├── converge.yml            ← playbook apply role
        ├── verify.yml              ← test assertions
        └── prepare.yml             ← optional: setup trước khi apply role
```

---

## `molecule.yml` — Cấu hình

```yaml
# molecule/default/molecule.yml
---
dependency:
  name: galaxy
  options:
    requirements-file: requirements.yml   # cài role dependencies

driver:
  name: docker

platforms:
  # Test trên nhiều OS
  - name: ubuntu-22.04
    image: geerlingguy/docker-ubuntu2204-ansible:latest
    pre_build_image: true              # dùng image có sẵn (không build)
    command: /lib/systemd/systemd     # init system cho systemd support
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
    privileged: true                   # cần cho systemd

  - name: debian-12
    image: geerlingguy/docker-debian12-ansible:latest
    pre_build_image: true
    command: /lib/systemd/systemd
    privileged: true

  - name: rockylinux-9
    image: geerlingguy/docker-rockylinux9-ansible:latest
    pre_build_image: true
    command: /lib/systemd/systemd
    privileged: true

provisioner:
  name: ansible
  inventory:
    host_vars:
      ubuntu-22.04:
        nginx_port: 8080
      debian-12:
        nginx_port: 80

lint: |
  set -e
  yamllint .
  ansible-lint

verifier:
  name: ansible    # dùng ansible tasks để verify
  # name: testinfra  # hoặc dùng Testinfra (Python)
```

---

## `converge.yml` — Apply role

```yaml
# molecule/default/converge.yml
---
- name: Converge
  hosts: all
  become: true
  vars:
    nginx_port: 80
    nginx_worker_processes: 2

  roles:
    - role: "{{ lookup('env', 'MOLECULE_PROJECT_DIRECTORY') | basename }}"
    # Lấy tên role từ tên thư mục project
```

---

## `verify.yml` — Test với Ansible tasks

```yaml
# molecule/default/verify.yml
---
- name: Verify
  hosts: all
  become: true
  gather_facts: false

  tasks:
    - name: Check nginx is installed
      package:
        name: nginx
        state: present
      check_mode: true                 # dry-run: không thay đổi, chỉ check
      register: nginx_pkg
      failed_when: nginx_pkg.changed   # fail nếu package chưa install

    - name: Check nginx service is running
      service:
        name: nginx
        state: started
        enabled: true
      check_mode: true
      register: nginx_svc
      failed_when: nginx_svc.changed

    - name: Check nginx is listening on port 80
      wait_for:
        port: 80
        host: localhost
        timeout: 5

    - name: Check nginx config is valid
      command: nginx -t
      changed_when: false

    - name: HTTP request to nginx
      uri:
        url: http://localhost:80
        status_code: 200
      register: nginx_response

    - name: Verify nginx config file exists
      stat:
        path: /etc/nginx/nginx.conf
      register: nginx_conf
      failed_when: not nginx_conf.stat.exists

    - name: Check nginx config content
      command: grep "worker_processes" /etc/nginx/nginx.conf
      register: wp_check
      changed_when: false
      failed_when: wp_check.rc != 0
```

---

## Verify với Testinfra (Python)

```python
# molecule/default/tests/test_nginx.py
import pytest


def test_nginx_installed(host):
    """Nginx package should be installed."""
    nginx = host.package("nginx")
    assert nginx.is_installed


def test_nginx_service_running(host):
    """Nginx service should be running and enabled."""
    service = host.service("nginx")
    assert service.is_running
    assert service.is_enabled


def test_nginx_listening_on_port_80(host):
    """Nginx should listen on port 80."""
    socket = host.socket("tcp://0.0.0.0:80")
    assert socket.is_listening


def test_nginx_config_valid(host):
    """Nginx config should pass syntax check."""
    cmd = host.run("nginx -t")
    assert cmd.rc == 0


def test_nginx_config_file(host):
    """Config file should exist with correct permissions."""
    config = host.file("/etc/nginx/nginx.conf")
    assert config.exists
    assert config.mode == 0o644
    assert config.user == "root"


def test_nginx_worker_processes(host, variables):
    """Worker processes should match configuration."""
    expected = str(variables.get("nginx_worker_processes", "auto"))
    config = host.file("/etc/nginx/nginx.conf").content_string
    assert f"worker_processes  {expected}" in config


@pytest.fixture
def variables(host):
    """Get Ansible variables for the host."""
    return host.ansible.get_variables()
```

```yaml
# molecule.yml — switch sang testinfra
verifier:
  name: testinfra
  options:
    v: true
    p: no:cacheprovider
```

---

## Chạy Molecule

```bash
# Full test cycle
molecule test

# Từng bước riêng lẻ
molecule create      # tạo container/VM
molecule prepare     # chạy prepare.yml
molecule converge    # apply role (converge.yml)
molecule verify      # chạy tests (verify.yml)
molecule destroy     # xoá container

# Idempotency check — chạy converge 2 lần, lần 2 không được có "changed"
molecule converge
molecule converge    # lần 2: tất cả tasks phải "ok", không "changed"
# Hoặc dùng: molecule test (tự include idempotency check)

# Debug — giữ container sau test
molecule converge --no-destroy
molecule login --host ubuntu-22.04   # SSH vào container
molecule destroy                      # cleanup thủ công

# Chỉ test 1 platform
molecule test --parallel   # chạy parallel các platforms

# Lint
molecule lint
```

---

## Multiple Scenarios

```
molecule/
├── default/       ← test basic functionality
│   ├── molecule.yml
│   ├── converge.yml
│   └── verify.yml
├── with-ssl/      ← test với SSL enabled
│   ├── molecule.yml  (nginx_ssl: true)
│   ├── converge.yml
│   └── verify.yml
└── upgrade/       ← test upgrade từ old version
    ├── molecule.yml
    ├── prepare.yml   (install old version)
    ├── converge.yml  (run role để upgrade)
    └── verify.yml
```

```bash
# Chạy scenario cụ thể
molecule test --scenario-name with-ssl
molecule test --scenario-name upgrade
```

---

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/molecule.yml
name: Molecule Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        scenario: [default, with-ssl]

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: |
          pip install molecule molecule-plugins[docker] pytest-testinfra

      - name: Run Molecule tests
        run: molecule test --scenario-name ${{ matrix.scenario }}
        working-directory: roles/nginx
        env:
          PY_COLORS: "1"
          ANSIBLE_FORCE_COLOR: "1"
```

---

## Gotchas

- **systemd trong Docker**: nhiều role cần systemd (service management). Docker container mặc định không chạy systemd. Dùng image của Jeff Geerling (`geerlingguy/docker-*-ansible`) — đã setup systemd sẵn.
- **`check_mode` để verify**: khi dùng Ansible tasks để verify, `check_mode: true` + `failed_when: task.changed` là pattern phổ biến. Nếu package chưa install → `check_mode` báo "would change" → `changed=true` → test fail.
- **Idempotency**: `molecule test` tự chạy converge 2 lần và fail nếu lần 2 có `changed`. Đây là test quan trọng nhất — đảm bảo role không làm gì thừa mỗi lần chạy.
- **Testinfra vs Ansible verify**: Testinfra linh hoạt hơn (Python, parameterize tests), nhưng cần thêm dependency. Ansible verify đơn giản hơn cho team đã biết Ansible.
- **Image pull**: lần đầu chạy pull Docker image → chậm. CI cache Docker layer để tăng tốc.
