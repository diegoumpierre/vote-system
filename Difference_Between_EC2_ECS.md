## What they are

- **EC2**: Virtual machines (instances). You manage VMs (OS, kernel, agents).
- **ECS**: Container orchestration service for running Docker tasks/services. ECS schedules containers on instances (EC2) or in serverless mode (Fargate).
- **AWS Fargate**: A serverless compute engine for containers that lets you run containers without managing EC2 instances.

## Level of abstraction

- **EC2** = infrastructure (instance, OS, network, storage).
- **ECS** = platform to run and orchestrate containers (tasks, services, auto-restart, load balancing).

## ECS launch modes

- **Launch type: EC2** — you provision EC2 instances in a cluster and ECS schedules containers onto them.
- **Fargate** — serverless: no instance management, billed per CPU/memory per task.

## Operation & responsibility

- **EC2**: more operational responsibility and control (kernel, security, agents).
- **ECS + Fargate**: less operational overhead; focus on the application.
- **ECS + EC2**: ECS handles orchestration, but you still manage instances.

## Scalability & cost

- **EC2**: cost per instance; may be cheaper for steady workloads (reservations / Spot).
- **Fargate**: cost per resource used by each task; good for variable workloads and lower ops overhead.

## Typical use cases

- Use **EC2** when you need VMs for non-containerized apps or complete control.
- Use **ECS + EC2** when you want container orchestration but need control over instances (custom AMIs, drivers, GPUs).
- Use **ECS + Fargate** when you want to minimize operations and pay per
