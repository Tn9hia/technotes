---
title: Kubernetes Production Operations
tags:
  - kubernetes
  - operations
  - etcd
  - velero
  - upgrade
  - multi-cluster
date: 2026-04-26
---

# Production Operations

## etcd Operations

etcd là database của K8s — mọi cluster state lưu ở đây. Operational hygiene quan trọng.

### etcd Architecture

```
3-node etcd cluster (minimum HA):
  etcd-1 (leader) ──── Raft consensus ────┐
  etcd-2 (follower) ──────────────────────┤
  etcd-3 (follower) ──────────────────────┘

  Quorum = (n/2) + 1
  3 nodes → quorum = 2 → tolerate 1 failure
  5 nodes → quorum = 3 → tolerate 2 failures

  KHÔNG dùng even number: 4 nodes → quorum = 3 (same as 3 but double the write cost)
```

### Compaction

etcd giữ history của tất cả key-value changes. History này tích lũy → tốn disk space. Compaction xóa historical revisions, giữ chỉ current state.

```bash
# Check current revision
ETCDCTL_API=3 etcdctl \
  --endpoints=https://localhost:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table

# Manual compaction (compact tới revision hiện tại)
REV=$(ETCDCTL_API=3 etcdctl \
  --endpoints=https://localhost:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=json | jq '.[] | .Status.header.revision')

ETCDCTL_API=3 etcdctl compact $REV \
  --endpoints=https://localhost:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

```yaml
# Auto-compaction (configure trong etcd config)
# /etc/kubernetes/manifests/etcd.yaml
spec:
  containers:
  - command:
    - etcd
    - --auto-compaction-mode=periodic
    - --auto-compaction-retention=8h    # compact history older than 8 hours
```

### Defrag

Sau compaction, disk space không được tự động free — cần defrag.

```bash
# Defrag từng member (không làm toàn bộ cùng lúc — có downtime ngắn)
# Làm follower trước, leader cuối

for endpoint in \
  https://etcd-2:2379 \
  https://etcd-3:2379 \
  https://etcd-1:2379; do  # etcd-1 là leader → cuối cùng
  echo "Defragmenting $endpoint"
  ETCDCTL_API=3 etcdctl defrag \
    --endpoints=$endpoint \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key
  sleep 5    # cho endpoint recover trước khi làm tiếp
done

# Verify size sau defrag
ETCDCTL_API=3 etcdctl endpoint status \
  --endpoints=https://etcd-1:2379,https://etcd-2:2379,https://etcd-3:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --write-out=table
```

### Backup & Restore

```bash
# Backup — snapshot toàn bộ etcd data
ETCDCTL_API=3 etcdctl snapshot save \
  /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://localhost:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-20260426.db \
  --write-out=table
# → Hash, Revision, Total Keys, Total Size

# Automated backup với CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 2 * * *"    # 2am daily
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          tolerations:
            - key: node-role.kubernetes.io/control-plane
              effect: NoSchedule
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          containers:
            - name: backup
              image: bitnami/etcd:3.5
              command:
                - sh
                - -c
                - |
                  ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
                    --endpoints=https://localhost:2379 \
                    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                    --cert=/etc/kubernetes/pki/etcd/server.crt \
                    --key=/etc/kubernetes/pki/etcd/server.key
                  # Upload to S3
                  aws s3 cp /backup/etcd-$(date +%Y%m%d).db s3://my-backup-bucket/etcd/
              volumeMounts:
                - mountPath: /etc/kubernetes/pki/etcd
                  name: etcd-certs
                  readOnly: true
                - mountPath: /backup
                  name: backup-dir
          volumes:
            - hostPath:
                path: /etc/kubernetes/pki/etcd
              name: etcd-certs
            - hostPath:
                path: /tmp
              name: backup-dir
```

```bash
# RESTORE (disaster recovery)
# 1. Stop kube-apiserver (move manifest ra ngoài)
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# 2. Restore snapshot
ETCDCTL_API=3 etcdctl snapshot restore \
  /backup/etcd-snapshot-20260426.db \
  --data-dir=/var/lib/etcd-restore \
  --name=etcd-1 \
  --initial-cluster="etcd-1=https://etcd-1:2380" \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://etcd-1:2380

# 3. Update etcd manifest data-dir
# Đổi --data-dir=/var/lib/etcd sang /var/lib/etcd-restore trong etcd.yaml

# 4. Restart etcd
# 5. Move kube-apiserver manifest back
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

---

## Velero — Cluster Backup

Velero backup toàn bộ K8s resources (YAML) và PersistentVolume data.

```bash
# Install Velero (AWS S3 backend)
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-velero-backup \
  --backup-location-config region=ap-southeast-1 \
  --snapshot-location-config region=ap-southeast-1 \
  --secret-file ./credentials-velero   # AWS credentials

# credentials-velero
[default]
aws_access_key_id=AKIAIOSFODNN7EXAMPLE
aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

```bash
# Manual backup
velero backup create production-backup \
  --include-namespaces production,staging \
  --snapshot-volumes \
  --ttl 720h          # giữ 30 ngày

# Scheduled backup
velero schedule create daily-backup \
  --schedule="0 1 * * *" \       # 1am daily
  --include-namespaces production \
  --snapshot-volumes \
  --ttl 720h

# Check backup status
velero backup describe production-backup --details
velero backup logs production-backup

# List backups
velero backup get
```

```bash
# Restore
velero restore create \
  --from-backup production-backup

# Restore specific namespace
velero restore create \
  --from-backup production-backup \
  --include-namespaces production

# Restore to different namespace
velero restore create \
  --from-backup production-backup \
  --namespace-mappings production:production-restore

# Verify restore
velero restore describe <restore-name>
```

---

## Cluster Upgrade Strategy

### In-Place Upgrade (kubeadm)

```bash
# Quy trình: upgrade control plane → upgrade workers (one at a time)

# 1. Check available versions
apt-cache madison kubeadm

# 2. Upgrade kubeadm
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.31.0-00
apt-mark hold kubeadm

# 3. Plan upgrade
kubeadm upgrade plan
# Output: version checks, etcd version, API server version

# 4. Apply upgrade (control plane)
kubeadm upgrade apply v1.31.0

# 5. Upgrade kubelet + kubectl trên control plane
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.31.0-00 kubectl=1.31.0-00
apt-mark hold kubelet kubectl
systemctl daemon-reload
systemctl restart kubelet

# 6. Upgrade worker nodes (one at a time)
# Drain node trước
kubectl drain node-worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data

# SSH vào worker node
apt-mark unhold kubeadm kubelet kubectl
apt-get install -y kubeadm=1.31.0-00 kubelet=1.31.0-00 kubectl=1.31.0-00
apt-mark hold kubeadm kubelet kubectl
kubeadm upgrade node
systemctl daemon-reload
systemctl restart kubelet

# Uncordon sau khi upgraded
kubectl uncordon node-worker-1

# Repeat cho các nodes còn lại
```

### Upgrade Strategy cho Managed K8s (EKS/GKE/AKS)

```bash
# EKS — upgrade cluster
eksctl upgrade cluster --name my-cluster --version 1.31 --approve

# Upgrade managed node groups (rolling)
eksctl upgrade nodegroup \
  --name workers \
  --cluster my-cluster \
  --kubernetes-version 1.31

# GKE — upgrade
gcloud container clusters upgrade my-cluster --master --cluster-version 1.31
gcloud container node-pools upgrade default-pool --cluster my-cluster

# AKS
az aks upgrade --resource-group myRG --name my-cluster --kubernetes-version 1.31
```

### Blue/Green Cluster Upgrade (zero-risk)

```
Approach: tạo cluster mới với version mới, migrate workloads

Old Cluster (v1.29) ──── migration ──── New Cluster (v1.31)

1. Provision new cluster v1.31
2. Deploy tất cả workloads sang new cluster
3. Migrate traffic (DNS cutover)
4. Run cả 2 clusters song song để verify
5. Decommission old cluster
```

Pros: zero-risk, easy rollback. Cons: cost (2x clusters), complexity.

---

## Certificate Rotation

```bash
# Check cert expiry
kubeadm certs check-expiration
# CERTIFICATE         EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY
# admin.conf          Nov 26, 2026 10:00 UTC   364d            ca
# apiserver           Nov 26, 2026 10:00 UTC   364d            ca
# ...

# Rotate tất cả certs (certificates expire sau 1 năm mặc định)
kubeadm certs renew all

# Restart control plane components sau khi renew
# (static pods tự restart khi detect cert change)

# Kiểm tra cert của specific component
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -enddate

# Tự động renew với cert-manager (cho in-cluster certs)
# kubeadm certs có auto-renew khi cert < 80% lifetime expired
```

---

## Multi-Cluster Patterns

### Tại sao Multi-Cluster?

```
Single cluster limitations:
- Hard blast radius isolation (production + staging in same cluster = risky)
- Scale limits (~5000 nodes, ~150k pods per cluster)
- Regulatory: data residency requirements
- Availability: cluster-level failure affects everything

Multi-cluster strategies:
1. Active-Active: traffic load-balanced giữa clusters
2. Active-Passive: primary cluster, DR cluster standby
3. Federation: single control plane, nhiều clusters
4. Regional: cluster per region
```

### Cluster Federation với ArgoCD

```yaml
# ArgoCD ApplicationSet — deploy tới nhiều clusters
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-global
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production
    # → tự động discover clusters trong ArgoCD với label này

  template:
    metadata:
      name: "myapp-{{name}}"       # name = cluster name
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/k8s-manifests
        targetRevision: HEAD
        path: apps/myapp
      destination:
        server: "{{server}}"       # cluster API endpoint
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### Istio Multi-Cluster (mTLS cross-cluster)

```bash
# Primary cluster setup
istioctl install \
  --set profile=default \
  --set values.global.multiCluster.clusterName=cluster-primary \
  --set values.global.network=network1

# Remote cluster setup
istioctl install \
  --set profile=remote \
  --set values.global.multiCluster.clusterName=cluster-remote \
  --set values.global.remotePilotAddress=<primary-pilot-ip> \
  --set values.global.network=network2

# Service discovery across clusters
# Services trong cluster-primary visible từ cluster-remote và ngược lại
# mTLS tự động apply cho cross-cluster traffic
```

### KubeFed (Kubernetes Federation)

```yaml
# FederatedDeployment — deploy tới multiple clusters từ một resource
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    spec:
      replicas: 3
      # ... standard deployment spec
  placement:
    clusters:
      - name: cluster-ap-southeast-1
      - name: cluster-ap-northeast-1
  overrides:
    - clusterName: cluster-ap-southeast-1
      clusterOverrides:
        - path: /spec/replicas
          value: 5    # more replicas in primary region
```

---

## Health Checks & Runbooks

```bash
# Daily cluster health check script
#!/bin/bash

echo "=== Node Status ==="
kubectl get nodes -o wide

echo "=== Control Plane Pods ==="
kubectl get pods -n kube-system -l tier=control-plane

echo "=== etcd Health ==="
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://etcd-1:2379,https://etcd-2:2379,https://etcd-3:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

echo "=== Certificate Expiry ==="
kubeadm certs check-expiration | grep -v "CERTIFICATE"

echo "=== Pending Pods ==="
kubectl get pods -A --field-selector=status.phase=Pending

echo "=== CrashLooping Pods ==="
kubectl get pods -A | grep -E "CrashLoop|Error|OOMKilled"

echo "=== PV Status ==="
kubectl get pv | grep -v Bound

echo "=== Resource Quotas ==="
kubectl get resourcequota -A

echo "=== Recent Events (warnings) ==="
kubectl get events -A --field-selector=type=Warning --sort-by='.lastTimestamp' | tail -20
```

---

## Gotchas

- **etcd compaction trước defrag**: Compaction xóa revisions (logical), defrag giải phóng disk space (physical). Cần làm cả hai. Chỉ defrag mà không compact = không free disk.
- **etcd defrag và leader election**: Defrag trên leader có thể trigger leader election (brief pause). Luôn defrag followers trước, leader cuối.
- **Velero và CRDs**: Velero backup CRD instances nhưng cần restore CRD definitions trước. Nếu restore vào empty cluster → phải restore CRDs trước, sau đó restore namespaced resources.
- **Cluster upgrade version skew policy**: kube-apiserver phải luôn >= kubelet version. kubelet có thể lag 2 minor versions. Controller manager/scheduler phải cùng version với apiserver. Vi phạm skew policy → undefined behavior.
- **Upgrade và PodDisruptionBudget**: Khi drain nodes cho upgrade, PDB prevent drain nếu would violate minAvailable. Cần đủ replicas và đủ nodes before drain. Script drain tự động retry → check PDB trước khi drain.
- **Multi-cluster và service discovery**: Services trong cluster A không tự visible trong cluster B. Cần Istio federation, CoreDNS stub zones, hoặc external service registry (Consul).
