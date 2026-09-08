---
title: Cilium Service Mesh
tags:
  - cilium
  - service-mesh
  - envoy
  - mtls
  - traffic-management
date: 2026-04-30
---

# Cilium Service Mesh

Cilium Service Mesh là sidecar-free service mesh — dùng eBPF + embedded Envoy trong Cilium Agent thay vì inject sidecar vào mỗi pod. Giảm resource overhead đáng kể so với Istio.

```
Istio (sidecar model):
  Pod: [App container] + [Envoy sidecar]
  → Mỗi pod tốn thêm ~50-100MB RAM, ~0.5 CPU
  → Hàng trăm pods = hàng trăm Envoy sidecars

Cilium Service Mesh (sidecar-free):
  Pod: [App container]
  → Envoy chạy trong cilium-agent DaemonSet (1 per node)
  → Traffic routing qua eBPF + loopback tới node-level Envoy
  → Tiết kiệm resource đáng kể
```

---

## Enable Service Mesh

```yaml
# Helm values
envoy:
  enabled: true          # enable standalone Envoy DaemonSet

# Hoặc dùng embedded Envoy trong cilium-agent (default cho L7 policy)
# Không cần config thêm nếu chỉ cần L7 policy
```

---

## Mutual TLS (mTLS)

### mTLS với CiliumNetworkPolicy (đơn giản)

```yaml
# Enforce mTLS giữa services — dùng SPIFFE/SPIRE identity
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: require-mtls
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend

  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      authentication:
        mode: required      # require mutual TLS authentication
```

### SPIFFE/SPIRE Integration

```yaml
# Cilium dùng SPIFFE (Secure Production Identity Framework For Everyone)
# để cấp X.509 certificates cho pods

# Helm values — enable SPIRE
authentication:
  mutual:
    spire:
      enabled: true
      install:
        enabled: true          # Cilium tự install SPIRE server/agent
        namespace: cilium-spire
        server:
          serviceAccountName: spire-server
        agent:
          serviceAccountName: spire-agent
```

---

## Traffic Management — Ingress (L7)

### CiliumEnvoyConfig

`CiliumEnvoyConfig` cho phép configure Envoy listeners, routes, clusters trực tiếp — powerful hơn Ingress annotations.

```yaml
apiVersion: cilium.io/v2
kind: CiliumEnvoyConfig
metadata:
  name: frontend-routing
  namespace: production
spec:
  services:
    - name: frontend-service
      namespace: production
      
  resources:
    # Envoy Listener
    - "@type": type.googleapis.com/envoy.config.listener.v3.Listener
      name: frontend-listener
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: frontend
                route_config:
                  virtual_hosts:
                    - name: frontend
                      domains: ["frontend.production.svc.cluster.local"]
                      routes:
                        # Route theo header
                        - match:
                            prefix: "/api/v2/"
                            headers:
                              - name: "x-version"
                                exact_match: "v2"
                          route:
                            cluster: backend-v2
                            
                        # Default route
                        - match:
                            prefix: "/"
                          route:
                            cluster: backend-v1
                            
                # HTTP filters
                http_filters:
                  - name: envoy.filters.http.router
```

### Canary Deployment với Traffic Splitting

```yaml
apiVersion: cilium.io/v2
kind: CiliumEnvoyConfig
metadata:
  name: canary-split
  namespace: production
spec:
  services:
    - name: my-service
      namespace: production
      
  resources:
    - "@type": type.googleapis.com/envoy.config.listener.v3.Listener
      name: my-listener
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: my-service
                route_config:
                  virtual_hosts:
                    - name: my-service
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/"
                          route:
                            weighted_clusters:
                              clusters:
                                - name: my-service-stable
                                  weight: 90       # 90% traffic đến stable
                                - name: my-service-canary
                                  weight: 10       # 10% đến canary
                http_filters:
                  - name: envoy.filters.http.router

    # Clusters
    - "@type": type.googleapis.com/envoy.config.cluster.v3.Cluster
      name: my-service-stable
      connect_timeout: 5s
      type: EDS

    - "@type": type.googleapis.com/envoy.config.cluster.v3.Cluster
      name: my-service-canary
      connect_timeout: 5s
      type: EDS
```

---

## Ingress Controller

Cilium có built-in Ingress Controller (thay nginx/traefik) — L7 routing qua Envoy, tích hợp với LB-IPAM.

```yaml
# Helm values
ingressController:
  enabled: true
  default: true            # default ingress class
  loadbalancerMode: shared  # shared | dedicated
                            # shared: tất cả Ingresses share 1 LoadBalancer IP
                            # dedicated: mỗi Ingress có IP riêng
  service:
    annotations:
      cilium.io/lb-ipam-pool: production-pool   # dùng pool cụ thể
```

```yaml
# Ingress resource
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: production
  annotations:
    ingress.cilium.io/loadbalancer-mode: dedicated    # override global setting
spec:
  ingressClassName: cilium
  rules:
    - host: myapp.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
  tls:
    - hosts:
        - myapp.internal
      secretName: myapp-tls
```

---

## Gateway API (Cilium 1.15+)

Cilium hỗ trợ Kubernetes Gateway API — API mới hơn, expressiveness hơn Ingress.

```yaml
# Enable Gateway API
helm upgrade cilium cilium/cilium \
  --set gatewayAPI.enabled=true \
  --reuse-values

# Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

```yaml
# GatewayClass
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium
spec:
  controllerName: io.cilium/gateway-controller

---
# Gateway (load balancer endpoint)
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: production-gateway
  namespace: production
spec:
  gatewayClassName: cilium
  listeners:
    - name: http
      port: 80
      protocol: HTTP
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        certificateRefs:
          - name: production-tls

---
# HTTPRoute (traffic routing)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-route
  namespace: production
spec:
  parentRefs:
    - name: production-gateway

  hostnames:
    - "myapp.internal"

  rules:
    # Header-based routing
    - matches:
        - headers:
            - name: "x-beta-user"
              value: "true"
      backendRefs:
        - name: my-service-beta
          port: 80
          weight: 100

    # Default
    - backendRefs:
        - name: my-service-stable
          port: 80
          weight: 90
        - name: my-service-canary
          port: 80
          weight: 10
```

---

## Cilium vs Istio

| Feature | Cilium Service Mesh | Istio |
|---------|--------------------|----|
| Architecture | Sidecar-free (eBPF + node Envoy) | Sidecar (Envoy per pod) |
| Resource overhead | Thấp (~0 per pod) | Cao (~50-100MB RAM/pod) |
| mTLS | Có (SPIFFE/SPIRE) | Có (Citadel CA) |
| L7 traffic management | CiliumEnvoyConfig, Gateway API | VirtualService, DestinationRule |
| Observability | Hubble (built-in, eBPF) | Kiali, Jaeger (cần install thêm) |
| Maturity | Beta/RC | Stable, CNCF Graduated |
| Learning curve | Medium | High |
| Feature completeness | ~80% của Istio | 100% |
| Best for | Performance-sensitive, Cilium-native | Full-featured service mesh |

---

## Gotchas

- **CiliumEnvoyConfig và Envoy version**: Cilium bundle specific Envoy version. Không phải tất cả Envoy xDS API v3 resources đều supported. Check Cilium changelog khi upgrade.
- **Gateway API và CRD version**: Gateway API có nhiều versions (v1alpha1, v1beta1, v1). Cilium support phụ thuộc vào version. Luôn check compatibility matrix.
- **mTLS và non-mTLS mixed**: Khi có `authentication.mode: required`, services không support mTLS sẽ bị block. Migration path: `required` → `test` mode (log failures nhưng không block) → `required`.
- **Shared LoadBalancer mode**: Nhiều Ingresses share 1 IP → routing dựa trên Host header. Nếu client không gửi Host header → không match → 404. Test với curl -H "Host: myapp.internal".
- **CiliumEnvoyConfig scope**: CiliumEnvoyConfig apply cho tất cả Envoy listeners trong namespace. Misconfiguration có thể affect tất cả traffic của namespace. Test trên staging trước.
