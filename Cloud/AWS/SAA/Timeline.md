## Lịch học đề xuất (17 ngày)

```
Tuần 1 (16/3 - 22/3): Core foundations
├── Day 1-2: IAM, Organizations, SCP
├── Day 3-4: EC2, EBS, AMI, Auto Scaling
├── Day 5-6: VPC deep dive (quan trọng nhất)
└── Day 7: S3 + Storage Gateway

Tuần 2 (23/3 - 29/3): Services breadth
├── Day 8-9: RDS, Aurora, DynamoDB, ElastiCache
├── Day 10: Route53, CloudFront
├── Day 11: ELB (ALB/NLB), API Gateway
├── Day 12: SQS, SNS, EventBridge
└── Day 13: ECS, EKS basics, Lambda

Tuần 3 (30/3 - 2/4): Exam prep
├── Day 14: Security — KMS, Secrets Manager, WAF, Shield
├── Day 15: Monitoring — CloudWatch, CloudTrail, Config
├── Day 16: Practice exam 1 + review sai
└── Day 17: Practice exam 2 + cram weak spots
```

---

## Resource stack — không cần mua lung tung

**Primary:**

- 🎯 **Stephane Maarek** (Udemy) — best instructor cho SAA, mua lúc sale ~$15
- 📝 **Tutorials Dojo practice exams** — ~$15, cực kỳ sát đề thật

**Free supplement:**

- AWS Skill Builder (free tier) — official but dry
- **Jon Bonso** cheat sheets trên Tutorials Dojo

**Không cần:**

- AWS Whitepapers (quá dài, không worth với timeline này)
- Sách giấy

---

## Danger zones mày cần chú ý

```
HIGH WEIGHT topics trong SAA-C03:
- VPC (subnets, routing, NAT, peering, endpoints)
- IAM (policies, roles, cross-account)
- S3 (storage classes, lifecycle, replication)
- RDS vs Aurora vs DynamoDB — khi nào dùng cái nào
- Disaster Recovery patterns (Pilot Light, Warm Standby...)
```

⚠️ Đây là exam **associate level** nhưng scenario-based — không hỏi "EC2 là gì" mà hỏi "scenario này nên dùng gì và tại sao". Đọc kỹ từng option.

---

## Verdict

**Học kịp không?** — **Có**, nếu:

- ⏱ Cam kết **3-4 tiếng/ngày** (workday) hoặc **5-6h** (weekend)
- Không học kiểu đọc lý thuyết — phải làm practice exam từ **ngày 10 trở đi**
- Target **>75% practice exam** trước khi thi (đề thật ~720/1000 là pass)

**Risk:** Nếu công việc ở Viettel IDC đang busy → timeline sẽ bị squeeze. Có thể dời sang **10/4** làm buffer thêm 1 tuần cũng không sao.

Mày đang học Terraform/Ansible song song không? Nếu có thì nên **tạm pause** mấy cái đó, focus 1 thứ trong 2.5 tuần này.