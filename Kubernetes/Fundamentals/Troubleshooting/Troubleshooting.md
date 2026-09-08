## Metrics Server

Metrics Server dùng cAdvisor để thu thập metrics từ node/pod, lưu tạm trong memory (không phải time-series database — xem [[Monitoring]] cho giải pháp production dùng Prometheus).

```bash
kubectl top nodes
kubectl top pods
```

## Log

```bash
kubectl logs -f <pod-name>
```
