---
title: "Data Residency & Retention"
linkTitle: "Data Residency"
weight: 5
description: >
  Data storage tiers, retention policies, and archival architecture.
---

The Sankofa Engine uses a multi-tier storage architecture to balance performance, cost, and regulatory compliance. This page documents where data resides, how long it is retained, and how it moves between tiers.

## Storage Tiers

| Tier | Technology | Purpose | Default Retention |
|------|-----------|---------|-------------------|
| **Hot** | ScyllaDB 6.2 | Transaction ledger, account state | Configurable (default 2 years) |
| **Projection** | PostgreSQL 16 | CQRS read model (balances, query views) | Current state (always up-to-date) |
| **Cold** | S3 / Local filesystem | Archived transactions | Long-term (configurable) |
| **Event Log** | NATS JetStream | Transaction events, signed receipts | 7 years |

### Tier Descriptions

**Hot Tier (ScyllaDB 6.2)**
The hot tier stores the active transaction ledger and account state. ScyllaDB provides low-latency reads and writes for real-time transaction processing. Data remains in the hot tier for the configured retention period (default 2 years) before becoming eligible for archival.

**Projection Tier (PostgreSQL 16)**
The projection tier maintains CQRS read models — materialized views of account balances and other derived state. This tier always reflects the current state and is continuously updated as transactions are processed. It does not store historical transactions; it stores the computed result of applying all transactions.

**Cold Tier (S3 / Local Filesystem)**
The cold tier stores archived transactions that have aged out of the hot tier. Cold storage provides cost-effective long-term retention. The storage backend is configurable — S3 for cloud deployments, local filesystem for on-premises deployments.

**Event Log (NATS JetStream)**
The event log retains all transaction events and signed receipts for 7 years by default. It serves as the system of record for event replay, audit reconstruction, and disaster recovery.

## Data Classification

The Sankofa Engine classifies data to apply appropriate encryption keys and retention policies:

| Classification | Examples | Storage Location | Encryption |
|---------------|----------|-----------------|------------|
| Financial | Transaction amounts, balances, ledger entries | Hot, Cold, Event Log | AES-GCM-256 (financial DEK) |
| Identity | Account identifiers, customer references | Hot, Projection | AES-GCM-256 (identity DEK) |
| Operational | Health metrics, system events, processing logs | Event Log | AES-GCM-256 (operational DEK) |
| Audit | Hash chain values, signed receipts | Hot, Event Log | Integrity-protected (signed) |

### Data Flow Between Tiers

```text
                    ┌─────────────┐
  Incoming Txn ────▶│  NATS       │  Event published (retained 7 years)
                    │  JetStream  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Shard      │  Transaction processed
                    │  Worker     │
                    └──┬─────┬───┘
                       │     │
              ┌────────▼┐  ┌─▼──────────┐
              │ ScyllaDB │  │ PostgreSQL │
              │ (Hot)    │  │(Projection)│
              └────┬─────┘  └────────────┘
                   │
                   │  After retention period
                   │
              ┌────▼─────┐
              │ S3 / FS  │
              │ (Cold)   │
              └──────────┘
```

## Retention Policies

### Default Retention Periods

| Tier | Default Retention | Configurable |
|------|-------------------|-------------|
| Hot (ScyllaDB) | 2 years | Yes — per deployment |
| Projection (PostgreSQL) | Indefinite (current state) | N/A — always reflects current state |
| Cold (S3 / Filesystem) | Indefinite | Yes — per deployment |
| Event Log (NATS JetStream) | 7 years | Yes — `max_message_age_seconds` |

### Customizing Retention

Retention periods are configured at deployment time. Customers can adjust retention to meet their specific regulatory requirements:

- **Banking regulations** may require 5-7 year retention of financial transaction records.
- **Anti-money laundering (AML)** requirements may mandate specific retention windows for transaction data.
- **Tax compliance** may require retention of financial records for a defined period after the tax year closes.
- **Data minimization** regulations (e.g., GDPR) may require shorter retention for certain data categories.

Retention policies are evaluated nightly by the Archival Service.

## Archival Process

### Archival Service

The Archival Service runs as a Kubernetes CronJob on a nightly schedule:

| Step | Description |
|------|-------------|
| 1. Policy evaluation | The service evaluates retention policies for each shard to identify transactions that have exceeded the hot tier retention period |
| 2. Batch selection | Eligible transactions are selected in batches to limit resource consumption |
| 3. Cold storage write | Transactions are written to the cold storage backend (S3 or local filesystem) with their encryption intact |
| 4. Root reference | An archival root reference is stored in the hot tier, enabling cross-tier queries |
| 5. Hot tier cleanup | Archived transactions are removed from ScyllaDB after successful cold storage write is confirmed |
| 6. Verification | The service verifies that archived data is readable from cold storage before completing the cycle |

### Cross-Tier Queries

Queries seamlessly bridge hot and cold tiers:

- The query engine first checks the hot tier (ScyllaDB) for matching records.
- If the query time range extends beyond the hot tier retention period, the engine follows archival root references to retrieve data from cold storage.
- Results from both tiers are merged and returned as a unified response.
- The caller does not need to know which tier the data resides in.

## Data Deletion

### Purge Capabilities

The Sankofa Engine supports data deletion for regulatory compliance:

| Method | Description | Use Case |
|--------|-------------|----------|
| Retention-based expiry | Data automatically ages out of each tier per configured retention policies | Standard lifecycle management |
| Cryptographic erasure | Destroy the DEK for a data classification — all data encrypted with that key becomes permanently unrecoverable | Right-to-erasure requests, decommissioning |
| Explicit purge | API-driven removal of specific records from all tiers | Targeted data disposal |

### Cryptographic Erasure

Cryptographic erasure is the fastest and most complete method of data disposal:

1. The KMS destroys the DEK for the target data classification or account.
2. All data encrypted with that DEK becomes immediately and permanently unrecoverable.
3. The ciphertext may remain in storage but is indistinguishable from random data without the key.
4. This approach satisfies data disposal requirements without requiring physical media destruction or individual record deletion.

### Retention Policy Enforcement

The Archival Service enforces retention policies automatically:

- Transactions exceeding the hot tier retention period are archived to cold storage.
- Cold storage data exceeding the cold tier retention period (if configured) is deleted.
- Event log messages exceeding the NATS JetStream retention period are automatically purged by NATS.
- All deletion events are logged for audit purposes.

## Geographic Considerations

### Deployment-Dependent Data Residency

Data residency in the Sankofa Engine is determined by the deployment configuration, not by the application code:

| Factor | Configuration |
|--------|--------------|
| Compute location | Kubernetes cluster region and zone selection |
| Storage location | ScyllaDB node placement, S3 bucket region, NATS cluster location |
| Network boundaries | Kubernetes network policies and cloud provider VPC configuration |
| Replication | ScyllaDB replication factor and topology-aware placement |

### Customer-Configurable Residency

Customers can specify data residency requirements during deployment:

- **Region selection**: Deploy the entire stack in a specific geographic region (e.g., US-East, EU-West, AP-Southeast).
- **Node affinity**: Kubernetes node affinity rules ensure pods run only on nodes in the target geography.
- **Storage class selection**: Kubernetes storage classes can be configured to use region-specific storage backends.
- **Cross-region restrictions**: Network policies can prevent data from leaving the designated region.

### Regulatory Alignment

| Regulation | Residency Requirement | Engine Support |
|------------|----------------------|----------------|
| GDPR | Data processing within EEA or approved jurisdictions | Region-specific deployment |
| Data localization laws | Data must remain within national borders | Single-region deployment with node affinity |
| Banking regulations | Transaction records in regulated jurisdiction | Storage class and node affinity configuration |
| Cross-border transfer | Restrictions on data leaving the jurisdiction | Network policy enforcement |

Sankofa Labs works with customers to configure deployments that meet their specific data residency obligations.
