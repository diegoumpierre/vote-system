## 🏛️ Structure

### 1. 🎯 Problem Statement and Context

The problem is to enable a global audience to vote in real time during a Live TV Show. Key challenges include handling enormous, short-lived traffic spikes from a user base of hundreds of millions, guaranteeing each registered viewer can cast exactly one vote, and ensuring no votes are lost even under failures. At the same time, the system must prevent bots and bad actors from skewing results, provide near real time aggregated results for on-air presentation, and keep an audit trail for post-event review.

### 2. 🎯 Goals

1. Data Integrity & Durability - exactly-once voting, zero data loss, cross-AZ replication
2. Performance & Scale - handle up to 300M users and 250k RPS peak load
3. Low Latency - p99 <150 ms write, <200 ms read, UI refresh <1 s
4. Security & Anti-Fraud - duplicate vote prevention, bot mitigation, rate limiting, CAPTCHA
5. Observability & Auditability - end-to-end tracing, immutable logs, metrics, dashboards
6. Fault Tolerance - queue-based backpressure, retries, no dropped votes
7. Modular Architecture - independently scalable and replaceable components
8. Cost Efficiency – balance SLA vs operational cost through dynamic scaling
9. Compliance & Privacy - ensure GDPR/CCPA compliance and data anonymization
10. Operational Simplicity - IaC, automated deploys, minimal manual ops

### 3. 🎯 Non-Goals

1. Detailed UI/UX or native mobile apps (backend scope only)
2. Legal/government-grade auditing - limited to entertainment context
3. Indefinite raw data storage - retention policies will apply
4. Scalability beyond 300M users or 250k RPS
5. Serverless, MongoDB, OpenShift, mainframes, or monolithic solutions (per restrictions)
6. ML-based fraud detection - only rule-based protection
7. Payment or monetary transaction handling

### 📐 3. Principles

```
1. Isolation & Fault Containment: Must isolate failures using queues and retries. Infrastructure and application components are deployed separately to avoid cascading failures.
2. Data Integrity: Preventing double voting and data loss has higher priority than raw throughput.
3. Observability: All critical paths must expose metrics, logs, and traces.
4. Security: Rate limiting, bot detection, duplicate prevention, and auditability are architectural concerns, not optional add-ons or application-level patches.
5. Automated Testing: CI pipelines enforce automated tests, including load, integration, and failure scenarios. The system must be continuously validated under degraded conditions.
```

### 🏗️ 4. Overall Diagrams

Here there will be a bunch of diagrams, to understand the solution.
```
🗂️ 4.1 Overall architecture: Show the big picture, relationship between macro components.
🗂️ 4.2 Deployment: Show the infra in a big picture. 
🗂️ 4.3 Use Cases: Make 1 macro use case diagram that list the main capability that needs to be covered. 
```
Recommended Reading: http://diego-pacheco.blogspot.com/2020/10/uml-hidden-gems.html

---

#### 🗂️ 4.2 Deployment Diagram

**Full document:** [diagrams/4.2-deployment-diagram.md](diagrams/4.2-deployment-diagram.md)

**Summary:** AWS multi-region infrastructure (US-East, EU-West, AP-Southeast) to reach 250k RPS.

| Component | Per Region |
|----------|------------|
| API Gateway | 100k RPS |
| ECS Fargate | Auth, Vote, Results Services |
| RDS PostgreSQL | Primary + 2 Read Replicas |
| ElastiCache Redis | Vote aggregates + Session |
| SQS | Vote buffering + DLQ |

---

#### 🗂️ 4.3 Use Cases Diagram

**Full document:** [diagrams/4.3-use-cases-diagram.md](diagrams/4.3-use-cases-diagram.md)

**Summary:** 21 Use Cases organized by 4 actors.

| Actor | Use Cases |
|-------|-----------|
| Viewer | Register, Login, Submit Vote, View Results |
| Admin | Create Election, Manage Candidates, Audit Logs |
| TV Host | Open/Close Voting, Stream Results |
| System | Process Queue, Aggregate Counts, Auto-scale |

---

### 🧭 5. Trade-offs

Major Decisions:
```
1. ECS Fargate for containerized microservices
2. Auth0 for managed authentication and authorization
3. RDS (PostgreSQL) as primary datastore over NoSQL alternatives
4. Multi-region deployment to handle 250k RPS requirement
5. API Gateway with ALB per service for traffic management
6. SQS for asynchronous vote processing and traffic buffering
```

Tradeoffs:

**1. ECS Fargate vs (EC2 with Kubernetes or ECS on EC2)**

PROS (+)
* Zero server management: No need to provision, configure, or scale EC2 instances, reducing operational overhead.
* Auto-scaling efficiency: Fargate scales containers independently based on CPU/memory metrics without pre-provisioning capacity, optimizing cost during traffic valleys.
* Built-in isolation: Each task runs in its own kernel, providing better security boundaries compared to shared EC2 instances.
* Pay-per-use pricing: Only pay for vCPU and memory consumed during task runtime, eliminating idle capacity costs during off-peak hours.

CONS (-)
* Cold start latency: New task provisioning takes 30-60 seconds, potentially causing p99 latency spikes during rapid scale-up events.
* Cost at sustained scale: For 24/7 high-utilization workloads, Fargate is 20-30% more expensive than Reserved Instance EC2, impacting long-term operational budget.
* Limited instance type control: Cannot choose specific CPU/memory ratios or specialized hardware (GPU, high network bandwidth), restricting optimization for specific workload profiles.
* No direct host access: Debugging requires CloudWatch logs only; cannot SSH into underlying infrastructure, complicating deep troubleshooting scenarios.

**2. Auth0 vs (Custom-built authentication with Cognito or self-hosted Keycloak)**

PROS (+)
* Accelerated time-to-market: Pre-built OAuth2/OIDC flows, MFA, social logins eliminate months of authentication development and security hardening.
* Enterprise-grade security: SOC 2, ISO 27001 compliance, automatic breach detection, and credential stuffing prevention offload critical security responsibilities.
* Advanced features out-of-box: Passwordless authentication, device fingerprinting, anomaly detection, and user management UI require zero custom development.
* Reduced liability: Security incidents (credential leaks, token compromise) are Auth0's contractual responsibility, limiting organizational legal exposure.

CONS (-)
* Vendor lock-in: Migrating 300M user profiles away from Auth0 would require complex data export, password re-hashing, and session migration strategies.
* Recurring licensing costs: Enterprise tier for millions of users has a significant cost compared to self-hosted alternatives.
* Availability dependency: Auth0 outage directly impacts authentication; their 99.99% SLA still allows ~52 minutes downtime/year, potentially during peak voting windows.
* Data sovereignty concerns: User PII stored in Auth0's infrastructure may conflict with GDPR/CCPA residency requirements in certain jurisdictions.

**3. RDS PostgreSQL vs (NoSQL: DynamoDB, MongoDB, Cassandra)**

PROS (+)
* ACID guarantees: Transactions ensure exactly-once vote recording even under concurrent requests or network retries, preventing duplicate votes without application-level coordination.
* Strong consistency: Read-after-write guarantees allow immediate vote count queries without eventual consistency delays that could show incorrect results on live TV.
* SQL analytics: Complex post-event queries (JOIN across users, candidates, regions) use standard SQL instead of custom MapReduce jobs or denormalized table copies.
* Familiar tooling: Standard PostgreSQL ecosystem reduces learning curve and onboarding time for development teams.

CONS (-)
* Write throughput ceiling: PostgreSQL caps at ~20-50k writes/sec even with tuning; requires SQS buffering to handle 250k RPS peaks without overwhelming database.
* Vertical scaling limits: Cannot horizontally partition votes table easily; must scale up instance size (expensive r6g.16xlarge at $4.50/hr) instead of adding cheap nodes.
* Connection pool constraints: Each ECS task holds DB connections; 1000 concurrent tasks × 10 connections = 10k connections approaches PostgreSQL max_connections limit.

**4. Multi-Region Deployment vs (Single Region with Quota Increase)**

PROS (+)
* 250k RPS achievable: 3 regions × 100k API Gateway RPS each = 300k total capacity, exceeding peak requirement with 20% headroom for traffic spikes.
* Geographic latency reduction: US-East, EU-West, AP-Southeast placement ensures <100ms response time for almost 100% of global user base.
* Disaster recovery built-in: Region failure automatically routes traffic to healthy regions via Route53 failover, maintaining 99.95% availability SLA.
* Regulatory compliance: EU user data processed in EU region satisfies GDPR data residency requirements without custom routing logic.

CONS (-)
* Cross-region data synchronization: RDS Multi-Region replication introduces 100-500ms lag; vote totals may briefly diverge between regions during peak traffic.
* 3x infrastructure cost: Running full stack (ECS, RDS, ALB) in 3 regions triples baseline costs even during low-traffic periods outside voting windows.
* Deployment complexity: Schema migrations and application releases must coordinate across 3 regions; rollback requires orchestrating 3 separate actions.
* DNS failover limitations: Route53 health checks have 30-second detection window; region outage may expose users to 30s of errors before failover completes.

**5. API Gateway + ALB per Service vs (Direct ALB Exposure or API Gateway Only)**

PROS (+)
* Centralized authentication: API Gateway native JWT Authorizer validates JWT tokens once before routing (configured with Auth0 issuer + audience), preventing duplicate auth logic in each service.
* Request throttling: API Gateway enforces per-user rate limits, blocking abuse before it consumes backend resources.
* Protocol flexibility: API Gateway handles HTTP/HTTPS/WebSocket unification while ALBs focus on health-checked load distribution to ECS tasks.
* Cost efficiency for low traffic: API Gateway's cost per requests is cheaper than ALB's cost for services with <10 RPS baseline.

CONS (-)
* Double hop latency: Request traverses API Gateway → ALB → ECS instead of direct ALB → ECS, inflating p99 latency.
* Increased complexity: Managing both API Gateway stages (dev/prod) and ALB target groups doubles the configuration surface area for errors.
* API Gateway RPS bottleneck: Even with multi-region, each region's API Gateway caps at 100k RPS, forcing traffic sharding logic if single service exceeds limit.
* Higher cost at scale: At 250k sustained RPS, API Gateway costs + ALB costs exceed direct ALB-only approach.

**6. SQS for Async Vote Processing vs (Synchronous Direct-to-Database Writes)**

PROS (+)
* Traffic buffering: SQS absorbs 250k RPS spikes without overwhelming Vote Service or RDS; queue acts as shock absorber between ingestion and processing.
* Guaranteed delivery: Messages persist in SQS until successfully processed; Vote Service crashes don't lose votes, just delay processing until recovery.
* Decoupled scaling: API Gateway can scale to 250k RPS while Vote Service processes at sustainable 20k/sec rate; no forced coupling between tiers.
* Retry logic built-in: Failed votes automatically retry (up to maxReceiveCount) without custom application code; handles transient RDS connection errors gracefully.

CONS (-)
* Eventual consistency: User receives "Vote submitted" response before database write; cannot guarantee "Your vote is counted" until SQS message processed.
* Queue lag monitoring: Must track SQS ApproximateAgeOfOldestMessage metric; 5-minute lag means votes not reflected in live results, violating real-time requirement.
* Message ordering: Standard SQS doesn't guarantee FIFO; user's second vote (candidate change) might process before first vote, creating wrong final state.
* Dead-letter queue handling: Votes failing maxReceiveCount attempts move to DLQ; requires manual intervention or batch job to replay, risking data loss if ignored.

<BR/>Recommended reading: http://diego-pacheco.blogspot.com/2023/07/tradeoffs.html

### 🌏 6. For each key major component

What is a majore component? A service, a lambda, a important ui, a generalized approach for all uis, a generazid approach for computing a workload, etc...
```
6.1 - Class Diagram              : classic uml diagram with attributes and methods
6.2 - Contract Documentation     : Operations, Inputs and Outputs
6.3 - Persistence Model          : Diagrams, Table structure, partiotioning, main queries.
6.4 - Algorithms/Data Structures : Spesific algos that need to be used, along size with spesific data structures.
```

#### 6.2 Contract documentation

---

#### **Auth Service**
##### **Operations**

| Operation | Method | Endpoint | Description |
|-----------|--------|----------|-------------|
| Login | POST | `/api/v1/auth/login` | Authenticate user and return JWT token |
| Register | POST | `/api/v1/auth/register` | Create new user account |
| Logout | POST | `/api/v1/auth/logout` | Invalidate token and clear session |
| Validate Token | GET | `/api/v1/auth/validate` | Validate JWT and return user info |
| Refresh Token | POST | `/api/v1/auth/refresh` | Generate new access token |
---

##### **Operations**

| Operation | Method | Endpoint | Description |
|-----------|--------|----------|-------------|
| Submit Vote | POST | `/api/v1/votes` | Submit a vote for an election |
| Check Vote Status | GET | `/api/v1/votes/status/{userId}` | Verify if user voted |

**Input:**
```json
{
  "electionId": "550e8400-e29b-41d4-a716-446655440000",
  "candidateId": "660e8400-e29b-41d4-a716-446655440000",
  "userId": "770e8400-e29b-41d4-a716-446655440000",
  "captchaToken": "03AGdBq27X8kJYZ9..."
}
```

**Output (Success - 200):**
```json
{
  "voteId": "880e8400-e29b-41d4-a716-446655440000",
  "status": "accepted",
  "acceptedAt": "2026-01-15T10:30:00Z"
}
```

**Submit vote Logic flow:**
1. Validate JWT token
2. Verify CAPTCHA token
3. Check duplicate vote
4. Submit to SQS queue for async processing
5. Return acceptance confirmation

**Notes:**
- Vote status is "accepted" (queued), not "counted"
- Actual database write happens asynchronously
- SQS ensures zero vote loss even under failures

#### **Results Service**

**Authentication:** Public endpoints (GET results), JWT required for statistics and WebSocket

##### **Operations**

| Operation | Method | Endpoint | Description |
|-----------|--------|----------|-------------|
| Get Election Results | GET | `/api/v1/results/{electionId}` | Get aggregated results |
| Get Live Count | GET | `/api/v1/results/live/{electionId}` | Get current vote count (polling) |
| Stream Results | WebSocket | `/api/v1/results/stream/{electionId}` | Real-time results via WebSocket |
| Get Statistics | GET | `/api/v1/results/statistics/{electionId}` | Detailed analytics (admin only) |

Exemplos of other components: Batch jobs, Events, 3rd Party Integrations, Streaming, ML Models, ChatBots, etc... 

Recommended Reading: http://diego-pacheco.blogspot.com/2018/05/internal-system-design-forgotten.html

#### 6.1 Contract Documentation


```
Component:Vote Service

OPERATION: Submit a vote
Purpose: Submit a vote for elections.
Preconditions: user authenticated
Postconditions: vote accounted

INPUTS:
- ElectionId
- CandidateId
- UserId
- captchaToken

OUTPUTS:
Success (200):
- voteId
- status
- acceptedAt

Errors:
- 400: Bad Request
- 401: Invalid credentials
- 429: Too many attempts

```

```
Component:Vote Service

OPERATION: Check vote status
Purpose: Check vote status for election.
Preconditions: user authenticated
Postconditions:

INPUTS:
- ElectionId
- UserId

OUTPUTS:
Success (200):
- electionId
- userId
- hasVoted
- voteId
- timestamp

Errors:
- 400: Bad Request
- 401: Invalid credentials
- 429: Too many attempts

```

```
Component:Results Service

OPERATION:  getElectionResults
Purpose:  Get aggregated election results
Preconditions:
Postconditions: Results are cached in Redis for 5 minutes

INPUTS:
- electionId
- includePercentages

OUTPUTS:
Success (200):
- electionId
- electionTitle
- status
- totals (candidateId, name, party, votes, percentage)
- totalVotes
- generatedAt

Errors:
- 400: Bad Request
- 429: Too many attempts

```

```
Component:Results Service

OPERATION:  getLiveCount
Purpose:  Retrieve current vote count for an active election (polling-based).
Preconditions:
Postconditions:

INPUTS:
- electionId

OUTPUTS:
Success (200):
- electionId
- currentVotes
- lastUpdated

Errors:
- 400: Bad Request
- 429: Too many attempts

```

```
Component:Results Service

OPERATION: streamResults
Purpose:  WebSocket endpoint for real-time election result updates.
Preconditions:
Postconditions:

INPUTS:
- electionId
- token

OUTPUTS:
Success (101):
- type
- electionId
- timestamp
- candidates
- candidates

Errors:
- 400: Bad Request
- 429: Too many attempts

```

```
Component:Results Service

OPERATION: getElectionStatistics
Purpose:  Retrieve comprehensive statistical analysis for an election.
Preconditions: Requires authentication (ADMIN or AUDITOR role)
Postconditions:

INPUTS:
- electionId
- includeHourly
- includeRegional

OUTPUTS:
Success (101):
- electionId
- timestamp
- candidates
- statistics

Errors:
- 400: Bad Request
- 429: Too many attempts

```

### 🖹 7. Migrations

IF Migrations are required describe the migrations strategy with proper diagrams, text and tradeoffs.

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

Orchestration: Amazon ECS Fargate

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
