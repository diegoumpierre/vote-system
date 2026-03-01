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

#### 8.1 Testing Principles

1. **Shift-Left Testing** - Testing starts from design phase, not post-development
2. **Pyramid Approach** - Many unit tests, fewer integration tests, minimal E2E tests
3. **Production-like Testing** - Test environments mirror production infrastructure
4. **Chaos Engineering as Standard** - Proactively inject failures to validate resilience
5. **Performance Testing Continuous** - Load tests run in CI/CD, not just pre-release
6. **Test Data Isolation** - Each test uses isolated data to prevent cascading failures
7. **Observability-Driven Testing** - Tests validate metrics, logs, and traces are emitted correctly

---

#### 8.2 Types of Tests

##### Unit Tests (70% coverage target)
**Technology:** JUnit 5, Mockito, AssertJ

**Scope:**
- Domain logic (vote validation, duplicate detection algorithms)
- Business rules (exactly-once voting, election state transitions)
- Data transformations (DTO ↔ Entity mapping)

**Mock Strategy:**
- Repository interfaces mocked with Mockito
- Redis/SQS clients mocked to avoid external dependencies
- Auth0 JWT validation mocked with test tokens

**Key Test Scenarios:**
- Vote Service: Duplicate detection validation
- Results Service: Vote aggregation across multiple candidates
- Error handling: Invalid inputs, missing fields, malformed data

**Execution:**
- Run on every commit via GitHub Actions
- Fail build if coverage drops below 70%
- Execution time target: <5 minutes

---

##### Integration Tests (Component-level)
**Technology:** Testcontainers, Spring Boot Test, WireMock

**Scope:**
- Service ↔ PostgreSQL integration (vote persistence, transaction rollback)
- Service ↔ Redis integration (cache invalidation, TTL expiration)
- Service ↔ SQS integration (message publishing, retry logic, DLQ handling)
- Auth0 integration (JWT validation with WireMock-stubbed responses)

**Test Environment:**
- **Testcontainers** - Spin up PostgreSQL 17, Redis 8.4, LocalStack (SQS emulator)
- **Isolated Docker Network** - Each test suite runs in isolated containers
- **Data Reset** - Database schema recreated before each test class

**Key Test Scenarios:**
- Vote persistence to PostgreSQL with Redis counter increment
- Transaction rollback on failure
- SQS retry logic with Dead Letter Queue handling
- Cache invalidation and TTL expiration

**Execution:**
- Run on PR merge to main branch
- Execution time target: <10 minutes

---

##### Contract Tests
**Technology:** Spring Cloud Contract, Pact

**Scope:**
- Validate API contracts between Frontend ↔ Backend
- Ensure backward compatibility during API changes
- Test error response schemas (4xx, 5xx formats)

**Contract Definition:**
- Request: POST /api/v1/votes with JWT authorization, election/candidate/user IDs, CAPTCHA token
- Response: 200 status with vote ID, acceptance status, timestamp
- Error schemas: 4xx for validation errors, 5xx for server errors

**Execution:**
- Run on every API change
- Publish contracts to shared repository
- Frontend team validates against published contracts

---

##### End-to-End Tests
**Technology:** Selenium, Cypress

**Scope:**
- User registration → Login → Vote submission → Results view
- Admin creates election → Opens voting → Monitors results → Closes voting
- Validate WebSocket real-time updates appear in UI

**Test Data Strategy:**
- Dedicated E2E test environment (separate AWS account)
- Synthetic users (test-user-001@example.com to test-user-100@example.com)
- Pre-seeded elections and candidates via API

**Key Test Flows:**
- User registration → Login → Vote submission → Results view
- Admin creates election → Opens voting → Monitors results → Closes voting
- WebSocket real-time updates validation

**Execution:**
- Run nightly against staging environment
- Run manually before production deployment
- Execution time target: <30 minutes

---

#### 8.3 Performance and Load Testing

##### Load Testing
**Technology:** Apache JMeter, k6, Gatling

**Objectives:**
- Validate system handles **250,000 RPS** peak load
- Verify p99 latency <150ms for writes, <200ms for reads
- Ensure no vote loss under sustained load

**Test Scenarios:**

| Scenario | Virtual Users | Duration | Target RPS | Success Criteria |
|---------|---------------|----------|-----------|------------------|
| **Baseline Load** | 10,000 | 10 min | 50,000 | p99 <100ms, 0% errors |
| **Peak Traffic** | 100,000 | 5 min | 250,000 | p99 <150ms, <0.01% errors |
| **Sustained Load** | 50,000 | 2 hours | 100,000 | Memory stable, no leaks |
| **Traffic Spike** | 0→150,000 (ramp 30s) | 10 min | Variable | Auto-scale triggers, no 503s |

**Load Test Configuration:**
- Ramp-up stages: 50k users → 100k users → ramp-down
- Thresholds: p99 <150ms, error rate <0.01%
- Request validation: Status 200, vote accepted confirmation
- Rate limiting: 1 second delay between requests per user

**Mock Data Strategy:**
- **Users:** Generate 1M synthetic user accounts with Auth0 test tenant
- **Elections:** 10 pre-configured elections with 5 candidates each
- **JWT Tokens:** Pre-generated 100k valid JWT tokens (rotated every 15 minutes)
- **CAPTCHA:** Mock CAPTCHA service returns success for load test tokens

**Execution Environment:**
- Run against dedicated staging environment (identical to production)
- 3 AWS regions (US-East, EU-West, AP-Southeast)
- RDS PostgreSQL db.r6g.4xlarge (same as production)
- Redis cluster (3 nodes per region)

**Metrics Validation:**

Target Metrics (must pass):
- API Gateway RPS: ≥250,000
- Vote Service p99 latency: <150ms
- Results Service p99 latency: <200ms
- SQS queue lag: <5 seconds
- Database write throughput: ≥20,000 TPS
- Redis GET latency: <5ms
- Error rate: <0.01%
- Zero vote loss (verify DB count = SQS sent count)

Warning Thresholds:
- SQS queue depth >10,000 messages
- Database CPU >80%
- ECS task CPU >85%
- Redis memory >75%

**Execution:**
- Weekly automated runs (every Sunday 2 AM UTC)
- On-demand before major releases
- Report published to Grafana dashboard + Slack

---

##### Stress Testing
**Objective:** Find the breaking point of the system

**Approach:**
- Gradually increase load beyond 250k RPS until system fails
- Identify bottleneck (database, SQS, ECS, network)
- Document failure mode (graceful degradation vs crash)

**Expected Breaking Points:**
- PostgreSQL write throughput: ~50,000 TPS (before connection pool exhaustion)
- API Gateway per-region limit: 100,000 RPS (hard AWS limit)
- Redis memory: 25GB (then eviction policy kicks in)

**Stress Test Scenario:**
```
Phase 1: 100k RPS (2 min) - Expected: Pass
Phase 2: 200k RPS (2 min) - Expected: Pass
Phase 3: 300k RPS (2 min) - Expected: Pass with SQS backlog
Phase 4: 400k RPS (2 min) - Expected: API Gateway 503 errors
Phase 5: 500k RPS (2 min) - Expected: Cascading failures
```

**Recovery Validation:**
- After stress, reduce load to 100k RPS
- System should auto-recover within 5 minutes (drain SQS queue)
- No manual intervention required

---

#### 8.4 Chaos Engineering

**Philosophy:** "Break things on purpose to ensure they don't break in production"

**Technology:** AWS Fault Injection Simulator (FIS), Chaos Mesh, custom scripts

##### Chaos Experiments

| Experiment | Chaos Action | Expected Behavior | Pass Criteria |
|-----------|-------------|-------------------|---------------|
| **AZ Failure** | Terminate all ECS tasks in AZ-a | Traffic routes to AZ-b and AZ-c | <5s downtime, no vote loss |
| **Database Failover** | Force RDS primary failover to standby | Read replicas serve reads, writes pause 30-60s | SQS buffers votes, processing resumes post-failover |
| **Redis Cluster Node Failure** | Kill 1 of 3 Redis nodes | Redis cluster reshards, reads continue | Cache hit rate drops <20%, no errors |
| **Network Partition** | Isolate Vote Service from PostgreSQL for 60s | Votes queue in SQS, DLQ triggers after max retries | No votes lost, DLQ processed after recovery |
| **API Gateway Throttling** | Inject 503 errors at 20% rate | Frontend retries with exponential backoff | User sees "Please retry" message, eventual success |
| **SQS Delay** | Increase SQS message visibility timeout to 5 min | Vote processing delayed, results lag | UI shows "Results updating..." message |
| **Auth0 Outage** | Block Auth0 API calls | New logins fail, existing sessions continue (JWT cached) | Existing users can vote, new users see auth error |
| **Cross-Region Replication Lag** | Delay RDS replication to EU region by 10 minutes | EU reads show stale data | Frontend polls primary region for critical reads |
| **Memory Pressure** | Stress test ECS task to trigger OOM | ECS restarts task, ALB health check removes from pool | <10s downtime per task, load balances to healthy tasks |
| **Disk Full** | Fill CloudWatch Logs disk on ECS host | Logs stop writing, application continues | Alert triggers, auto-cleanup old logs |

**Chaos Test Execution:**
- Use AWS Fault Injection Simulator (FIS) for automated chaos experiments
- Monitor impact via CloudWatch metrics (ErrorRate, Latency, Throughput)
- Real-time dashboards track recovery time and system behavior

**Chaos Schedule:**
- **Weekly:** Random AZ failure (Tuesday 10 AM UTC)
- **Bi-weekly:** Database failover (Wednesday 3 PM UTC)
- **Monthly:** Full region failure (first Saturday of month, 2 AM UTC)
- **Quarterly:** Game Day (simulate live show with injected failures)

**Chaos Goals:**
1. **Zero Vote Loss** - Even under failures, every vote persisted in SQS must reach the database
2. **Graceful Degradation** - System slows down but doesn't crash
3. **Auto-Recovery** - No manual intervention required for common failures
4. **Alert Accuracy** - Every chaos event triggers correct alert within 60 seconds

---

#### 8.5 Security Testing

##### Penetration Testing
**Scope:**
- SQL injection attempts on `/api/v1/votes`, `/api/v1/results`
- JWT token tampering (modified claims, expired tokens, invalid signatures)
- CAPTCHA bypass attempts (replay attacks, token reuse)
- Rate limit bypass (distributed IPs, header spoofing)
- CORS misconfiguration testing

**Tools:**
- OWASP ZAP, Burp Suite
- Manual testing by external security firm (annual)

##### Vulnerability Scanning
**Tools:**
- **SAST:** SonarQube (static code analysis)
- **DAST:** OWASP ZAP (dynamic API scanning)
- **Dependency Scanning:** Snyk, OWASP Dependency-Check

**Execution:**
- SAST runs on every PR
- DAST runs weekly against staging
- Dependency scan runs daily (alerts on critical CVEs)

---

#### 8.6 Testing Responsibilities

| Role | Responsibilities |
|------|----------------|
| **Developers** | Write unit tests, integration tests; fix failing tests within 1 day |
| **QA Engineers** | Design E2E tests, execute chaos experiments, analyze performance reports |
| **DevOps Team** | Maintain test infrastructure (Testcontainers, staging environment), CI/CD pipelines |
| **Security Team** | Conduct penetration tests, review SAST/DAST findings quarterly |
| **Product Owner** | Define acceptance criteria, approve test coverage thresholds |

---

#### 8.7 Test Environment Strategy

| Environment | Purpose | Data | Refresh Frequency |
|------------|---------|------|------------------|
| **Local** | Unit + integration tests | Testcontainers (fresh DB per run) | Per test execution |
| **Staging** | E2E, load tests, chaos | Synthetic (10M users, 100 elections) | Weekly (Sunday 2 AM UTC) |
| **Production-Mirror** | Pre-release validation | Anonymized production snapshot | Monthly |

### 🖹 9. Observability strategy

Our observability is designed to guarantee end-to-end visibility of the voting journey (ingestion, processing, and counting), focusing on:

- log vote proccess with objetive to check zero loss;
- early detection of manipulation at peaks of 250k RPS;
- audit trail for security and compliance.
#### 9.1
1. **Observability by default**: Every critical component should emit logs, metrics, and traces.
2. **End-to-end correlation**: Each request/event should carry `ID`.
3. **Alert by user impact**: Prioritize error signals, latency, and queuing before isolated infrastructure.
4. **Automation first**: Alarms should trigger automatic runbooks whenever possible.
5. **Cost conscious**: Retention, sampling, and cardinality under control.

**For that as part of observability layer we have this services and responsabilities**

- **CloudWatch Logs**: centralizes application, access, and audit logs.
- **CloudWatch Metrics**: infrastructure, platform, and business metrics.
- **Container Insights**: detailed view of ECS (task/service/cluster).
- **X-Ray**: end-to-end distributed tracing.
- **CloudWatch Alarms**: threshold/anomaly deviation detection.
- **EventBridge + Systems Manager**: event routing and response automation.
- **SNS**: alerts for teams (on-call, security, product).
- **Grafana**: unified visualization and operational analysis.
- **CloudTrail + S3**: audit trail and long-term retention.

**Critical events to log**
- **Vote Service**: accepted vote (queued), deduplication, idempotence, publish SQS, persistence failure.
- **Results Service**: aggregate update, cache invalidation, update delay.
- **Auth Service**: login, authentication/authorization failures, token refresh.
- **Infrastructure**: 5xx API Gateway/ALB, ECS task failures, RDS failover, integration errors with Auth0.

**Observability Flow:**
ECS Services + API Gateway + ALB + SQS + RDS + Redis
→ CloudWatch Logs / CloudWatch Metrics / X-Ray / Container Insights
→ CloudWatch Alarms
→ EventBridge
→ Systems Manager (self-remediation) + SNS (notification)
→ Grafana (operational and business dashboards)

### 🖹 10. Data Store Designs

For each different kind of data store i.e (Postgres, Memcached, Elasticache, S3, Neo4J etc...) describe the schemas, what would be stored there and why, main queries, expectations on performance. Diagrams are welcome but you really need some dictionaries.

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
