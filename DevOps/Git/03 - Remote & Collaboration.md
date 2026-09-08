# 03 - Remote & Collaboration

> **Course:** [[00 - Git Course Index]]
> **Prev:** [[02 - Branching & Merging]] | **Next:** [[04 - Git Workflow Strategies]]

---

## 🌐 Remote là gì?

Remote là **alias cho URL của repo trên server**. Convention: remote đầu tiên luôn đặt là `origin`.

```text
  LOCAL REPO                    REMOTE (GitHub/GitLab)
  ──────────                    ──────────────────────
  
  main          ──push──►       origin/main
  feature/x     ──push──►       origin/feature/x
                ◄─fetch──       origin/main (updated by teammate)
  
  Remote-tracking branches:
  origin/main   → copy LOCAL của trạng thái remote/main
  (chỉ update khi bạn fetch/pull)
```

---

## ⬆️⬇️ fetch vs pull vs push

```text
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  git fetch origin                                        │
  │  → Download commits từ remote VÀO origin/main           │
  │  → KHÔNG thay đổi working tree của bạn                  │
  │  → An toàn, không bao giờ gây conflict                  │
  │                                                          │
  │  git pull origin main                                    │
  │  → fetch + merge (hoặc rebase nếu config)               │
  │  → CÓ THỂ gây conflict nếu có local changes             │
  │                                                          │
  │  git push origin main                                    │
  │  → Upload local commits lên remote                      │
  │  → Sẽ bị reject nếu remote có commits mà bạn chưa có   │
  │                                                          │
  └──────────────────────────────────────────────────────────┘

  Safe workflow:
  git fetch → git log origin/main → git merge origin/main
  (thay vì git pull blind)
```

---

## 🔄 Remote Operations

```bash
# Xem remotes
git remote -v

# Thêm remote
git remote add origin https://github.com/user/repo.git
git remote add upstream https://github.com/original/repo.git  # fork workflow

# Đổi URL
git remote set-url origin git@github.com:user/repo.git

# Xoá remote
git remote remove upstream

# Fetch tất cả remotes
git fetch --all

# Push và set upstream cùng lúc
git push -u origin feature/new-api
# Sau lần đầu, chỉ cần: git push
```

---

## 🍴 Fork Workflow

Phổ biến trong open-source contribution:

```text
  UPSTREAM (original)                YOUR FORK
  ───────────────────                ─────────
  github.com/org/repo                github.com/you/repo
         │                                  │
         │ (fork)                           │
         └──────────────────────────────────┘
                                            │
                                     git clone ↓
                                      LOCAL REPO
                                     /           \
                             remote: origin    remote: upstream
                             (your fork)       (original)
  
  Workflow:
  1. git fetch upstream
  2. git merge upstream/main  (sync with original)
  3. git checkout -b fix/bug
  4. ... make changes ...
  5. git push origin fix/bug
  6. Tạo Pull Request: your fork → upstream
```

---

## 📋 Pull Request / Merge Request Best Practices

```text
  PR Checklist:
  ┌─────────────────────────────────────────────┐
  │ ✅ Title: clear, follows conventional commit │
  │ ✅ Description: WHY thay đổi, không chỉ WHAT │
  │ ✅ Linked issue/ticket                       │
  │ ✅ Self-reviewed trước khi assign reviewer   │
  │ ✅ Tests added/updated                       │
  │ ✅ No secrets, credentials trong code        │
  │ ✅ CI/CD passed                              │
  │ ✅ PR size nhỏ (<400 lines nếu có thể)       │
  └─────────────────────────────────────────────┘
```

---

## 🔑 SSH vs HTTPS Authentication

```text
  HTTPS                          SSH
  ─────                          ───
  git clone https://...          git clone git@github.com:...
  
  Cần: username + token          Cần: SSH key pair
  
  Pros: Đơn giản setup           Pros: Không cần nhập password
  Cons: Token quản lý phức tạp   Cons: Setup key 1 lần
  
  Cho DevSecOps → Dùng SSH + key rotation policy
```

```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "nghia@company.com" -f ~/.ssh/id_ed25519_github

# Add to ssh-agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_github

# Copy public key → paste vào GitHub Settings > SSH Keys
cat ~/.ssh/id_ed25519_github.pub

# Test connection
ssh -T git@github.com
```

---

## 🏷️ Tags & Releases

```bash
# Lightweight tag (chỉ là pointer)
git tag v1.0.0

# Annotated tag (có metadata, dùng cho releases)
git tag -a v1.0.0 -m "Release version 1.0.0 - Production ready"

# Push tags
git push origin v1.0.0          # 1 tag
git push origin --tags           # tất cả tags

# List tags
git tag -l "v1.*"

# Xoá tag
git tag -d v1.0.0-beta
git push origin --delete v1.0.0-beta
```

---

## 🧪 Bài tập Module 03

Xem chi tiết tại: [[08 - Exercises & Use Cases#Module 03 Remote]]

**Quick exercises:**
1. Setup SSH key và clone repo qua SSH
2. Simulate fork workflow với 2 local repos (dùng local path làm remote)
3. Tạo conflict khi 2 người cùng push, resolve theo đúng workflow
4. Tạo annotated tag, push lên remote, verify trên GitHub

---

## Tags
#git #remote #collaboration #ssh #pull-request #fork
