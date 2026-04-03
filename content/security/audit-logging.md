---
title: "Audit Logging"
linkTitle: "Audit Logging"
weight: 4
description: >
  Cryptographic audit hash chains, signed receipts, and tamper-evident logging.
---

The Sankofa Engine provides cryptographic guarantees of data integrity through three complementary mechanisms: SHA-256 audit hash chains, ECDSA P-256 signed receipts, and immutable event retention. Together, these mechanisms ensure that every transaction is independently verifiable and any tampering is detectable.

## SHA-256 Audit Hash Chain

Every transaction processed by the Sankofa Engine extends the account's audit hash chain. Each hash incorporates the previous hash, creating an append-only cryptographic log where modification of any entry invalidates all subsequent hashes.

### Hash Chain Construction

```text
  Txn 1                    Txn 2                    Txn 3
    │                        │                        │
    ▼                        ▼                        ▼
┌─────────┐            ┌─────────┐            ┌─────────┐
│ hash(   │            │ hash(   │            │ hash(   │
│  "" ||  │            │  H1 ||  │            │  H2 ||  │
│  txnID  │            │  txnID  │            │  txnID  │
│  acctID │            │  acctID │            │  acctID │
│  amount │            │  amount │            │  amount │
│  type   │            │  type   │            │  type   │
│  ts     │            │  ts     │            │  ts     │
│ )       │            │ )       │            │ )       │
└────┬────┘            └────┬────┘            └────┬────┘
     │                      │                      │
     ▼                      ▼                      ▼
   Hash H1 ──────────▶   Hash H2 ──────────▶   Hash H3
```

### Hash Formula

Each hash in the chain is computed as:

```
hash = SHA-256( prevHash || txnID || accountID || amount || type || timestamp )
```

| Field | Description |
|-------|-------------|
| `prevHash` | The hash of the previous transaction in this account's chain (empty string for the first transaction) |
| `txnID` | Unique transaction identifier |
| `accountID` | The account this transaction belongs to |
| `amount` | Transaction amount |
| `type` | Transaction type (e.g., credit, debit, transfer) |
| `timestamp` | Transaction timestamp |

### Tamper Detection

The chain structure ensures:

- **Modification** of any transaction changes its hash, which breaks the chain for all subsequent transactions.
- **Deletion** of a transaction creates a gap — the next transaction's `prevHash` will not match the preceding transaction's hash.
- **Insertion** of a transaction requires recomputing all subsequent hashes, which would be detected by any verification check.
- **Reordering** of transactions invalidates the `prevHash` linkage.

### Checkpoint-Based Verification

For accounts with large transaction histories, verifying the entire chain from genesis on every check is impractical. The engine supports checkpoint-based verification:

1. Periodic checkpoints record a known-good hash at a specific position in the chain.
2. Verification can start from the most recent checkpoint rather than from the first transaction.
3. Full-chain verification from genesis remains available for comprehensive audits.
4. Checkpoints themselves are signed to prevent checkpoint tampering.

## ECDSA P-256 Signed Receipts

Every transaction processed by the Sankofa Engine receives a digitally signed receipt. The receipt provides independent, cryptographic proof that the transaction was processed by the engine and has not been modified since processing.

### Receipt Contents

Each signed receipt contains:

| Field | Description |
|-------|-------------|
| `group_id` | The transaction group this receipt belongs to |
| `account_id` | The account affected by the transaction |
| `amount` | Transaction amount |
| `type` | Transaction type |
| `audit_hash` | The audit hash chain value after this transaction |
| `shard_id` | The shard that processed the transaction |
| `timestamp` | Time the transaction was processed |

### Signing Process

1. Receipt fields are serialized in a **canonical byte order** to ensure deterministic output regardless of field ordering in memory or serialization format.
2. The canonical byte representation is signed using **ECDSA with the P-256 curve**.
3. The signature is attached to the receipt and returned to the caller.

### Signature Verification

Any party with the engine's public key can verify a receipt:

1. Reconstruct the canonical byte representation from the receipt fields.
2. Verify the ECDSA P-256 signature against the public key.
3. If verification succeeds, the receipt is authentic and unmodified.

### Tamper Evidence

- **Any modification** to any receipt field (amount, timestamp, account, etc.) invalidates the signature.
- **Signatures are non-forgeable** — only the holder of the private signing key can produce valid signatures.
- **Receipts are self-contained** — verification does not require contacting the engine or any online service.

## NFT Provenance

The Sankofa Engine maintains a full ownership history for every NFT instance, providing a complete and independently verifiable provenance chain.

### Provenance Tracking

| Capability | Description |
|------------|-------------|
| Full history | Every transfer event is recorded with source, destination, timestamp, and transaction reference |
| Provenance queries | API endpoints return the complete ownership chain from minting to current holder |
| Independent verification | Each transfer in the chain is backed by a signed receipt and audit hash |
| Immutability | Provenance records are part of the audit hash chain and cannot be altered retroactively |

### Provenance Query

A provenance query returns the ordered list of ownership transfers:

```text
Mint → Owner A    (txn: 001, ts: 2025-01-01T00:00:00Z, receipt: ...)
Owner A → Owner B (txn: 002, ts: 2025-03-15T12:30:00Z, receipt: ...)
Owner B → Owner C (txn: 003, ts: 2025-06-20T09:15:00Z, receipt: ...)
```

Each entry includes the signed receipt for that transfer, enabling end-to-end verification of the entire ownership chain.

## Event Retention

### NATS JetStream Immutable Event Log

All transaction events are published to NATS JetStream, which serves as the engine's immutable event log:

| Parameter | Value |
|-----------|-------|
| Retention period | 7 years (configurable via `max_message_age_seconds: 220898160`) |
| Durability | Durable subscriptions ensure no events are lost |
| Immutability | Events are append-only — published events cannot be modified or deleted within the retention window |
| Replay | Full event replay from any point in time within the retention window |

### Event Replay

The event log supports replaying events from any point in the retention window:

- **Full replay**: Replay all events from the beginning of the retention window to rebuild state.
- **Point-in-time replay**: Replay events from a specific timestamp or sequence number.
- **Filtered replay**: Replay events for a specific account, shard, or transaction type.

Event replay enables disaster recovery, audit reconstruction, and state verification without relying on the primary database.

### Retention Configuration

The default 7-year retention period aligns with common banking and financial services regulatory requirements. The retention period is configurable per deployment to meet specific regulatory or business requirements:

```yaml
max_message_age_seconds: 220898160  # 7 years (default)
```

Customers subject to different retention requirements (e.g., 5 years, 10 years) can adjust this value during deployment configuration.
