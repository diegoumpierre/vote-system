# Potential Questions

## 1. How do you guarantee that a person can only vote once?

Two layers of protection:

**Fast path -- Redis:** When a vote arrives, Vote Ingestion runs `EXISTS vote:{electionId}:{userId}` on Redis. If the key exists, the vote is rejected immediately with a 409 Conflict. If it doesn't exist, we set the key with a 48-hour TTL before sending the message to SQS. This handles 99.9% of duplicates at sub-2ms latency.

**Safety net -- PostgreSQL:** The votes table has a unique constraint on `(election_id, user_id)`. If somehow a duplicate gets past Redis (e.g., Redis failover, race condition in a tiny window), the INSERT fails at the database level. The Vote Processor catches this constraint violation and discards the duplicate -- no vote is counted twice.

So even if Redis goes down entirely, the database constraint still guarantees exactly-once voting.

---

## 2. Why do you have an ALB for each service instead of a single balancer for both?

Three reasons:

**Independent scaling:** Vote traffic and Results traffic have very different patterns. During peak voting, Vote Ingestion may need 50+ ECS tasks while Result Service needs only a handful. A shared ALB would mean a single target group or path-based routing mixing both workloads, making auto-scaling policies harder to tune.

**Fault isolation:** If Vote ALB is under heavy load or degraded, Result Service continues serving live results unaffected. A single ALB would be a shared failure point -- a misconfigured health check or a target group issue could take down both services.

**Different health checks and timeouts:** Vote Ingestion needs short timeouts (fast rejection of bad requests), while Result Service WebSocket connections need long-lived connections. Separate ALBs let us configure each independently.

---

## 3. Why 2 services for vote processing instead of 1 service that sends to SQS and consumes from it?

**Scaling independently in opposite directions.** Vote Ingestion is CPU-light and I/O-bound -- it validates the request, checks Redis, and pushes to SQS. It needs to scale out aggressively to handle 250k RPS. Vote Processor is heavier -- it writes to PostgreSQL and updates Redis counters, and its throughput is capped by the database at 20-50k TPS.

If they were a single service, we'd need to scale the whole service to 250k RPS capacity, but 80% of those instances would have idle SQS consumers because the DB can't absorb that many writes. By splitting them, we run many small Vote Ingestion tasks (cheap, stateless) and fewer Vote Processor tasks (sized to match DB throughput).

**Deployment independence.** If we need to change how votes are persisted (e.g., batching strategy, retry logic), we redeploy only Vote Processor without touching the ingestion path. Zero risk to the hot path during a deploy.

**Queue as a buffer.** The SQS queue between them is what absorbs the gap between 250k RPS ingestion and 20-50k TPS database writes. If both were in the same service, we'd need internal buffering logic that SQS already provides for free -- with guaranteed delivery, retries, and DLQ.

---

## 4. How do you know that the current architecture can handle 300M users and 250k RPS?

**Capacity math:**

- 3 regions x 100k RPS per API Gateway = 300k RPS capacity (20% headroom over the 250k target)
- Redis handles 250k EXISTS ops/s for dedup -- well within a single ElastiCache cluster's capability (Redis benchmarks at 500k+ ops/s)
- SQS has no practical throughput limit for standard queues
- PostgreSQL at 20-50k TPS with SQS buffering means we don't need the DB to match ingestion speed -- the queue absorbs the difference

**Validation through load testing:**

We have 4 k6 scenarios specifically designed to prove this:
- Baseline at 50k RPS to confirm normal operation
- Peak at 250k RPS for 5 minutes to validate the actual target load
- Sustained at 100k RPS for 2 hours to catch memory leaks and connection pool exhaustion
- Spike from 0 to 150k in 30 seconds to validate auto-scaling triggers before a live event

**Chaos engineering:**

AWS Fault Injection Simulator tests confirm the system survives AZ failures, database failovers, and Redis node losses without losing votes -- SQS buffers everything until the infrastructure recovers.

**Not all 300M vote at the same time.** The 250k RPS peak assumes a realistic distribution where voting happens over a window of minutes, not all 300M users hitting submit simultaneously. Route 53 latency-based routing distributes load geographically across the 3 regions.
