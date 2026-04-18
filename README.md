# System Design

A collection of complex system design studies — architecture diagrams, design rationales, and trade-off notes for non-trivial distributed systems.

Each subdirectory is a self-contained design for one system, with its own diagram (`.drawio` + `.png`) and write-up.

## Designs

| Folder | System |
|---|---|
| [PaymentSystemDesign](./PaymentSystemDesign) | Distributed payment processing platform (CP ledger + AP async tier, transactional outbox, idempotency) |

## Conventions

- **Diagrams** are authored in [diagrams.net / draw.io](https://app.diagrams.net) and committed as both `.drawio` (source) and `.png` (rendered).
- **README** in each folder explains the flow, components, and the *why* behind the design choices — not just the *what*.
- Designs are reasoned through CAP-theorem and availability/consistency trade-offs where relevant.

## Scope

These are design exercises, not production implementations. The goal is to explore how real-world systems handle scale, failure, consistency, and observability — and to document the trade-offs clearly enough that the reasoning holds up under review.
