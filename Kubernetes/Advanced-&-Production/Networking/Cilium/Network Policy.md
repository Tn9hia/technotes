---
title: Cilium Network Policy
tags:
  - cilium
  - network-policy
  - security
  - l7
date: 2026-04-30
---

# Cilium Network Policy

## Policy Types

Cilium hỗ trợ 3 loại policy, tất cả cùng apply (AND logic):

| Type | Scope | L7 support | Khi nào dùng |
|------|-------|------------|--------------|
| `NetworkPolicy` | Namespaced | Không | Compatibility với tools hiện có |
| `CiliumNetworkPolicy` | Namespaced | Có | L7 rules, entity selectors |
| `CiliumClusterwideNetworkPolicy` | Cluster-wide | Có | Base policies cho toàn cluster |

---

## Policy Enforcement Modes

```bash
# default: chỉ enforce policies cho pods có CiliumNetworkPolicy
# → pods không có policy = allow all ingress/egress

# always: enforce cho tất cả pods — pods không có policy = deny all
helm upgrade cilium cilium/cilium \
  --set policyEnforcementMode=always \
  --reuse-values
```

> **Với `always` mode**: cần có allow-DNS policy trước khi enable, nếu không pods sẽ không resolve DNS.

---

## CiliumNetworkPolicy — Cú pháp

### L3/L4 — Pod Selector (identity-based)

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: backend-allow-from-frontend
  namespace: production
spec:
  endpointSelector:               # áp dụng cho pods nào
    matchLabels:
      app: backend

  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend         # chỉ cho phép từ pods có label app=frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP

  egress:
    - toEndpoints:
        - matchLabels:
            app: postgres
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
```

### L3 — Namespace Selector

```yaml
spec:
  endpointSelector:
    matchLabels:
      app: backend

  ingress:
    - fromEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: monitoring   # từ namespace monitoring
            app: prometheus
```

### L3 — CIDR (external traffic)

```yaml
spec:
  endpointSelector:
    matchLabels:
      app: payment-service

  egress:
    # Allow đến payment gateway (external)
    - toCIDR:
        - "203.0.113.10/32"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP

    # Block một CIDR cụ thể
    - toCIDRSet:
        - cidr: "0.0.0.0/0"
          except:
            - "10.0.0.0/8"      # allow internet nhưng block private ranges
```

### Entity Selectors (đặc trưng của Cilium)

```yaml
# Entity là các reserved identities — không cần biết IP
spec:
  endpointSelector:
    matchLabels:
      app: frontend

  egress:
    # Allow ra internet
    - toEntities:
        - world              # tất cả traffic ngoài cluster

    # Allow đến host (node)
    - toEntities:
        - host

    # Allow đến kube-apiserver
    - toEntities:
        - kube-apiserver

    # Allow DNS (cần thiết khi enforcement=always)
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
            - port: "53"
              protocol: TCP

# Entities: world, host, cluster, remote-node, init, unmanaged, health, kube-apiserver
```

---

## L7 Policy — HTTP

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: api-gateway-l7
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: api-server

  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              # Chỉ cho phép GET /api/v1/users và GET /api/v1/products
              - method: "GET"
                path: "/api/v1/users"
              - method: "GET"
                path: "/api/v1/products"
              # Cho phép tất cả method với path prefix
              - method: ".*"
                path: "/api/v1/health"

    - fromEndpoints:
        - matchLabels:
            app: admin-panel
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              # Admin có thể POST/PUT/DELETE
              - method: ".*"
                path: "/api/v1/.*"    # regex

              # Thêm header requirement
              - method: "POST"
                path: "/api/v1/users"
                headers:
                  - "X-Admin-Token: .*"
```

## L7 Policy — gRPC

```yaml
spec:
  endpointSelector:
    matchLabels:
      app: grpc-server

  ingress:
    - fromEndpoints:
        - matchLabels:
            app: grpc-client
      toPorts:
        - ports:
            - port: "50051"
              protocol: TCP
          rules:
            http:    # gRPC dùng HTTP/2 — Cilium model dưới dạng HTTP rules
              # Cho phép tất cả methods trong service UserService
              - method: "POST"
                path: "/mypackage.UserService/.*"

              # Chỉ cho phép specific method
              - method: "POST"
                path: "/mypackage.UserService/GetUser"
```

## L7 Policy — Kafka

```yaml
spec:
  endpointSelector:
    matchLabels:
      app: kafka-consumer

  egress:
    - toEndpoints:
        - matchLabels:
            app: kafka
      toPorts:
        - ports:
            - port: "9092"
              protocol: TCP
          rules:
            kafka:
              # Chỉ cho phép consume từ topic "orders"
              - role: consume
                topic: "orders"

              # Cho phép produce vào topic "events"
              - role: produce
                topic: "events"

              # Chỉ cho phép Fetch và Produce API calls (apiKey)
              - apiKey: 1    # Fetch
              - apiKey: 0    # Produce
```

## L7 Policy — DNS

```yaml
# DNS-based policy — resolve FQDN tới IPs, tự động update khi IP đổi
spec:
  endpointSelector:
    matchLabels:
      app: external-api-client

  egress:
    - toFQDNs:
        - matchName: "api.external-service.com"    # exact match
        - matchPattern: "*.s3.amazonaws.com"        # wildcard
        - matchPattern: "*.amazonaws.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP

    # DNS lookup phải được allow riêng
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"    # allow tất cả DNS queries
```

---

## CiliumClusterwideNetworkPolicy — Base Policies

```yaml
# Default deny toàn cluster (apply trước tất cả namespace policies)
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: default-deny-all
spec:
  endpointSelector: {}    # tất cả pods
  ingress:
    - {}                  # empty = deny all ingress (sẽ bị override bởi namespace policies)

---
# Allow DNS cho tất cả pods (cluster-wide)
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: allow-dns
spec:
  endpointSelector: {}
  egress:
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
            - port: "53"
              protocol: TCP
          rules:
            dns:
              - matchPattern: "*"

---
# Allow Prometheus scrape từ monitoring namespace
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: allow-prometheus-scrape
spec:
  endpointSelector: {}
  ingress:
    - fromEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: monitoring
            app: prometheus
      toPorts:
        - ports:
            - port: "9090"    # hoặc port metrics của app
              protocol: TCP
```

---

## Policy Selectors — Tổng Hợp

```yaml
# fromEndpoints / toEndpoints — pod selector (identity-based)
fromEndpoints:
  - matchLabels:
      app: frontend
      env: production

# fromRequires — additional constraint (AND với fromEndpoints)
# Tất cả sources phải match fromRequires
fromRequires:
  - matchLabels:
      env: production    # chỉ accept từ pods trong env=production

# fromCIDR / toCIDR — IP range
toCIDR:
  - "10.0.0.0/8"

toCIDRSet:
  - cidr: "0.0.0.0/0"
    except:
      - "169.254.0.0/16"    # link-local

# toEntities / fromEntities — reserved identities
toEntities:
  - world       # external
  - host        # node
  - cluster     # trong cluster
  - kube-apiserver

# toServices — K8s Service selector
toServices:
  - k8sService:
      serviceName: my-service
      namespace: production

# toFQDNs — DNS-based egress
toFQDNs:
  - matchName: "api.github.com"
  - matchPattern: "*.example.com"
```

---

## Policy Debugging

```bash
# Xem tất cả policies đang apply
kubectl get cnp,ccnp -A
cilium policy get

# Trace policy verdict cho specific traffic
cilium policy trace \
  --src-k8s-pod production/frontend-abc \
  --dst-k8s-pod production/backend-xyz \
  --dport 8080/TCP
# Output: Allowed/Denied + rules matched

# Xem dropped packets realtime
hubble observe --verdict DROPPED --namespace production
hubble observe --verdict DROPPED --last 50 -o json | jq '{src:.source.pod_name, dst:.destination.pod_name, reason:.drop_reason_desc}'

# Xem policy trên specific endpoint
kubectl exec -n kube-system <cilium-pod> -- \
  cilium endpoint get <endpoint-id> | jq '.policy'

# Xem L7 events
cilium monitor --type l7 --from-source <pod-ip>
```

---

## Gotchas

- **AND semantics giữa NetworkPolicy và CiliumNetworkPolicy**: Nếu có cả hai, một traffic phải được allow bởi CẢ HAI mới được pass. Đây là nguồn gốc của rất nhiều "why is this blocked?" debugging sessions.
- **L7 policy cần Envoy**: Khi có HTTP/gRPC rule, Cilium tự động spin up embedded Envoy. Điều này thêm một loopback hop (~0.1-0.5ms). Không dùng L7 policy cho latency-critical paths nếu không cần thiết.
- **DNS policy và TTL**: `toFQDNs` resolve DNS và lưu IPs vào BPF map. Khi DNS TTL expire và IP đổi, có thể có window (~TTL duration) traffic bị block. Monitor `cilium_dns_proxy_responses_total` và `cilium_policy_l7_denied_total`.
- **`matchPattern: "*"` trong DNS rule**: Allow DNS lookup cho tất cả FQDNs, nhưng không có nghĩa là allow tất cả egress. Phải có separate `toFQDNs` rule để allow actual connection.
- **Empty `endpointSelector: {}`**: Trong CiliumClusterwideNetworkPolicy, `{}` = match all pods. Trong CiliumNetworkPolicy, `{}` = match all pods trong namespace. Khác nhau — cẩn thận.
- **fromRequires vs fromEndpoints**: `fromRequires` là global constraint — nếu set, TẤT CẢ sources phải match, kể cả sources được allow bởi `fromEndpoints`. Dùng để implement cluster-level constraints (ví dụ: chỉ accept từ `env=production`).
