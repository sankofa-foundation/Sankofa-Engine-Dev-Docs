---
title: "Transactions"
linkTitle: "Transactions"
weight: 3
description: >
  Submit, query, and track transactions.
---

The Transactions API lets you submit financial operations (debits, credits, transfers, exchanges, mints, and burns), check their processing status, and query transaction history. All transactions are processed asynchronously and are immutable once finalized.

For interactive exploration, see the [API Explorer](/api-reference/swagger/).

{{% alert title="Important" color="warning" %}}
All monetary amounts are represented as **strings**, not numbers. This avoids IEEE 754 floating-point precision issues. For example, use `"100.00"` instead of `100.00`.
{{% /alert %}}

## Submit a Transaction

**POST /v1/transactions**

Submits a transaction for asynchronous processing. Returns immediately with a receipt and a `"pending"` status.

### Request

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

### Request Body

| Field             | Type   | Required | Description                                                                 |
|-------------------|--------|----------|-----------------------------------------------------------------------------|
| `account_id`      | string | Yes      | The account identifier                                                      |
| `amount`          | string | Yes      | Positive decimal amount as a string (e.g., `"250.00"`)                      |
| `type`            | enum   | Yes      | One of: `debit`, `credit`, `transfer`, `exchange`, `mint`, `burn`           |
| `signature`       | string | Yes      | ECDSA P-256 signature of the canonical payload (see [Authentication](/api-reference/authentication/)) |
| `idempotency_key` | string | No       | Client-generated unique key to prevent duplicate processing (recommended)   |

### Response (202 Accepted)

```json
{
  "txn_id": "txn_01HYX3KPVW9QDMZ6R8F5GT7N4E",
  "account_id": "alice@example.com",
  "status": "pending",
  "shard_id": "shard-07",
  "enqueued_at": "2026-04-03T14:22:01.456Z"
}
```

| Field         | Type   | Description                                         |
|---------------|--------|-----------------------------------------------------|
| `txn_id`      | string | Unique transaction identifier (ULID format)         |
| `account_id`  | string | The account that submitted the transaction          |
| `status`      | string | Always `"pending"` on initial submission            |
| `shard_id`    | string | The ledger shard assigned to process this transaction |
| `enqueued_at` | string | ISO 8601 timestamp when the transaction was enqueued |

### Idempotency

Including an `idempotency_key` is strongly recommended for all production integrations. If you submit a transaction with the same `idempotency_key` as a previous request, the engine returns the original receipt without re-processing. This makes retries safe across network failures.

### Error Response (400 Bad Request)

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "Request body failed validation.",
  "instance": "/v1/transactions",
  "errors": [
    {
      "field": "amount",
      "message": "Amount must be a positive decimal string (e.g., \"100.00\")."
    },
    {
      "field": "type",
      "message": "Invalid type. Must be one of: debit, credit, transfer, exchange, mint, burn."
    }
  ]
}
```

## Get Transaction Status

**GET /v1/transactions/{id}**

Retrieve the current state of a transaction by its ID.

```bash
curl https://api.example.com/v1/transactions/txn_01HYX3KPVW9QDMZ6R8F5GT7N4E \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

### Response (200 OK)

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
  "audit_hash": "sha256:a3f8c2d1e9b04567..."
}
```

### Error Response (404 Not Found)

```json
{
  "type": "https://api.example.com/errors/not-found",
  "title": "Not Found",
  "status": 404,
  "detail": "Transaction 'txn_invalid' does not exist.",
  "instance": "/v1/transactions/txn_invalid"
}
```

## Query Transactions

**GET /v1/transactions**

Search and filter transactions with pagination support.

```bash
curl "https://api.example.com/v1/transactions?account_id=alice@example.com&type=debit&date_from=2026-04-01T00:00:00Z&page_size=50" \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

### Query Parameters

| Parameter    | Type   | Default | Description                                           |
|--------------|--------|---------|-------------------------------------------------------|
| `account_id` | string | —       | Filter by account identifier                          |
| `date_from`  | string | —       | ISO 8601 start date (inclusive)                        |
| `date_to`    | string | —       | ISO 8601 end date (exclusive)                         |
| `type`       | enum   | —       | Filter by transaction type                            |
| `token_id`   | string | —       | Filter by token identifier                            |
| `token_class`| string | —       | Filter by token class (`fungible` or `nft`)           |
| `group_id`   | string | —       | Filter by settlement group ID                         |
| `amount_min` | string | —       | Minimum amount (inclusive), as a decimal string        |
| `amount_max` | string | —       | Maximum amount (inclusive), as a decimal string        |
| `status`     | string | —       | Filter by status (`pending`, `finalized`, `rejected`) |
| `page_token` | string | —       | Cursor for the next page of results                   |
| `page_size`  | int    | 100     | Number of results per page (max 1000)                 |

### Response (200 OK)

```json
{
  "transactions": [
    {
      "txn_id": "txn_01HYX3KPVW9QDMZ6R8F5GT7N4E",
      "account_id": "alice@example.com",
      "amount": "250.00",
      "type": "debit",
      "status": "finalized",
      "shard_id": "shard-07",
      "enqueued_at": "2026-04-03T14:22:01.456Z",
      "finalized_at": "2026-04-03T14:22:02.112Z"
    }
  ],
  "next_page_token": "eyJsYXN0X2lkIjoiMTIzNCJ9",
  "total_count": 147
}
```

To fetch the next page, pass the `next_page_token` value as the `page_token` query parameter. When `next_page_token` is `null` or absent, you have reached the last page.

## Query Transactions by Group

**GET /v1/transactions/group/{groupID}**

Returns all transactions that belong to a settlement group. The `groupID` must be in UUID format.

```bash
curl https://api.example.com/v1/transactions/group/grp_01HYX3MQNW8RDAZ5T7F4HS6K3D \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

### Response (200 OK)

```json
{
  "group_id": "grp_01HYX3MQNW8RDAZ5T7F4HS6K3D",
  "transactions": [
    {
      "txn_id": "txn_01HYX3KPVW9QDMZ6R8F5GT7N4E",
      "account_id": "alice@example.com",
      "amount": "250.00",
      "type": "debit",
      "status": "finalized"
    },
    {
      "txn_id": "txn_01HYX3LPXW0SEBZ7S9G6HU8M5F",
      "account_id": "bob@example.com",
      "amount": "250.00",
      "type": "credit",
      "status": "finalized"
    }
  ],
  "settled_at": "2026-04-03T14:22:02.200Z"
}
```
