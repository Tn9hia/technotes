A `kubeconfig` file is like your **passport to a Kubernetes cluster** - it tells the `kubectl` command-line tool **which cluster to talk to, who you are, and how to securely connect**.
## What It Contains
A _context_ element in a kubeconfig file is used to group access parameters under a convenient name. Each context has three parameters: cluster, namespace, and user. By default, the `kubectl` command-line tool uses parameters from the _current context_ to communicate with the cluster.

It’s usually a YAML file that includes:
- **Clusters**: The addresses and certificate info of the clusters you can access.
- **Users**: Authentication details (like tokens or client certificates).
- **Contexts**: Named combinations of cluster + user + namespace.

### Sample kubeConfig file
the two clusters, two users, and three contexts:

```
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: fake-ca-file
    //certificate-authority-data: base64 of fake-ca-file
    server: https://1.2.3.4
  name: development
- cluster:
    insecure-skip-tls-verify: true
    server: https://5.6.7.8
  name: test
contexts:
- context:
    cluster: development
    namespace: frontend
    user: developer
  name: dev-frontend
- context:
    cluster: development
    namespace: storage
    user: developer
    namespace: finance
  name: dev-storage
- context:
    cluster: test
    namespace: default
    user: experimenter
  name: exp-test
current-context: ""
kind: Config
preferences: {}
users:
- name: developer
  user:
    client-certificate: fake-cert-file
    client-key: fake-key-file
- name: experimenter
  user:
    # Documentation note (this comment is NOT part of the command output).
    # Storing passwords in Kubernetes client config is risky.
    # A better alternative would be to use a credential plugin
    # and store the credentials separately.
    # See https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins
    password: some-password
    username: exp
```

### Useful command
```
kubectl config view
```

```
kubectl use-context prod-user@production
```

## Create a new user for cluster
### Create private key + CSR
```bash
# Generate private key
openssl genrsa -out nghia.key 2048

# Tạo CSR (Certificate Signing Request)
openssl req -new -key nghia.key -out nghia.csr -subj "/CN=nghia/O=system:masters"
```
**Note**:
-  `/CN=nghia` → username trong k8s
- `/O=system:masters` → group có quyền cluster-admin built-in sẵn trong k8s

### Encode CSR with base64
```bash
cat nghia.csr | base64 | tr -d "\n"
```

### Create CertificateSigningResource
```yaml
# file: nghia-csr.yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: nghia
spec:
  request: <PASTE_BASE64_CSR_Ở_ĐÂY>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 31536000  # 1 năm (tùy chỉnh nếu cần)
  usages:
  - client auth
```

### Approve CSR
```bash
kubectl certificate approve nghia
```

### Get Certificate
```bash
kubectl get csr nghia -o jsonpath='{.status.certificate}' | base64 -d > nghia.crt
```

### Create kubeconfig file
```bash
# Set cluster info
kubectl config set-cluster <TÊN_CLUSTER> \
  --certificate-authority=/path/to/ca.crt \
  --server=https://<K8S_API_SERVER>:6443 \
  --kubeconfig=nghia-kubeconfig

# Set credentials
kubectl config set-credentials nghia \
  --client-certificate=nghia.crt \
  --client-key=nghia.key \
  --kubeconfig=nghia-kubeconfig

# Set context
kubectl config set-context nghia@<TÊN_CLUSTER> \
  --cluster=<TÊN_CLUSTER> \
  --user=nghia \
  --kubeconfig=nghia-kubeconfig
  
# Add cluserrolebiding
kubectl create clusterrolebinding nghia-cluster-admin \
  --clusterrole=cluster-admin \
  --user=nghia

# Use context
kubectl config use-context nghia@<TÊN_CLUSTER> --kubeconfig=nghia-kubeconfig
```

### Full file kubeconfig
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <PASTE_CA_CERT_BASE64_HERE>  # ← Thay vì certificate-authority
    server: https://10.100.10.25:6443
  name: prod-cluster
contexts:
- context:
    cluster: prod-cluster
    user: nghia
  name: nghia@prod-cluster
current-context: nghia@prod-cluster
kind: Config
users:
- name: nghia
  user:
    client-certificate-data: <PASTE_CLIENT_CERT_BASE64_HERE>  # ← Thay vì client-certificate
    client-key-data: <PASTE_CLIENT_KEY_BASE64_HERE>           # ← Thay vì client-key
```
### Test permission
```bash
kubectl --kubeconfig=nghia-kubeconfig auth can-i "*" "*"
# Output: yes (vì đã ở group system:masters)
```

## Reference
- https://kubernetes.io/docs/tasks/tls/certificate-issue-client-csr/
- https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#set-the-kubeconfig-environment-variable