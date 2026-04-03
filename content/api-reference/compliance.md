---
title: "Compliance & Attestations"
linkTitle: "Compliance"
weight: 6
description: >
  Zero-knowledge proofs, compliance assertions, and asset attestations.
---

The Compliance API enables privacy-preserving audits and verifiable attestations. Using zero-knowledge proofs (ZKPs), the Sankofa Engine can generate cryptographic assertions that prove specific facts about the ledger -- such as solvency or asset provenance -- without revealing the underlying data. This allows regulated entities to demonstrate compliance to auditors and regulators while preserving the confidentiality of individual account balances and transaction details.

For interactive exploration, see the [API Explorer](/api-reference/swagger/).

---

## Zero-Knowledge Proof Assertions

The assertions API generates ZKP proofs that can be independently verified. Each proof is a cryptographic object that demonstrates a specific claim is true, without disclosing any of the private inputs used to construct it.

### Proof of Liabilities

**POST /v1/assertions/proof-of-liabilities**

Generates a zero-knowledge proof that the engine's total liabilities equal a committed value, without revealing any individual account balances.

```bash
curl -X POST https://api.example.com/v1/assertions/proof-of-liabilities \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "requester": "auditor@regulator.gov"
  }'
```

#### Request Body

| Field       | Type   | Required | Description                                    |
|-------------|--------|----------|------------------------------------------------|
| `requester` | string | Yes      | Identifier of the entity requesting the proof  |

#### Response (200 OK)

```json
{
  "proof_id": "prf_01HYX4ABCDEF123456789",
  "type": "proof-of-liabilities",
  "requester": "auditor@regulator.gov",
  "proof": "0x03a1b2c3d4e5f6...hex-encoded-proof...",
  "commitment": "0xf6e5d4c3b2a1...hex-encoded-commitment...",
  "generated_at": "2026-04-03T15:00:00.000Z",
  "expires_at": "2026-04-10T15:00:00.000Z"
}
```

| Field         | Type   | Description                                               |
|---------------|--------|-----------------------------------------------------------|
| `proof_id`    | string | Unique identifier for this proof                          |
| `type`        | string | The assertion type                                        |
| `requester`   | string | The entity that requested the proof                       |
| `proof`       | string | Hex-encoded ZKP proof bytes                               |
| `commitment`  | string | Hex-encoded cryptographic commitment to the proven value  |
| `generated_at`| string | ISO 8601 timestamp when the proof was generated           |
| `expires_at`  | string | ISO 8601 timestamp after which the proof should be regenerated |

### Proof of Provenance

**POST /v1/assertions/proof-of-provenance**

Generates a zero-knowledge proof that an asset's origin chain is valid, without revealing the intermediate holders.

```bash
curl -X POST https://api.example.com/v1/assertions/proof-of-provenance \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "requester": "compliance@partner.com",
    "asset_class": "carbon-credits-2026",
    "asset_instance": "cc-inst-00042"
  }'
```

#### Response (200 OK)

```json
{
  "proof_id": "prf_01HYX4GHIJKL987654321",
  "type": "proof-of-provenance",
  "requester": "compliance@partner.com",
  "asset_class": "carbon-credits-2026",
  "asset_instance": "cc-inst-00042",
  "proof": "0x07b8c9d0e1f2...hex-encoded-proof...",
  "generated_at": "2026-04-03T15:05:00.000Z",
  "expires_at": "2026-04-10T15:05:00.000Z"
}
```

### Proof of Compliance

**POST /v1/assertions/proof-of-compliance**

Generates a zero-knowledge proof that a set of transactions or account states satisfy a specified regulatory rule, without revealing the underlying data.

```bash
curl -X POST https://api.example.com/v1/assertions/proof-of-compliance \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "requester": "auditor@regulator.gov",
    "rule_id": "aml-threshold-check",
    "scope": {
      "account_id": "alice@example.com",
      "date_from": "2026-01-01T00:00:00Z",
      "date_to": "2026-04-01T00:00:00Z"
    }
  }'
```

#### Response (200 OK)

```json
{
  "proof_id": "prf_01HYX4MNOPQR111222333",
  "type": "proof-of-compliance",
  "requester": "auditor@regulator.gov",
  "rule_id": "aml-threshold-check",
  "result": "compliant",
  "proof": "0x0ab3c4d5e6f7...hex-encoded-proof...",
  "generated_at": "2026-04-03T15:10:00.000Z",
  "expires_at": "2026-04-10T15:10:00.000Z"
}
```

### Verify a Proof

**POST /v1/assertions/verify**

Verifies a previously generated zero-knowledge proof. This endpoint can be called by any party that holds the proof, without needing to know the original private inputs.

```bash
curl -X POST https://api.example.com/v1/assertions/verify \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "proof_id": "prf_01HYX4ABCDEF123456789",
    "proof": "0x03a1b2c3d4e5f6...hex-encoded-proof..."
  }'
```

#### Request Body

| Field      | Type   | Required | Description                                         |
|------------|--------|----------|-----------------------------------------------------|
| `proof_id` | string | Yes      | The proof identifier returned by the assertion endpoint |
| `proof`    | string | Yes      | The hex-encoded proof bytes to verify               |

#### Response (200 OK)

```json
{
  "proof_id": "prf_01HYX4ABCDEF123456789",
  "valid": true,
  "type": "proof-of-liabilities",
  "verified_at": "2026-04-03T15:15:00.000Z"
}
```

#### Verification Failure Response (200 OK)

When a proof is invalid or expired, the endpoint still returns 200 but with `valid: false`:

```json
{
  "proof_id": "prf_01HYX4ABCDEF123456789",
  "valid": false,
  "type": "proof-of-liabilities",
  "reason": "Proof has expired. Please request a new proof from the issuing party.",
  "verified_at": "2026-04-11T10:00:00.000Z"
}
```

#### Error Response (400 Bad Request)

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "Request body failed validation.",
  "instance": "/v1/assertions/verify",
  "errors": [
    {
      "field": "proof",
      "message": "Proof must be a hex-encoded string."
    }
  ]
}
```

---

## Attestations

Attestations are signed declarations about an asset's properties, origin, or condition. Unlike ZKP proofs, attestations are human-readable records that can be attached to NFT instances or linked to audit trails.

### Submit an Attestation

**POST /v1/attestations**

```bash
curl -X POST https://api.example.com/v1/attestations \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "attester": "auditor@thirdparty.com",
    "subject_class": "carbon-credits-2026",
    "subject_instance": "cc-inst-00042",
    "attestation_type": "verification",
    "statement": "Carbon offset project verified against Gold Standard methodology. 15.5 tonnes CO2 confirmed.",
    "evidence_url": "https://registry.example.com/reports/2026/042.pdf"
  }'
```

#### Response (201 Created)

```json
{
  "attestation_id": "att_01HYX5STUVWX444555666",
  "attester": "auditor@thirdparty.com",
  "subject_class": "carbon-credits-2026",
  "subject_instance": "cc-inst-00042",
  "attestation_type": "verification",
  "statement": "Carbon offset project verified against Gold Standard methodology. 15.5 tonnes CO2 confirmed.",
  "evidence_url": "https://registry.example.com/reports/2026/042.pdf",
  "created_at": "2026-04-03T16:00:00.000Z"
}
```

### Get Attestation

**GET /v1/attestations/{id}**

```bash
curl https://api.example.com/v1/attestations/att_01HYX5STUVWX444555666 \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

#### Response (200 OK)

```json
{
  "attestation_id": "att_01HYX5STUVWX444555666",
  "attester": "auditor@thirdparty.com",
  "subject_class": "carbon-credits-2026",
  "subject_instance": "cc-inst-00042",
  "attestation_type": "verification",
  "statement": "Carbon offset project verified against Gold Standard methodology. 15.5 tonnes CO2 confirmed.",
  "evidence_url": "https://registry.example.com/reports/2026/042.pdf",
  "created_at": "2026-04-03T16:00:00.000Z"
}
```

### List Attestations

**GET /v1/attestations**

```bash
curl "https://api.example.com/v1/attestations?subject_class=carbon-credits-2026&subject_instance=cc-inst-00042" \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..."
```

#### Response (200 OK)

```json
{
  "attestations": [
    {
      "attestation_id": "att_01HYX5STUVWX444555666",
      "attester": "auditor@thirdparty.com",
      "subject_class": "carbon-credits-2026",
      "subject_instance": "cc-inst-00042",
      "attestation_type": "verification",
      "statement": "Carbon offset project verified against Gold Standard methodology. 15.5 tonnes CO2 confirmed.",
      "created_at": "2026-04-03T16:00:00.000Z"
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
  "detail": "Attestation 'att_invalid' does not exist.",
  "instance": "/v1/attestations/att_invalid"
}
```
