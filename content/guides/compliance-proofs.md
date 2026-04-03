---
title: "Compliance Proofs"
linkTitle: "Compliance Proofs"
weight: 5
description: >
  Generate and verify zero-knowledge proofs for regulatory compliance.
---

This guide covers generating and verifying zero-knowledge proofs (ZKPs) with the Sankofa Engine's Compliance API. ZKPs enable privacy-preserving audits -- you can demonstrate regulatory compliance without disclosing sensitive ledger data.

## What Are Zero-Knowledge Proofs?

A zero-knowledge proof is a cryptographic protocol that lets one party (the prover) convince another party (the verifier) that a statement is true, without revealing any information beyond the truth of the statement itself. The verifier learns nothing about the underlying data -- only that the claim is valid.

In the context of the Sankofa Engine, ZKPs solve a fundamental tension between transparency and privacy. Regulators and auditors need assurance that an organization is solvent, that assets have legitimate provenance, or that transaction patterns comply with anti-money-laundering rules. But satisfying those requirements by exposing raw account balances, transaction histories, or customer data creates privacy and competitive risks.

With ZKPs, the Sankofa Engine can generate a cryptographic proof that, for example, total liabilities exceed a threshold -- without revealing any individual account balance. An auditor can verify this proof independently, using only the proof data and a verification key. They never see the underlying balances, and they do not need access to the ledger.

## Proof Types

The Sankofa Engine supports three types of compliance proofs:

| Proof Type | What It Proves | When to Use It | Who Typically Requests It |
|---|---|---|---|
| **Proof of Liabilities** | Total liabilities equal a committed value, without revealing individual balances | Solvency audits, reserve attestations | External auditors, regulators |
| **Proof of Provenance** | An asset's origin chain is valid, without revealing intermediate holders | Supply chain verification, asset authenticity | Compliance teams, trading partners |
| **Proof of Compliance** | A set of transactions or account states satisfy a regulatory rule | AML/KYC checks, transaction monitoring | Regulators, internal compliance |

## Generate a Proof of Liabilities

A proof of liabilities demonstrates that the engine's total liabilities match a committed value. This is commonly used for solvency audits where a regulator needs assurance that reserves back all outstanding obligations.

```bash
curl -X POST https://api.example.com/v1/assertions/proof-of-liabilities \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "requester": "auditor@regulator.gov"
  }'
```

**Response (200 OK):**

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

**What's happening:** The engine computes the total liabilities across all accounts, constructs a cryptographic commitment to that value, and generates a ZKP proving the commitment is correct. The `proof` field contains the hex-encoded proof bytes that can be independently verified. The `commitment` is the cryptographic commitment to the proven value.

{{% alert title="Note" color="info" %}}
Proofs have an expiration date (default 7 days). After expiration, the proof should be regenerated to reflect the current ledger state. Verifying an expired proof returns `valid: false` with a reason.
{{% /alert %}}

## Generate a Proof of Provenance

A proof of provenance demonstrates that an asset's origin chain is legitimate without revealing the intermediate holders. This is useful for supply chain verification and asset authenticity checks.

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

**Response (200 OK):**

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

**What's happening:** The engine traces the full provenance chain of the specified NFT instance -- from minting through all transfers and attestations -- and generates a ZKP proving the chain is valid. The verifier can confirm the asset was minted by a legitimate issuer and passed through a valid chain of custody, without learning who the intermediate holders were.

## Generate a Proof of Compliance

A proof of compliance demonstrates that a set of transactions or account states satisfy a specific regulatory rule. The `rule_id` identifies the compliance rule to check, and the `scope` defines the accounts and time range to evaluate.

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

**Response (200 OK):**

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

**What's happening:** The engine evaluates all transactions for the specified account within the given date range against the `aml-threshold-check` rule. It generates a ZKP proving the result (`compliant` or `non_compliant`) without revealing the individual transactions, amounts, or counterparties. The `result` field indicates the outcome; the `proof` field provides the cryptographic evidence.

## Verify a Proof

Any party that holds a proof can verify it independently using the verification endpoint. The verifier does not need access to the underlying ledger data.

```bash
curl -X POST https://api.example.com/v1/assertions/verify \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "proof_id": "prf_01HYX4ABCDEF123456789",
    "proof": "0x03a1b2c3d4e5f6...hex-encoded-proof..."
  }'
```

**Successful verification (200 OK):**

```json
{
  "proof_id": "prf_01HYX4ABCDEF123456789",
  "valid": true,
  "type": "proof-of-liabilities",
  "verified_at": "2026-04-03T15:15:00.000Z"
}
```

**Failed verification (200 OK):**

```json
{
  "proof_id": "prf_01HYX4ABCDEF123456789",
  "valid": false,
  "type": "proof-of-liabilities",
  "reason": "Proof has expired. Please request a new proof from the issuing party.",
  "verified_at": "2026-04-11T10:00:00.000Z"
}
```

{{% alert title="Note" color="info" %}}
The verification endpoint returns `200 OK` in both cases. Check the `valid` field to determine the result. A `valid: false` response includes a `reason` explaining the failure.
{{% /alert %}}

## Workflow: Sharing Proofs with Auditors

A typical compliance workflow involves three parties: the organization (proof requester), the Sankofa Engine (proof generator), and the auditor (proof verifier).

```text
  Organization              Sankofa Engine              Auditor
      │                          │                          │
      │  POST proof-of-liabilities                          │
      │─────────────────────────►│                          │
      │                          │  compute, generate ZKP   │
      │  proof_id + proof data   │                          │
      │◄─────────────────────────│                          │
      │                          │                          │
      │  share proof_id + proof ─────────────────────────── │
      │                          │                          │
      │                          │  POST /assertions/verify │
      │                          │◄─────────────────────────│
      │                          │  valid: true             │
      │                          │─────────────────────────►│
      │                          │                          │
```

1. The organization requests a proof from the engine.
2. The organization shares the `proof_id` and `proof` data with the auditor (via secure channel, email, or API).
3. The auditor submits the proof to the verification endpoint and receives an independent confirmation.

## Best Practices

### When to Generate Proofs

- **Quarterly:** Generate proof-of-liabilities on a quarterly cadence aligned with financial reporting periods. This provides a regular solvency attestation for regulators.
- **On-demand:** Generate proof-of-provenance and proof-of-compliance as needed -- for example, when a trading partner requests asset verification or when a regulator initiates an examination.
- **Event-driven:** Consider generating proofs automatically after significant events (large transactions, new asset issuances) to maintain a continuous compliance record.

### Retention of Proof Records

- Store proof records (including `proof_id`, `type`, `requester`, `generated_at`, and `expires_at`) in your organization's compliance management system.
- Retain proof data for at least the duration required by your regulatory framework (typically 5-7 years).
- The proof data itself is self-contained -- it does not depend on the Sankofa Engine being available for future verification, as long as the verification key is preserved.

### Sharing Proofs with Auditors

- Transmit proof data over secure channels (TLS, encrypted email, or secure file transfer).
- Include the `proof_id` for traceability. Both parties can reference the same proof by ID.
- Provide auditors with read-only API credentials so they can call the `/v1/assertions/verify` endpoint directly.
- Consider maintaining an audit log of who requested and verified each proof.

### Proof Expiration

- Proofs expire after 7 days by default. After expiration, generate a new proof to reflect the current ledger state.
- Expired proofs return `valid: false` with a reason of `"Proof has expired"` when verified. This is not an error -- it means the proof was valid at generation time but the data may have changed since then.
- For long-running audit engagements, generate fresh proofs at the start of each review session.
