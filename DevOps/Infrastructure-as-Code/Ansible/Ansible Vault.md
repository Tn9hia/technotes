---
title: Ansible Vault
tags:
  - ansible
  - vault
  - security
  - deep-dive
date: 2026-04-26
---

# Ansible Vault — Secret Management

## Ansible Vault là gì?

Ansible Vault encrypt file hoặc variables để **an toàn commit lên Git** mà không lộ secret. Sử dụng AES-256-CBC encryption.

---

## Encrypt / Decrypt File

```bash
# Encrypt file mới hoặc file có sẵn
ansible-vault encrypt group_vars/all/vault.yaml
# → hỏi password và confirm

# Decrypt (override file gốc)
ansible-vault decrypt group_vars/all/vault.yaml

# Xem nội dung file đã encrypt (không decrypt file)
ansible-vault view group_vars/all/vault.yaml

# Edit file đang encrypt (decrypt temp, mở editor, re-encrypt)
ansible-vault edit group_vars/all/vault.yaml

# Tạo file mới và encrypt ngay
ansible-vault create group_vars/production/vault.yaml

# Re-key — đổi password
ansible-vault rekey group_vars/all/vault.yaml
```

### Encrypt string (inline variable)

```bash
# Encrypt single string value
ansible-vault encrypt_string 'supersecret' --name 'db_password'
# Output:
db_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          3736303136346131383263386562303839356336343736...

# Paste vào file yaml thông thường
# group_vars/all/main.yaml:
db_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          3736303136346131383263386562303839356336343736...
```

---

## Vault ID — Multiple passwords

Dùng khi cần nhiều password khác nhau (dev vault vs prod vault):

```bash
# Encrypt với vault ID label
ansible-vault encrypt --vault-id dev@prompt group_vars/dev/vault.yaml
ansible-vault encrypt --vault-id prod@prompt group_vars/prod/vault.yaml

# Chạy playbook với nhiều vault password
ansible-playbook site.yml \
  --vault-id dev@prompt \
  --vault-id prod@prompt

# Hoặc dùng password file
ansible-playbook site.yml \
  --vault-id dev@.vault_pass_dev \
  --vault-id prod@.vault_pass_prod
```

---

## Password File — Không nhập password mỗi lần

```bash
# Tạo password file
echo "my_vault_password" > .vault_pass
chmod 600 .vault_pass
echo ".vault_pass" >> .gitignore   # QUAN TRỌNG: không commit password file

# Dùng trong command
ansible-playbook site.yml --vault-password-file .vault_pass

# Hoặc config trong ansible.cfg
```

```ini
# ansible.cfg
[defaults]
vault_password_file = .vault_pass
# Hoặc: vault_password_file = ~/.ansible_vault_pass  (ngoài project dir, an toàn hơn)
```

### Password script (dynamic password)

```python
#!/usr/bin/env python3
# .vault_pass.py — lấy password từ external source
import subprocess
result = subprocess.run(['aws', 'secretsmanager', 'get-secret-value',
                        '--secret-id', 'ansible-vault-password',
                        '--query', 'SecretString',
                        '--output', 'text'],
                       capture_output=True, text=True)
print(result.stdout.strip())
```

```bash
chmod +x .vault_pass.py
ansible-playbook site.yml --vault-password-file .vault_pass.py
```

---

## Pattern chuẩn — Tách vault vars và plain vars

```
group_vars/
├── all/
│   ├── main.yaml       ← plain variables, reference vault vars
│   └── vault.yaml      ← ENCRYPTED: actual secret values
├── webservers/
│   ├── main.yaml
│   └── vault.yaml
└── production/
    ├── main.yaml
    └── vault.yaml      ← production secrets (khác password với dev)
```

```yaml
# group_vars/all/vault.yaml (ENCRYPTED)
vault_db_password: "prod_supersecret_123"
vault_api_key: "sk-prod-abc123def456"
vault_ssl_private_key: |
  -----BEGIN PRIVATE KEY-----
  MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQC...
  -----END PRIVATE KEY-----

# group_vars/all/main.yaml (PLAIN — commit bình thường)
db_password: "{{ vault_db_password }}"
api_key: "{{ vault_api_key }}"
ssl_private_key: "{{ vault_ssl_private_key }}"
```

**Tại sao tách 2 file?**
- `main.yaml` thấy tên variable → dễ grep, dễ hiểu structure
- `vault.yaml` chứa giá trị thực → encrypt
- Không lộ secret nhưng vẫn track được "có những secrets nào"

---

## Integrate với CI/CD

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
- name: Create vault password file
  run: echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > .vault_pass
  
- name: Run playbook
  run: |
    ansible-playbook site.yml \
      --vault-password-file .vault_pass \
      -i inventory/production/
  
- name: Cleanup
  if: always()
  run: rm -f .vault_pass
```

### GitLab CI

```yaml
# .gitlab-ci.yml
deploy:
  script:
    - echo "$ANSIBLE_VAULT_PASSWORD" > .vault_pass
    - ansible-playbook site.yml --vault-password-file .vault_pass
    - rm -f .vault_pass
  after_script:
    - rm -f .vault_pass   # cleanup kể cả khi fail
```

### Jenkins

```groovy
pipeline {
    stages {
        stage('Deploy') {
            steps {
                withCredentials([string(credentialsId: 'ansible-vault-password', variable: 'VAULT_PASS')]) {
                    sh 'echo $VAULT_PASS > .vault_pass'
                    sh 'ansible-playbook site.yml --vault-password-file .vault_pass'
                }
            }
            post {
                always { sh 'rm -f .vault_pass' }
            }
        }
    }
}
```

---

## `no_log` — Ẩn output sensitive

```yaml
- name: Create database user
  mysql_user:
    name: appuser
    password: "{{ db_password }}"
    priv: "mydb.*:ALL"
  no_log: true          # không print task vars ra stdout
                        # vẫn log "TASK [Create database user]" nhưng ẩn params
```

---

## Security Best Practices

- [ ] **Không commit password file** — `.vault_pass` luôn trong `.gitignore`
- [ ] **Tách vault theo environment**: dev vault và prod vault dùng password khác nhau
- [ ] **Encrypt toàn bộ `vault.yaml`**, không encrypt `main.yaml` — giữ structure visible
- [ ] **Rotate vault password** định kỳ (`rekey`) — đặc biệt khi nhân viên nghỉ việc
- [ ] **Không dùng `ansible-vault encrypt_string` inline quá nhiều** — khó review diff
- [ ] **`no_log: true`** cho task xử lý password/key
- [ ] **Password file ngoài project directory**: `~/.ansible_vault_pass` thay vì `.vault_pass` trong repo
- [ ] **Xét dùng ESO thay Vault** nếu đã có K8s và Vault/AWS SM — centralized, rotatable

---

## Gotchas

- **`ansible-vault edit` cần EDITOR**: mặc định mở `vi`. Set `export EDITOR=nano` nếu muốn.
- **Re-encrypt sau `rekey`**: `ansible-vault rekey` chỉ đổi password của file đó, không ảnh hưởng file khác. Phải `rekey` từng file riêng hoặc script loop.
- **Vault và diff**: khi file vault thay đổi, `git diff` chỉ thấy encrypted blob → không review được. Giải pháp: dùng `ansible-vault diff` hoặc `git diff` với custom filter.
- **`vault_id` phải match**: nếu encrypt với `--vault-id prod@...` nhưng chạy không có `--vault-id prod@...` → decrypt fail với lỗi confusing.
- **Binary files và vault**: `ansible-vault encrypt` hoạt động với binary file nhưng khi decrypt in ra terminal có thể bị garbled. Dùng `--output` flag.
