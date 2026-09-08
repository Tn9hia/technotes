---
title: Cluster Setup and Hardening
tags:
  - kubernetes
  - security
  - cks
  - cluster-hardening
date: 2026-08-18
---

# Cluster Setup and Hardening

Đây là domain lớn nhất trong CKS (Cluster Setup 10% + Cluster Hardening 15% = 25% tổng điểm thi). Nội dung bao trùm: bảo mật hạ tầng cluster, TLS/PKI, authentication/authorization, RBAC, network policy, kubelet, node metadata, CIS benchmark, nâng cấp cluster, Docker daemon, Ingress, auditing.

## Tổng quan: Kubernetes Security Primitives

Trước khi đi vào chi tiết, cần nắm 4 trụ cột bảo mật của một cluster:

1. **Securing cluster hosts** — vô hiệu hóa root login, dùng SSH key-based auth thay vì password, hardening OS bên dưới. Nếu hạ tầng bị compromise thì toàn bộ cluster mất an toàn, bất kể Kubernetes có cấu hình đúng đến đâu.
2. **Kube API Server — cửa ngõ duy nhất**: mọi request (qua `kubectl` hay REST API trực tiếp) đều đi qua kube-apiserver. Hai câu hỏi cốt lõi: *Ai có thể truy cập?* (Authentication) và *Họ được làm gì?* (Authorization).
3. **Securing intra-cluster communication**: mọi giao tiếp giữa các component (etcd, controller-manager, scheduler, apiserver, kubelet, kube-proxy) đều phải được bảo vệ bằng TLS.
4. **Network Policies**: mặc định mọi pod có thể giao tiếp tự do với nhau ("allow all"). Network Policy dùng để giới hạn traffic pod-to-pod.

Kubernetes hỗ trợ các cơ chế authentication: static password file, static token file, certificates, tích hợp bên thứ ba (LDAP, Kerberos), và service account cho machine-to-machine. Kubernetes **không tự quản lý user account** như một hệ thống người dùng nội bộ — nó luôn dựa vào nguồn bên ngoài (file, certificate, identity provider). Ngược lại, **service account thì được quản lý native qua API**.

Về authorization, Kubernetes hỗ trợ: Node Authorization, ABAC (Attribute-Based), RBAC (Role-Based — phổ biến nhất), và Webhook-based authorization (ví dụ Open Policy Agent).

## Verify Platform Binaries Before Deploying

Trước khi triển khai bất kỳ binary Kubernetes nào tải từ Internet (kubelet, kubeadm, kubectl...), phải xác minh tính toàn vẹn bằng checksum để tránh bị tấn công man-in-the-middle (kẻ tấn công thay thế file gốc bằng file độc hại). Binary tải từ [Kubernetes GitHub release page](https://github.com/kubernetes/kubernetes/releases) luôn kèm SHA512 checksum công bố sẵn.

```bash
# Tải binary
curl https://dl.k8s.io/v1.20.0/kubernetes.tar.gz -L -o kubernetes.tar.gz

# Tạo checksum để so sánh với giá trị công bố trên trang release
shasum -a 512 kubernetes.tar.gz      # macOS / Linux
sha512sum kubernetes.tar.gz          # Linux
```

Chỉ cần khác 1 bit trong file, hash SHA-512 sẽ hoàn toàn khác — nếu hash không khớp, file đã bị chỉnh sửa và không nên triển khai.

## TLS và PKI trong Kubernetes

### Kiến thức nền TLS/SSL

TLS certificate mã hóa communication và xác thực danh tính server. Có hai loại mã hóa:

- **Symmetric encryption**: cùng một key dùng để mã hóa và giải mã — rủi ro nếu key bị chặn khi truyền qua mạng.
- **Asymmetric encryption**: dùng cặp key (public/private). Dữ liệu mã hóa bằng public key chỉ giải mã được bằng private key tương ứng. Đây là nền tảng của TLS.

Quy trình TLS đầy đủ khi client kết nối HTTPS tới server:

| Bước | Mô tả |
|---|---|
| 1. Key Pair Generation | Server tạo cặp key cho HTTPS (admin tạo cặp key cho SSH tương tự) |
| 2. CSR Creation | Server tạo Certificate Signing Request (CSR) và gửi cho CA |
| 3. Certificate Signing | CA ký certificate bằng private key của CA, trả lại certificate đã ký |
| 4. Certificate Distribution | Khi user truy cập website, server gửi certificate đã ký (chứa public key) |
| 5. Certificate Validation | Browser xác thực certificate bằng public key của CA |
| 6. Symmetric Key Exchange | Browser tạo symmetric key, mã hóa bằng public key của server, gửi đi |
| 7. Secure Communication | Server giải mã symmetric key bằng private key riêng; từ đó giao tiếp dùng symmetric key |

Toàn bộ framework này (CA, key pair, certificate, quản lý key) gọi là **Public Key Infrastructure (PKI)**.

**Quy ước đặt tên file:**
- File chứa **public key/certificate**: đuôi `.crt` hoặc `.pem` (vd `server.crt`, `client.pem`).
- File chứa **private key**: đuôi `.key`, hoặc có chữ "key" trong tên file (vd `server-key.pem`).

Ví dụ tạo SSH key pair và cấu hình web server TLS bằng OpenSSL:

```bash
# SSH key pair
ssh-keygen
# tạo ra id_rsa (private) và id_rsa.pub (public)
ssh -i id_rsa user1@server1

# Key pair cho web server
openssl genrsa -out my-bank.key 1024
openssl rsa -in my-bank.key -pubout > mybank.pem

# Tạo CSR để gửi cho CA ký
openssl req -new -key my-bank.key -out my-bank.csr \
  -subj "/C=US/ST=CA/O=MyOrg, Inc./CN=my-bank.com"
```

Certificate chứa các thông tin quan trọng: **Subject (CN)** — danh tính chủ sở hữu, **Issuer** — CA đã ký, **Validity** — thời hạn hiệu lực, **Subject Alternative Name (SAN)** — các DNS/IP thay thế được certificate đó bao phủ (rất quan trọng khi 1 certificate cần phục vụ nhiều domain/IP).

### Các certificate trong một cluster Kubernetes

Một cluster cần tối thiểu 1 CA (`ca.crt` / `ca.key`) để ký toàn bộ certificate client và server. Certificate được chia làm 2 nhóm:

| Nhóm | Thành phần | Vai trò |
|---|---|---|
| **Server Certificates** | kube-apiserver (`apiserver.crt`/`apiserver.key`), etcd server (`etcd-server.crt`/`etcd-server.key`), kubelet (`kubelet.crt`/`kubelet.key`) | Phục vụ HTTPS, xác thực server với client kết nối tới |
| **Client Certificates** | admin user (`admin.crt`/`admin.key`), scheduler, controller-manager, kube-proxy | Xác thực client khi gọi tới kube-apiserver |

Lưu ý: kube-apiserver vừa là server (nhận request từ kubectl/kubelet) vừa đóng vai trò client khi gọi tới etcd — nó có thể dùng lại certificate của chính nó hoặc một cặp certificate riêng cho etcd.

### Tạo certificate bằng OpenSSL

```bash
# 1. Tạo CA — tự ký (self-signed)
openssl genrsa -out ca.key 2048
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt

# 2. Admin user certificate
openssl genrsa -out admin.key 2048
openssl req -new -key admin.key -subj "/CN=kube-admin" -out admin.csr
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt
```

**Quan trọng cho RBAC**: thêm Organization (`O=`) vào subject của CSR để gán user vào group. Ví dụ `O=system:masters` cấp quyền admin toàn cluster:

```bash
openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr
```

`CN` (Common Name) = **username** trong Kubernetes; `O` (Organization) = **group** mà user thuộc về. Đây là cách RBAC nhận diện user/group từ certificate — không có field riêng nào khác.

Test gọi API trực tiếp bằng certificate vừa tạo:

```bash
curl https://kube-apiserver:6443/api/v1/pods \
  --key admin.key --cert admin.crt --cacert ca.crt
```

**Certificate cho kube-apiserver** cần nhiều SAN (Subject Alternative Names) vì nó được truy cập qua nhiều tên/IP khác nhau (service name nội bộ, IP cluster, hostname...):

```ini
# openssl.cnf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation
subjectAltName = @alt_names

[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 172.17.0.87
```

```bash
openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr
```

**Certificate cho etcd** cần thêm peer certificate để giao tiếp giữa các node etcd trong cụm HA:

```yaml
etcd:
  --advertise-client-urls=https://127.0.0.1:2379
  --key-file=/path-to-certs/etcdserver.key
  --cert-file=/path-to-certs/etcdserver.crt
  --client-cert-auth=true
  --initial-advertise-peer-urls=https://127.0.0.1:2380
  --listen-client-urls=https://127.0.0.1:2379
  --listen-peer-urls=https://127.0.0.1:2380
  --peer-cert-file=/path-to-certs/etcdpeer1.crt
  --peer-client-cert-auth=true
  --peer-key-file=/path/to/etcd/peer.key
  --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
  --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

**Certificate cho kubelet** trên mỗi node được đặt tên theo node (node-01, node-02...). Client certificate mà kubelet dùng để xác thực với kube-apiserver tuân theo quy ước tên `system:node:<node-name>` — chính là cơ chế mà **Node Authorizer** dựa vào để cấp quyền.

Cấu hình kube-apiserver tham chiếu đầy đủ các certificate liên quan:

```bash
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=${INTERNAL_IP} \
  --allow-privileged=true \
  --authorization-mode=Node,RBAC \
  --etcd-cafile=/var/lib/kubernetes/ca.pem \
  --etcd-certfile=/var/lib/kubernetes/apiserver-etcd-client.crt \
  --etcd-keyfile=/var/lib/kubernetes/apiserver-etcd-client.key \
  --etcd-servers=https://127.0.0.1:2379 \
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \
  --kubelet-client-certificate=/var/lib/kubernetes/apiserver-kubelet-client.crt \
  --kubelet-client-key=/var/lib/kubernetes/apiserver-kubelet-client.key \
  --kubelet-https=true \
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \
  --client-ca-file=/var/lib/kubernetes/ca.pem \
  --tls-cert-file=/var/lib/kubernetes/apiserver.crt \
  --tls-private-key-file=/var/lib/kubernetes/apiserver.key
```

### Xem và kiểm tra chi tiết certificate (View Certificate Details)

Khi audit certificate trong cluster có sẵn, bước đầu tiên là xác định cluster được provision bằng cách nào:

- **systemd-managed** (tự cài từ đầu): xem unit file (`/etc/systemd/system/*.service` hoặc `/lib/systemd/system/`) để tìm flag `--tls-cert-file`, `--client-ca-file`...
- **kubeadm**: control plane chạy dưới dạng **static pod**, manifest nằm ở `/etc/kubernetes/manifests/`, certificate mặc định ở `/etc/kubernetes/pki/`.

```bash
# Xem manifest static pod của apiserver (kubeadm)
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Đọc chi tiết một certificate
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

Output gồm `Issuer` (CA đã ký), `Validity` (Not Before / Not After — hạn dùng), `Subject: CN=...`, và `X509v3 Subject Alternative Name` (DNS/IP được cấp).

Checklist khi audit certificate:

| Cần kiểm tra | Cách kiểm tra | Kỳ vọng |
|---|---|---|
| Vị trí cert/key của apiserver | Xem static manifest hoặc systemd unit | Trỏ đúng tới `/etc/kubernetes/pki` hoặc `/var/lib/kubernetes` |
| Metadata certificate | `openssl x509 -in <cert> -text -noout` | CN, SAN, issuer, Not After đầy đủ và chính xác |
| Hạn certificate | `openssl x509 -in <cert> -noout -dates` | `notAfter` còn hiệu lực |
| Lỗi TLS handshake | `journalctl -u <service> -l` hoặc `kubectl logs <pod> -n kube-system` | Không có lỗi "bad certificate" |
| Log container khi control plane down | `crictl logs <id>` hoặc `docker logs <id>` | Xác định lý do apiserver không start |

Khi control plane down và `kubectl` không kết nối được, phải dùng trực tiếp container runtime để xem log:

```bash
crictl ps -a
crictl logs <container-id>
# hoặc với Docker runtime
docker ps -a
docker logs <container-id>
```

### Certificates API — tự động hóa cấp phát certificate

Thay vì admin phải tự tay ký CSR bằng CA trên master node, Kubernetes cung cấp object `CertificateSigningRequest` để tự động hóa quy trình qua API.

Quy trình:

```bash
# 1. User (vd Jane) tự tạo private key + CSR
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -subj "/CN=jane" -out jane.csr
```

```yaml
# 2. Admin tạo CSR object, request là CSR đã base64-encode
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  groups:
  - system:authenticated
  usages:
  - digital signature
  - key encipherment
  - server auth
  request: <base64-encoded-CSR>
```

```bash
# 3. Xem các CSR đang chờ duyệt
kubectl get csr

# 4. Duyệt CSR
kubectl certificate approve jane

# 5. Lấy certificate đã ký (base64, cần decode)
kubectl get csr jane -o yaml
```

CSR được duyệt/ký bởi **kube-controller-manager** (thông qua CSR-APPROVING và CSR-SIGNING controller), controller-manager cần quyền truy cập CA root cert/key:

```yaml
spec:
  containers:
    - command:
      - kube-controller-manager
      - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
      - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
      - --root-ca-file=/etc/kubernetes/pki/ca.crt
      - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
```

> File CA (key + cert) là tài sản nhạy cảm nhất cluster — ai chiếm được nó có thể tự ký certificate với bất kỳ quyền nào (kể cả `system:masters`).

### etcd Hardening

etcd lưu **toàn bộ state của cluster** (bao gồm Secrets ở dạng base64, không tự động mã hóa) — tài sản nhạy cảm chỉ sau CA, cần hardening riêng ngoài việc có TLS certificate như đã cấu hình ở mục certificate phía trên.

**1. etcd chỉ nên listen trên localhost hoặc private network** — không bao giờ expose ra public interface:

```bash
ss -tlnp | grep 2379
# Kỳ vọng: 127.0.0.1:2379 hoặc private IP — KHÔNG phải 0.0.0.0:2379
```

**2. Bắt buộc client certificate khi gọi etcd** (`--client-cert-auth=true` như đã khai ở mục certificate) — verify bằng curl:

```bash
# Có cert hợp lệ — thành công
curl --cacert /etc/kubernetes/pki/etcd/ca.crt \
     --cert /etc/kubernetes/pki/apiserver-etcd-client.crt \
     --key /etc/kubernetes/pki/apiserver-etcd-client.key \
     https://localhost:2379/health

# Không có cert — phải bị từ chối
curl --cacert /etc/kubernetes/pki/etcd/ca.crt https://localhost:2379/health
```

> Xem thêm [[Minimize Microservice Vulnerabilities]] cho **Encryption at Rest** (mã hóa Secrets trong etcd bằng `EncryptionConfiguration`) và key rotation chi tiết — không lặp lại nội dung đó ở đây.

## Authentication, Authorization và RBAC

### Authentication cơ bản (Static Password/Token File)

Cách đơn giản nhất (không khuyến nghị cho production) — dùng CSV file chứa `password,username,uid,group`:

```csv
password123,user1,u0001,group1
password123,user2,u0002,group1
```

```bash
kube-apiserver --basic-auth-file=user-details.csv
```

Tương tự với static token file (`token,username,uid,group`):

```csv
KpjCVbI7cFAHYPkByTIzRb7gulcUc4B,user10,u0010,group1
```

```bash
kube-apiserver --token-auth-file=user-token-details.csv
```

```bash
curl -v -k https://master-node-ip:6443/api/v1/pods \
  --header "Authorization: Bearer KpjCVbI7cFAHYPkByTIzRb7gulcUc4B"
```

Static file authentication lưu **plain text** — chỉ dùng để học/lab, không dùng production. Ưu tiên certificate-based auth hoặc identity provider (LDAP/Kerberos/OIDC).

### Authorization modes

Kubernetes hỗ trợ nhiều authorization mode, có thể kết hợp nhiều mode cùng lúc (comma-separated, **thứ tự có ý nghĩa**):

| Mode | Mô tả |
|---|---|
| `AlwaysAllow` | Cho phép mọi request không kiểm tra — **mặc định nếu không set flag** |
| `AlwaysDeny` | Chặn mọi request |
| `Node` | Dùng cho request từ kubelet — user có tên `system:node:<node>` thuộc group `system:nodes` |
| `ABAC` (Attribute-Based) | Policy dạng JSON gắn permission trực tiếp cho từng user — khó quản lý khi user tăng, mỗi lần đổi phải sửa file + restart apiserver |
| `RBAC` (Role-Based) | Định nghĩa Role rồi gán user vào Role — thay đổi Role tự động áp dụng cho mọi user gắn Role đó |
| `Webhook` | Ủy quyền quyết định cho hệ thống bên ngoài (vd Open Policy Agent) |

```bash
# Cấu hình nhiều mode: Node kiểm tra trước, RBAC, cuối cùng Webhook
--authorization-mode=Node,RBAC,Webhook
```

Khi có nhiều mode: apiserver duyệt qua **theo đúng thứ tự khai báo**. Module nào **cho phép (allow)** thì dừng lại ngay, các module còn lại không được xét nữa. Nếu một module **deny/không có ý kiến**, request được chuyển tiếp cho module kế tiếp.

### API Groups (nền tảng của RBAC)

Toàn bộ API Kubernetes được tổ chức theo nhóm:

- **Core API Group** (`/api/v1`): namespaces, pods, replication controllers, events, endpoints, nodes, PV/PVC, configmaps, secrets, services.
- **Named API Group** (`/apis/<group>`): apps (deployments, replicasets, statefulsets), extensions, networking (network policies), storage, authentication, authorization, certificates...

Mỗi resource hỗ trợ tập **verbs**: `list`, `get`, `create`, `delete`, `update`, `watch` — đây chính là các action dùng trong `rules.verbs` của Role/ClusterRole.

```bash
# Xem danh sách API path không cần auth (giới hạn)
curl https://kube-master:6443/version
curl http://localhost:6443 -k    # liệt kê /api, /apis, /healthz, /metrics, /logs...

# Truy cập có certificate
curl http://localhost:6443 -k --key admin.key --cert admin.crt --cacert ca.crt
```

> `kube-proxy` ≠ `kubectl proxy`. `kube-proxy` quản lý network rule giữa pod/service trên node; `kubectl proxy` là HTTP proxy cục bộ để truy cập an toàn tới apiserver bằng credential trong kubeconfig — hai khái niệm hoàn toàn khác nhau, hay bị nhầm trên đề thi.

### RBAC — Role và RoleBinding (namespaced)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: default          # Role luôn scoped theo namespace (mặc định "default" nếu không khai)
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create -f developer-role.yaml
kubectl create -f devuser-developer-binding.yaml

kubectl get roles
kubectl get rolebindings
kubectl describe role developer
kubectl describe rolebinding devuser-developer-binding
```

**Kiểm tra quyền** — công cụ quan trọng nhất khi debug RBAC trên đề thi:

```bash
kubectl auth can-i create deployments
kubectl auth can-i delete nodes
kubectl auth can-i create deployments --as=dev-user   # kiểm tra thay cho user khác
kubectl auth can-i create pods --as=dev-user --namespace=dev
```

**Giới hạn theo resourceName cụ thể** (fine-grained hơn):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "create", "update"]
  resourceNames: ["blue", "orange"]     # chỉ áp dụng cho 2 pod này
```

### ClusterRole và ClusterRoleBinding (cluster-scoped)

Kubernetes resource chia làm 2 loại:

```bash
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

- **Namespaced**: pods, replicasets, jobs, deployments, services, secrets...
- **Cluster-scoped**: nodes, PV, namespaces, certificatesigningrequests...

Role/RoleBinding chỉ áp dụng trong 1 namespace → không dùng được cho resource cluster-scoped (như nodes, PV). Phải dùng **ClusterRole**/**ClusterRoleBinding**:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "get", "create", "delete"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

Lưu ý quan trọng: ClusterRole **cũng có thể** gán permission cho resource namespaced (vd pods) — khi đó permission áp dụng **trên toàn bộ mọi namespace**, khác với Role thường chỉ giới hạn 1 namespace. Kubernetes tự tạo sẵn nhiều default ClusterRole (vd `cluster-admin`, `admin`, `edit`, `view`) — thường gặp trong đề thi.

### Aggregated ClusterRoles

Thay vì sửa trực tiếp ClusterRole mặc định (`view`, `edit`, `admin`) mỗi khi có CRD mới cần cấp quyền, dùng label `rbac.authorization.k8s.io/aggregate-to-<role>` để tự động **hợp nhất** rule của ClusterRole tùy biến vào ClusterRole mặc định — có 1 controller riêng trong kube-controller-manager theo dõi label này và tự merge:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: custom-view-extension
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"   # merge rule vào ClusterRole "view"
rules:
  - apiGroups: ["monitoring.coreos.com"]
    resources: ["prometheuses", "servicemonitors"]
    verbs: ["get", "list", "watch"]
```

Ưu điểm: không cần `kubectl edit` trực tiếp ClusterRole `view`/`edit`/`admin` (dễ bị ghi đè khi upgrade cluster) — mỗi CRD/component tự khai ClusterRole nhỏ riêng kèm label aggregate, hệ thống tự gộp lại.

### Truy vết User/Group và Audit quyền RBAC (thực chiến)

Kubernetes **không lưu user như một resource** (không có `kubectl get users`) — danh tính chỉ tồn tại trong certificate. Muốn biết 1 certificate thuộc user/group nào, đọc trực tiếp CN/O:

```bash
openssl x509 -in nghia.crt -noout -subject
# subject=CN = nghia, O = viettel-ops
#            ↑ username     ↑ group
```

Vì không có resource "user", muốn liệt kê **toàn bộ user/group đang có quyền trong cluster** phải trace ngược từ RoleBindings/ClusterRoleBindings:

```bash
# Toàn bộ user có quyền (từ ClusterRoleBindings)
kubectl get clusterrolebindings -o json | jq -r '
.items[] | select(.subjects[]?.kind=="User") |
{binding: .metadata.name, user: .subjects[].name, role: .roleRef.name}'

# Toàn bộ group có quyền
kubectl get clusterrolebindings -o json | jq -r '
.items[] | select(.subjects[]?.kind=="Group") |
{binding: .metadata.name, group: .subjects[].name, role: .roleRef.name}'

# User/group trong 1 namespace cụ thể (RoleBindings)
kubectl get rolebindings -n production -o json | jq -r '
.items[] | {binding: .metadata.name, subjects: .subjects, role: .roleRef.name}'
```

Script tổng hợp — tìm nhanh ai đang có `cluster-admin` (mục tiêu audit số 1):

```bash
#!/bin/bash
# rbac-audit.sh — liệt kê toàn bộ permission trong cluster

echo "=== Users/groups có cluster-admin ==="
kubectl get clusterrolebindings -o json | jq -r '
.items[] | select(.roleRef.name=="cluster-admin") |
{binding: .metadata.name, subjects: .subjects}'

echo "=== Toàn bộ ClusterRoleBindings ==="
kubectl get clusterrolebindings -o custom-columns=\
NAME:.metadata.name,ROLE:.roleRef.name,\
SUBJECT_KIND:.subjects[*].kind,SUBJECT_NAME:.subjects[*].name

echo "=== Quyền rủi ro cao (admin/edit/cluster-admin) ==="
kubectl get clusterrolebindings -o json | jq -r '
.items[] | select(.roleRef.name | test("admin|edit|cluster-admin")) |
{binding: .metadata.name, role: .roleRef.name, subjects: .subjects}'
```

Kiểm tra quyền của riêng 1 ServiceAccount (hữu ích khi audit workload có bị over-privileged không):

```bash
kubectl auth can-i --list --as=system:serviceaccount:default:myapp-sa
kubectl auth can-i "*" "*" --as=system:serviceaccount:default:myapp-sa   # có phải cluster-admin trá hình?
```

### Service Accounts

Có 2 loại account: **User Account** (người dùng: admin, developer) và **Service Account** (ứng dụng/máy: Prometheus, Jenkins, dashboard...).

```bash
kubectl create serviceaccount dashboard-sa
kubectl get serviceaccount
kubectl describe serviceaccount dashboard-sa
```

Gán service account cho pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  serviceAccountName: dashboard-sa
  containers:
    - name: my-kubernetes-dashboard
      image: my-kubernetes-dashboard
```

Nếu không khai `serviceAccountName`, pod tự động dùng **default service account** của namespace, token được mount tại `/var/run/secrets/kubernetes.io/serviceaccount/` (`ca.crt`, `namespace`, `token`).

Muốn **tắt auto-mount** token (best practice cho pod không cần gọi API):

```yaml
spec:
  automountServiceAccountToken: false
```

Có thể tắt ở **2 cấp**: trên chính `ServiceAccount` (áp dụng cho mọi pod dùng SA đó) hoặc trên từng `Pod` (override):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  namespace: production
automountServiceAccountToken: false   # mọi pod dùng SA này mặc định không mount token
```

Nếu 1 pod cụ thể vẫn cần gọi API nhưng muốn dùng token **giới hạn audience + thời hạn ngắn** thay vì mount token mặc định, khai trực tiếp projected volume trong pod spec:

```yaml
spec:
  serviceAccountName: myapp
  automountServiceAccountToken: false
  volumes:
    - name: kube-api-access
      projected:
        sources:
          - serviceAccountToken:
              audience: https://kubernetes.default.svc.cluster.local
              expirationSeconds: 3600    # short-lived, khác với token mặc định
              path: token
          - configMap:
              name: kube-root-ca.crt
              items:
                - key: ca.crt
                  path: ca.crt
```

**Thay đổi lớn qua các version (hay bị hỏi trên đề thi):**

- **Trước v1.22**: mỗi service account tự động sinh 1 Secret chứa token **không có hạn (unbounded)** — rủi ro bảo mật nếu token bị rò rỉ vĩnh viễn hợp lệ.
- **Từ v1.22** (KEP 1205 — TokenRequest API): token sinh ra là **audience-bound + time-bound + object-bound**, được mount qua **projected volume** thay vì Secret tĩnh:

```yaml
volumes:
  - name: kube-api-access-6mtg8
    projected:
      defaultMode: 420
      sources:
        - serviceAccountToken:
            expirationSeconds: 3607
            path: token
        - configMap:
            name: kube-root-ca.crt
            items:
              - key: ca.crt
                path: ca.crt
        - downwardAPI:
            items:
              - fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.namespace
```

- **Từ v1.24**: service account **không còn tự sinh Secret** nữa. Muốn lấy token phải gọi TokenRequest API tường minh:

```bash
kubectl create token dashboard-sa   # token có hạn, mặc định 1 giờ
```

Nếu thật sự cần token không hết hạn (không khuyến nghị), tạo Secret thủ công với annotation:

```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: dashboard-sa
```

### Dashboard Security

Kubernetes Dashboard (nếu triển khai) là một attack vector phổ biến khi cấu hình sai — dashboard mặc định chạy với quyền khá rộng, nếu expose ra ngoài kẻ tấn công không cần credential vẫn có thể thao tác cluster.

**Không bao giờ** dùng các flag sau trên production:

```yaml
args:
  - --enable-skip-login              # cho phép bấm "Skip" bỏ qua đăng nhập — tuyệt đối không bật
  - --disable-settings-authorizer    # tắt authorization cho phần Settings — mất kiểm soát quyền
```

Cấu hình an toàn: bắt buộc đăng nhập bằng token, không expose ra ngoài qua NodePort/LoadBalancer:

```yaml
args:
  - --auto-generate-certificates
  - --namespace=kubernetes-dashboard
  - --authentication-mode=token      # bắt buộc token, không cho skip
  - --tls-cert-file=/certs/tls.crt
  - --tls-key-file=/certs/tls.key
```

Truy cập **chỉ qua `kubectl proxy`** (không expose ra internet) và luôn tạo token với quyền tối thiểu (vd `view` thay vì `cluster-admin`) thay vì dùng token admin có sẵn:

```bash
kubectl proxy
# http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

# Tạo service account + token với quyền tối thiểu để đăng nhập dashboard
kubectl create serviceaccount dashboard-viewer -n kubernetes-dashboard
kubectl create clusterrolebinding dashboard-viewer \
  --clusterrole=view \
  --serviceaccount=kubernetes-dashboard:dashboard-viewer
kubectl create token dashboard-viewer -n kubernetes-dashboard --duration=1h
```

### KubeConfig — quản lý credential gọn gàng

Thay vì gõ `--client-key`, `--client-certificate`, `--certificate-authority`, `--server` mỗi lần chạy `kubectl`, gom hết vào file kubeconfig (mặc định `$HOME/.kube/config`, hoặc chỉ định bằng `--kubeconfig`).

Cấu trúc gồm 3 phần chính:

- **clusters**: các cluster có thể truy cập (dev, test, prod...)
- **users**: các credential (admin, dev-user, prod-user...)
- **contexts**: kết hợp `cluster` + `user` (+ `namespace` tùy chọn) thành 1 "hồ sơ truy cập", vd `admin@production`

```yaml
apiVersion: v1
kind: Config
current-context: admin@production
clusters:
- name: production
  cluster:
    certificate-authority: ca.crt
    server: https://172.17.0.51:6443
contexts:
- name: admin@production
  context:
    cluster: production
    user: admin
    namespace: finance        # namespace mặc định khi dùng context này
users:
- name: admin
  user:
    client-certificate: admin.crt
    client-key: admin.key
```

```bash
kubectl config view
kubectl config view --kubeconfig=my-custom-config
kubectl config use-context prod-user@production   # đổi current-context
```

Có thể nhúng thẳng nội dung certificate base64 thay vì trỏ file path, dùng field `*-data`:

```yaml
clusters:
- name: production
  cluster:
    certificate-authority: /etc/kubernetes/pki/ca.crt
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJU...
```

### kube-apiserver Hardening Flags — bảng tham chiếu đầy đủ

Tổng hợp các flag hardening quan trọng của kube-apiserver theo từng nhóm chức năng (nhiều flag đã xuất hiện rải rác ở các mục trên — đây là bảng tham chiếu nhanh khi audit `/etc/kubernetes/manifests/kube-apiserver.yaml`):

| Nhóm | Flag | Ý nghĩa |
|---|---|---|
| Authentication | `--anonymous-auth=false` | Tắt anonymous access |
| Authentication (OIDC) | `--oidc-issuer-url`, `--oidc-client-id`, `--oidc-username-claim`, `--oidc-groups-claim` | Tích hợp identity provider ngoài (Keycloak, Dex...) qua OpenID Connect |
| Authorization | `--authorization-mode=Node,RBAC` | Không dùng `AlwaysAllow`/`ABAC` |
| TLS | `--tls-min-version=VersionTLS13` | Chặn handshake TLS phiên bản cũ |
| TLS | `--tls-cipher-suites=...` | Giới hạn cipher suite mạnh |
| TLS | `--client-ca-file`, `--kubelet-certificate-authority`, `--kubelet-client-certificate`, `--kubelet-client-key` | CA và cert dùng khi apiserver xác thực/được xác thực với kubelet |
| Bề mặt tấn công | `--profiling=false` | Tắt endpoint `/debug/pprof` |
| Bề mặt tấn công | `--insecure-port=0` | Tắt insecure port (mặc định đã là 0 từ v1.20+) |
| Admission | `--enable-admission-plugins=NodeRestriction,PodSecurity,ResourceQuota,LimitRanger` | Bật các admission controller hardening |
| Audit | `--audit-policy-file`, `--audit-log-path`, `--audit-log-maxage`, `--audit-log-maxbackup`, `--audit-log-maxsize` | Cấu hình audit logging (chi tiết ở mục Auditing) |
| Service Account | `--service-account-issuer`, `--service-account-key-file`, `--service-account-signing-key-file` | Ký và verify token của ServiceAccount |
| etcd client | `--etcd-cafile`, `--etcd-certfile`, `--etcd-keyfile` | TLS khi apiserver gọi tới etcd |

Verify flag đang active trên cluster đang chạy:

```bash
# Xem process đang chạy với flag nào
ps aux | grep kube-apiserver | tr ' ' '\n' | grep -E "^\-\-"

# Test anonymous access — phải trả 401
curl -k https://localhost:6443/api/v1/pods
# → {"kind":"Status","status":"Failure","message":"Unauthorized","code":401}

# Test profiling endpoint — phải bị chặn nếu đã tắt
curl -k https://localhost:6443/debug/pprof/
```

### Kubectl Proxy và Port Forward

**`kubectl proxy`**: mở HTTP proxy cục bộ (mặc định port 8001, bind vào `127.0.0.1` — chỉ truy cập được từ máy local), tự dùng credential từ kubeconfig để forward request tới apiserver — không cần truyền cert thủ công.

```bash
kubectl proxy
# Starting to serve on 127.0.0.1:8001

curl http://localhost:8001 -k
# Truy cập service nội bộ (ClusterIP) qua API server proxy path
curl http://localhost:8001/api/v1/namespaces/default/services/nginx/proxy/
```

**`kubectl port-forward`**: map thẳng 1 local port tới port của pod/service trong cluster, không đi qua toàn bộ API path.

```bash
kubectl port-forward service/nginx 28080:80
curl http://localhost:28080/
```

Không có auth (anonymous) sẽ bị từ chối:

```bash
curl http://<kube-api-server-ip>:6443 -k
# "message": "forbidden: User \"system:anonymous\" cannot get path \"/\""
```

## Kubelet Security

Kubelet đóng vai trò như "thuyền trưởng" của node — nhận lệnh triển khai pod từ apiserver, giao cho container runtime, rồi báo cáo trạng thái liên tục. Nếu ai đó giả mạo apiserver hoặc truy cập trái phép kubelet API, họ có thể đọc thông tin nhạy cảm hoặc điều khiển container trên node.

Kubelet mặc định phục vụ trên 2 port:

| Port | Chức năng | Rủi ro mặc định |
|---|---|---|
| **10250** | Full API access (exec, logs, port-forward...) | Anonymous access mặc định được cho phép |
| **10255** | Read-only API (metrics) | Không cần xác thực — expose dữ liệu nhạy cảm |

```bash
curl -sk http://localhost:10250/pods      # full API, anonymous mặc định enable
curl -sk http://localhost:10255/metrics   # read-only, hoàn toàn không auth
```

### Hardening kubelet — 4 bước

**1. Tắt anonymous authentication:**

```bash
# kubelet.service
--anonymous-auth=false
```
```yaml
# kubelet-config.yaml
authentication:
  anonymous:
    enabled: false
```

**2. Bật certificate-based authentication:**

```bash
--client-ca-file=/path/to/ca.crt
```
```yaml
authentication:
  x509:
    clientCAFile: /path/to/ca.crt
```

Khi đó apiserver (đóng vai trò client với kubelet) phải cấu hình client cert riêng:

```bash
--kubelet-client-certificate=/path/to/kubelet-cert.pem
--kubelet-client-key=/path/to/kubelet-key.pem
```

> Nếu cả certificate lẫn token auth đều không tường minh từ chối request, kubelet **fallback về anonymous**. Do đó luôn phải khai rõ `--anonymous-auth=false`.

**3. Bật authorization mode Webhook** (thay vì mặc định `AlwaysAllow`):

```bash
--authorization-mode=Webhook
```
```yaml
authorization:
  mode: Webhook
```

**4. Tắt read-only port (10255):**

```bash
--read-only-port=0
```
```yaml
readOnlyPort: 0
```

### Kubelet configuration file

Từ v1.10, phần lớn flag được chuyển vào file config (`--config`) thay vì command-line — dùng **camelCase** thay vì kebab-case:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
clusterDomain: cluster.local
clusterDNS:
  - 10.96.0.10
healthzPort: 10248
rotateCertificates: true
staticPodPath: /etc/kubernetes/manifests
```

Nếu 1 tham số vừa đặt trong file config vừa đặt trên command-line, **command-line flag luôn thắng**. `kubeadm` không tự cài kubelet nhưng quản lý config file này qua `kubeadm join`.

**Thêm 3 field hardening hay bị bỏ sót** vào cùng file config trên:

```yaml
protectKernelDefaults: true          # CKS: kubelet refuse start nếu kernel param bị thay đổi khác giá trị mong đợi — ngăn container/process sửa kernel param của node
eventRecordQPS: 5                    # giới hạn tốc độ ghi event, tránh event flood làm quá tải apiserver (0 = không giới hạn, rủi ro DoS)
streamingConnectionIdleTimeout: 5m   # tự đóng connection idle của exec/attach/port-forward — tránh session bị treo vô hạn nếu bị chiếm quyền
```

`protectKernelDefaults: true` đặc biệt quan trọng: nếu 1 process (kể cả container privileged) cố sửa kernel parameter (vd qua `sysctl`) khác với giá trị kubelet kỳ vọng, kubelet sẽ **crash/refuse to start** thay vì âm thầm chấp nhận — biến việc kernel bị tamper thành lỗi rõ ràng thay vì lỗ hổng ẩn.

## Bảo mật Node Metadata

Node metadata (labels, annotations, system info, IP, taints, kubelet version...) là thông tin nhạy cảm — nếu bị chỉnh sửa/đọc trái phép có thể dẫn tới:

1. **Workload bị schedule sai chỗ**: vd taint bảo vệ node production bị gỡ nhầm → workload không phù hợp bị schedule vào.

   ```bash
   kubectl taint nodes node-1 key=value:NoSchedule-   # gỡ taint (dấu "-" cuối)
   ```

2. **Lộ thông tin nhạy cảm** (vd kubelet version) → dùng để tấn công theo lỗ hổng đã biết của version đó:

   ```bash
   kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.kubeletVersion}'
   ```

3. **Dựng bản đồ mạng nội bộ** từ danh sách IP node → phục vụ tấn công DDoS có chủ đích:

   ```bash
   kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
   ```

4. **Vi phạm compliance** (GDPR, HIPAA) nếu thông tin hệ thống (kernel version...) bị lộ:

   ```bash
   kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.kernelVersion}'
   ```

Node metadata gồm: Node Name/UID, Labels (vd `region: us-east-1`), Annotations (vd dữ liệu nội bộ của Flannel `flannel.alpha.coreos.com/backend-data`), Architecture, System Info (Machine ID, System UUID, Boot ID, Kernel Version, OS Image, Container Runtime Version, Kubelet Version), Addresses (internal/external IP), Node Conditions, Resource Capacity, Taints/Tolerations, Pod CIDR, các external ID theo cloud provider (EC2/GCE/Azure).

**Biện pháp bảo vệ**: giới hạn quyền `get/list nodes` qua RBAC nghiêm ngặt, audit log mọi thay đổi metadata, không cấp quyền đọc `nodes` rộng rãi cho user thường.

**Chặn pod truy cập cloud metadata endpoint bằng NetworkPolicy** — nhiều cloud provider (AWS/GCP/Azure) expose instance metadata (có thể chứa IAM credentials) tại địa chỉ link-local cố định `169.254.169.254`; AWS ECS/EKS còn có thêm `169.254.170.2` cho credential endpoint. Đây là target kinh điển để pod bị compromise leo thang chiếm quyền cloud:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-cloud-metadata
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32    # AWS/GCP/Azure instance metadata
              - 169.254.170.2/32      # AWS ECS/EKS credential endpoint
```

## Protection Strategies (tổng hợp — ẩn dụ khách sạn)

Bài học dùng ẩn dụ khách sạn để tóm tắt các lớp phòng thủ:

- **RBAC** ≈ phân quyền nhân viên khách sạn (quản lý vào được mọi phòng kể cả phòng an ninh; nhân viên dọn phòng chỉ vào được phòng khách).
- **Node isolation** ≈ dành riêng phòng VIP cho khách VIP — dùng taints/tolerations, nodeSelector để cô lập node cho workload nhạy cảm.
- **Network Policy** ≈ khu vực chỉ dành cho nhân viên / tầng VIP riêng — kiểm soát ai được giao tiếp với ai.
- **Audit log** ≈ sổ ghi ai vào phòng nào lúc nào — phục vụ security monitoring và compliance.
- **Cập nhật định kỳ** ≈ nâng cấp khóa/camera an ninh — vá lỗi bảo mật kịp thời trên node.

## Network Policy

### Khái niệm Ingress/Egress

Với luồng traffic User → Web (port 80) → API (port 5000) → DB (port 3306):

- **Ingress** = traffic đi vào (hướng bắt nguồn traffic, không tính traffic phản hồi).
- **Egress** = traffic đi ra.

| Component | Ingress | Egress |
|---|---|---|
| Web | port 80 từ user | tới API port 5000 |
| API | port 5000 từ Web | tới DB port 3306 |
| DB | port 3306 từ API | (không cần) |

### Default allow-all và NetworkPolicy

Mặc định Kubernetes cho phép **tất cả pod giao tiếp tự do** với nhau qua IP/tên/service (không cần cấu hình route riêng). `NetworkPolicy` là object dùng label selector để giới hạn traffic.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
    ports:
    - protocol: TCP
      port: 3306
```

Chỉ khi `policyTypes` khai rõ `Ingress`/`Egress` thì hướng traffic đó mới bị cô lập — không khai thì hướng đó **không bị giới hạn** (vẫn allow-all). Traffic ingress được allow thì response traffic tự động được phép — **không cần** khai thêm egress rule chỉ để cho phép response.

### Kết hợp namespaceSelector + podSelector (AND vs OR)

```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        name: api-pod
    namespaceSelector:               # cùng 1 item -> AND (phải thỏa cả 2)
      matchLabels:
        name: prod
  ports:
  - protocol: TCP
    port: 3306
```

**Lỗi thường gặp**: tách `podSelector` và `namespaceSelector` thành 2 phần tử khác nhau trong list (`- podSelector` và `- namespaceSelector` riêng dấu `-`) sẽ biến thành quan hệ **OR** — vô tình mở rộng quyền truy cập hơn dự tính. Đây là bẫy hay gặp trên đề thi.

### ipBlock — cho phép nguồn ngoài cluster

```yaml
ingress:
- from:
  - ipBlock:
      cidr: 192.168.5.10/32
  ports:
  - protocol: TCP
    port: 3306
```

Kết hợp nhiều selector (AND trong 1 item, OR giữa các item) và cả `Egress`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
      namespaceSelector:
        matchLabels:
          name: prod
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 3306
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 80
```

> **NetworkPolicy chỉ có tác dụng nếu CNI plugin hỗ trợ nó.** Calico, Weave Net, Kube-router, Romana hỗ trợ; **Flannel không hỗ trợ**. Nếu dùng CNI không hỗ trợ, `kubectl apply` policy vẫn thành công (không báo lỗi) nhưng **hoàn toàn không được enforce** — đây là bẫy kinh điển trên đề thi CKS.

### NetworkPolicy Advanced Patterns

**Default Deny All** — baseline nên áp dụng cho mọi namespace production, sau đó mở dần theo nhu cầu thay vì bắt đầu từ allow-all:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}      # match mọi pod trong namespace
  policyTypes:
    - Ingress
    - Egress
  # Không khai ingress/egress rules → deny tất cả
```

```bash
# Áp dụng cho toàn bộ namespace, trừ các namespace hệ thống
for ns in $(kubectl get namespaces -o name | grep -vE 'kube-system|kube-public|kube-node-lease'); do
  kubectl apply -f default-deny-all.yaml -n ${ns##*/}
done
```

**Allow DNS** — bắt buộc phải có nếu dùng default-deny egress, nếu không pod sẽ không resolve được service name:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
      to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
```

**Multi-tier app (frontend → backend → db)** — mỗi tầng chỉ nhận traffic từ tầng liền trước, chỉ gửi traffic tới tầng liền sau (deny-by-default giữa các tầng không liền kề):

```yaml
# frontend — nhận từ ingress-nginx, chỉ gửi tới backend + DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports: [{port: 8080}]
  egress:
    - to: [{podSelector: {matchLabels: {app: backend}}}]
      ports: [{port: 8080}]
    - ports: [{port: 53, protocol: UDP}]
---
# backend — chỉ nhận từ frontend, chỉ gửi tới database + DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes: [Ingress, Egress]
  ingress:
    - from: [{podSelector: {matchLabels: {app: frontend}}}]
      ports: [{port: 8080}]
  egress:
    - to: [{podSelector: {matchLabels: {app: database}}}]
      ports: [{port: 5432}]
    - ports: [{port: 53, protocol: UDP}]
---
# database — chỉ nhận từ backend, egress chỉ cho DNS (strict nhất)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes: [Ingress, Egress]
  ingress:
    - from: [{podSelector: {matchLabels: {app: backend}}}]
      ports: [{port: 5432}]
  egress:
    - ports: [{port: 53, protocol: UDP}]
```

**Cross-namespace** (vd cho phép Prometheus ở namespace `monitoring` scrape metrics của pod ở namespace `production`) — lưu ý selector cùng 1 item là AND như đã nêu ở trên:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-scrape
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:              # cùng item với namespaceSelector ở trên -> AND
            matchLabels:
              app: prometheus
      ports: [{port: 9090}]
```

**ipBlock với `except`** — cho egress ra 1 dải CIDR ngoại trừ vài IP cụ thể trong dải đó:

```yaml
egress:
  - to:
      - ipBlock:
          cidr: 203.0.113.0/24
          except:
            - 203.0.113.100/32
    ports:
      - port: 443
```

**Testing NetworkPolicy** — luôn verify policy thực sự enforce sau khi apply, đừng chỉ tin vào YAML:

```bash
# Test từ cùng namespace — phải thành công nếu được allow
kubectl run test-client --image=busybox --rm -it -- \
  wget -qO- http://backend-service:8080/health

# Test từ namespace khác không được allow — phải timeout
kubectl run test-client -n staging --image=busybox --rm -it -- \
  wget --timeout=2 -qO- http://backend-service.production:8080/health
```

## Ingress

NodePort/LoadBalancer service đủ dùng cho 1 ứng dụng đơn giản, nhưng khi có nhiều service (vd `/wear`, `/watch` trên cùng domain) thì việc dựng nhiều LoadBalancer riêng biệt tốn kém và khó quản lý SSL tập trung. **Ingress** giải quyết bài toán này: hoạt động như **layer-7 load balancer** built-in, dùng object Kubernetes để định nghĩa routing rule (theo path hoặc theo hostname) và xử lý SSL termination tập trung.

> Kubernetes **không có sẵn Ingress controller** — Ingress resource chỉ là "luật", cần deploy 1 controller (NGINX, GCE, Contour, HAProxy, Traefik, Istio...) để thực thi luật đó. Tạo Ingress resource mà chưa có controller thì **không có tác dụng gì**.

### Triển khai NGINX Ingress Controller (thành phần đầy đủ)

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      name: http
    - port: 443
      targetPort: 443
      name: https
  selector:
    name: nginx-ingress
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
```

### Ingress resource — routing theo path

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  rules:
  - http:
      paths:
      - path: /wear
        backend:
          serviceName: wear-service
          servicePort: 80
      - path: /watch
        backend:
          serviceName: watch-service
          servicePort: 80
```

### Ingress resource — routing theo domain (host)

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  rules:
  - host: wear.my-online-store.com
    http:
      paths:
      - backend:
          serviceName: wear-service
          servicePort: 80
  - host: watch.my-online-store.com
    http:
      paths:
      - backend:
          serviceName: watch-service
          servicePort: 80
```

```bash
kubectl describe ingress ingress-wear-watch
```

Nếu URL không khớp rule nào, có thể khai `backend` mặc định để trả trang 404 tùy chỉnh.

### TLS Termination tại Ingress

Ingress có thể vừa route traffic vừa xử lý TLS termination tập trung — chỉ cần 1 Secret chứa cert/key và annotation trên Ingress resource (ví dụ dưới đây dùng NGINX Ingress Controller):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-tls
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"          # ép HTTPS
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "Strict-Transport-Security: max-age=31536000; includeSubDomains; preload";
spec:
  ingressClassName: nginx
  tls:
    - hosts: [myapp.example.com]
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port: {number: 80}
```

### cert-manager (tự động hóa TLS)

Thay vì tự tay renew certificate, **cert-manager** tự động xin và gia hạn cert (vd từ Let's Encrypt) rồi ghi thẳng vào Secret mà Ingress tham chiếu:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-cert
  namespace: production
spec:
  secretName: myapp-tls          # cert-manager tự tạo/renew Secret này
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - myapp.example.com
  duration: 2160h                # 90 ngày (mặc định Let's Encrypt)
  renewBefore: 360h              # renew trước 15 ngày khi hết hạn
```

Hoặc chỉ cần khai annotation trực tiếp trên Ingress mà không cần tạo `Certificate` resource riêng:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts: [myapp.example.com]
      secretName: myapp-tls
```

```bash
kubectl get certificate -n production
kubectl describe certificate myapp-cert -n production   # → Ready: True, Not Before/After
```

### Mutual TLS tại tầng Ingress

Bắt client phải trình certificate hợp lệ trước khi Ingress forward request vào backend — hữu ích khi cần xác thực 2 chiều cho API nội bộ mà không muốn triển khai cả service mesh:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-secret: "production/client-ca"
    nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
    nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
```

### Network Security Checklist

```bash
# 1. Mọi namespace production đều có default-deny-all?
kubectl get networkpolicy -A | grep default-deny

# 2. Test policy thực sự enforce từ nhiều namespace
kubectl run nettest --image=busybox --rm -it -n staging -- \
  wget --timeout=2 -O- http://backend.production:8080
# → phải timeout hoặc refused

# 3. Cert của Ingress còn hạn không?
kubectl get secret -A | grep tls | \
  xargs -I{} kubectl get secret {} -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -enddate

# 4. Inspect traffic thực tế khi nghi ngờ policy sai (cần debug container có tcpdump)
kubectl debug -n production node/<node-name> -it --image=nicolaka/netshoot -- \
  tcpdump -i any -w /tmp/capture.pcap port 8080
```

## Docker Daemon Security

### Rủi ro khi Docker daemon không được bảo vệ

Nếu kẻ tấn công truy cập được Docker daemon: xóa container/volume (mất dữ liệu, downtime), chạy container tùy ý (vd đào coin trái phép), hoặc chiếm quyền root trên host — từ đó lan ra toàn mạng.

Mặc định, Docker API chỉ bind vào **Unix socket** (`/var/run/docker.sock`) — chỉ user đăng nhập local mới thao tác được. Rủi ro chỉ phát sinh khi cấu hình expose ra TCP.

### Hardening host trước khi expose daemon

- Tắt đăng nhập root trực tiếp.
- Giới hạn truy cập cho user tin cậy.
- Dùng SSH key-based auth thay vì password.
- Đóng/giới hạn các port mạng không dùng tới.

### Expose Docker daemon qua TCP + TLS

```json
// /etc/docker/daemon.json — chỉ bind interface nội bộ (không nên bind public interface)
{
  "hosts": [ "tcp://192.168.1.10:2375" ]
}
```

> Cổng **2375** = plain TCP (không mã hóa) — chỉ nên dùng nội bộ, tuyệt đối không expose ra ngoài. Cổng **2376** = TCP + TLS.

Bật TLS + certificate-based auth (khuyến nghị bắt buộc khi expose ra ngoài):

```json
{
  "hosts": ["tcp://192.168.1.10:2376"],
  "tls": true,
  "tlscert": "/var/docker/server.pem",
  "tlskey": "/var/docker/serverkey.pem",
  "tlsverify": true,
  "tlscacert": "/var/docker/cacert.pem"
}
```

> **TLS chỉ mã hóa đường truyền — không tự động xác thực client.** Phải bật `tlsverify: true` kèm certificate CA-signed cho từng client mới thật sự chặn được truy cập trái phép. Đây là điểm hay bị nhầm: "có TLS = an toàn" là sai nếu thiếu `tlsverify`.

Phía client:

```bash
export DOCKER_TLS_VERIFY=true
export DOCKER_HOST="tcp://192.168.1.10:2376"
docker --tlscert=<path_to_cert> --tlskey=<path_to_key> --tlscacert=<path_to_ca_cert> ps
```

### Quản lý Docker daemon qua systemd

```bash
systemctl status docker
systemctl start docker
systemctl stop docker

# Chạy foreground để debug (log in console trực tiếp)
dockerd --debug
```

```bash
dockerd --debug --host=tcp://192.168.1.10:2375
```

> Nếu 1 tham số vừa khai trong `daemon.json` vừa khai trên command-line, Docker sẽ **báo conflict và không khởi động được** — khác với kubelet (nơi command-line thắng). Đây là điểm khác biệt dễ nhầm giữa 2 công cụ.

## Auditing

Audit log ghi lại **ai làm gì, khi nào, ở đâu, thay đổi gì** trong cluster — phục vụ security monitoring, compliance (GDPR, HIPAA, PCI DSS), và troubleshooting.

Ví dụ audit event:

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "stage": "ResponseComplete",
  "requestURI": "/apis/apps/v1/namespaces/default/deployments",
  "verb": "create",
  "user": { "username": "admin", "groups": ["system:masters", "system:authenticated"] },
  "sourceIPs": ["192.168.1.10"],
  "objectRef": { "resource": "deployments", "namespace": "default", "name": "nginx-deployment" },
  "responseStatus": { "code": 201 },
  "annotations": {
    "authorization.k8s.io/decision": "allow",
    "authorization.k8s.io/reason": "RBAC: allowed by ClusterRoleBinding \"cluster-admin\"..."
  }
}
```

### 4 mức audit level

| Level | Ghi gì |
|---|---|
| `None` | Không ghi gì |
| `Metadata` | Chỉ metadata (user, timestamp, resource, verb) — không có request/response body |
| `Request` | Metadata + request body — không có response body |
| `RequestResponse` | Metadata + request body + response body (chi tiết nhất, tốn dung lượng nhất) |

### Audit Policy YAML

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  verbs: ["get", "list", "create", "delete", "update"]
  resources:
  - group: ""
    resources: ["pods"]
```

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Request
  verbs: ["create", "delete", "update"]
  resources:
  - group: ""
    resources: ["pods"]
```

Audit policy được nạp vào kube-apiserver qua flag (không có trong nội dung gốc lấy được nhưng cần nhớ cho đề thi): `--audit-policy-file` và `--audit-log-path`.

### Query thực tế trên audit log (cheat-sheet)

Audit log (`--audit-log-path`) là file JSON-lines, mỗi dòng 1 event — dùng `jq` để lọc theo nhu cầu điều tra:

```bash
# 1. Toàn bộ hành động của 1 user cụ thể
sudo jq -r 'select(.user.username=="nghia")' /var/log/kubernetes/audit.log

# 2. Ai đã xóa pod trong namespace production?
sudo jq -r 'select(.verb=="delete" and
  .objectRef.resource=="pods" and
  .objectRef.namespace=="production")' /var/log/kubernetes/audit.log

# 3. Ai đã tạo/sửa ClusterRoleBinding? (dấu hiệu leo thang quyền)
sudo jq -r 'select(.objectRef.resource=="clusterrolebindings" and
  (.verb=="create" or .verb=="update" or .verb=="patch"))' /var/log/kubernetes/audit.log

# 4. Các lần authenticate/authorize thất bại
sudo jq -r 'select(.responseStatus.code >= 400 and .stage=="ResponseComplete")' \
  /var/log/kubernetes/audit.log

# 5. Hành động xuất phát từ 1 IP cụ thể (vd IP lạ, không thuộc dải nội bộ)
sudo jq -r 'select(.sourceIPs[] | contains("192.168.1.100"))' /var/log/kubernetes/audit.log
```

### Ship audit log ra hệ thống ngoài

Log ghi ra file trên control plane node dễ bị mất khi node down hoặc bị kẻ tấn công xóa dấu vết — production nên ship ra hệ thống ngoài.

**Fluent Bit → Elasticsearch/Loki** (tail file audit log, forward ra log aggregator):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: kube-system
data:
  fluent-bit.conf: |
    [INPUT]
        Name              tail
        Path              /var/log/kubernetes/audit.log
        Parser            json
        Tag               kube.audit
        Refresh_Interval  5

    [OUTPUT]
        Name   es
        Match  kube.audit
        Host   elasticsearch.logging.svc
        Port   9200
        Index  kubernetes-audit
```

**Audit webhook → Splunk/Datadog** (apiserver tự POST event tới endpoint ngoài, không cần tail file):

```bash
# Thêm vào kube-apiserver
- --audit-webhook-config-file=/etc/kubernetes/audit-webhook.yaml
- --audit-webhook-batch-max-wait=5s
```

```yaml
# /etc/kubernetes/audit-webhook.yaml
apiVersion: v1
kind: Config
clusters:
- name: splunk
  cluster:
    server: https://splunk.internal:8088/services/collector
contexts:
- context:
    cluster: splunk
    user: ""
  name: default-context
current-context: default-context
```

> Runtime security monitoring theo thời gian thực (phát hiện exec vào container, đọc secret, tạo privileged pod...) dùng Falco — xem chi tiết đầy đủ ở [[Monitoring Logging and Runtime Security]], không lặp lại ở đây.

## CIS Benchmarks và kube-bench

### CIS Benchmark là gì

**CIS (Center for Internet Security)** là tổ chức phi lợi nhuận xây dựng best-practice bảo mật cho hơn 25 nhóm công nghệ (OS, cloud, mobile, network device, server software, virtualization — bao gồm Kubernetes và Docker). Mỗi khuyến nghị trong benchmark gồm: giải thích rủi ro, lệnh kiểm tra để phát hiện vấn đề, và hướng dẫn khắc phục.

Ví dụ 1 khuyến nghị: quyền file của API server pod spec phải là `644`:

```bash
stat -c %a /etc/kubernetes/manifests/kube-apiserver.yaml
chmod 644 /etc/kubernetes/manifests/kube-apiserver.yaml
```

Các khuyến nghị điển hình cho kube-apiserver: tắt anonymous authentication, không dùng basic-auth-file/token-auth-file, bắt buộc HTTPS + certificate hợp lệ.

**CIS CAT** (Configuration Assessment Tool) tự động so sánh cấu hình hệ thống với benchmark, sinh report HTML — nhưng bản miễn phí (lite) **không hỗ trợ Kubernetes** (chỉ Windows, Ubuntu, Chrome, macOS...). Với Kubernetes, dùng công cụ mã nguồn mở khác: **kube-bench**.

### kube-bench

Phát triển bởi **Aqua Security**, kube-bench tự động chấm điểm cluster theo từng mục CIS Benchmark (pass/fail cho từng recommendation).

Cách triển khai:

- Chạy như **Docker container**.
- Chạy như **Kubernetes Job** (phù hợp để giám sát định kỳ).
- Cài trực tiếp bằng **binary** hoặc **compile từ source** trên master node.

Quy trình: chọn version ổn định từ [GitHub kube-bench](https://github.com/aquasecurity/kube-bench) → cài trên master node → chạy assessment → review report pass/fail → khắc phục theo hướng dẫn → chạy lại để xác nhận đã fix.

### Key checks — bảng tham chiếu nhanh

Một số check ID hay gặp nhất trong đề thi và thực tế, cùng cách khắc phục:

| Check ID | Nội dung | Cách fix |
|---|---|---|
| 1.2.1 | `--anonymous-auth` phải là `false` | `--anonymous-auth=false` |
| 1.2.4 | `--profiling` phải là `false` | `--profiling=false` |
| 1.2.22 | Audit logging phải được bật | Thêm `--audit-policy-file`, `--audit-log-path` |
| 1.3.2 | etcd phải dùng TLS cert riêng | Kiểm tra `--etcd-certfile`/`--etcd-keyfile`/`--client-cert-auth=true` |
| 4.2.1 | Kubelet `--anonymous-auth` phải là `false` | `authentication.anonymous.enabled: false` |
| 4.2.2 | Kubelet authorization không được là `AlwaysAllow` | `authorization.mode: Webhook` |
| 4.2.4 | Kubelet `--read-only-port` phải là `0` | `readOnlyPort: 0` |

```bash
# Chạy lại đúng các check cụ thể sau khi remediate
kube-bench run --check 1.2.1,1.2.4,4.2.1

# Xuất JSON để tích hợp CI, chỉ lọc các check FAIL
kube-bench --json 2>/dev/null | jq '.[] | select(.state=="FAIL")'
```

> **kube-bench version phải match version Kubernetes** đang chạy (vd kube-bench 0.6.x cho K8s 1.24, 0.7.x+ cho 1.27+) — version lệch có thể cho kết quả sai lệch.

## Cluster Upgrade Process

### Quy tắc version skew

kube-apiserver là "mốc chuẩn" — không component nào được **vượt** version của apiserver.

| Component | Được phép lệch so với apiserver |
|---|---|
| kube-apiserver | (mốc chuẩn — version cao nhất) |
| controller-manager, scheduler | bằng hoặc thấp hơn **1 minor version** |
| kubelet, kube-proxy | bằng hoặc thấp hơn **2 minor version** |
| kubectl | cao hơn 1, bằng, hoặc thấp hơn 1 minor version (linh hoạt nhất) |

Kubernetes chỉ hỗ trợ **3 minor version gần nhất**. Ví dụ đang ở 1.10, khi 1.11/1.12 ra mắt vẫn còn hỗ trợ 1.10, 1.11, 1.12 — nhưng khi 1.13 ra mắt thì 1.10 rớt khỏi danh sách hỗ trợ. **Luôn nâng cấp từng 1 minor version một** (không nhảy cóc, vd 1.10 → 1.13 phải qua 1.11 rồi 1.12 rồi 1.13).

### Chiến lược nâng cấp worker node

1. **Upgrade toàn bộ cùng lúc** — đơn giản nhất nhưng gây downtime toàn bộ.
2. **Upgrade từng node một** (rolling) — pod tự động reschedule sang node còn lại, giảm downtime.
3. **Thêm node mới rồi migrate** — deploy node version mới, chuyển workload sang, rồi decommission node cũ. Hiệu quả nhất trên cloud.

### Quy trình dùng kubeadm

```bash
kubeadm upgrade plan          # xem version hiện tại, version khả dụng, việc cần làm thủ công (thường là kubelet)

apt-get upgrade -y kubeadm=1.12.0-00     # nâng kubeadm trước, đúng đúng 1 minor version
kubeadm upgrade apply v1.12.0            # áp dụng upgrade control plane

kubectl get nodes                        # kiểm tra — version hiển thị là version KUBELET, không phải control plane

# Nâng kubelet trên control plane node (nếu master cũng chạy kubelet)
apt-get upgrade -y kubelet=1.12.0-00
systemctl restart kubelet
```

Nâng từng worker node (tuần tự, an toàn):

```bash
kubectl drain node-1                              # evict pod + đánh dấu unschedulable
apt-get upgrade -y kubeadm=1.12.0-00
apt-get upgrade -y kubelet=1.12.0-00
kubeadm upgrade node config --kubelet-version v1.12.0
systemctl restart kubelet
kubectl uncordon node-1                           # mở lại scheduling
```

Lặp lại cho từng worker node còn lại.

### Demo thực tế: nâng cấp 1.28 → 1.29 (kubeadm, Ubuntu/Debian)

Từ 2023, repository package cũ (`apt.kubernetes.io`, `yum.kubernetes.io`) đã **deprecated**, phải chuyển sang `pkgs.k8s.io`:

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core/stable/v1.29/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo apt-get update
```

Xác định version mới nhất trong series:

```bash
sudo apt-cache madison kubeadm
```

**Bước 1 — nâng kubeadm trên control plane:**

```bash
sudo apt-mark unhold kubeadm && \
sudo apt-get update && \
sudo apt-get install -y kubeadm='1.29.3-1.1' && \
sudo apt-mark hold kubeadm
kubeadm version
```

**Bước 2 — dry-run kế hoạch:**

```bash
sudo kubeadm upgrade plan
```

**Bước 3 — áp dụng upgrade control plane:**

```bash
sudo kubeadm upgrade apply v1.29.3
```

**Nâng kubelet/kubectl trên control plane node:**

```bash
kubectl drain controlplane --ignore-daemonsets
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get install -y kubelet='1.29.3-1.1' kubectl='1.29.3-1.1' && \
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet
kubectl uncordon controlplane
```

**Nâng worker node** (chạy `kubeadm upgrade node`, không phải `apply` — chỉ control plane node mới `apply`):

```bash
# Trên worker: nâng kubeadm
sudo apt-mark unhold kubeadm && sudo apt-get install -y kubeadm='1.29.3-1.1' && sudo apt-mark hold kubeadm

# Trên control plane: khởi tạo upgrade cho worker
sudo kubeadm upgrade node

# Trên worker: drain, nâng kubelet/kubectl, restart, uncordon
kubectl drain <node-name> --ignore-daemonsets
sudo apt-mark unhold kubelet kubectl && sudo apt-get install -y kubelet='1.29.3-1.1' kubectl='1.29.3-1.1' && sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet
kubectl uncordon <node-name>
```

## Gotchas

- **CN vs O trong certificate**: `CN` (Common Name) = username Kubernetes dùng để authenticate; `O` (Organization) = group. Muốn cấp quyền `system:masters` (admin toàn cluster) cho user, thêm `/O=system:masters` vào subject của CSR — không có field "role" riêng trong certificate, RBAC hoàn toàn dựa vào CN/O.
- **`kube-proxy` ≠ `kubectl proxy`** — dễ nhầm tên trên đề thi. Cái đầu quản lý network rule pod/service; cái sau là HTTP proxy cục bộ dùng credential kubeconfig.
- **Authorization mode mặc định là `AlwaysAllow`** nếu không set `--authorization-mode` — một cluster "trần" không hardening sẽ cho phép mọi request.
- **Nhiều authorization mode xử lý theo đúng thứ tự khai báo**, dừng ngay khi có module nào **allow** (không chờ tất cả). Ví dụ `Node,RBAC,Webhook`: Node xét trước, RBAC xét tiếp nếu Node không quyết, Webhook là chốt cuối.
- **Kubelet port 10255 (read-only) không cần auth theo mặc định** — luôn set `--read-only-port=0` khi hardening; nhiều đề thi/kịch bản compromise khai thác đúng port này.
- **Nếu kubelet auth không rõ ràng chấp nhận/từ chối, nó fallback về anonymous** — luôn phải set `--anonymous-auth=false` tường minh, không chỉ dựa vào bật certificate auth.
- **kubelet-config.yaml dùng camelCase**, còn command-line flag dùng kebab-case (`httpCheckFrequency` vs `--http-check-frequency`). Khi trùng, **command-line thắng**.
- **Docker daemon.json vs command-line flag**: nếu trùng tham số, Docker **báo lỗi conflict và không khởi động** — ngược lại hoàn toàn so với kubelet (nơi command-line âm thầm override).
- **Port 2375 (Docker) = không TLS**, **2376 = có TLS** — nhớ đúng số port này cho đề thi.
- **TLS không tự động = authentication**. Bật `tls: true` chỉ mã hóa đường truyền; phải thêm `tlsverify: true` + client certificate CA-signed mới chặn được truy cập trái phép.
- **NetworkPolicy chỉ có tác dụng khi CNI plugin hỗ trợ.** Flannel **không** hỗ trợ NetworkPolicy — tạo policy vẫn `kubectl apply` thành công nhưng im lặng không enforce gì cả. Calico, Weave Net, Kube-router, Romana có hỗ trợ.
- **Selector trong cùng 1 item của `from`/`to` là AND; tách thành nhiều item (nhiều dấu `-`) là OR.** Đây là lỗi cấu hình NetworkPolicy phổ biến nhất — vô tình mở rộng quyền truy cập.
- **`policyTypes` phải khai rõ Ingress/Egress** thì hướng đó mới bị cô lập (default deny cho hướng được khai, allow-all cho hướng không khai). Ingress traffic được allow thì response tự động đi qua — không cần thêm egress rule chỉ vì lý do đó.
- **Ingress cần Ingress Controller mới hoạt động** — Kubernetes không có controller mặc định; tạo Ingress resource suông không có tác dụng gì nếu chưa deploy controller (NGINX, GCE, Traefik...).
- **Version skew khi upgrade cluster**: apiserver luôn phải là version cao nhất; controller-manager/scheduler lệch tối đa 1 minor; kubelet/kube-proxy lệch tối đa 2 minor; kubectl linh hoạt ±1 minor. Luôn upgrade **từng 1 minor version một**, không nhảy cóc.
- **`kubeadm upgrade apply` chỉ chạy trên control plane node; worker node dùng `kubeadm upgrade node`.** Sau khi upgrade control plane, `kubectl get nodes` vẫn hiển thị version **kubelet** cũ cho tới khi kubelet thật sự được nâng cấp và restart.
- **Luôn `drain` trước khi nâng kubelet, `uncordon` sau khi xong** — quên uncordon là lỗi rất dễ mắc, khiến node bị treo ở trạng thái `SchedulingDisabled`.
- **Service account token đổi hành vi qua version**: trước v1.22 token không hạn (Secret tĩnh); từ v1.22 token audience/time-bound qua projected volume; từ v1.24 không tự sinh Secret nữa — phải `kubectl create token <sa>` tường minh hoặc tự tạo Secret với annotation `kubernetes.io/service-account.name`.
- **Quy ước tên file certificate**: public cert dùng `.crt`/`.pem`; private key dùng `.key` hoặc có chữ "key" trong tên (`server-key.pem`). Không có quy tắc bắt buộc về mặt kỹ thuật, nhưng đề thi và tài liệu luôn theo convention này.
- **kubeadm lưu static pod manifest ở `/etc/kubernetes/manifests/` và certificate ở `/etc/kubernetes/pki/`** theo mặc định — đây là 2 đường dẫn cần thuộc lòng để debug nhanh trên đề thi.
- **CIS CAT bản miễn phí không hỗ trợ Kubernetes** — công cụ đúng cho Kubernetes CIS Benchmark là **kube-bench** (Aqua Security), không phải CIS CAT.
- **Audit policy có 4 level tăng dần chi tiết**: `None` → `Metadata` → `Request` → `RequestResponse`. Càng chi tiết càng tốn dung lượng log — chọn level phù hợp theo resource, đừng mặc định dùng `RequestResponse` cho mọi thứ.
- **Kubelet `readOnlyPort: 0` có thể làm gãy monitoring cũ** nếu agent nào đó vẫn scrape port 10255 — kiểm tra và chuyển các agent sang dùng port 10250 (có auth) trước khi tắt hẳn 10255.
- **RBAC bị "quyền trôi dạt" (permission drift) theo thời gian**: quyền cấp tạm cho debug/incident không bị thu hồi sẽ tồn đọng mãi — nên có lịch audit RBAC định kỳ (vd hàng quý) bằng script liệt kê cluster-admin/admin/edit bindings thay vì chỉ audit khi có sự cố.
- **`kube-bench` version phải khớp version Kubernetes đang chạy** (vd kube-bench 0.6.x cho K8s 1.24, 0.7.x+ cho 1.27+) — version lệch có thể báo sai kết quả pass/fail.
- **`NodeRestriction` admission plugin chỉ hoạt động đúng nếu kubelet dùng certificate với CN đúng định dạng `system:node:<nodename>`** — nếu CN sai định dạng, kubelet không được nhận diện đúng và NodeRestriction không áp dụng được như kỳ vọng.
- **etcd tuyệt đối không nên listen trên `0.0.0.0`** — luôn kiểm tra bằng `ss -tlnp | grep 2379`, chỉ chấp nhận `127.0.0.1` hoặc private IP; kèm `--client-cert-auth=true` để bắt buộc client certificate.
- **Aggregated ClusterRole cần đúng label `rbac.authorization.k8s.io/aggregate-to-<role>: "true"`** — thiếu hoặc sai label này, rule trong ClusterRole tùy biến sẽ **không bao giờ** được merge vào `view`/`edit`/`admin`, âm thầm không có tác dụng mà không báo lỗi gì.
- **Dashboard: không bao giờ bật `--enable-skip-login` hay `--disable-settings-authorizer`** — 2 flag này biến dashboard thành cửa sau bỏ qua authentication/authorization hoàn toàn; luôn truy cập qua `kubectl proxy` và cấp token quyền tối thiểu, không expose dashboard ra ngoài bằng NodePort/LoadBalancer.
- **Default-deny egress mà quên allow DNS (port 53 UDP/TCP) → toàn bộ pod trong namespace mất khả năng resolve service name**, ứng dụng báo lỗi "connection refused" khó hiểu tưởng là bug ứng dụng — luôn deploy `allow-dns` policy cùng lúc với `default-deny-all`.
- **NetworkPolicy không áp dụng cho pod có `hostNetwork: true`** — các pod này dùng thẳng network stack của host nên bypass NetworkPolicy hoàn toàn; phải kiểm soát bằng firewall cấp OS (iptables/nftables) trên node.
- **cert-manager với wildcard certificate bắt buộc dùng DNS-01 challenge** (không dùng được HTTP-01) — cần cấu hình thêm DNS provider webhook cho cert-manager, quên bước này là nguyên nhân phổ biến khiến cert wildcard không issue được.
