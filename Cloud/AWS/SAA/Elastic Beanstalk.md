• Elastic Beanstalk is a developer centric view of deploying an application on AWS
• It uses all the component’s we’ve seen before: EC2, ASG, ELB, RDS, …
**• Managed service**
• Automatically handles capacity provisioning, load balancing, scaling, application health monitoring, instance configuration, …
• Just the application code is the responsibility of the developer
• We still have full control over the configuration
**• Beanstalk is free but you pay for the underlying instances**

## Component
• **Application**: collection of Elastic Beanstalk components (environments,
versions, configurations, …)
• **Application Version**: an iteration of your application code
**• Environment**
	• Collection of AWS resources running an application version (only one application
	version at a time)
	• **Tiers**: Web Server Environment Tier & Worker Environment Tier
	• You can create multiple environments (dev, test, prod, …)
![[Pasted image 20260317001655.png]]

## Web Server Tier vs. Worker Tier
![[Pasted image 20260317004026.png]]

**Web Server Tier** — phục vụ HTTP request trực tiếp từ client. Có ELB phía trước, auto-scaling theo traffic, response trả về ngay lập tức. Dùng cho API, web app, bất kỳ thứ gì cần synchronous response.

**Worker Tier** — không có ELB, không expose ra internet. Nó poll message từ một SQS queue, xử lý xong rồi thôi. Dùng cho background jobs: resize ảnh, gửi email, ETL, long-running task. Classic producer-consumer pattern.

Sự khác biệt cốt lõi: Web Tier = "ai đó đang đợi response". Worker Tier = "xử lý khi rảnh, không ai đợi".

| |Web Tier|Worker Tier|
|---|---|---|
|Trigger|HTTP request|SQS message|
|ELB|Có|Không|
|Public endpoint|Có|Không|
|Response|Synchronous|Async (fire & forget)|
|Scale metric|Request count / latency|Queue depth|
|Use case|API, web app|Email, resize, ETL|

## Elastic Beanstalk Deployment Modes
![[Pasted image 20260317005030.png]]
**All at Once** — nhanh nhất, ngu nhất. Deploy xong là downtime. Chỉ dùng cho dev/test, production mà dùng cái này thì bị đồng nghiệp ghét.

- Rollback: redeploy lại version cũ → lại downtime thêm lần nữa
- Risk: 🔴 High

**Rolling** — deploy từng batch, capacity giảm trong lúc deploy. Vấn đề chính là **mixed version** đang chạy song song — nếu app không backward-compatible thì đây là bug đang chờ nổ.

- Rollback: rolling lại → chậm
- Risk: 🟡 Medium

**Rolling + Extra Batch** — giống Rolling nhưng spin thêm 1 batch instance trước, đảm bảo không mất capacity. Tốn thêm tiền nhưng xứng đáng hơn plain Rolling.

- Rollback: rolling lại
- Risk: 🟡 Medium-Low

**Immutable** — tạo ASG mới hoàn toàn với instance chạy v2. Health check xong mới swap vào ELB. Rollback chỉ cần terminate ASG mới → cực nhanh.

- Rollback: terminate new ASG, gần như instant
- Risk: 🟢 Low (recommended cho production)

**Blue/Green** — clone nguyên environment, test kỹ, rồi swap DNS. Không phải native Elastic Beanstalk feature, phải làm thủ công qua `Swap environment URLs`. Rollback cũng chỉ cần swap DNS lại.

- Rollback: swap DNS lại
- Risk: 🟢 Very Low (gold standard)

**Traffic Splitting (Canary)** — route một % traffic nhỏ sang v2, monitor lỗi, rồi tăng dần. Cần Elastic Beanstalk enhanced health reporting để observe đúng cách.

- Rollback: shift traffic về 100% v1
- Risk: 🟢 Low (best for gradual rollout)
