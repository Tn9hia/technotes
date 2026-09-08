## Rollout - Deploy application
Apply for:
- deployments
- daemonsets
- statefulsets

### Command
```
kubectl rollout status <resource>
```

```
kubectl rollout history <resource>
```

```
kubectl rollout undo <resource>
```

```
kubectl rollout undo <resource>
```


## Deployment strategies
- Recreate: take down all pod and deploy a new one
- Rolling update: replace one by one pod => **Default** deployment