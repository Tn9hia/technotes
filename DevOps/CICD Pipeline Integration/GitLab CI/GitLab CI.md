---
title: GitLab CI/CD
tags:
  - cicd
  - gitlab
  - deep-dive
date: 2026-04-26
---

# GitLab CI/CD

## Architecture

```
GitLab Repository
    │
    └── .gitlab-ci.yml         ← pipeline definition
    │
GitLab CI/CD
    │
    ├── Pipeline (triggered by push/MR/schedule)
    │       │
    │       ├── Stage: build
    │       │       └── Job: compile
    │       │
    │       ├── Stage: test
    │       │       ├── Job: unit-test  (parallel)
    │       │       └── Job: lint       (parallel)
    │       │
    │       └── Stage: deploy
    │               └── Job: deploy-staging
    │
    └── Runner (executor)
            ├── Shell executor
            ├── Docker executor
            └── Kubernetes executor
```

---

## .gitlab-ci.yml Structure

```yaml
# .gitlab-ci.yml

# ─── Stages (thứ tự chạy) ───
stages:
  - build
  - test
  - security
  - package
  - deploy-staging
  - deploy-production

# ─── Global defaults ───
default:
  image: node:20-alpine           # default image cho tất cả jobs
  before_script:
    - npm ci --cache .npm --prefer-offline
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - .npm/
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
  tags:
    - docker                     # chỉ chạy trên runners có tag "docker"

# ─── Variables ───
variables:
  REGISTRY: registry.internal
  IMAGE_NAME: platform/myapp
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"   # cho DinD
  FF_USE_FASTZIP: "true"         # faster artifact compression

# ─── Job templates (anchors) ───
.deploy-template: &deploy-template
  image: bitnami/kubectl:1.29
  before_script:
    - kubectl config use-context $KUBE_CONTEXT
  dependencies: []

# ─── Build stage ───
compile:
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour
  only:
    - main
    - merge_requests

# ─── Test stage ───
unit-test:
  stage: test
  services:
    - name: postgres:16-alpine
      alias: postgres
      variables:
        POSTGRES_PASSWORD: testpass
        POSTGRES_DB: testdb
  variables:
    DATABASE_URL: postgresql://postgres:testpass@postgres:5432/testdb
  script:
    - npm run test:coverage
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'   # parse coverage %
  artifacts:
    when: always
    reports:
      junit: test-results.xml              # GitLab test report
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
    paths:
      - coverage/
    expire_in: 7 days

lint:
  stage: test
  script:
    - npm run lint
  allow_failure: false

# ─── Security stage ───
sast:
  stage: security
  include:
    - template: Security/SAST.gitlab-ci.yml   # GitLab built-in SAST

dependency-scan:
  stage: security
  include:
    - template: Security/Dependency-Scanning.gitlab-ci.yml

secret-detection:
  stage: security
  include:
    - template: Security/Secret-Detection.gitlab-ci.yml

# ─── Package stage ───
build-image:
  stage: package
  image: docker:26.1
  services:
    - docker:26.1-dind
  variables:
    IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build
        --cache-from $CI_REGISTRY_IMAGE:latest
        --build-arg BUILDKIT_INLINE_CACHE=1
        -t $IMAGE_TAG
        -t $CI_REGISTRY_IMAGE:latest
        .
    - docker push $IMAGE_TAG
    - docker push $CI_REGISTRY_IMAGE:latest
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_COMMIT_TAG

trivy-scan:
  stage: package
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  needs: [build-image]
  script:
    - trivy image
        --exit-code 1
        --severity HIGH,CRITICAL
        --format sarif
        --output trivy-results.sarif
        $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  artifacts:
    reports:
      container_scanning: trivy-results.sarif   # GitLab security dashboard
  allow_failure: false

# ─── Deploy staging ───
deploy-staging:
  <<: *deploy-template
  stage: deploy-staging
  needs: [trivy-scan]
  variables:
    KUBE_CONTEXT: company-k8s/staging-cluster
  script:
    - |
      helm upgrade --install myapp ./charts/myapp \
        --namespace staging \
        --set image.tag=$CI_COMMIT_SHORT_SHA \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --values ./charts/myapp/values-staging.yaml \
        --wait \
        --timeout 5m
  environment:
    name: staging
    url: https://staging.myapp.internal
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# ─── Deploy production (manual) ───
deploy-production:
  <<: *deploy-template
  stage: deploy-production
  needs: [deploy-staging]
  variables:
    KUBE_CONTEXT: company-k8s/production-cluster
  script:
    - |
      helm upgrade --install myapp ./charts/myapp \
        --namespace production \
        --set image.tag=$CI_COMMIT_SHORT_SHA \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --values ./charts/myapp/values-production.yaml \
        --wait \
        --timeout 10m
  environment:
    name: production
    url: https://myapp.internal
  when: manual                    # require manual trigger
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
      when: manual
```

---

## Predefined Variables

```yaml
# Quan trọng nhất:
$CI_COMMIT_SHA              # full git SHA
$CI_COMMIT_SHORT_SHA        # 8-char SHA (abc12345)
$CI_COMMIT_BRANCH           # branch name
$CI_COMMIT_TAG              # tag name (nếu triggered bởi tag)
$CI_COMMIT_MESSAGE          # commit message
$CI_PIPELINE_ID             # unique pipeline ID
$CI_JOB_ID                  # unique job ID
$CI_PROJECT_NAME            # repository name
$CI_PROJECT_PATH            # namespace/repo-name
$CI_REGISTRY                # registry URL (GitLab Container Registry)
$CI_REGISTRY_IMAGE          # full image path: registry/namespace/project
$CI_REGISTRY_USER           # login user
$CI_REGISTRY_PASSWORD       # login password
$CI_ENVIRONMENT_NAME        # environment name (staging/production)
$CI_MERGE_REQUEST_IID       # MR number (chỉ có khi trigger bởi MR)
```

---

## Rules & Workflow

`rules:` thay thế `only/except` (hiện đại hơn):

```yaml
job:
  rules:
    # Chạy khi push lên main
    - if: $CI_COMMIT_BRANCH == "main"
      when: on_success

    # Chạy khi là semver tag
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
      when: on_success

    # Chạy khi MR (với thay đổi trong src/)
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - src/**/*
        - package*.json
      when: on_success

    # Manual trigger cho schedule pipeline
    - if: $CI_PIPELINE_SOURCE == "schedule"
      when: manual

    # Default: không chạy
    - when: never
```

**Workflow rules** — control khi nào tạo pipeline:
```yaml
workflow:
  rules:
    # Không tạo pipeline cho branch bắt đầu bằng "wip/"
    - if: $CI_COMMIT_BRANCH =~ /^wip\//
      when: never
    # Tạo pipeline cho push và MR
    - if: $CI_PIPELINE_SOURCE == "push"
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    # Tạo pipeline cho tags
    - if: $CI_COMMIT_TAG
```

---

## Artifacts & Dependencies

```yaml
build:
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - dist/          # pass files tới subsequent jobs
    expire_in: 1 day

deploy:
  stage: deploy
  needs:
    - job: build
      artifacts: true  # download artifacts từ "build"
  script:
    - ls dist/         # available here

# Chỉ download artifacts từ specific jobs (không download tất cả)
test:
  needs:
    - job: build
      artifacts: true
    - job: other-job
      artifacts: false  # chạy sau other-job nhưng không cần artifacts
```

---

## Cache

```yaml
# Cache theo branch (mỗi branch cache riêng)
cache:
  key: $CI_COMMIT_REF_SLUG
  paths:
    - node_modules/
    - .npm/

# Cache shared (fallback sang main nếu branch cache miss)
cache:
  - key:
      files:
        - package-lock.json
      prefix: npm
    paths:
      - .npm/
    policy: pull           # chỉ download cache, không upload

# Policy: pull-push (default), pull, push
# pull: nhanh — chỉ download, không upload sau job
# push: chỉ upload (warm cache cho người khác)
```

---

## Environments & Deployments

```yaml
deploy-staging:
  environment:
    name: staging
    url: https://staging.myapp.internal
    on_stop: stop-staging    # job để tear down
    auto_stop_in: 1 week     # auto stop sau 1 tuần

stop-staging:
  script:
    - helm uninstall myapp --namespace staging
  environment:
    name: staging
    action: stop
  when: manual
```

GitLab track deployments per environment → history, rollback từ UI.

---

## GitLab Runners

```yaml
# Cài GitLab Runner
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-runner

# Register runner
sudo gitlab-runner register \
  --url https://gitlab.company.com \
  --token <registration-token> \
  --executor docker \
  --docker-image alpine:3.19 \
  --description "Docker runner" \
  --tag-list docker,linux
```

```toml
# /etc/gitlab-runner/config.toml
concurrent = 4               # max parallel jobs per runner
check_interval = 0

[[runners]]
  name = "docker-runner"
  url = "https://gitlab.company.com"
  token = "TOKEN"
  executor = "docker"
  [runners.docker]
    image = "alpine:3.19"
    privileged = false       # true nếu cần DinD
    volumes = ["/cache"]
    pull_policy = "if-not-present"
    shm_size = 0
  [runners.cache]
    Type = "s3"
    [runners.cache.s3]
      ServerAddress = "minio.internal:9000"
      BucketName = "gitlab-runner-cache"
      BucketLocation = "us-east-1"
      Insecure = false
```

---

## Include & Extends

```yaml
# Chia nhỏ pipeline thành nhiều files
include:
  - local: '.gitlab/ci/build.yml'           # file trong cùng repo
  - local: '.gitlab/ci/test.yml'
  - project: 'company/ci-templates'         # file từ repo khác
    ref: main
    file: '/templates/docker-build.yml'
  - template: 'Security/SAST.gitlab-ci.yml' # GitLab built-in template
  - remote: 'https://example.com/ci.yml'    # remote URL

# Extends (inherit job config)
.base-deploy:
  image: bitnami/kubectl:1.29
  before_script:
    - echo "Setup kubectl"

deploy-staging:
  extends: .base-deploy
  script:
    - kubectl apply -f k8s/staging/
```

---

## Pipeline Optimization

```yaml
# Parallel jobs trong 1 job
test:
  parallel: 4              # split tests across 4 runners
  script:
    - bundle exec rspec --format progress $(ls spec/ | awk "NR % $CI_NODE_TOTAL == $CI_NODE_INDEX")

# DAG (Directed Acyclic Graph) — bypass stage order
deploy:
  needs: [build, test-unit]  # chạy ngay khi dependencies done, không chờ cả stage
  stage: deploy

# Interruptible jobs (cancel outdated pipelines)
build:
  interruptible: true       # cancel job này nếu pipeline mới hơn được trigger

# Resource groups (prevent parallel deploys)
deploy-production:
  resource_group: production-deploy   # chỉ 1 job cùng lúc
```

---

## Gotchas

- **`only/except` vs `rules`**: `rules` ưu tiên hơn và mạnh hơn `only/except`. Không mix cả hai trong cùng job. Dùng `rules` cho projects mới.
- **Cache key collision**: cùng cache key giữa jobs → overwrite nhau. Dùng `$CI_JOB_NAME` hoặc `$CI_COMMIT_REF_SLUG` trong key.
- **Artifact size limit**: mặc định artifact limit 100MB. Large artifacts cần dùng object storage hoặc S3 cache.
- **`needs:` bypass stage order**: với `needs`, job có thể chạy trước stage của nó. Tránh circular dependencies.
- **DinD và privileged**: `docker:dind` cần `privileged: true` trong runner config → security risk. Dùng Kaniko hoặc Buildah cho rootless image build.
- **Environment variables scope**: variables trong `.gitlab-ci.yml` thấy bởi tất cả jobs. Để restrict, dùng environment-specific variables trong GitLab UI (Settings → CI/CD → Variables → Environments).
- **Protected branches và tags**: protected variables chỉ available cho jobs chạy trên protected branches/tags. Staging pipeline không thấy production secrets.
