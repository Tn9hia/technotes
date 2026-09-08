## Create simple resource with imperative command
### Workload and scheduling
#### Pod
- Create pod
```shell
# Pod (quick testing) 
k run nginx --image=nginx --port=80 --labels="app=web,env=prod" 

k run busybox --image=busybox --rm -it -- /bin/sh # ephemeral debug pod
```

#### Deployment
- Create deployment
```shell
k create deploy webapp --image=nginx --replicas=3 --port=80 

k create deploy api --image=myapi:v1 --dry-run=client -o yaml > deploy.yaml
```
- Scale deployment: `k scale`
```shell
kubectl scale deployment <name> --replicas=<number>
```
- Update deployment: `k set`
```shell
kubectl set image deployment/<deployment-name> <container-name>=<new-image>
```
- Rollout: `k rollout`
```shell
# Check status
kubectl rollout status deployment/<deployment-name>

# Check history
kubectl rollout history deployment/<deployment-name>

# Pause rollout 
kubectl rollout pause deployment/<deployment-name>

# Resume rollout 
kubectl rollout resume deployment/<deployment-name>

# Rollback
kubectl rollout undo deployment/<deployment-name>

# Restart pods (recreate without changing image) 
kubectl rollout restart deployment/<deployment-name>
```

#### Service
- Expose service: `k expose` or `k create svc`
```shell
# Service - ClusterIP (internal traffic)
k expose deploy webapp --port=80 --target-port=8080 --name=webapp-svc

# Service - NodePort (external access qua Node IP)
k expose deploy webapp --type=NodePort --port=80 --name=webapp-np

# Service - LoadBalancer (external IP từ cloud provider)
k expose deploy webapp --type=LoadBalancer --port=80 --name=webapp-lb
```

- Create service: `k create svc` or `k create ingress`
```shell
# Service - từ đầu (không cần expose)
k create svc clusterip my-svc --tcp=80:8080
k create svc nodeport my-np --tcp=80:8080 --node-port=30080

# Ingress (phải cài Ingress Controller trước nhé)
k create ingress webapp-ing --rule="example.com/api=api-svc:80" --class=nginx
```

#### Scheduling

- Job: `k create job`
```shell
k create job db-migration --image=migrate-tool -- ./migrate.sh
```

- Cron job: `k create cronjob`
```shell
k create cronjob backup --image=backup-tool --schedule="0 2 * * *" -- /backup.sh
```

### Services & Networking

#### Services
- Create services: `k create service`
```shell
kubectl create service <type> <name> --tcp=<port>
```

#### Network Policies
- Create network policies: `kubectl create networkpolicy`
```shell
kubectl create networkpolicy <policy-name> --namespace=<namespace> --spec=<spec>

kubectl create networkpolicy allow-frontend-to-backend \
  --pod-selector=app=backend \
  --ingress \
  --from=podSelector=app=frontend \
  --port=80 \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Security
#### User Security

```shell
# Role (namespace-scoped permissions)
k create role pod-reader --verb=get,list,watch --resource=pods

# ClusterRole (cluster-wide permissions)
k create clusterrole pod-reader --verb=get,list,watch --resource=pods

# RoleBinding (bind Role to User/SA)
k create rolebinding pod-reader-binding --role=pod-reader --serviceaccount=default:my-sa

# ClusterRoleBinding
k create clusterrolebinding pod-reader-binding --clusterrole=pod-reader --serviceaccount=default:my-sa
```
#### Application Security

- ConfigMap: `k create cm`
```shell
# ConfigMap (non-sensitive data) 
k create cm app-config --from-literal=DB_HOST=postgres --from-literal=DB_PORT=5432 

k create cm app-config --from-file=config.json # từ file 

k create cm app-config --from-env-file=.env # từ .env file
```

- Secret: `k create secret`
```shell
# Secret (sensitive data - base64 encoded) 
k create secret generic db-creds --from-literal=user=admin --from-literal=pass=secret123 

k create secret docker-registry regcred \ 
	--docker-server=myregistry.io \ 
	--docker-username=user \ 
	--docker-password=pass \ 
	--docker-email=user@example.com 

# TLS Secret (for Ingress) 
k create secret tls my-tls --cert=cert.pem --key=key.pem
```

- Namespace: `k create sa`
```shell
# ServiceAccount
k create sa my-sa
```

- ServiceAccount: `k create ns`
```shell
# Namespace k create ns dev 
k create ns prod 

# ResourceQuota (limit resources per namespace) 
k create quota dev-quota --hard=cpu=10,memory=20Gi,pods=50 -n dev
```

### Maintenance cluster
- Drain node `k drain`
```shell
kubectl drain <node-name> --ignore-daemonsets --delete-local-data
```
- Cordon node `k cordon`
```shell
kubectl uncordon <node-name>
```
- Uncordon node
```shell
kubectl uncordon <node-name>
```

### Backup/Restore
#### Backup etcd Manually

```shell
ETCDCTL_API=3 etcdctl snapshot save <backup-file> \
--endpoints=https://127.0.0.1:2379 \
--cacert=<path-to-cafile> \
--cert=<path-to-certfile> \
--key=<path-to-keyfile>
```

#### Restore etcd manually
```shell
ETCDCTL_API=3 etcdctl snapshot restore <backup-file> \
  --data-dir=/var/lib/etcd-from-backup
```

#### Backup Using kube-apiserver
- Backup 
```shell
# Backup kube controller manager
kubectl get --raw /apis/coordination.k8s.io/v1/namespaces/kube-system/leases/kube-controller-manager -o json > kube-controller-manager.json

# Backup kube scheduller
kubectl get --raw /apis/coordination.k8s.io/v1/namespaces/kube-system/leases/kube-scheduler -o json > kube-scheduler.json
```
- Restore
```shell
# Restore kube controller manager
kubectl apply -f kube-controller-manager.json

# Restore kube scheduller
kubectl apply -f kube-scheduler.json
```


### Upgrade cluster
- Upgrade the Control Plane
```shell
kubeadm upgrade plan
kubeadm upgrade apply <version>
```

- Upgrade kubelet and kubectl
```shell
apt-get update && apt-get install -y kubelet=<version> kubectl=<version>
systemctl restart kubelet
```
- Upgrade kubelet and kubectl
```shell
kubectl drain <node-name> --ignore-daemonsets --delete-local-data
kubeadm upgrade node
kubectl uncordon <node-name>
```

> [!note]
> More detail about cluster upgrade procedure, follow [[Upgrade Cluster | this article]]


### OpenSSL Certificate
#### Certificates
OpenSSL Commands for Generating Keys and Certificates
- Generate a Private Key for the CA
```
openssl genrsa -out ca.key 2048
```

- Create a Self-Signed Certificate for the CA
Generates a self-signed CA certificate valid for 365 days. You will be prompted to enter information about the CA.

```
openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 -out ca.crt
```

#### **Generating Public and Private Keys**
- Generate a Private Key
Generates a 2048-bit private key.
```
openssl genrsa -out server.key 2048
```

- Create a Certificate Signing Request (CSR)
Generates a CSR using the private key. You will be prompted to enter information about the certificate.
```
openssl req -new -key server.key -out server.csr
```

- Sign the CSR with the CA to Create the Certificate
Signs the CSR with the CA’s private key to generate a certificate valid for 365 days.

```
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256
```

#### **Viewing the Public Key Information**
- View Public Key Information from the Certificate
Displays the content of the certificate, including the public key information.
```
openssl x509 -in server.crt -text -noout
```

- Extract the Public Key from the Private Key
Extracts the public key from the private key and saves it to a file.
```
openssl rsa -in server.key -pubout -out server_public.key
```
- View Public Key Information from the Public Key File
Displays the content of the public key file.
```
openssl rsa -pubin -in server_public.key -text -noout
```


- Other cheatcode

```shell
# Set alias ngay từ đầu
alias k=kubectl
export do="--dry-run=client -o yaml"  # k run nginx --image=nginx $do

# Generate YAML rồi edit (faster than typing from scratch)
k create deploy webapp --image=nginx $do > deploy.yaml
vim deploy.yaml  # edit rồi apply

# Force replace (khi object đã tồn tại)
k replace --force -f pod.yaml

# Quick edit in-place
k edit deploy webapp
k set image deploy/webapp nginx=nginx:1.21

# Scale nhanh
k scale deploy webapp --replicas=5

# Autoscale
k autoscale deploy webapp --min=2 --max=10 --cpu-percent=80

# Expose port cho debugging
k port-forward svc/webapp 8080:80

# Copy file vào/ra pod
k cp localfile.txt mypod:/tmp/
k cp mypod:/tmp/remotefile.txt ./localfile.txt
```


## ConfigMap & Secret Mount Methods trong K8s 🎯

Có **2 cách chính** để inject ConfigMap/Secret vào Pod, mỗi cách có sub-methods riêng:

---

### 1. Environment Variables

### 1a. Từng key một (`env.valueFrom`)

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:        # hoặc secretKeyRef
        name: my-config
        key: database_host
```

### 1b. Toàn bộ CM/Secret (`envFrom`)

```yaml
envFrom:
  - configMapRef:
      name: my-config
  - secretRef:
      name: my-secret
```

> ⚠️ Dùng `envFrom` thì tất cả keys của CM/Secret đều thành env vars — cẩn thận conflict tên.

---

### 2. Volume Mount (file-based)

### 2a. Mount toàn bộ CM/Secret → directory

```yaml
volumes:
  - name: config-vol
    configMap:
      name: my-config       # hoặc secret: secretName: my-secret

containers:
  - volumeMounts:
      - name: config-vol
        mountPath: /etc/config
```

> Mỗi key → 1 file trong `/etc/config/`

### 2b. Mount từng key → file cụ thể (`items`)

```yaml
volumes:
  - name: config-vol
    configMap:
      name: my-config
      items:
        - key: database_host
          path: db/host.conf   # custom filename/path
```

### 2c. Mount vào file đơn lẻ (`subPath`)

```yaml
volumeMounts:
  - name: config-vol
    mountPath: /etc/nginx/nginx.conf
    subPath: nginx.conf        # chỉ mount 1 file, không overwrite cả dir
```

> ⚠️ CKA hay hỏi `subPath` lắm — nhớ kỹ cái này. Nhược điểm: **không tự update** khi CM thay đổi.

### 2d. Set permission cho file (`defaultMode`)

```yaml
volumes:
  - name: secret-vol
    secret:
      secretName: my-secret
      defaultMode: 0400       # read-only cho owner
```

---

### So sánh nhanh

```
┌─────────────────┬──────────────┬─────────────────────────────┐
│ Method          │ Access in    │ Live update khi CM đổi?     │
├─────────────────┼──────────────┼─────────────────────────────┤
│ env/envFrom     │ $ENV_VAR     │ ❌ Cần restart pod           │
│ Volume (full)   │ file         │ ✅ ~1 phút (kubelet sync)   │
│ Volume (subPath)│ file         │ ❌ Cần restart pod           │
└─────────────────┴──────────────┴─────────────────────────────┘
```

---

### Bonus: Secret-specific lưu ý cho CKA 🔐

- Secret được encode **base64**, không phải encrypt — đừng nhầm
- Trong exam hay có bài: tạo secret từ literal rồi mount → nhớ `kubectl create secret generic`
- `imagePullSecrets` là dạng Secret đặc biệt dùng riêng cho pull private registry, mount khác hoàn toàn:

```yaml
spec:
  imagePullSecrets:
    - name: registry-secret
```

## Time management
**Đợt 1: Speed run (45-50 phút)**

- Quét qua TẤT CẢ các câu, làm ngay những câu "dễ ăn":
    - Pod/Deployment cơ bản
    - ConfigMap/Secret
    - Service expose
    - kubectl commands đơn giản
- Mục tiêu: Chốt được 40-50% điểm trong đợt này
- Rule: Nếu câu nào đọc xong mà không biết ngay cách làm → **SKIP**, đánh dấu lại

**Đợt 2: Grind the mid-tier (40-45 phút)**

- Quay lại làm các câu medium:
    - RBAC/ServiceAccount
    - PV/PVC
    - NetworkPolicy
    - Troubleshooting các resource bị lỗi
- Mục tiêu: Cày thêm 30-35% điểm
- Nếu câu nào mắc quá 8-10 phút → GHI CHÚ LẠI và next

**Đợt 3: Boss fight (20-25 phút)**

- Tấn công các câu khó/điểm cao:
    - Cluster upgrade
    - etcd backup/restore
    - Multi-container pods phức tạp
- Ưu tiên câu nào điểm cao hơn
- Chấp nhận bỏ 1-2 câu nếu quá khó

**Đợt 4: Final check (10 phút cuối)**

- Review lại các câu đã làm
- Verify namespaces, context switching (dính sai namespace = 0 điểm câu đó)
- Double-check typo trong YAML

**Time per question guideline:**

- Easy (1-2%): 3-5 phút
- Medium (4-7%): 8-12 phút
- Hard (10-13%): 15-20 phút

**Câu nào weight thấp (<4%) mà mắc quá 10 phút → BỎ QUA luôn**, không đáng trade-off!
## Best strategy
### Hybrid approach

```bash
# Tạo base manifest nhanh bằng imperative + dry-run
k run web --image=nginx --port=80 $do > pod.yaml

# Edit nhanh những gì cần (VD: add resources, env)
vim pod.yaml

# Apply
k apply -f pod.yaml
```

### Practice speed typing commands
```bash
# Thay vì nhớ full command, nhớ pattern:
k run <name> --image=<img> [options]
k create deploy <name> --image=<img> --replicas=N
k expose deploy <name> --port=X --target-port=Y
k create cm <name> --from-literal=key=value
```

### Red flag question
- "NetworkPolicy"
- "Ingress with TLS"
- "PersistentVolume with accessModes"
- "RBAC with specific verbs"
- "Pod Security Policy"
- "Upgrade cluster" (kubeadm upgrade docs)