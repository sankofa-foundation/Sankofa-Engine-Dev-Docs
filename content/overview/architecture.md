---
title: "Architecture"
linkTitle: "Architecture"
weight: 2
description: >
  Hexagonal architecture, service topology, and data flow.
---

## Architectural Style

Sankofa Engine follows **Hexagonal Architecture** (Ports & Adapters) combined with **Domain-Driven Design** (DDD). This separation ensures the business logic remains independent of infrastructure concerns and can be tested, reviewed, and evolved without touching database drivers, message brokers, or HTTP frameworks.

The codebase is organized into four layers:

| Layer | Path | Responsibility |
|---|---|---|
| **Domain Core** | `internal/core/` | Pure business logic and domain models. Zero infrastructure dependencies. |
| **Ports** | `internal/core/port/` | Go interfaces that define contracts between the domain and the outside world (e.g., `LedgerRepository`, `EventPublisher`). |
| **Adapters** | `internal/adapter/` | Concrete implementations of ports — ScyllaDB repositories, NATS publishers, OpenBao key providers, etc. |
| **Services** | `internal/service/` | Application services that orchestrate domain operations, coordinate across ports, and enforce transaction boundaries. |

Dependencies always point inward: adapters depend on ports, ports depend on domain types, and the domain core depends on nothing external.

## Service Topology

All services run inside a Kubernetes cluster. The following diagram shows the primary communication paths:

```text
                         ┌─────────────────────────────────────────────┐
                         │              Kubernetes Cluster             │
                         │                                             │
  Clients ──────────────►│  ┌──────────────────┐                       │
                         │  │   API Gateway     │                       │
                         │  │   :8080 (HTTP)    │                       │
                         │  │   :9090 (health)  │                       │
                         │  └────────┬─────────┘                       │
                         │           │                                  │
                         │           ▼                                  │
                         │  ┌──────────────────┐      ┌──────────────┐ │
                         │  │      NATS         │◄────►│   Shard      │ │
                         │  │   (JetStream)     │      │ Orchestrator │ │
                         │  └──┬───┬───┬───┬───┘      └──────────────┘ │
                         │     │   │   │   │                            │
                         │     ▼   │   │   ▼                            │
                         │  ┌──────┐   │  ┌────────────┐               │
                         │  │Shard │   │  │ Projection  │               │
                         │  │Worker│   │  │  Service    │───►PostgreSQL │
                         │  │ (x3) │   │  └────────────┘               │
                         │  └──┬───┘   │                                │
                         │     │       ▼                                │
                         │     │  ┌────────────┐   ┌──────────────┐    │
                         │     │  │ Compliance  │   │  Settlement  │    │
                         │     │  │  Service    │   │   Service    │    │
                         │     │  └────────────┘   └──────────────┘    │
                         │     │                                        │
                         │     ▼                                        │
                         │  ScyllaDB          OpenBao                   │
                         │                                              │
                         │  ┌────────────┐                              │
                         │  │  Archival   │───► S3 / Local Storage      │
                         │  │  (CronJob)  │                             │
                         │  └────────────┘                              │
                         └──────────────────────────────────────────────┘
```

## Transaction Data Flow

Every transaction passes through five stages from submission to archival.

### Stage 1 — Submission

1. A client sends `POST /v1/transactions` to the API Gateway.
2. The gateway validates the request against the JSON schema, authenticates the caller (JWT or ECDSA signature), and enforces RBAC policies via Casbin.
3. A deterministic transaction ID is generated from the caller-supplied idempotency key.
4. The validated request is published to NATS JetStream with `Nats-Msg-Id` set to the idempotency key for broker-level deduplication.

### Stage 2 — Shard Routing

1. The API Gateway computes `FNV-1a(account_id) % shard_count` to select the target shard.
2. The message is published to the shard-specific NATS subject.
3. Routing is fully deterministic — no cross-shard coordination, no distributed locks, no consensus protocol required for normal operation.

### Stage 3 — Block Processing

1. The assigned Shard Worker receives the message from its NATS subscription.
2. It reads the account's current balance from its in-memory cache (populated from ScyllaDB on startup).
3. The transaction is validated: sufficient balance, no duplicate idempotency key, business rule checks.
4. The balance is updated in-memory.
5. The SHA-256 hash chain is extended: `new_hash = SHA-256(previous_hash || block_data)`.
6. An ECDSA P-256 receipt is signed using a key retrieved from OpenBao.
7. The block (containing one or more transactions) is batch-inserted into ScyllaDB.
8. The signed receipt is published to NATS for downstream consumers.
9. The original NATS message is acknowledged only after the ScyllaDB write succeeds (ack-after-commit).

### Stage 4 — Projection

1. The Projection Service subscribes to the receipt stream on NATS.
2. For each receipt, it updates the PostgreSQL balance views — maintaining a CQRS read model optimized for queries.
3. Projections are idempotent: replaying the same receipt produces the same state.

### Stage 5 — Archival

1. A Kubernetes CronJob runs the Archival Service nightly.
2. The service evaluates retention policies and identifies transactions older than the configured hot-storage window (default 2 years).
3. Aged transactions are offloaded from ScyllaDB to cold storage (S3-compatible object store or local filesystem).
4. Archival root references are recorded so that archived data can be retrieved and verified against the hash chain.

## Exactly-Once Semantics

Sankofa Engine guarantees exactly-once transaction processing through three complementary mechanisms:

| Mechanism | Layer | How It Works |
|---|---|---|
| **Idempotency Keys** | Application | Every transaction request carries a caller-supplied idempotency key. The Shard Worker rejects duplicates before processing. |
| **NATS JetStream Dedup** | Broker | The `Nats-Msg-Id` header is set to the idempotency key. JetStream discards duplicate publishes within its deduplication window. |
| **Ack-After-Commit** | Infrastructure | The NATS message is acknowledged only after the ScyllaDB batch write succeeds. If the worker crashes before ack, JetStream redelivers the message; the idempotency check prevents double-processing. |

Together these layers ensure that network retries, broker redeliveries, and worker restarts never result in duplicate ledger entries.

## Storage Architecture

| Tier | Technology | Retention | Purpose |
|---|---|---|---|
| **Hot** | ScyllaDB | Configurable (default 2 years) | Active ledger blocks, hash chains, and signed receipts. High-throughput writes. |
| **Projection** | PostgreSQL | Current state | CQRS read model — account balances, transaction history views, query-optimized indexes. |
| **Cold** | S3 / Local filesystem | Long-term | Archived transactions offloaded by the Archival Service. Verifiable against hash chain roots. |
| **Event Log** | NATS JetStream | 7 years | Durable event stream for all published messages. Supports replay, audit, and compliance requirements. |
