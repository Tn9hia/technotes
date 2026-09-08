
Admission controllers are code within the Kubernetes [API server](https://kubernetes.io/docs/concepts/architecture/#kube-apiserver) that check the data arriving in a request to modify a resource.

Admission controllers apply to requests that create, delete, or modify objects. Admission controllers can also block custom verbs, such as a request to connect to a pod via an API server proxy. Admission controllers do _not_ (and cannot) block requests to read (**get**, **watch** or **list**) objects, because reads bypass the admission control layer.

Admission control mechanisms may be _validating_, _mutating_, or both. Mutating controllers may modify the data for the resource being modified; validating controllers may not.

## Phase
```
Request → Authentication → Authorization 
    ↓
Mutating Admission (phase 1)
    ↓
Object Schema Validation
    ↓
Validating Admission (phase 2)
    ↓
Persist to etcd
```

**Why 2 phase?**
- Mutating: Edit object (inject sidecar, set defaults, add labels...)
- Validating: check rules (quotas, policies, naming conventions...)

## Important Admission Controllers (built-in)
**Mutating:**
- `MutatingAdmissionWebhook` - Custom logic through webhook
- `DefaultStorageClass` - auto assign storageclass
- `ServiceAccount` - auto attach SA token to pod
- `PodSecurity` 

**Validating:**
- `ValidatingAdmissionWebhook` - custom validation
- `ResourceQuota` - enforce quotas
- `LimitRanger` - check limits/requests
- `PodSecurity` - enforce Pod Security Standards (baseline/restricted/privileged)
- `NamespaceLifecycle` - prevent create resources in terminating namespace
## Command
- View enabled Admission
```
kubectl exec kube-apiserver-k8s-master01 -n kube-system -- kube-apiserver -h |grep  --enable-admission-plugins 
```

- Enable
```shell
--enable-admission-plugins=NodeRestriction,PodSecurity,LimitRanger
```
- Disable
```shell
--disable-admission-plugins=ServiceAccount
```

## Debug tips
```shell
# Check webhook config
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations

# Describe để xem rules
kubectl describe validatingwebhookconfiguration <name>

# Check logs của webhook service
kubectl logs -n <namespace> <webhook-pod>

# Test với dry-run
kubectl apply --dry-run=server -f manifest.yaml
```



## Reference
- https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/