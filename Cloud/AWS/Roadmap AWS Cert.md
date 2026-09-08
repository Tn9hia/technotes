# AWS Certification Roadmap
> **Target role:** DevOps Engineer / Cloud Engineer / System Engineer
> **Background:** CKA certified, VMware/Linux sysadmin, đang học Terraform + Ansible
> **Source:** [AWS Certification Paths (official PDF)](https://d1.awsstatic.com/training-and-certification/docs/AWS_certification_paths.pdf)

---

## Bức tranh toàn cảnh AWS Certifications

```
FOUNDATIONAL          ASSOCIATE                    PROFESSIONAL
─────────────────────────────────────────────────────────────────────
                  ┌─ SAA-C03 (Solutions Architect) ──► SAP-C02 (SA Pro)
CLF-C02           ├─ SOA-C02 (SysOps Admin)        ─┐
(Cloud            └─ DVA-C02 (Developer)            └► DOP-C02 (DevOps Pro) ◄── TARGET
Practitioner)
                            SPECIALTY (chọn sau)
                  ┌─ ANS-C01 (Advanced Networking)
                  ├─ SCS-C02 (Security)
                  ├─ MLS-C01 (Machine Learning)
                  ├─ DBS-C01 (Database)
                  └─ DAS-C01 (Data Analytics)
```

---

## 🎯 Recommended Path — DevOps / Cloud / System Engineer

```
[Bạn hiện tại]
CKA ✅ + VMware + Linux
        │
        ▼
┌───────────────────┐
│  CLF-C02          │  ← SKIP nếu đã có background IT/cloud
│  Cloud Practitioner│  ← Nên làm để nắm big picture AWS
│  ~1-2 tháng       │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  SAA-C03          │  ← QUAN TRỌNG NHẤT — mọi DevOps đều cần
│  Solutions        │  ← Hiểu architecture AWS toàn diện
│  Architect Assoc  │  ← Prerequisites cho cả SAP & DOP
│  ~2-3 tháng       │
└────────┬──────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌─────────┐  ┌──────────────┐
│ SOA-C02 │  │   DVA-C02    │
│ SysOps  │  │  Developer   │  ← Chọn 1 trong 2 (hoặc cả 2)
│ Admin   │  │  Associate   │
│~2 tháng │  │  ~2 tháng    │
└────┬────┘  └──────┬───────┘
     └──────┬───────┘
            │ (cần 1 trong 2 Associate + SAA)
            ▼
┌───────────────────────┐
│  DOP-C02              │  ◄── 🏆 END GOAL
│  DevOps Engineer      │
│  Professional         │
│  ~3-4 tháng           │
└───────────────────────┘
```

**Tổng thời gian ước tính:** 8-12 tháng (nếu học nghiêm túc song song với đi làm)

---

## Chi tiết từng chứng chỉ

### 1. CLF-C02 — AWS Certified Cloud Practitioner
| | |
|---|---|
| **Level** | Foundational |
| **Exam code** | CLF-C02 |
| **Thời gian thi** | 90 phút |
| **Số câu** | 65 câu |
| **Điểm đậu** | 700/1000 |
| **Giá** | $100 USD |
| **Hiệu lực** | 3 năm |

**Nên học nếu:** Chưa có background AWS, cần nắm big picture trước.
**Có thể skip nếu:** Đã quen với cloud concepts và dùng AWS rồi.

**Topics chính:**
- Cloud concepts: IaaS, PaaS, SaaS, HA, fault tolerance
- AWS core services: EC2, S3, RDS, VPC, IAM, Route53, CloudFront
- AWS pricing model: On-demand, Reserved, Spot, Savings Plans
- AWS shared responsibility model
- AWS support plans

**Resources:**
- [AWS Skill Builder — Cloud Practitioner Essentials (free)](https://skillbuilder.aws/learn/course/134/aws-cloud-practitioner-essentials)
- [Stephane Maarek — Udemy CLF-C02](https://www.udemy.com/course/aws-certified-cloud-practitioner-new/)
- [ExamTopics — CLF-C02](https://www.examtopics.com/exams/amazon/aws-certified-cloud-practitioner/)

---

### 2. SAA-C03 — AWS Certified Solutions Architect – Associate ⭐ PRIORITY
| | |
|---|---|
| **Level** | Associate |
| **Exam code** | SAA-C03 |
| **Thời gian thi** | 130 phút |
| **Số câu** | 65 câu |
| **Điểm đậu** | 720/1000 |
| **Giá** | $150 USD |
| **Hiệu lực** | 3 năm |

**Tại sao cần:** Đây là cert quan trọng nhất trong ecosystem AWS. Là prerequisite cho cả SAP-C02 lẫn DOP-C02. Mọi Cloud/DevOps Engineer đều nên có.

**Topics chính:**
- **Compute:** EC2 (instance types, purchasing options, AMI), Auto Scaling, ECS, EKS, Lambda, Fargate
- **Storage:** S3 (storage classes, lifecycle, replication, encryption), EBS, EFS, FSx, Storage Gateway, Snow Family
- **Database:** RDS (Multi-AZ, Read Replica), Aurora, DynamoDB, ElastiCache, Redshift
- **Networking:** VPC (subnets, route tables, IGW, NAT, NACL vs SG), VPN, Direct Connect, Transit Gateway, VPC Peering, PrivateLink
- **High Availability:** ELB (ALB, NLB, CLB), Route 53 (routing policies), CloudFront, Global Accelerator
- **Security:** IAM (policies, roles, STS, federation), KMS, Secrets Manager, ACM, WAF, Shield, GuardDuty, Inspector
- **Monitoring:** CloudWatch (metrics, logs, alarms, Events/EventBridge), CloudTrail, AWS Config
- **Integration:** SQS, SNS, Kinesis, API Gateway
- **Architecture patterns:** Well-Architected Framework (6 pillars), Disaster Recovery strategies

**Resources:**
- [Stephane Maarek — SAA-C03 (Udemy)](https://www.udemy.com/course/aws-certified-solutions-architect-associate-saa-c03/) ← Best
- [Adrian Cantrill — SAA-C03](https://learn.cantrill.io/p/aws-certified-solutions-architect-associate) ← Deep dive
- [Tutorials Dojo Practice Exams](https://tutorialsdojo.com/courses/aws-certified-solutions-architect-associate-practice-exams/)
- [ExamTopics SAA-C03](https://www.examtopics.com/exams/amazon/aws-certified-solutions-architect-associate/)

---

### 3a. SOA-C02 — AWS Certified SysOps Administrator – Associate ⭐ HIGHLY RELEVANT
| | |
|---|---|
| **Level** | Associate |
| **Exam code** | SOA-C02 |
| **Thời gian thi** | 180 phút |
| **Số câu** | 65 câu + lab thực hành |
| **Điểm đậu** | 720/1000 |
| **Giá** | $150 USD |
| **Hiệu lực** | 3 năm |

**Tại sao cần:** Phù hợp nhất với background System Engineer của bạn. Covers monitoring, automation, operations — daily work của DevOps.

**Topics chính:**
- **Monitoring & Reporting:** CloudWatch (custom metrics, composite alarms, dashboards), CloudTrail, AWS Config rules, Health Dashboard
- **Reliability:** Auto Scaling (lifecycle hooks, warm pools), ELB health checks, RDS failover, Aurora failover, Elastic Disaster Recovery
- **Deployment & Provisioning:** CloudFormation (drift detection, StackSets, nested stacks), Systems Manager (SSM Agent, Patch Manager, Run Command, Session Manager, Parameter Store), Elastic Beanstalk
- **Security & Compliance:** IAM (password policies, access analyzer), KMS key rotation, Macie, Security Hub, Config conformance packs
- **Networking:** Route 53 (health checks, failover routing), VPC Flow Logs, Transit Gateway, Network Firewall
- **Storage:** S3 (versioning, MFA delete, object lock, replication), EBS snapshots lifecycle, EFS backup
- **Cost Optimization:** Cost Explorer, Budgets, Savings Plans, Trusted Advisor

**⚠️ Lưu ý:** SOA-C02 có phần **exam lab** (thực hành trực tiếp trên AWS Console/CLI trong 20 phút). Phải thực hành hands-on nhiều.

**Resources:**
- [Stephane Maarek — SOA-C02 (Udemy)](https://www.udemy.com/course/ultimate-aws-certified-sysops-administrator-associate/)
- [Tutorials Dojo Practice Exams — SOA-C02](https://tutorialsdojo.com/courses/aws-certified-sysops-administrator-associate-practice-exams/)

---

### 3b. DVA-C02 — AWS Certified Developer – Associate
| | |
|---|---|
| **Level** | Associate |
| **Exam code** | DVA-C02 |
| **Thời gian thi** | 130 phút |
| **Số câu** | 65 câu |
| **Điểm đậu** | 720/1000 |
| **Giá** | $150 USD |

**Tại sao cần:** Prerequisite cho DOP-C02. Covers CI/CD tools native AWS (CodePipeline, CodeBuild, CodeDeploy) — rất quan trọng cho DevOps path.

**Topics chính:**
- **Development:** Lambda (event sources, layers, versions/aliases, concurrency), API Gateway, DynamoDB (streams, DAX, GSI/LSI), SQS/SNS, Kinesis
- **Security:** Cognito, STS AssumeRole, KMS SDK encryption, Secrets Manager vs Parameter Store
- **Deployment:** Elastic Beanstalk (deployment policies: all-at-once, rolling, rolling with batch, immutable, blue/green), ECS (task definitions, service discovery)
- **CI/CD:** CodeCommit, CodeBuild (buildspec.yml), CodeDeploy (appspec.yml, deployment groups, lifecycle hooks), CodePipeline
- **Monitoring:** X-Ray (tracing, sampling rules, annotations vs metadata), CloudWatch Logs Insights
- **IaC:** CloudFormation (SAM — Serverless Application Model)

**Resources:**
- [Stephane Maarek — DVA-C02 (Udemy)](https://www.udemy.com/course/aws-certified-developer-associate-dva-c01/)

---

### 4. DOP-C02 — AWS Certified DevOps Engineer – Professional 🏆
| | |
|---|---|
| **Level** | Professional |
| **Exam code** | DOP-C02 |
| **Thời gian thi** | 180 phút |
| **Số câu** | 75 câu |
| **Điểm đậu** | 750/1000 |
| **Giá** | $300 USD |
| **Hiệu lực** | 3 năm |
| **Prerequisites** | SAA-C03 + (SOA-C02 hoặc DVA-C02) |

**Tại sao cần:** Đây là chứng chỉ cao nhất dành riêng cho DevOps trên AWS. Là mục tiêu chính của path này.

**Topics chính:**
- **SDLC Automation (22%):** CodePipeline (complex workflows, cross-account), CodeBuild (caching, test reports, Docker builds), CodeDeploy (all deployment types: EC2, Lambda, ECS), CodeStar, testing strategies (canary, blue/green, A/B)
- **Config Management & IaC (17%):** CloudFormation (advanced: Custom Resources, macros, StackSets across accounts/regions, drift), Systems Manager (OpsCenter, Automation documents, State Manager), AWS CDK
- **Resilient Cloud Solutions (15%):** Multi-region architecture, Auto Scaling (predictive, scheduled, step), ELB (connection draining, slow start), RDS Multi-AZ + cross-region read replica, DynamoDB global tables, Route 53 failover
- **Monitoring & Logging (15%):** CloudWatch (Contributor Insights, Synthetics, Container Insights), AWS X-Ray distributed tracing, OpenSearch (ELK on AWS), Kinesis Data Firehose to S3/ES
- **Incident & Event Response (14%):** EventBridge rules, Systems Manager Incident Manager, Auto Scaling lifecycle hooks, Runbook automation, AWS Health events
- **Security & Compliance (17%):** SCPs (Service Control Policies), AWS Organizations, IAM boundary policies, Config rules (managed + custom), Security Hub standards, Inspector v2, GuardDuty findings automation

**Resources:**
- [Stephane Maarek — DOP-C02 (Udemy)](https://www.udemy.com/course/aws-certified-devops-engineer-professional-hands-on/)
- [Tutorials Dojo Practice Exams — DOP-C02](https://tutorialsdojo.com/courses/aws-certified-devops-engineer-professional-practice-exams/)
- [Adrian Cantrill — DevOps Pro](https://learn.cantrill.io/p/aws-certified-devops-engineer-professional)

---

## Specialty — Học sau khi có DOP-C02

### ANS-C01 — Advanced Networking Specialty
> Phù hợp nếu muốn đi sâu vào networking: VPN, Direct Connect, Transit Gateway, hybrid networking
> **Recommend cho:** Network Engineer / Cloud Architect

### SCS-C02 — Security Specialty
> Phù hợp nếu muốn chuyển hướng DevSecOps
> Cover: IAM advanced, KMS, CloudTrail forensics, incident response, compliance frameworks
> **Recommend cho:** DevSecOps / Security Engineer

---

## Timeline thực tế

| Giai đoạn | Cert | Thời gian | Ghi chú |
|---|---|---|---|
| Q2 2026 | CLF-C02 | 1-1.5 tháng | Warm-up, nắm big picture |
| Q3 2026 | SAA-C03 | 2-3 tháng | Core cert — invest time vào đây |
| Q4 2026 | SOA-C02 | 2 tháng | Gần với daily work nhất |
| Q1 2027 | DVA-C02 | 1.5-2 tháng | Có thể skip nếu không cần |
| Q2-Q3 2027 | DOP-C02 | 3-4 tháng | End goal 🏆 |

---

## Tips học AWS hiệu quả

### Hands-on là bắt buộc
- Tạo AWS Free Tier account (12 tháng miễn phí nhiều services)
- Setup billing alert tại $5 để tránh bill bất ngờ
- Dùng `aws-nuke` hoặc tự cleanup resources sau khi lab

### Học đúng thứ tự
```
Lý thuyết (video/docs) → Lab tay → Practice exam → Review sai → Thi
```

### Practice exam strategy
- Làm practice exam khi đã học xong → xem điểm
- Review kỹ **từng câu sai** — đọc explanation
- Làm lại đến khi đạt 80%+ practice → book thi thật
- Tutorials Dojo > ExamTopics (explanations tốt hơn)

### Tận dụng AWS Skill Builder
- [skillbuilder.aws](https://skillbuilder.aws) — nhiều content free
- Exam Prep Official Practice Question Sets — free
- Official exam readiness course — free

---

## Quick Reference — Exam Codes

| Cert | Code | Level | Giá |
|---|---|---|---|
| Cloud Practitioner | CLF-C02 | Foundational | $100 |
| Solutions Architect Associate | SAA-C03 | Associate | $150 |
| Developer Associate | DVA-C02 | Associate | $150 |
| SysOps Administrator Associate | SOA-C02 | Associate | $150 |
| Solutions Architect Professional | SAP-C02 | Professional | $300 |
| **DevOps Engineer Professional** | **DOP-C02** | **Professional** | **$300** |
| Advanced Networking Specialty | ANS-C01 | Specialty | $300 |
| Security Specialty | SCS-C02 | Specialty | $300 |
| Machine Learning Specialty | MLS-C01 | Specialty | $300 |
| Database Specialty | DBS-C01 | Specialty | $300 |

---

*Last updated: March 2026 | Source: [AWS Certification Paths PDF](https://d1.awsstatic.com/training-and-certification/docs/AWS_certification_paths.pdf)*
