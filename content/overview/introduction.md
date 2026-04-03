---
title: "Introduction"
linkTitle: "Introduction"
weight: 1
description: >
  What is the Sankofa Engine and what problems does it solve.
---

## What is Sankofa Engine

Sankofa Engine is a **sharded, privacy-preserving financial ledger engine** purpose-built for digital assets. It delivers cryptographically auditable transaction processing with zero-knowledge proof capabilities, enabling organizations to operate compliant, high-throughput ledger infrastructure without exposing sensitive financial data.

Traditional ledger systems force a choice between transparency and privacy. Sankofa Engine eliminates that trade-off: every transaction is recorded in a tamper-evident hash chain and signed with a cryptographic receipt, while encrypted balances and ZKP assertions ensure that only authorized parties can observe account state.

## Key Capabilities

| Capability | Description |
|---|---|
| **Sharded Ledger** | Deterministic FNV-1a hash routing distributes accounts across independent shards for horizontal scalability. |
| **Privacy-Preserving** | Encrypted balances combined with zero-knowledge proof assertions protect sensitive financial data at rest and in transit. |
| **Cryptographic Auditability** | SHA-256 hash chains and ECDSA P-256 signed receipts provide tamper-evident, independently verifiable transaction histories. |
| **Multi-Asset Support** | Native support for fungible tokens and NFTs within a unified ledger model. |
| **Exactly-Once Processing** | NATS JetStream message deduplication paired with application-level idempotency keys guarantees each transaction is processed exactly once. |
| **Regulatory Ready** | SOC 2 aligned controls, 7-year event retention, and built-in compliance proof generation for audit and regulatory reporting. |

## How It Works

Sankofa Engine processes every transaction through six deterministic stages:

1. **Submit** — A client sends a transaction request via the REST API. Authentication uses JWT tokens or ECDSA request signing. The API Gateway validates the request schema, enforces RBAC policies, and generates a transaction ID from the caller-supplied idempotency key.

2. **Route** — The API Gateway computes `FNV-1a(account_id) % shard_count` to determine the target shard. This deterministic routing requires no cross-shard coordination and ensures all operations for a given account land on the same shard.

3. **Process** — The assigned Shard Worker receives the transaction via NATS JetStream, reads the in-memory balance cache, validates the transaction (sufficient balance, duplicate check), updates the account balance, and batches the transaction into the current block.

4. **Sign** — The Shard Worker extends the shard's SHA-256 hash chain with the new block, signs a receipt using ECDSA P-256, batch-inserts the block into ScyllaDB, publishes the signed receipt to NATS, and acknowledges the original message.

5. **Project** — The Projection Service subscribes to signed receipts and updates PostgreSQL balance views, maintaining a CQRS read model that supports fast balance queries without touching the ledger directly.

6. **Prove** — The Compliance Service generates zero-knowledge proofs (proof-of-liabilities, proof-of-provenance, proof-of-compliance) on demand, enabling regulatory reporting and third-party audits without revealing underlying account data.

## Deployment Model

Sankofa Engine is **managed by Sankofa Labs Inc.** The source code is available for enterprise review under a proprietary license. The platform is Kubernetes-native, designed to run on any conformant K8s cluster with standard operators for ScyllaDB, PostgreSQL, and NATS.

A **free non-commercial version** is planned for early 2027, enabling developers and open-source projects to run the engine in non-production environments.

## Technology Stack

| Component | Technology |
|---|---|
| HTTP Framework | Go + Fiber v3 |
| Ledger Storage | ScyllaDB 6.2 |
| Projection Store | PostgreSQL 16 |
| Messaging | NATS 2.11+ with JetStream |
| Key Management | OpenBao |
| Authorization | Casbin v2 (RBAC) |
| Authentication | golang-jwt/v5 |
| Encryption | AES-GCM-256 + KMS envelope encryption |
