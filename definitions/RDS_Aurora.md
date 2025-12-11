## What is RDS?
Amazon RDS (Relational Database Service) is AWS's managed database service for traditional relational databases. It handles the operational work of running a database so you don't have to:

#### What it manages: automatic backups, software patching, monitoring, hardware provisioning, database setup
Supported database engines: MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, and Aurora
Your responsibility: designing your database schema, writing queries, optimizing your application

It's like having a database administrator working 24/7, but automated.

## What is Aurora?
Amazon Aurora is AWS's own proprietary relational database engine, designed specifically for the cloud. It's compatible with MySQL and PostgreSQL, meaning your existing applications can often switch to it without code changes.

#### Key features:

Performance: Up to 5x faster than standard MySQL, 3x faster than PostgreSQL
Availability: Data is replicated 6 ways across 3 availability zones automatically
Scalability: Storage auto-scales up to 128TB, and you can add read replicas easily
Serverless option: Aurora Serverless automatically scales capacity based on demand

Aurora is available through RDS - it's one of the database engine options you can choose when using RDS.

## Pricing Components

#### Instance (Compute) Costs
Standard RDS MySQL/PostgreSQL:

- db.t3.medium (2 vCPUs, 4GB RAM): ~$0.072/hour or ~$52-80/month Amazon EC2

#### Aurora:

- Instance costs are generally 20-40% higher than equivalent RDS instances
- Graviton2 instances (r6g) offer ~20% better price-performance than x86 instances
- Global Database: Additional fees per 1 million write operations plus data transfer costs between regions
- RDS Proxy: Extra charges for connection pooling
- Blue/Green Deployments: Additional environment costs

