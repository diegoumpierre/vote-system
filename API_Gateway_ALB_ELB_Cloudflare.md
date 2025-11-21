# AWS: API Gateway vs ALB vs ELB + Cloudflare Overview

## API Gateway
API Gateway is an AWS service used to create, publish, maintain, monitor, and secure REST and WebSocket APIs. It acts as a centralized entry point for clients, handling communication between external consumers and backend services.

### Key Capabilities
- Traffic management  
- Authorization and access control  
- Monitoring and logging  
- API version management  
- Request/response transformation  
- Throttling and rate limiting  

API Gateway is commonly used for:
- Public APIs  
- Serverless applications (Lambda)  
- Fine-grained access control  
- API lifecycle management  

---

## Application Load Balancer (ALB)
The Application Load Balancer is a Layer 7 AWS load balancer designed to distribute HTTP and HTTPS traffic. It understands application-level content (headers, paths, hostnames, cookies), enabling intelligent routing.

### What ALB Provides
- Path-based and host-based routing  
- Header and cookie-based routing  
- Load distribution across EC2, ECS, EKS, or IP targets  
- Health checks  
- WebSocket support  
- Basic authentication via Amazon Cognito  

ALB is ideal for:
- Microservices  
- Containerized applications  
- Routing logic based on URL or headers  

---

## Elastic Load Balancing (ELB)
Elastic Load Balancing is the overarching AWS service that includes several load balancer types. The term "ELB" is often used ambiguously.

### ELB Includes
- **CLB (Classic Load Balancer)** — older, limited L4/L7  
- **ALB (Application Load Balancer)** — modern L7  
- **NLB (Network Load Balancer)** — ultra-fast L4  

### General Features
- Automatic traffic distribution  
- Health checks  
- Scalability and availability  

---

## API Gateway vs ALB vs ELB

These services all handle incoming client requests, but they operate at different layers and serve different purposes.

### Comparison by Layer

| Service | Layer | Primary Purpose |
|--------|--------|----------------------------|
| NLB / CLB (ELB family) | L4 | Route traffic to the correct machine or container |
| ALB | L7 | Route HTTP/HTTPS traffic based on application data |
| API Gateway | API Layer | Manage, secure, and expose APIs |

### Conceptual Flow
These services are **not** always used sequentially. Architectures typically use:
- **API Gateway + Lambda** for serverless APIs  
- **ALB** for containerized workloads (ECS/EKS)  
- **NLB** for high-performance network-level routing  

---

## Cloudflare
Cloudflare is a globally distributed reverse proxy and CDN that sits in front of your application. It accelerates performance and enhances security by caching static content and serving it from the closest edge location.

### Main Functions
- Global CDN  
- Reverse proxy  
- DDoS mitigation  
- WAF (Web Application Firewall)  
- DNS management  
- Performance optimization  
- Caching (static + optional dynamic)  

Cloudflare usually sits **before AWS**, handling edge caching and security before traffic reaches your infrastructure.

---
