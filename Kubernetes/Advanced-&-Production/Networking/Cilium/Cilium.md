---
title: Cilium
tags:
  - cilium
  - cni
  - ebpf
  - networking
  - index
date: 2026-04-30
---

# Cilium

Cilium là eBPF-based CNI cho Kubernetes — thay thế iptables/kube-proxy, cung cấp L7-aware security policy, observability qua Hubble, và service mesh không cần sidecar. Đây là CNI chính của team.

## Contents

| File | Nội dung |
|------|----------|
| [[Kubernetes/Advanced-&-Production/Networking/Cilium/Overview]] | What/Why/When/Where/How, key config, security, ops runbook, gotchas |
| [[eBPF & Architecture]] | eBPF primer, BPF maps, datapath (pod→pod, VXLAN, native routing), identity model, components (Agent/Operator/Envoy), install & migration |
| [[Network Policy]] | CiliumNetworkPolicy vs NetworkPolicy, L3/L4/L7 (HTTP/gRPC/Kafka/DNS), entity selectors, CiliumClusterwideNetworkPolicy, policy debugging |
| [[Hubble]] | Flow observation, CLI queries, service map UI, Prometheus metrics, alert rules, proxy-visibility annotation |
| [[BGP & LoadBalancer]] | CiliumBGPPeeringPolicy, LB-IPAM (CiliumLoadBalancerIPPool), L2 Announcements, so sánh với MetalLB |
| [[Encryption]] | WireGuard vs IPsec, enable/verify, key rotation, performance |
| [[Service Mesh]] | Sidecar-free architecture, mTLS + SPIFFE, CiliumEnvoyConfig, canary traffic splitting, Cilium Ingress, Gateway API |
| [[Cluster Mesh]] | Multi-cluster setup, global services, affinity-based routing, cross-cluster NetworkPolicy, external workloads |

---

## Quick Reference

### Install

```bash
helm repo add cilium https://helm.cilium.io/

helm install cilium cilium/cilium \
  --version 1.15.5 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=<API_SERVER_IP> \
  --set k8sServicePort=6443 \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true

cilium status --wait
cilium connectivity test
```

### Status & Debug

```bash
cilium status
cilium connectivity test

# Policy trace
cilium policy trace \
  --src-k8s-pod <ns>/<pod> \
  --dst-k8s-pod <ns>/<pod> \
  --dport 8080/TCP

# Flow observation
hubble observe --namespace <ns> --verdict DROPPED
hubble observe --from-service <ns>/<svc> --protocol http

# BPF tables
cilium bpf endpoint list
cilium bpf lb list
cilium bpf policy get <endpoint-id>

# Sysdump
cilium sysdump
```

### Key CRDs

| CRD | Purpose |
|-----|---------|
| `CiliumNetworkPolicy` | Namespaced L3/L4/L7 policy |
| `CiliumClusterwideNetworkPolicy` | Cluster-wide policy |
| `CiliumBGPPeeringPolicy` | BGP peer configuration |
| `CiliumLoadBalancerIPPool` | LB IP pool definition |
| `CiliumEnvoyConfig` | Envoy proxy configuration |
| `CiliumClusterWideEnvoyConfig` | Cluster-wide Envoy config |
| `CiliumExternalWorkload` | Non-K8s workloads trong mesh |

---

## Cross-References

- Standard NetworkPolicy (CKA level): [[Network Security]]
- Hubble metrics → Prometheus: [[Monitoring]]
- mTLS certificates via cert-manager: [[Network Security]]
- Vault secrets cho Cilium config: [[Kubernetes Integration]]
