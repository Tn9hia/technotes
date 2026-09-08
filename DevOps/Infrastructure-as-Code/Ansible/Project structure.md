```
project-root/
├── ansible.cfg          # File cấu hình Ansible (tùy chọn)
├── inventory/           # Thư mục chứa inventory (máy chủ đích)
│   ├── hosts            # File inventory mặc định
│   └── production/      # Inventory cho môi trường production
│   └── staging/         # Inventory cho môi trường staging
├── playbooks/           # Thư mục chứa playbooks
│   ├── site.yml         # Playbook chính (entry point)
│   ├── webservers.yml   # Playbook cho web servers
│   └── databases.yml    # Playbook cho databases
├── roles/               # Thư mục chứa các roles
│   └── common/          # Ví dụ một role
│       ├── tasks/       # Các task chính
│       │   └── main.yml
│       ├── handlers/    # Handlers (khởi động lại dịch vụ, v.v.)
│       │   └── main.yml
│       ├── templates/   # Jinja2 templates (file cấu hình động)
│       │   └── nginx.conf.j2
│       ├── files/       # File tĩnh (copy nguyên bản)
│       ├── vars/        # Biến dành riêng cho role
│       │   └── main.yml
│       ├── defaults/    # Biến mặc định của role (có thể ghi đè)
│       │   └── main.yml
│       └── meta/        # Metadata (phụ thuộc role, author, v.v.)
│           └── main.yml
├── group_vars/          # Biến áp dụng cho nhóm máy chủ
│   ├── all.yml          # Biến cho tất cả hosts
│   ├── webservers.yml   # Biến cho nhóm webservers
│   └── databases.yml
├── host_vars/           # Biến áp dụng cho từng host cụ thể
│   └── server1.yml
├── vault/               # Thư mục chứa biến mã hóa (Ansible Vault)
│   └── secrets.yml
└── requirements.yml     # File khai báo phụ thuộc (roles từ Galaxy)
```