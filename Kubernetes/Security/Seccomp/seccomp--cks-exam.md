# Seccomp — Ghi chú cho kỳ thi CKS
Tier: 2
Parent: [[seccomp]]
Related: [[seccomp--kubernetes]], [[seccomp--sandbox-runtimeclass]], [[seccomp--ops-debug]]
Tags: #cks #exam
Last updated: 2026-09-06
Verified against: CKS_Curriculum v1.34.pdf — tải trực tiếp từ repo chính thức [cncf/curriculum](https://github.com/cncf/curriculum), đọc nguyên văn ngày 2026-09-06

## What it does

Định vị chính xác seccomp nằm ở đâu trong CKS curriculum chính thức, và những gì thực hành nên luyện tay trước khi thi — vì CKS là thi **performance-based** (làm trên cluster thật, không phải trắc nghiệm).

## Nguyên văn từ curriculum chính thức (v1.34, đọc trực tiếp từ PDF)

CKS Curriculum có 6 domain:

| Domain | Trọng số |
|---|---|
| Cluster Setup | 15% |
| Cluster Hardening | 15% |
| **System Hardening** | **10%** |
| Minimize Microservice Vulnerabilities | 20% |
| Supply Chain Security | 20% |
| Monitoring, Logging and Runtime Security | 20% |

Seccomp nằm nguyên văn trong domain **System Hardening (10%)**, dòng cuối:
> "Appropriately use kernel hardening tools such as AppArmor, seccomp"

Domain **Minimize Microservice Vulnerabilities (20%)** có mục liên quan gián tiếp:
> "Understand and implement isolation techniques (multi-tenancy, sandboxed containers, etc.)"
→ đây là chỗ gVisor/Kata/RuntimeClass thuộc về ([[seccomp--sandbox-runtimeclass]]), không phải seccomp trực tiếp nhưng cùng nhóm tư duy "kernel/isolation hardening" nên hay bị hỏi liên tiếp trong 1 câu.

**Lưu ý về version**: curriculum được CNCF cập nhật theo từng chu kỳ release Kubernetes (số phiên bản trong tên file trùng với version K8s, ví dụ file hiện tại là `v1.34`). Môi trường thi thực tế thường được cập nhật theo K8s minor version mới trong vòng vài tuần sau khi K8s release — **tự kiểm tra trên trang Linux Foundation tại thời điểm gần ngày thi** để biết chính xác version cluster sẽ gặp, đừng giả định cố định.

## Kỹ năng thực hành nên luyện tay (không suy đoán — dựa trên nội dung API/tutorial chính thức đã note ở [[seccomp--kubernetes]])

1. **Đọc và sửa 1 Pod spec để thêm `seccompProfile`** ở cả pod-level và container-level, hiểu đúng thứ tự override (container > pod).
2. **Tạo `Localhost` profile**: biết chính xác phải đặt file ở đâu trên node (`/var/lib/kubelet/seccomp/...`) — trong môi trường thi thường phải `ssh`/dùng `crictl`/copy file vào đúng node trước khi Pod chạy được.
3. **Đọc lỗi `CreateContainerError` từ `kubectl describe pod`** và suy ra nguyên nhân (thiếu file vs profile chặn syscall app cần) — xem quy trình debug ở [[seccomp--ops-debug]].
4. **Dùng `audit.json` / `violation.json` / fine-grained profile mẫu** của tutorial chính thức để hiểu nhanh sự khác biệt `SCMP_ACT_LOG` vs `SCMP_ACT_ERRNO` — rất dễ bị hỏi dạng "sửa profile này để nó chặn đúng 1 syscall X, cho phép còn lại".
5. **Phân biệt `Unconfined` / `RuntimeDefault` / `Localhost`** và biết khi nào PSS `restricted` sẽ reject Pod (thiếu hoặc `Unconfined`).
6. **Biết `RuntimeClass`** cú pháp cơ bản (`handler`, `runtimeClassName`) dù không cần cài thật gVisor/Kata trong lúc luyện — khả năng cao câu hỏi chỉ yêu cầu tạo/gán RuntimeClass, không yêu cầu cài runtime mới.

## Gotchas & Lessons Learned (khi luyện thi)

- Đề thi performance-based **chấm theo trạng thái cluster cuối cùng**, không chấm theo cách bạn làm — nếu quên field `localhostProfile` hoặc gõ sai path tương đối, Pod sẽ **không** báo lỗi syntax mà chỉ fail lúc container thực sự tạo (`CreateContainerError`) — luôn `kubectl get pod` để confirm `Running` trước khi chuyển câu tiếp theo, đừng chỉ tin `kubectl apply` chạy thành công.
- Nhớ **`localhostProfile` là path tương đối**, không phải path tuyệt đối tới file trên node — 1 lỗi rất dễ mắc khi đang vội trong lúc thi có giới hạn thời gian.
- Nếu đề bài yêu cầu "áp seccomp cho pod" mà không nói rõ profile cụ thể, `RuntimeDefault` gần như luôn là câu trả lời an toàn nhất/nhanh nhất — không cần viết profile JSON custom trừ khi đề yêu cầu rõ ràng chặn/cho phép 1 syscall cụ thể.
- Container `privileged: true` + yêu cầu áp seccomp trong cùng 1 câu hỏi là dấu hiệu đề đang test kiến thức "privileged luôn Unconfined bất kể khai gì" — đọc kỹ có phải đề đang bẫy hay không trước khi sửa sai chỗ khác.

## Refs
- [CKS Curriculum v1.34 — cncf/curriculum (chính thức, PDF)](https://github.com/cncf/curriculum)
- [Linux Foundation — CKS certification page](https://training.linuxfoundation.org/certification/certified-kubernetes-security-specialist/)
- [Kubernetes — Restrict a Container's Syscalls with seccomp (tutorial để luyện tay)](https://kubernetes.io/docs/tutorials/security/seccomp/)
