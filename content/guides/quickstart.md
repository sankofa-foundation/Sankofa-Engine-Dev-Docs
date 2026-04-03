---
title: "Quickstart"
linkTitle: "Quickstart"
weight: 1
description: >
  Submit your first transaction in under 5 minutes.
---

This guide walks you through a complete first integration with the Sankofa Engine API. By the end, you will have authenticated, submitted a transaction, checked its status, and queried an account balance.

## Prerequisites

Before you begin, you need:

- **API credentials** from Sankofa Labs: a `client_id` and `client_secret` provisioned during onboarding.
- **curl** (or any HTTP client) installed on your machine.
- A **test account** set up in the Sankofa Engine (your onboarding contact will provide the account ID).

{{% alert title="Sandbox Environment" color="info" %}}
During development, use the sandbox base URL `https://sandbox.api.example.com`. All examples in this guide use `https://api.example.com` -- replace this with your environment's base URL.
{{% /alert %}}

## Step 1: Authenticate

Exchange your client credentials for a JWT access token.

```bash
curl -X POST https://api.example.com/v1/auth/token \
  -H "Content-Type: application/json" \
  -d '{
    "client_id": "your-client-id",
    "client_secret": "your-client-secret"
  }'
```

**Expected response (200 OK):**

```json
{
  "access_token": "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ5b3VyLWNsaWVudC1pZCIsImlhdCI6MTcxNTAwMDAwMCwiZXhwIjoxNzE1MDAzNjAwfQ.signature",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4..."
}
```

**What's happening:** The engine validates your credentials and returns a short-lived JWT token (1 hour TTL) signed with ES256 (ECDSA P-256 + SHA-256). You will include this token in the `Authorization` header on every subsequent request.

Save the `access_token` value for the remaining steps:

```bash
export TOKEN="eyJhbGciOiJFUzI1NiIs..."
```

## Step 2: Submit a Transaction

Submit a debit transaction against a test account.

```bash
curl -X POST https://api.example.com/v1/transactions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "alice@example.com",
    "amount": "250.00",
    "type": "debit",
    "signature": "MEUCIQDx...base64url-encoded...",
    "idempotency_key": "quickstart-txn-001"
  }'
```

**Expected response (202 Accepted):**

```json
{
  "txn_id": "txn_01HYX3KPVW9QDMZ6R8F5GT7N4E",
  "account_id": "alice@example.com",
  "status": "pending",
  "shard_id": 7,
  "enqueued_at": "2026-04-03T14:22:01.456Z"
}
```

**What's happening:** The API Gateway validates your request, computes a deterministic shard assignment using `FNV-1a(account_id) % shard_count`, and publishes the transaction to NATS JetStream. The `202 Accepted` status means the transaction has been enqueued for processing -- it has not been completed yet. The `txn_id` is your handle for tracking it.

Save the `txn_id` for the next step:

```bash
export TXN_ID="txn_01HYX3KPVW9QDMZ6R8F5GT7N4E"
```

{{% alert title="Idempotency Keys" color="info" %}}
Always include an `idempotency_key`. If a network failure causes you to retry the request, the engine will return the original receipt without re-processing the transaction.
{{% /alert %}}

## Step 3: Check Transaction Status

Poll the transaction to watch it progress from `pending` to `completed`.

```bash
curl https://api.example.com/v1/transactions/$TXN_ID \
  -H "Authorization: Bearer $TOKEN"
```

**Response while processing (200 OK):**

```json
{
  "txn_id": "txn_01HYX3KPVW9QDMZ6R8F5GT7N4E",
  "account_id": "alice@example.com",
  "amount": "250.00",
  "type": "debit",
  "status": "pending",
  "shard_id": 7,
  "enqueued_at": "2026-04-03T14:22:01.456Z"
}
```

Wait a moment and poll again. **Response after completion (200 OK):**

```json
{
  "txn_id": "txn_01HYX3KPVW9QDMZ6R8F5GT7N4E",
  "account_id": "alice@example.com",
  "amount": "250.00",
  "type": "debit",
  "status": "completed",
  "shard_id": 7,
  "group_id": "grp_01HYX3MQNW8RDAZ5T7F4HS6K3D",
  "token_id": "USD-COIN",
  "token_class": "fungible",
  "created_at": "2026-04-03T14:22:01.456Z",
  "audit_hash": "sha256:a3f8c2d1e9b04567890abcdef1234567890abcdef1234567890abcdef12345678"
}
```

**What's happening:** The Shard Worker picked up your transaction from the NATS queue, validated the balance, updated the in-memory ledger, extended the SHA-256 audit hash chain, signed an ECDSA receipt, and persisted the block to ScyllaDB. The `audit_hash` is the latest link in the account's tamper-evident hash chain. The `created_at` timestamp records when the transaction was created.

## Step 4: Check Account Balance

Verify the balance reflects the debit.

```bash
curl https://api.example.com/v1/accounts/alice@example.com/balances \
  -H "Authorization: Bearer $TOKEN"
```

**Expected response (200 OK):**

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

**What's happening:** This balance is served from the PostgreSQL projection layer (a CQRS read model). The Projection Service subscribes to finalized receipts and updates the read model, so balances are eventually consistent with the ledger. In practice, projection lag is typically under 100 milliseconds.

{{% alert title="Note" color="info" %}}
All balance values are returned as **strings** to avoid IEEE 754 floating-point precision issues. Use the `decimals` field to determine display formatting.
{{% /alert %}}

## Next Steps

You have completed a full transaction lifecycle. Here is where to go next:

- **[Transaction Flow Guide]({{< relref "transaction-flow" >}})** -- deep dive into the 7-stage transaction lifecycle, including failure handling and exactly-once semantics.
- **[Self-Custody Signing Guide]({{< relref "self-custody" >}})** -- sign transactions with your own ECDSA private key instead of relying on custodial authorization.
- **[NFT Lifecycle Guide]({{< relref "nft-lifecycle" >}})** -- register NFT classes, mint instances, transfer ownership, and query provenance.
- **[Compliance Proofs Guide]({{< relref "compliance-proofs" >}})** -- generate and verify zero-knowledge proofs for regulatory compliance.
- **[API Reference](/api-reference/)** -- full endpoint documentation with request/response schemas.
