---
title: Cilium BGP & LoadBalancer IP Management
tags:
  - cilium
  - bgp
  - loadbalancer
  - on-premise
  - ipam
date: 2026-04-30
---

# Cilium BGP & LoadBalancer IP Management

Đây là tính năng cực kỳ quan trọng cho **on-premise** — thay thế MetalLB, cho phép K8s Services type LoadBalancer nhận real IP và được advertise ra network qua BGP. Không cần cloud provider.

```
┌────────────────────────────────────────────────────────────────┐
│                      On-Premise Network                         │
│                                                                 │
│   TOR Switch / Router                                          │
│   192.168.1.1 (ASN 65000)                                      │
│      │   BGP session                                           │
│      │   "Tôi có thể reach 10.10.0.0/24 (LoadBalancer pool)"  │
│      │                                                          │
│   ┌──▼─────────────────────────────────────────────────────┐  │
│   │              Kubernetes Cluster                          │  │
│   │                                                          │  │
│   │   Node 1 (ASN 65001) ──BGP peer──► Router               │  │
│   │   Node 2 (ASN 65001) ──BGP peer──► Router               │  │
│   │   Node 3 (ASN 65001) ──BGP peer──► Router               │  │
│   │                                                          │  │
│   │   Service: type=LoadBalancer                             │  │
│   │   → Cilium assign IP: 10.10.0.5                         │  │
│   │   → BGP advertise 10.10.0.5/32 ra Router                │  │
│   └──────────────────────────────────────────────────────── ┘  │
└────────────────────────────────────────────────────────────────┘
```

---

## Cilium BGP Control Plane

### Enable BGP

```yaml
# Helm values
bgpControlPlane:
  enabled: true
```

### CiliumBGPPeeringPolicy

```yaml
# Định nghĩa BGP peering cho nodes
apiVersion: "cilium.io/v2alpha1"
kind: CiliumBGPPeeringPolicy
metadata:
  name: rack-1-bgp
spec:
  # Áp dụng cho nodes nào (node selector)
  nodeSelector:
    matchLabels:
      rack: rack-1              # chỉ nodes trong rack 1

  virtualRouters:
    - localASN: 65001           # ASN của K8s nodes
      exportPodCIDR: true       # advertise pod CIDRs (native routing mode)
      
      neighbors:
        - peerAddress: "192.168.1.1/32"   # router IP
          peerASN: 65000                   # router ASN
          
          # BGP timers
          connectRetryTimeSeconds: 120
          holdTimeSeconds: 90
          keepAliveTimeSeconds: 30
          
          # eBGP multi-hop (nếu peer không cùng L2)
          eBGPMultihopTTL: 10
          
          # Chỉ advertise LoadBalancer IPs (không advertise pod CIDRs)
          advertisements:
            - advertisementType: "Service"
              service:
                addresses:
                  - LoadBalancerIP

---
# Multi-rack với different peers
apiVersion: "cilium.io/v2alpha1"
kind: CiliumBGPPeeringPolicy
metadata:
  name: rack-2-bgp
spec:
  nodeSelector:
    matchLabels:
      rack: rack-2
  virtualRouters:
    - localASN: 65002
      neighbors:
        - peerAddress: "192.168.2.1/32"
          peerASN: 65000
          advertisements:
            - advertisementType: "Service"
              service:
                addresses:
                  - LoadBalancerIP
```

### BGP Password Authentication

```yaml
neighbors:
  - peerAddress: "192.168.1.1/32"
    peerASN: 65000
    authSecretRef: bgp-auth-secret   # K8s Secret name

---
apiVersion: v1
kind: Secret
metadata:
  name: bgp-auth-secret
  namespace: kube-system
type: Opaque
stringData:
  password: "bgp-shared-secret"
```

---

## LB-IPAM — LoadBalancer IP Address Management

LB-IPAM tự động assign IP cho Services type=LoadBalancer từ defined pools. Không cần MetalLB.

### CiliumLoadBalancerIPPool

```yaml
# Pool cho production services
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: production-pool
spec:
  cidrs:
    - cidr: "10.10.0.0/24"     # 254 IPs available

  # Optional: chỉ assign cho services có specific label
  serviceSelector:
    matchLabels:
      pool: production

---
# Pool riêng cho internal services
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: internal-pool
spec:
  cidrs:
    - cidr: "10.20.0.0/28"     # 14 IPs (small internal pool)
  serviceSelector:
    matchLabels:
      expose: internal

---
# Pool với specific IPs (không phải range)
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: static-pool
spec:
  cidrs:
    - cidr: "10.30.0.1/32"     # single IP
    - cidr: "10.30.0.2/32"
    - cidr: "10.30.0.3/32"
```

### Service với LB-IPAM

```yaml
# Tự động get IP từ pool
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: production
  labels:
    pool: production            # match serviceSelector
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080

---
# Request specific IP
apiVersion: v1
kind: Service
metadata:
  name: my-service-static-ip
  namespace: production
  annotations:
    # Request specific IP từ pool
    cilium.io/lb-ipam-ips: "10.10.0.100"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 443

---
# Share IP giữa nhiều services (different ports)
apiVersion: v1
kind: Service
metadata:
  name: service-http
  annotations:
    cilium.io/lb-ipam-sharing-key: "shared-frontend"
    cilium.io/lb-ipam-sharing-cross-namespace: "true"
spec:
  type: LoadBalancer
  ports:
    - port: 80
```

### Verify LB-IPAM

```bash
# Check pool status
kubectl get ciliumloadbalancerippools
# NAME              DISABLED  CONFLICTING  IPS-AVAILABLE  IPS-USED
# production-pool   false     false        252            8

# Check service IP assignment
kubectl get svc -n production
# NAME         TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)
# my-service   LoadBalancer   10.96.10.50    10.10.0.5      80:30080/TCP

# BGP advertised routes
kubectl exec -n kube-system <cilium-pod> -- \
  cilium bgp routes advertised ipv4 unicast
```

---

## BGP Troubleshooting

```bash
# Check BGP peer state
kubectl exec -n kube-system <cilium-pod> -- \
  cilium bgp peers
# PeerAddress    PeerASN  State       UpTime
# 192.168.1.1   65000    ESTABLISHED  1d 2h 30m

# BGP peer state per node
kubectl get ciliumnodes -o yaml | grep -A 20 bgp

# Advertised routes
kubectl exec -n kube-system <cilium-pod> -- \
  cilium bgp routes advertised ipv4 unicast

# Received routes from peer
kubectl exec -n kube-system <cilium-pod> -- \
  cilium bgp routes available ipv4 unicast

# Check operator logs cho LB-IPAM
kubectl logs -n kube-system deploy/cilium-operator | grep -i "ipam\|bgp"

# Cilium BGP state
kubectl get ciliumbgppeeringpolicies
kubectl describe ciliumbgppeeringpolicy rack-1-bgp
```

---

## L2 Announcements (thay thế ARP/MetalLB Layer 2 mode)

Cho environments không có BGP — dùng L2 announcements (gratuitous ARP/NDP).

```yaml
# Enable L2 announcements
helm upgrade cilium cilium/cilium \
  --set l2announcements.enabled=true \
  --set l2podAnnouncements.enabled=true \
  --reuse-values

---
# CiliumL2AnnouncementPolicy
apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: l2-policy
spec:
  # Announce LoadBalancer IPs
  loadBalancerIPs: true
  # Announce ExternalIPs
  externalIPs: true
  
  # Network interfaces để send ARP (regex)
  interfaces:
    - ^eth[0-9]+

  # Node selector (announce từ nodes nào)
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/worker: ""
```

---

## So sánh với MetalLB

| Feature | MetalLB | Cilium BGP + LB-IPAM |
|---------|---------|----------------------|
| BGP mode | Có | Có (native, không cần frrouting) |
| L2 mode | Có | Có (L2 Announcements) |
| IP pool management | IPAddressPool CRD | CiliumLoadBalancerIPPool CRD |
| Integration với CNI | External, cần rules thêm | Native, không có friction |
| Observability | Hạn chế | Tích hợp với Hubble |
| Dependencies | Frrouting (BGP mode) | Không thêm dependency |
| Maturity | Stable | Alpha/Beta (1.15+) |

---

## Gotchas

- **BGP và native routing**: Cilium BGP thường kết hợp với `tunnel: disabled` (native routing). Với VXLAN tunnel, BGP chỉ advertise LoadBalancer IPs, không advertise pod CIDRs. Nếu cần pod-to-pod direct routing qua BGP, cần native routing mode.
- **ECMP và consistent hashing**: Khi nhiều nodes advertise cùng LoadBalancer IP (ECMP), router phân tải qua nodes. Cilium dùng Maglev consistent hashing để đảm bảo cùng flow luôn đến cùng node — tránh connection reset. Enable: `loadBalancer.algorithm: maglev`.
- **BGP session flap**: Nếu Cilium agent restart (update, OOM), BGP session drop → routes withdraw → traffic gián đoạn ~BGP convergence time (vài giây). Graceful restart BGP: `gracefulRestart.enabled: true` trong peer config.
- **LB-IPAM và external-dns**: Khi dùng cùng external-dns để tự động tạo DNS records, external-dns cần quyền read Services. Không có conflict với Cilium LB-IPAM.
- **`cilium.io/lb-ipam-ips` annotation và pool conflict**: Nếu request IP nằm ngoài tất cả pools, Service sẽ ở `Pending` state. Check events: `kubectl describe svc <name>`.
