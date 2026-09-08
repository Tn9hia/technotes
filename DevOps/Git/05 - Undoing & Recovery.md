# 05 - Undoing & Recovery

> **Course:** [[00 - Git Course Index]]
> **Prev:** [[04 - Git Workflow Strategies]] | **Next:** [[06 - Advanced Git]]

---

## 🗺️ Undo Decision Tree

```text
  Bạn muốn undo cái gì?
  │
  ├──► Chưa commit (working tree / staging)
  │         │
  │         ├──► Unstage file     → git restore --staged <file>
  │         ├──► Discard changes  → git restore <file>
  │         └──► Discard ALL      → git restore .  ⚠️ KHÔNG thể undo!
  │
  ├──► Đã commit, CHƯA push
  │         │
  │         ├──► Sửa commit message → git commit --amend
  │         ├──► Thêm file vào commit → git commit --amend
  │         ├──► Undo commit, GIỮ changes → git reset --soft HEAD~1
  │         ├──► Undo commit, DISCARD changes → git reset --hard HEAD~1
  │         └──► Squash/reorder commits → git rebase -i
  │
  └──► Đã commit, ĐÃ push
            │
            ├──► Tạo commit đảo ngược (SAFE) → git revert <hash>
            └──► Force push (NGUY HIỂM, chỉ branch riêng) → git push --force-with-lease
```

---

## 🔄 git reset — 3 Modes

```text
  BEFORE reset:
  
  HEAD → C5 ← main
  C4
  C3 ← point ta muốn reset về
  C2
  C1
  
  ┌──────────────────┬────────────────────┬────────────────────┐
  │                  │  git reset --soft  │  git reset --mixed │ git reset --hard
  │                  │    HEAD~2          │    HEAD~2          │   HEAD~2
  ├──────────────────┼────────────────────┼────────────────────┤
  │ Repo (commits)   │  C5, C4 removed    │  C5, C4 removed    │ C5, C4 removed
  │ Staging Area     │  C4+C5 changes     │  EMPTY             │ EMPTY
  │                  │  still staged      │                    │
  │ Working Tree     │  Unchanged         │  C4+C5 changes     │ EMPTY (GONE!)
  │                  │                    │  still there       │
  └──────────────────┴────────────────────┴────────────────────┘
  
  --soft:  "Uncommit nhưng giữ staged changes" → hay dùng để squash
  --mixed: "Uncommit và unstage" (default) → hay nhất cho general use
  --hard:  "Xoá hết" → NGUY HIỂM, mất code nếu chưa backup
```

---

## ↩️ git revert — Safe Undo

```text
  History trước revert:
  C1 ──► C2 ──► C3 ──► C4 (bug introduced here)
                             │
                           HEAD

  git revert C4:
  C1 ──► C2 ──► C3 ──► C4 ──► R4
                                │
                              HEAD
  
  R4 = commit MỚI với nội dung ĐẢO NGƯỢC của C4
  → Lịch sử KHÔNG bị thay đổi
  → An toàn cho public/shared branches
  → Đồng nghiệp vẫn pull được bình thường
```

```bash
# Revert 1 commit
git revert abc123

# Revert nhiều commits
git revert HEAD~3..HEAD

# Revert nhưng không tạo commit ngay (để review trước)
git revert --no-commit abc123
git revert --no-commit def456
git commit -m "revert: rollback payment service changes"
```

---

## 🛟 git reflog — The Safety Net

`reflog` ghi lại MỌI thay đổi của HEAD trong 90 ngày. Đây là "undo của undo":

```bash
git reflog
# Output:
# abc123 (HEAD -> main) HEAD@{0}: commit: feat: add logging
# def456 HEAD@{1}: reset: moving to HEAD~1
# ghi789 HEAD@{2}: commit: wip: half-done feature  ← BỊ LOST sau reset
# ...

# Recover commit bị "mất" sau git reset --hard
git checkout ghi789                    # vào commit đó
git checkout -b recovery/lost-feature  # tạo branch mới
# hoặc ngắn gọn:
git branch recovery/lost-feature ghi789
```

> 💡 **Fact**: Với `reflog`, bạn hầu như không bao giờ mất code nếu đã từng commit. `git reset --hard` không xoá object, chỉ xoá pointer. Object còn đó 90 ngày.

---

## 🧹 git restore vs git checkout vs git reset

```text
  Lệnh cũ (gây nhầm lẫn):      Lệnh mới (Git 2.23+, rõ ràng hơn):
  ─────────────────────────     ─────────────────────────────────────
  
  git checkout <file>      →    git restore <file>
  (discard working changes)     (discard working changes)
  
  git checkout HEAD <file> →    git restore --source=HEAD <file>
  
  git reset HEAD <file>    →    git restore --staged <file>
  (unstage)                     (unstage)
  
  git checkout <branch>    →    git switch <branch>
  git checkout -b <branch> →    git switch -c <branch>
```

---

## 🔥 Emergency Scenarios

### Scenario 1: Push nhầm secrets lên remote

```bash
# ⚠️ CRITICAL: Ngay lập tức revoke credential đó trước!

# Xoá file khỏi toàn bộ history
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch path/to/secret.env" \
  --prune-empty --tag-name-filter cat -- --all

# Hoặc dùng BFG Repo Cleaner (nhanh hơn)
java -jar bfg.jar --delete-files secret.env
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push origin --force --all

# Nhắc đồng nghiệp: xoá local repo và clone lại
```

### Scenario 2: Merge nhầm branch vào main

```bash
# Tìm commit ngay trước merge
git log --oneline -10

# Option A: Revert merge commit (SAFE, giữ history)
git revert -m 1 <merge-commit-hash>
# -m 1: giữ parent thứ 1 (main), loại bỏ feature branch

# Option B: Reset (CHỈ nếu chưa ai pull)
git reset --hard <commit-before-merge>
git push --force-with-lease
```

### Scenario 3: Xoá nhầm branch

```bash
# Tìm trong reflog
git reflog | grep "branch-name"

# Recover
git checkout -b branch-name <sha-from-reflog>
```

---

## 🧪 Bài tập Module 05

Xem chi tiết tại: [[08 - Exercises & Use Cases#Module 05 Recovery]]

**Quick exercises:**
1. Commit, `reset --hard`, dùng `reflog` để recover
2. Push lên remote, dùng `revert` để undo an toàn
3. Simulate "merge nhầm branch" và recover bằng `revert -m 1`
4. Dùng `git stash` để context-switch giữa tasks

---

## Tags
#git #undo #recovery #reset #revert #reflog #git-restore
