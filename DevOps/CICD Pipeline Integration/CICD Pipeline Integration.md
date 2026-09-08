---
title: CI/CD Pipeline Integration
tags:
  - cicd
  - devops
  - index
date: 2026-04-26
---

# CI/CD Pipeline Integration

## Overview

- [[CICD Concepts & Overview]] — CI vs CD vs CD, pipeline anatomy, DORA metrics, tool landscape, best practices

## CI Platforms

- [[GitHub Actions]] — Triggers, jobs, matrix, secrets, environments, reusable workflows, self-hosted runners
- [[GitLab CI]] — `.gitlab-ci.yml`, stages, rules, artifacts, cache, runners, DinD

## Pipeline Design

- [[Pipeline Patterns]] — App repo vs manifest repo, image tag update flow (git push / github-script / yq), ArgoCD Image Updater, Kargo promotion, mono-repo strategy

## Deployment Strategies

- [[Deployment Strategies]] — Recreate, Rolling, Blue/Green, Canary (Nginx Ingress + Argo Rollouts), A/B Testing

## Security

- [[DevSecOps]] — Secret scanning (Gitleaks), SAST (Semgrep, CodeQL), SCA (Snyk, Dependabot), Container scan (Trivy), IaC scan (Checkov), SBOM, Falco runtime security

---

## Big Picture Flow

```
Developer pushes code
        │
        ▼
┌───────────────────────────────────────────────────┐
│  CI Pipeline  (app-repo)                          │
│                                                   │
│  Secret Scan → SAST → SCA                        │
│       │                                           │
│  Build → Unit Test → Lint                         │
│       │                                           │
│  Docker Build → Trivy Scan → Push to Registry    │
│       │                                           │
│  Update manifest-repo (image tag)                 │
└───────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────┐
│  manifest-repo  (k8s-manifests)                   │
│                                                   │
│  apps/myapp/staging/values.yaml                   │
│    image.tag: sha-abc1234   ← updated by CI       │
└───────────────────────────────────────────────────┘
        │ ArgoCD watches
        ▼
┌───────────────────────────────────────────────────┐
│  CD  (ArgoCD)                                     │
│                                                   │
│  Detect drift → Sync → Deploy staging             │
│       │                                           │
│  Smoke test → Kargo/manual promotion              │
│       │                                           │
│  Update production manifest → Sync prod           │
└───────────────────────────────────────────────────┘
```
