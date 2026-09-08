# 01 - Git Basics & Core Model

> **Course:** [[00 - Git Course Index]]
> **Next:** [[02 - Branching & Merging]]

---

## 🧠 Mental Model — Git là gì?

Git là một **distributed version control system**. Điểm khác biệt với SVN/CVS:

```text
  SVN (Centralized)              Git (Distributed)
  ──────────────────             ──────────────────
       SERVER                    LOCAL A    LOCAL B
       [Repo]                    [Repo]     [Repo]
      /      \                      \        /
  Dev A     Dev B                   [Remote]
  (no local                         [Repo]
   history)
                                 Mỗi người có FULL history
                                 → Offline vẫn commit được
```

---

## 🏗️ Core Model — 3 Zones

```text
┌─────────────────────────────────────────────────────────┐
│                      LOCAL MACHINE                       │
│                                                          │
│  ┌──────────────┐   git add    ┌──────────────┐         │
│  │              │ ──────────►  │              │         │
│  │ Working Tree │              │ Staging Area │         │
│  │  (dirty)     │ ◄──────────  │  (Index)     │         │
│  │              │  git restore │              │         │
│  └──────────────┘              └──────┬───────┘         │
│                                       │ git commit       │
│                                       ▼                  │
│                               ┌──────────────┐          │
│                               │  Local Repo  │          │
│                               │  (.git/)     │          │
│                               └──────┬───────┘          │
└──────────────────────────────────────┼──────────────────┘
                                       │ git push
                                       ▼
                               ┌──────────────┐
                               │ Remote Repo  │
                               │ (GitHub/Lab) │
                               └──────────────┘
```

| Zone | Lưu ở đâu | Trạng thái |
|------|-----------|------------|
| Working Tree | Filesystem | Untracked / Modified |
| Staging Area | `.git/index` | Staged |
| Local Repo | `.git/objects` | Committed |
| Remote Repo | Server | Pushed |

---

## 🔑 Objects trong Git

Git lưu mọi thứ dưới dạng 4 loại object:

```text
  COMMIT ──────────────────────────────────────┐
  │ tree: abc123                               │
  │ parent: def456                             │
  │ author: Nghia                              │
  │ message: "feat: add login"                 │
  └──► TREE ────────────────────────────────┐  │
       │ blob: 111aaa  src/main.go          │  │
       │ blob: 222bbb  src/auth.go          │  │
       │ tree: 333ccc  src/handlers/        │  │
       └──► BLOB                            │  │
            (actual file content)           │  │
                                            │  │
  TAG ──────────────────────────────────────┘  │
  │ object: (commit SHA)                       │
  │ name: v1.0.0                               │
  └─────────────────────────────────────────── ┘
```

> 💡 **Key insight**: Git không lưu diff — lưu **snapshots**. Mỗi commit là 1 snapshot toàn bộ project (nhưng dùng deduplication để tiết kiệm disk).

---

## 📍 HEAD là gì?

```text
  HEAD ──► main ──► commit C3
                    │
                    ├── C2
                    │
                    └── C1

  Khi checkout branch khác:
  HEAD ──► feature/login ──► commit C5

  Detached HEAD (checkout 1 commit cụ thể):
  HEAD ──► C2  (không point vào branch nào)
           ⚠️  Commit mới sẽ bị mồ côi nếu không tạo branch!
```

---

## ⚙️ Setup cơ bản

```bash
# Identity (bắt buộc)
git config --global user.name "Nghia"
git config --global user.email "nghia@example.com"

# Editor
git config --global core.editor "nvim"  # hoặc code --wait cho VSCode

# Default branch
git config --global init.defaultBranch main

# Xem config hiện tại
git config --list --global
```

---

## 🔄 Lifecycle của file

```text
  ┌──────────┐  git add  ┌──────────┐  git commit  ┌───────────┐
  │Untracked │ ─────────►│  Staged  │ ────────────► │ Committed │
  └──────────┘           └──────────┘               └───────────┘
       ▲                      │                           │
       │                git restore                       │
       │                 --staged                   git restore
       │                      │                     HEAD <file>
       │                      ▼                           │
  git rm      ┌──────────────────────────┐               │
  --cached    │        Modified          │◄──────────────┘
              │  (Working tree changed)  │
              └──────────────────────────┘

  git status  →  xem trạng thái các file
  git diff    →  xem thay đổi chưa staged
  git diff --staged → xem thay đổi đã staged
```

---

## 📝 Commit best practices

### Conventional Commits format

```text
<type>(<scope>): <subject>

<body>

<footer>
```

| Type | Dùng khi |
|------|----------|
| `feat` | Thêm feature mới |
| `fix` | Bug fix |
| `docs` | Chỉ thay đổi docs |
| `refactor` | Refactor, không fix bug/feature |
| `chore` | Build, tooling, dependencies |
| `ci` | CI/CD config |
| `security` | Security patch |

```bash
# Good commit message
git commit -m "feat(auth): add JWT refresh token rotation"

# Bad
git commit -m "fix stuff"
git commit -m "update"
```

---

## 📦 .gitignore

```text
# Pattern matching
*.log           # tất cả file .log
!important.log  # ngoại trừ file này
/dist           # chỉ thư mục dist ở root
build/          # bất kỳ thư mục tên build
doc/**/*.txt    # tất cả .txt trong doc/ (recursive)
```

> 🔗 Generate tại: https://gitignore.io

---

## 🧪 Bài tập Module 01

Xem chi tiết tại: [[08 - Exercises & Use Cases#Module 01 Basics]]

**Quick exercises:**
1. Init repo, tạo 3 file, commit từng file riêng biệt
2. Thay đổi 2 file cùng lúc, chỉ stage và commit 1 file
3. Dùng `git log --oneline --graph` để xem history
4. Thử `git diff HEAD~1` để so sánh với commit trước

---

## Tags
#git #basics #core-model #working-tree #staging #commit
