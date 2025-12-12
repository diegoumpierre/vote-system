# Circuit Breaker

## What It Is

A circuit breaker is a resilience pattern that prevents cascading failures in distributed systems.
It monitors requests to external services and "breaks" the circuit when failures exceed a threshold.

The pattern has three states:

1. **CLOSED** (normal operation): Requests flow through normally. Failures are counted. If the failure threshold is reached, the circuit opens.

2. **OPEN** (failure protection): Requests immediately fail fast without attempting the call. After a timeout period, the circuit transitions to half-open to test recovery.

3. **HALF-OPEN** (recovery testing): A limited number of test requests are allowed through. If they succeed, the circuit closes. If they fail, it reopens.

This prevents your system from wasting resources on operations that are likely to fail and gives failing services time to recover.

## How It Fits in the Architecture

**Auth0 Integration:**
- Single point of failure for authentication
- If Auth0 goes down, circuit breaker prevents 250k RPS from piling up with timeout failures
- Fallback: serve cached JWT validations or enable read-only public access

**Database Writes:**
- Aurora/RDS maxes out at 10k-20k write IOPS
- Under 250k RPS load, circuit breaker prevents retry storms when database saturates
- Fallback: queue writes to SQS for async processing

**Cache Layer (Redis/DynamoDB):**
- Deduplication lookups that could fail under load
- Circuit breaker prevents connection pool exhaustion
- Fallback: temporarily skip deduplication (accept duplicate vote risk vs complete outage)

**Inter-Service Communication:**
- ECS services calling each other (Vote Validator → Vote Ingestion → Results Aggregator)
- Circuit breaker prevents one failing service from cascading to others
- Fallback: cached responses, degraded functionality

## Practical Example: Auth0 Outage Scenario

Without circuit breaker:
1. Auth0 goes down
2. Each authentication request waits for 30s timeout
3. 250,000 requests/second × 30s = 7.5M blocked threads
4. Connection pools exhausted
5. Entire system crashes

With circuit breaker:
1. Auth0 goes down
2. First 5 requests fail (failure threshold)
3. Circuit opens—all subsequent requests fail fast in <1ms
4. System serves cached authentication decisions or public-only access
5. After 60s timeout, circuit tests recovery with limited requests
6. When Auth0 recovers, circuit gradually closes with controlled ramp-up

Result: **Graceful degradation instead of complete system failure**

## Configuration Guidelines

Circuit breakers should be tuned based on the dependency's criticality and recovery characteristics:

**Auth0 Circuit Breaker:**
- Failure threshold: 3 (fast-fail on external service)
- Timeout: 60s (longer recovery time)
- Half-open requests: 2
- Fallback: cached JWT validation, read-only mode

**Database Write Circuit Breaker:**
- Failure threshold: 10 (tolerate transient slowness)
- Timeout: 10s (quick recovery testing)
- Half-open requests: 5
- Fallback: SQS queue for async writes

**Redis/DynamoDB Circuit Breaker:**
- Failure threshold: 5
- Timeout: 5s (fast recovery for cache)
- Half-open requests: 3
- Fallback: skip deduplication temporarily

**Inter-Service Circuit Breaker:**
- Failure threshold: 5
- Timeout: 15s
- Half-open requests: 3
- Fallback: cached responses or reduced functionality

## Benefits at Scale

**Resource Conservation:**
- Failed requests return in <1ms vs waiting 30s for timeout
- Threads and connections freed immediately
- CPU and memory not wasted on doomed operations

**Prevents Thundering Herd:**
- When service recovers, half-open state provides controlled ramp-up
- Not all 250k requests hit the recovered service simultaneously
- Gradual load increase allows service to stabilize

**Improved User Experience:**
- Fast-fail = immediate error response to client
- Graceful degradation = partial functionality vs complete outage
- "Service temporarily unavailable" vs hanging requests

**Observability:**
- Circuit state metrics provide real-time health dashboard
- Open circuits = immediate alerts for operations team
- Half-open transitions = automatic recovery detection
- CloudWatch metrics → SNS → PagerDuty integration

**Cascading Failure Prevention:**
- Vote Validator failure doesn't kill Results Aggregator
- Database saturation doesn't crash API Gateway
- Auth0 outage doesn't exhaust connection pools across all services