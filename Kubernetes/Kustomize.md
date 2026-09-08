
## 🎯 Core Concepts

```
kustomization.yaml = brain của mọi thứ
├── bases/          → template gốc (shared config)
├── overlays/       → env-specific shit (dev/staging/prod)
└── patches/        → sửa config không động vào base
```

## 📁 Cấu trúc thường gặp

```
my-app/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patch-deployment.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patch-deployment.yaml
    └── prod/
        ├── kustomization.yaml
        └── patch-deployment.yaml
```

| Folder/file        | Purpose                                                                                               |
| ------------------ | ----------------------------------------------------------------------------------------------------- |
| base               | General configuration for all the enviroment (Deployment, Service, ConfigMap,..)                      |
| overlays           | Contains each specific environment (dev, staging, prod...) to patch or override values ​​in the base. |
| kustomization.yaml | Define resources, patches, labels, images, namespace, v.v.                                            |
| patch-\*.yaml      | Patch file to override each part (eg replica count, image tag, env vars).                             |
## 🔧 Basic Commands (nhớ nằm lòng)

```bash
# Preview trước khi apply (QUAN TRỌNG trong thi)
kubectl kustomize ./overlays/prod

# Apply trực tiếp
kubectl apply -k ./overlays/prod

# Diff để check thay đổi
kubectl diff -k ./overlays/prod

# Build ra file (nếu cần debug)
kustomize build ./overlays/prod > output.yaml
```

## 📝 kustomization.yaml - Base Example

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Resources gốc
resources:
  - deployment.yaml
  - service.yaml

# Prefix/Suffix cho tất cả resources
namePrefix: app-
nameSuffix: -v1

# Common labels (đánh hết)
commonLabels:
  app: myapp
  team: platform

# Common annotations
commonAnnotations:
  managed-by: kustomize

# ConfigMap/Secret generators
configMapGenerator:
  - name: app-config
    literals:
      - DB_HOST=localhost
    files:
      - config.properties

secretGenerator:
  - name: app-secret
    literals:
      - password=super-secret

# Images (update version không cần sửa YAML)
images:
  - name: nginx
    newTag: "1.21.0"
```

### Common Transformations
- `CommonLabel`: adds a label to all kubernetes resource
- `namePrefix/Suffix`: adds a common prefix-suffix to all resource names
- `Namespace`: adds a common namespace to all resources
- `commonAnotations`: adds an annotation to all resources

## 🎨 Overlays Example (prod)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Kế thừa base
bases:
  - ../../base

# Override namespace
namespace: production

# Patch replicas
patchesStrategicMerge:
  - replica-patch.yaml

# Hoặc dùng JSON6902 patch (fancy hơn)
patchesJson6902:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: myapp
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5

# Update image tag
images:
  - name: nginx
    newTag: 1.21.0-prod
```

## 💉 Patch Strategies

### Strategic Merge Patch (dễ nhất)

```yaml
# replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3  # chỉ cần ghi cái muốn đổi
```

### JSON Patch (hardcore mode)

```yaml
- op: add|replace|remove
  path: /spec/replicas
  value: 5
```

## 🔑 ConfigMap/Secret Tricks

```yaml
# From literals
configMapGenerator:
  - name: my-config
    literals:
      - KEY=value

# From files
configMapGenerator:
  - name: my-config
    files:
      - application.properties
      - config.json

# From env file
configMapGenerator:
  - name: my-config
    envs:
      - .env

# Behavior: create/replace/merge
configMapGenerator:
  - name: my-config
    behavior: merge  # hoặc create/replace
    literals:
      - KEY=value
```

## 🎯 CKA Exam Tips

**Commands phải thuộc lòng:**

```bash
# Validate trước
kubectl kustomize <dir>

# Apply
kubectl apply -k <dir>

# Delete
kubectl delete -k <dir>

# Dry-run (quan trọng!)
kubectl apply -k <dir> --dry-run=client -o yaml
```

**Common mistakes trong thi:**

- Quên `-k` flag → kubectl sẽ không hiểu kustomization
- Đường dẫn sai → phải chỉ đến folder chứa `kustomization.yaml`
- Không verify output trước khi apply

**Quick debug flow:**

1. `kubectl kustomize ./` → xem output
2. Có lỗi → check `kustomization.yaml` syntax
3. OK → `kubectl apply -k ./`

## 🚀 Pro Tips

```yaml
# nameReference - auto update tên ConfigMap/Secret trong Deployment
configurations:
  - kustomizeconfig.yaml

# Replicas transformer
replicas:
  - name: myapp
    count: 3

# commonLabels vs labels
# commonLabels → thêm vào selector (NGUY HIỂM)
# labels → chỉ thêm vào metadata
```

## ⚠️ Lưu ý thi CKA

- Kustomize built-in kubectl từ version 1.14+
- Không cần install riêng
- Folder structure phải đúng
- `kustomization.yaml` phải có trong folder
- `-k` flag = `--kustomize`

**Sample question pattern:**

> "Create a kustomization in /path/to/app that applies deployment.yaml and service.yaml with namespace 'production' and adds label 'env=prod' to all resources"

```bash
cd /path/to/app
cat > kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
commonLabels:
  env: prod
resources:
  - deployment.yaml
  - service.yaml
EOF

kubectl apply -k .
```

---
