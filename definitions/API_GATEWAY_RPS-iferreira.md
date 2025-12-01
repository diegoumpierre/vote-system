# AWS API Gateway - RPS Limits and Quotas

## Overview

Amazon API Gateway enforces **account-level throttling quotas per Region** to protect the service and ensure fair resource allocation. These limits apply across all API types (HTTP APIs, REST APIs, WebSocket APIs) within a single AWS account and region.

Understanding these limits is critical when designing high-scale systems like a voting platform that requires **250,000 RPS**.

---

## Account-Level Quotas per Region

### Standard Regions

| Resource / Operation | Default Quota | Burst Capacity | Can Be Increased? |
|---------------------|---------------|----------------|-------------------|
| **Throttle quota per account per Region**<br/>(Across HTTP APIs, REST APIs, WebSocket APIs, and WebSocket callback APIs) | **10,000 RPS** | **5,000 requests**<br/>(Token bucket algorithm) | **Yes**<br/>(Via AWS Support) |
| **Throttle quota without access control**<br/>(Portal without authorization) | **250,000 RPS** | N/A | **No** |
| **Throttle quota with access control**<br/>(Portal with authorization) | **10,000 RPS** | N/A | **No** |

### Limited Regions (Lower Default Quotas)

The following regions have **reduced default quotas**:

| Region | Default RPS | Default Burst | Can Be Increased? |
|--------|-------------|---------------|-------------------|
| Africa (Cape Town) | **2,500 RPS** | **1,250 requests** | Yes |
| Europe (Milan) | **2,500 RPS** | **1,250 requests** | Yes |
| Europe (Spain) | **2,500 RPS** | **1,250 requests** | Yes |
| Europe (Zurich) | **2,500 RPS** | **1,250 requests** | Yes |
| Asia Pacific (Jakarta) | **2,500 RPS** | **1,250 requests** | Yes |
| Asia Pacific (Hyderabad) | **2,500 RPS** | **1,250 requests** | Yes |
| Asia Pacific (Melbourne) | **2,500 RPS** | **1,250 requests** | Yes |
| Asia Pacific (Malaysia) | **2,500 RPS** | **1,250 requests** | Yes |
| Asia Pacific (Thailand) | **2,500 RPS** | **1,250 requests** | Yes |
| Middle East (UAE) | **2,500 RPS** | **1,250 requests** | Yes |
| Israel (Tel Aviv) | **2,500 RPS** | **1,250 requests** | Yes |
| Canada West (Calgary) | **2,500 RPS** | **1,250 requests** | Yes |
| Mexico (Central) | **2,500 RPS** | **1,250 requests** | Yes |

---

## Understanding Burst Capacity

### What is Token Bucket Algorithm?

API Gateway uses a **token bucket algorithm** to handle burst traffic:

```
Token Bucket Capacity: 5,000 tokens (standard regions)
Refill Rate: 10,000 tokens/second

Example burst scenario:
- System is idle (bucket full: 5,000 tokens)
- Sudden spike: 15,000 requests in 1 second
  ├─ First 5,000 requests: Use bucket tokens ✅
  ├─ Next 10,000 requests: Use refill rate ✅
  └─ Remaining 0 requests: Would be throttled ❌

Result: 15,000 RPS sustained for 1 second, then drops to 10,000 RPS
```

### Key Points:

| Aspect | Details |
|--------|---------|
| **Burst capacity** | Determined by API Gateway service team (not customer-controllable) |
| **Refill rate** | Equals the sustained RPS quota (10,000 RPS → 10,000 tokens/sec) |
| **Bucket size** | Maximum tokens that can accumulate (5,000 for standard regions) |
| **Throttling behavior** | Requests exceeding quota return **HTTP 429 Too Many Requests** |

---

## Increase Process table

| Step | Action | Timeline | Maximum Achievable |
|------|--------|----------|-------------------|
| 1 | Open AWS Support case (Service Quotas) | Immediate | N/A |
| 2 | Justify business need (expected traffic, use case) | 1-2 business days | Up to **100,000 RPS** (typical approval) |
| 3 | AWS reviews account health, payment history | 2-5 business days | Rarely above 100k RPS per region |
| 4 | Quota increase applied | Immediate after approval | Hard limit: ~**120,000 RPS** per region |

### Notes:

**Burst capacity is NOT adjustable** - It scales proportionally with the RPS quota but remains under AWS control.
**Portal quotas (250k RPS without access control) are NOT increasable** - These are service-wide limits.

---

## Implications for 250,000 RPS Voting System

### Problem:

```
Required: 250,000 RPS (peak voting traffic)
API Gateway limit: 10,000 RPS (default) or ~100,000 RPS (with increase)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Gap: 150,000 RPS still unhandled ❌
```

---

## Acronyms

| Acronym | Meaning |
|---------|---------|
| **RPS** | Requests Per Second |
| **API** | Application Programming Interface |
| **HTTP** | Hypertext Transfer Protocol |
| **REST** | Representational State Transfer |
| **AWS** | Amazon Web Services |
| **ALB** | Application Load Balancer |
| **ECS** | Elastic Container Service |
| **CDN** | Content Delivery Network |
| **WAF** | Web Application Firewall |

---

## Sources


- **API Gateway Quotas:** https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html
- **Service Quotas Console:** https://console.aws.amazon.com/servicequotas/
- **Request Quota Increase:** https://console.aws.amazon.com/support/home#/case/create
- **Throttling Best Practices:** https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html
