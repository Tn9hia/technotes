---
title: Jinja2 Templating
tags:
  - ansible
  - jinja2
  - templating
  - deep-dive
date: 2026-04-26
---

# Jinja2 Templating

## Syntax cơ bản

```jinja2
{{ variable }}          ← output giá trị biến
{% statement %}         ← control flow (if, for, set)
{# comment #}           ← comment (không render ra output)
```

---

## Variable & Filters

### Cơ bản

```jinja2
{{ nginx_port }}
{{ database.host }}
{{ users[0].name }}
{{ ansible_default_ipv4.address }}
```

### Filters — Biến đổi giá trị

```jinja2
{# String filters #}
{{ app_name | upper }}                    → MYAPP
{{ app_name | lower }}                    → myapp
{{ app_name | capitalize }}               → Myapp
{{ app_name | replace('my', 'our') }}     → ourapp
{{ "  hello  " | trim }}                  → hello
{{ app_name | truncate(10) }}             → myapp...

{# Number filters #}
{{ disk_size | int }}                     → convert to int
{{ "3.14" | float }}                      → 3.14
{{ price | round(2) }}                    → 3.14

{# Default value #}
{{ undefined_var | default('fallback') }}
{{ port | default(80) }}
{{ config.timeout | default(30) | int }}

{# Type check #}
{{ var | type_debug }}                    → xem type của variable

{# List filters #}
{{ packages | join(', ') }}               → nginx, curl, git
{{ packages | length }}                   → 3
{{ packages | sort }}                     → sorted list
{{ packages | unique }}                   → deduplicate
{{ packages | reverse | list }}
{{ numbers | min }}
{{ numbers | max }}
{{ numbers | sum }}
{{ packages | first }}
{{ packages | last }}
{{ packages | random }}

{# Dict filters #}
{{ dict_var | dict2items }}               → [{key: k, value: v}, ...]
{{ list_var | items2dict }}               → {k: v, ...} (list of {key:, value:})
{{ dict_var | combine(other_dict) }}      → merge dicts
{{ dict_var | keys | list }}
{{ dict_var | values | list }}

{# Select / filter list #}
{{ users | selectattr('active', 'equalto', true) | list }}
{{ users | selectattr('role', 'in', ['admin', 'ops']) | list }}
{{ users | rejectattr('locked', 'equalto', true) | list }}
{{ packages | select('match', '^nginx') | list }}

{# Map — extract field từ list of dicts #}
{{ users | map(attribute='name') | list }}
{{ users | map(attribute='email') | join(', ') }}

{# Formatting #}
{{ size_bytes | filesizeformat }}         → 1.4 MB
{{ timestamp | strftime('%Y-%m-%d') }}
```

### Filters quan trọng cho config files

```jinja2
{# Boolean to string #}
{{ ssl_enabled | bool | lower }}          → "true" hoặc "false"
{{ ssl_enabled | ternary('on', 'off') }}  → "on" hoặc "off"

{# Path manipulation #}
{{ '/etc/nginx/nginx.conf' | dirname }}   → /etc/nginx
{{ '/etc/nginx/nginx.conf' | basename }}  → nginx.conf

{# Hash / checksum #}
{{ 'password' | password_hash('sha512') }}

{# Base64 #}
{{ 'hello' | b64encode }}
{{ encoded_string | b64decode }}

{# JSON #}
{{ config_dict | to_json }}
{{ config_dict | to_nice_json(indent=2) }}
{{ json_string | from_json }}

{# YAML #}
{{ config_dict | to_yaml }}
{{ config_dict | to_nice_yaml(indent=2) }}

{# Regex #}
{{ 'nginx-1.24.0' | regex_replace('(\d+\.\d+\.\d+)', 'VERSION') }}
{{ 'web01.internal.com' | regex_search('(\w+)\.', '\1') }}  → web01
```

---

## Conditionals trong template

```jinja2
{% if nginx_ssl_enabled %}
server {
    listen 443 ssl;
    ssl_certificate     {{ nginx_ssl_certificate }};
    ssl_certificate_key {{ nginx_ssl_certificate_key }};
}
{% endif %}

{# if/elif/else #}
{% if ansible_distribution == "Ubuntu" %}
include /etc/nginx/ubuntu-extras.conf;
{% elif ansible_distribution == "Debian" %}
include /etc/nginx/debian-extras.conf;
{% else %}
# Unknown distribution
{% endif %}

{# Inline ternary #}
worker_processes {{ nginx_worker_processes if nginx_worker_processes != 'auto' else ansible_processor_count }};
```

---

## Loops trong template

```jinja2
{# Basic loop #}
upstream backend {
{% for host in groups['webservers'] %}
    server {{ hostvars[host]['ansible_default_ipv4']['address'] }}:{{ app_port }};
{% endfor %}
}

{# Loop với index #}
{% for server in ntp_servers %}
server {{ server }}{% if loop.first %} prefer{% endif %} iburst
{% endfor %}

{# Loop variables #}
{# loop.index      → 1-based counter #}
{# loop.index0     → 0-based counter #}
{# loop.first      → true nếu lần đầu #}
{# loop.last       → true nếu lần cuối #}
{# loop.length     → tổng số items #}
{# loop.revindex   → reverse index #}

{# Loop qua dict #}
{% for key, value in sysctl_settings.items() %}
{{ key }} = {{ value }}
{% endfor %}

{# Conditional trong loop #}
{% for user in users if user.active %}
    {{ user.name }}
{% endfor %}
```

---

## Template thực tế

### nginx.conf.j2

```jinja2
# Managed by Ansible — DO NOT EDIT MANUALLY
# {{ ansible_managed }}

user  nginx;
worker_processes  {{ nginx_worker_processes }};
pid        /run/nginx.pid;

events {
    worker_connections  {{ nginx_worker_connections }};
}

http {
    sendfile        on;
    keepalive_timeout  {{ nginx_keepalive_timeout }};
    client_max_body_size {{ nginx_client_max_body_size }};

    access_log  {{ nginx_access_log }};
    error_log   {{ nginx_error_log }} {{ nginx_error_log_level | default('warn') }};

{% if nginx_gzip %}
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
{% endif %}

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### chrony.conf.j2

```jinja2
# Managed by Ansible

{% for server in ntp_servers %}
server {{ server }} iburst{% if loop.first %} prefer{% endif %}

{% endfor %}

makestep {{ chrony_makestep_threshold }} {{ chrony_makestep_limit }}
driftfile /var/lib/chrony/drift
rtcsync

{% if chrony_is_server %}
# Act as NTP server for internal clients
{% for subnet in ntp_allow_subnets %}
allow {{ subnet }}
{% endfor %}
local stratum {{ chrony_local_stratum }}
{% endif %}

logdir /var/log/chrony
log measurements statistics tracking
```

### hosts.j2 (custom /etc/hosts)

```jinja2
127.0.0.1   localhost
::1         localhost ip6-localhost

# Managed hosts
{% for host in groups['all'] %}
{% if hostvars[host]['ansible_default_ipv4'] is defined %}
{{ hostvars[host]['ansible_default_ipv4']['address'] }}  {{ hostvars[host]['ansible_hostname'] }} {{ hostvars[host]['inventory_hostname'] }}
{% endif %}
{% endfor %}
```

### systemd service.j2

```jinja2
[Unit]
Description={{ app_description | default(app_name) }}
After=network.target
{% if app_requires_database %}
After=postgresql.service
Requires=postgresql.service
{% endif %}

[Service]
Type={{ app_service_type | default('simple') }}
User={{ app_user }}
Group={{ app_group | default(app_user) }}
WorkingDirectory={{ app_home }}
ExecStart={{ app_executable }} {{ app_args | default('') }}
ExecReload=/bin/kill -HUP $MAINPID
Restart={{ app_restart_policy | default('on-failure') }}
RestartSec=5
StandardOutput=journal
StandardError=journal
SyslogIdentifier={{ app_name }}

{% if app_environment is defined %}
[Service]
{% for key, value in app_environment.items() %}
Environment="{{ key }}={{ value }}"
{% endfor %}
{% endif %}

[Install]
WantedBy=multi-user.target
```

---

## `ansible_managed` — Best practice

```jinja2
{# Luôn thêm vào đầu file được generate #}
# {{ ansible_managed }}
# Generates: Ansible managed: /path/to/template.j2, modified YYYY-MM-DD by user on control_host
```

Config trong `ansible.cfg`:
```ini
ansible_managed = Ansible managed: {file} modified on %Y-%m-%d by {uid} on {host}
```

---

## Whitespace control

```jinja2
{# Mặc định: newline sau block tag được giữ lại #}
{% for item in list %}
{{ item }}
{% endfor %}

{# Trim whitespace trước block tag #}
{%- for item in list %}

{# Trim whitespace sau block tag #}
{% for item in list -%}

{# Trim cả hai #}
{%- for item in list -%}
```

---

## Test template locally

```bash
# Render template với variables để xem output trước khi deploy
ansible web01 -m template \
  -a "src=templates/nginx.conf.j2 dest=/tmp/nginx_test.conf" \
  --check --diff

# Validate rendered config
ansible web01 -m command \
  -a "nginx -t -c /tmp/nginx_test.conf"
```

---

## Gotchas

- **`{{ }}` trong task vs template**: trong task YAML, `{{ var }}` cần được quote nếu bắt đầu expression: `dest: "{{ path }}/file"` — bắt buộc có quotes. Trong `.j2` file thì không cần.
- **Whitespace và YAML**: thêm/bỏ newline trong template ảnh hưởng đến format file config. `{%- -%}` để control whitespace chính xác.
- **`ansible_managed` thay đổi mỗi lần chạy**: timestamp trong `ansible_managed` thay đổi → task luôn report `changed`. Nếu không muốn: dùng custom string không có timestamp: `ansible_managed = "Ansible managed — do not edit"`.
- **Undefined variable trong template**: Jinja2 raise error nếu variable không tồn tại. Luôn có `| default()` cho optional variables: `{{ nginx_extra_config | default('') }}`.
- **Loop và trailing newline**: loop thường tạo trailing newline hay extra blank lines. Dùng `{%- endfor %}` hoặc `loop.last` để kiểm soát.
