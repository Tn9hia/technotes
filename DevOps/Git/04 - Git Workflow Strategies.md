# 04 - Git Workflow Strategies

> **Course:** [[00 - Git Course Index]]
> **Prev:** [[03 - Remote & Collaboration]] | **Next:** [[05 - Undoing & Recovery]]

---

## 🗺️ So sánh tổng quan

```text
  Complexity  ↑
              │
  GitFlow     │  ████████████████████  (Most complex)
              │
  GitHub Flow │  ██████████            (Medium)
              │
  Trunk-based │  ████                  (Simple but needs CI/CD maturity)
              │
              └─────────────────────────────────► Team size / Release frequency
```

---

## 1️⃣ GitHub Flow

Đơn giản, phù hợp với **continuous deployment**:

```text
  main (always deployable)
  │
  ├──► feature/user-auth ──► PR ──► merge ──► deploy
  │
  ├──► fix/payment-bug ──► PR ──► merge ──► deploy
  │
  └──► chore/update-deps ──► PR ──► merge ──► deploy

  Rules:
  ✅ main luôn deployable
  ✅ Mọi thay đổi qua branch + PR
  ✅ Deploy ngay sau khi merge
  ✅ Branch sống ngắn (< 2 ngày)
```

**Best for**: SaaS, web apps, team nhỏ-vừa, deploy liên tục.

---

## 2️⃣ GitFlow

Phức tạp hơn, phù hợp với **scheduled releases** (mobile apps, libraries):

```text
  main ─────────────────────────────────────────────────►
    │                    │                    │
    │  (tag v1.0)        │  (tag v1.1)        │ (tag v2.0)
    │                    │                    │
  develop ──────────────────────────────────────────────►
    │          │         │          │
    │     feature/A      │     feature/B
    │          │         │          │
  release/1.1 ──────────┘
    │  (bugfix only here)
    │
  hotfix/critical-fix
    │  (merge vào CẢ main VÀ develop)
    │
```

### Branches trong GitFlow

| Branch | Tạo từ | Merge vào | Tồn tại |
|--------|---------|-----------|---------|
| `main` | — | — | Permanent |
| `develop` | `main` | — | Permanent |
| `feature/*` | `develop` | `develop` | Tạm thời |
| `release/*` | `develop` | `main` + `develop` | Tạm thời |
| `hotfix/*` | `main` | `main` + `develop` | Tạm thời |

```bash
# Dùng git-flow CLI
brew install git-flow-avh

git flow init
git flow feature start user-authentication
git flow feature finish user-authentication  # auto merge vào develop
git flow release start 1.1.0
git flow release finish 1.1.0               # auto tag + merge
```

**Best for**: Mobile apps, versioned libraries, enterprise software.

---

## 3️⃣ Trunk-Based Development (TBD)

Mọi người commit thẳng vào `main` (hoặc branch rất ngắn < 1 ngày):

```text
  main ──► C1 ──► C2 ──► C3 ──► C4 ──► C5 ──►
               │              │
           (Dev A)        (Dev B)
  
  Feature chưa xong? → Feature Flags!
  
  if (featureFlag.isEnabled("new-checkout")) {
      showNewCheckout();
  }
  
  Code deployed nhưng HIDDEN. Toggle flag để enable khi ready.
```

**Yêu cầu**: CI/CD mạnh, test coverage cao, feature flags, trunk luôn green.

**Best for**: Google, Facebook scale. Cần team mature với CI/CD pipeline.

---

## 4️⃣ Trunk-Based với Short-Lived Branches

Compromise giữa TBD và GitHub Flow:

```text
  main ──────────────────────────────────────────►
    │                │                │
    └──► feat/x      └──► fix/y       └──► feat/z
    (< 1 ngày)       (< 4 giờ)       (< 1 ngày)
    │                │                │
    └────────────────┴────────────────┘
         (small PRs, fast review, squash merge)
```

---

## 🔄 Branch Naming Conventions

```text
  Prefix        Pattern                   Example
  ──────        ───────                   ───────
  feature/      feature/<ticket>-<desc>   feature/AUTH-123-jwt-refresh
  fix/          fix/<ticket>-<desc>       fix/PAY-456-null-pointer
  hotfix/       hotfix/<version>-<desc>   hotfix/v2.1.1-payment-crash
  release/      release/<version>         release/2.3.0
  chore/        chore/<desc>              chore/update-go-modules
  docs/         docs/<desc>              docs/api-authentication
  ci/           ci/<desc>               ci/add-sast-scan
```

---

## 🏭 Choosing Your Strategy

```text
  START HERE
      │
      ▼
  Multiple parallel releases? ──► YES ──► GitFlow
      │
      NO
      │
      ▼
  Continuous deployment? ──► YES ──► GitHub Flow or TBD
      │
      NO
      │
      ▼
  Scheduled releases? ──► YES ──► Simplified GitFlow (no hotfix complexity)
      │
      NO
      │
      ▼
  → Default: GitHub Flow (simplest, works for most teams)
```

---

## 🔐 Branch Protection Rules (GitHub/GitLab)

Luôn enforce cho `main`:

```text
  ✅ Require pull request reviews (min 1-2 reviewers)
  ✅ Dismiss stale reviews when new commits pushed
  ✅ Require status checks (CI must pass)
  ✅ Require branches to be up to date before merge
  ✅ Restrict who can push to main
  ✅ Require signed commits (GPG/SSH signing)
  ✅ No force pushes
  ✅ No deletions
```

---

## 🧪 Bài tập Module 04

Xem chi tiết tại: [[08 - Exercises & Use Cases#Module 04 Workflow]]

**Quick exercises:**
1. Setup Git repo theo GitHub Flow: tạo branch, PR, merge
2. Simulate GitFlow: feature → develop → release → main với tagging
3. Viết branch protection rules cho một project thật của bạn

---

## Tags
#git #workflow #gitflow #github-flow #trunk-based #branching-strategy
