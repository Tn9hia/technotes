---
title: Packer
tags:
  - packer
  - iac
  - infrastructure-as-code
  - index
date: 2026-07-18
---

# Packer

Packer (HashiCorp) là tool để **build machine image** từ code — một template định nghĩa "cài gì, config gì" rồi Packer tạo ra image hoàn chỉnh (AMI, VirtualBox `.box`, Docker image, qcow2...) sẵn sàng dùng. Khác với Vagrant/Terraform vốn *dùng* image có sẵn để tạo instance, Packer là bước **đứng trước** — nó *tạo ra* cái image đó.

## Contents

| File | Nội dung |
|------|----------|
| [[Core Concepts]] | HCL2 template structure, `source`/`build` block, builders (amazon-ebs, virtualbox-iso, docker, qemu), variables, plugins |
| [[Provisioners & Post-Processors]] | Shell/Ansible provisioner, post-processors (vagrant, compress, manifest, docker-tag/push), multi-target build |
| [[CLI Reference]] | `packer init/build/validate/fmt/inspect`, env vars, debug recipes |

---

## Vấn đề Packer giải quyết

```
KHÔNG CÓ PACKER:
  vagrant up → chạy shell/ansible provisioner cài mọi thứ từ đầu
                → chậm (apt install, download dependencies mỗi lần)
                → không deterministic (mirror down, version trôi theo thời gian)

CÓ PACKER:
  packer build → bake sẵn OS + dependencies + config vào 1 image
                → vagrant up chỉ boot lên, hầu như tức thì
                → mọi máy dùng CHUNG 1 image → giống hệt nhau
```

**Ý tưởng cốt lõi — Immutable Infrastructure:** thay vì configure server sau khi tạo (provision-on-boot), bake toàn bộ config vào image trước, server chỉ cần boot lên là xong. Đổi version/config → build image mới, không patch image cũ.

---

## Quick Reference

```bash
packer init template.pkr.hcl      # tải plugin cần thiết (khai báo trong required_plugins)
packer fmt template.pkr.hcl       # format code
packer validate template.pkr.hcl  # syntax + logic check
packer build template.pkr.hcl     # build image thật sự
```

### Workflow tổng thể

```
template.pkr.hcl (HCL2)
    │
    ▼
packer build
    │
    ├─ Builder: tạo temporary instance/VM từ base ISO/AMI
    ├─ Provisioner: chạy shell/ansible để cài đặt, config (giống Vagrant provisioner)
    ├─ Post-processor: đóng gói kết quả (AMI, .box, docker image, compress...)
    └─ Cleanup: xoá temporary instance/VM
    │
    ▼
Output: image sẵn sàng dùng (AMI ID, .box file, docker tag...)
```

---

## Packer + Vagrant + Terraform — Combo thực tế

```
┌──────────┐   build image   ┌───────────────────┐
│  Packer  │ ───────────────►│  Image (.box/AMI) │
└──────────┘                  └─────────┬─────────┘
                                          │
                     ┌────────────────────┴────────────────────┐
                     ▼                                          ▼
              ┌─────────────┐                          ┌──────────────┐
              │  Vagrantfile │  config.vm.box = image  │  Terraform    │
              │  (local dev) │                          │ (production)  │
              └─────────────┘                          │ ami = image_id │
                                                          └──────────────┘
```

Một Packer template có thể build ra **nhiều target cùng lúc** từ cùng bộ provisioning script: `.box` cho dev (Vagrant) và AMI cho production (Terraform) — đảm bảo dev environment và production image chạy **cùng version, cùng config**, giảm "works on my machine".

---

## Cross-References

- Dùng image do Packer build làm `config.vm.box`: xem [[Vagrant]] → [[Core Concepts]] (mục Boxes)
- Terraform tham chiếu AMI ID do Packer build ra (thường qua `data "aws_ami"` filter theo tag/name): xem [[Terraform]] → [[Core Concepts]] (mục Data Sources)
- Provisioner Ansible dùng trong Packer giống hệt cú pháp dùng trong Vagrant: xem [[Ansible - Overview]]
