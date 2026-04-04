---
title: "Data Model"
linkTitle: "Data Model"
weight: 3
description: >
  Domain entities, field definitions, and key design decisions.
---

## Entities

### Transaction

The central entity in the Sankofa Engine. Represents any ledger operation -- debits, credits, transfers, exchanges, minting, and burning.

| Field | Type | Description |
|---|---|---|
| `account_id` | string | Account identifier. ASCII-only, max 128 characters, allowed: `a-zA-Z0-9_.-:` |
| `amount` | string | Transaction amount (string, not float -- see [Amounts as Strings](#amounts-as-strings)). Max 18 integer + 18 decimal digits. |
| `type` | enum | Transaction type: `debit`, `credit`, `transfer`, `exchange`, `mint`, `burn` |
| `status` | enum | Processing status: `pending`, `completed`, `failed` |
| `signature` | string | ECDSA P-256 signature from the submitting client |
| `shard_id` | uint32 | Assigned shard (computed via FNV-1a hash of `account_id`) |
| `idempotency_key` | string | Client-provided deduplication key |
| `group_id` | string (optional) | UUID linking related transactions (e.g., both sides of a transfer) |
| `token_id` | string (optional) | Associated fungible token identifier |
| `token_class` | string (optional) | Associated NFT class identifier |
| `audit_hash` | string | SHA-256 hash chain entry for tamper detection |
| `receipt_signature` | bytes | ECDSA P-256 DER-encoded receipt signature from the engine |
| `created_at` | datetime | Creation timestamp |

### FungibleToken

Represents a fungible token definition (e.g., a currency or point system).

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique token identifier |
| `symbol` | string | Token symbol (e.g., `USD`, `POINTS`) |
| `decimals` | integer | Decimal precision, 0--18 |
| `issuer` | string | Account ID of the token issuer |
| `total_supply` | string | Total supply (string for precision -- see [Amounts as Strings](#amounts-as-strings)) |
| `status` | enum | Token status: `active`, `frozen`, `deprecated` |
| `created_at` | datetime | Creation timestamp |

### NFTClass

Defines a class (template) for non-fungible tokens.

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique class identifier |
| `name` | string | Human-readable class name |
| `description` | string | Class description |
| `issuer` | string | Account ID of the class creator |
| `metadata_schema` | JSON Schema | JSON Schema defining valid metadata for instances of this class |
| `status` | enum | Class status: `active` |
| `created_at` | datetime | Creation timestamp |

### NFTInstance

An individual non-fungible token, minted from an NFTClass.

| Field | Type | Description |
|---|---|---|
| `class_id` | string | Parent NFTClass identifier |
| `instance_id` | string | Unique instance identifier within the class |
| `owner_account` | string | Current owner's account ID |
| `metadata` | JSON | Instance-specific metadata (validated against class `metadata_schema`) |
| `status` | enum | Instance status: `active`, `burned` |
| `minted_at` | datetime | Minting timestamp |

### SignedReceipt

A cryptographically signed proof that the engine processed a transaction. Returned to clients as confirmation.

| Field | Type | Description |
|---|---|---|
| `group_id` | string | Group identifier linking related transactions |
| `account_id` | string | Account that submitted the transaction |
| `amount` | string | Transaction amount |
| `type` | enum | Transaction type |
| `audit_hash` | string | Hash chain entry at time of processing |
| `shard_id` | uint32 | Shard that processed the transaction |
| `timestamp` | datetime | Processing timestamp |
| `signature` | bytes | ECDSA P-256 DER-encoded engine signature over receipt fields |

## Key Design Decisions

### Amounts as Strings

IEEE 754 floating-point arithmetic cannot represent decimal fractions exactly. For example, `0.1 + 0.2` produces `0.30000000000000004` in most languages. Financial systems require exact decimal arithmetic to avoid rounding errors that compound over millions of transactions.

All amounts in the Sankofa Engine are represented as **strings** (e.g., `"100.50"`, not `100.50`). Arithmetic is performed using arbitrary-precision decimal libraries, never floating-point.

### FNV-1a Shard Routing

Transactions are assigned to shards deterministically using the FNV-1a hash of the account ID:

```
shard_id = FNV-1a(account_id) % shard_count
```

This approach ensures that all transactions for a given account are routed to the same shard, enabling per-account ordering guarantees without a centralized lookup service.

### Idempotency

Every transaction submission **must** include a client-provided `idempotency_key`. If a duplicate key is received, the engine returns the original receipt without reprocessing. This key is also used as the NATS `Nats-Msg-Id` header, leveraging NATS JetStream's built-in server-side deduplication window.

### Audit Hash Chain

Each transaction's `audit_hash` is computed as a chained SHA-256 hash over the previous hash and the current transaction fields:

```
SHA-256(prevHash || idempotencyKey || accountID || amount || type || createdAt_RFC3339Nano)
```

This creates a per-account append-only hash chain. Any tampering with historical records breaks the chain, making unauthorized modifications detectable.
