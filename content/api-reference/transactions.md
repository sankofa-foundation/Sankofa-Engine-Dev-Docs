---
title: "Transactions"
linkTitle: "Transactions"
weight: 3
description: >
  Submit, query, and track transactions.
---

The Transactions API lets you submit financial operations (debits, credits, transfers, exchanges, mints, and burns), check their processing status, and query transaction history. All transactions are processed asynchronously and are immutable once completed.

For interactive exploration, see the [API Explorer](/api-reference/swagger/).

{{% alert title="Important" color="warning" %}}
All monetary amounts are represented as **strings**, not numbers. This avoids IEEE 754 floating-point precision issues. For example, use `"100.00"` instead of `100.00`. Amounts are limited to 18 integer digits and 18 decimal digits.
{{% /alert %}}

{{% alert title="Request Limits" color="info" %}}
All `POST` requests must include the `Content-Type: application/json` header. The maximum request body size is **1 MB**.
{{% /alert %}}

## Transaction Types

| Type | Description | Example Use Case |
|------|-------------|------------------|
| `credit` | Funds in — increases account balance | Cash deposit, incoming wire |
| `debit` | Funds out — decreases account balance | Withdrawal, fee deduction |
| `transfer` | Move between accounts (pair with `group_id`) | Internal account-to-account transfer |
| `exchange` | Asset swap between token types | Currency conversion |
| `mint` | Create new token supply | Token issuance |
| `burn` | Destroy token supply | Token redemption |

## Single-Leg vs. Multi-Leg Transactions

**Single-leg** transactions operate on one account with no counterpart — a cash deposit (credit) or withdrawal (debit) where funds enter or leave the system.

**Multi-leg** transactions involve two or more accounts and are linked by a shared `group_id`. For example, transferring $250 from Alice to Bob requires a debit leg on Alice's account and a credit leg on Bob's account, both sharing the same `group_id`. The Settlement Service tracks all legs and marks the group as settled once all complete.

## Submit a Transaction

**POST /v1/transactions**

Submits a transaction for asynchronous processing. Returns immediately with a receipt and a `"pending"` status.

### Request — Single-Leg Credit (Cash Deposit)

```bash
curl -X POST https://api.example.com/v1/transactions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "alice@example.com",
    "amount": "1000.00",
    "type": "credit",
    "token_id": "USD",
    "signature": "MEUCIQDx...base64url-encoded...",
    "idempotency_key": "deposit-alice-001"
  }'
```

### Request — Multi-Leg Transfer (Debit Leg)

```bash
curl -X POST https://api.example.com/v1/transactions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "alice@example.com",
    "amount": "250.00",
    "type": "debit",
    "token_id": "USD",
    "group_id": "550e8400-e29b-41d4-a716-446655440000",
    "signature": "MEUCIQDy...base64url-encoded...",
    "idempotency_key": "transfer-001-debit"
  }'
```

### Request Body

| Field | Type | Required | Description |
|---|---|---|---|
| `account_id` | string | Yes | The account identifier. Must be ASCII-only, max 128 characters, allowed characters: `a-zA-Z0-9_.-:` |
| `amount` | string | Yes | Positive decimal amount as a string (e.g., `"250.00"`). Max 18 integer digits, 18 decimal digits. |
| `type` | enum | Yes | One of: `debit`, `credit`, `transfer`, `exchange`, `mint`, `burn` |
| `token_id` | string | No | Fungible token identifier (e.g., `"USD"`, `"EUR"`). References a registered token. |
| `group_id` | string | No | UUID linking multiple transaction legs together for settlement. Required for multi-leg transfers. |
| `signature` | string | Yes | ECDSA P-256 signature of the canonical payload (see [Authentication](/api-reference/authentication/)) |
| `idempotency_key` | string | Yes | Client-generated unique key to prevent duplicate processing |

### Response (202 Accepted)

```json
{
  "txn_id": "txn_01HYX3KPVW9QDMZ6R8F5GT7N4E",
  "account_id": "alice@example.com",
  "status": "pending",
  "shard_id": 7,
  "enqueued_at": "2026-04-03T14:22:01.456Z"
}
```

| Field | Type | Description |
|---|---|---|
| `txn_id` | string | Unique transaction identifier |
| `account_id` | string | The account that submitted the transaction |
| `status` | string | Always `"pending"` on initial submission |
| `shard_id` | integer | The ledger shard assigned to process this transaction |
| `enqueued_at` | string | ISO 8601 timestamp when the transaction was enqueued |

### Idempotency

The `idempotency_key` field is **required** on all transaction submissions. If you submit a transaction with the same key as a previous request, the engine returns the original receipt without re-processing. This makes retries safe across network failures.

### Error Response (400 Bad Request)

```json
{
  "type": "https://api.sankofa.engine/errors/invalid-request",
  "title": "Bad Request",
  "status": 400,
  "detail": "amount must be a valid positive decimal number"
}
```

## Get Transaction Status

**GET /v1/transactions/{id}**

Retrieve the current state of a transaction by its ID.

```bash
curl https://api.example.com/v1/transactions/txn_01HYX3KPVW9QDMZ6R8F5GT7N4E \
  -H "Authorization: Bearer $TOKEN"
```

### Response (200 OK)

```json
{
  "txn_id": "txn_01HYX3KPVW9QDMZ6R8F5GT7N4E",
  "account_id": "alice@example.com",
  "amount": "1000.00",
  "type": "credit",
  "status": "completed",
  "shard_id": 7,
  "token_id": "USD",
  "created_at": "2026-04-03T14:22:01.456Z",
  "audit_hash": "sha256:a3f8c2d1e9b04567890abcdef1234567890abcdef1234567890abcdef12345678"
}
```

### Transaction Statuses

| Status | Description |
|--------|-------------|
| `pending` | Transaction enqueued, waiting for shard worker to process |
| `completed` | Successfully processed, receipt signed, persisted to ledger |
| `failed` | Processing failed (e.g., insufficient balance for debit) |

### Error Response (404 Not Found)

```json
{
  "type": "https://api.sankofa.engine/errors/not-found",
  "title": "Not Found",
  "status": 404,
  "detail": "resource not found"
}
```

## Query Transactions

**GET /v1/transactions**

Search and filter transactions with pagination support.

```bash
curl "https://api.example.com/v1/transactions?account_id=alice@example.com&token_id=USD&type=debit&page_size=50" \
  -H "Authorization: Bearer $TOKEN"
```

### Query Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `account_id` | string | — | Filter by account identifier. Must match account ID format (ASCII, max 128 chars). |
| `date_from` | string | — | Start date (inclusive). Must be RFC 3339 format (e.g., `2026-04-01T00:00:00Z`). |
| `date_to` | string | — | End date (exclusive). Must be RFC 3339 format. |
| `type` | enum | — | Filter by transaction type |
| `token_id` | string | — | Filter by fungible token identifier |
| `token_class` | string | — | Filter by token class |
| `group_id` | string | — | Filter by settlement group ID |
| `amount_min` | string | — | Minimum amount (inclusive). Must be a valid decimal string. |
| `amount_max` | string | — | Maximum amount (inclusive). Must be a valid decimal string. |
| `status` | enum | — | Filter by status (`pending`, `completed`, `failed`) |
| `page_token` | string | — | Cursor for the next page of results (max 512 characters) |
| `page_size` | int | 100 | Number of results per page (max 1000) |

All query parameters are validated server-side. Invalid formats (e.g., non-RFC 3339 dates, non-decimal amounts, or oversized page tokens) return a **400 Bad Request** error.

### Response (200 OK)

```json
{
  "transactions": [
    {
      "txn_id": "txn_01HYX3KPVW9QDMZ6R8F5GT7N4E",
      "account_id": "alice@example.com",
      "amount": "1000.00",
      "type": "credit",
      "token_id": "USD",
      "status": "completed",
      "shard_id": 7,
      "created_at": "2026-04-03T14:22:01.456Z"
    }
  ],
  "next_page_token": "eyJsYXN0X2lkIjoiMTIzNCJ9",
  "total_count": 147
}
```

## Query Transactions by Group

**GET /v1/transactions/group/{groupID}**

Returns all transactions that belong to a settlement group. Use this to verify that all legs of a multi-leg transaction have completed.

```bash
curl https://api.example.com/v1/transactions/group/550e8400-e29b-41d4-a716-446655440000 \
  -H "Authorization: Bearer $TOKEN"
```

### Response (200 OK)

```json
{
  "group_id": "550e8400-e29b-41d4-a716-446655440000",
  "transactions": [
    {
      "txn_id": "txn_01HYX3LPXW0SEBZ7S9G6HU8M5F",
      "account_id": "alice@example.com",
      "amount": "250.00",
      "type": "debit",
      "token_id": "USD",
      "status": "completed"
    },
    {
      "txn_id": "txn_01HYX3MQNW8RDAZ5T7F4HS6K3D",
      "account_id": "bob@example.com",
      "amount": "250.00",
      "type": "credit",
      "token_id": "USD",
      "status": "completed"
    }
  ]
}
```
