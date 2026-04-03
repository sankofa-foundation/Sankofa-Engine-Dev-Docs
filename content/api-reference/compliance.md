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
  "assertion_type": "proof-of-liabilities",
  "requester": "auditor@regulator.gov",
  "result": {
    "total_liabilities": "25000000.00",
    "merkle_root": "sha256:4a7d1ed414474e4033ac29ccb8653d9b",
    "generated_at": "2026-04-03T15:00:00.000Z"
  },
  "signature": "MEUCIBkL...base64..."
}
```

| Field            | Type   | Description                                               |
|------------------|--------|-----------------------------------------------------------|
| `assertion_type` | string | The type of assertion generated                           |
| `requester`      | string | Account ID of the entity that requested the proof         |
| `result`         | object | Assertion-specific result payload                         |
| `signature`      | string | Base64-encoded ECDSA P-256 DER signature over the result  |

### Proof of Provenance

**POST /v1/assertions/proof-of-provenance**

Generates a zero-knowledge proof that an asset's origin chain is valid, without revealing the intermediate holders.

```bash
curl -X POST https://api.example.com/v1/assertions/proof-of-provenance \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "requester": "compliance@partner.com"
  }'
```

#### Response (200 OK)

```json
{
  "assertion_type": "proof-of-provenance",
  "requester": "compliance@partner.com",
  "result": {
    "assets_verified": 142,
    "provenance_valid": true,
    "generated_at": "2026-04-03T15:05:00.000Z"
  },
  "signature": "MEQCIF7x...base64..."
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
    "requester": "auditor@regulator.gov"
  }'
```

#### Response (200 OK)

```json
{
  "assertion_type": "proof-of-compliance",
  "requester": "auditor@regulator.gov",
  "result": {
    "compliant": true,
    "frameworks": ["AML/KYC", "SOC2"],
    "generated_at": "2026-04-03T15:10:00.000Z"
  },
  "signature": "MEYCIQCv...base64..."
}
```

### Verify an Assertion

**POST /v1/assertions/verify**

Verifies a previously generated assertion. This endpoint can be called by any party that holds the assertion result and signature, without needing to know the original private inputs.

```bash
curl -X POST https://api.example.com/v1/assertions/verify \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "assertion_type": "proof-of-liabilities",
    "result": {
      "total_liabilities": "25000000.00",
      "merkle_root": "sha256:4a7d1ed414474e4033ac29ccb8653d9b",
      "generated_at": "2026-04-03T15:00:00.000Z"
    },
    "signature": "MEUCIBkL...base64..."
  }'
```

#### Request Body

| Field            | Type   | Required | Description                                              |
|------------------|--------|----------|----------------------------------------------------------|
| `assertion_type` | string | Yes      | The type of assertion to verify                          |
| `result`         | object | Yes      | The assertion result payload to verify                   |
| `signature`      | string | Yes      | The signature to verify against the result               |

#### Response (200 OK)

```json
{
  "valid": true,
  "verified_at": "2026-04-03T15:15:00.000Z"
}
```

#### Verification Failure Response (200 OK)

When an assertion is invalid, the endpoint still returns 200 but with `valid: false`:

```json
{
  "valid": false,
  "reason": "Signature verification failed.",
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
