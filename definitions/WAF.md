# AWS WAF (Web Application Firewall)

## What It Is

AWS WAF is a web application firewall that helps protect your applications from common web exploits and attacks. It allows you to create custom rules to filter web traffic based on conditions like IP addresses, HTTP headers, request body, URI strings, SQL injection patterns, and cross-site scripting (XSS) attempts.

WAF operates at Layer 7 (application layer), inspecting HTTP/HTTPS requests before they reach your application. Unlike traditional firewalls that work at network layers, WAF understands application-level threats and can make intelligent decisions about which requests to allow, block, or count.

## How It Works

WAF sits between users and your application infrastructure. The typical flow looks like:

```
User Request
    ↓
Cloudflare (optional CDN layer)
    ↓
AWS WAF (inspection happens here)
    ↓
ALB / API Gateway / CloudFront
    ↓
Your Application (ECS, EC2, Lambda)
```

**Mechanism:** You define Web ACLs (Access Control Lists) containing rules that are evaluated in priority order. Each rule contains conditions (what to inspect) and actions (allow, block, count, CAPTCHA). WAF inspects incoming requests in real-time, matching them against your rules. If a request triggers a BLOCK rule, it never reaches your application—the user receives a 403 Forbidden response.

### Rule Types

- **Managed Rules**: Pre-configured rule sets maintained by AWS or marketplace vendors (e.g., OWASP Top 10, known bad inputs, bot control)
- **Rate-based Rules**: Block IPs exceeding request thresholds (e.g., 2000 requests per 5 minutes)
- **Custom Rules**: Your own conditions using string matching, regex, geo-blocking, IP sets
- **IP Reputation Lists**: Automatically block known malicious IPs

## Pros / Cons

| Aspect | Pros | Cons |
|--------|------|------|
| **Security** | Real-time protection against OWASP Top 10, DDoS mitigation, bot protection | Requires tuning to avoid false positives blocking legitimate users |
| **Integration** | Native integration with ALB, API Gateway, CloudFront, AppSync | Only protects AWS-hosted resources; not portable to other clouds |
| **Customization** | Highly flexible rule creation, regex support, custom responses | Complex rule authoring requires security expertise |
| **Cost** | Pay-per-use model, no upfront commitments | Can become expensive at scale (per million requests + per rule) |
| **Visibility** | Detailed CloudWatch metrics and logs, sampled requests | Logs must be sent to S3/CloudWatch (additional costs) |
| **Performance** | Minimal latency impact (~1-5ms), scales automatically | Rate-based rules have 5-minute evaluation windows (not instant) |

## When to Use WAF

Use AWS WAF when you need to:

- **Protect against common web exploits**: SQL injection, XSS, CSRF
- **Block malicious traffic**: Known bad IPs, bot traffic, scraping attempts
- **Rate limiting**: Prevent abuse, API throttling, DDoS mitigation
- **Geo-blocking**: Restrict access based on country/region
- **Compliance requirements**: PCI-DSS, HIPAA often require WAF protection
- **Custom security logic**: Block specific patterns, headers, or request characteristics

### Don't Use WAF If

- You only need network-level protection (use Security Groups/NACLs instead)
- You have extremely high traffic and cost is prohibitive (consider alternatives like Cloudflare)
- Your application is internal-only with no public internet exposure

## Practical Example: Voting System Protection

For a high-traffic voting platform, you'd typically configure:

1. **AWS Managed Rules for Core Protection**
   - Enable "Core Rule Set" (OWASP Top 10 protection)
   - Enable "Known Bad Inputs" (CVE patterns, malware signatures)

2. **Rate-Based Rules**
   - Block IPs exceeding 100 votes per 5 minutes (prevent ballot stuffing)
   - Block IPs exceeding 1000 requests per 5 minutes (DDoS protection)

3. **Geo-Blocking** (if applicable)
   - Allow only specific countries if voting is region-restricted

4. **Bot Control** (optional, additional cost)
   - Challenge suspicious automated traffic with CAPTCHA
   - Block verified bots attempting to scrape or manipulate votes

5. **Custom Rules**
   - Block requests without proper authentication headers
   - Rate-limit specific endpoints like `/api/vote` separately

### Example Rule Configuration

```
Web ACL: VotingSystemProtection
  Priority 1: AWS Managed - Core Rule Set (Block)
  Priority 2: AWS Managed - Known Bad Inputs (Block)
  Priority 3: Rate Limit - 100 requests/5min to /api/vote (Block)
  Priority 4: Rate Limit - 1000 requests/5min global (Block)
  Priority 5: Allow all remaining traffic
```

## Pricing

AWS WAF charges for:

- **Web ACL**: $5.00 per month per Web ACL
- **Rules**: $1.00 per month per rule (managed rules may have additional costs)
- **Requests**: $0.60 per million requests processed
- **Bot Control** (optional): $10.00/month + $1.00 per million requests
- **CAPTCHA**: $0.40 per 1,000 challenge attempts

### Cost Estimation Example

For a voting system with 50 million requests/month:
- Web ACL: $5.00
- 5 rules: $5.00
- 50M requests: $30.00
- **Total**: ~$40/month (basic protection)

With Bot Control added: ~$100/month

### Cost Optimization Tips

- Use AWS Managed Rules instead of building everything custom (saves time, often same cost)
- Start with COUNT mode for new rules, analyze logs, then switch to BLOCK (avoid false positives)
- Use IP sets for bulk blocking instead of multiple individual IP rules
- Consider Cloudflare as WAF alternative for extremely high-traffic scenarios (flat pricing)

## Integration with Other Services

AWS WAF integrates directly with:
- **ALB**: Protect HTTP/HTTPS traffic to load balancers
- **API Gateway**: Protect REST and WebSocket APIs
- **CloudFront**: Global edge protection for CDN content
- **AppSync**: Protect GraphQL APIs
- **Cognito**: Can trigger WAF rules based on authentication state

Logs can be sent to:
- **S3**: Long-term storage and compliance
- **CloudWatch Logs**: Real-time monitoring and alerting
- **Kinesis Firehose**: Stream to analytics platforms

## Conclusion

AWS WAF is essential for any public-facing application with security or compliance requirements. For a voting system, it's particularly critical because:
- Elections are high-value targets for manipulation
- Traffic patterns can be unpredictable (election day spikes)
- Regulatory requirements often mandate web application protection
- Reputation damage from security breaches is catastrophic

The key is starting with managed rules for baseline protection, then layering custom rules based on your application's specific threat model. Monitor logs closely in the first weeks, tune rules to minimize false positives, and adjust rate limits based on legitimate usage patterns.

For a kata/architectural exercise, WAF demonstrates understanding of defense-in-depth: it's not enough to have authentication and authorization—you need protection at multiple layers, and WAF is your first line of defense against web-based attacks.
