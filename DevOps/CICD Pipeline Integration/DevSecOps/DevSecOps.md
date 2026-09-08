---
title: DevSecOps
tags:
  - devsecops
  - security
  - cicd
  - deep-dive
date: 2026-04-26
---

# DevSecOps

## What is DevSecOps?

**DevSecOps = Shift Security Left** — tích hợp security vào mọi giai đoạn của pipeline thay vì chỉ kiểm tra ở cuối.

```
Old way (Security at the end):
Code → Build → Test → Deploy → SECURITY CHECK ← too late, too expensive

DevSecOps (Security everywhere):
Code   → Secret Scan + SAST + IaC Scan
Build  → SCA (dependency scan)
Package→ Container Image Scan
Deploy → Config compliance, RBAC audit
Runtime→ Runtime security (Falco), Network policy
```

**SAST vs DAST vs SCA:**

| | SAST | DAST | SCA |
|---|---|---|---|
| Full name | Static Application Security Testing | Dynamic Application Security Testing | Software Composition Analysis |
| When | Compile time / pre-build | Runtime (running app) | Pre-build (dependencies) |
| Scans | Source code | HTTP requests/responses | Third-party libraries |
| Finds | Logic flaws, insecure patterns | XSS, SQLi (behavior) | Known CVEs in deps |
| Tools | Semgrep, CodeQL, SonarQube | OWASP ZAP, Burp Suite | Snyk, Dependabot, OWASP DC |

---

## Secret Scanning

Phát hiện secrets (API keys, passwords, private keys) bị commit vào git.

### Gitleaks

```yaml
# .github/workflows/ci.yml
- name: Run Gitleaks
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}   # for org-level
```

```bash
# Local scan
brew install gitleaks

# Scan toàn bộ git history
gitleaks detect --source . --verbose

# Scan chỉ staged files (pre-commit hook)
gitleaks protect --staged -v

# Scan specific commit range
gitleaks detect --log-opts "HEAD~5..HEAD"
```

**.gitleaks.toml** (custom rules + allowlist):
```toml
[extend]
useDefault = true                  # dùng default rules

[[rules]]
id = "custom-internal-token"
description = "Internal API token"
regex = '''INT-[0-9A-Z]{32}'''
tags = ["internal", "token"]

[allowlist]
commits = [
  "abc123def",                     # known false positive commit
]
paths = [
  "test/fixtures/",
  "docs/examples/",
]
regexes = [
  '''example_api_key''',           # known example string
  '''AKIA[0-9A-Z]{16}''',         # AWS key in tests
]
```

### TruffleHog

```bash
# Scan git repo (tất cả history)
trufflehog git file://. --only-verified

# Scan GitHub repo
trufflehog github --org=mycompany

# Scan Docker image
trufflehog docker --image myapp:latest
```

### Pre-commit hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks

  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

```bash
pip install pre-commit
pre-commit install          # cài hooks vào .git/hooks/
pre-commit run --all-files  # chạy thủ công
```

---

## SAST — Static Analysis

### Semgrep

```yaml
# .github/workflows/sast.yml
- name: Run Semgrep
  uses: semgrep/semgrep-action@v1
  with:
    config: >
      p/owasp-top-ten
      p/cwe-top-25
      p/nodejs
      p/python
      p/docker
  env:
    SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}
```

```bash
# Local
pip install semgrep

# Scan với OWASP rules
semgrep --config p/owasp-top-ten ./src

# Custom rule
semgrep --config custom-rules.yml ./src
```

```yaml
# custom-rules.yml — ví dụ rule phát hiện hardcoded credentials
rules:
  - id: hardcoded-password
    patterns:
      - pattern: |
          $X = "..."
      - metavariable-regex:
          metavariable: $X
          regex: (password|passwd|pwd|secret|token|key)
    message: "Possible hardcoded credential in variable $X"
    severity: ERROR
    languages: [python, javascript, typescript]
```

### CodeQL (GitHub)

```yaml
# .github/workflows/codeql.yml
name: CodeQL Analysis

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'           # mỗi thứ 2 lúc 6am

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read

    strategy:
      matrix:
        language: [javascript, python]   # hoặc go, java, cpp, csharp

    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
          queries: +security-extended    # thêm extended security queries

      - name: Autobuild
        uses: github/codeql-action/autobuild@v3

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: /language:${{ matrix.language }}
```

### SonarQube

```yaml
# .github/workflows/sonar.yml
- name: SonarQube Scan
  uses: SonarSource/sonarqube-scan-action@master
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
```

```properties
# sonar-project.properties
sonar.projectKey=company:myapp
sonar.projectName=My Application
sonar.sources=src
sonar.tests=tests
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.coverage.exclusions=**/*.test.js,**/node_modules/**

# Quality Gate: fail pipeline nếu không đạt
sonar.qualitygate.wait=true
```

---

## SCA — Dependency Scanning

### Dependabot (GitHub)

```yaml
# .github/dependabot.yml
version: 2
updates:
  # NPM
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
      day: monday
      time: "09:00"
    open-pull-requests-limit: 10
    reviewers:
      - security-team
    labels:
      - dependencies
      - security
    ignore:
      - dependency-name: "lodash"
        versions: ["4.x"]            # ignore specific version
    groups:
      dev-dependencies:
        dependency-type: development
        update-types:
          - minor
          - patch

  # Docker base images
  - package-ecosystem: docker
    directory: /
    schedule:
      interval: weekly

  # GitHub Actions
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

### Snyk

```yaml
# .github/workflows/snyk.yml
- name: Run Snyk
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high --fail-on=all

# Container scan
- name: Snyk container scan
  uses: snyk/actions/docker@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    image: myapp:${{ github.sha }}
    args: --file=Dockerfile --severity-threshold=critical
```

```bash
# Local
npm install -g snyk
snyk auth
snyk test                          # test current project
snyk test --severity-threshold=high
snyk monitor                       # continuous monitoring
snyk container test myapp:latest
```

### OWASP Dependency-Check

```yaml
- name: OWASP Dependency Check
  uses: dependency-check/Dependency-Check_Action@main
  with:
    project: 'myapp'
    path: '.'
    format: 'HTML'
    args: >
      --enableRetired
      --failOnCVSS 7           # fail nếu CVSS score >= 7
      --suppression suppression.xml  # false positives
```

---

## Container Image Scanning

### Trivy trong CI

```yaml
# .github/workflows/trivy.yml
- name: Build image
  run: docker build -t myapp:${{ github.sha }} .

- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: sarif
    output: trivy-results.sarif
    severity: HIGH,CRITICAL
    exit-code: 1              # fail pipeline nếu có
    ignore-unfixed: true      # bỏ qua CVE chưa có fix

- name: Upload Trivy results to GitHub Security
  uses: github/codeql-action/upload-sarif@v3
  if: always()
  with:
    sarif_file: trivy-results.sarif

# Scan Dockerfile misconfigurations
- name: Trivy config scan
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: config
    scan-ref: .
    exit-code: 1
    severity: HIGH,CRITICAL
```

**.trivyignore:**
```
# Ignore known false positive / accepted risk
CVE-2023-45853   # zlib - mitigated by network policy (no external exposure)
CVE-2024-12345   # tracked in JIRA SEC-456, patch available in base image next release
```

### GitLab built-in Container Scanning

```yaml
# .gitlab-ci.yml
include:
  - template: Security/Container-Scanning.gitlab-ci.yml

container_scanning:
  variables:
    CS_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
    CS_SEVERITY_THRESHOLD: HIGH
    TRIVY_TIMEOUT: 5m
```

---

## IaC Security Scanning

### Checkov (Terraform, K8s, Dockerfile)

```bash
# Install
pip install checkov

# Scan Terraform
checkov -d ./terraform/ --framework terraform

# Scan Kubernetes manifests
checkov -d ./k8s/ --framework kubernetes

# Scan Dockerfile
checkov -f Dockerfile --framework dockerfile

# Fail on specific severity
checkov -d . --soft-fail-on LOW,MEDIUM    # pass cho LOW/MEDIUM, fail cho HIGH/CRITICAL
```

```yaml
# .github/workflows/iac-scan.yml
- name: Run Checkov
  uses: bridgecrewio/checkov-action@master
  with:
    directory: terraform/
    framework: terraform
    soft_fail: false
    output_format: sarif
    output_file_path: checkov.sarif

- name: Upload Checkov results
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: checkov.sarif
```

### tfsec / Trivy (Terraform)

```bash
# tfsec
tfsec .
tfsec . --minimum-severity HIGH

# Trivy IaC scan
trivy config ./terraform/
trivy config ./k8s/
```

---

## Full DevSecOps Pipeline

```yaml
# .github/workflows/devsecops.yml — complete security pipeline
name: DevSecOps Pipeline

on:
  push:
    branches: [main]
  pull_request:

jobs:
  # 1. Secret detection (fail fast)
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # 2. SAST
  sast:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - name: Semgrep
        uses: semgrep/semgrep-action@v1
        with:
          config: p/owasp-top-ten p/nodejs
      - name: CodeQL
        uses: github/codeql-action/analyze@v3

  # 3. SCA
  sca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Snyk test
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  # 4. Build & container scan
  container-security:
    needs: [secret-scan, sast, sca]
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: HIGH,CRITICAL
          exit-code: 1

      - name: Upload Trivy SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif

  # 5. IaC scan
  iac-security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Checkov IaC scan
        uses: bridgecrewio/checkov-action@master
        with:
          directory: k8s/
          framework: kubernetes
          soft_fail: false

  # 6. Deploy (chỉ khi tất cả security checks pass)
  deploy:
    needs: [container-security, iac-security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to staging
        run: echo "Deploy!"
```

---

## Security Gates & Policies

### Branch Protection + Required Checks

```
GitHub → Settings → Branches → Branch protection rules

main branch:
  ✓ Require pull request reviews (2 approvals)
  ✓ Require status checks to pass:
    - secret-scan
    - sast
    - sca
    - container-security
  ✓ Require branches to be up to date
  ✓ Restrict who can push to matching branches
```

### SBOM — Software Bill of Materials

```bash
# Generate SBOM với Syft
syft myapp:latest -o spdx-json > sbom.spdx.json
syft myapp:latest -o cyclonedx-json > sbom.cyclonedx.json

# Attach SBOM tới image (OCI artifact)
cosign attach sbom --sbom sbom.spdx.json myapp:latest

# Verify SBOM
cosign download sbom myapp:latest
```

```yaml
# Trong CI — generate SBOM sau build
- name: Generate SBOM
  uses: anchore/sbom-action@v0
  with:
    image: myapp:${{ github.sha }}
    format: spdx-json
    output-file: sbom.spdx.json

- name: Upload SBOM
  uses: actions/upload-artifact@v4
  with:
    name: sbom
    path: sbom.spdx.json
```

---

## Runtime Security — Falco

Falco detect abnormal behavior tại runtime dựa trên syscall rules.

```yaml
# falco-rules.yaml (custom rules)
- rule: Unexpected outbound connection from container
  desc: Detect outbound connections to unexpected IPs
  condition: >
    outbound and container and
    not proc.name in (curl, wget, apt-get) and
    not fd.sip in (allowed_ips)
  output: >
    Unexpected outbound connection (user=%user.name container=%container.name
    image=%container.image.repository ip=%fd.rip port=%fd.rport)
  priority: WARNING

- rule: Shell spawned in container
  desc: Detect shell execution in container
  condition: >
    spawned_process and container and
    proc.name in (bash, sh, zsh, fish) and
    not container.image.repository contains "debug"
  output: >
    Shell spawned in container (user=%user.name container=%container.name
    image=%container.image.repository shell=%proc.name parent=%proc.pname)
  priority: CRITICAL
```

---

## Gotchas

- **False positives**: SAST/secret scan báo false positive nhiều → developers disable hoặc ignore → miss real issues. Tune rules, maintain allowlists, đo false positive rate.
- **Scan time và pipeline speed**: tất cả security scans song song (không serial) để giảm pipeline time. Secret scan < 1 phút, SAST ~2-5 phút, image scan ~3-5 phút.
- **Vulnerability triage**: Trivy báo 100 CVEs không có nghĩa là 100 risk — phần lớn là informational hoặc không exploitable trong context. Build triage process: severity + exploitability + exposure.
- **`.trivyignore` drift**: ignore list cũ → accepted CVEs vẫn được ignore dù patch đã available. Review và clean `.trivyignore` định kỳ (quarterly).
- **SCA và transitive deps**: direct deps an toàn không có nghĩa là transitive deps an toàn. Snyk/Dependabot scan cả transitive — nhưng fixing transitive dep cần careful testing.
- **Container scan vs runtime scan**: image scan (CI) phát hiện known CVEs trong packages. Runtime threats (privilege escalation, crypto mining) cần Falco/runtime security. Cả hai đều cần.
- **SBOM và supply chain**: có SBOM không đủ — cần verify integrity (cosign sign SBOM), store securely, query khi CVE mới xuất hiện để biết images nào affected.
