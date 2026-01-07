# Observability Infrastructure for Voting System

## What is Observability?

Observability is the ability to monitor, measure, and understand the state of a system by examining its outputs, logs, and performance metrics. Essential for detecting issues in real-time, tracing root causes, and auto-remediating problems before users notice.

## The Three Pillars of Observability

1. **Logs** - Record of events and actions (what happened, when, and where)
2. **Metrics** - Numerical measurements over time (CPU, memory, request rates, latency)
3. **Traces** - End-to-end journey of requests through distributed services

![alt text](./images/cloudwatch.png)

## Services

| Service | Description |
|---------|-------------|
| **CloudWatch Logs** | Centralized log aggregation from ECS services, API Gateway, and CloudTrail. Stores application logs, access logs, and audit trails with long-term archival to S3. |
| **CloudWatch Metrics** | Collects performance metrics from ECS containers, API Gateway, RDS, and SQS. Monitors CPU, memory, request rates, queue depth, and custom business metrics. |
| **Container Insights** | Provides detailed metrics and diagnostics for ECS tasks, including container restart failures and resource utilization at cluster, service, and task levels. |
| **CloudWatch Alarms** | Threshold-based alerts that trigger Auto Scaling, EventBridge events, or notifications when metrics breach defined limits (CPU > 80%, queue backlog, error rates). |
| **X-Ray** | Distributed tracing service that tracks requests through API Gateway, Vote Service, SQS, Results Service, and RDS. Identifies bottlenecks and latency sources. |
| **Grafana** | Custom dashboards combining multiple data sources (CloudWatch, RDS Performance Insights) with advanced visualizations and flexible alerting to Slack or PagerDuty. |
| **EventBridge** | Event bus that routes CloudWatch Alarms to automated remediation actions via Systems Manager (restart tasks, scale services, block IPs via WAF). |
| **Systems Manager** | Executes automated remediation runbooks triggered by EventBridge events (flush connections, scale ECS tasks, restart unhealthy containers). |

## Sources

### AWS Documentation
- **Redhat:**: https://www.redhat.com/en/topics/devops/what-is-observability
- **Three pillars of observability:**: https://www.crowdstrike.com/en-us/cybersecurity-101/observability/three-pillars-of-observability/#:~:text=of%20system%20issues.-,What%20are%20the%20three%20pillars%20of%20observability?,that%20interfere%20with%20business%20objectives.
- **CloudWatch:** https://docs.aws.amazon.com/cloudwatch/
- **CloudWatch Logs:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html
- **CloudWatch Metrics:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/working_with_metrics.html
- **X-Ray:** https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html
- **EventBridge:** https://docs.aws.amazon.com/eventbridge/latest/userguide/what-is-amazon-eventbridge.html
- **Systems Manager:** https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html
- **Container Insights:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html
- **CloudTrail:** https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html
- **Auto Scaling:** https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html
