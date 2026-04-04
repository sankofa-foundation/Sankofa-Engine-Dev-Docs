---
title: "Accounts"
linkTitle: "Accounts"
weight: 4
description: >
  Query account state, balances, and owned NFTs.
---

The Accounts API provides read-only access to account state, token balances, and NFT ownership. These endpoints are served from a PostgreSQL projection layer, which means they are fast but eventually consistent with the underlying ledger.

### Account ID Format

Account identifiers must be ASCII-only, at most 128 characters, and may only contain: `a-zA-Z0-9_.-:`. Requests with invalid account IDs are rejected with a **400 Bad Request** error.

For interactive exploration, see the [API Explorer](/api-reference/swagger/).

## Get Account State

**GET /v1/accounts/{id}/state**

Returns the current state of an account, including its latest audit hash.

```bash
curl https://api.example.com/v1/accounts/alice@example.com/state \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

### Response (200 OK)

```json
{
  "account_id": "alice@example.com",
  "audit_hash": "sha256:a3f8c2d1e9b04567890abcdef1234567890abcdef1234567890abcdef12345678",
  "updated_at": "2026-04-03T14:22:02.112Z"
}
```

| Field        | Type   | Description                                                                 |
|--------------|--------|-----------------------------------------------------------------------------|
| `account_id` | string | The account identifier                                                      |
| `audit_hash` | string | The latest SHA-256 hash in the account's audit chain. Each transaction appends a new hash that covers the previous hash plus the transaction data, forming a tamper-evident chain. |
| `updated_at` | string | ISO 8601 timestamp of the last state change                                 |

### Error Response (404 Not Found)

Returned when the account does not exist on any shard.

```json
{
  "type": "https://api.example.com/errors/not-found",
  "title": "Not Found",
  "status": 404,
  "detail": "Account 'unknown@example.com' does not exist.",
  "instance": "/v1/accounts/unknown@example.com/state"
}
```

### Error Response (503 Service Unavailable)

Returned when the shard that owns this account is temporarily unavailable (e.g., during shard rebalancing or ownership transfer). Retry with exponential backoff.

```json
{
  "type": "https://api.sankofa.engine/errors/shard-unavailable",
  "title": "Service Unavailable",
  "status": 503,
  "detail": "shard not available after retry (ownership in flux)"
}
```

## Get All Balances

**GET /v1/accounts/{id}/balances**

Returns an array of token balances held by the account. Balances are served from the PostgreSQL projection and are eventually consistent with the ledger.

```bash
curl https://api.example.com/v1/accounts/alice@example.com/balances \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

### Response (200 OK)

```json
{
  "account_id": "alice@example.com",
  "balances": [
    {
      "token_id": "USD-COIN",
      "token_class": "fungible",
      "balance": "10450.75",
      "decimals": 2,
      "updated_at": "2026-04-03T14:22:02.112Z"
    },
    {
      "token_id": "LOYALTY-PTS",
      "token_class": "fungible",
      "balance": "8200",
      "decimals": 0,
      "updated_at": "2026-04-02T09:15:33.800Z"
    }
  ]
}
```

{{% alert title="Note" color="info" %}}
All balance values are returned as **strings** to avoid IEEE 754 floating-point precision issues. The `decimals` field indicates the number of decimal places for display formatting.
{{% /alert %}}

## Get Single Token Balance

**GET /v1/accounts/{id}/balances/{tokenId}**

Returns the balance for a specific token.

```bash
curl https://api.example.com/v1/accounts/alice@example.com/balances/USD-COIN \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

### Response (200 OK)

```json
{
  "account_id": "alice@example.com",
  "token_id": "USD-COIN",
  "token_class": "fungible",
  "balance": "10450.75",
  "decimals": 2,
  "updated_at": "2026-04-03T14:22:02.112Z"
}
```

### Error Response (404 Not Found)

```json
{
  "type": "https://api.example.com/errors/not-found",
  "title": "Not Found",
  "status": 404,
  "detail": "Token 'UNKNOWN-TOKEN' not found for account 'alice@example.com'.",
  "instance": "/v1/accounts/alice@example.com/balances/UNKNOWN-TOKEN"
}
```

## List Owned NFTs

**GET /v1/accounts/{id}/nfts**

Returns a list of NFT instances currently owned by the account.

```bash
curl https://api.example.com/v1/accounts/alice@example.com/nfts \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

### Response (200 OK)

```json
{
  "account_id": "alice@example.com",
  "nfts": [
    {
      "class_id": "carbon-credits-2026",
      "instance_id": "cc-inst-00042",
      "class_name": "Carbon Credits 2026",
      "metadata": {
        "vintage_year": 2026,
        "registry": "Gold Standard",
        "tonnes_co2": "15.5"
      },
      "acquired_at": "2026-03-15T10:30:00.000Z"
    },
    {
      "class_id": "fine-art-provenance",
      "instance_id": "fap-inst-00007",
      "class_name": "Fine Art Provenance",
      "metadata": {
        "title": "Sunset Over Lagos",
        "artist": "Adaeze Okafor",
        "year": 2024
      },
      "acquired_at": "2026-02-20T16:45:12.000Z"
    }
  ]
}
```

| Field         | Type   | Description                                    |
|---------------|--------|------------------------------------------------|
| `class_id`    | string | The NFT class this instance belongs to         |
| `instance_id` | string | The unique instance identifier                 |
| `class_name`  | string | Human-readable name of the NFT class           |
| `metadata`    | object | Instance-specific metadata (schema varies by class) |
| `acquired_at` | string | ISO 8601 timestamp when the account acquired this NFT |
