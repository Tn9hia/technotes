---
title: Kubernetes Fundamentals
tags:
  - kubernetes
  - fundamentals
  - cka
  - index
date: 2026-07-18
---

# Kubernetes Fundamentals

Kiến thức nền tảng K8s — tương ứng phạm vi thi **CKA**. Kiến thức nâng cao/production xem [[Advanced & Production]], bảo mật chuyên sâu cho **CKS** xem [[CKS]].

## Contents

| Folder | Nội dung |
|---|---|
| [[Kubernetes/Fundamentals/Overview\|Overview]] | Port quan trọng trong cluster |
| Core Components | API Server, Controller Manager, ETCD, Kube Scheduler, Kubelet, Kube proxy, Pod, Deployment, ReplicaSet, Namespace, Services, PriorityClass & Preemption, cài đặt cluster ([[Install k8s]]) |
| Scheduling | Manual Scheduling, Static Pod, Multiple Scheduler, Scheduler Profiles, Label & Selector, NodeSelector, Node Affinity, Taints & Tolerations, Priority Classes, Resource requirements & limits, DaemonSet, Admission control |
| Networking | [[Kubernetes/Fundamentals/Networking/Networking\|Networking]] tổng quan, [[CNI]], DNS trong K8s, Ingress, Gateway API, Network Policies |
| Storage | Volumes, Persistent Volume, Persistent Volume Claim, Storage Class, Volume driver plugin |
| Security | RBAC, Service Account, Secret, Security Context, TLS trong K8s, Image security, KubeConfig File |
| Application Lifecycle | ConfigMaps, Command/Argument & Environment variable, Multi Container Pod, CRD, Pod QoS, Updates & Rollbacks, Autoscaling, Vertical Pod Autoscaling |
| Cluster Maintenance | Node Maintenance, Backup & restore resource, Upgrade Cluster |
| Troubleshooting | Metrics Server (`kubectl top`), xem log |
