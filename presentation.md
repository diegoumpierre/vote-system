# Presentation Script -- Live TV Show Voting System

Format: scroll through arch-doc-template.md while speaking. Each section below matches a section in the document.

---

## Section 1 -- Problem Statement and Context

"The challenge is to build a system that allows a global audience to vote in real time during a Live TV Show. The expected audience is about 300 million users, and peaks of 250 thousand requests per second. Every vote must count, we should have zero data loss, no duplicate votes per user and no bots."

---

## Section 2 -- Goals

"Our goals are basically: data integrity, zero vote loss, handle peaks of 250k RPS, p99 latency under 150ms for writes and under 200ms for reads, and strong anti-fraud with CAPTCHA and rate limiting.

We have observability via logs and metrics.

Fault tolerance with a queue backpressure to avoid losing votes

Modular architecture, meaning we have independently scalable and replaceable components

Cost efficiency and operational simplicity"

---

## Section 3 -- Non-Goals and Principles

"What we're NOT doing: no detailed UI, no legal auditing, no scaling beyond 300M users. No serverless, MongoDB, OpenShift, mainframes, or monoliths. And no machine learn based fraud detection"


# Section 3 -- Principles

"Our principles are basically Isolation and fault containment so one component failing doesn't bring everything down; data integrity as the top priority; observability on all critical paths; security with rate limiting and bot detection; and automated testing including load and chaos scenarios in CI."

---

## Section 4 -- Overall Diagrams

"The client is a PWA application that authenticates via Auth0 to get a JWT. Requests hit Route 53, which does latency-based routing to the closest of our three regions. 

On the static content side, CloudFront serves the frontend files from S3. We have WAF for DDoS and bot protection.

Each region has an API Gateway with WAF also for DDoS and bot protection."

"Inside each region, traffic goes through load balancers: one for the Vote path and one for the Results path. 

Vote Ingestion receives the HTTP request, checks Redis for duplicate votes, and if it's a new vote, pushes it to SQS queue. 

The Vote Processor consumes from SQS, writes to PostgreSQL, and increments the Redis counter. The Result Service reads directly from Redis to serve live results."

"For data stores: PostgreSQL has a single writer in US-East-1 with read replicas in the other two regions. Redis follows the same topology, the primary in US-East-1, replicas elsewhere."




When showing the **Deployment Diagram** image:

"From a deployment perspective, we have three JAR artifacts running on ECS: result-service.jar, ingestion-service.jar, and processor-service.jar. All images are stored in ECR. The frontend is a static app served through CloudFront. On the data side, RDS hosts PostgreSQL, ElastiCache hosts Redis, and SQS holds the vote queue."



When showing the **Use Cases Diagram** image:

"Two actor types: Authenticated users can submit a vote and view results. Anonymous users can register, login, and logout. The Admin inherits from Authenticated and can also manage elections, manage candidates, and open or close the voting window."

---

## Section 5 -- Trade-offs

"Let's walk through the key trade-offs"

"ECS vs EC2 over EKS -- lower ops overhead, no control plane fee, deep AWS integration, and zero cold start. The trade-off is we must manage EC2 instances and AMI patching, and we lose Kubernetes portability."

"Auth0 vs Cognito or Keycloak -- pre-built OAuth2, MFA, SOC 2 compliant, anomaly detection built in. The risk is vendor lock-in, migrating 300M profiles would be painful, and an Auth0 outage blocks new logins."

"RDS PostgreSQL vs DynamoDB o -- ACID guarantees for exactly one votie per person, strong consistency, standard SQL for analytics. The downside is write throughput is around 20-50k transactions per second, which is why we buffer with SQS."

"Multi-region vs Single region -- three regions give us 300k RPS capacity with some buffer and less latency globally. The downside is that cost is 3 times higher, and we have the delay to synchronize the database and cache replicas across the regions."

"SQS async processing -- this is how we absorb the 250k RPS spikes. SQS guarantees delivery, no votes are lost on crashes, and ingestion scales independently from processing. The trade-off is eventual consistency -- the user gets 'accepted' before the DB write happens."

---

## Section 6.1 -- Class Diagram

"Our domain model is very simple. We have classes for Vote, election, candidate and some classes for results service, like ElectionResults with a list of CandidateTotal, which shows the number of votes per candidate."

---

## Section 6.2 -- Contract Documentation

"We have two OpenAPI contracts: one for the Vote Service, which includes both voting endpoints and admin endpoints for managing elections and candidates, and one for the Results Service, which covers aggregated results, live counts, WebSocket streaming, and statistics. You can click through to Swagger UI to explore the full specs."

---

## Section 6.3 -- Persistence Model

"Three tables in PostgreSQL: elections, candidates, and votes. The key constraint is the unique index on election_id plus user_id in the votes table -- this is another way to guarantee one vote per user."

"On the Redis side, three key patterns: a STRING key per election-user with 48-hour TTL for the fast duplicate check, a counter per election-candidate pair that gets incremented with INCR for live results, and a HASH per election for cached metadata."

"The two main Redis operations are EXISTS for the dedup check and INCR for the vote counter."

---

## Section 8 -- Testing Strategy

"We have three layers of testing. Unit tests with JUnit 5 and Mockito, this runs on every commit. Integration tests using Testcontainers and WireMock for PostgreSQL, Redis, and SQS, this is triggered on PR merge. End-to-end tests with Cypress covering the vote submission and admin flows, this runs every night."

"For load testing, we use k6 with four scenarios: baseline with 50k RPS, peak traffic at 250k RPS, a 2-hour sustained test with 100k RPS, and a spike test from zero to 150k in 30 seconds to validate the auto-scaling."

"For chaos engineering, we use AWS Fault Injection Simulator to run four experiments: AZ failure, database failover, Redis node failure, and Auth0 outage. The main validation is that SQS buffers votes during any infrastructure failure and no votes are lost."

---

## Section 9 -- Observability Strategy

"We have CloudWatch to handle logs and metrics, X-Ray handles distributed tracing. For that, every request carries a correlation ID from API Gateway through to the database.

Grafana to show dashboards, alerts with CloudWatch Alarms and audit with CloudTrail"

"We have three dashboards: Live Voting for operations showing RPS and latency, Vote Integrity for audit showing SQS sent versus DB persisted, and Infrastructure for DevOps showing ECS, RDS, and Redis health."

"The most important part is the zero vote loss reconciliation at the bottom. Vote Ingestion publishes a votes.accepted metric on every SQS send. Vote Processor publishes votes.persisted on every DB write. If the difference is greater than zero for more than 5 minutes, an alarm fires. Any message in the DLQ triggers an immediate alert."

---

## Section 10 -- Data Store Designs

"Two data stores. PostgreSQL is the source of truth -- primary in US-East-1, read replicas in the other two regions. It handles 20 to 50 thousand inserts per second from the SQS consumer with a latency target under 50ms."

"Redis is the fast layer -- same topology with primary and replicas. It handles 250k EXISTS operations per second for dedup at under 2ms, and 50k INCR operations per second for the vote counters."

"Below you can see the full data dictionaries for both PostgreSQL and Redis, with every column, type, and purpose documented."

---

## Section 11 -- Technology Stack

"Backend: Java 25 with Virtual Threads, Spring Boot 4, Spring Data JPA for PostgreSQL, and Spring Security with Auth0 for JWT auth. Frontend: React PWA with Redux Toolkit and Tailwind. Infrastructure: Docker on ECS with EC2, Route 53 for routing, API Gateway for rate limiting, ALB per service, WAF for security, and CloudWatch plus X-Ray for observability. CI/CD through GitHub Actions with Maven builds."

---

## Closing (after scrolling past References)

"To summarize: this architecture handles 250k RPS through SQS buffering, guarantees zero vote loss with a two-layer dedup strategy -- Redis for speed, PostgreSQL unique constraint for safety -- scales across three regions for global coverage, and provides full observability to detect any anomaly within minutes. Questions?"