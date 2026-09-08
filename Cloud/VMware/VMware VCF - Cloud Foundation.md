VMware Cloud Foundation (VCF) là một **giải pháp tích hợp toàn diện** để triển khai và quản lý **hạ tầng trung tâm dữ liệu định nghĩa bằng phần mềm (SDDC)**. Nó kết hợp các sản phẩm chủ lực của VMware thành một nền tảng duy nhất, giúp bạn xây dựng **private cloud hoặc hybrid cloud** một cách dễ dàng.

- VCF là giải pháp **SDDC tích hợp toàn diện**, bao gồm:
    - **vSphere**: Compute virtualization
    - **vSAN**: Storage virtualization
    - **NSX-T**: Network & Security virtualization
    - **SDDC Manager**: Tự động hoá triển khai, vận hành, nâng cấp
- VCF hỗ trợ **hybrid cloud** (on-prem + public cloud như VMware Cloud on AWS)

## Các thành phần
| Thành phần            | Mô tả                                                              |
| --------------------- | ------------------------------------------------------------------ |
| **Management Domain** | Domain chứa SDDC Manager, vCenter, NSX Manager, vSAN               |
| **Workload Domain**   | Chứa workload VM, có thể chia ra thành các WLD (VI WLD)            |
| **SDDC Manager**      | Công cụ trung tâm để triển khai, cập nhật và quản lý toàn bộ stack |



The following VMware software components may be optionally deployed as part of VMware Cloud Foundation: 
- **VMware Aria Operations** - Correlates data from applications to storage in a unified, easy-to-use management tool that provides control over performance, capacity, and configuration, with predictive analytics driving proactive action, and policy based automation. 
- **VMware Aria Automation** - Automates the delivery of the compute, storage, and network resources on a per-application basis, delivered through repeatable blueprints and accessed through a self-service user portal. •VMware Aria Operations for Logs – Allows administrators to view, manage, and analyze log information from various points within the solution. 
- **VMware Data Services Manager** - Provides a data-as-a-service toolkit for on-demand provisioning and automated management of PostgreSQL and MySQL databases in vSphere environment

| VCF 5.x Component                                      | VCF 9.0 Component                        |
|--------------------------------------------------------|------------------------------------------|
| VMware Cloud Builder                                   | VCF Installer                            |
| SDDC Manager                                           | SDDC Manager                             |
| VMware Aria Operations                                 | VCF Operations                           |
| VMware Aria Operations for Logs                        | VCF Operations for Logs                  |
| VMware Aria Operations for Networks                    | VCF Operations for Networks              |
| VMware Aria Lifecycle                                  | VCF Operations fleet management          |
| Workspace ONE Access                                   | -                                        |
| -                                                      | VCF Identity Broker                      |
| VMware HCX                                             | VCF Operations HCX                       |
| VMware Aria Automation                                 | VCF Automation                           |
| VMware Aria Automation Orchestrator                    | VCF Operations Orchestrator              |
| VMware Cloud Director                                  | VCF Automation                           |
| VMware vSphere with Tanzu / VMware IaaS control plane  | vSphere Supervisor                       |

## Workload domain
| Loại                            | Mô tả                                                                                               | Ưu điểm                                              | Hạn chế                                                          |
| ------------------------------- | --------------------------------------------------------------------------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------------- |
| **Management Domain**           | Domain đầu tiên được triển khai, chứa các thành phần quản lý như vCenter, NSX Manager, SDDC Manager | Tách biệt quản lý khỏi workload, dễ bảo trì          | Cần phần cứng riêng, có thể chưa tận dụng hết tài nguyên ban đầu |
| **Consolidated Domain**         | Kết hợp cả quản lý và workload trong cùng một domain                                                | Tiết kiệm phần cứng, phù hợp môi trường nhỏ          | Không tách biệt quản lý và workload, khó mở rộng                 |
| **VI Workload Domain**          | Domain riêng để chạy workload khách hàng, chia sẻ SSO với domain quản lý                            | Quản lý tập trung, lifecycle độc lập                 | Không tách biệt SSO cho từng khách hàng                          |
| **Isolated VI Workload Domain** | Giống VI Workload Domain nhưng có SSO riêng biệt                                                    | Tách biệt hoàn toàn giữa các khách hàng, bảo mật cao | Quản lý phức tạp hơn, nhiều vCenter riêng biệt                   |

**A workload domain** can consist of one or more vSphere clusters, provisioned automatically by SDDC Manager. Each workload domain contains the following components:
- ESXi hosts
- One VMware vCenter Server™ instance
- At least one vSphere cluster with vSphere HA and vSphere DRS enabled.
- One vSphere Distributed Switch per cluster for system traffic and NSX segments for workloads.
- One NSX Manager cluster for configuring and implementing software-defined networking.
- One NSX Edge cluster, added after you create the workload domain, that connects the workloads in the workload domain for logical switching, logical dynamic routing, and load balancing.
- One or more shared storage allocations.
**VMware Cloud Foundation** supports two types of workload domains 
- *The management domain* 
- *Virtual infrastructure* (VI) workload domains.
### Management Domain
The management domain is created during the bring-up process by *VMware Cloud Builder* and contains the VMware Cloud Foundation management components as follows:
- Minimum four ESXi hosts
- An instance of vCenter Server
- A three-node NSX Manager cluster
- SDDC Manager
- vSAN datastore
- One or more vSphere clusters each of which can scale up to the vSphere maximum of 64
### VI Workload Domains

- A `VI workload domain` consists of one or more vSphere clusters
- SDDC Manager automates the creation of the VI workload domain
- SDDC Manager deploys an additional vCenter Server instance

> [!note]
>New VI workload domains can share the same NSX Manager cluster with an existing VI workload domain or you can deploy a new NSX Manager cluster. VI workload domains cannot use the NSX Manager cluster for the management domain.







## Thuật ngữ
- **SDDC (Software-Defined Data Center)**:
    - Trung tâm dữ liệu được định nghĩa bằng phần mềm, tích hợp ảo hóa tính toán (vSphere), lưu trữ (vSAN), và mạng (NSX) trong một nền tảng thống nhất.
    - VCF là triển khai thực tế của SDDC.
- **SDDC Manager**:
    - Công cụ quản lý trung tâm của VCF, chịu trách nhiệm triển khai, cấu hình, và quản lý vòng đời (lifecycle management) của các thành phần VCF.
    - Quản lý **Management Domain** và **Workload Domains**.
- **Management Domain**:
    - Cụm (cluster) đầu tiên được triển khai trong VCF, chứa các thành phần quản lý như vCenter Server, NSX Manager, SDDC Manager, và vRealize Suite.
    - Yêu cầu tối thiểu: 4 host ESXi, vSAN, và NSX-T.
- **Workload Domain**:
    - Các cụm riêng biệt để chạy ứng dụng (VM hoặc container).
    - Có hai loại:
        - **VI Workload Domain**: Dành cho ứng dụng truyền thống (VM-based).
        - **Tanzu Workload Domain**: Dành cho ứng dụng containerized (Kubernetes-based).
- **Bring-Up Process**:
    - Quy trình khởi tạo và triển khai VCF, bao gồm cấu hình Management Domain và các thành phần cốt lõi (vSphere, vSAN, NSX).
- **Bill of Materials (BOM)**:
    - Danh sách các thành phần phần mềm (vSphere, vSAN, NSX, vRealize) và phiên bản tương thích được sử dụng trong một bản VCF cụ thể.
- **Domain**:
    - Một nhóm tài nguyên (compute, storage, networking) được quản lý riêng biệt trong VCF.
    - Bao gồm **Management Domain** và **Workload Domains**.
- **Cloud Builder**:
    - Công cụ tự động hóa được sử dụng để triển khai Management Domain trong quá trình cài đặt VCF.
- **Workload**:
    - Ứng dụng hoặc dịch vụ chạy trên VCF, có thể là máy ảo (VM) hoặc container (Kubernetes).

## Function areas
VCF Operations has the following main functional areas:
- **Fleet management**. Enables operational consistency and efficient resource management of the VCF infrastructure at scale.
	*By using the fleet management functionality of VCF Operations, you can manage centrally your VCF components.*
	- License Management
	- Lifecycle Management
	- Identity and Access Management
	- Certificate Management
	- Password Management
	- Configuration Management
	- Tag Management
- **Operations management**. Provides monitoring and optimization of performance, cost and capacity, and faster troubleshooting.
- **Workload operations**. Ensures that critical applications are running as expected.
- **Performance monitoring**. Ensures applications have continuous access to resources
- **FinOps and capacity**. Helps you analyze your infrastructure expenses and optimize capacity usage.
- **Workload mobility**. Provides support for migrating and interconnecting workloads within and across VCF private cloud.
- **Security management**. Provides security management to ensure that your VCF private cloud is operationally secure.
