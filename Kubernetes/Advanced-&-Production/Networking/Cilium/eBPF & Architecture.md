---
title: Cilium — eBPF & Architecture
tags:
  - cilium
  - ebpf
  - architecture
  - datapath
date: 2026-04-30
---

# Cilium — eBPF & Architecture

## eBPF Primer

eBPF (extended Berkeley Packet Filter) cho phép chạy sandboxed programs trong Linux kernel mà không cần modify kernel source hoặc load kernel modules. Programs được verify bởi eBPF verifier trước khi chạy — đảm bảo không crash kernel, không infinite loop, không access memory ngoài bounds.

```
User Space
  │
  │  cilium-agent viết eBPF bytecode
  │  → compile bằng clang/LLVM → ELF object
  │  → load vào kernel qua bpf() syscall
  │
Kernel
  │
  ├─ eBPF Verifier → kiểm tra safety (CFG, bounds, types)
  │
  ├─ JIT Compiler → native machine code (x86_64/ARM64)
  │
  └─ Hook Points (attach points):
       XDP     — earliest hook, trước khi packet vào kernel stack
       TC      — Traffic Control ingress/egress (Cilium chủ yếu dùng đây)
       kprobe  — kernel function entry/exit
       tracepoint — kernel trace events
       socket  — socket-level (connect, sendmsg, recvmsg)
```

**BPF Maps** là key-value store trong kernel, dùng để:
- Share data giữa BPF programs và userspace (cilium-agent)
- Share data giữa BPF programs với nhau
- Lưu state: policy, routing table, conntrack, service endpoints

```
Types của BPF Maps Cilium dùng:
  BPF_MAP_TYPE_HASH        → policy lookup, endpoint identity
  BPF_MAP_TYPE_ARRAY       → config, per-CPU counters
  BPF_MAP_TYPE_LRU_HASH    → connection tracking
  BPF_MAP_TYPE_PERCPU_HASH → per-CPU stats (lockless)
  BPF_MAP_TYPE_PROG_ARRAY  → tail calls (jump giữa BPF programs)
```

---

## Cilium Datapath

### Packet Flow — Pod đến Pod (cùng node)

```
Pod A (eth0) ──► veth pair ──► lxcXXXX (host side)
                                    │
                              TC egress hook
                              [eBPF program]
                                    │
                              ┌─────▼──────────────┐
                              │  Policy check       │
                              │  - src identity?    │
                              │  - dst identity?    │
                              │  - L4 port allowed? │
                              └─────┬──────────────┘
                                    │ ALLOW
                              ┌─────▼──────────────┐
                              │  Routing/forwarding │
                              │  (direct via BPF    │
                              │   redirect)         │
                              └─────┬──────────────┘
                                    │
                              lxcYYYY ──► veth pair ──► Pod B (eth0)
```

### Packet Flow — Pod đến Pod (khác node, VXLAN mode)

```
Pod A (node1)
  │
  TC egress hook → policy check → VXLAN encap + identity metadata
  │
  eth0 (node1) ──── network ────► eth0 (node2)
                                       │
                                  TC ingress hook
                                  → VXLAN decap
                                  → identity extraction
                                  → policy check
                                  → deliver to Pod B
```

### Kube-proxy Replacement — Service Load Balancing

```
App container
  │ connect("10.96.0.1:443")   ← ClusterIP (virtual IP)
  │
  SocketLB (BPF cgroup hook)
  │ Intercept tại connect() syscall
  │ Lookup: ClusterIP:port → [endpoint1, endpoint2, endpoint3]
  │ Select endpoint (consistent hashing / random)
  │
  │ connect("10.244.1.5:8443")  ← actual pod IP (redirected)
  │
  TCP connection established directly to pod
  (no DNAT in network path — faster than iptables)
```

---

## Identity Model

Cilium không dùng IP để identify workloads — dùng **numeric identity** được derive từ pod labels.

```
Pod labels:
  app=frontend
  env=production
  version=v2

→ Hash labels → Identity: 12345 (32-bit number)

Identity được encode trong:
  - VXLAN: Geneve option hoặc VNI field
  - Native routing: IP packet options hoặc separate BPF metadata
  - WireGuard/IPsec: trong encrypted packet metadata
```

### Identity Scopes

```
┌─────────────────────────────────────────────────────┐
│ Reserved Identities (0-255)                          │
│   1 = host (node itself)                             │
│   2 = world (traffic từ/đến internet)                │
│   3 = unmanaged (pods không có Cilium)               │
│   4 = health (Cilium health check)                   │
│   5 = init (pods đang khởi tạo)                      │
│   7 = remote-node (node khác trong cluster)          │
│   8 = kube-apiserver                                 │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│ Local Identities (256 - 16383)                       │
│   Tạo bởi Cilium operator cho pods trong cluster     │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│ World Identities (> 16383) — CIDR-based              │
│   Tạo cho external CIDRs trong policy                │
└─────────────────────────────────────────────────────┘
```

```bash
# Xem identities
kubectl exec -n kube-system <cilium-pod> -- cilium identity list

# Xem identity của specific endpoint
kubectl exec -n kube-system <cilium-pod> -- cilium endpoint list
kubectl exec -n kube-system <cilium-pod> -- cilium endpoint get <id>
```

---

## Cilium Components Chi Tiết

### Cilium Agent (DaemonSet)

Chạy trên mỗi node, là "brain" của Cilium trên node đó:

```
cilium-agent
  │
  ├─ Watch Kubernetes API (Pods, Services, NetworkPolicy, CiliumNetworkPolicy)
  ├─ Compute policy per endpoint
  ├─ Compile + load eBPF programs vào kernel
  ├─ Manage BPF maps (update khi policy/service thay đổi)
  ├─ IPAM: assign IP cho pods trên node này
  ├─ Health checking: ping tới agents trên nodes khác
  └─ Hubble: export flow events
```

### Cilium Operator (Deployment)

Chạy 1-2 replicas trong cluster:

```
cilium-operator
  │
  ├─ Manage CRDs (CiliumNetworkPolicy, CiliumEndpoint, CiliumNode...)
  ├─ IPAM pool management (allocate IP ranges cho nodes)
  ├─ BGP control plane (CiliumBGPPeeringPolicy)
  ├─ GC: cleanup stale CiliumEndpoints khi pods bị xóa
  └─ NodePort health checking
```

### Envoy (Embedded, không phải sidecar)

Cilium nhúng Envoy proxy để handle L7 traffic inspection:

```
Khi CiliumNetworkPolicy có L7 rule:
  → Cilium tự động redirect traffic qua embedded Envoy
  → Envoy inspect HTTP/gRPC và enforce L7 policy
  → Kết quả (allow/deny) được pass lại cho eBPF

Envoy chạy trong cilium-agent pod, không phải trong app pod
→ Không có sidecar overhead
→ Nhưng có thêm một hop qua loopback cho L7 traffic
```

---

## Networking Modes

### Tunnel Mode (default — dễ setup)

```yaml
# Helm values
tunnel: vxlan    # hoặc geneve

# Ưu điểm:
# - Không cần underlay routing configuration
# - Works với bất kỳ L2/L3 network
# - Underlay không cần biết về pod CIDRs

# Nhược điểm:
# - Overhead ~50 bytes/packet (VXLAN header)
# - MTU giảm: 1500 - 50 = 1450 bytes effective
```

### Native Routing Mode (production recommended)

```yaml
# Helm values
tunnel: disabled
autoDirectNodeRoutes: true          # add routes tới pod CIDRs của nodes khác
ipv4NativeRoutingCIDR: "10.0.0.0/8"  # CIDR mà không cần masquerade

# Ưu điểm:
# - Không có tunnel overhead
# - Full MTU (1500 bytes)
# - Hiệu năng cao hơn ~10-15%

# Điều kiện:
# - Underlay phải route pod CIDRs giữa các nodes
# - Với BGP: dùng Cilium BGP để advertise pod routes
# - Với cloud: thường có built-in routing (AWS VPC CNI mode)
```

### IPAM Modes

```yaml
# kubernetes mode (default) — dùng node.spec.podCIDR
ipam:
  mode: kubernetes

# cluster-pool — Cilium operator tự quản lý IP pool
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList: ["10.0.0.0/8"]
    clusterPoolIPv4MaskSize: 24    # /24 per node = 256 pods/node
```

---

## Installation

### Helm Install

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

# Basic install (thay thế kube-proxy)
helm install cilium cilium/cilium \
  --version 1.15.5 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}') \
  --set k8sServicePort=6443

# Verify
cilium status --wait
cilium connectivity test
```

### Migration từ existing CNI

```bash
# 1. Install Cilium với --set cni.chainingMode=generic-veth
#    (chạy cùng CNI cũ, không replace datapath)
helm install cilium cilium/cilium \
  --set cni.chainingMode=generic-veth \
  --set enableIPv4Masquerade=false

# 2. Verify Cilium hoạt động
cilium status

# 3. Drain nodes từng cái, replace CNI config
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data
# Trên node1: remove old CNI plugin, update /etc/cni/net.d/
kubectl uncordon node1

# 4. Sau khi tất cả nodes migrate → remove cni.chainingMode
helm upgrade cilium cilium/cilium --set cni.chainingMode=none
```

---

## Debugging & Observability

```bash
# Monitor all packet events (verbose)
kubectl exec -n kube-system <cilium-pod> -- cilium monitor

# Filter by type
cilium monitor --type drop           # dropped packets
cilium monitor --type trace          # packet trace
cilium monitor --type l7             # L7 (HTTP/gRPC) events

# BPF map inspection
cilium bpf endpoint list             # all endpoints + identities
cilium bpf policy get <endpoint-id>  # policy for endpoint
cilium bpf lb list                   # service load balancer entries
cilium bpf nat list                  # NAT table
cilium bpf ct list global            # connection tracking table

# Policy trace (simulate packet)
cilium policy trace \
  --src-k8s-pod default/frontend-xxx \
  --dst-k8s-pod default/backend-xxx \
  --dport 8080/TCP
# → Allowed/Denied + policy chain

# Full diagnostic dump
cilium sysdump
```

---

## Gotchas

- **eBPF verifier reject**: Nếu kernel version quá cũ hoặc có kernel config bị disable, eBPF verifier reject Cilium programs → Cilium agent crash loop. Check `cilium status` và kernel version. Minimum kernel: 4.9, recommended: 5.10 LTS.
- **BPF map full**: Với cluster lớn, BPF maps có max entries. Default sizing thường đủ cho vài trăm pods/node. Nếu exceed: `cilium bpf map events` show errors. Tăng `bpf-map-dynamic-size-ratio` trong Helm values.
- **Tail call limit**: eBPF có stack size limit và không support recursion. Cilium dùng tail calls (BPF_MAP_TYPE_PROG_ARRAY) để chain programs. Limit là 33 tail calls — với rất nhiều policy rules → có thể hit limit. Cilium 1.13+ có policy compression.
- **conntrack table và high-throughput**: Default conntrack table size limit có thể gây "conntrack table full" error trên high-traffic nodes. Monitor `cilium_bpf_map_ops_total` và tăng `bpf-ct-global-tcp-max` nếu cần.
