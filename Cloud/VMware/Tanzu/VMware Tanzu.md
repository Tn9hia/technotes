**VMware Tanzu** là một nền tảng phát triển và vận hành ứng dụng hiện đại, giúp doanh nghiệp **xây dựng, triển khai, quản lý và mở rộng ứng dụng cloud-native** một cách nhanh chóng và nhất quán.

## Chức năng và các thành phần
### 🏗️ **1. Build – Dành cho Developer & App Platform Team**

|Thành phần|Vai trò|
|---|---|
|**Tanzu Application Platform (TAP)**|Tạo nền tảng developer portal: code → build → scan → deploy (CI/CD, Inner Loop, Outer Loop)|
|**Tanzu Build Service (TBS)**|Build image từ source code bằng Cloud Native Buildpacks (không cần Dockerfile)|
|**Tanzu Application Catalog (TAC)**|Thư viện app/code template bảo mật, ready-to-use (open-source đã harden)|
|**Tanzu Developer Tools**|Plugin cho IDE (IntelliJ, VS Code...) + CLI cho dev dễ push app|

---

### 🚀 **2. Run – Dành cho người vận hành Kubernetes**

|Thành phần|Vai trò|
|---|---|
|**Tanzu Kubernetes Grid (TKG)**|Triển khai Kubernetes clusters (trên vSphere, AWS, Azure...) chuẩn hóa từ VMware|
|**Tanzu Kubernetes Grid Integrated Edition (TKGI)**|(Trước là PKS) – Kubernetes tự động hóa + tích hợp sâu với NSX-T, vSphere|
|**Tanzu Application Service (TAS)**|PaaS như Cloud Foundry – `cf push` là chạy app luôn, không cần biết container|
|**Tanzu Service Mesh**|Mesh layer dựa trên Istio – quản lý microservices communication, security, traffic|

---

### 🛠️ **3. Manage – Quản lý, theo dõi, bảo mật mọi thứ**

|Thành phần|Vai trò|
|---|---|
|**Tanzu Mission Control (TMC)**|Trung tâm quản lý multi-cluster Kubernetes (lifecycle, backup, policy...)|
|**Tanzu Observability (Wavefront)**|Theo dõi metrics, logs, traces, dashboard cực chi tiết|
|**Tanzu Guardrails**|Thiết lập & enforce policy (security, compliance) cho workload & cluster|
|**Tanzu Insights / Intelligence Services**|Phân tích AI/ML cho hệ thống (sắp ra mắt/dang mở rộng)|

---

### 🧰 **4. Infrastructure & Integrations – Tích hợp với môi trường sẵn có**

|Thành phần|Vai trò|
|---|---|
|**vSphere with Tanzu**|Gắn Kubernetes trực tiếp vào vSphere (vSAN + NSX + vCenter)|
|**NSX Advanced Load Balancer (Avi)**|Load balancing và ingress controller cho app/container|
|**Velero + Backup integration**|Backup/restore clusters hoặc workload Kubernetes|
|**Harbor Registry**|Container image registry riêng, hỗ trợ scan, policy, replication|

---
## Lộ trình học
### 1. **Standalone Components** (Bắt buộc học trước)

- Bao gồm **Tanzu Kubernetes Grid (TKG)**, **Tanzu CLI**, **Tanzu Application Platform**, **Tanzu Mission Control**, **Tanzu Service Mesh**.  
    👉 Đây là core của Tanzu, vì nó liên quan trực tiếp đến Kubernetes cluster, quản lý, và vận hành.
    

> Nếu chưa hiểu TKG + Mission Control thì học các phần sau cũng hơi khó follow.

#### **1. Tanzu Kubernetes Grid (TKG)**

- Đây là **nền tảng Kubernetes runtime** của Tanzu.
    
- Bạn cần nắm:
    
    - Tạo cluster TKG trên vSphere/VMware Cloud.
        
    - Quản lý node pools, lifecycle (scale up/down, upgrade).
        
    - Cách nó khác so với kubeadm/vanilla K8s.
        

👉 **Quan trọng nhất, học đầu tiên.**

---

#### **2. Tanzu CLI**

- CLI là công cụ điều khiển toàn bộ stack (TKG, TAP, TMC).
    
- Thực hành:
    
    - Tạo cluster với `tanzu cluster create`.
        
    - Deploy workload.
        
    - Tích hợp với kubeconfig.
        

👉 **Đi kèm TKG, học song song.**

---

#### **3. Tanzu Mission Control (TMC)**

- Centralized management cho nhiều cluster (multi-cloud, multi-team).
    
- Bạn cần hiểu:
    
    - Policy management (RBAC, security).
        
    - Cluster lifecycle ops.
        
    - Tích hợp với vSphere + public cloud.
        

👉 **Học sau khi bạn đã biết deploy 1 cluster với TKG.**

---

#### **4. Cluster Essentials for Tanzu**

- Bộ công cụ open-source (Carvel, kapp, ytt, kbld).
    
- Đây là nền tảng bắt buộc để các dịch vụ khác (TAP, Build Service) hoạt động.  
    👉 Học căn bản (biết dùng, không cần đi quá sâu).
    

---

#### **5. Tanzu Application Platform (TAP)**

- Platform để build/deploy ứng dụng trên Kubernetes.
    
- Nắm:
    
    - Các component của TAP (supply chain, developer portal, workload management).
        
    - Cách TAP tích hợp với CI/CD pipelines.
        

👉 Nếu bạn thiên về **DevOps App Platform** thì học kỹ, còn thuần Infra thì chỉ cần overview.

---

#### **6. Tanzu Build Service + Buildpacks**

- Build Service = Tự động build image.
    
- Buildpacks = Framework/runtime support cho app.  
    👉 Đây là **CI/CD & developer productivity**, học sau TAP.
    

---

#### **7. Application Configuration Service**

- Dành cho Spring app config runtime trên Kubernetes.  
    👉 Nếu không phải Java dev thì **có thể skip**.
    

---

#### **8. Tanzu Service Mesh**

- Dựa trên Istio, cung cấp service-to-service communication, security, observability.  
    👉 Quan trọng nếu bạn làm **microservices architecture**. Nếu chưa đụng microservices nhiều thì học sau.
    

---




---

### 2. **Reference Architectures**

- Đây là các design đã được validate, kiểu như "best practice" để triển khai Tanzu.  
    👉 Học phần này để hiểu **cách ghép các mảnh ghép của Tanzu vào 1 môi trường thật**.
    

---

### 3. **Tanzu Platform**

- Cái này thiên về **Cloud Foundry + application delivery**.  
    👉 Nếu bạn quan tâm tới **DevOps/App Platform** thì học tiếp chỗ này.
    

---

### 4. **Bitnami Secure Images**

- Học để deploy app nhanh bằng **package open-source an toàn**.  
    👉 Tiện lợi khi test ứng dụng trên Tanzu, nhưng không phải core.
    

---

### 5. **Tanzu Spring**

- Dành cho Java devs. Nếu bạn không chuyên Java, có thể skip hoặc học sau.
    

---

### 6. **Tanzu Data**

- Bao gồm database & messaging services (RabbitMQ, MySQL, Postgres…).  
    👉 Học khi đã vững TKG + platform, vì nó thiên về **data services trên Tanzu**.
    

---

### 7. **Tanzu CloudHealth**

- Tập trung quản lý cost multi-cloud.  
    👉 Nếu bạn làm FinOps/CloudOps thì học, còn không thì để sau.
    

---

### 8. **Compliance Resources**

- Học cuối cùng. Tài liệu về security & compliance. Quan trọng khi triển khai production.