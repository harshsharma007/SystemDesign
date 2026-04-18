# Payment System Design

A high-level system design for a payment processing platform, split along CAP-theorem boundaries: a strongly consistent ledger for money movement and an eventually consistent async tier for side-effects. The request path is horizontally scalable via a stateless payment cluster behind an L7 load balancer.

## Architecture

![Architecture](Architecture.png)

The source diagram is in [`Architecture.drawio`](Architecture.drawio) (edit at [app.diagrams.net](https://app.diagrams.net)).

## Flow

1. **Actor** initiates a transaction at a client terminal.
2. The request enters through the **API Gateway** (auth, routing, rate limiting).
3. The **L7 Load Balancer** distributes the request across the payment cluster.
4. A node in the **Payment Service Cluster (Stateless)** validates and orchestrates the transaction.
5. State is committed to the **Ledger DB** inside the **Ledger Layer (CP System)**.
6. The payment node publishes an event to **SQS** to fan out downstream work.
7. The **Async Service (AP System)** consumes events and runs **Notification** and **Audit** workflows.

## Components

| Component | Role | Tier |
|---|---|---|
| API Gateway | Request entry point, auth, routing | Edge |
| L7 Load Balancer | Application-layer routing across payment nodes | Edge |
| Payment Service Cluster | Stateless transaction orchestration and validation | Core |
| Ledger DB | Source of truth for balances and transactions | Ledger (CP) |
| SQS | Durable decoupling between core and async work | Messaging |
| Notification Service | Sends transaction notifications | Async (AP) |
| Audit Service | Records audit trail for compliance | Async (AP) |

## Design Rationale

- **Stateless payment cluster**: keeping the service tier stateless lets the L7 load balancer route freely, allows horizontal scaling, and makes node failures recoverable without session loss.
- **CP ledger**: money movement requires strong consistency — the ledger cannot tolerate split-brain or lost writes.
- **AP async**: notifications and audit trails can tolerate brief delays in exchange for higher availability and decoupling.
- **SQS as the seam**: isolates failure between tiers so an async outage never blocks a payment.
