## What is AWS Kinesis
• Collect and store streaming data in **real-time**
![[Pasted image 20260326232841.png]]

• Retention between up to 365 days
• Ability to reprocess (replay) data by consumers
• Data can’t be deleted from Kinesis (until it expires)
• Data up to 10MiB (typical use case is lot of “small” **real-time data**)
• Data ordering guarantee for data with the same “Partition ID”
• At-rest KMS encryption, in-flight HTTPS encryption
• Kinesis Producer Library (KPL) to write an optimized producer application
• Kinesis Client Library (KCL) to write an optimized consumer application

### Capacity mode
• Provisioned mode:
	• Choose number of shards
	• Each shard gets 1MB/s in (or 1000 records per second)
	• Each shard gets 2MB/s out
	• Scale manually to increase or decrease the number of shards
	• You pay per shard provisioned per hour
• On-demand mode:
	• No need to provision or manage the capacity
	• Default capacity provisioned (4 MB/s in or 4000 records per second)
	• Scales automatically based on observed throughput peak during the last 30 days
	• Pay per stream per hour & data in/out per GB
## Amazon Data Firehose
Amazon Data Firehose (formerly Kinesis Data Firehose) is **a fully managed, serverless AWS service designed to reliably capture, transform, and load streaming data into data lakes, warehouses, and analytics tools**
![[Pasted image 20260328182113.png]]

*• Note: used to be called “Kinesis Data Firehose”*
• Fully Managed Service
	• Amazon Redshift / Amazon S3 / Amazon OpenSearch Service
	• 3rd party: Splunk / MongoDB / Datadog / NewRelic / …
	• Custom HTTP Endpoint
• Automatic scaling, serverless, pay for what you use
• **Near Real-Time** with buffering capability based on size / time
• Supports CSV, JSON, Parquet, Avro, Raw Text, Binary data
• Conversions to Parquet / ORC, compressions with gzip / snappy
• Custom data transformations using AWS Lambda (ex: CSV to JSON)

## Amazon Kinesis Data Streams & Amazon Data Firehose

| Kinesis Data Streams         | Amazon Data Firehose                                                             |
| ---------------------------- | -------------------------------------------------------------------------------- |
| Streaming data collection    | Load streaming data into S3 / Redshift /<br>OpenSearch / 3rd party / custom HTTP |
| Producer & Consumer code     | Fully managed                                                                    |
| Real-time                    | Near real-time                                                                   |
| Provisioned / On-Demand mode | Automatic scaling                                                                |
| Data storage up to 365 days  | No data store                                                                    |
| Replay Capability            | Doesn’t support replay capability                                                |

## Global Tables
• Make a DynamoDB table accessible with **low latency** in multiple-regions
• Active-Active replication
• Applications can **READ** and **WRITE** to the table in any region
• Must enable DynamoDB Streams as a pre-requisite

![[Pasted image 20260329160954.png | 500]]
## Time To Live (TTL)
• Automatically delete items after an expiryvtimestamp
• Use cases: reduce stored data by keeping only current items, adhere to regulatory obligations, web session handling…

## Backups for disaster recovery
**• Continuous backups using point-in-time recovery (PITR)**
	• Optionally enabled for the last 35 days
	• Point-in-time recovery to any time within the backup window
	• The recovery process creates a new table
**• On-demand backups**
	• Full backups for long-term retention, until explicitely deleted
	• Doesn’t affect performance or latency
	• Can be configured and managed in AWS Backup (enables cross-region copy)
	• The recovery process creates a new 
## Integration with Amazon S3
• Export to S3 (must enable PITR)
	• Works for any point of time in the last 35 days
	• Doesn’t affect the read capacity of your table
	• Perform data analysis on top of DynamoDB
	• Retain snapshots for auditing
	• ETL on top of S3 data before importing back into DynamoDB
	• Export in DynamoDB JSON or ION format
• Import from S3
	• Import CSV, DynamoDB JSON or ION format
	• Doesn’t consume any write capacity
	• Creates a new table
	• Import errors are logged in CloudWatch Logs

![[Pasted image 20260329161615.png | 500]]