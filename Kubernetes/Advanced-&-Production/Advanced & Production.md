---
title: Kubernetes — Advanced & Production
tags:
  - kubernetes
  - production
  - index
date: 2026-07-18
---

# Kubernetes — Advanced & Production

> Kiến thức **nâng cao** và best practices vận hành production. Kiến thức CKA cơ bản được assumed ([[Fundamentals]]). Bảo mật chuyên sâu cho CKS đã tách qua [[CKS]].

## Observability

- [[Monitoring]] — Prometheus Operator, ServiceMonitor/PodMonitor CRDs, PrometheusRule alerts, SLI/SLO với error budget, USE/RED method, key PromQL queries
- [[Logging]] — Fluent Bit + Loki stack, LogQL queries, K8s audit policy tiers, audit log ship to Loki, Falco custom rules, structured logging

---

## Advanced Scheduling

- [[Advanced Scheduling]] — TopologySpreadConstraints, PodDisruptionBudget, KEDA event-driven autoscaling, Cluster Autoscaler, Descheduler, PriorityClass

---

## Production Operations

- [[Production Ops]] — etcd compaction/defrag/backup/restore, Velero cluster backup, cluster upgrade strategy (kubeadm + managed K8s + blue/green), certificate rotation, multi-cluster patterns

---

## Networking (Cilium)

- [[Cilium]] — eBPF & Architecture, Network Policy, BGP & LoadBalancer, Cluster Mesh, Encryption, Hubble, Service Mesh

---

## Cross-References

- CKS security domains (Cluster/System Hardening, Supply Chain, Runtime Security): [[CKS]]
- Deployment strategies (canary, blue/green): [[Deployment Strategies]]
