
## Action flow
### 1. ETCD BACKUP & RESTORE💾

#### Backup etcd

```bash
# Step 1: Tìm etcd certificates (thường ở /etc/kubernetes/manifests/etcd.yaml)
cat /etc/kubernetes/manifests/etcd.yaml | grep -E 'cert|key|cacert|endpoints'

# Step 2: Backup
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Step 3: Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table
```

#### Restore etcd

```bash
# Step 1: Restore vào thư mục mới
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore

# Step 2: Update etcd manifest để point tới data-dir mới
vim /etc/kubernetes/manifests/etcd.yaml
# Sửa: --data-dir=/var/lib/etcd-restore
# Sửa volumeMounts path: /var/lib/etcd-restore

# Step 3: kubelet sẽ tự restart etcd pod (vì static pod)
# Hoặc force restart bằng cách move file ra ngoài rồi move vào lại
mv /etc/kubernetes/manifests/etcd.yaml /tmp/
# Đợi pod die
mv /tmp/etcd.yaml /etc/kubernetes/manifests/

# Step 4: Verify cluster health
k get nodes
k get pods -A
```

**Gotchas:**

- Phải dùng `ETCDCTL_API=3` (version 2 khác syntax)
- Cert paths khác nhau giữa các cluster → phải check manifest
- Restore tạo **data-dir mới** → phải update manifest

---

### 2. CLUSTER UPGRADE (kubeadm) 🚀

#### Upgrade Control Plane Node

```bash
# Step 1: Check upgrade plan
kubeadm upgrade plan

# Step 2: Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.28.x-00
apt-mark hold kubeadm

# Step 3: Verify kubeadm version
kubeadm version

# Step 4: Apply upgrade
kubeadm upgrade apply v1.28.x

# Step 5: Drain node (evacuate workloads)
k drain controlplane --ignore-daemonsets

# Step 6: Upgrade kubelet & kubectl
apt-mark unhold kubelet kubectl
apt-get update && apt-get install -y kubelet=1.28.x-00 kubectl=1.28.x-00
apt-mark hold kubelet kubectl

# Step 7: Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# Step 8: Uncordon node
k uncordon controlplane
```

#### Upgrade Worker Node

```bash
# (Trên control plane) Drain worker trước
k drain worker01 --ignore-daemonsets --delete-emptydir-data

# (SSH vào worker node) Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.28.x-00
apt-mark hold kubeadm

# Upgrade node config
kubeadm upgrade node

# Upgrade kubelet & kubectl
apt-mark unhold kubelet kubectl
apt-get update && apt-get install -y kubelet=1.28.x-00 kubectl=1.28.x-00
apt-mark hold kubelet kubectl

# Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# (Quay lại control plane) Uncordon worker
k uncordon worker01
```

**Gotchas:**

- Upgrade **control plane trước**, worker sau
- Upgrade **1 minor version tại 1 thời điểm** (1.27 → 1.28, không được nhảy 1.27 → 1.29)
- Phải `drain` node trước khi upgrade (exam sẽ trừ điểm nếu quên)

---

### 3. NODE TROUBLESHOOTING 🔧

#### Node NotReady → Fix kubelet

```bash
# Step 1: Check node status
k get nodes

# Step 2: SSH vào node bị lỗi
ssh worker01

# Step 3: Check kubelet status
systemctl status kubelet

# Step 4: Nếu kubelet died → check logs
journalctl -u kubelet -f

# Common issues:
# - kubelet config sai → fix /var/lib/kubelet/config.yaml
# - Certificate expired → renew certs
# - Port conflict → check netstat -tulpn

# Step 5: Restart kubelet
systemctl restart kubelet

# Step 6: Verify
systemctl status kubelet
k get nodes
```

#### Static Pod không chạy

```bash
# Static pods nằm ở /etc/kubernetes/manifests/
ls /etc/kubernetes/manifests/

# Check logs
journalctl -u kubelet | grep -i error

# Common issues:
# - YAML syntax error → vim /etc/kubernetes/manifests/xxx.yaml
# - Image pull error → check imagePullPolicy
# - Port conflict → netstat -tulpn | grep <port>

# Fix xong kubelet tự restart pod (không cần k apply)
```

---

### 4. NETWORKING TROUBLESHOOTING 🌐

#### Pod không reach được Service

```bash
# Step 1: Test DNS resolution từ pod
k run test --image=busybox --rm -it -- nslookup my-service

# Step 2: Check Service endpoints
k get svc my-service
k get ep my-service  # phải có IP của pods

# Step 3: Check Pod labels match Service selector
k get pods --show-labels
k describe svc my-service | grep Selector

# Step 4: Test connectivity
k run test --image=busybox --rm -it -- wget -O- my-service:80
```

#### CoreDNS không hoạt động

```bash
# Check CoreDNS pods
k get pods -n kube-system | grep coredns

# Check logs
k logs -n kube-system coredns-xxxx

# Common fix: Restart CoreDNS
k rollout restart deploy/coredns -n kube-system

# Check ConfigMap
k get cm coredns -n kube-system -o yaml
```

---

### 5. SECURITY & RBAC 🔒

#### Create ServiceAccount & Bind Role

```bash
# Step 1: Create SA
k create sa app-sa -n dev

# Step 2: Create Role
k create role pod-reader --verb=get,list --resource=pods -n dev

# Step 3: Bind Role to SA
k create rolebinding pod-reader-binding \
  --role=pod-reader \
  --serviceaccount=dev:app-sa \
  -n dev

# Step 4: Test permissions (dùng --as để impersonate)
k auth can-i get pods --as=system:serviceaccount:dev:app-sa -n dev
```

#### Create NetworkPolicy (isolate pod)

```bash
# Example: Chỉ cho phép traffic từ pods có label app=frontend
cat <<EOF | k apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: dev
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
EOF
```

---

### 6. PERSISTENT STORAGE 💾

#### Create PV → PVC → Mount vào Pod

```bash
# Step 1: Create PV (admin task)
cat <<EOF | k apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /mnt/data
EOF

# Step 2: Create PVC (user task)
cat <<EOF | k apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
EOF

# Step 3: Mount vào Pod
k run webapp --image=nginx --dry-run=client -o yaml > pod.yaml
# Edit pod.yaml thêm:
#   volumes:
#   - name: data
#     persistentVolumeClaim:
#       claimName: my-pvc
#   volumeMounts:
#   - name: data
#     mountPath: /data
k apply -f pod.yaml
```

---

### 7. LOGGING & MONITORING 📊

#### Get logs từ multi-container pod

```bash
# List containers trong pod
k get pod my-pod -o jsonpath='{.spec.containers[*].name}'

# Get logs từ specific container
k logs my-pod -c sidecar-container

# Tail logs real-time
k logs -f my-pod -c app

# Get logs từ previous crashed container
k logs my-pod -c app --previous
```

#### Top nodes/pods (cần metrics-server)

```bash
k top nodes
k top pods -A --sort-by=memory
k top pods -A --sort-by=cpu
```

---

### 8. JSONPATH QUERIES (CKA thích ra lắm) 🎯

```bash
# Get node internal IP
k get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# Get all pod IPs
k get pods -o jsonpath='{.items[*].status.podIP}'

# Get image của tất cả pods
k get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'

# Sort pods by restart count
k get pods --sort-by=.status.containerStatuses[0].restartCount

# Get pods not in Running state
k get pods -A --field-selector=status.phase!=Running
```

---

### PRO TIPS CHO TỪNG FLOW 💡

#### **ETCD Backup/Restore**

- Bookmark: `https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster`
- Cert paths khác nhau → **luôn check manifest trước**

#### **Cluster Upgrade**

- Bookmark: `https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/`
- **Drain → Upgrade → Uncordon** (thuộc vẹt flow này)

#### **Troubleshooting**

- **Luôn check logs trước**: `journalctl -u kubelet`, `k logs`
- Static pods → check `/etc/kubernetes/manifests/`
- DNS issues → restart CoreDNS

#### **RBAC**

- Test permissions: `k auth can-i <verb> <resource> --as=<user/sa>`

#### **JSONPath**

- Practice trước vì syntax khó nhớ
- Dùng `-o jsonpath='{...}'` hoặc `-o custom-columns=...`
