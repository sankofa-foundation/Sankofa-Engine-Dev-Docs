---
title: "NFT Lifecycle"
linkTitle: "NFT Lifecycle"
weight: 4
description: >
  Register classes, mint instances, transfer ownership, and query provenance.
---

This guide walks through the complete lifecycle of a non-fungible token (NFT) in the Sankofa Engine: defining a class, minting instances, transferring ownership, querying provenance, and burning instances.

NFTs in the Sankofa Engine are organized into **classes** and **instances**. A class defines the schema and metadata structure (like a template), while instances are individual items minted from that class. Each instance has a unique owner and a full provenance chain tracked on the ledger.

## Step 1: Register an NFT Class

Define a new NFT class with a name, description, issuer, and a JSON Schema that governs instance metadata.

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
        "tonnes_co2": { "type": "string" },
        "project_id": { "type": "string" }
      },
      "required": ["vintage_year", "registry", "tonnes_co2"]
    }
  }'
```

**Response (201 Created):**

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
      "tonnes_co2": { "type": "string" },
      "project_id": { "type": "string" }
    },
    "required": ["vintage_year", "registry", "tonnes_co2"]
  },
  "created_at": "2026-04-03T10:15:00.000Z"
}
```

**What's happening:** The engine registers the class and generates a `class_id` derived from the name. The `metadata_schema` is a standard JSON Schema that all instances must conform to when minted. This ensures data consistency across all instances in the class.

{{% alert title="Tip" color="info" %}}
Design your `metadata_schema` carefully before minting instances. The schema is immutable once instances exist. Include all fields that downstream consumers (auditors, compliance, marketplace) will need.
{{% /alert %}}

## Step 2: Mint an NFT Instance

Minting an NFT is done by submitting a transaction with `type: "mint"`. The transaction framework handles minting the same way it handles any other transaction -- with idempotency, shard routing, and receipt signing.

```bash
curl -X POST https://api.example.com/v1/transactions \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "registry@example.com",
    "amount": "1",
    "type": "mint",
    "signature": "MEUCIQDx...base64url-encoded...",
    "idempotency_key": "mint-cc-00042",
    "token_class": "nft",
    "token_id": "carbon-credits-2026",
    "nft_metadata": {
      "instance_id": "cc-inst-00042",
      "metadata": {
        "vintage_year": 2026,
        "registry": "Gold Standard",
        "tonnes_co2": "15.5",
        "project_id": "GS-2026-WIND-042"
      }
    }
  }'
```

**Response (202 Accepted):**

```json
{
  "txn_id": "txn_01HYW1ABCDEF123456789",
  "account_id": "registry@example.com",
  "status": "pending",
  "shard_id": "shard-03",
  "enqueued_at": "2026-04-03T10:30:00.000Z"
}
```

**What's happening:** The mint transaction is enqueued and processed by the shard worker just like any other transaction. The worker validates the metadata against the class's `metadata_schema`, creates the NFT instance, assigns ownership to the `account_id` (the issuer in this case), and extends the audit hash chain. Once finalized, the instance exists on the ledger with full provenance tracking.

Poll for finalization:

```bash
curl https://api.example.com/v1/transactions/txn_01HYW1ABCDEF123456789 \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

**Response (200 OK, finalized):**

```json
{
  "txn_id": "txn_01HYW1ABCDEF123456789",
  "account_id": "registry@example.com",
  "amount": "1",
  "type": "mint",
  "status": "finalized",
  "shard_id": "shard-03",
  "token_id": "carbon-credits-2026",
  "token_class": "nft",
  "enqueued_at": "2026-04-03T10:30:00.000Z",
  "finalized_at": "2026-04-03T10:30:01.200Z",
  "audit_hash": "sha256:b4c7d2e8f1a03456..."
}
```

## Step 3: Query the Instance

Retrieve the minted instance and its metadata.

```bash
curl https://api.example.com/v1/nft-classes/carbon-credits-2026/instances/cc-inst-00042 \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

**Response (200 OK):**

```json
{
  "class_id": "carbon-credits-2026",
  "instance_id": "cc-inst-00042",
  "owner": "registry@example.com",
  "metadata": {
    "vintage_year": 2026,
    "registry": "Gold Standard",
    "tonnes_co2": "15.5",
    "project_id": "GS-2026-WIND-042"
  },
  "minted_at": "2026-04-03T10:30:01.200Z"
}
```

The `owner` field reflects the current owner of the instance. After minting, the owner is the issuer account. This will change when the instance is transferred.

## Step 4: Transfer Ownership

Transfer the NFT from one account to another by submitting a transfer transaction.

```bash
curl -X POST https://api.example.com/v1/transactions \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "registry@example.com",
    "amount": "1",
    "type": "transfer",
    "signature": "MEUCIQDy...base64url-encoded...",
    "idempotency_key": "transfer-cc-00042-to-alice",
    "token_class": "nft",
    "token_id": "carbon-credits-2026",
    "nft_metadata": {
      "instance_id": "cc-inst-00042",
      "to_account": "alice@example.com"
    }
  }'
```

**Response (202 Accepted):**

```json
{
  "txn_id": "txn_01HYX2GHIJKL987654321",
  "account_id": "registry@example.com",
  "status": "pending",
  "shard_id": "shard-03",
  "enqueued_at": "2026-04-03T11:00:00.000Z"
}
```

After finalization, query the instance to confirm the ownership change:

```bash
curl https://api.example.com/v1/nft-classes/carbon-credits-2026/instances/cc-inst-00042 \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

**Response (200 OK) -- owner updated:**

```json
{
  "class_id": "carbon-credits-2026",
  "instance_id": "cc-inst-00042",
  "owner": "alice@example.com",
  "metadata": {
    "vintage_year": 2026,
    "registry": "Gold Standard",
    "tonnes_co2": "15.5",
    "project_id": "GS-2026-WIND-042"
  },
  "minted_at": "2026-04-03T10:30:01.200Z"
}
```

The `owner` field now shows `alice@example.com`. The transfer is recorded in both the ownership history and the provenance chain.

## Step 5: Query the Provenance Chain

The provenance chain traces the full origin and custody history of an NFT instance, including minting, transfers, and attestations.

```bash
curl "https://api.example.com/v1/nft-classes/carbon-credits-2026/instances/cc-inst-00042/provenance?max_depth=10" \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

**Response (200 OK):**

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
      "timestamp": "2026-04-03T10:30:01.200Z",
      "attestation_id": null
    },
    {
      "depth": 1,
      "event": "attestation",
      "actor": "auditor@thirdparty.com",
      "attestation_id": "att_01HYX5STUVWX444555666",
      "timestamp": "2026-04-03T12:00:00.000Z",
      "details": "Carbon offset project verified against Gold Standard methodology. 15.5 tonnes CO2 confirmed."
    },
    {
      "depth": 2,
      "event": "transfer",
      "actor": "registry@example.com",
      "txn_id": "txn_01HYX2GHIJKL987654321",
      "timestamp": "2026-04-03T11:00:00.000Z",
      "details": "Transferred to alice@example.com"
    }
  ]
}
```

The provenance chain provides a complete, tamper-evident history of the asset. Each event links back to a transaction ID or attestation ID that can be independently verified. Use the `max_depth` query parameter to control how far back the chain is traced (default is 10).

You can also query the ownership history specifically:

```bash
curl https://api.example.com/v1/nft-classes/carbon-credits-2026/instances/cc-inst-00042/history \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

**Response (200 OK):**

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
      "timestamp": "2026-04-03T10:30:01.200Z"
    },
    {
      "from": "registry@example.com",
      "to": "alice@example.com",
      "txn_id": "txn_01HYX2GHIJKL987654321",
      "type": "transfer",
      "timestamp": "2026-04-03T11:00:00.000Z"
    }
  ]
}
```

## Step 6: Burn an Instance

Burning permanently removes an NFT instance from circulation. The instance record remains on the ledger for audit purposes, but its status changes to `"burned"` and it can no longer be transferred.

```bash
curl -X POST https://api.example.com/v1/transactions \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "alice@example.com",
    "amount": "1",
    "type": "burn",
    "signature": "MEUCIQDz...base64url-encoded...",
    "idempotency_key": "burn-cc-00042",
    "token_class": "nft",
    "token_id": "carbon-credits-2026",
    "nft_metadata": {
      "instance_id": "cc-inst-00042"
    }
  }'
```

**Response (202 Accepted):**

```json
{
  "txn_id": "txn_01HYX4RSTUVW555666777",
  "account_id": "alice@example.com",
  "status": "pending",
  "shard_id": "shard-07",
  "enqueued_at": "2026-04-03T15:00:00.000Z"
}
```

After finalization, query the instance to confirm the burn:

```bash
curl https://api.example.com/v1/nft-classes/carbon-credits-2026/instances/cc-inst-00042 \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

**Response (200 OK) -- status changed to burned:**

```json
{
  "class_id": "carbon-credits-2026",
  "instance_id": "cc-inst-00042",
  "owner": "alice@example.com",
  "status": "burned",
  "metadata": {
    "vintage_year": 2026,
    "registry": "Gold Standard",
    "tonnes_co2": "15.5",
    "project_id": "GS-2026-WIND-042"
  },
  "minted_at": "2026-04-03T10:30:01.200Z",
  "burned_at": "2026-04-03T15:00:01.100Z"
}
```

{{% alert title="Important" color="warning" %}}
Burning is irreversible. Once an instance is burned, it cannot be transferred or un-burned. Only the current owner can burn an instance. The burn event is recorded in the provenance chain.
{{% /alert %}}

## Lifecycle Summary

| Step | Operation | Endpoint | Result |
|------|-----------|----------|--------|
| 1 | Register class | `POST /v1/nft-classes` | Class created with metadata schema |
| 2 | Mint instance | `POST /v1/transactions` (type: mint) | Instance created, owned by issuer |
| 3 | Query instance | `GET /v1/nft-classes/{classId}/instances/{instanceId}` | Instance metadata and current owner |
| 4 | Transfer | `POST /v1/transactions` (type: transfer) | Ownership updated |
| 5 | Query provenance | `GET /v1/nft-classes/{classId}/instances/{instanceId}/provenance` | Full origin and custody chain |
| 6 | Burn | `POST /v1/transactions` (type: burn) | Instance permanently retired |
