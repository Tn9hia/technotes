## Template Chuẩn

```
# [Technology Name] — [version nếu có]
Tags: #proxy #security #infra ...
Last updated: YYYY-MM-DD
```

---

### **1. What — Nó là cái gì?**

- 2-3 câu max. Giải thích như đang nói với đồng nghiệp, không phải docs.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

- Đừng chỉ liệt kê feature. Trả lời: _"Nếu không có nó, mình làm gì?"_

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

- Phần **KHÔNG dùng** quan trọng hơn nhiều người nghĩ.

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

- Vẽ sơ đồ ASCII/Mermaid. Dù đơn giản cũng phải có.
- Vị trí trong hệ thống
-  Các component chính
- Traffic/data flow
- Dependency là gì

### **5. How — Cơ chế hoạt động**

- Core concepts (3-5 cái quan trọng nhất)
- Request lifecycle / flow nếu có
- Backlinks → [[ child-concepts ]]
- Không cần giải thích hết, chỉ cần hiểu _tại sao nó làm vậy_

### **6. Key Config — Cấu hình cần nhớ**

Không phải copy-paste docs. Chỉ note:

- Config hay bị misconfigure nhất
- Config ảnh hưởng performance/security
- Default value nào nguy hiểm

### **7. Security Considerations**

- Attack surface ở đâu?
- Misconfiguration nào có thể gây breach?
- Hardening checklist tối thiểu

### **8. Ops Runbook — Production Notes**

- Health check như thế nào?
- Log quan trọng cần monitor
- Metric cần alert
- Restart/rollback procedure
- link to [[tech--runbook]] nếu phức tạp

### **9. Gotchas & Lessons Learned**

Phần này mày tự điền khi đã dùng thực tế. _"Cái bẫy mà docs không nói"_

### **10. Resources**
 - Official docs link
 - Bài viết thực tế (không phải tutorial Hello World)
 - Repo/config example đáng tham khảo

---
## Writing Guide

> 📌 **Decomposition Rule — 2-Tier System:**
>
> Trước khi viết, phân loại concept vào 1 trong 2 tier:
>
> **Tier 1 — Inline Note** (giữ trong section 5 của file gốc)
> Dấu hiệu nhận biết:
> - Là định nghĩa / khái niệm thuần túy
> - Không có config riêng
> - Không có failure mode riêng
> - Hiểu 1 lần, không cần tra lại
> Format: 3-5 dòng + backlink nếu có liên quan
>
> **Tier 2 — Standalone File** (tách ra `[tech]--[concept].md`)
> Dấu hiệu nhận biết:
> - Có operational behavior riêng (start/stop/rotate/expire...)
> - Có config quan trọng cần nhớ
> - Có security implication độc lập
> - Mày sẽ Google lại nó → thay bằng tra note của mày
> Template file con: xem bên dưới

**Cách đặt tên file:** `[tech]--[concept].md`  
Ví dụ: `vault--seal-unseal.md`, `vault--lease-renew-revoke.md`

```markdown
# [Tech] — [Concept Name]
Tier: 2
Parent: [[tech-root-note]]
Related: [[concept-a]], [[concept-b]]   ← thêm cái này
Tags: #vault #concept

## What it does
## Why it exists
## How it works (flow/diagram)
## Config gotchas
## Security notes
## Refs
```
