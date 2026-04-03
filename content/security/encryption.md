---
title: "Encryption"
linkTitle: "Encryption"
weight: 2
description: >
  Encryption at rest, in transit, and key management architecture.
---

The Sankofa Engine encrypts data both at rest and in transit. This page documents the cryptographic algorithms, key management architecture, and certificate lifecycle used to protect customer data.

## Encryption at Rest

All data at rest is protected using **AES-GCM-256 envelope encryption**. Envelope encryption separates the key that encrypts data (Data Encryption Key, or DEK) from the key that protects the DEK (the KMS master key), providing defense in depth and enabling key rotation without re-encrypting all data.

### Envelope Encryption Flow

```text
┌─────────────┐     ┌──────────────────┐     ┌────────────────────────────┐
│ Application  │     │   DEK (cached)   │     │      Ciphertext Output     │
│   Plaintext  │────▶│  AES-GCM-256     │────▶│  Encrypted Data            │
│              │     │  Encrypt         │     │  + KMS-Encrypted DEK       │
└─────────────┘     └──────────────────┘     │  + Nonce/IV                │
                           ▲                  └────────────────────────────┘
                           │
                    ┌──────┴───────┐
                    │     KMS      │
                    │  (OpenBao /  │
                    │   AWS KMS)   │
                    │              │
                    │  Derives &   │
                    │  wraps DEKs  │
                    └──────────────┘
```

### Key Derivation by Data Classification

The KMS derives separate DEKs for each `dataClass`, ensuring cryptographic isolation between data classifications:

| Data Class | Description | Key Isolation |
|------------|-------------|---------------|
| `financial` | Transaction amounts, balances, ledger entries | Dedicated DEK per class |
| `identity` | Account identifiers, customer references | Dedicated DEK per class |
| `operational` | System events, health metrics | Dedicated DEK per class |

This design ensures that compromise of one DEK does not expose data in other classifications.

### DEK Caching

To avoid a KMS round-trip on every encryption operation, DEKs are cached in memory after derivation:

- DEKs are cached for a configurable duration (default: aligned with key rotation schedule).
- Cache entries are evicted on rotation or service restart.
- Cached DEKs exist only in process memory and are never written to disk.

### Cryptographic Parameters

| Parameter | Value |
|-----------|-------|
| Algorithm | AES-GCM-256 |
| Key length | 256 bits |
| Nonce/IV | 96 bits, randomly generated per encryption operation |
| Authentication tag | 128 bits (GCM default) |
| Key derivation | KMS-managed (OpenBao transit backend or AWS KMS) |

## Encryption in Transit

### Internal Service Communication (mTLS)

All communication between internal services uses mutual TLS (mTLS):

- **Shard Workers ↔ NATS JetStream**: mTLS
- **API Gateway ↔ NATS JetStream**: mTLS
- **Shard Workers ↔ ScyllaDB**: mTLS
- **API Gateway ↔ PostgreSQL**: mTLS
- **All services ↔ OpenBao**: mTLS

Both the client and server present certificates and verify each other's identity before establishing a connection. This prevents man-in-the-middle attacks and ensures only authorized services communicate.

### Client-to-API Communication (TLS)

Client applications connect to the API Gateway over TLS 1.2 or higher. The API Gateway terminates TLS and authenticates the client via JWT or API key before forwarding requests to internal services over mTLS.

### Certificate Management

The `CertificateManager` interface provides automated certificate lifecycle management:

| Capability | Description |
|------------|-------------|
| File watching | Monitors certificate files on disk for changes |
| Auto-reload | Reloads certificates automatically when files are updated, without service restart |
| Graceful rotation | Active connections continue using the previous certificate until they close naturally |
| Health reporting | Exposes certificate expiration status via health endpoints |

This design supports zero-downtime certificate rotation, whether certificates are managed manually or by an automated certificate authority.

## Key Management

### Key Management Service Architecture

The Sankofa Engine supports two KMS backends:

| Backend | Use Case | Description |
|---------|----------|-------------|
| **OpenBao** (HashiCorp Vault fork) | Default / self-hosted | Transit secrets engine for key derivation, wrapping, and rotation |
| **AWS KMS** | Cloud production deployments | AWS-managed hardware security modules (HSMs) for master key protection |

### Key Hierarchy

```text
┌──────────────────────┐
│     Master Key       │  Managed by KMS (OpenBao or AWS KMS)
│  (never leaves KMS)  │  Never exported, never stored on disk
└──────────┬───────────┘
           │ derives / wraps
           ▼
┌──────────────────────┐
│  Data Encryption     │  One per dataClass
│  Keys (DEKs)         │  Cached in memory, wrapped copy stored with data
└──────────┬───────────┘
           │ encrypts
           ▼
┌──────────────────────┐
│       Data           │  Ciphertext stored in ScyllaDB, S3, etc.
└──────────────────────┘
```

- The **Master Key** never leaves the KMS boundary. It is used only to derive and wrap DEKs.
- **DEKs** are generated per data classification and used for bulk encryption. The encrypted (wrapped) DEK is stored alongside the ciphertext.
- **Data** is encrypted with the DEK using AES-GCM-256. Decryption requires unwrapping the DEK via the KMS.

### Key Rotation

Key rotation replaces the active DEK with a new one:

1. The KMS generates a new DEK version for the target data class.
2. New writes use the new DEK. The old DEK version remains available for decrypting existing data.
3. Background re-encryption can migrate existing ciphertext to the new DEK version on a configurable schedule.
4. Old DEK versions are retired after all data has been re-encrypted.

Rotation does not require downtime or service restart.

### Key Access Controls and Audit Logging

| Control | Description |
|---------|-------------|
| Policy-based access | OpenBao policies restrict which services can access which key paths |
| Per-service credentials | Each service authenticates to the KMS with its own identity |
| Audit logging | Every key operation (derive, wrap, unwrap, rotate) is logged by the KMS |
| No shared credentials | No two services share KMS authentication credentials |

### Cryptographic Erasure

Data can be rendered permanently unrecoverable by destroying the corresponding DEK in the KMS. Once the DEK is deleted, all data encrypted with that key becomes undecryptable — even if the ciphertext remains in storage. This supports data disposal compliance requirements without requiring physical media destruction.
