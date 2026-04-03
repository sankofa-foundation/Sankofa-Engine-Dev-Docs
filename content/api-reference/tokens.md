---
title: "Tokens & NFTs"
linkTitle: "Tokens"
weight: 5
description: >
  Fungible token registration and NFT class/instance management.
---

The Tokens API covers registration and querying of both fungible tokens and non-fungible tokens (NFTs). Fungible tokens represent interchangeable assets (currencies, loyalty points, carbon credits). NFTs represent unique items with individual provenance tracking.

For interactive exploration, see the [API Explorer](/api-reference/swagger/).

{{% alert title="Important" color="warning" %}}
All numeric values representing quantities or supplies are encoded as **strings** to avoid IEEE 754 floating-point precision issues. The `decimals` field (0-18) defines the token's precision.
{{% /alert %}}

---

## Fungible Tokens

### Register a Token

**POST /v1/fungible-tokens**

Register a new fungible token with the engine.

```bash
curl -X POST https://api.example.com/v1/fungible-tokens \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "symbol": "USD-COIN",
    "decimals": 2,
    "issuer": "treasury@example.com",
    "total_supply": "1000000.00"
  }'
```

#### Request Body

| Field          | Type   | Required | Description                                           |
|----------------|--------|----------|-------------------------------------------------------|
| `symbol`       | string | Yes      | Unique token symbol (e.g., `"USD-COIN"`)              |
| `decimals`     | int    | Yes      | Number of decimal places (0-18)                       |
| `issuer`       | string | Yes      | Account ID of the token issuer                        |
| `total_supply` | string | Yes      | Total supply as a decimal string                      |

#### Response (201 Created)

```json
{
  "token_id": "USD-COIN",
  "symbol": "USD-COIN",
  "decimals": 2,
  "issuer": "treasury@example.com",
  "total_supply": "1000000.00",
  "created_at": "2026-04-03T10:00:00.000Z"
}
```

#### Error Response (400 Bad Request)

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "Request body failed validation.",
  "instance": "/v1/fungible-tokens",
  "errors": [
    {
      "field": "decimals",
      "message": "Decimals must be an integer between 0 and 18."
    }
  ]
}
```

### List Tokens

**GET /v1/fungible-tokens**

Returns all registered fungible tokens.

```bash
curl https://api.example.com/v1/fungible-tokens \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

#### Response (200 OK)

```json
{
  "tokens": [
    {
      "token_id": "USD-COIN",
      "symbol": "USD-COIN",
      "decimals": 2,
      "issuer": "treasury@example.com",
      "total_supply": "1000000.00",
      "created_at": "2026-04-03T10:00:00.000Z"
    },
    {
      "token_id": "LOYALTY-PTS",
      "symbol": "LOYALTY-PTS",
      "decimals": 0,
      "issuer": "rewards@example.com",
      "total_supply": "500000",
      "created_at": "2026-03-28T08:30:00.000Z"
    }
  ]
}
```

### Get Token Details

**GET /v1/fungible-tokens/{id}**

```bash
curl https://api.example.com/v1/fungible-tokens/USD-COIN \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

#### Response (200 OK)

```json
{
  "token_id": "USD-COIN",
  "symbol": "USD-COIN",
  "decimals": 2,
  "issuer": "treasury@example.com",
  "total_supply": "1000000.00",
  "created_at": "2026-04-03T10:00:00.000Z"
}
```

---

## NFT Classes

NFTs in the Sankofa Engine are organized into **classes** (templates that define the schema and metadata structure) and **instances** (individual items minted from a class).

### Register an NFT Class

**POST /v1/nft-classes**

```bash
curl -X POST https://api.example.com/v1/nft-classes \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Carbon Credits 2026",
    "description": "Verified carbon offset credits for the 2026 vintage year.",
    "issuer": "registry@example.com",
    "metadata_schema": {
      "type": "object",
      "properties": {
        "vintage_year": { "type": "integer" },
        "registry": { "type": "string" },
        "tonnes_co2": { "type": "string" }
      },
      "required": ["vintage_year", "registry", "tonnes_co2"]
    }
  }'
```

#### Request Body

| Field             | Type   | Required | Description                                              |
|-------------------|--------|----------|----------------------------------------------------------|
| `name`            | string | Yes      | Human-readable class name                                |
| `description`     | string | Yes      | Description of the NFT class                             |
| `issuer`          | string | Yes      | Account ID of the class issuer                           |
| `metadata_schema` | object | Yes      | JSON Schema defining the structure of instance metadata  |

#### Response (201 Created)

```json
{
  "class_id": "carbon-credits-2026",
  "name": "Carbon Credits 2026",
  "description": "Verified carbon offset credits for the 2026 vintage year.",
  "issuer": "registry@example.com",
  "metadata_schema": {
    "type": "object",
    "properties": {
      "vintage_year": { "type": "integer" },
      "registry": { "type": "string" },
      "tonnes_co2": { "type": "string" }
    },
    "required": ["vintage_year", "registry", "tonnes_co2"]
  },
  "created_at": "2026-04-03T10:15:00.000Z"
}
```

### List NFT Classes

**GET /v1/nft-classes**

```bash
curl https://api.example.com/v1/nft-classes \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

#### Response (200 OK)

```json
{
  "classes": [
    {
      "class_id": "carbon-credits-2026",
      "name": "Carbon Credits 2026",
      "issuer": "registry@example.com",
      "created_at": "2026-04-03T10:15:00.000Z"
    },
    {
      "class_id": "fine-art-provenance",
      "name": "Fine Art Provenance",
      "issuer": "gallery@example.com",
      "created_at": "2026-03-20T14:00:00.000Z"
    }
  ]
}
```

### Get NFT Class

**GET /v1/nft-classes/{id}**

```bash
curl https://api.example.com/v1/nft-classes/carbon-credits-2026 \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

#### Response (200 OK)

```json
{
  "class_id": "carbon-credits-2026",
  "name": "Carbon Credits 2026",
  "description": "Verified carbon offset credits for the 2026 vintage year.",
  "issuer": "registry@example.com",
  "metadata_schema": {
    "type": "object",
    "properties": {
      "vintage_year": { "type": "integer" },
      "registry": { "type": "string" },
      "tonnes_co2": { "type": "string" }
    },
    "required": ["vintage_year", "registry", "tonnes_co2"]
  },
  "created_at": "2026-04-03T10:15:00.000Z"
}
```

---

## NFT Instances

### List Instances

**GET /v1/nft-classes/{classId}/instances**

```bash
curl https://api.example.com/v1/nft-classes/carbon-credits-2026/instances \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

#### Response (200 OK)

```json
{
  "class_id": "carbon-credits-2026",
  "instances": [
    {
      "instance_id": "cc-inst-00042",
      "owner": "alice@example.com",
      "metadata": {
        "vintage_year": 2026,
        "registry": "Gold Standard",
        "tonnes_co2": "15.5"
      },
      "minted_at": "2026-04-01T09:00:00.000Z"
    },
    {
      "instance_id": "cc-inst-00043",
      "owner": "bob@example.com",
      "metadata": {
        "vintage_year": 2026,
        "registry": "Gold Standard",
        "tonnes_co2": "22.0"
      },
      "minted_at": "2026-04-01T09:00:01.000Z"
    }
  ]
}
```

### Get Instance

**GET /v1/nft-classes/{classId}/instances/{instanceId}**

```bash
curl https://api.example.com/v1/nft-classes/carbon-credits-2026/instances/cc-inst-00042 \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

#### Response (200 OK)

```json
{
  "class_id": "carbon-credits-2026",
  "instance_id": "cc-inst-00042",
  "owner": "alice@example.com",
  "metadata": {
    "vintage_year": 2026,
    "registry": "Gold Standard",
    "tonnes_co2": "15.5"
  },
  "minted_at": "2026-04-01T09:00:00.000Z"
}
```

### Get Instance Ownership History

**GET /v1/nft-classes/{classId}/instances/{instanceId}/history**

Returns the full ownership history of an NFT instance, from minting to the current owner.

```bash
curl https://api.example.com/v1/nft-classes/carbon-credits-2026/instances/cc-inst-00042/history \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

#### Response (200 OK)

```json
{
  "class_id": "carbon-credits-2026",
  "instance_id": "cc-inst-00042",
  "history": [
    {
      "from": null,
      "to": "registry@example.com",
      "txn_id": "txn_01HYW1ABCDEF123456789",
      "type": "mint",
      "timestamp": "2026-04-01T09:00:00.000Z"
    },
    {
      "from": "registry@example.com",
      "to": "alice@example.com",
      "txn_id": "txn_01HYX2GHIJKL987654321",
      "type": "transfer",
      "timestamp": "2026-04-02T11:30:00.000Z"
    }
  ]
}
```

### Get Instance Provenance

**GET /v1/nft-classes/{classId}/instances/{instanceId}/provenance**

Returns the provenance chain for an NFT instance, tracing its origin through minting, attestations, and transformations.

```bash
curl "https://api.example.com/v1/nft-classes/carbon-credits-2026/instances/cc-inst-00042/provenance?max_depth=5" \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

#### Query Parameters

| Parameter   | Type | Default | Description                              |
|-------------|------|---------|------------------------------------------|
| `max_depth` | int  | 10      | Maximum depth of the provenance chain    |

#### Response (200 OK)

```json
{
  "class_id": "carbon-credits-2026",
  "instance_id": "cc-inst-00042",
  "provenance": [
    {
      "depth": 0,
      "event": "mint",
      "actor": "registry@example.com",
      "txn_id": "txn_01HYW1ABCDEF123456789",
      "timestamp": "2026-04-01T09:00:00.000Z",
      "attestation_id": "att_01HYW1MNOPQR111222333"
    },
    {
      "depth": 1,
      "event": "attestation",
      "actor": "auditor@example.com",
      "attestation_id": "att_01HYW1MNOPQR111222333",
      "timestamp": "2026-03-30T15:00:00.000Z",
      "details": "Third-party verification of carbon offset project"
    }
  ]
}
```

#### Error Response (404 Not Found)

```json
{
  "type": "https://api.example.com/errors/not-found",
  "title": "Not Found",
  "status": 404,
  "detail": "NFT instance 'cc-inst-99999' not found in class 'carbon-credits-2026'.",
  "instance": "/v1/nft-classes/carbon-credits-2026/instances/cc-inst-99999/provenance"
}
```
