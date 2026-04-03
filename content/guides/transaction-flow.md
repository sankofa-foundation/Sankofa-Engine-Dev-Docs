---
title: "Transaction Flow"
linkTitle: "Transaction Flow"
weight: 2
description: >
  End-to-end transaction lifecycle from submission to receipt.
---

This guide walks through the complete lifecycle of a transaction in the Sankofa Engine, from the moment a client submits it to the point where the finalized receipt is available for query. Understanding this flow is essential for building reliable integrations, debugging processing delays, and reasoning about failure scenarios.

## Overview

Every transaction passes through seven stages:

```text
  Client                API Gateway           NATS JetStream         Shard Worker
    │                       │                       │                       │
    │  POST /v1/transactions│                       │                       │
    │──────────────────────►│                       │                       │
    │                       │  validate, auth,      │                       │
    │                       │  compute shard        │                       │
    │                       │                       │                       │
    │   202 Accepted        │  publish to           │                       │
    │◄──────────────────────│  LEDGER.shard-{N}     │                       │
    │                       │──────────────────────►│                       │
    │                       │                       │  deliver to worker    │
    │                       │                       │──────────────────────►│
    │                       │                       │                       │  validate balance
    │                       │                       │                       │  update in-memory
    │                       │                       │                       │  extend hash chain
    │                       │                       │                       │  sign receipt
    │                       │                       │                       │  write ScyllaDB
    │                       │                       │◄──────────────────────│  ack
    │                       │                       │                       │
    │                       │                       │         Projection Service
    │                       │                       │──────────────────────►│
    │                       │                       │  receipt event        │  update PostgreSQL
    │                       │                       │                       │
    │  GET /v1/transactions/{txn_id}                │                       │
    │──────────────────────►│  query projection     │                       │
    │   200 OK (finalized)  │                       │                       │
    │◄──────────────────────│                       │                       │
```

## Stage 1: Client Submit

The client sends a `POST /v1/transactions` request to the API Gateway.

```bash
curl -X POST https://api.example.com/v1/transactions \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "alice@example.com",
    "amount": "250.00",
    "type": "debit",
    "signature": "MEUCIQDx...base64url-encoded...",
    "idempotency_key": "txn-20260403-001"
  }'
```

**Key details:**

- The `idempotency_key` is a client-generated unique string. It serves as the basis for deduplication at every layer of the stack.
- The `signature` field contains the ECDSA P-256 signature of the canonical payload (required for self-custody accounts; see the [Self-Custody Guide]({{< relref "self-custody" >}})).
- The request body is validated against a strict JSON schema. Missing required fields or invalid types result in a `400 Bad Request` with detailed error messages.

**On failure:** If the request is malformed, the gateway returns `400` immediately. If authentication fails, the gateway returns `401` or `403`. The transaction is never enqueued.

## Stage 2: API Gateway Processing

The API Gateway performs three operations before enqueueing the transaction:

1. **Request validation** -- the JSON body is validated against the transaction schema. Amounts must be positive decimal strings. The `type` must be one of: `debit`, `credit`, `transfer`, `exchange`, `mint`, `burn`.

2. **Transaction ID generation** -- a deterministic transaction ID is derived from the idempotency key. Submitting the same idempotency key a second time will return the original receipt without re-processing.

3. **Shard assignment** -- the gateway computes `FNV-1a(account_id) % shard_count` to determine which shard will process this transaction. This is a pure function of the account ID, so all transactions for the same account always land on the same shard. This eliminates cross-shard coordination for single-account operations.

The gateway returns `202 Accepted` with the `txn_id` and assigned `shard_id`:

```json
{
  "txn_id": "txn_01HYX3KPVW9QDMZ6R8F5GT7N4E",
  "account_id": "alice@example.com",
  "status": "pending",
  "shard_id": "shard-07",
  "enqueued_at": "2026-04-03T14:22:01.456Z"
}
```

**On failure:** If the gateway cannot reach NATS, it returns `503 Service Unavailable`. The client should retry with the same idempotency key.

## Stage 3: NATS JetStream Publishing

The API Gateway publishes the transaction message to the NATS JetStream subject `LEDGER.shard-{shardID}`.

**Key details:**

- The `Nats-Msg-Id` header is set to the idempotency key. JetStream uses this for broker-level deduplication: if the same `Nats-Msg-Id` is published twice within the deduplication window, JetStream silently discards the duplicate.
- The message is persisted to the JetStream stream with at-least-once delivery guarantees. It remains in the stream until a consumer acknowledges it.
- Each shard has its own subject and consumer group, so messages are processed in order within a shard.

**On failure:** If the gateway publishes but crashes before returning the `202` to the client, the client will retry (same idempotency key). JetStream deduplication prevents the message from being published twice. The client receives the original receipt on retry.

## Stage 4: Shard Worker Block Processing

The assigned Shard Worker receives the message from its NATS subscription and processes it within a block:

1. **Read balance** -- the worker reads the account's current balance from its in-memory cache. This cache is populated from ScyllaDB on worker startup and kept up-to-date as transactions are processed.

2. **Validate** -- the worker performs business-rule validation:
   - For **debits**: the account must have sufficient balance. If the balance is insufficient, the transaction is rejected.
   - **Amount parsing**: the amount string is parsed to a fixed-precision decimal. Invalid amounts are rejected.
   - **Idempotency check**: if the worker has already processed this idempotency key, the original receipt is returned.

3. **Update balance** -- the in-memory balance is updated (decremented for debits, incremented for credits).

4. **Extend hash chain** -- the SHA-256 audit hash chain is extended:
   ```
   new_hash = SHA-256(previous_hash || block_data)
   ```
   This creates a tamper-evident chain where any modification to a historical block invalidates all subsequent hashes.

5. **Batch into block** -- multiple transactions may be batched into a single block for write efficiency. The block contains all transaction data, updated balances, and the new audit hash.

**On failure:** If the worker crashes mid-processing, the NATS message is not acknowledged. JetStream redelivers it to another worker instance (or the same worker after restart). The idempotency check prevents double-processing.

## Stage 5: Receipt Signing

After block processing, the Shard Worker signs a receipt for each transaction in the block:

1. The worker constructs a canonical receipt payload containing:
   - `group_id` -- the settlement group identifier
   - `account_id` -- the account that submitted the transaction
   - `amount` -- the transaction amount
   - `type` -- the transaction type
   - `audit_hash` -- the new hash chain value
   - `shard_id` -- the shard that processed the transaction
   - `timestamp` -- the finalization time

2. The payload is signed using **ECDSA P-256** with a signing key retrieved from OpenBao (HashiCorp Vault fork).

3. The signed receipt and block data are batch-inserted into **ScyllaDB** in a single write operation.

4. Only after the ScyllaDB write succeeds does the worker acknowledge the original NATS message (**ack-after-commit**).

```json
{
  "txn_id": "txn_01HYX3KPVW9QDMZ6R8F5GT7N4E",
  "account_id": "alice@example.com",
  "amount": "250.00",
  "type": "debit",
  "status": "finalized",
  "shard_id": "shard-07",
  "group_id": "grp_01HYX3MQNW8RDAZ5T7F4HS6K3D",
  "audit_hash": "sha256:a3f8c2d1e9b04567890abcdef1234567890abcdef1234567890abcdef12345678",
  "receipt_signature": "MEYCIQDy...base64url-encoded...",
  "finalized_at": "2026-04-03T14:22:02.112Z"
}
```

**On failure:** If the ScyllaDB write fails, the NATS message is not acknowledged. JetStream redelivers the message, and the worker reprocesses it. Because the idempotency key check runs against ScyllaDB (which did not commit), the reprocessing produces the correct result.

## Stage 6: Projection Update

The **Projection Service** subscribes to the receipt stream on NATS JetStream. For each finalized receipt, it updates the PostgreSQL read model:

- Account balances are updated to reflect the new state.
- Transaction history records are inserted for query access.
- The projection is a CQRS read model -- optimized for fast queries but eventually consistent with the authoritative ledger in ScyllaDB.

**Idempotency:** Projections are idempotent. Replaying the same receipt produces the same state in PostgreSQL. This means the Projection Service can safely be restarted or replayed from any point in the NATS stream without corrupting the read model.

**On failure:** If the Projection Service crashes, it resumes from its last acknowledged position in the NATS stream. Unprocessed receipts are replayed, and PostgreSQL converges to the correct state. During the gap, balance queries may return stale data, but no data is lost.

## Stage 7: Client Poll

The client polls `GET /v1/transactions/{txn_id}` to retrieve the finalized transaction:

```bash
curl https://api.example.com/v1/transactions/txn_01HYX3KPVW9QDMZ6R8F5GT7N4E \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

**Response (200 OK):**

```json
{
  "txn_id": "txn_01HYX3KPVW9QDMZ6R8F5GT7N4E",
  "account_id": "alice@example.com",
  "amount": "250.00",
  "type": "debit",
  "status": "finalized",
  "shard_id": "shard-07",
  "group_id": "grp_01HYX3MQNW8RDAZ5T7F4HS6K3D",
  "token_id": "USD-COIN",
  "token_class": "fungible",
  "enqueued_at": "2026-04-03T14:22:01.456Z",
  "finalized_at": "2026-04-03T14:22:02.112Z",
  "audit_hash": "sha256:a3f8c2d1e9b04567890abcdef1234567890abcdef1234567890abcdef12345678"
}
```

The `audit_hash` field allows the client to independently verify that the transaction was included in the hash chain. The `status` field will be one of:

| Status      | Meaning                                                  |
|-------------|----------------------------------------------------------|
| `pending`   | Transaction is enqueued but not yet processed by a shard worker |
| `finalized` | Transaction has been processed, persisted, and signed    |
| `rejected`  | Transaction failed validation (e.g., insufficient balance) |

## Exactly-Once Semantics

The Sankofa Engine guarantees exactly-once transaction processing through three complementary mechanisms working at different layers:

| Mechanism              | Layer          | How It Works                                                                                         |
|------------------------|----------------|------------------------------------------------------------------------------------------------------|
| **Idempotency Keys**   | Application    | Every transaction carries a client-supplied idempotency key. The Shard Worker rejects duplicates before processing. |
| **NATS JetStream Dedup** | Broker       | The `Nats-Msg-Id` header is set to the idempotency key. JetStream discards duplicate publishes within its deduplication window. |
| **Ack-After-Commit**   | Infrastructure | The NATS message is acknowledged only after the ScyllaDB batch write succeeds. If the worker crashes before ack, JetStream redelivers; the idempotency check prevents double-processing. |

These three layers together ensure that network retries, broker redeliveries, and worker restarts never result in duplicate ledger entries.

## Failure Scenarios Summary

| Failure Point                          | What Happens                                                                                 | Client Action                         |
|----------------------------------------|----------------------------------------------------------------------------------------------|---------------------------------------|
| Client network failure before 202      | Transaction may or may not have been enqueued                                                | Retry with same idempotency key       |
| Gateway crashes after NATS publish     | Transaction is enqueued; 202 was not returned                                                | Retry with same idempotency key (dedup) |
| NATS JetStream unavailable             | Gateway returns 503                                                                          | Retry with backoff                    |
| Shard Worker crashes mid-processing    | NATS redelivers after timeout; idempotency check prevents double-processing                  | Poll for status; it will finalize     |
| ScyllaDB write failure                 | NATS message is not acked; redelivered and reprocessed                                       | Poll for status; it will finalize     |
| Projection Service crashes             | Balances may be stale; receipts are not lost; projection catches up on restart               | Balance queries may lag briefly        |
| Worker rejects transaction (e.g., insufficient balance) | Transaction status becomes `rejected` with a reason | Check status; handle rejection        |
