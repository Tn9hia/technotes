![[Pasted image 20250523134355.png]]
## Phase of scheduling


## Sample
```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
  - schedulerName: no-scoring-scheduler
    plugins:
      preScore:
        disabled:
        - name: '*'
      score:
        disabled:
        - name: '*'
```

## Reference
- https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/