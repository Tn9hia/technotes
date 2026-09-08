---
title: Supply Chain Security
tags:
  - kubernetes
  - security
  - cks
  - supply-chain-security
date: 2026-08-18
---

# Supply Chain Security

## Supply Chain Overview & Risks

"Supply chain" trong software cũng giống dây chuyền sản xuất vật lý: nguyên liệu đầu vào (source code, dependencies, base image) phải được kiểm tra chất lượng/an toàn ở từng công đoạn trước khi ra sản phẩm cuối (container chạy trong production). Một supply chain an toàn trải qua các giai đoạn tương ứng:

1. **Source** – code được viết và test trong môi trường tin cậy (trusted dev environment).
2. **Build** – code được compile/đóng gói; cần cô lập môi trường build, giữ dependency cập nhật, và scan lỗ hổng bằng công cụ như OWASP Dependency-Check, Snyk.
3. **Test** – container image được scan tìm lỗ hổng trước khi deploy (Clair, Trivy).
4. **Deploy** – áp dụng các biện pháp bảo vệ production: Pod Security (Policies/Standards), Network Policies, RBAC.

Bỏ qua bất kỳ giai đoạn nào cũng làm tăng rủi ro. Lợi ích của việc làm chuẩn từng giai đoạn: phát hiện lỗ hổng sớm, quản lý resource tốt hơn (không bị gián đoạn sản xuất vì incident bảo mật), dễ đạt compliance, incident response nhanh hơn, và tăng security posture tổng thể.

### Các rủi ro cụ thể khi supply chain management yếu

- **Unpatched vulnerabilities → data breach lớn**: bỏ sót một lỗ hổng đã biết (known CVE) trong một component có thể dẫn tới lộ hàng triệu bản ghi, thiệt hại tài chính, phạt theo quy định, mất uy tín.
- **Untrusted third-party components**: tích hợp package/image chưa được xác minh từ vendor không tin cậy có thể chứa malware ẩn, tạo backdoor xâm nhập hệ thống — chi phí remediation rất tốn kém.
- **Credential không được mã hoá**: lưu secrets/credentials ở dạng plaintext (ví dụ trong config, env var) khiến attacker chiếm được dễ dàng khi truy cập trái phép.
- **Cấu hình quá lỏng lẻo (overly permissive)**: thiếu Network Policy hoặc RBAC trong cluster giúp attacker dễ dàng xâm nhập pod và di chuyển ngang (lateral movement) trong mạng.
- **Container misconfiguration → host compromise**: container cấu hình sai (ví dụ chạy privileged) cho phép attacker "escape" ra khỏi container, chiếm quyền kiểm soát toàn bộ host.

Hậu quả tích luỹ: cyber attack, gián đoạn vận hành, thiệt hại tài chính, hậu quả pháp lý/quy định, và mất lợi thế cạnh tranh. Cần giám sát và cập nhật security practice liên tục ở mọi giai đoạn.

## Image Security

### Tên image và registry

Khi khai báo `image: nginx` trong pod spec, đây thực chất là dạng rút gọn của `docker.io/library/nginx` — tức image chính thức trong tài khoản mặc định `library` trên Docker Hub. Nếu tự host registry riêng, thay `library` bằng tên user/tổ chức, ví dụ `your-company/nginx`.

Khi không chỉ định registry, Kubernetes mặc định pull từ Docker Hub (`docker.io`). Có thể chỉ định registry khác tường minh:

```yaml
image: docker.io/library/nginx
image: gcr.io/kubernetes-e2e-test-images/dnsutils
```

Với ứng dụng nội bộ (không nên public), nên dùng **private registry** — AWS, Azure, GCP đều có sẵn dịch vụ private registry.

### Pull image từ private registry

```bash
# Đăng nhập registry riêng
docker login private-registry.io
docker run private-registry.io/apps/internal-app
```

Pod cần trỏ tới full path của image trong private registry:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: private-registry.io/apps/internal-app
```

Vì kubelet/container runtime trên worker node mới là bên thực sự pull image, Kubernetes cần credentials để xác thực với registry — lưu dưới dạng Secret loại `docker-registry`:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=private-registry.io \
  --docker-username=registry-user \
  --docker-password=registry-password \
  --docker-email=registry-user@org.com
```

Rồi tham chiếu secret này trong pod qua `imagePullSecrets`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: private-registry.io/apps/internal-app
  imagePullSecrets:
    - name: regcred
```

### Minimize base image footprint

Mỗi image được build dựa trên một **parent image**; image không có parent (build từ `FROM scratch`) gọi là **base image**. Ví dụ chuỗi kế thừa: `httpd` image của bạn → base trên `httpd` chính thức → base trên `debian:buster-slim` → base trên `scratch`.

```dockerfile
# Dockerfile – My Custom Webapp
FROM httpd
COPY index.html htdocs/index.html
```

**Best practice khi build image:**

1. **Modular hoá**: không nhồi nhiều ứng dụng (web server + database...) vào một image; mỗi container chỉ nên làm một việc — dễ quản lý dependency, dễ scale độc lập, tăng bảo mật nhờ cô lập.
2. **Container ephemeral, không lưu state bên trong**: dùng external volume hoặc caching service (Redis...) để persist data, không lưu trong container.
3. **Chọn base image cẩn thận**: ưu tiên image chính thức/verified publisher, được cập nhật thường xuyên.
4. **Giảm kích thước image**: dùng bản OS tối giản, chỉ cài lib cần thiết, xoá file tạm và tool không cần cho production (curl, wget — dễ bị lợi dụng), cân nhắc bỏ luôn package manager (yum/apt) nếu không cần ở runtime.
5. **Tách biệt image dev và production**: image dev có thể có thêm debug tool, không nên mang các tool đó vào production.

Giảm số lượng package = giảm attack surface = giảm số lỗ hổng. Ví dụ Google **distroless images** chỉ chứa app + runtime dependency, không có shell, package manager, hay network tool.

So sánh minh hoạ bằng Trivy — cùng là httpd nhưng bản Alpine (nhỏ hơn) có ít lỗ hổng hơn hẳn:

```bash
trivy image httpd
# httpd (debian 10.8)
# Total: 124 (UNKNOWN: 0, LOW: 88, MEDIUM: 9, HIGH: 25, CRITICAL: 2)

trivy image httpd:alpine
# httpd:alpine (alpine 3.12.4)
# Total: 0 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 0)
```

Luôn kiểm tra base image chọn có được vá lỗi bảo mật thường xuyên hay không trước khi dùng.

## Image Signing với Cosign

Sau khi build và scan image, bước tiếp theo trong supply chain là **ký (sign) image** để đảm bảo tính toàn vẹn (integrity) và nguồn gốc (provenance) — khi image được ký, downstream consumer (cluster, admission controller) có thể verify chữ ký trước khi chạy, tránh chạy nhầm image đã bị tamper hoặc image giả mạo được đẩy lên registry qua một pipeline không tin cậy. Công cụ phổ biến nhất cho việc này là **Cosign** (thuộc dự án Sigstore) — sign/verify container image, chữ ký được lưu ngay trong OCI registry cạnh image (dưới dạng manifest riêng), không cần hạ tầng PKI phức tạp.

Cosign hỗ trợ hai cách ký: **keyless** (dựa trên OIDC identity, không cần quản lý private key) và **key-based** (cặp khoá public/private truyền thống).

### Keyless Signing (OIDC)

Đây là cách được khuyến nghị cho production vì không cần lưu trữ/luân chuyển private key — danh tính người ký được xác thực qua OIDC token (ví dụ GitHub Actions OIDC), Cosign lấy short-lived certificate từ Fulcio CA và ghi log ký vào Rekor (transparency log, xem phần riêng bên dưới). Chạy trong CI:

```bash
# Trong GitHub Actions CI
- name: Sign image
  run: |
    cosign sign \
      --yes \
      --identity-token=$(cat $ACTIONS_ID_TOKEN_REQUEST_TOKEN) \
      ghcr.io/myorg/myapp@${{ steps.build.outputs.digest }}
  env:
    COSIGN_EXPERIMENTAL: "1"    # bật keyless mode
```

Runner cần quyền `id-token: write` để lấy được OIDC token từ GitHub.

### Key-based Signing

Khi cần kiểm soát khoá thủ công (ví dụ air-gapped environment, hoặc chưa sẵn sàng dùng keyless), dùng cặp khoá Cosign:

```bash
# Generate key pair
cosign generate-key-pair
# → cosign.key (private, bảo vệ bằng passphrase)
# → cosign.pub (public, phân phối cho consumer)

# Lưu private key vào K8s secret hoặc Vault
kubectl create secret generic cosign-key \
  --from-file=cosign.key=./cosign.key \
  -n cicd

# Ký image (sau khi đã push)
cosign sign \
  --key cosign.key \
  ghcr.io/myorg/myapp:v1.2.3

# Verify
cosign verify \
  --key cosign.pub \
  ghcr.io/myorg/myapp:v1.2.3
```

Có thể đính kèm annotation (metadata) vào chữ ký, ví dụ ghi lại build-id/git-sha/pipeline URL để trace provenance, và bắt buộc verify phải khớp annotation đó:

```bash
# Ký kèm annotations
cosign sign \
  --key cosign.key \
  --annotations "build-id=$CI_BUILD_ID" \
  --annotations "git-sha=$GIT_COMMIT" \
  --annotations "pipeline=$CI_PIPELINE_URL" \
  ghcr.io/myorg/myapp:v1.2.3

# Verify kèm kiểm tra annotation
cosign verify \
  --key cosign.pub \
  --annotations "git-sha=$GIT_COMMIT" \
  ghcr.io/myorg/myapp:v1.2.3
```

### Full CI Pipeline: Build → Scan → Sign → SBOM → Attest

Trong thực tế, các bước supply chain (build, scan, sign, sinh SBOM) được nối liền trong một pipeline duy nhất — mỗi bước fail sẽ chặn bước sau, đảm bảo image lọt tới registry production đã qua đủ kiểm tra:

```yaml
# .github/workflows/build-sign.yml
jobs:
  build-sign-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write    # cần cho keyless cosign

    steps:
      - uses: actions/checkout@v4

      - name: Setup cosign
        uses: sigstore/cosign-installer@v3

      - name: Login to registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push image
        id: build-push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Trivy scan (fail on CRITICAL)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}@${{ steps.build-push.outputs.digest }}
          exit-code: 1
          severity: CRITICAL

      - name: Sign image (keyless)
        run: |
          cosign sign --yes \
            ghcr.io/${{ github.repository }}@${{ steps.build-push.outputs.digest }}

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: ghcr.io/${{ github.repository }}@${{ steps.build-push.outputs.digest }}
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Attach SBOM as attestation
        run: |
          cosign attest --yes \
            --predicate sbom.spdx.json \
            --type spdxjson \
            ghcr.io/${{ github.repository }}@${{ steps.build-push.outputs.digest }}
```

Pipeline này minh hoạ toàn bộ luồng supply chain đã học: build → scan (Trivy) → sign (Cosign) → sinh SBOM (sbom-action, dùng Syft bên trong) → attest (gắn SBOM đã ký vào image). Bước "attest" được nói rõ hơn ở phần SBOM bên dưới.

## Vulnerability Scanning

### CVE là gì

CVE (Common Vulnerabilities and Exposures) là hệ thống định danh chuẩn hoá cho các lỗ hổng đã biết — giúp tránh báo cáo trùng lặp, gán ID duy nhất cho từng lỗ hổng, cung cấp thông tin chi tiết để dev/sysadmin ưu tiên xử lý. CVE được phân làm hai nhóm chính:

1. Lỗ hổng cho phép **bypass security control** (ví dụ truy cập trái phép dữ liệu nhạy cảm).
2. Lỗ hổng làm **giảm hiệu năng / gây gián đoạn dịch vụ / mất ổn định** hệ thống.

Mỗi CVE có điểm **severity** từ 0–10 (theo CVSS v2/v3): điểm ≥ ~9 hoặc rating "critical" nghĩa là cần remediate ngay lập tức. Ví dụ CVE-2020-5911 (NGINX Controller installer tải package qua HTTP không mã hoá thay vì HTTPS) có severity 7.3 (High).

Khi phát hiện lỗ hổng trong package, có 3 hướng xử lý: nâng cấp lên version đã fix, áp thêm biện pháp bảo mật bổ sung, hoặc gỡ bỏ package không cần thiết. Nguyên tắc chung: **image càng ít package, attack surface càng nhỏ**.

### Trivy

Trivy (của Aqua Security) là vulnerability scanner đơn giản, mạnh, tích hợp tốt với CI/CD.

Cài đặt trên hệ Debian-based:

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

Scan một image (tên image y hệt như dùng trong `docker run`):

```bash
trivy image nginx:1.18.0
```

Output liệt kê từng CVE theo library, kèm severity, installed version, mô tả. Có thể lọc theo mức độ nghiêm trọng — rất hữu ích để gate CI/CD:

```bash
trivy image --severity CRITICAL nginx:1.18.0
trivy image --severity CRITICAL,HIGH nginx:1.18.0
trivy image --ignore-unfixed nginx:1.18.0
```

Scan image đã lưu dưới dạng tar (không cần pull lại từ registry):

```bash
docker save nginx:1.18.0 > nginx.tar
trivy image --input nginx.tar
```

**Best practice khi scan image:**

- Rescan định kỳ — image "sạch" hôm nay có thể phát sinh CVE mới sau này.
- Tích hợp scan vào Admission Controller để chặn deploy image chưa qua kiểm tra (lưu ý: có thể gây delay).
- Duy trì internal registry chỉ chứa image đã được scan/tin cậy để giảm overhead scan lặp lại.
- Đưa vulnerability scanning vào CI/CD pipeline để tự động phát hiện vấn đề ở mỗi build mới.

### KubeLinter

KubeLinter phân tích **manifest file** (không phải image) để tìm misconfiguration và enforce best practice, ví dụ phát hiện:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1                     # single replica -> không có redundancy
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app-container
        image: my-app-image:latest   # dùng tag "latest" -> discouraged
        ports:
        - containerPort: 8080
        # thiếu resource requests/limits, securityContext, liveness/readiness probe
```

Các check tiêu biểu: cảnh báo thiếu liveness/readiness probe, deployment chỉ có 1 replica, thiếu securityContext, cấm dùng tag `latest`, kiểm tra resource requests/limits. Toàn bộ rule đều **configurable** để enforce policy riêng của tổ chức.

Cài đặt và chạy cơ bản:

```bash
curl -Lo kube-linter.tar.gz \
  https://github.com/stackrox/kube-linter/releases/download/0.2.3/kube-linter-linux-amd64.tar.gz
tar -xzf kube-linter.tar.gz
sudo mv kube-linter /usr/local/bin/

cd path/to/k8s-configs
kube-linter lint .
```

Tích hợp CI/CD — luồng chuẩn: checkout code → lint bằng kube-linter → build image (nếu lint pass) → publish kết quả/gửi cảnh báo nếu có vấn đề.

Ví dụ Jenkins pipeline:

```groovy
stages {
    stage('Checkout') {
        steps { checkout scm }
    }
    stage('Install KubeLinter') {
        steps {
            sh 'curl -sSfL https://raw.githubusercontent.com/stackrox/kubelinter/main/scripts/install.sh | sh -'
        }
    }
    stage('Lint Kubernetes Manifests') {
        steps { sh 'kube-linter lint .' }
    }
}
```

Ví dụ GitHub Actions:

```yaml
name: Lint Kubernetes Manifests
on: [push]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run KubeLinter
        run: kube-linter lint .
```

## SBOM (Software Bill of Materials)

### SBOM là gì và tại sao quan trọng

SBOM là danh sách đầy đủ mọi thành phần cấu tạo nên một ứng dụng — giống "nhãn thành phần" của một món ăn: liệt kê library, dependency bên thứ ba, license, version, patch status. Khi xảy ra incident bảo mật, team có thể nhanh chóng xác định component nào bị ảnh hưởng và hành động (patch/thay thế) kịp thời.

Lợi ích chính:

- Minh bạch về thành phần software.
- Incident response nhanh hơn.
- Quản lý dependency hiệu quả (theo dõi interdependency, tránh dependency lỗi thời).
- Tăng cường bảo mật nhờ theo dõi lỗ hổng chi tiết theo từng component.
- Hỗ trợ compliance nhờ ghi nhận đầy đủ thông tin license.

### Hai format phổ biến: SPDX vs CycloneDX

| Tiêu chí | SPDX | CycloneDX |
|---|---|---|
| Trọng tâm | Licensing & legal compliance | Security & vulnerability tracking |
| Định dạng | JSON, RDF | JSON, XML |
| Độ phức tạp | Nhiều metadata hơn, phức tạp hơn | Gọn nhẹ, tập trung vào security |
| Dùng khi nào | Dự án open-source / doanh nghiệp cần trace nguồn gốc software, audit license, compliance pháp lý | Muốn tăng cường vulnerability management xuyên suốt lifecycle, đảm bảo software integrity |

**SPDX** chia thành các section: Document Information (metadata document), Relationships (quan hệ giữa các component), Package Information (tên, version, supplier, checksum...), Snippets (đoạn code trích từ thư viện open-source), File Information (thông tin từng file), và Additional Metadata (ghi chú, license, review record).

Ví dụ document header SPDX (JSON, sinh bởi Anchore/syft):

```json
{
  "spdxVersion": "SPDX-2.3",
  "dataLicense": "CC0-1.0",
  "SPDXID": "SPDXRef-DOCUMENT",
  "name": "nginx",
  "documentNamespace": "https://anchore.com/syft/image/nginx-2a35db70-da10-45cd-b82d-00921857780f",
  "creationInfo": {
    "licenseListVersion": "3.25",
    "creators": ["Organization: Anchore, Inc", "Tool: syft-1.13.0"]
  },
  "created": "2024-09-24T18:17:42Z"
}
```

Ví dụ package entry SPDX (package `grep`), có cả `externalRefs` để trace security/package-manager:

```json
{
  "package": {
    "name": "grep",
    "SPDXID": "SPDXRef-Package-deb-grep-a86139312d2f5a59d",
    "versionInfo": "3.8-5",
    "supplier": "Person: Anibal Monsalve Salazar (anibal@debian.org)",
    "licenseDeclared": "GPL-3.0-only AND GPL-3.0-or-later",
    "externalRefs": [
      {
        "referenceCategory": "SECURITY",
        "referenceType": "cpe23Type",
        "referenceLocator": "cpe:2.3:a:grep:grep:3.8-5:*****:*:*:*:*:*:*"
      },
      {
        "referenceCategory": "PACKAGE-MANAGER",
        "referenceType": "url",
        "referenceLocator": "pkg:deb/debian/grep@3.8-5?arch=amd64&distro=debian-12"
      }
    ]
  }
}
```

Khi không có thông tin xác định (ví dụ license), SPDX ghi `NOASSERTION` thay vì bỏ trống.

**CycloneDX** gồm: BOM Metadata (version, timestamp, tool tạo BOM), Components List, Vulnerabilities, Software Services, Annotations, Dependencies/Extensions.

Ví dụ CycloneDX BOM:

```json
{
  "$schema": "http://cyclonedx.org/schema/bom-1.6.schema.json",
  "bomFormat": "CycloneDX",
  "specVersion": "1.4",
  "serialNumber": "urn:uuid:e7f6caab-6589-430d-bb7f-0076d23e9efb",
  "version": 1,
  "metadata": {
    "timestamp": "2024-09-24T18:46:28Z",
    "tools": {
      "components": [
        { "type": "application", "author": "anchore", "name": "syft", "version": "1.13.0" }
      ]
    }
  },
  "component": {
    "bom-ref": "eb2d7db1213e6155",
    "type": "container",
    "name": "nginx",
    "version": "sha256:edf555d07d2ddeb6b616d9024442feac12a91310c9a156fa6f60cd602881a"
  },
  "components": [
    {
      "bom-ref": "pkg:deb/debian/adduser@3.134?arch=all&distro=debian-12&package-id=8a498975e59f569c2",
      "type": "library",
      "publisher": "Debian Adduser Developers <adduser@packages.debian.org>"
    }
  ]
}
```

### SBOM Workflow

Quy trình chuẩn gồm 6 bước: **Generate → Store → Scan → Analyze → Remediate → Monitor**.

1. **Generate** — dùng **Syft** để sinh SBOM từ image hoặc source code:

```bash
# Cài Syft
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh

# Sinh SBOM định dạng SPDX-JSON từ image
syft <image-name>:<tag> -o spdx-json

# Sinh SBOM từ source code local
syft /path/to/source/code -o spdx-json
```

2. **Store** — lưu SBOM ở repository an toàn (JFrog, Sonatype Nexus, GitHub Packages...).

3. **Scan** — dùng **Grype** để quét lỗ hổng dựa trên SBOM đã sinh:

```bash
# Cài Grype
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh

# Scan SBOM tìm lỗ hổng
grype sbom:nginx-sbom.cyclonedx.json
```

4. **Analyze** — xem chi tiết từng vulnerability trong kết quả scan, ví dụ:

```json
{
  "vulnerability": {
    "id": "CVE-2020-11724",
    "severity": "Medium",
    "links": ["http://security-tracker.debian.org/tracker/CVE-2020-11724"]
  },
  "cvss-v2": { "base-score": 5, "vector": "AV:N/AC:L/Au:N/C:N/I:P/A:N" },
  "matched-by": {
    "matcher": "dpkg-matcher",
    "search-key": "distro[debian 9] constraint[< 1.10.3-1+deb9u5 (deb)]"
  },
  "artifact": {
    "name": "libnginx-mod-http-xslt-filter",
    "version": "1.10.3-1+deb9u3",
    "type": "deb"
  }
}
```

5. **Remediate** — cập nhật package lên version an toàn hoặc thay thế; luôn test trong môi trường kiểm soát trước khi đưa vào production.

6. **Monitor** — thiết lập giám sát và cảnh báo liên tục trong CI/CD pipeline để tự động phát hiện dependency lỗi thời hoặc lỗ hổng mới phát sinh.

Chọn format theo mục đích: **SPDX** cho open-source/enterprise cần compliance license và trace nguồn gốc; **CycloneDX** khi ưu tiên vulnerability management xuyên suốt lifecycle.

### Gắn SBOM vào image với Cosign (attach & attest)

SBOM sinh ra ở bước Generate nên được **gắn liền với image** trên registry thay vì lưu tách rời — như vậy ai pull image cũng lấy được đúng SBOM tương ứng, và có thể verify SBOM đó chưa bị sửa. Cosign hỗ trợ hai cách gắn:

```bash
# Gắn SBOM vào image (không ký — chỉ attach)
cosign attach sbom --sbom sbom.spdx.json \
  ghcr.io/myorg/myapp@sha256:abc123...

# Attest SBOM — gắn kèm chữ ký (khuyến nghị hơn attach thường,
# vì attestation được ký nên không thể bị giả mạo sau khi gắn)
cosign attest \
  --predicate sbom.spdx.json \
  --type spdxjson \
  --key cosign.key \
  ghcr.io/myorg/myapp@sha256:abc123...
```

Để lấy lại và verify SBOM đã attest từ image (ví dụ ở một node khác, hoặc lúc audit):

```bash
cosign verify-attestation \
  --type spdxjson \
  --key cosign.pub \
  ghcr.io/myorg/myapp:v1.2.3 | \
  jq '.payload | @base64d | fromjson | .predicate'
```

`cosign verify-attestation` vừa verify chữ ký của attestation vừa trả về nội dung SBOM gốc (decode từ base64) — đảm bảo SBOM lấy được chính là SBOM đã được ký lúc build, không bị chỉnh sửa qua trung gian.

## Kyverno Image Verification

Có image đã ký chỉ có giá trị nếu cluster **thực sự verify chữ ký trước khi chạy** — nếu không, một image không ký hoặc ký sai vẫn được admission accept bình thường. **Kyverno** là policy engine phổ biến để enforce việc này ngay tại admission, thông qua rule `verifyImages`: pod dùng image chưa được ký (hoặc ký bởi identity/khoá không khớp) sẽ bị reject.

### Basic Signature Verification (keyless)

```yaml
# ClusterPolicy — verify tất cả images phải được signed
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  background: false    # chỉ check lúc admission (không audit resource đã tồn tại)
  rules:
    - name: verify-signature
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces:
                - production
                - staging
      verifyImages:
        - imageReferences:
            - "ghcr.io/myorg/*"
          attestors:
            - count: 1       # cần ít nhất 1 attestor pass
              entries:
                - keyless:
                    subject: "https://github.com/myorg/myapp/.github/workflows/build-sign.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
                    rekor:
                      url: https://rekor.sigstore.dev
```

`subject` và `issuer` chính là claim trong OIDC token dùng lúc ký keyless (workflow path + identity provider) — Kyverno đối chiếu ngược với Rekor để xác nhận chữ ký hợp lệ và đúng identity mong muốn (tránh trường hợp ai đó ký bằng một GitHub Actions workflow khác không được tin cậy).

### Key-based Verification

```yaml
# Verify bằng public key thay vì keyless/OIDC
verifyImages:
  - imageReferences:
      - "ghcr.io/myorg/*"
    attestors:
      - count: 1
        entries:
          - keys:
              publicKeys: |-
                -----BEGIN PUBLIC KEY-----
                MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
                -----END PUBLIC KEY-----
              rekor:
                url: https://rekor.sigstore.dev
```

### Verify SBOM Attestation

Ngoài verify chữ ký image, Kyverno cũng có thể **bắt buộc image phải có SBOM attestation đính kèm** (đã ký) trước khi được chấp nhận — hữu ích khi tổ chức yêu cầu mọi image production đều truy vết được thành phần cấu tạo:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-sbom
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-sbom
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      verifyImages:
        - imageReferences:
            - "ghcr.io/myorg/*"
          attestations:
            - predicateType: https://spdx.dev/Document
              attestors:
                - entries:
                    - keyless:
                        subject: "https://github.com/myorg/*"
                        issuer: "https://token.actions.githubusercontent.com"
```

### Mutate Image Tag sang Digest

Một rủi ro khác: pod spec dùng **tag** (`:v1.2.3`) thay vì **digest** (`@sha256:...`) — tag có thể bị overwrite sau khi verify (time-of-check-to-time-of-use), trong khi digest là immutable. Kyverno có thể tự động mutate tag thành digest ngay tại admission để đảm bảo image chạy đúng là image đã verify:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: resolve-image-to-digest
spec:
  rules:
    - name: resolve-tag-to-digest
      match:
        any:
          - resources:
              kinds: [Pod]
      mutate:
        foreach:
          - list: "request.object.spec.containers"
            patchStrategicMerge:
              spec:
                containers:
                  - name: "{{ element.name }}"
                    image: "{{ element.image | image_normalize(@) }}"
                    # image_normalize resolve tag → sha256 digest
```

## Registry Allowlisting / Image Policy Webhook

Mặc định, bất kỳ user nào có quyền tạo pod trong cluster đều có thể chỉ định image từ **bất kỳ registry nào**, kể cả nguồn không tin cậy:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-pod
spec:
  containers:
    - name: sample-app
      image: some-registry.io/a-very-vulnerable-image
```

Đây là rủi ro thực sự: image chứa lỗ hổng có thể bị khai thác để đạt quyền truy cập vào OS bên dưới, ảnh hưởng tới các app khác trong cluster. Cần enforce chỉ cho phép image từ registry đã được duyệt.

### Cách 1: Custom validating admission webhook

Request tạo pod đi qua các bước: authentication → authorization → admission control. Một validating webhook server tự viết có thể kiểm tra field `image` và reject request nếu không khớp registry cho phép. Ví dụ webhook viết bằng Python (Flask):

```python
@app.route("/validate", methods=["POST"])
def validate():
    image_name = request.json["request"]["object"]["spec"]["containers"][0]["image"]
    status = True
    message = ""
    if "internal-registry.io" not in image_name:
        message = "You can only use images from the internal-registry.io"
        status = False
    return jsonify({
        "response": {
            "allowed": status,
            "uid": request.json["request"]["uid"],
            "status": {"message": message},
        }
    })
```

Webhook server này cần được deploy **highly available** — nếu không reachable, việc tạo pod trong cluster có thể bị chặn hoàn toàn (tuỳ cấu hình fail policy).

### Cách 2: Open Policy Agent (OPA) + Rego

OPA cho phép viết policy bằng ngôn ngữ Rego, deploy như một validating webhook. Ví dụ chặn mọi image không bắt đầu bằng `internal-registry.io/`:

```rego
package kubernetes.admission

deny[msg] {
    input.request.kind.kind == "Pod"
    image := input.request.object.spec.containers[_].image
    not startswith(image, "internal-registry.io/")
    msg := sprintf("Image '%s' is not from a trusted registry", [image])
}
```

### Cách 3: Built-in ImagePolicyWebhook admission controller

Kubernetes có sẵn admission controller tên **ImagePolicyWebhook**, hoạt động cùng một external webhook server, cấu hình qua **AdmissionConfiguration** file.

File admission configuration:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: <path-to-kubeconfig-file>
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: true
```

`kubeConfigFile` trỏ tới một kubeconfig cấu hình endpoint của webhook server:

```yaml
# <path-to-kubeconfig-file>
clusters:
- name: name-of-remote-imagepolicy-service
  cluster:
    certificate-authority: /path/to/ca.pem
    server: https://images.example.com/policy
users:
- name: name-of-api-server
  user:
    client-certificate: /path/to/cert.pem
    client-key: /path/to/key.pem
```

Cần **bật tường minh** plugin này trên kube-apiserver bằng `--enable-admission-plugins=ImagePolicyWebhook`, và trỏ tới file AdmissionConfiguration bằng `--admission-control-config-file`:

```bash
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=${INTERNAL_IP} \
  --allow-privileged=true \
  --apiserver-count=3 \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --enable-swagger-ui=true \
  --etcd-servers=https://127.0.0.1:2379 \
  --event-ttl=1h \
  --runtime-config=api/all \
  --service-cluster-ip-range=10.32.0.0/24 \
  --service-node-port-range=30000-32767 \
  --v=2 \
  --enable-admission-plugins=ImagePolicyWebhook \
  --admission-control-config-file=/etc/kubernetes/admission-config.yaml
```

Nếu kube-apiserver chạy dạng static pod (kubeadm), thêm flag tương tự vào manifest pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
    - --advertise-address=172.17.0.107
    - --allow-privileged=true
    - --enable-bootstrap-token-auth=true
    - --enable-admission-plugins=ImagePolicyWebhook
    image: k8s.gcr.io/kube-apiserver-amd64:v1.13.3
    name: kube-apiserver
```

### Cách 4: Kyverno

Ngoài custom webhook, OPA và ImagePolicyWebhook built-in, **Kyverno** cũng có thể enforce registry allowlist bằng một `ClusterPolicy` khai báo (không cần viết code hay Rego), phù hợp khi cluster đã có sẵn Kyverno để verify image signature (xem phần "Kyverno Image Verification" ở trên) — dùng chung một policy engine cho cả ký/verify lẫn allowlist:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: check-registry
      match:
        any:
          - resources:
              kinds: [Pod]
      exclude:
        any:
          - resources:
              namespaces:
                - kube-system
      validate:
        message: "Images must come from approved registries."
        foreach:
          - list: "request.object.spec.containers"
            deny:
              conditions:
                all:
                  - key: "{{ element.image }}"
                    operator: AnyNotIn
                    value:
                      - "ghcr.io/myorg/*"
                      - "harbor.internal/*"
                      - "*.amazonaws.com/*"    # ECR
```

`background: true` nghĩa là Kyverno còn audit lại các resource **đã tồn tại** trong cluster (không chỉ check tại thời điểm admission), giúp phát hiện pod nào đang chạy image ngoài allowlist mà lọt qua trước khi policy được áp dụng.

## ImagePullPolicy và Immutability

Allowlist registry hay verify signature đều vô nghĩa nếu **image tag không cố định** — vì tag có thể bị đẩy đè (re-push) sau khi đã pass mọi kiểm tra, khiến node sau đó pull về một image khác hẳn dù tên/tag không đổi.

```yaml
# Production: luôn dùng digest hoặc imagePullPolicy: Always + versioned tag
spec:
  containers:
    - name: app
      image: ghcr.io/myorg/myapp@sha256:abc123...   # tốt nhất: digest reference
      # hoặc
      image: ghcr.io/myorg/myapp:v1.2.3             # semver tag cụ thể
      imagePullPolicy: Always                        # không dùng lại image cũ đã cache trên node

# Tránh:
      image: myapp:latest     # :latest + IfNotPresent = không bao giờ pull bản mới
      imagePullPolicy: IfNotPresent
```

- **Digest reference** (`@sha256:...`) là immutable tuyệt đối — chính là nội dung image đã được scan/sign, không thể bị thay đổi ngầm.
- **`imagePullPolicy: Always`** buộc kubelet luôn hỏi lại registry (dù node đã có image cùng tag trong cache) — cần thiết khi tag có thể bị overwrite, đảm bảo lấy đúng version mới nhất/đã verify.
- Tag `:latest` kết hợp `IfNotPresent` là tổ hợp rủi ro nhất: node có thể chạy mãi một bản image cũ đã lỗi thời/có lỗ hổng vì không bao giờ pull lại.

Kyverno cũng có thể chặn hẳn việc dùng tag `:latest` (hoặc không khai báo tag) ngay tại admission:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-image-tag
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production, staging]
      validate:
        message: "Image tag ':latest' is not allowed. Use a specific version."
        foreach:
          - list: "request.object.spec.containers"
            deny:
              conditions:
                any:
                  - key: "{{ element.image }}"
                    operator: Equals
                    value: "*:latest"
                  - key: "{{ element.image }}"
                    operator: NotContains
                    value: ":"    # không có tag = mặc định latest
```

## Rekor Transparency Log

Cosign keyless signing (xem phần "Image Signing với Cosign") không lưu private key ở đâu cả — vậy điều gì đảm bảo một chữ ký là hợp lệ và không bị giả mạo sau này? Câu trả lời là **Rekor**: một log công khai, append-only (không thể sửa/xoá) ghi lại mọi sự kiện ký. Mỗi lần `cosign sign --yes` (keyless) chạy, một entry mới được ghi vào Rekor kèm certificate ngắn hạn (do Fulcio cấp) và digest của image — tạo thành audit trail độc lập, ai cũng verify được mà không cần tin tưởng riêng vào bên ký.

```bash
# Tìm log-upload entry ứng với một image
cosign search tlog-upload --rekor-url https://rekor.sigstore.dev \
  ghcr.io/myorg/myapp@sha256:abc123...

# Lấy chi tiết một entry trong Rekor theo UUID
rekor-cli get --rekor_server https://rekor.sigstore.dev \
  --uuid <log-entry-uuid>
```

Attestor `keyless` của Kyverno (phần "Kyverno Image Verification" ở trên) cũng ngầm tra cứu Rekor (`rekor.url`) để xác nhận chữ ký thực sự nằm trong transparency log trước khi accept — đây chính là cơ chế chống giả mạo cốt lõi của toàn bộ mô hình keyless signing.

## Gotchas

- **ImagePolicyWebhook không bật sẵn** — nó phải được thêm tường minh vào `--enable-admission-plugins`, và **bắt buộc** đi kèm `--admission-control-config-file` trỏ tới file `AdmissionConfiguration` (không phải file webhook config kiểu cũ `--admission-control-config-file` alone-format). Thiếu file config này, apiserver sẽ không khởi động được.
- `defaultAllow: true` trong AdmissionConfiguration nghĩa là **nếu webhook server không reachable, pod vẫn được tạo** (fail-open) — cần cân nhắc kỹ theo yêu cầu bảo mật; `false` là fail-closed, an toàn hơn nhưng có thể chặn toàn bộ việc tạo pod nếu webhook down.
- Webhook server (dù tự viết hay dùng OPA) phải **highly available** — nó nằm trên đường đi của mọi request tạo pod.
- `image: nginx` không có registry/namespace tường minh = `docker.io/library/nginx`. Đề thi hay test khả năng nhận diện shorthand này.
- **Distroless / scratch-based image**: giảm attack surface tối đa (không shell, không package manager, không network tool) nhưng đánh đổi là **rất khó debug** trực tiếp trong container (không có `sh`, `curl`, `ps`...) — cần công cụ debug ngoài như `kubectl debug` với ephemeral container.
- **Trivy severity filter** dùng đúng cú pháp `--severity CRITICAL,HIGH` (phân tách bằng dấu phẩy, không có khoảng trắng) để gate CI/CD chỉ fail khi có lỗ hổng nghiêm trọng; `--ignore-unfixed` bỏ qua các CVE chưa có bản vá (tránh false-positive block khi vendor chưa release fix).
- Trivy có thể scan trực tiếp image trong registry (`trivy image <name>`) hoặc scan file tar đã export (`trivy image --input file.tar`) — hữu ích khi image chưa được push lên registry nào (air-gapped/offline scan).
- **KubeLinter lint manifest YAML**, không scan image — dễ nhầm với Trivy (scan image tìm CVE trong package) hoặc kube-bench (audit cấu hình cluster theo CIS Benchmark). Ba công cụ có phạm vi khác nhau, hay bị hỏi phân biệt trong đề thi.
- **SPDX vs CycloneDX**: SPDX thiên về licensing/compliance (nhiều metadata hơn, JSON/RDF); CycloneDX thiên về security/vulnerability tracking (gọn nhẹ hơn, JSON/XML). Cả hai đều có thể sinh bởi Syft.
- SBOM workflow chuẩn: **Generate (Syft) → Store → Scan (Grype) → Analyze → Remediate → Monitor** — nhớ Syft dùng để tạo SBOM, Grype dùng để scan SBOM đã tạo (khác vai trò, hay bị nhầm với nhau hoặc với Trivy).
- Field `NOASSERTION` trong SPDX không phải là lỗi — nó là giá trị hợp lệ khi công cụ generate SBOM không xác định được thông tin đó (ví dụ license, copyright).
- Khi tạo `imagePullSecrets` để pull từ private registry, secret phải là loại **`docker-registry`** cụ thể (không phải `generic` hay `Opaque`), và phải được tham chiếu đúng tên trong `spec.imagePullSecrets` của pod (không phải ở container spec).
- **Cosign ký digest, không ký tag**: tag có thể mutable (bị overwrite). Luôn ký `image@sha256:...` chứ không phải `image:tag` — lấy digest từ output của `docker buildx`/`docker inspect` hoặc từ output bước build trong CI.
- **Kyverno `verifyImages` mặc định chỉ check `containers[]`** — cần thêm rule riêng cho `initContainers[]` và `ephemeralContainers[]` nếu muốn cover toàn bộ pod spec, nếu không attacker có thể lách qua init/ephemeral container.
- **Registry mirror (pull-through cache) không tự động mirror chữ ký**: nếu dùng private registry mirror, signature ở upstream registry không tự theo về mirror — cần setup signature discovery endpoint riêng hoặc mirror signature thủ công, nếu không verify sẽ fail dù image đúng.
- **Keyless signing (OIDC) yêu cầu subject URL khớp chính xác** đường dẫn workflow (ví dụ `.github/workflows/build-sign.yml@refs/heads/main`) — đổi tên/đường dẫn workflow file sẽ khiến verify fail cho cả các image cũ đã ký trước đó.
- **SBOM của image lớn (nhiều dependency, ví dụ Java) có thể nặng** (5–10MB) — nên attach dưới dạng OCI artifact riêng (`cosign attach sbom`/`cosign attest`) thay vì nhúng inline annotation.
- **`imagePullPolicy: Always` cộng dồn với rate limit của registry**: pull lại ở mọi lần pod restart; Docker Hub giới hạn ~100 pulls/6h cho request unauthenticated — cần cấu hình registry credentials hoặc dùng private registry để tránh bị throttle khi cluster restart nhiều pod cùng lúc.
