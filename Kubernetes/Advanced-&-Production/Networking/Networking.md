---
title: Kubernetes Networking
tags:
  - kubernetes
  - networking
  - cilium
  - cni
  - index
date: 2026-04-30
---

# Kubernetes Networking

## Sections

| Section | Index | Focus |
|---------|-------|-------|
| Cilium | [[Cilium]] | eBPF CNI — team's primary CNI |

---

## Cilium

- [[Kubernetes/Advanced-&-Production/Networking/Cilium/Overview]] — What/Why/When, architecture, key config, ops runbook
- [[eBPF & Architecture]] — eBPF fundamentals, datapath, identity model, components
- [[Network Policy]] — L3/L4/L7 policies, entity selectors, DNS-based, cluster-wide
- [[Hubble]] — Flow visibility, CLI, metrics, Grafana dashboards
- [[BGP & LoadBalancer]] — On-premise LoadBalancer IPs via BGP, LB-IPAM
- [[Encryption]] — WireGuard & IPsec transparent node encryption
- [[Service Mesh]] — Sidecar-free mTLS, traffic management, Gateway API
- [[Cluster Mesh]] — Multi-cluster networking, global services, failover

---

## K8s Networking Fundamentals (CKA — đã biết)

- Pod networking model: mỗi pod có unique IP, pods communicate directly (no NAT)
- Service types: ClusterIP, NodePort, LoadBalancer, ExternalName
- DNS: CoreDNS, `<service>.<namespace>.svc.cluster.local`
- Standard `NetworkPolicy` (namespace-scoped, L3/L4 only)
- Ingress resources

> Với Cilium là CNI chính: standard NetworkPolicy vẫn được support (và coexist với CiliumNetworkPolicy).
> Focus notes mới vào Cilium-specific features.
