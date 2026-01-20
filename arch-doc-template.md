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

List in form of bullets what design principles you want to be followed, it's great to have 5-10 lines.
Example:
```
1. Low Coupling: We need to watch for coupling all times.
2. Flexibility: Users should be able to customize behavior without leaking the internals of the system. Leverage interfaces.
3. Observability: we should expose all key metrics on main features. Sucess and errors counters need to be exposed.
4. Testability: Chaos engineering is a must and property testing. Testing should be done by engineers all times.
5. Cache efficiency: Should leverage SSD caches and all forms of caches as much as possible.
```
Recommended Reading: http://diego-pacheco.blogspot.com/2018/01/stability-principles.html

### 🏗️ 4. Overall Diagrams

Here there will be a bunch of diagrams, to understand the solution.
```
🗂️ 4.1 Overall architecture: Show the big picture, relationship between macro components.
🗂️ 4.2 Deployment: Show the infra in a big picture. 
🗂️ 4.3 Use Cases: Make 1 macro use case diagram that list the main capability that needs to be covered. 
```
Recommended Reading: http://diego-pacheco.blogspot.com/2020/10/uml-hidden-gems.html

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
* Centralized authentication: API Gateway Lambda Authorizer validates JWT tokens once before routing, preventing duplicate auth logic in each service.
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

### 🖹 7. Migrations

IF Migrations are required describe the migrations strategy with proper diagrams, text and tradeoffs.

### 🖹 8. Testing strategy

Explain the techniques, principles, types of tests and will be performaned, and spesific details how to mock data, stress test it, spesific chaos goals and assumptions.

### 🖹 9. Observability strategy

Explain the techniques, principles,types of observability that will be used, key metrics, what would be logged and how to design proper dashboards and alerts.

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
