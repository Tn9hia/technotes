---
title: Vault Secret Engines
tags:
  - vault
  - secrets
  - pki
  - database
  - transit
date: 2026-04-27
---

# Vault Secret Engines

Secret Engines là plugins xử lý và lưu trữ secrets. Mỗi engine được mount tại một path.

```bash
# Enable secret engine
vault secrets enable -path=secret kv-v2       # KV v2 tại path "secret/"
vault secrets enable database                 # Database engine tại "database/"
vault secrets enable pki                      # PKI tại "pki/"
vault secrets enable transit                  # Transit tại "transit/"

# List mounted engines
vault secrets list
```

---

## KV v2 — Key-Value Store

KV v2 thêm versioning và soft delete so với KV v1.

```bash
# Enable
vault secrets enable -path=secret kv-v2

# Write secret
vault kv put secret/production/myapp \
  db_password="supersecret" \
  api_key="abc123" \
  endpoint="https://api.internal"

# Read
vault kv get secret/production/myapp
vault kv get -field=db_password secret/production/myapp

# Versioning
vault kv put secret/production/myapp db_password="newpass"   # creates version 2
vault kv get secret/production/myapp                         # đọc latest
vault kv get -version=1 secret/production/myapp             # đọc version 1

# List keys
vault kv list secret/production/

# Delete (soft — versions still accessible)
vault kv delete secret/production/myapp
vault kv undelete -versions=2 secret/production/myapp        # restore

# Destroy (hard — permanent)
vault kv destroy -versions=1,2 secret/production/myapp

# Metadata
vault kv metadata get secret/production/myapp
vault kv metadata put -max-versions=10 \
  -delete-version-after=720h \    # auto-delete versions sau 30 ngày
  secret/production/myapp
```

```bash
# Patch (update một field, giữ nguyên các field khác)
vault kv patch secret/production/myapp api_key="newkey"
# → creates new version, db_password vẫn giữ nguyên
```

### KV v2 via API (cho automation)

```bash
# API endpoint: /v1/<mount>/data/<path>
curl -H "X-Vault-Token: $VAULT_TOKEN" \
     -X POST \
     -d '{"data": {"password": "secret"}}' \
     https://vault.internal:8200/v1/secret/data/production/myapp

# Read
curl -H "X-Vault-Token: $VAULT_TOKEN" \
     https://vault.internal:8200/v1/secret/data/production/myapp

# With specific version
curl -H "X-Vault-Token: $VAULT_TOKEN" \
     "https://vault.internal:8200/v1/secret/data/production/myapp?version=2"
```

---

## Database — Dynamic Credentials

Database engine tạo temporary credentials với TTL — không cần hardcode passwords.

```
Flow:
  App → Vault (authenticate) → request DB credentials
  Vault → CREATE USER 'v-token-abc' WITH PASSWORD '...' VALID UNTIL 'TTL'
  App uses credentials → expires after TTL
  Vault → auto-revoke: DROP ROLE 'v-token-abc'
```

### PostgreSQL

```bash
# Enable và configure
vault secrets enable database

vault write database/config/myapp-postgres \
  plugin_name=postgresql-database-plugin \
  allowed_roles="myapp-readonly,myapp-readwrite" \
  connection_url="postgresql://{{username}}:{{password}}@postgres-primary:5432/myapp?sslmode=require" \
  username="vault-root" \
  password="vaultpass"

# Rotate root credentials ngay (không để biết root password)
vault write -force database/rotate-root/myapp-postgres

# Tạo roles — SQL để CREATE/REVOKE users
vault write database/roles/myapp-readonly \
  db_name=myapp-postgres \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}' IN ROLE readonly_role;" \
  revocation_statements="DROP ROLE IF EXISTS \"{{name}}\";" \
  default_ttl=1h \
  max_ttl=24h

vault write database/roles/myapp-readwrite \
  db_name=myapp-postgres \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  revocation_statements="DROP ROLE IF EXISTS \"{{name}}\";" \
  default_ttl=1h \
  max_ttl=4h

# Request credentials
vault read database/creds/myapp-readonly
# Key        Value
# lease_id   database/creds/myapp-readonly/AbCdEfGh...
# username   v-token-myapp-AbCdEf
# password   A1B2C3D4-E5F6-...

# Renew lease
vault lease renew database/creds/myapp-readonly/AbCdEfGh...

# Revoke immediately
vault lease revoke database/creds/myapp-readonly/AbCdEfGh...

# Revoke all leases cho role này
vault lease revoke -prefix database/creds/myapp-readonly/
```

### MySQL / MariaDB

```bash
vault write database/config/myapp-mysql \
  plugin_name=mysql-database-plugin \
  allowed_roles="myapp-mysql-role" \
  connection_url="{{username}}:{{password}}@tcp(mysql:3306)/myapp" \
  username="vault_root" \
  password="vaultpass"

vault write database/roles/myapp-mysql-role \
  db_name=myapp-mysql \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}'; GRANT SELECT ON myapp.* TO '{{name}}'@'%';" \
  revocation_statements="DROP USER IF EXISTS '{{name}}'@'%';" \
  default_ttl=1h \
  max_ttl=24h
```

### Static Roles (credential rotation không dynamic)

```bash
# Static role: Vault rotate password định kỳ (không tạo user mới)
# Dùng cho legacy apps không support dynamic creds
vault write database/static-roles/myapp-static \
  db_name=myapp-postgres \
  username="app_static_user" \
  rotation_period=24h       # rotate password mỗi 24h
  rotation_statements="ALTER USER \"{{name}}\" WITH PASSWORD '{{password}}';"

# Read current password
vault read database/static-creds/myapp-static
```

---

## PKI — Certificate Authority

PKI engine biến Vault thành Certificate Authority — issue TLS certs on demand.

### Root CA + Intermediate CA (Best Practice)

```
Root CA (offline / Vault) → sign → Intermediate CA → sign → Leaf certs
```

```bash
# === Root CA ===
vault secrets enable -path=pki pki
vault secrets tune -max-lease-ttl=87600h pki   # 10 years

# Generate root CA cert (self-signed, hoặc import external root)
vault write -field=certificate pki/root/generate/internal \
  common_name="GlobalTech Internal Root CA" \
  organization="GlobalTechJSC" \
  country="VN" \
  ttl=87600h > /tmp/root_ca.crt

# Configure URLs
vault write pki/config/urls \
  issuing_certificates="https://vault.internal:8200/v1/pki/ca" \
  crl_distribution_points="https://vault.internal:8200/v1/pki/crl"

# === Intermediate CA ===
vault secrets enable -path=pki_int pki
vault secrets tune -max-lease-ttl=43800h pki_int   # 5 years

# Generate CSR
vault write -format=json pki_int/intermediate/generate/internal \
  common_name="GlobalTech Internal Intermediate CA" \
  | jq -r '.data.csr' > /tmp/intermediate.csr

# Sign CSR với Root CA
vault write -format=json pki/root/sign-intermediate \
  csr=@/tmp/intermediate.csr \
  format=pem_bundle \
  ttl=43800h \
  | jq -r '.data.certificate' > /tmp/intermediate.cert.pem

# Import signed cert back
vault write pki_int/intermediate/set-signed \
  certificate=@/tmp/intermediate.cert.pem

# Configure Intermediate CA URLs
vault write pki_int/config/urls \
  issuing_certificates="https://vault.internal:8200/v1/pki_int/ca" \
  crl_distribution_points="https://vault.internal:8200/v1/pki_int/crl"
```

### Issue Certificates

```bash
# Tạo role cho loại cert
vault write pki_int/roles/internal-services \
  allowed_domains="internal,svc.cluster.local" \
  allow_subdomains=true \
  allow_glob_domains=false \
  max_ttl=720h \     # max 30 days
  key_type=ec \      # ECDSA (smaller, faster than RSA)
  key_bits=256 \
  require_cn=true \
  enforce_hostnames=true

# Issue certificate
vault write -format=json pki_int/issue/internal-services \
  common_name="myapp.production.svc.cluster.local" \
  alt_names="myapp.internal,myapp-service" \
  ttl=720h \
  | tee /tmp/cert.json

# Extract components
cat /tmp/cert.json | jq -r '.data.certificate' > myapp.crt
cat /tmp/cert.json | jq -r '.data.private_key' > myapp.key
cat /tmp/cert.json | jq -r '.data.issuing_ca' > ca.crt

# Revoke certificate
vault write pki_int/revoke serial_number=<serial>

# Tidy (cleanup expired certs và CRL)
vault write pki_int/tidy \
  tidy_cert_store=true \
  tidy_revocation_list=true \
  safety_buffer=72h
```

### cert-manager + Vault Issuer

cert-manager có thể request certs từ Vault PKI tự động:

```yaml
# ClusterIssuer với Vault PKI
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: vault-issuer
spec:
  vault:
    path: pki_int/sign/internal-services
    server: https://vault.internal:8200
    caBundle: <base64-encoded-vault-ca>
    auth:
      kubernetes:
        role: cert-manager
        mountPath: /v1/auth/kubernetes
        secretRef:
          name: cert-manager-vault-token
          key: token

---
# Certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-cert
  namespace: production
spec:
  secretName: myapp-tls
  issuerRef:
    name: vault-issuer
    kind: ClusterIssuer
  commonName: myapp.production.svc.cluster.local
  dnsNames:
    - myapp.internal
  duration: 720h
  renewBefore: 168h    # renew 7 days before expiry
```

---

## Transit — Encryption as a Service

Transit engine cung cấp cryptographic operations mà không expose keys. Application encrypt/decrypt data, nhưng key luôn ở trong Vault.

```bash
vault secrets enable transit

# Tạo encryption key
vault write -f transit/keys/myapp-data \
  type=aes256-gcm96     # symmetric encryption
  # type=rsa-2048       # asymmetric
  # type=ecdsa-p256     # ECDSA signing

# Key có thể rotate:
vault write -f transit/keys/myapp-data/rotate

# Encrypt
vault write transit/encrypt/myapp-data \
  plaintext=$(echo -n "sensitive data" | base64)
# → ciphertext: vault:v1:AbCdEfGhIjKlMnOpQrStUvWxYzAb...

# Decrypt
vault write transit/decrypt/myapp-data \
  ciphertext="vault:v1:AbCdEfGhIjKlMnOpQrStUvWxYzAb..."
# → plaintext: c2Vuc2l0aXZlIGRhdGE= (base64 của "sensitive data")

# Rewrap (re-encrypt ciphertext với key version mới sau rotation)
vault write transit/rewrap/myapp-data \
  ciphertext="vault:v1:old-ciphertext..."
# → vault:v2:new-ciphertext...

# Sign / Verify (dùng asymmetric key)
vault write transit/sign/myapp-signing-key \
  input=$(echo -n "document content" | base64)
vault write transit/verify/myapp-signing-key \
  input=$(echo -n "document content" | base64) \
  signature="vault:v1:..."
```

### Use Case: Database Column Encryption

```python
# Application encrypt PII trước khi lưu DB
import hvac
import base64

client = hvac.Client(url='https://vault.internal:8200', token=vault_token)

def encrypt_pii(plaintext: str) -> str:
    encoded = base64.b64encode(plaintext.encode()).decode()
    result = client.secrets.transit.encrypt_data(
        name='pii-encryption',
        plaintext=encoded
    )
    return result['data']['ciphertext']

def decrypt_pii(ciphertext: str) -> str:
    result = client.secrets.transit.decrypt_data(
        name='pii-encryption',
        ciphertext=ciphertext
    )
    return base64.b64decode(result['data']['plaintext']).decode()

# Lưu DB
user.ssn = encrypt_pii("123456789")    # lưu "vault:v1:AbCd..."
# Đọc DB
real_ssn = decrypt_pii(user.ssn)       # "123456789"
```

---

## SSH Secrets Engine

```bash
vault secrets enable ssh

# OTP mode: Vault generate one-time password cho SSH
vault write ssh/roles/otp-role \
  key_type=otp \
  default_user=ubuntu \
  cidr_list="10.0.0.0/8"

# Request OTP
vault write ssh/creds/otp-role ip=10.0.0.5
# → key: AbCdEfGhIjKl...  (one-time password, expires soon)

# SSH với OTP
ssh ubuntu@10.0.0.5
# Password: AbCdEfGhIjKl...
```

---

## Gotchas

- **KV v2 path prefix**: KV v2 API path là `/v1/<mount>/data/<path>` nhưng CLI dùng `<mount>/<path>`. Khi viết API calls, nhớ thêm `data/` prefix. Policy path cũng cần phân biệt `secret/data/*` vs `secret/metadata/*`.
- **Database root credential rotation**: Sau `rotate-root`, Vault biết password mới nhưng bạn thì không. Nếu cần debug trực tiếp DB → dùng static role hoặc tạo separate admin account ngoài Vault. Không thể recover root password sau rotate.
- **PKI CRL và OCSP**: Certs đã issue sẽ expire tự nhiên. Revoke cần CRL (Certificate Revocation List). Clients phải check CRL — nếu Vault unreachable, CRL không refresh được → clients có thể accept revoked certs (nếu CRL grace period còn). Configure short CRL TTL và ensure Vault HA.
- **Transit key rotation và rewrap**: Rotate key tạo version mới. Old ciphertexts vẫn decrypt được (Vault giữ all versions). Cần rewrap periodically để migrate ciphertexts sang key version mới. Min decrypt version có thể set để prevent dùng old keys.
- **Lease TTL và app restarts**: Dynamic DB creds có TTL. Nếu app restart và reuse creds từ config file sau TTL expire → auth fail. Luôn dùng Vault Agent hoặc request new creds on startup.
