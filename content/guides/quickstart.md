---
title: "Quickstart"
linkTitle: "Quickstart"
weight: 1
description: >
  Submit your first transaction in under 5 minutes.
---

This guide walks you through a complete first integration with the Sankofa Engine API. You will authenticate, deposit funds into an account, transfer between accounts, and query balances.

## Prerequisites

Before you begin, you need:

- **API credentials** from Sankofa Labs: a `client_id` and `client_secret` provisioned during onboarding.
- **curl** (or any HTTP client) installed on your machine.
- **Test accounts** set up in the Sankofa Engine with a registered fungible token (your onboarding contact will provide these).

{{% alert title="Sandbox Environment" color="info" %}}
During development, use the sandbox base URL `https://sandbox.api.example.com`. All examples in this guide use `https://api.example.com` — replace this with your environment's base URL.
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
  "access_token": "eyJhbGciOiJFUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**What's happening:** The engine validates your credentials and returns a short-lived JWT token (1 hour TTL) signed with ES256 (ECDSA P-256 + SHA-256).

Save the token for the remaining steps:

```bash
export TOKEN="eyJhbGciOiJFUzI1NiIs..."
```

## Step 2: Deposit Funds (Single-Leg Credit)

The simplest transaction is a **credit** — a cash deposit into an account. This is a single-leg operation: funds are credited to one account with no corresponding debit (the funds originate from outside the system, e.g., a bank wire).

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

**What's happening:** The API Gateway validates your request, routes it to the correct shard using `FNV-1a(account_id) % shard_count`, and publishes it to NATS JetStream. The `202 Accepted` means the transaction is enqueued — not yet completed.

{{% alert title="Idempotency Keys" color="info" %}}
Always include an `idempotency_key`. If a network failure causes you to retry the request, the engine returns the original receipt without re-processing the transaction.
{{% /alert %}}

## Step 3: Check Transaction Status

Poll the transaction to watch it progress from `pending` to `completed`.

```bash
export TXN_ID="txn_01HYX3KPVW9QDMZ6R8F5GT7N4E"

curl https://api.example.com/v1/transactions/$TXN_ID \
  -H "Authorization: Bearer $TOKEN"
```

**Response after completion (200 OK):**

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

**What's happening:** The Shard Worker picked up your transaction, validated it, updated the ledger, extended the SHA-256 audit hash chain, signed an ECDSA P-256 receipt, and persisted the block to ScyllaDB.

## Step 4: Transfer Between Accounts (Multi-Leg)

Real-world transactions often involve multiple accounts. A **transfer** from Alice to Bob requires two legs: a debit from Alice and a credit to Bob. These are linked by a `group_id` so the settlement service can track them as a single atomic operation.

**Debit leg (Alice):**

```bash
export GROUP_ID="550e8400-e29b-41d4-a716-446655440000"

curl -X POST https://api.example.com/v1/transactions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "alice@example.com",
    "amount": "250.00",
    "type": "debit",
    "token_id": "USD",
    "group_id": "'$GROUP_ID'",
    "signature": "MEUCIQDy...base64url-encoded...",
    "idempotency_key": "transfer-alice-bob-001-debit"
  }'
```

**Credit leg (Bob):**

```bash
curl -X POST https://api.example.com/v1/transactions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "bob@example.com",
    "amount": "250.00",
    "type": "credit",
    "token_id": "USD",
    "group_id": "'$GROUP_ID'",
    "signature": "MEUCIQDz...base64url-encoded...",
    "idempotency_key": "transfer-alice-bob-001-credit"
  }'
```

**What's happening:** Each leg is routed independently to its account's shard. The `group_id` links them together. The Settlement Service tracks both legs and marks the group as settled once both complete. If one leg fails, compensating transactions can be issued to reverse the other.

**Query the group to verify both legs settled:**

```bash
curl https://api.example.com/v1/transactions/group/$GROUP_ID \
  -H "Authorization: Bearer $TOKEN"
```

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

## Step 5: Withdraw Funds (Single-Leg Debit)

A **debit** without a corresponding credit represents a withdrawal — funds leaving the system (e.g., a bank wire out).

```bash
curl -X POST https://api.example.com/v1/transactions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "alice@example.com",
    "amount": "100.00",
    "type": "debit",
    "token_id": "USD",
    "signature": "MEUCIQDw...base64url-encoded...",
    "idempotency_key": "withdrawal-alice-001"
  }'
```

## Step 6: Check Account Balance

Verify the balance reflects all operations (deposit, transfer, withdrawal).

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
      "token_id": "USD",
      "balance": "650.00",
      "updated_at": "2026-04-03T14:22:05.789Z"
    }
  ]
}
```

Alice started with $0, received a $1,000 deposit (credit), sent $250 to Bob (debit), and withdrew $100 (debit) — resulting in a $650 balance.

{{% alert title="Note" color="info" %}}
Balances are served from the PostgreSQL projection layer (CQRS read model) and are eventually consistent with the ledger. In practice, projection lag is typically under 100 milliseconds. All amounts are **strings** to avoid IEEE 754 floating-point precision issues.
{{% /alert %}}

## Transaction Types Summary

| Type | Description | Use Case |
|------|-------------|----------|
| `credit` | Funds in — increases account balance | Cash deposits, incoming wires, payment receipts |
| `debit` | Funds out — decreases account balance | Withdrawals, outgoing wires, fee deductions |
| `transfer` | Move between accounts (used with `group_id`) | Internal account-to-account transfers |
| `exchange` | Asset swap between token types | Currency conversion, token exchange |
| `mint` | Create new token supply | Token issuance, initial distribution |
| `burn` | Destroy token supply | Token redemption, supply reduction |

{{% alert title="Multi-Leg Transactions" color="warning" %}}
When moving funds between accounts, always submit paired legs (debit + credit) with the same `group_id`. The Settlement Service tracks both legs and ensures atomicity. Single-leg debits/credits are valid for external cash movements (deposits and withdrawals).
{{% /alert %}}

## Next Steps

- **[Transaction Flow Guide]({{< relref "transaction-flow" >}})** — deep dive into the 7-stage transaction lifecycle, including failure handling and exactly-once semantics.
- **[Self-Custody Signing Guide]({{< relref "self-custody" >}})** — sign transactions with your own ECDSA private key instead of relying on custodial authorization.
- **[NFT Lifecycle Guide]({{< relref "nft-lifecycle" >}})** — register NFT classes, mint instances, transfer ownership, and query provenance.
- **[API Reference](/api-reference/)** — full endpoint documentation with request/response schemas.
