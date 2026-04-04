---
title: "Services"
linkTitle: "Services"
weight: 3
description: >
  Microservices that comprise the Sankofa Engine.
---

Sankofa Engine is composed of seven microservices plus a shared health-check contract. Each service is independently deployable and communicates with other services exclusively through NATS JetStream.

## API Gateway

| Property | Value |
|---|---|
| **Port** | 8080 (HTTP) / 9090 (health) |
| **Replicas** | 2 + HPA |
| **Dependencies** | NATS |

The API Gateway is the public REST API entry point for all client interactions with the Sankofa Engine.

**Key Responsibilities:**

- Request validation against JSON schemas
- JWT authentication and ECDSA request signature verification
- RBAC policy enforcement via Casbin
- Rate limiting per tenant and per endpoint
- FNV-1a shard routing — deterministically maps `account_id` to the correct shard subject
- NATS RPC proxying — publishes validated requests to JetStream and returns signed receipts to callers

## Shard Worker

| Property | Value |
|---|---|
| **Port** | 9090 (health only) |
| **Replicas** | 3 + HPA |
| **Dependencies** | NATS, ScyllaDB, OpenBao |

Shard Workers are the core transaction-processing units. Each worker is assigned one or more shards by the Shard Orchestrator and processes all transactions routed to those shards.

**Key Responsibilities:**

- Block processing — batches transactions into blocks for efficient storage
- In-memory balance cache — maintains current account balances for fast validation without ScyllaDB reads on the hot path
- SHA-256 hash chains — extends a per-shard tamper-evident chain with each new block
- ECDSA P-256 receipt signing — produces a cryptographic receipt for every processed block using keys from OpenBao
- ScyllaDB batch writes — persists blocks, updated balances, and hash chain entries in atomic batch operations

## Shard Orchestrator

| Property | Value |
|---|---|
| **Port** | 9090 (health only) |
| **Replicas** | 3 (quorum) |
| **Dependencies** | NATS |

The Shard Orchestrator manages the lifecycle and assignment of shards across the pool of Shard Workers.

**Key Responsibilities:**

- Leader election among orchestrator replicas to ensure a single source of truth for shard assignments
- Shard assignment and rebalancing when workers join, leave, or fail health checks
- Publishing the shard map to NATS KV so that the API Gateway and workers can discover assignments
- Continuous worker health monitoring and automatic shard reassignment on failure

## Projection Service

| Property | Value |
|---|---|
| **Port** | 9090 (health only) |
| **Replicas** | 2 + HPA |
| **Dependencies** | NATS, PostgreSQL |

The Projection Service implements the read side of the CQRS pattern, maintaining query-optimized views of ledger state in PostgreSQL.

**Key Responsibilities:**

- Consuming block and settlement events from NATS JetStream
- Updating PostgreSQL balance projection tables with current account state
- Maintaining idempotent projections — replaying the same event produces identical state
- Supporting balance queries, transaction history lookups, and reporting views

## Compliance Service

| Property | Value |
|---|---|
| **Port** | 9090 (health only) |
| **Replicas** | 2 + HPA |
| **Dependencies** | NATS |

The Compliance Service generates and verifies zero-knowledge proofs that allow third parties to validate ledger properties without accessing raw financial data.

**Key Responsibilities:**

- **Proof-of-liabilities** — proves that reported liabilities match the ledger without revealing individual account balances
- **Proof-of-provenance** — proves the origin and chain of custody for specific assets
- **Proof-of-compliance** — proves adherence to regulatory rules (e.g., reserve requirements) without exposing transaction details
- **Proof verification** — validates previously generated proofs for auditors and regulators

## Settlement Service

| Property | Value |
|---|---|
| **Port** | 9090 (health only) |
| **Replicas** | 1 |
| **Dependencies** | NATS |

The Settlement Service coordinates multi-leg (compound) transactions and handles settlement finalization.

**Key Responsibilities:**

- Compound transaction registration — tracks all legs of a multi-leg transaction by `group_id`
- Receipt tracking — records signed receipts for each leg, with deduplication to prevent double-counting
- Settlement finalization — marks the group as settled once all expected legs have completed
- Partial revert — if a compound transaction must be rolled back, the service publishes compensating transactions for each completed leg. Compensation continues even if some legs fail to revert, and the group is marked `reverted` with a partial-failure report.

## Archival Service

| Property | Value |
|---|---|
| **Port** | N/A (CronJob) |
| **Replicas** | K8s CronJob (nightly) |
| **Dependencies** | ScyllaDB, S3 / Local storage |

The Archival Service runs as a Kubernetes CronJob and manages the lifecycle of ledger data across storage tiers.

**Key Responsibilities:**

- Retention policy evaluation — identifies transactions and blocks that have aged past the configured hot-storage window
- Cold storage offload — moves aged data from ScyllaDB to S3-compatible object storage or local filesystem
- Archival root references — records references that link archived data back to the hash chain, ensuring archived transactions remain cryptographically verifiable

## Health Checks

All services expose health endpoints on **port 9090**:

| Endpoint | Purpose |
|---|---|
| `/healthz/liveness` | Indicates the process is running and not deadlocked. Used by Kubernetes liveness probes to restart unhealthy pods. |
| `/healthz/readiness` | Indicates the service is ready to accept traffic. Readiness checks verify downstream dependencies: ScyllaDB connectivity for Shard Workers, PostgreSQL connectivity for the Projection Service, and NATS connectivity for all services. The NATS health check verifies active connection status and reports `unhealthy` during disconnection periods. |

Kubernetes probes are configured with appropriate thresholds so that a transiently unavailable dependency triggers traffic removal (readiness failure) before pod restart (liveness failure), preventing cascading restarts during brief infrastructure blips.
