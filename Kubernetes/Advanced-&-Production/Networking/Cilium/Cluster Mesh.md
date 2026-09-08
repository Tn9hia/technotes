---
title: Cilium Cluster Mesh
tags:
  - cilium
  - cluster-mesh
  - multi-cluster
  - high-availability
date: 2026-04-30
---

# Cilium Cluster Mesh

Cluster Mesh kết nối nhiều Kubernetes clusters thành một logical network — pods trên cluster A có thể reach pods trên cluster B bằng DNS/service name, như thể cùng cluster.

```
┌─────────────────────────────┐      ┌─────────────────────────────┐
│      Cluster A (primary)    │      │      Cluster B (failover)   │
│      10.0.0.0/8             │      │      10.1.0.0/8             │
│                             │      │                             │
│  Service: backend           │      │  Service: backend           │
│  (global: true)             │      │  (global: true)             │
│  10.96.10.50                │      │  10.97.10.50                │
│                             │      │                             │
│  Cilium Mesh API Server     │◄────►│  Cilium Mesh API Server     │
│  (exposed endpoint)         │      │  (exposed endpoint)         │
└─────────────────────────────┘      └─────────────────────────────┘

Client trong Cluster A → backend.default.svc.cluster.local
→ Resolved as "global service"
→ Load balanced giữa endpoints trong BOTH clusters
```

---

## Setup Cluster Mesh

### Prerequisites

```bash
# Mỗi cluster cần:
# 1. Unique cluster name và clusterID (1-255)
# 2. Non-overlapping pod CIDRs
# 3. Cilium cài đặt với clustermesh.useAPIServer=true
# 4. Network connectivity giữa nodes của các clusters

# Cluster A
helm upgrade cilium cilium/cilium \
  --set cluster.name=cluster-a \
  --set cluster.id=1 \
  --set clustermesh.useAPIServer=true \
  --set clustermesh.apiserver.replicas=2 \
  --reuse-values

# Cluster B
helm upgrade cilium cilium/cilium \
  --set cluster.name=cluster-b \
  --set cluster.id=2 \
  --set clustermesh.useAPIServer=true \
  --set clustermesh.apiserver.replicas=2 \
  --reuse-values
```

### Connect Clusters

```bash
# Dùng cilium CLI để connect (từ môi trường có access cả 2 cluster kubeconfig)
cilium clustermesh connect \
  --context cluster-a \
  --destination-context cluster-b

# Verify connection
cilium clustermesh status --context cluster-a
# Cluster ID: 1
# Cluster Name: cluster-a
# Connected Clusters: cluster-b (Ready)

# Xem tất cả clusters connected
kubectl exec -n kube-system <cilium-pod> -- \
  cilium-dbg debuginfo | grep -A 20 "clustermesh"
```

---

## Global Services

Global Services là Services được expose tới tất cả clusters trong mesh.

### Tạo Global Service

```yaml
# Service trong Cluster A
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: production
  annotations:
    service.cilium.io/global: "true"     # expose globally
    # service.cilium.io/shared: "false"  # có thể set false để chỉ nhận traffic, không contribute endpoints
spec:
  selector:
    app: backend
  ports:
    - port: 8080
      targetPort: 8080
```

```yaml
# Service cùng tên, cùng namespace trong Cluster B
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: production
  annotations:
    service.cilium.io/global: "true"
spec:
  selector:
    app: backend
  ports:
    - port: 8080
      targetPort: 8080
```

Khi cả hai đều có annotation `global: "true"`, Cilium load balance tự động qua endpoints của cả 2 clusters.

---

## Affinity-Based Routing

```yaml
# Ưu tiên endpoints trong cluster local — chỉ failover qua cluster khác khi local không có
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: production
  annotations:
    service.cilium.io/global: "true"
    service.cilium.io/affinity: "local"    # local | remote | none
    # local: ưu tiên endpoints trong same cluster
    # remote: ưu tiên endpoints trong remote clusters
    # none: load balance đều (default)
spec:
  selector:
    app: backend
  ports:
    - port: 8080
```

### Failover Pattern

```yaml
# Cluster A: primary (local affinity)
annotations:
  service.cilium.io/global: "true"
  service.cilium.io/affinity: "local"

# Behavior:
# → Cluster A có pods healthy → 100% traffic tới Cluster A
# → Cluster A pods = 0 (all down) → failover tới Cluster B
# → Cluster A pods recover → traffic tự động back
```

---

## Cross-Cluster NetworkPolicy

```yaml
# Cho phép traffic từ pods trong cluster-b đến pods trong cluster-a
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: allow-from-cluster-b
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend

  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
            io.cilium.k8s.policy.cluster: cluster-b    # cross-cluster selector
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
```

---

## Cluster Mesh với External Workloads

Cho phép non-Kubernetes workloads (VMs, bare metal) tham gia vào Cluster Mesh.

```bash
# Register external workload
cilium external-workload install \
  --config ciliumexternalworkload-vm1.yaml

# CiliumExternalWorkload CRD
cat <<EOF | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumExternalWorkload
metadata:
  name: vm-worker-1
spec:
  ipv4AllocCIDR: "10.192.1.0/30"   # IP range cho VM
  ipv6AllocCIDR: ""
EOF

# Trên VM: install Cilium agent (external mode)
curl -sfL https://github.com/cilium/cilium/releases/download/v1.15.5/cilium-linux-amd64.tar.gz | tar xz
# Configure với cluster API server endpoint
```

---

## Monitoring Cluster Mesh

```bash
# Status
cilium clustermesh status

# Connected endpoints từ remote clusters
kubectl exec -n kube-system <cilium-pod> -- \
  cilium endpoint list | grep "remote"

# Hubble — cross-cluster flows
hubble observe --cluster cluster-b
hubble observe --from-cluster cluster-a --to-cluster cluster-b

# Metrics
# cilium_clustermesh_global_services — số global services
# cilium_clustermesh_remote_cluster_last_failure_ts — last failure timestamp
# cilium_clustermesh_remote_cluster_readiness_status — 1=ready, 0=not ready
```

### Alert Rules

```yaml
- alert: ClusterMeshRemoteClusterDown
  expr: cilium_clustermesh_remote_cluster_readiness_status == 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Cluster Mesh: remote cluster {{ $labels.cluster_id }} not ready"
```

---

## Gotchas

- **Non-overlapping pod CIDRs là bắt buộc**: Cluster A dùng 10.0.0.0/8, Cluster B phải dùng range khác (ví dụ 10.1.0.0/8). Overlap → routing ambiguity → random drops.
- **Global service và headless services**: Headless Services (ClusterIP: None) cũng có thể làm global, nhưng client nhận DNS response chứa pod IPs từ tất cả clusters. Client phải handle multiple IPs — không phải tất cả clients làm được.
- **Cluster Mesh API Server và certificate rotation**: Cluster Mesh dùng mutual TLS giữa clusters. Certificates có expiry. Cilium tự rotate, nhưng nếu rotation fail → clusters disconnect. Monitor cert expiry và check logs của clustermesh-apiserver.
- **Affinity và load balancing imbalance**: `local` affinity tốt cho latency nhưng có thể tạo imbalance — Cluster A overloaded, Cluster B underutilized. Monitor request rates per cluster.
- **Network connectivity requirements**: Mỗi node trong Cluster A phải có TCP connectivity tới Cluster Mesh API Server của Cluster B. Nếu có firewall giữa clusters, cần open port 2379 (etcd-like API của Cluster Mesh).
- **Service name collision**: Global service hoạt động dựa trên `namespace/name` matching. Nếu có service cùng tên/namespace nhưng không muốn global → không add annotation. Chỉ services có annotation mới tham gia mesh.
