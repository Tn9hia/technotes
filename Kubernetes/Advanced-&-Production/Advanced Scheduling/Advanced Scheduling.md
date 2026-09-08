---
title: Advanced Kubernetes Scheduling
tags:
  - kubernetes
  - scheduling
  - keda
  - autoscaling
  - topology
date: 2026-04-26
---

# Advanced Scheduling

## TopologySpreadConstraints

TopologySpreadConstraints phân phối pods đều giữa failure domains (zones, nodes) — cải thiện availability và load distribution.

### Tại sao cần?

```
Không có constraints:
  Node A (zone-a): pod1, pod2, pod3
  Node B (zone-b): (empty)
  → Node A down = all pods down

Với TopologySpreadConstraints:
  Node A (zone-a): pod1, pod2
  Node B (zone-b): pod3, pod4
  → Node A down = 50% pods vẫn up
```

### Basic Usage

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
        # Spread đều giữa availability zones
        - maxSkew: 1              # tối đa 1 pod chênh lệch giữa zones
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule   # hard requirement
          labelSelector:
            matchLabels:
              app: myapp

        # Spread đều giữa nodes (trong mỗi zone)
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway  # soft — cố gắng nhưng không block
          labelSelector:
            matchLabels:
              app: myapp
```

### whenUnsatisfiable Options

```yaml
# DoNotSchedule (hard): pod pending nếu không thể satisfy constraint
# ScheduleAnyway (soft): schedule dù vi phạm, nhưng minimize skew

# Production recommendation:
# - DoNotSchedule cho zone spread (availability critical)
# - ScheduleAnyway cho node spread (performance optimization)
```

### minDomains (K8s 1.25+)

```yaml
# Yêu cầu minimum số domains có pods
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: myapp
    minDomains: 3    # phải có pods ở ít nhất 3 zones
    # Nếu cluster chỉ có 2 zones → pods pending cho đến khi có zone thứ 3
```

### nodeAffinityPolicy và nodeTaintsPolicy (K8s 1.26+)

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: myapp
    nodeAffinityPolicy: Honor    # Honor: chỉ count nodes matching nodeAffinity
    nodeTaintsPolicy: Honor      # Honor: exclude tainted nodes từ calculation
```

---

## Pod Disruption Budget (PDB)

PDB đảm bảo minimum số pods available khi có disruptions (node drain, rolling update).

```yaml
# Minimum 2 pods available bất kỳ lúc nào
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 2      # hoặc percentage: "80%"
  selector:
    matchLabels:
      app: myapp

# Hoặc maxUnavailable
spec:
  maxUnavailable: 1    # tối đa 1 pod unavailable cùng lúc
  selector:
    matchLabels:
      app: myapp
```

```bash
# Xem PDB status
kubectl get pdb -n production
# NAME        MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# myapp-pdb   2               N/A               3                     5d

# Node drain sẽ respect PDB
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
# Nếu drain sẽ vi phạm PDB → drain blocked, error message
```

---

## KEDA — Event-Driven Autoscaling

KEDA (Kubernetes Event-Driven Autoscaling) scale deployments/jobs dựa trên external metrics (queue length, Kafka lag, cron schedule, etc.) — không chỉ CPU/memory như HPA.

### Install

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda -n keda --create-namespace
```

### ScaledObject (scale Deployment)

```yaml
# Scale dựa trên Kafka consumer lag
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: order-processor        # Deployment name
  minReplicaCount: 1
  maxReplicaCount: 50
  cooldownPeriod: 300           # giây trước khi scale down
  pollingInterval: 30           # check trigger mỗi 30s

  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka.kafka:9092
        consumerGroup: order-processors
        topic: orders
        lagThreshold: "100"     # 1 replica cho mỗi 100 messages
        offsetResetPolicy: latest

---
# Scale dựa trên RabbitMQ queue length
triggers:
  - type: rabbitmq
    metadata:
      host: amqp://user:pass@rabbitmq:5672/
      queueName: task-queue
      queueLength: "20"         # 1 replica cho mỗi 20 messages

---
# Scale dựa trên Prometheus metric
triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      metricName: http_requests_in_flight
      threshold: "100"          # 1 replica cho mỗi 100 in-flight requests
      query: sum(http_requests_in_flight{job="myapp"})

---
# Cron-based scaling (business hours)
triggers:
  - type: cron
    metadata:
      timezone: Asia/Ho_Chi_Minh
      start: "0 8 * * 1-5"     # 8am thứ 2-6
      end: "0 18 * * 1-5"      # 6pm thứ 2-6
      desiredReplicas: "10"
```

### ScaledJob (scale Jobs)

```yaml
# KEDA tạo Jobs thay vì scale Deployment replicas
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: image-processor
  namespace: production
spec:
  jobTargetRef:
    template:
      spec:
        containers:
          - name: processor
            image: image-processor:v1
        restartPolicy: Never
  maxReplicaCount: 20
  pollingInterval: 10
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5

  triggers:
    - type: aws-sqs-queue
      authenticationRef:
        name: keda-sqs-auth       # TriggerAuthentication resource
      metadata:
        queueURL: https://sqs.ap-southeast-1.amazonaws.com/123/image-queue
        queueLength: "1"          # 1 job per message
        awsRegion: ap-southeast-1
```

```yaml
# TriggerAuthentication — creds cho KEDA triggers
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-sqs-auth
  namespace: production
spec:
  podIdentity:
    provider: aws    # dùng IRSA trên EKS
```

---

## Cluster Autoscaler

Cluster Autoscaler (CA) tự động thêm/xóa nodes khi pods pending hoặc nodes idle.

### Khi nào CA scale up?

```
Pod pending (không schedule được do thiếu resources)
      ↓
CA check: có thể schedule pod này nếu thêm node không?
      ↓ yes
CA request cloud provider API để provision node
      ↓
Node join cluster → pod scheduled
```

### Khi nào CA scale down?

```
Node utilization < threshold (mặc định 50%)
và
Tất cả pods trên node có thể schedule ở nodes khác
và
PDB không bị vi phạm
      ↓
CA cordon + drain + delete node
```

### Install (EKS example)

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  -n kube-system \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=ap-southeast-1 \
  --set rbac.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::123:role/CA-Role \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.skip-nodes-with-system-pods=false \
  --set extraArgs.scale-down-utilization-threshold=0.5
```

```yaml
# Node group annotations (cho CA discovery)
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
nodeGroups:
  - name: workers
    labels:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/my-cluster: "owned"
    minSize: 2
    maxSize: 20
```

### CA Annotations

```yaml
# Prevent CA từ scaling down specific pods
metadata:
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
    # Dùng cho: pods với local state, critical single-instance components

# Prevent CA từ scaling down node
# Trên Node (thêm bởi admin hoặc tool):
kubectl annotate node node-1 \
  cluster-autoscaler.kubernetes.io/scale-down-disabled=true
```

---

## Descheduler

Descheduler evict pods để rebalance cluster sau khi scheduling suboptimal decisions:
- Nodes overloaded sau khi CA scale down
- Pods stuck trên failed nodes (đã recovered)
- Taint/affinity rules thay đổi sau khi pods scheduled

```bash
helm install descheduler kubernetes-sigs/descheduler \
  -n kube-system \
  --set schedule="*/5 * * * *"   # chạy mỗi 5 phút
```

```yaml
# Descheduler policy
apiVersion: "descheduler/v1alpha1"
kind: "DeschedulerPolicy"
profiles:
  - name: default
    pluginConfig:
      - name: DefaultEvictor
        args:
          evictSystemCriticalPods: false
          evictFailedBarePods: true
          evictLocalStoragePods: false    # không evict pods có emptyDir
          nodeFit: true                   # chỉ evict nếu có chỗ khác

    plugins:
      balance:
        enabled:
          - RemoveDuplicates          # xóa duplicate pods trên cùng node
          - RemovePodsViolatingTopologySpreadConstraint   # fix topology violations
          - LowNodeUtilization        # di chuyển pods từ underutilized nodes

      deschedule:
        enabled:
          - RemovePodsHavingTooManyRestarts   # evict crash-looping pods
          - PodLifeTime                        # evict pods quá già
          - RemoveFailedPods

    pluginConfig:
      - name: LowNodeUtilization
        args:
          thresholds:
            cpu: 20
            memory: 20
            pods: 20
          targetThresholds:
            cpu: 50
            memory: 50
            pods: 50
      - name: RemovePodsHavingTooManyRestarts
        args:
          podRestartThreshold: 5
          includingInitContainers: true
```

---

## Priority Classes

PriorityClass quyết định thứ tự scheduling và eviction khi cluster tight.

```yaml
# Tạo priority classes
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-system
value: 1000000          # cao hơn = ưu tiên hơn
globalDefault: false
preemptionPolicy: PreemptLowerPriority   # có thể preempt lower priority pods
description: "System critical components (monitoring, logging)"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
preemptionPolicy: PreemptLowerPriority

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: default-priority
value: 0
globalDefault: true    # pods không có priorityClassName dùng class này

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: -100
preemptionPolicy: Never    # không preempt, chỉ schedule khi có resources
```

```yaml
# Dùng trong pod
spec:
  priorityClassName: high-priority
  containers:
    - name: app
      ...
```

```
Priority hierarchy:
  system-cluster-critical (2000000001) — K8s built-in
  system-node-critical    (2000001000) — K8s built-in
  critical-system         (1000000)    — custom
  high-priority           (100000)     — custom
  default-priority        (0)          — custom default
  low-priority            (-100)       — custom
```

---

## Gotchas

- **TopologySpreadConstraints và PDB xung đột**: PDB `minAvailable: 2` + TopologySpreadConstraints `DoNotSchedule` + 2 zones = 1 pod/zone. Nếu 1 zone mất → CA cố spin up zone khác, nhưng PDB prevent drain → deadlock. Cần plan capacity carefully.
- **KEDA và HPA conflict**: Không dùng cả KEDA ScaledObject và HPA cho cùng 1 Deployment. KEDA tạo HPA internally — nếu có HPA manual → conflict. KEDA sẽ override HPA settings nhưng behavior unpredictable.
- **Cluster Autoscaler và spot instances**: Spot instances bị terminate unexpectedly. CA không evict pods khi spot instance bị terminate bởi cloud provider (đó là forced eviction). Dùng PDB và topologySpreadConstraints để survive spot interruptions.
- **Descheduler eviction và application restart cost**: Descheduler evict pods = pods restart. Nếu app startup time lâu (Java 30-60s) → quá nhiều descheduling = availability issues. Tune descheduler schedule interval và thresholds.
- **PriorityClass và resource starvation**: Low-priority workloads có thể bị evict liên tục khi cluster tight → starvation. Set proper resource quotas per namespace để balance.
