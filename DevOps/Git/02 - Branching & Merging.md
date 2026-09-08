# 02 - Branching & Merging

> **Course:** [[00 - Git Course Index]]
> **Prev:** [[01 - Git Basics & Core Model]] | **Next:** [[03 - Remote & Collaboration]]

---

## 🌿 Branch là gì?

Branch là một **con trỏ nhẹ (lightweight pointer)** đến một commit. Tạo branch = tạo 1 file 41 bytes. Cực kỳ cheap!

```text
  main   ──► C1 ──► C2 ──► C3
                            │
                          HEAD

  Sau khi: git checkout -b feature/login
                            │
  main   ──► C1 ──► C2 ──► C3
                            │
  feature/login ────────────┘
                            │
                          HEAD  (now points to feature/login)
```

---

## 🔀 Merge Strategies

### 1. Fast-Forward Merge

```text
  BEFORE:                    AFTER git merge feature:
  
  main ──► C1 ──► C2         main ──► C1 ──► C2 ──► C3 ──► C4
                   │                                         │
  feature ─────────┴──► C3 ──► C4                         HEAD
  
  → Chỉ xảy ra khi main không có commit mới sau khi branch.
  → Không tạo merge commit, history linear và clean.
```

### 2. 3-Way Merge (Merge Commit)

```text
  BEFORE:                    AFTER git merge feature:
  
  main ──► C1 ──► C2 ──► C3     main ──► C1 ──► C2 ──► C3 ──► M
                   │                               │             │
  feature ─────────┴──► C4 ──► C5     feature ────┴──► C4 ──► C5┘
  
  M = Merge commit (có 2 parents: C3 và C5)
  → Preserves branch history.
  → History có thể phức tạp hơn.
```

### 3. Squash Merge

```text
  BEFORE:                    AFTER git merge --squash feature:
  
  feature: C4 ──► C5 ──► C6     main: ... ──► C3 ──► S
                                 
  S = 1 commit duy nhất chứa toàn bộ changes từ C4+C5+C6
  → Clean main history
  → Mất granular commit history của feature branch
```

### 4. Rebase

```text
  BEFORE:                    AFTER git rebase main:
  
  main ──► C1 ──► C2 ──► C3     main ──► C1 ──► C2 ──► C3
                   │                                      │
  feature ─────────┴──► C4'──► C5'     feature ──────────┴──► C4' ──► C5'
  
  → C4, C5 được "replay" lên đầu C3, tạo ra commits MỚI (C4', C5')
  → Linear history, không có merge commit
  ⚠️  GOLDEN RULE: NEVER rebase shared/public branches!
```

---

## ⚔️ Conflict Resolution

Conflict xảy ra khi 2 branch cùng sửa 1 dòng code:

```text
  <<<<<<< HEAD (main)
  password = hashlib.sha256(password)
  =======
  password = bcrypt.hash(password, rounds=12)
  >>>>>>> feature/secure-auth
  
  Bạn phải chọn: giữ HEAD, giữ feature, hoặc merge cả 2
```

```bash
# Xem các file đang conflict
git status

# Sau khi resolve thủ công
git add <resolved-file>
git commit  # hoặc git merge --continue

# Abort nếu muốn huỷ
git merge --abort
```

### Merge Tools

```bash
# Built-in 3-way diff
git mergetool

# Config VS Code làm merge tool
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
```

---

## 🔄 Rebase Interactive — Rewrite History

```bash
git rebase -i HEAD~4  # Chỉnh sửa 4 commits gần nhất
```

```text
  Editor mở ra:
  ┌─────────────────────────────────────────┐
  │ pick a1b2c3 feat: add login page        │
  │ pick d4e5f6 wip: debugging              │  ← squash vào trước
  │ pick g7h8i9 fix: typo in button         │  ← squash vào trước
  │ pick j0k1l2 feat: add logout            │
  │                                         │
  │ Commands:                               │
  │ p, pick   = use commit                  │
  │ s, squash = meld into previous commit   │
  │ r, reword = edit commit message         │
  │ d, drop   = remove commit               │
  │ e, edit   = stop and amend              │
  └─────────────────────────────────────────┘
  
  Sau khi sửa:
  pick a1b2c3 feat: add login page
  squash d4e5f6 wip: debugging
  squash g7h8i9 fix: typo in button
  pick j0k1l2 feat: add logout
  
  Result: 2 commits clean thay vì 4 commits lộn xộn
```

---

## 🌲 Branch Management

```bash
# Tạo và switch
git checkout -b feature/new-api      # cách cũ
git switch -c feature/new-api        # cách mới (Git 2.23+)

# List branches
git branch                           # local
git branch -r                        # remote
git branch -a                        # all

# Rename
git branch -m old-name new-name

# Delete
git branch -d feature/done           # safe delete (đã merged)
git branch -D feature/abandoned      # force delete

# Set upstream
git branch --set-upstream-to=origin/main main
```

---

## 📊 Visualize Branch Graph

```bash
# ASCII graph trong terminal
git log --oneline --graph --all --decorate

# Output ví dụ:
# * a1b2c3 (HEAD -> main) feat: deploy pipeline
# * d4e5f6 fix: health check endpoint
# | * g7h8i9 (feature/cache) feat: add Redis caching
# | * j0k1l2 chore: add Redis dependency
# |/
# * k1l2m3 feat: initial API structure
```

---

## 🆚 Merge vs Rebase — Khi nào dùng gì?

```text
  USE MERGE WHEN:                  USE REBASE WHEN:
  ┌────────────────────┐           ┌────────────────────┐
  │ • Feature branch   │           │ • Cập nhật local   │
  │   → main (PR)      │           │   feature branch   │
  │ • Cần preserve     │           │ • Clean up commits │
  │   full history     │           │   trước khi PR     │
  │ • Public/shared    │           │ • Branch chỉ mình  │
  │   branches         │           │   bạn đang dùng    │
  └────────────────────┘           └────────────────────┘
          ↓                                ↓
  History shows "what                 History is clean
  actually happened"                  and linear
```

> ⚠️ **Production rule**: Không bao giờ `git push --force` lên `main`/`master`. Nếu cần force push, dùng `--force-with-lease` và chỉ trên branch của riêng bạn.

---

## 🧪 Bài tập Module 02

Xem chi tiết tại: [[08 - Exercises & Use Cases#Module 02 Branching]]

**Quick exercises:**
1. Tạo 2 branch song song, cùng sửa 1 file, merge và resolve conflict
2. Tạo 5 "wip" commits, dùng `rebase -i` squash thành 1 commit đẹp
3. Simulate fast-forward vs 3-way merge, quan sát `git log --graph`
4. Thử `git merge --no-ff` và so sánh với merge thường

---

## Tags
#git #branch #merge #rebase #conflict #workflow
