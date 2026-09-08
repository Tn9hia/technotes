---
title: GitHub Actions
tags:
  - cicd
  - github-actions
  - deep-dive
date: 2026-04-26
---

# GitHub Actions

## Architecture

```
GitHub Repository
    │
    ├── .github/
    │   └── workflows/
    │       ├── ci.yml          ← triggered on push/PR
    │       ├── release.yml     ← triggered on tag
    │       └── nightly.yml     ← scheduled
    │
Workflow (yml file)
    │
    └── Job 1 (chạy trên runner)
    │       └── Step 1: actions/checkout
    │       └── Step 2: run: npm test
    │       └── Step 3: docker build
    │
    └── Job 2 (chạy sau Job 1)
            └── Step 1: deploy
```

**Concepts:**
- **Workflow**: YAML file trong `.github/workflows/` — định nghĩa toàn bộ pipeline
- **Event/Trigger**: cái gì khởi động workflow (push, pull_request, schedule, ...)
- **Job**: group of steps chạy trên 1 runner (có thể parallel với jobs khác)
- **Step**: 1 task trong job (chạy action hoặc shell command)
- **Action**: reusable unit (`uses: actions/checkout@v4`)
- **Runner**: máy chạy jobs (GitHub-hosted hoặc self-hosted)

---

## Workflow Syntax

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

# ─── Triggers ───
on:
  push:
    branches: [main, develop]
    paths:
      - 'src/**'
      - 'Dockerfile'
      - 'package*.json'
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
  schedule:
    - cron: '0 2 * * *'          # mỗi ngày 2am UTC
  workflow_dispatch:              # manual trigger
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]

# ─── Environment variables (tất cả jobs) ───
env:
  REGISTRY: registry.internal
  IMAGE_NAME: platform/myapp

# ─── Jobs ───
jobs:
  # ─── Job 1: Build & Test ───
  build-test:
    name: Build & Test
    runs-on: ubuntu-latest         # GitHub-hosted runner
    # runs-on: self-hosted          # self-hosted runner

    # Job-level env
    env:
      NODE_ENV: test

    # Services (sidecar containers)
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    # Permissions (least privilege)
    permissions:
      contents: read
      packages: write

    steps:
      # Checkout code
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0            # full history (for semantic versioning)

      # Setup language runtime
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'             # auto-cache node_modules

      # Cache (manual control)
      - name: Cache dependencies
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      # Install & test
      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm run test:ci
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true

  # ─── Job 2: Security Scan ───
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: []                      # chạy parallel với build-test
    permissions:
      security-events: write       # upload SARIF to GitHub Security tab

    steps:
      - uses: actions/checkout@v4

      - name: Run Semgrep
        uses: semgrep/semgrep-action@v1
        with:
          config: p/owasp-top-ten

      - name: Run Gitleaks (secret scan)
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # ─── Job 3: Build & Push Docker image ───
  docker:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [build-test, security]  # chạy sau khi cả 2 pass
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      # Setup BuildKit
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # Login to registry
      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ secrets.REGISTRY_USER }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      # Generate tags & labels
      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-,format=short    # sha-abc1234
            type=ref,event=branch                # main
            type=semver,pattern={{version}}      # v1.2.3 (từ tag)
            type=semver,pattern={{major}}.{{minor}}

      # Build & push
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}  # không push trong PR
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha                 # cache từ GitHub Actions cache
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64   # multi-arch

      # Scan image after build
      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: HIGH,CRITICAL
          exit-code: 1

      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif

  # ─── Job 4: Deploy Staging ───
  deploy-staging:
    name: Deploy Staging
    runs-on: ubuntu-latest
    needs: docker
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://staging.myapp.internal

    steps:
      - name: Update manifest repo
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.MANIFEST_REPO_TOKEN }}
          script: |
            const { data } = await github.rest.repos.getContent({
              owner: 'company',
              repo: 'k8s-manifests',
              path: 'apps/myapp/staging/values.yaml'
            })
            const content = Buffer.from(data.content, 'base64').toString()
            const newContent = content.replace(
              /tag: .*/,
              `tag: sha-${context.sha.slice(0, 7)}`
            )
            await github.rest.repos.createOrUpdateFileContents({
              owner: 'company',
              repo: 'k8s-manifests',
              path: 'apps/myapp/staging/values.yaml',
              message: `chore: update myapp to sha-${context.sha.slice(0, 7)}`,
              content: Buffer.from(newContent).toString('base64'),
              sha: data.sha
            })

  # ─── Job 5: Deploy Production (manual approval) ───
  deploy-production:
    name: Deploy Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment:
      name: production               # requires approval in GitHub settings
      url: https://myapp.internal

    steps:
      - name: Update production manifests
        run: |
          # Clone manifest repo
          git clone https://x-access-token:${{ secrets.MANIFEST_REPO_TOKEN }}@github.com/company/k8s-manifests.git
          cd k8s-manifests
          # Update image tag
          sed -i "s|tag: .*|tag: sha-${{ github.sha }}|" apps/myapp/production/values.yaml
          git config user.email "ci@company.com"
          git config user.name "CI Bot"
          git commit -am "chore: promote myapp sha-${{ github.sha }} to production"
          git push
```

---

## Secrets & Variables

```yaml
# 3 loại secret trong GitHub Actions

# 1. Repository secrets (Settings → Secrets)
${{ secrets.REGISTRY_PASSWORD }}

# 2. Environment secrets (chỉ available khi job dùng environment đó)
environment:
  name: production
# → ${{ secrets.PROD_DB_URL }} chỉ available trong job này

# 3. Organization secrets (shared across repos)
${{ secrets.ORG_SLACK_WEBHOOK }}

# Variables (non-secret, public)
${{ vars.REGISTRY_URL }}         # Settings → Variables
```

**GITHUB_TOKEN** — auto-generated token, không cần tạo:
```yaml
permissions:
  contents: read       # checkout code
  packages: write      # push to GHCR
  pull-requests: write # comment on PR
  security-events: write # upload SARIF
```

---

## Matrix Strategy

Build/test trên nhiều OS/versions cùng lúc:

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20, 22]
        exclude:
          - os: windows-latest
            node: 18
        include:
          - os: ubuntu-latest
            node: 20
            experimental: false
      fail-fast: false          # không cancel jobs khác khi 1 fail
      max-parallel: 6

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
```

---

## Reusable Workflows

Tránh duplicate workflow code:

```yaml
# .github/workflows/reusable-docker-build.yml
on:
  workflow_call:                   # callable từ workflow khác
    inputs:
      image-name:
        required: true
        type: string
      push:
        required: false
        type: boolean
        default: true
    secrets:
      registry-password:
        required: true
    outputs:
      image-digest:
        value: ${{ jobs.build.outputs.digest }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        id: build
        with:
          tags: ${{ inputs.image-name }}
          push: ${{ inputs.push }}
```

```yaml
# .github/workflows/ci.yml — gọi reusable workflow
jobs:
  build-image:
    uses: ./.github/workflows/reusable-docker-build.yml
    with:
      image-name: registry.internal/myapp:${{ github.sha }}
    secrets:
      registry-password: ${{ secrets.REGISTRY_PASSWORD }}
```

---

## Composite Actions

Custom action tái sử dụng trong repo:

```yaml
# .github/actions/setup-app/action.yml
name: Setup Application
description: Install dependencies and setup environment

inputs:
  node-version:
    description: Node.js version
    default: '20'

runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: npm
    - run: npm ci
      shell: bash
    - run: npm run db:migrate
      shell: bash
      env:
        DATABASE_URL: ${{ env.DATABASE_URL }}
```

```yaml
# Dùng trong workflow
- uses: ./.github/actions/setup-app
  with:
    node-version: '20'
```

---

## Self-hosted Runners

```yaml
# Dùng self-hosted runner
jobs:
  build:
    runs-on: [self-hosted, linux, x64, docker]
    # Labels: self-hosted + custom labels
```

```bash
# Setup self-hosted runner
# GitHub → Settings → Actions → Runners → New self-hosted runner
mkdir actions-runner && cd actions-runner
curl -O -L https://github.com/actions/runner/releases/download/v2.317.0/actions-runner-linux-x64-2.317.0.tar.gz
tar xzf actions-runner-linux-x64-2.317.0.tar.gz
./config.sh --url https://github.com/company/repo --token <TOKEN>
./run.sh

# Hoặc chạy như systemd service
sudo ./svc.sh install
sudo ./svc.sh start
```

**Self-hosted runner với Docker-in-Docker (DinD):**
```yaml
# docker-compose.yml cho runner
services:
  runner:
    image: myoung34/github-runner:latest
    environment:
      RUNNER_SCOPE: repo
      REPO_URL: https://github.com/company/repo
      RUNNER_TOKEN: ${RUNNER_TOKEN}
      LABELS: self-hosted,linux,docker
      RUNNER_WORKDIR: /tmp/runner
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /tmp/runner:/tmp/runner
    restart: unless-stopped
```

---

## Conditional Execution

```yaml
steps:
  # Chỉ chạy trên main branch
  - name: Deploy
    if: github.ref == 'refs/heads/main'

  # Chỉ chạy khi job trước fail
  - name: Notify failure
    if: failure()

  # Chạy kể cả khi pipeline fail
  - name: Upload logs
    if: always()

  # Chỉ chạy khi event là push (không phải PR)
  - name: Push image
    if: github.event_name == 'push'

  # Check output từ step trước
  - name: Use output
    if: steps.check.outputs.changed == 'true'

  # Chỉ chạy khi không phải fork (secrets không available trong fork)
  - name: Publish
    if: github.repository == 'company/myapp'
```

---

## Useful Patterns

### Detect changed files
```yaml
- name: Get changed files
  id: changes
  uses: dorny/paths-filter@v3
  with:
    filters: |
      src:
        - 'src/**'
      docker:
        - 'Dockerfile'
        - 'docker-compose.yml'

- name: Run tests (only if src changed)
  if: steps.changes.outputs.src == 'true'
  run: npm test
```

### Notify Slack on failure
```yaml
- name: Notify Slack
  if: failure()
  uses: slackapi/slack-github-action@v1.26.0
  with:
    payload: |
      {
        "text": "❌ Pipeline failed: ${{ github.workflow }} on ${{ github.ref }}",
        "attachments": [{
          "color": "danger",
          "fields": [
            {"title": "Repository", "value": "${{ github.repository }}"},
            {"title": "Commit", "value": "${{ github.sha }}"},
            {"title": "Author", "value": "${{ github.actor }}"},
            {"title": "Run URL", "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}
          ]
        }]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Concurrency control (prevent parallel deploys)
```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false    # false = queue, true = cancel in-flight
```

---

## Gotchas

- **`actions/checkout` depth**: mặc định `fetch-depth: 1` (shallow clone). Nếu dùng git history (versioning, blame) → `fetch-depth: 0`. Shallow clone nhanh hơn nhưng thiếu history.
- **Secrets trong forks**: fork PRs không có access vào repository secrets (bảo mật). Dùng `pull_request_target` cẩn thận — nó có access secrets nhưng chạy code từ fork (nguy hiểm).
- **GITHUB_TOKEN permissions**: mặc định có write permissions. Best practice: set `permissions: {}` ở top-level rồi add chỉ những gì cần cho từng job.
- **Cache miss giữa branches**: cache key include branch → push feature branch không dùng được cache từ main. Dùng `restore-keys` fallback sang main cache.
- **Action version pinning**: `uses: actions/checkout@v4` có thể bị overwrite nếu tag v4 bị push lại. An toàn nhất: pin theo SHA `uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683`.
- **Workflow không chạy khi push từ workflow**: push commit từ GITHUB_TOKEN không trigger workflow khác (tránh infinite loop). Dùng Personal Access Token hoặc GitHub App token để trigger cross-workflow.
