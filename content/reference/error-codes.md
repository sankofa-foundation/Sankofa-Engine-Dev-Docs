---
title: "Error Codes"
linkTitle: "Error Codes"
weight: 4
description: >
  Complete error code reference with RFC 7807 Problem Details format.
---

## Response Format

The Sankofa Engine returns errors using the **RFC 7807 Problem Details** format. Every error response is a JSON object with a consistent structure:

```json
{
  "type": "https://api.sankofa.engine/errors/invalid-request",
  "title": "Bad Request",
  "status": 400,
  "detail": "amount must be a valid positive decimal number",
  "error_codes": ["ERR_INVALID_AMOUNT"]
}
```

### Field Definitions

| Field | Type | Description |
|---|---|---|
| `type` | string (URI) | A URI that uniquely identifies the error category. Can be used as a stable reference in documentation and client error handling. |
| `title` | string | A short, human-readable summary of the error category (e.g., "Bad Request", "Not Found"). |
| `status` | integer | The HTTP status code for this response. Mirrors the HTTP response status. |
| `detail` | string | A human-readable explanation specific to this occurrence of the error. Useful for debugging. |
| `error_codes` | []string | One or more machine-readable error codes. A single request may trigger multiple validation errors. |

## Error Code Reference

### General

| Code | HTTP Status | Description |
|---|---|---|
| `ERR_NOT_FOUND` | 404 | Resource not found |
| `ERR_CONFLICT` | 409 | Resource already exists (e.g., duplicate NFT mint, duplicate token symbol) |
| `ERR_PAYLOAD_TOO_LARGE` | 413 | Request body exceeds the 1 MB limit |
| `ERR_UNSUPPORTED_CONTENT_TYPE` | 415 | Request must use `Content-Type: application/json` |
| `ERR_SHARD_UNAVAILABLE` | 503 | Shard ownership in flux — retry with backoff |
| `ERR_DOWNSTREAM_FAILURE` | 503 | An internal dependency is temporarily unavailable |

### Transaction Validation

| Code | HTTP Status | Description |
|---|---|---|
| `ERR_INVALID_ACCOUNT_ID` | 400 | Account ID is missing or invalid. Must be ASCII-only, max 128 characters, allowed: `a-zA-Z0-9_.-:` |
| `ERR_INVALID_AMOUNT` | 400 | Amount is required and must be a valid positive decimal (max 18 integer + 18 decimal digits) |
| `ERR_INVALID_TYPE` | 400 | Transaction type is required and must be a valid enum value (`debit`, `credit`, `transfer`, `exchange`, `mint`, `burn`) |
| `ERR_INVALID_SIGNATURE` | 400 | Signature is required |
| `ERR_INVALID_IDEMPOTENCY_KEY` | 400 | Idempotency key is required for all transaction submissions |
| `ERR_INVALID_GROUP_ID` | 400 | Group ID must be a valid UUID |
| `ERR_SELF_TRANSFER` | 400 | Cannot transfer to the same account |

### Token Validation

| Code | HTTP Status | Description |
|---|---|---|
| `ERR_INVALID_TOKEN_ID` | 400 | Token ID is required |
| `ERR_INVALID_SYMBOL` | 400 | Symbol is required |
| `ERR_INVALID_DECIMALS` | 400 | Decimal precision must be between 0 and 18 |
| `ERR_INVALID_ISSUER` | 400 | Issuer is required |
| `ERR_INVALID_TOTAL_SUPPLY` | 400 | Total supply must be non-negative |
| `ERR_INVALID_INITIAL_SUPPLY` | 400 | Initial supply must be non-negative |
| `ERR_INVALID_BALANCE` | 400 | Encrypted balance is required |

### NFT Validation

| Code | HTTP Status | Description |
|---|---|---|
| `ERR_INVALID_CLASS_ID` | 400 | NFT class ID is required |
| `ERR_INVALID_NAME` | 400 | Name is required |
| `ERR_INVALID_SCHEMA` | 400 | Metadata schema is required and must be valid JSON |

### Shard and System Validation

| Code | HTTP Status | Description |
|---|---|---|
| `ERR_INVALID_HEALTH_STATE` | 400 | Invalid shard health state |
| `ERR_INVALID_TXN_COUNT` | 400 | Transaction count must be non-negative |
| `ERR_NO_SHARDS` | 503 | No shards configured |
| `ERR_NO_HEALTHY_SHARDS` | 503 | No healthy shards available |

### Audit and Proof Validation

| Code | HTTP Status | Description |
|---|---|---|
| `ERR_INVALID_PROOF_HASH` | 400 | Proof hash is required |
| `ERR_INVALID_REQUESTER` | 400 | Requester field is required |
| `ERR_INVALID_PROOF_SYSTEM` | 400 | Invalid proof system |

### Timestamp Validation

| Code | HTTP Status | Description |
|---|---|---|
| `ERR_INVALID_CREATED_AT` | 400 | `created_at` timestamp is required |
| `ERR_INVALID_UPDATED_AT` | 400 | `updated_at` timestamp is required |
| `ERR_INVALID_TIMESTAMP` | 400 | Timestamp is required |

## Error Handling Best Practices

### Use `status` for Response Category

Check the HTTP status code to determine the general category of the error:

- **400** -- Client error. The request is malformed or contains invalid data. Fix the request and retry.
- **404** -- The requested resource does not exist. Verify the identifier.
- **409** -- Conflict. The resource already exists (e.g., duplicate NFT mint or token symbol). Do not retry.
- **413** -- Payload too large. The request body exceeds the 1 MB limit. Reduce the payload size.
- **415** -- Unsupported media type. The request must use `Content-Type: application/json`.
- **503** -- The service is temporarily unavailable (e.g., no healthy shards, shard ownership in flux). Retry with exponential backoff.

### Use `error_codes` for Programmatic Handling

The `error_codes` array contains machine-readable codes suitable for `switch`/`case` logic in client applications. A single response may contain multiple codes when several validation rules fail simultaneously:

```json
{
  "type": "https://api.sankofa.engine/errors/invalid-request",
  "title": "Bad Request",
  "status": 400,
  "detail": "multiple validation errors",
  "error_codes": ["ERR_INVALID_AMOUNT", "ERR_INVALID_SIGNATURE"]
}
```

### Use `detail` for Debugging

The `detail` field provides a human-readable description of what went wrong. Log this value for debugging but do not parse it programmatically -- the message text may change between releases. Rely on `error_codes` for stable programmatic checks.
