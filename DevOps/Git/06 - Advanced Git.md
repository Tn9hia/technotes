# 06 - Advanced Git

> **Course:** [[00 - Git Course Index]]
> **Prev:** [[05 - Undoing & Recovery]] | **Next:** [[07 - Git for DevSecOps]]

---

## 🍒 git cherry-pick

Lấy **1 commit cụ thể** từ branch khác mà không merge toàn bộ branch:

```text
  BEFORE:
  main    ──► A ──► B ──► C
  feature ──► A ──► D ──► E ──► F
  
  Chỉ muốn lấy commit E (hotfix) vào main:
  
  git checkout main
  git cherry-pick E
  
  AFTER:
  main    ──► A ──► B ──► C ──► E'
  feature ──► A ──► D ──► E ──► F
  
  E' = bản copy của E với SHA mới
```

```bash
# Cherry-pick 1 commit
git cherry-pick abc123

# Cherry-pick range
git cherry-pick abc123..def456

# Cherry-pick nhưng không commit ngay (để review/edit)
git cherry-pick --no-commit abc123

# Abort nếu có conflict không muốn resolve
git cherry-pick --abort
```

---

## 📦 git stash

Tạm thời "cất" thay đổi để context-switch:

```text
  Working Tree (dirty)
  │
  git stash push ──► Stash Stack
                     ├── stash@{0}: WIP: fix payment  ← latest
                     ├── stash@{1}: WIP: refactor auth
                     └── stash@{2}: WIP: add tests
  
  git stash pop ──► Lấy stash@{0} ra và XOÁ khỏi stack
  git stash apply stash@{1} ──► Lấy ra NHƯNG GIỮ trong stack
```

```bash
# Stash với message có ý nghĩa (không dùng default message!)
git stash push -m "WIP: jwt refresh token - half done"

# Include untracked files
git stash push -u -m "WIP: new feature files"

# List stashes
git stash list

# Apply specific stash
git stash apply stash@{2}

# Xem nội dung stash trước khi apply
git stash show -p stash@{0}

# Tạo branch từ stash (cách clean nhất)
git stash branch feature/from-stash stash@{0}

# Xoá stash
git stash drop stash@{1}
git stash clear  # xoá hết ⚠️
```

---

## 🔍 git bisect — Bug Hunting

Binary search tìm commit gây ra bug:

```text
  C1  C2  C3  C4  C5  C6  C7  C8  C9  C10
  ✅  ✅  ✅  ?   ?   ?   ?   ?   ?   ❌
  
  git bisect:
  Round 1: Test C5 → ✅ good
  Round 2: Test C8 → ❌ bad
  Round 3: Test C6 → ✅ good
  Round 4: Test C7 → ❌ bad
  
  Result: C7 là commit gây bug! (log₂(10) ≈ 4 bước)
```

```bash
# Start bisect
git bisect start
git bisect bad HEAD              # current state: broken
git bisect good v1.2.0           # last known good state

# Git tự checkout commit ở giữa, bạn test và report
git bisect good                  # nếu commit này ổn
git bisect bad                   # nếu commit này broken

# Tự động với test script
git bisect run go test ./...
# Script exit 0 = good, exit 1 = bad

# Kết thúc
git bisect reset
```

---

## 🪝 Git Hooks

Scripts chạy tự động tại các điểm trong Git workflow:

```text
  LOCAL HOOKS (trong .git/hooks/)         REMOTE HOOKS (server-side)
  ─────────────────────────────           ──────────────────────────
  
  pre-commit                              pre-receive
  ↓ (before commit is created)           ↓ (before push accepted)
  
  commit-msg                              update
  ↓ (validate commit message)            ↓ (per-branch validation)
  
  pre-push                                post-receive
  ↓ (before pushing)                      ↓ (after push, trigger CI/CD)
  
  post-checkout
  ↓ (after switching branch)
```

### pre-commit hook ví dụ (Go project)

```bash
#!/bin/sh
# .git/hooks/pre-commit

# Run tests
echo "Running tests..."
go test ./... || { echo "Tests failed! Commit aborted."; exit 1; }

# Check for secrets
echo "Scanning for secrets..."
grep -rE "(password|secret|api_key|token)\s*=\s*['\"][^'\"]{8,}" --include="*.go" . \
  && { echo "Potential secret detected! Commit aborted."; exit 1; }

# Run linter
golangci-lint run || { echo "Linting failed! Fix issues before commit."; exit 1; }

echo "All checks passed ✓"
```

```bash
# Enable hook
chmod +x .git/hooks/pre-commit
```

### Dùng pre-commit framework (recommended)

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: detect-private-key
      - id: check-added-large-files
      - id: no-commit-to-branch
        args: ['--branch', 'main', '--branch', 'master']
  
  - repo: https://github.com/zricethezav/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
  
  - repo: https://github.com/dnephin/pre-commit-golang
    rev: v0.5.1
    hooks:
      - id: go-fmt
      - id: go-vet
      - id: golangci-lint
```

```bash
pip install pre-commit
pre-commit install
pre-commit run --all-files  # chạy một lần để test
```

---

## 📊 git log — Power Queries

```bash
# Đẹp và có graph
git log --oneline --graph --all --decorate

# Tìm commit theo content (pickaxe)
git log -S "password_reset_token"       # thêm/xoá string này
git log -G "def.*auth"                  # regex

# Tìm commit theo author và date
git log --author="Nghia" --since="2 weeks ago"

# Xem file đã thay đổi trong mỗi commit
git log --stat --oneline

# Blame: ai viết dòng nào
git blame -L 10,25 src/auth/handler.go

# So sánh 2 branches
git log main..feature/new-api --oneline   # commits trong feature nhưng không trong main
git log feature/new-api..main --oneline   # commits trong main nhưng không trong feature

# Tìm commit xoá function (rất hữu ích!)
git log --all -S "functionName" --source -- "*.go"
```

---

## 🔧 git worktree — Làm việc với nhiều branches cùng lúc

```text
  Thay vì stash và switch branch → mở branch thứ 2 trong folder khác:
  
  ~/projects/myapp/          → main branch
  ~/projects/myapp-hotfix/   → hotfix/critical-bug (cùng repo!)
  
  git worktree add ../myapp-hotfix hotfix/critical-bug
```

```bash
# Tạo worktree
git worktree add ../myapp-hotfix hotfix/payment-crash

# List worktrees
git worktree list

# Xoá worktree
git worktree remove ../myapp-hotfix
```

---

## 🧪 Bài tập Module 06

Xem chi tiết tại: [[08 - Exercises & Use Cases#Module 06 Advanced]]

**Quick exercises:**
1. Dùng `git bisect` tìm commit gây ra bug trong 20 commits
2. Setup `pre-commit` với gitleaks và golangci-lint
3. Practice `cherry-pick` để backport fix từ main vào release branch
4. Thử `git worktree` để fix hotfix trong khi vẫn đang develop feature

---

## Tags
#git #advanced #cherry-pick #stash #bisect #hooks #worktree #git-log
