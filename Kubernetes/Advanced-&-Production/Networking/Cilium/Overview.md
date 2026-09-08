# Cilium — 1.15.x
Tags: #cilium #cni #ebpf #networking #kubernetes #security
Last updated: 2026-04-30

---

## 1. What — Nó là cái gì?

Cilium là CNI (Container Network Interface) plugin cho Kubernetes, được xây dựng trên nền **eBPF** (extended Berkeley Packet Filter) — cho phép chạy code trực tiếp trong Linux kernel mà không cần modify kernel source. Cilium không chỉ làm network connectivity mà còn cung cấp security policy enforcement ở L3/L4/L7, load balancing thay thế kube-proxy, và observability qua **Hubble** — tất cả đều hoạt động ở kernel level, không cần sidecar proxy.

---

## 2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?

CNI truyền thống (Calico, Flannel) dùng **iptables** để implement network policy và service routing. Với cluster lớn, iptables có vấn đề nghiêm trọng:

- **iptables O(n) performance**: 10,000 services = 10,000+ iptables rules, mỗi packet phải traverse tất cả. Latency tăng tuyến tính theo số services.
- **Không có L7 visibility**: iptables chỉ biết IP/port, không hiểu HTTP path, gRPC method, hay Kafka topic. Security policy không thể nói "chỉ cho phép GET /api/v1/users".
- **IP-based identity không ổn**: Pod IP thay đổi liên tục khi reschedule. Policy dựa trên IP phải update liên tục.

Cilium giải quyết bằng eBPF:
- **O(1) lookups** với BPF hash maps thay vì iptables chain traversal
- **L7-aware policy**: hiểu HTTP, gRPC, Kafka, DNS ở kernel level
- **Identity-based security**: dùng cryptographic identity gắn với pod label, không phụ thuộc IP
- **Kube-proxy replacement**: eBPF-based service load balancing nhanh hơn iptables DNAT

---

## 3. When — Dùng khi nào / KHÔNG dùng khi nào?

**Dùng khi:**
- Cluster > 100 nodes hoặc > 1000 services — iptables bottleneck rõ ràng
- Cần L7 network policy (HTTP path, gRPC method, Kafka topic filtering)
- Cần deep network observability (ai nói chuyện với ai, latency per service, drop reason) — Hubble
- On-premise và cần LoadBalancer IPs mà không có cloud LB — Cilium BGP + LB-IPAM thay MetalLB
- Muốn node-to-node encryption không cần Istio — WireGuard transparent encryption
- Muốn service mesh không có sidecar overhead

**KHÔNG dùng khi:**
- Kernel version < 4.9 (eBPF features hạn chế), recommend >= 5.10 LTS
- Team chưa quen với eBPF troubleshooting — learning curve cao hơn Calico
- Cần tính năng Windows nodes — Cilium chỉ hỗ trợ Linux
- Môi trường có strict kernel lockdown policies ngăn BPF programs load

---

## 4. Where — Architecture — Nó nằm ở đâu trong hệ thống?

```
┌─────────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                          │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Control Plane                          │   │
│  │   kube-apiserver ◄──────────────── Cilium Operator       │   │
│  │       │                            (1 instance, manages  │   │
│  │       │                             CRDs, IPAM, BGP)     │   │
│  └───────┼──────────────────────────────────────────────────┘   │
│          │                                                       │
│  ┌───────▼──────────────────────────────────────────────────┐   │
│  │                    Worker Nodes (DaemonSet)               │   │
│  │                                                           │   │
│  │   ┌─────────────────────────────────────────────────┐    │   │
│  │   │              Cilium Agent                        │    │   │
│  │   │  - Load/manage eBPF programs                    │    │   │
│  │   │  - Enforce network policy                       │    │   │
│  │   │  - Handle IPAM (assign pod IPs)                 │    │   │
│  │   │  - Expose Hubble gRPC server                    │    │   │
│  │   └────────────────┬────────────────────────────────┘    │   │
│  │                    │ load eBPF programs                   │   │
│  │   ┌────────────────▼────────────────────────────────┐    │   │
│  │   │              Linux Kernel                        │    │   │
│  │   │  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │    │   │
│  │   │  │ BPF Maps │  │TC hooks  │  │ XDP hooks    │  │    │   │
│  │   │  │(identity,│  │(policy,  │  │(fast drop,   │  │    │   │
│  │   │  │ routing) │  │ LB, NAT) │  │ LB offload)  │  │    │   │
│  │   │  └──────────┘  └──────────┘  └──────────────┘  │    │   │
│  │   └─────────────────────────────────────────────────┘    │   │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              Hubble (Observability)                        │  │
│  │   Hubble Agent (per node) → Hubble Relay → Hubble UI      │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Components:**
- **Cilium Agent**: DaemonSet trên mỗi node, quản lý eBPF programs và policy enforcement
- **Cilium Operator**: Deployment (1-2 replicas), quản lý cluster-wide resources (IPAM pools, BGP, CRD)
- **Hubble Agent**: built-in trong Cilium Agent, export flow events
- **Hubble Relay**: aggregates flows từ tất cả nodes
- **Hubble UI**: web UI hiển thị service map và flows

---

## 5. How — Cơ chế hoạt động

**eBPF programs tại network hooks:**
Cilium attach eBPF programs vào TC (Traffic Control) ingress/egress hooks của mỗi network interface. Mọi packet đi vào hoặc ra khỏi pod đều được xử lý bởi eBPF program — policy check, NAT, load balancing — trực tiếp trong kernel, không cần userspace.

**Identity-based security (không phải IP-based):**
Mỗi pod được gán một **numeric identity** (32-bit) dựa trên tổ hợp labels của nó. Identity được encode vào packet thông qua VXLAN/Geneve header hoặc IPsec/WireGuard metadata. Policy engine so sánh identity, không so sánh IP — pod IP thay đổi nhưng identity giữ nguyên nếu labels không đổi.

**Kube-proxy replacement với eBPF:**
Thay vì iptables DNAT rules cho mỗi Service endpoint, Cilium dùng BPF hash map lưu service → endpoints mapping. Lookup O(1) bất kể số lượng services. Socket-level load balancing: redirect ngay tại `connect()` syscall, trước khi packet đi vào network stack.

**L7 policy với Envoy:**
Khi CiliumNetworkPolicy có L7 rules (HTTP/gRPC), Cilium tự động spawn embedded Envoy proxy process để inspect và enforce L7 traffic. Envoy chạy trong Cilium Agent pod, không phải sidecar trong app pod — transparent với applications.

**Hubble flow visibility:**
eBPF programs emit flow events (connection open/close, policy verdict, drop reason) vào perf ring buffer. Hubble Agent đọc ring buffer và expose qua gRPC. Zero overhead khi không có consumer — events drop nếu ring buffer full.

---

## 6. Key Config — Cấu hình cần nhớ

```yaml
# Helm values quan trọng nhất
kubeProxyReplacement: true          # thay thế kube-proxy hoàn toàn (default: false)

# Tunnel mode (mặc định, dễ setup)
tunnel: vxlan                        # vxlan | geneve | disabled (native routing)

# Native routing (hiệu năng cao hơn, cần underlay support)
# tunnel: disabled
# autoDirectNodeRoutes: true
# ipv4NativeRoutingCIDR: "10.0.0.0/8"

# IPAM mode
ipam:
  mode: kubernetes                  # kubernetes | cluster-pool | azure | eni

# Hubble
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
  metrics:
    enabled:
      - drop
      - tcp
      - flow
      - port-distribution
      - icmp
      - http

# Encryption (chọn 1)
encryption:
  enabled: true
  type: wireguard                   # wireguard | ipsec

# BGP (on-premise)
bgpControlPlane:
  enabled: true
```

**Default values cần biết:**
- `kubeProxyReplacement: false` — Cilium chạy cùng kube-proxy by default. Phải explicitly set `true` để replace.
- `tunnel: vxlan` — overhead ~50 bytes/packet so với native routing. Cho production high-throughput nên dùng native routing nếu underlay hỗ trợ.
- `policyEnforcementMode: default` — policy chỉ enforce trên pods có CiliumNetworkPolicy. Pods không có policy = allow all. Set `always` để default-deny cho toàn cluster.

---

## 7. Security Considerations

**Attack surface:**
- Cilium Agent chạy với `CAP_SYS_ADMIN` và BPF capabilities — nếu agent bị compromise, có thể modify eBPF programs của toàn node.
- CiliumNetworkPolicy và NetworkPolicy coexist — cả hai đều apply (AND logic). Misconfiguration một trong hai có thể unexpected block hoặc allow.
- Hubble flow data chứa full L7 request info (URLs, headers, methods) — sensitive data exposure nếu không restrict access tới Hubble API.

**Hardening checklist:**
```bash
# 1. Restrict Hubble access
# Hubble UI nên đứng sau auth proxy (OAuth2, Dex)
# Không expose Hubble Relay ra ngoài cluster

# 2. Enable policy enforcement mode = always (default-deny)
helm upgrade cilium cilium/cilium --set policyEnforcementMode=always

# 3. Enable WireGuard encryption cho node-to-node traffic
helm upgrade cilium cilium/cilium \
  --set encryption.enabled=true \
  --set encryption.type=wireguard

# 4. Restrict Cilium Agent RBAC
# Agent cần quyền đọc Nodes, Pods, Services, Endpoints
# Không nên có write access vào arbitrary secrets

# 5. Audit CiliumNetworkPolicy với Hubble
cilium policy get    # xem effective policy
hubble observe --verdict DROPPED    # xem traffic bị drop
```

---

## 8. Ops Runbook — Production Notes

**Health check:**
```bash
# Cilium status
cilium status --wait              # wait until all components healthy
cilium status -o json | jq .      # detailed JSON output

# Per-node status
kubectl get pods -n kube-system -l k8s-app=cilium
kubectl exec -n kube-system <cilium-pod> -- cilium status

# Connectivity test (end-to-end)
cilium connectivity test           # deploy test pods, verify L3/L4/L7 connectivity

# BPF datapath
kubectl exec -n kube-system <cilium-pod> -- cilium bpf endpoint list
kubectl exec -n kube-system <cilium-pod> -- cilium bpf lb list    # service LB entries
```

**Metrics cần alert:**

| Metric | Threshold | Ý nghĩa |
|--------|-----------|---------|
| `cilium_drop_count_total` | tăng đột biến | Policy drop hoặc networking issue |
| `cilium_endpoint_regenerations_total{outcome="fail"}` | > 0 | Policy compilation failure |
| `cilium_agent_api_process_time_seconds` | p99 > 1s | Agent overloaded |
| `hubble_drop_total` | tăng liên tục | Traffic bị drop — xem drop reason |
| `cilium_bpf_map_ops_total{outcome="err"}` | > 0 | BPF map operation failure |

**Log quan trọng:**
```bash
# Cilium agent logs
kubectl logs -n kube-system <cilium-pod> | grep -E "level=(error|warning)"

# Policy verdict
hubble observe --verdict DROPPED --last 100
hubble observe --namespace production --verdict DROPPED

# Endpoint regeneration failure
kubectl exec -n kube-system <cilium-pod> -- cilium endpoint list
kubectl exec -n kube-system <cilium-pod> -- cilium endpoint get <endpoint-id>
```

**Troubleshooting commands:**
```bash
# Sysdump (full diagnostic dump)
cilium sysdump --output-filename cilium-sysdump-$(date +%Y%m%d)

# Check policy on specific pod
kubectl exec -n kube-system <cilium-pod> -- \
  cilium policy get

# Trace packet flow
kubectl exec -n kube-system <cilium-pod> -- \
  cilium monitor --type trace --from-source <pod-ip>

# Restart Cilium agent (graceful — giữ existing connections)
kubectl rollout restart daemonset/cilium -n kube-system
```

---

## 9. Gotchas & Lessons Learned

- **Migration từ kube-proxy**: Khi enable `kubeProxyReplacement=true`, phải xóa kube-proxy DaemonSet TRƯỚC khi restart Cilium. Nếu cả hai cùng chạy, iptables rules của kube-proxy conflict với BPF programs.
- **Tunnel vs native routing và MTU**: VXLAN overhead 50 bytes làm giảm effective MTU từ 1500 xuống 1450. Nếu applications gửi full 1500-byte packets → fragmentation → performance drop. Set MTU explicitly hoặc dùng jumbo frames (9000 MTU) trên underlay.
- **`policyEnforcementMode: always` và DNS**: Khi enforce always, pods không có policy sẽ không resolve DNS (port 53 bị block). Phải có allow-dns policy TRƯỚC khi enable enforcement mode.
- **CiliumNetworkPolicy và NetworkPolicy coexist**: Cả hai đều apply, kết quả là AND. Một policy allow nhưng policy kia deny → vẫn deny. Debug bằng `hubble observe --verdict DROPPED` để xem policy nào drop.
- **Hubble ring buffer đầy**: Với cluster traffic cao, Hubble ring buffer có thể full → flow events bị drop → missing observability. Tăng `hubble.eventQueueSize` và `hubble.eventBufferCapacity` trong Helm values.
- **BPF map size limits**: Mỗi BPF map có max entries. Với cluster rất lớn (10k+ endpoints), cần tăng `config.bpfMapDynamicSizeRatio` để auto-scale map sizes.

---

## 10. Resources

- [Cilium Documentation](https://docs.cilium.io/) — official, section "Concepts" cần đọc trước
- [Cilium GitHub](https://github.com/cilium/cilium) — release notes, issue tracker
- [eBPF.io](https://ebpf.io/) — eBPF fundamentals trước khi đào sâu Cilium internals
- [Cilium Helm Reference](https://docs.cilium.io/en/stable/helm-reference/) — tất cả Helm values
- [Isovalent Labs](https://isovalent.com/labs/) — hands-on labs miễn phí (Cilium company)
- [Network Policy Editor](https://editor.networkpolicy.io/) — visual editor cho CiliumNetworkPolicy
