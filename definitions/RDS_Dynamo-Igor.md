# RDS, DynamoDB and How to Choose

## RDS

It is the "Relational Database Service," designed to operate databases in the AWS environment. It offers security features, integration with other AWS services and, most importantly, can scale the database by provisioning multiple instances, bringing one up when another fails. It has Multi-AZ features, so it can synchronize instances based on availability zones.

## Pros / Cons

| Aspect | Pros | Cons |
|--------|------|------|
| **Multi-AZ** | High availability and automatic failover | EXPENSIVE - Cost driven by instance class / database size |
| **Scalability** | Can provision multiple instances | Complex configuration |
| **Synchronization and Security** | Synchronous replication across availability zones | Requires management and monitoring |

## DynamoDB

It is, in other words, Amazon's document database that is built to be fast, serverless, designed to work with non-constant data—I mean flexible data, like varied JSON structures. DynamoDB is built for high volume, high scalability and high performance; I read an article in the sources that makes it VERY EXPENSIVE.

Here is a diagram summarizing how it works:
Going a bit further, commenting on things I didn't know, it uses a leader scheme and makes decisions based on request metadata. In other words, there is engineering to ensure everything is fast, so it uses LAMBDA (cloud functions that are instantiated only at execution time) to perform actions and can replicate automatically.

## Pros / Cons

| Aspect | Pros | Cons |
|--------|------|------|
| **Multi-AZ** | Automatic multi-region replication available | VERY EXPENSIVE - Cost driven per operation |
| **Scalability** | Guaranteed horizontal scalability | 400KB limit per object instantiated, stored in the database |
| **Use Cases** | Very fast speed for real-time operations | Very specific use cases |
| **Management** | Fully managed, serverless | Can be expensive at scale if not optimized |



## How to Choose

They are different data storage proposals. We use DynamoDB when we need low latency, large-scale data access for both reading and writing, stored models cannot be larger than 400KB, and typically DynamoDB can be used to handle specific data segments—for example, serverless applications using Lambda, things that need real-time low latency like game leaderboards, or advertising events (marketing advertisement), processing spiky traffic such as e-commerce carts or metadata from highly accessed websites.

RDS is more commonly used, firstly because it is "cheaper" and uses SQL-based relational databases—in other words, almost everything. RDS allows us to use SQL queries, meaning joins, transactions, and scales vertically (increasing database size). RDS has lower performance, but if we have a gap or predictability in I/O volume, in RDS we need to manage CPU, RAM, etc.

## Conclusion

DynamoDB is very specific in my humble opinion, and in the cases I observed in my research now and before, I believe its best use is for serverless Lambda integrations and using DynamoDB together with RDS to store data that will be used all the time, like session data, things that need to be answered at a speed that RDS does not provide, like scaling rapidly from 0 to millions. 

We could have a use case on top of this—my opinion is we should consult Nogueira haha—it can be implemented at the time of voting, for example, to be used in a duplicate data deduction service, but not for the final storage volume.

### Recommended Hybrid Approach for Voting System:

**We can use RDS for:**
- Permanent vote records (transactional integrity, exactly-once guarantee)
- User profiles and authentication data
- Complex reporting and analytics queries
- Audit logs requiring SQL joins

**We can use DynamoDB (or Redis) for:**
- Real-time deduplication keys (fast lookups, TTL-based expiration)
- Session tokens and temporary authentication states
- Rate limiting counters (per-user, per-IP tracking)
- Live voting metrics and leaderboards
- Fraud detection metadata (flexible schema for evolving rules)

**Why this works:**
The hybrid model leverages RDS for ACID compliance and data integrity where it matters most (the canonical vote record), while DynamoDB/Redis handles the high-velocity, ephemeral operations that need sub-10ms response times. This keeps costs manageable—RDS handles the predictable baseline load, and DynamoDB absorbs traffic spikes without requiring constant over-provisioning.

**Cost optimization tip:** For deduplication specifically, Redis might be more cost-effective than DynamoDB since keys expire automatically and you don't pay per operation—just for the cluster capacity. DynamoDB shines when you need durable storage with flexible querying, but for pure speed on throwaway data, Redis often wins both on performance and cost.

In the end, on my sight, it's not about choosing one over the other—it's about using each database for what it does best. RDS gives you reliability and queryability, DynamoDB gives you speed and scale, and together they cover the full spectrum of a high-traffic voting platform's needs.


## Acronyms

| Acronym | Meaning |
|---------|---------|
| **RDS** | Relational Database Service |
| **AWS** | Amazon Web Services |
| **AZ** | Availability Zone |
| **Multi-AZ** | Multi-Availability Zone (deployment across multiple data centers) |
| **SQL** | Structured Query Language |
| **NoSQL** | Not Only SQL (non-relational database) |
| **JSON** | JavaScript Object Notation |
| **TTL** | Time To Live (automatic expiration mechanism) |
| **ACID** | Atomicity, Consistency, Isolation, Durability (transaction properties) |
| **I/O** | Input/Output (read/write operations) |
| **CPU** | Central Processing Unit |
| **RAM** | Random Access Memory |
| **KB** | Kilobyte (unit of data measurement) |


## Sources:

### RDS

https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html

### DynamoDB

https://en.wikipedia.org/wiki/Amazon_DynamoDB
https://aws.amazon.com/pt/dynamodb/
https://medium.com/@joudwawad/dynamodb-architecture-5a38761501a7
https://medium.com/@christopheradamson253/deep-dive-into-aws-dynamodb-a-nosql-database-for-high-performance-applications-4c80d1410533
