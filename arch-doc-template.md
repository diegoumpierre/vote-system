## 🏛️ Structure

### 1. 🎯 Problem Statement and Context

The problem is to enable a global audience to vote in real time during a Live TV Show. Key challenges include handling enormous, short-lived traffic spikes from a user base of hundreds of millions, guaranteeing each registered viewer can cast exactly one vote, and ensuring no votes are lost even under failures. At the same time, the system must prevent bots and bad actors from skewing results, provide near real time aggregated results for on-air presentation.

### 2. 🎯 Goals

1. **Data Integrity & Durability:** one vote per user, zero data loss, multiple AZ
2. **Performance & Scale:** handle up to 300M users and 250k RPS peak load
3. **Low Latency:** p99 <150 ms write, <200 ms read, UI refresh <1 s
4. **Security & Anti-Fraud:** duplicate vote prevention, bot mitigation, rate limiting, CAPTCHA
5. **Observability & Auditability:**  logs, metrics, dashboards
6. **Fault Tolerance:** queue-based backpressure, retries, no dropped votes
7. **Modular Architecture:** independently scalable and replaceable components
8. **Cost Efficiency:** balance SLA vs operational cost through dynamic scaling
9. **Operational Simplicity:** IaC, automated deploys, minimal manual ops

### 3. 🎯 Non-Goals

1. Detailed UI/UX or native mobile apps
2. Legal/government auditing
3. Scalability beyond 300M users or 250k RPS
4. Serverless, MongoDB, OpenShift, mainframes, or monolithic solutions
5. ML-based fraud detection

### 📐 3. Principles

1. **Isolation & Fault Containment:** Application components are deployed separately to avoid cascading failures.
2. **Data Integrity:** Preventing double voting and data loss as high priority.
3. **Observability:** All critical paths expose metrics, logs, and traces.
4. **Security:** Rate limiting, bot detection and duplicate prevention.
5. **Automated Testing:** CI pipelines enforce automated tests, including load, integration, and failure scenarios.

### 🏗️ 4. Overall Diagrams

| Name | Image |
|------|-------|
| Overall Architecture | [Link](diagrams/4.1-overview-diagram.png) |
| Deployment Diagram | [Link](diagrams/4.2-deployment-diagram.png) |
| Use Cases Diagram | [Link](diagrams/4.3-use-cases-diagram.png) |

### 🧭 5. Trade-offs


| Decision | Alternative | Pros | Cons |
|----------|-------------|------|------|
| **ECS on EC2** | EKS with Kubernetes | AWS-native, less ops overhead; No EKS control plane fee, EC2 RI saves 40-60%; Deep ALB/CloudWatch/IAM integration; Two-level auto-scaling (ASG + ECS Service); Zero cold start, tasks start in seconds | Must manage EC2 instances, AMI patching, capacity planning; AWS-only, no cloud portability; Lacks K8s ecosystem (Helm, service mesh, GitOps); EC2 scaling takes 2-5 min, needs pre-warming before events |
| **Auth0** | Cognito or self-hosted Keycloak | Pre-built OAuth2/OIDC, MFA, social logins; SOC 2, ISO 27001 compliant, breach detection built-in; Passwordless auth, device fingerprinting, anomaly detection; Reduced security liability | Vendor lock-in, migrating 300M profiles is complex; High licensing cost at enterprise scale; Auth0 outage blocks authentication (~52 min/year allowed by SLA) |
| **RDS PostgreSQL** | DynamoDB, Cassandra | ACID guarantees for exactly-once vote recording; Strong consistency for immediate vote count queries; Standard SQL for post-event analytics (JOINs, aggregations); Familiar tooling, low learning curve | Write throughput caps at ~20-50k TPS, needs SQS buffering; Vertical scaling only, expensive large instances; Connection pool pressure at high ECS task count |
| **Multi-Region (3 regions)** | Single region with quota increase | 3 × 100k API Gateway RPS = 300k capacity with 20% headroom; <100ms latency for global users; Built-in DR via Route53 failover | Single-writer in US-East-1, 80-150ms cross-region write latency; 3x infrastructure cost; Coordinated deployments and rollbacks across regions; Route53 failover has 30s detection window |
| **API Gateway + ALB per service** | Direct ALB or API Gateway only | Centralized JWT validation via Auth0; Per-user rate limiting before hitting backend; HTTP/HTTPS/WebSocket unification; Cost-efficient at low traffic | Double hop latency (API GW + ALB + ECS); Double config surface (stages + target groups); 100k RPS per-region API Gateway cap; More expensive than ALB-only at sustained high RPS |
| **SQS async processing** | Synchronous direct-to-DB writes | Absorbs 250k RPS spikes, buffers for DB throughput; Guaranteed delivery, no vote loss on crashes; Decoupled scaling between ingestion and processing; Built-in retry with configurable maxReceiveCount | Eventual consistency, user gets "accepted" before DB write; Must monitor queue lag to stay within real-time SLA; DLQ requires manual intervention or batch replay |

### 🌏 6. For each key major component

#### 6.1 Class Diagram

| Diagram | Image |
|---------|-------|
| [6.1-class-diagram.drawio](diagrams/6.1-class-diagram.drawio) | [6.1-class-diagram.png](diagrams/6.1-class-diagram.png) |

#### 6.2 Contract Documentation

| Service | Swagger UI | Source |
|---------|-----------|--------|
| Vote Service | [View in Swagger UI](https://petstore.swagger.io/?url=https://raw.githubusercontent.com/diegoumpierre/vote-system/feature/nicolas-fixes/contracts/vote-service.yaml) | [vote-service.yaml](contracts/vote-service.yaml) |
| Results Service | [View in Swagger UI](https://petstore.swagger.io/?url=https://raw.githubusercontent.com/diegoumpierre/vote-system/feature/nicolas-fixes/contracts/results-service.yaml) | [results-service.yaml](contracts/results-service.yaml) |

#### 6.3 Persistence Model

**Full document:** [6.3-persistence-model.md](diagrams/6.3-persistence-model.md)

### 🖹 7. Migrations

No migrations.

### 🖹 8. Testing strategy

#### 8.1 Test Types

| Type | Technology | Scope | Trigger |
|------|-----------|-------|---------|
| **Unit** (70% coverage) | JUnit 5, Mockito | Domain logic, vote validation, election state transitions | Every commit |
| **Integration** | Testcontainers, WireMock | PostgreSQL, Redis, SQS integration | PR merge |
| **E2E** | Cypress | Vote submission flow, admin election flow, WebSocket updates | Nightly / pre-deploy |

---

#### 8.2 Load Testing

**Technology:** k6

| Scenario | Virtual Users | Duration | Target RPS | Success Criteria |
|---------|---------------|----------|-----------|------------------|
| **Baseline** | 10,000 | 10 min | 50,000 | p99 <100ms, 0% errors |
| **Peak Traffic** | 100,000 | 5 min | 250,000 | p99 <150ms, <0.01% errors |
| **Sustained** | 50,000 | 2 hours | 100,000 | Memory stable, no leaks |
| **Spike** | 0→150,000 (30s ramp) | 10 min | Variable | Auto-scale triggers, no 503s |

---

#### 8.3 Chaos Engineering

**Technology:** AWS Fault Injection Simulator (FIS)

| Experiment | Action | Expected Behavior | Pass Criteria |
|-----------|--------|-------------------|---------------|
| **AZ Failure** | Terminate all ECS tasks in AZ-a | Traffic routes to AZ-b and AZ-c | <5s downtime, no vote loss |
| **Database Failover** | Force RDS primary failover | Writes pause, SQS buffers votes | Processing resumes post-failover |
| **Redis Node Failure** | Kill 1 of 3 Redis nodes | Cluster reshards, reads continue | Cache hit rate drops <20%, no errors |
| **Auth0 Outage** | Block Auth0 API calls | Existing JWT sessions continue | Existing users can vote, new users see auth error |

### 🖹 9. Observability strategy

#### 9.1 Tooling

| Pillar | Tool | Purpose |
|--------|------|---------|
| **Logs** | CloudWatch Logs | Centralized application and access logs |
| **Metrics** | CloudWatch Metrics + Container Insights | Infrastructure, ECS task/service/cluster metrics |
| **Traces** | X-Ray | End-to-end distributed tracing with correlation ID per request |
| **Dashboards** | Grafana | Unified operational and business visualization |
| **Alerts** | CloudWatch Alarms → SNS | Threshold-based alerts routed to on-call teams |
| **Audit** | CloudTrail + S3 | Immutable audit trail with long-term retention |

Every request carries a **correlation ID** propagated across API Gateway, Vote Ingestion, SQS, Vote Processor, DB, so we can trace full vote lifecycle.

#### 9.2 Key Metrics and Alerts

| Metric | Source | Alert Threshold |
|--------|--------|----------------|
| API Gateway 5xx rate | CloudWatch | > 0.1% |
| Vote Ingestion p99 latency | X-Ray | > 150ms |
| Results Service p99 latency | X-Ray | > 200ms |
| SQS queue age (oldest message) | CloudWatch | > 30 seconds |
| SQS DLQ message count | CloudWatch | > 0 (immediate) |
| RDS CPU utilization | CloudWatch | > 80% |
| ECS task CPU | Container Insights | > 85% |
| Redis memory usage | CloudWatch | > 75% |
| Vote loss (SQS sent - DB rows) | Custom metric | > 0 |

#### 9.3 What Gets Logged

| Service | Events |
|---------|--------|
| **Vote Ingestion** | Vote accepted (queued), duplicate rejected, CAPTCHA failed, election not found |
| **Vote Processor** | Vote persisted, Redis counter incremented, DB write failed, message sent to DLQ |
| **Results Service** | Cache miss, aggregation served |
| **Infrastructure** | 5xx from API Gateway/ALB, ECS task restart, RDS failover, Auth0 integration error |

#### 9.4 Dashboards

| Dashboard | Audience | Key Panels |
|-----------|----------|------------|
| **Live Voting** | Operations | RPS, SQS queue depth, p99 latency, error rate, votes/second |
| **Vote Integrity** | Operations + Audit | SQS sent vs DB persisted, DLQ count, duplicate rejection rate |
| **Infrastructure** | DevOps | ECS CPU/memory, RDS connections/CPU, Redis memory, ALB health |

#### 9.5 Zero Vote Loss Reconciliation

- Vote Ingestion publishes `votes.accepted` metric on every SQS send
- Vote Processor publishes `votes.persisted` on every successful DB write
- CloudWatch alarm triggers if `votes.accepted - votes.persisted > 0` for more than 5 minutes
- DLQ count > 0 triggers immediate alert for manual investigation

### 🖹 10. Data Store Designs

#### 10.1 Store Overview

| Store | Service | Role | Topology |
|-------|---------|------|----------|
| PostgreSQL (RDS) | Vote Processor | Source of truth for votes, elections, candidates | Primary in US-East-1, read replicas in EU-West-1 and AP-Southeast-1 |
| Redis (ElastiCache) | Vote Ingestion, Vote Processor, Result Service | Dedup check, live vote counters, election metadata cache | Primary in US-East-1, replicas in EU-West-1 and AP-Southeast-1 |

#### 10.2 Data Dictionary — PostgreSQL

**elections**

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | UUID | No | Primary key |
| title | VARCHAR(255) | No | Election display name |
| description | VARCHAR(1000) | Yes | Optional description |
| status | VARCHAR(20) | No | DRAFT, SCHEDULED, ACTIVE, CLOSED, CANCELLED |
| start_date | TIMESTAMP | No | Voting window start (UTC) |
| end_date | TIMESTAMP | No | Voting window end (UTC) |
| created_at | TIMESTAMP | No | Row creation time |
| updated_at | TIMESTAMP | No | Last modification time |

**candidates**

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | UUID | No | Primary key |
| election_id | UUID | No | FK → elections.id (CASCADE DELETE) |
| name | VARCHAR(255) | No | Candidate display name |
| party | VARCHAR(100) | Yes | Party or group affiliation |
| photo_url | VARCHAR(500) | Yes | URL to candidate photo in S3 |
| created_at | TIMESTAMP | No | Row creation time |

**votes**

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | UUID | No | Primary key |
| election_id | UUID | No | FK → elections.id |
| candidate_id | UUID | No | FK → candidates.id |
| user_id | VARCHAR(255) | No | Auth0 subject identifier |
| accepted_at | TIMESTAMP | No | Time vote was accepted by Vote Ingestion |

#### 10.3 Data Dictionary — Redis

| Key Pattern | Type | TTL | Written By | Read By |
|-------------|------|-----|------------|---------|
| `vote:{electionId}:{userId}` | STRING | 48h | Vote Ingestion | Vote Ingestion |
| `results:{electionId}:{candidateId}` | STRING (counter) | None | Vote Processor | Result Service |
| `election:{electionId}` | HASH | 1h | Vote Ingestion | Vote Ingestion, Result Service |

#### 10.4 Performance Expectations

| Store | Operation | Expected Throughput | Latency Target |
|-------|-----------|--------------------:|----------------|
| Redis | EXISTS (dedup check) | 250k ops/s | < 2ms |
| Redis | INCR (vote counter) | 50k ops/s | < 2ms |
| Redis | HGETALL (election metadata) | 10k ops/s | < 2ms |
| PostgreSQL | INSERT vote (from SQS consumer) | 20–50k rows/s | < 50ms |
| PostgreSQL | SELECT aggregation (results) | 100 req/s | < 200ms |
| PostgreSQL | CRUD elections/candidates (admin) | < 100 req/s | < 100ms |

### 🖹 11. Technology Stack

#### Backend
Language: Java 25 (LTS)
* Strong performance under high concurrency.
* Modern language features improve domain expressiveness.
* Virtual Threads (Project Loom) enable high throughput using a simple blocking programming model.

Framework: Spring Boot 4.0.1
* Orchestrates use cases.
* Coordinates domain logic through clearly defined ports.
* Thread-per-request using Virtual Threads.

REST API: Spring MVC 7.0.2
* Exposes HTTP endpoints.
* Translates external requests into domain use case calls.

API Contracts: OpenAPI (Swagger) 3.0.4

Database: PostgreSQL 17.6 (Amazon RDS)
* Spring Data JPA + Hibernate.
* ACID guarantees ensure vote integrity and exactness.

Cache: Redis 8.4
* Cached vote aggregates.
* Session data.

Messaging: Amazon SQS
* Buffers vote submissions.
* Decouples ingestion from processing.
* Protects the system from traffic spikes.

Security:
* Spring Security
* JWT
* Auth0

#### Frontend
Framework: React.js

State Management: Redux Toolkit

Styling: Tailwind CSS

Data Fetching: Axios

#### Infrastructure & Deployment
Containerization: Docker

Orchestration: Amazon ECS on EC2

Load Balancing / Gateway:

* AWS Application Load Balancer
* AWS API Gateway (rate limiting, edge security)

Secrets & Configuration: AWS Secrets Manager

#### Security
* JWT authentication
* TLS for all communications
* AWS WAF for DDoS and abuse protection

#### Observability & Operations
* Metrics: Prometheus
* Dashboards: Grafana

#### Development & CI/CD
Build Tools: Maven

CI/CD: GitHub Actions

#### Architectural Rationale

Hexagonal Architecture:
* Ensures business logic is independent of frameworks and infrastructure.
* Simplifies testing by mocking ports instead of infrastructure.

Scalability:
* Stateless Spring Boot services scale horizontally under load.

Reliability:
* PostgreSQL transactions + SQS buffering prevent vote loss or duplication.

Maintainability:
* Clear separation of concerns reduces long-term complexity.

Frontend Decoupling:
* React frontend interacts exclusively through APIs, enabling independent evolution.

### 🖹 12. References

* Architecture Anti-Patterns: https://architecture-antipatterns.tech/
* EIP https://www.enterpriseintegrationpatterns.com/
* SOA Patterns https://patterns.arcitura.com/soa-patterns
* API Patterns https://microservice-api-patterns.org/
* Anti-Patterns https://sourcemaking.com/antipatterns/software-development-antipatterns
* Refactoring Patterns https://sourcemaking.com/refactoring/refactorings
* Database Refactoring Patterns https://databaserefactoring.com/
* Data Modelling Redis https://redis.com/blog/nosql-data-modeling/
* Cloud Patterns https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/introduction.html
* 12 Factors App https://12factor.net/
* Relational DB Patterns https://www.geeksforgeeks.org/design-patterns-for-relational-databases/
* Rendering Patterns https://www.patterns.dev/vanilla/rendering-patterns/
* REST API Design https://blog.stoplight.io/api-design-patterns-for-rest-web-services
