# EKS (Elastic Kubernetes Service)

## What It Is

EKS is AWS's managed Kubernetes service. It runs the Kubernetes control plane for you, the brain that schedules containers, manages state, and handles API requests. You focus on deploying workloads, AWS handles the master nodes, etcd, and control plane availability.

## Why EKS?

- **Container orchestration**: You have 50 containers. Where do they run? What happens when one crashes? How do they find each other? Kubernetes solves this.
- **Declarative infrastructure**: You describe the desired state ("I want 3 replicas of this service"), Kubernetes makes it happen and keeps it that way.
- **Portability**: Kubernetes runs anywhere, like AWS, GCP, Azure, on-prem. Your workloads aren't locked to one cloud.

## Core Concepts

### Control Plane (Managed by AWS)
- **API Server**: Where you send kubectl commands
- **etcd**: Stores cluster state
- **Scheduler**: Decides which node runs which pod
- **Controller Manager**: Maintains desired state (restarts crashed pods, etc.)

You don't touch any of this. AWS runs it across multiple AZs with automatic failover.

### Data Plane (Managed by You)
This is where your containers actually run. You have options:

- **EC2 nodes**: You provision EC2 instances that join the cluster. Full control, you manage scaling and patching.
- **Managed Node Groups**: AWS provisions and manages EC2 instances for you. Less control, less work.
- **Fargate**: Serverless. No nodes to manage. AWS runs each pod in its own isolated environment. Pay per pod resources.

### Key Objects
- **Pod**: Smallest deployable unit. Usually one container, sometimes sidecar patterns.
- **Deployment**: Manages pod replicas and rolling updates.
- **Service**: Stable network endpoint to reach a set of pods.
- **Ingress**: Routes external HTTP traffic to services (usually via ALB Ingress Controller).

## How It Fits in the Architecture

For a voting system on EKS:

```
Route 53
    ↓
ALB (via AWS Load Balancer Controller)
    ↓
Ingress → Service → Pods (vote-api)
                  → Pods (auth-service)
                  → Pods (fraud-detection)
    ↓
RDS / DynamoDB / ElastiCache
```

Each microservice runs as a Deployment with its own Service. The ALB Ingress Controller automatically provisions ALBs based on your Ingress resources.