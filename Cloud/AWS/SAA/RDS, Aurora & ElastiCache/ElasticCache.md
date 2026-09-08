• **ElastiCache** is to get managed **Redis or Memcached**
• Caches are in-memory databases with really high performance, low latency
• Helps reduce load off of databases for read intensive workloads
• Helps make your application stateless
• AWS takes care of OS maintenance / patching, optimizations, setup, configuration, monitoring, failure recovery and backups
**• Using ElastiCache involves heavy application code changes**

## Solution Architecture - DB Cache
• Applications queries ElastiCache, if not available, get from RDS and store in ElastiCache.
• Helps relieve load in RDS
• Cache must have an invalidation strategy to make sure only the most current data is used in there.

![[Pasted image 20260316225703.png]]

## Solution Architecture – User Session Store 
• User logs into any of the application
• The application writes the session data into ElastiCache
• The user hits another instance of our application
• The instance retrieves the data and the user is already logged in

![[Pasted image 20260316225845.png]]

## ElastiCache – Redis vs Memcached
### REDIS

| Feature                      | Ý nghĩa thực tế                                                   |
| ---------------------------- | ----------------------------------------------------------------- |
| **Multi AZ + Auto-Failover** | Primary die → replica tự lên làm primary. Downtime gần như 0      |
| **Read Replicas**            | Scale read horizontally, app đọc từ replica, giảm tải primary     |
| **AOF Persistence**          | Ghi log từng write op xuống disk → restart không mất data         |
| **Backup & Restore**         | Snapshot định kỳ, restore được khi cần                            |
| **Sets / Sorted Sets**       | Data structure phong phú: leaderboard, queue, pub/sub, session... |

### MEMCACHED 

|Feature|Ý nghĩa thực tế|
|---|---|
|**Multi-node sharding**|Data được phân mảnh ra nhiều node → scale out dễ|
|**No HA / No replication**|Node die = data gone. Không có failover|
|**Non-persistent**|Restart = mất sạch. Chỉ là RAM cache thuần|
|**Backup (Serverless only)**|Chỉ Memcached Serverless mới có backup, classic thì không|
|**Multi-threaded**|Tận dụng nhiều CPU core → throughput cao hơn Redis single-thread|
## ElastiCache – Cache Security
• ElastiCache supports **IAM Authentication for Redis**
• **IAM policies on ElastiCache** are only used for **AWS API-level** security
**• Redis AUTH**
	• You can set a “password/token” when you create a Redis cluster
	• This is an extra level of security for your cache (on top of security groups)
	• Support SSL in flight encryption
• Memcached
	• Supports SASL-based authentication (advanced)

## Patterns for ElastiCache
• **Lazy Loading**: all the read data is cached, data can become stale in cache
• **Write Through**: Adds or update data in the cache when written to a DB (no stale data)
• **Session Store**: store temporary session data in a cache (using TTL features)
• Quote: There are only two hard things in Computer Science: cache invalidation and naming things

![[Pasted image 20260316231044.png]]

![[Pasted image 20260316231128.png]]

