# Git Course — From Zero to Hero

> Khoá học Git thực chiến cho DevSecOps Engineer. Từ concept cơ bản đến workflow nâng cao dùng trong production.

---

## 🗺️ Course Map

```text
                    ┌─────────────────────┐
                    │  00 - Course Index  │  ◄── Bạn đang ở đây
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
  │ 01 - Basics   │   │ 02 - Branch   │   │ 03 - Remote   │
  │ & Core Model  │   │ & Merge       │   │ & Collab      │
  └───────┬───────┘   └───────┬───────┘   └───────┬───────┘
          │                   │                   │
          └────────────┬──────┘                   │
                       ▼                          │
              ┌────────────────┐                  │
              │ 04 - Workflow  │◄─────────────────┘
              │ Strategies     │
              └───────┬────────┘
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
  ┌─────────────┐ ┌─────────┐ ┌──────────────┐
  │ 05 - Undo   │ │ 06 -    │ │ 07 -         │
  │ & Recovery  │ │ Advanced│ │ DevSecOps    │
  └─────────────┘ └─────────┘ └──────────────┘
                                      │
                                      ▼
                             ┌─────────────────┐
                             │ 08 - Exercises  │
                             │ & Use Cases     │
                             └─────────────────┘
```

---

## 📚 Modules

| # | Module | Topics | Level |
|---|--------|---------|-------|
| 01 | [[01 - Git Basics & Core Model]] | Working tree, staging, commit, HEAD | 🟢 Beginner |
| 02 | [[02 - Branching & Merging]] | Branch, merge, rebase, conflict | 🟢 Beginner |
| 03 | [[03 - Remote & Collaboration]] | remote, fetch, pull, push, PR | 🟡 Intermediate |
| 04 | [[04 - Git Workflow Strategies]] | GitFlow, Trunk-based, GitHub Flow | 🟡 Intermediate |
| 05 | [[05 - Undoing & Recovery]] | reset, revert, restore, reflog | 🟡 Intermediate |
| 06 | [[06 - Advanced Git]] | rebase -i, cherry-pick, stash, bisect, hooks | 🔴 Advanced |
| 07 | [[07 - Git for DevSecOps]] | Signing commits, secret scanning, CI/CD | 🔴 Advanced |
| 08 | [[08 - Exercises & Use Cases]] | Bài tập thực hành theo use case | 🧪 Practice |

---

## 🔑 Key Concepts Graph

```text
  commit ──► history ──► log
    │                      │
    │                   reflog
    ▼
  branch ──► merge ──► conflict resolution
    │          │
    │        rebase
    ▼
  remote ──► fetch ──► pull
               │
             push ──► PR/MR
```

---

### Lab
- https://learngitbranching.js.org/
## Tags
#git #devops #version-control #course
