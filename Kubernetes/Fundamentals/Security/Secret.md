**A Secret** is an object that contains a small amount of sensitive data such as a password, a token, or a key. Such information might otherwise be put in a Pod specification or in a container image. Using a Secret means that you don't need to include confidential data in your application code.

**Secrets** are similar to **ConfigMaps** but are specifically intended to hold confidential data.

> [!CAUTION]
>Kubernetes Secrets are, by default, stored unencrypted in the API server's underlying data store (etcd). Anyone with API access can retrieve or modify a Secret, and so can anyone with access to etcd. Additionally, anyone who is authorized to create a Pod in a namespace can use that access to read any Secret in that namespace; this includes indirect access such as the ability to create a Deployment.
>
In order to safely use Secrets, take at least the following steps:
>
>1. [Enable Encryption at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) for Secrets.
>2. [Enable or configure RBAC rules](https://kubernetes.io/docs/reference/access-authn-authz/authorization/) with least-privilege access to Secrets.
>3. Restrict Secret access to specific containers.
>4. [Consider using external Secret store providers](https://secrets-store-csi-driver.sigs.k8s.io/concepts.html#provider-for-the-secrets-store-csi-driver).
>
For more guidelines to manage and improve the security of your Secrets, refer to [Good practices for Kubernetes Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/).

## Type of secrets

- `Opaque`: Generic key-value pairs (most common)
```shell
kubectl create secret generic my-secret --from-literal=password=123456
```

- `kubernetes.io/dockerconfigjson`: Store Docker registry credentials
```shell
kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```

- `kubernetes.io/tls`: TLS certs (`tls.crt`, `tls.key`)
```shell
kubectl create secret tls my-tls-secret --cert=path/to/cert/file --key=path/to/key/file
```

- `kubernetes.io/service-account-token`: Auto-created for ServiceAccounts (Legacy)

## How to create secret?
*Imperative*
```bash
kubectl create secret generic <secret-name> --from-literal=key=value
```

*Declaretive*
```bash
kubectl create -f file.yaml
```

**Example**
```bash
echo "pass" |base64 
cGFzcwo=
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dotfile-secret
data:
  .secret-file: dmFsdWUtMg0KDQo=
  abc: cGFzcwo=
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-dotfiles-pod
spec:
  volumes:
    - name: secret-volume
      secret:
        secretName: dotfile-secret
  containers:
    - name: dotfile-test-container
      image: registry.k8s.io/busybox
      command:
        - ls
        - "-l"
        - "/etc/secret-volume"
      volumeMounts:
        - name: secret-volume
          readOnly: true
          mountPath: "/etc/secret-volume"
```
## How to add secret to a pod?

**ENV**
```yaml
envFrom:
	- SecretRef:
		Name: app-config
```
**Single Env**
```yaml
env:
	- Name: DB_Password
	  ValueFrom:
		SecretKeyRef:
			name: app-secret
			key: DB_Password
```
**Volume**
```yaml
Volumes
- name: app-secret-volume
  secret: 
	secretName: app-secret
```

`env` **vs** `envFrom`**?**
- **Dùng** `env` khi chỉ muốn lấy một số key từ secret.
- **Dùng** `envFrom` khi muốn lấy toàn bộ secret thành biến môi trường.
EX:
```yaml
spec:
  containers:
  - name: webapp
    image: kodekloud/simple-webapp-mysql
    env:
      - name: DB_HOST
        valueFrom:
          secretKeyRef:
            name: db-secret
            key: DB_Host
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: db-secret
            key: DB_Password
      - name: DB_USER
        valueFrom:
          secretKeyRef:
            name: db-secret
            key: DB_User

```

```yaml
spec:
  containers:
  - name: webapp
    image: kodekloud/simple-webapp-mysql
    envFrom:
    - secretRef:
        name: db-secret

```