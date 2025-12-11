# Service Communication (Vote, Result, and BackOffice)

- Synchronous communication via HTTP/gRPC
- Asynchronous communication via SNS/SQS
- Purpose of queues
- When to use each type of communication
- Detailed event flow

---

## 1️⃣ How does one service call another?

There are two main ways:

---

## ✔ A) Synchronous Communication (Direct) — HTTP/gRPC

Used when a service needs an immediate response.

### Example

The **Result Service** needs to query votes from the **Vote Service**:

```
Result Service → HTTP → Vote Service → Vote RDS
```

### When to use

- Immediate responses
- Data that must be up-to-date “right now”
- Processes that depend on each other to continue

### Advantages

- Simple to implement
- Immediate response
- Great for quick data lookups

---

## ✔ B) Asynchronous Communication — Events via SNS/SQS

In this model, a service does **not call another directly**.
It simply **publishes an event**, and other services **consume it when they can**.

### Example: vote flow

1. **Vote Service** writes the vote to **Vote RDS**
2. Publishes a `VoteRegistered` event to SNS
3. **Result Service** consumes the event and updates aggregated results
4. **BackOffice Service** consumes the same event for auditing/reporting

### When to use

- Processing tasks in parallel
- Avoiding blocked user requests
- Ensuring processing continues even if another service is offline
- Synchronizing data across microservices

---

## 2️⃣ What is the queue used for?

Queues (SQS) are used for:

### ✔ Decoupling

Vote Service does not depend on Result Service being online.

### ✔ Delivery guarantee

If Result Service is down, the queue stores the messages until it comes back.

### ✔ Scalability

Multiple consumers can process events in parallel.

### ✔ Avoiding overload

Vote Service stays fast even if 10,000 votes arrive at once.

---

## 3️⃣ What is event-based communication (SNS) used for?

- Replicating data across services
- Updating aggregated views
- Creating audit logs in BackOffice
- Decoupling components
- Reducing cross-database dependencies
- Avoiding excessive direct calls between microservices

---

## 4️⃣ Full event flow: `VoteRegistered`

```
Vote Service (ECS)

    ↓ writes vote

Vote RDS

    ↓ publishes event

SNS: VoteRegistered Topic

    ↓ delivers event

Result Service (ECS)

    ↓ updates aggregates

Result RDS

BackOffice Service (ECS)

    ↑ consumes events for auditing/reporting

BackOffice RDS
```

---

## 5️⃣ Synchronous Communication Between Services (HTTP)

```
Vote Service <───HTTP───> Result Service
Vote Service <───HTTP───> BackOffice Service
Result Service <──HTTP──> BackOffice Service
```

---

## 6️⃣ Summary

- Services communicate using **HTTP** when an immediate response is required.
- Services synchronize data **asynchronously** via **SNS/SQS**.
- Each service owns its own database (**RDS**).
- Databases never communicate directly with each other.
- Events keep all services updated without tight coupling.

---

## Ideal Microservices Architecture

- Synchronous communication → **HTTP/gRPC between ECS services**
- Asynchronous communication → **SNS/SQS events**
- Persistence → **One RDS per service**
- Cross-service consistency → **Event-driven (eventual consistency)**
