# Payment System Design

A high-level system design for a payment processing platform, split along CAP-theorem boundaries: a strongly consistent ledger for money movement and an eventually consistent async tier for side-effects. The request path is horizontally scalable via a stateless payment cluster behind an L7 load balancer; exactly-once semantics are preserved end-to-end through an idempotency store on the ingress side and a transactional outbox on the egress side.

## Architecture

![Architecture](Architecture.png)

The source diagram is in [`Architecture.drawio`](Architecture.drawio) (edit at [app.diagrams.net](https://app.diagrams.net)).

## Flow

1. **Actor** initiates a transaction at a client terminal.
2. The request enters through the **API Gateway** (auth, routing, rate limiting).
3. The **L7 Load Balancer** distributes the request across the payment cluster.
4. A node in the **Payment Service Cluster (Stateless)** validates the request.
5. The node **checks** the **Idempotency Store** using the request's idempotency key — duplicates short-circuit with the prior result and never reach the ledger.
6. The node executes an **atomic transaction** against the **Ledger Layer (CP System)** that writes to both the **Ledger DB** and the **Outbox Table** in a single commit.
7. After the commit, the node **saves** the result to the **Idempotency Store** so future retries return the same response.
8. The **Outbox Worker** polls the Outbox Table and publishes committed events to **SQS**, turning the atomic DB write into a reliable event.
9. The **Async Service (AP System, Idempotent)** consumes events and runs **Notification** and **Audit** workflows.
10. Failed messages are retried **3–5 times**; exhausted messages land in the **Dead Letter Queue** for inspection.

## Components

| Component | Role | Tier |
|---|---|---|
| API Gateway | Request entry point, auth, routing | Edge |
| L7 Load Balancer | Application-layer routing across payment nodes | Edge |
| Payment Service Cluster | Stateless transaction orchestration and validation | Core |
| Idempotency Store | Deduplicates retried requests via idempotency keys | Core |
| Ledger DB | Source of truth for balances and transactions | Ledger (CP) |
| Outbox Table | Records events in the same transaction as the ledger write | Ledger (CP) |
| Outbox Worker | Relays committed outbox rows to SQS | Messaging |
| SQS | Durable decoupling between core and async work | Messaging |
| Notification Service | Sends transaction notifications | Async (AP) |
| Audit Service | Records audit trail for compliance | Async (AP) |
| Dead Letter Queue | Captures messages that fail after 3–5 retries | Messaging |
| Monitoring | Metrics + alerts across all services | Observability |
| Centralized Logging System | Aggregated structured logs | Observability |
| Distributed Tracing | End-to-end request tracing across services | Observability |

## Design Rationale

- **Stateless payment cluster**: keeping the service tier stateless lets the L7 load balancer route freely, allows horizontal scaling, and makes node failures recoverable without session loss.
- **Idempotency store**: clients and networks retry — without a dedup layer, retries would double-charge. The store sits outside the ledger transaction so a duplicate can be rejected before any money moves.
- **CP ledger**: money movement requires strong consistency — the ledger cannot tolerate split-brain or lost writes.
- **Transactional outbox**: publishing to SQS directly after a ledger commit risks a lost event if the node crashes mid-publish. Writing the event to an Outbox Table in the same DB transaction makes it atomic with the ledger write; the Outbox Worker then relays rows to SQS with at-least-once semantics.
- **Idempotent async consumers**: since delivery is at-least-once (both from the outbox relay and from SQS redelivery), the Notification and Audit services must be idempotent to avoid duplicate emails or audit rows.
- **Bounded retries + DLQ**: transient failures retry 3–5 times; permanently poisoned messages are quarantined in the DLQ instead of blocking the queue, and can be replayed after investigation.
- **Observability as a cross-cutting concern**: monitoring, centralized logging, and distributed tracing span every tier so operators can correlate a failed payment from the edge through the ledger and async workflows.
