# Route 53

## What It Is

Route 53 is AWS's DNS service. It translates human-readable domain names (like `votingsystem.com`) into IP addresses that machines use to communicate. The name comes from port 53, the standard DNS port.

But Route 53 is more than a simple DNS—it also handles domain registration, health checks, and traffic routing policies that can make or break your application's availability.


## How It Fits in the Architecture

Route 53 is the **first thing** a user hits when accessing your application. The flow typically looks like:

```
User types votingsystem.com
    ↓
Route 53 resolves to IP/ALB endpoint
    ↓
Cloudflare (if using CDN/WAF)
    ↓
ALB / API Gateway
    ↓
Your application (EC2, ECS, Lambda)
```

For a voting system with global reach, you'd likely use:
- **Latency-based routing** to send users to the nearest region
- **Health checks** on each regional ALB
- **Failover** as a backup if an entire region goes down

## Practical Example: Multi-Region Voting Platform

Let's say you have voting infrastructure in `us-east-1` and `eu-west-1`:

1. Create a hosted zone for `votingsystem.com`
2. Create health checks for both regional ALBs
3. Create latency-based records pointing to each ALB
4. Route 53 automatically routes Brazilian users to `us-east-1` (closer) and German users to `eu-west-1`
5. If `eu-west-1` dies, health check fails, traffic automatically shifts to `us-east-1`

This happens at the DNS level—no code changes, no manual intervention.

## Pricing

Route 53 charges for:
- **Hosted zones**: ~$0.50/month per zone
- **Queries**: ~$0.40 per million queries (standard), more for latency-based/geo
- **Health checks**: ~$0.50/month per endpoint (basic), more for HTTPS or string matching
- **Domain registration**: Varies by TLD (.com ~$12/year)

For most applications, Route 53 costs are negligible compared to compute. The value it provides for availability is worth far more than the few dollars per month.

## Route 53 vs Cloudflare DNS

Both do DNS, but:
- **Route 53** integrates natively with AWS services (ALB, S3, CloudFront alias records)
- **Cloudflare** offers free tier, built-in CDN, and DDoS protection at the edge

You can use both together: Cloudflare as your public-facing DNS and CDN, with Route 53 handling internal AWS routing (private hosted zones).