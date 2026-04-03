---
title: "Security Overview"
linkTitle: "Security"
weight: 4
description: >
  Security model overview and links to detailed security documentation.
---

The Sankofa Engine implements defense-in-depth security across authentication, authorization, encryption, and audit logging. This page provides a high-level overview — for detailed documentation, see the [Security & Compliance](/security/) section.

## Security Architecture Summary

| Layer | Implementation |
|-------|---------------|
| **Authentication** | JWT tokens via API key exchange, ECDSA P-256 transaction signing for self-custody |
| **Authorization** | Casbin v2 RBAC with policy-based access control |
| **Encryption at Rest** | AES-GCM-256 envelope encryption with KMS-derived keys |
| **Encryption in Transit** | mTLS between services, TLS for client connections |
| **Audit Trail** | SHA-256 hash chains per account, ECDSA P-256 signed receipts |
| **Key Management** | OpenBao (Vault fork) transit backend, AWS KMS support |
| **Infrastructure** | Kubernetes namespace isolation, network policies, secret scoping |

## Detailed Documentation

- [Security Controls](/security/controls/) — SOC 2 Trust Service Criteria mapping
- [Encryption](/security/encryption/) — Encryption at rest, in transit, and key management
- [Access Control](/security/access-control/) — Authentication, authorization, and infrastructure access
- [Audit Logging](/security/audit-logging/) — Hash chains, signed receipts, and event retention
- [Data Residency](/security/data-residency/) — Storage tiers, retention policies, and archival
