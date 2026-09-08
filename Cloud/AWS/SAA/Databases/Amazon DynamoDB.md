## What is DynamoDB
• Fully managed, highly available with replication across multiple AZs
• NoSQL database - not a relational database - with transaction support
• Scales to massive workloads, distributed database
• Millions of requests per seconds, trillions of row, 100s of TB of storage
• Fast and consistent in performance (single-digit millisecond)
• Integrated with IAM for security, authorization and administration
• Low cost and auto-scaling capabilities
• No maintenance or patching, always available
• Standard & Infrequent Access (IA) Table Class

### Basics
• DynamoDB is made of **Tables**
• Each table has a **Primary Key** (must be decided at creation time)
• Each table can have an infinite number of items (= rows)
• Each item has **attributes** (can be added over time – can be null)
• Maximum size of an item is **400KB**
• Data types supported are:
• **Scalar Types** – String, Number, Binary, Boolean, Null
• **Document Types** – List, Map
• **Set Types** – String Set, Number Set, Binary Set
**• Therefore, in DynamoDB you can rapidly evolve schemas**

## Read/Write Capacity Modes
• Control how you manage your table’s capacity (read/write throughput)

**• Provisioned Mode (default)**
	• You specify the number of reads/writes per second
	**• You need to plan capacity beforehand**
	• Pay for **provisioned** Read Capacity Units (RCU) & Write Capacity Units (WCU)
	• Possibility to add **auto-scaling** mode for RCU & WCU
**• On-Demand Mode**
• Read/writes automatically scale up/down with your workloads
• No capacity planning needed
• Pay for what you use, more expensive (\$\$\$)
• Great for **unpredictable** workloads, **steep sudden spikes**

## DynamoDB Accelerator (DAX)
• Fully-managed, highly available, seamless in - memory cache for DynamoDB
**• Help solve read congestion by caching**
**• Microseconds latency for cached data**
• Doesn’t require application logic modification
(compatible with existing DynamoDB APIs)
• 5 minutes TTL for cache (default)

![[Pasted image 20260329120717.png | 600]]

### So sánh thẳng thắn

| |DAX|ElastiCache|
|---|---|---|
|Cache type|Item-level, Query/Scan|Bất kỳ thứ gì|
|Manage cache logic|Tự động|App tự viết|
|Dùng được với DB khác|❌ DynamoDB only|✅|
|Aggregation cache|❌ Kém|✅ Tốt hơn|
|Latency|Microseconds|Sub-millisecond|

---

### 👉 Khi nào dùng cái nào?

- **DAX** → read-heavy workload, cần giảm tải DynamoDB, không muốn viết cache logic
- **ElastiCache** → cần cache **computed/aggregated data**, hoặc app không chỉ dùng DynamoDB

Nhiều hệ thống production dùng **cả hai** — DAX cho item cache, ElastiCache cho session/aggregation.
## DynamoDB – Stream Processing
• Ordered stream of item-level modifications (create/update/delete) in a table
**• Use cases:**
	• React to changes in real-time (welcome email to users)
	• Real-time usage analytics
	• Insert into derivative tables
	• Implement cross-region replication
	• Invoke AWS Lambda on changes to your DynamoDB table

| **DynamoDB Streams**                                                     | **Kinesis Data Streams (newer)**                                                                      |
| ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| 24 hours retention                                                       | 1 year retention                                                                                      |
| Limited # of consumers                                                   | High # of consumers                                                                                   |
| Process using AWS Lambda Triggers, or<br>DynamoDB Stream Kinesis adapter | Process using AWS Lambda, Kinesis Data<br>Analytics, Kineis Data Firehose, AWS Glue<br>Streaming ETL… |
![[Pasted image 20260329160642.png]]