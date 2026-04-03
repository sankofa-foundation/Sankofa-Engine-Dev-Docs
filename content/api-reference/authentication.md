---
title: "Authentication"
linkTitle: "Authentication"
weight: 1
description: >
  How to authenticate with the Sankofa Engine API using JWT tokens or ECDSA transaction signing.
---

The Sankofa Engine supports two authentication models. Most integrations use **JWT bearer tokens** for API access. Self-custody deployments additionally use **ECDSA transaction signing** to authorize individual transactions without sharing private keys.

For interactive exploration, see the [API Explorer](/api-reference/swagger/).

## JWT Authentication (Default)

Sankofa Labs provisions API credentials (`client_id` and `client_secret`) during onboarding. Your application exchanges these credentials for a short-lived JWT token, then includes that token on every request.

### 1. Obtain a Token

**POST /v1/auth/token**

Exchange your client credentials for a JWT access token.

```bash
curl -X POST https://api.example.com/v1/auth/token \
  -H "Content-Type: application/json" \
  -d '{
    "client_id": "your-client-id",
    "client_secret": "your-client-secret"
  }'
```

**Response (200 OK):**

```json
{
  "access_token": "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ5b3VyLWNsaWVudC1pZCIsImlhdCI6MTcxNTAwMDAwMCwiZXhwIjoxNzE1MDAzNjAwfQ.signature",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4..."
}
```

| Field           | Type   | Description                                    |
|-----------------|--------|------------------------------------------------|
| `access_token`  | string | JWT token to include in the Authorization header |
| `token_type`    | string | Always `"Bearer"`                              |
| `expires_in`    | int    | Token lifetime in seconds (default 3600 / 1 hour) |
| `refresh_token` | string | Long-lived token used to obtain a new access token |

### 2. Use the Token

Pass the JWT as a Bearer token in the `Authorization` header on all subsequent requests.

```bash
curl https://api.example.com/v1/accounts/alice@example.com/balances \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

### 3. Refresh the Token

When the access token expires, use the refresh token to obtain a new one without re-submitting your client secret.

```bash
curl -X POST https://api.example.com/v1/auth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "refresh_token",
    "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4..."
  }'
```

**Token lifecycle summary:**

| Property          | Value                       |
|-------------------|-----------------------------|
| Access token TTL  | 1 hour (3600 s)             |
| Refresh token TTL | 30 days                     |
| Algorithm         | ES256 (ECDSA P-256 + SHA-256) |
| Refresh behavior  | Returns new access token; refresh token rotates on each use |

## Transaction Signing (Self-Custody Model)

In the self-custody model, each account holds its own ECDSA P-256 private key. Instead of delegating authority via a JWT, the account signs each transaction payload directly. The engine verifies the signature against the public key registered during account setup.

### Key Registration

During account creation, the account's public key is registered with the engine. All subsequent transactions from that account must be signed with the corresponding private key.

### Signing Flow

1. Construct the transaction payload (JSON).
2. Canonicalize the JSON (sorted keys, no extra whitespace).
3. Compute the SHA-256 digest of the canonical payload.
4. Sign the digest with the account's ECDSA P-256 private key.
5. Base64url-encode the signature and include it in the `signature` field.

```python
import hashlib
import json
import base64
from cryptography.hazmat.primitives.asymmetric import ec, utils
from cryptography.hazmat.primitives import hashes

# 1. Build the payload (without the signature field)
payload = {
    "account_id": "alice@example.com",
    "amount": "100.00",
    "type": "debit",
    "idempotency_key": "txn-20260403-001"
}

# 2. Canonicalize
canonical = json.dumps(payload, sort_keys=True, separators=(",", ":"))

# 3-4. Sign
private_key = ec.generate_private_key(ec.SECP256R1())
signature = private_key.sign(
    canonical.encode(),
    ec.ECDSA(hashes.SHA256())
)

# 5. Encode and attach
payload["signature"] = base64.urlsafe_b64encode(signature).decode()
```

The signed payload is then submitted via `POST /v1/transactions`:

```bash
curl -X POST https://api.example.com/v1/transactions \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "alice@example.com",
    "amount": "100.00",
    "type": "debit",
    "idempotency_key": "txn-20260403-001",
    "signature": "MEUCIQDx...base64url-encoded..."
  }'
```

{{% alert title="Note" color="info" %}}
Even in the self-custody model, a valid JWT is still required for API access. The transaction signature provides an additional layer of authorization proving the account holder approved the specific transaction.
{{% /alert %}}

## Error Responses

Authentication errors follow [RFC 7807 Problem Details](https://datatracker.ietf.org/doc/html/rfc7807) format.

### 401 Unauthorized

Returned when the token is missing, malformed, or expired.

```json
{
  "type": "https://api.example.com/errors/unauthorized",
  "title": "Unauthorized",
  "status": 401,
  "detail": "Access token has expired. Please refresh your token or request a new one.",
  "instance": "/v1/accounts/alice@example.com/balances"
}
```

### 403 Forbidden

Returned when the token is valid but the caller lacks the required role or permission (RBAC).

```json
{
  "type": "https://api.example.com/errors/forbidden",
  "title": "Forbidden",
  "status": 403,
  "detail": "Role 'viewer' does not have permission to submit transactions. Required role: 'transactor' or 'admin'.",
  "instance": "/v1/transactions"
}
```

### Common Causes

| Status | Cause                              | Resolution                                    |
|--------|------------------------------------|-----------------------------------------------|
| 401    | Missing `Authorization` header     | Add `Authorization: Bearer <token>`           |
| 401    | Expired access token               | Use the refresh token to obtain a new one     |
| 401    | Malformed token                    | Verify the token string is complete           |
| 403    | Insufficient role                  | Contact your admin to adjust RBAC permissions |
| 403    | Invalid transaction signature      | Verify signing key matches registered public key |
