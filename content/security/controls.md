---
title: "Security Controls"
linkTitle: "Controls"
weight: 1
description: >
  Security controls inventory mapped to SOC 2 Trust Service Criteria.
---

This page inventories the security controls implemented in the Sankofa Engine, organized by SOC 2 Trust Service Categories. Each control includes an ID, description, implementation details, and the evidence an auditor would examine to verify the control is operating effectively.

## CC6 — Logical and Physical Access Controls

### CC6.1 — User Authentication

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC6.1 |
| **Description** | The system authenticates users and services before granting access to protected resources. |
| **Implementation** | JWT-based authentication with configurable token lifetime. API keys are provisioned by Sankofa Labs during customer onboarding. Clients exchange `client_id` and `client_secret` for a JWT bearer token via the `/auth/token` endpoint. Token lifetimes are configurable per deployment. |
| **Evidence** | JWT validation middleware source code; token lifetime configuration values; API key provisioning records maintained by Sankofa Labs; authentication failure logs showing rejected requests. |

### CC6.2 — Encryption of Data in Transit

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC6.2 |
| **Description** | The system protects data in transit using encryption between all components. |
| **Implementation** | Mutual TLS (mTLS) is enforced between all internal services (API Gateway ↔ NATS JetStream, Shard Workers ↔ NATS JetStream, API Gateway ↔ ScyllaDB, etc.). TLS 1.2+ is required for client-to-API Gateway communication. Certificate management uses a `CertificateManager` interface that supports file watching and automatic reload on rotation. |
| **Evidence** | TLS configuration in service manifests; mTLS certificate chain; `CertificateManager` source code showing file-watch and auto-reload logic; network policy definitions requiring encrypted transport; TLS handshake logs. |

### CC6.3 — Role-Based Authorization

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC6.3 |
| **Description** | The system restricts access to resources based on assigned roles and permissions. |
| **Implementation** | Casbin v2 RBAC engine with policy-based authorization. Policies define subject (role or user), object (resource), and action (operation). Authorization is enforced at the API Gateway middleware layer before requests reach backend services. Role definitions are maintained in policy files and loaded at startup. |
| **Evidence** | Casbin policy files defining roles and permissions; middleware source code enforcing authorization checks; authorization denial logs; role assignment records. |

### CC6.4 — Infrastructure Access Restrictions

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC6.4 |
| **Description** | Infrastructure-level access is restricted to authorized services and personnel. |
| **Implementation** | Kubernetes RBAC controls access to cluster resources. All Sankofa Engine components run in an isolated `sankofa-engine` namespace. Kubernetes NetworkPolicies restrict pod-to-pod communication to only the required paths (e.g., shard workers may communicate with NATS and ScyllaDB but not directly with the API Gateway). |
| **Evidence** | Kubernetes RBAC role and role-binding manifests; namespace configuration; NetworkPolicy manifests; `kubectl` output showing effective policies; cluster audit logs. |

### CC6.5 — API Key Lifecycle Management

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC6.5 |
| **Description** | API keys are managed through a defined lifecycle including provisioning, rotation, and revocation. |
| **Implementation** | API keys are provisioned by Sankofa Labs during customer onboarding. Key rotation is supported without downtime — new keys can be issued while old keys remain valid during a configurable grace period. Revocation is immediate upon request. All key lifecycle events are logged. |
| **Evidence** | API key provisioning procedures; key rotation records; revocation logs; key lifecycle event audit trail maintained by Sankofa Labs. |

## CC7 — System Operations

### CC7.1 — Health Monitoring and Availability

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC7.1 |
| **Description** | The system is monitored for availability and automatically scales to meet demand. |
| **Implementation** | Kubernetes liveness and readiness probes on port 9090 for all services. Horizontal Pod Autoscaler (HPA) automatically scales shard workers based on CPU and memory utilization. Health endpoints expose service status, connection state, and dependency health. |
| **Evidence** | Kubernetes deployment manifests showing probe configuration; HPA configuration and scaling event logs; health endpoint responses; uptime monitoring records. |

### CC7.2 — Event Durability and Retention

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC7.2 |
| **Description** | System events are durably stored and retained for the required period. |
| **Implementation** | NATS JetStream provides durable subscriptions for all transaction events. Messages are retained for 7 years by default (`max_message_age_seconds: 220898160`). Full event replay is available from any point in the retention window. Events are append-only and immutable once published. |
| **Evidence** | NATS JetStream stream configuration showing retention settings; durable subscription configuration; event replay demonstration; storage utilization metrics. |

### CC7.3 — Monitoring and Alerting

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC7.3 |
| **Description** | The system provides monitoring data and alerting capabilities for operational awareness. |
| **Implementation** | Health endpoints on each service expose operational metrics including connection status, queue depths, and storage utilization. Storage metrics track disk usage, shard distribution, and replication status. Alerting thresholds are configurable per deployment. |
| **Evidence** | Health endpoint response samples; monitoring dashboard configuration; alert rule definitions; historical alert records. |

### CC7.4 — Incident Response

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC7.4 |
| **Description** | Security and operational incidents are identified, reported, and resolved through defined procedures. |
| **Implementation** | Incident response is managed by Sankofa Labs. Defined escalation paths cover severity levels from informational to critical. Customers are notified of security incidents affecting their deployment per contractual SLAs. Post-incident reviews produce root cause analysis and remediation actions. |
| **Evidence** | Incident response plan documentation; incident ticket history; post-incident review reports; customer notification records. |

## CC8 — Change Management

### CC8.1 — Controlled Deployments

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC8.1 |
| **Description** | All changes to the production system are managed through a controlled deployment process. |
| **Implementation** | All deployments are managed exclusively by Sankofa Labs. Customers do not have the ability to deploy code or configuration changes to the engine. Changes follow a defined release process with staging validation before production deployment. |
| **Evidence** | Deployment records and change logs; release approval records; staging validation results; access control evidence showing customers cannot initiate deployments. |

### CC8.2 — Version Management

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC8.2 |
| **Description** | Software versions are tracked and documented with clear change history. |
| **Implementation** | The Sankofa Engine follows semantic versioning (SemVer). Every release includes a changelog documenting new features, bug fixes, and breaking changes. Release notes are published to customers before upgrades are applied. |
| **Evidence** | Version history and changelog; release notes; semantic version tags in source control; customer communication records for version upgrades. |

### CC8.3 — Automated Testing in CI/CD

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC8.3 |
| **Description** | Changes are validated through automated testing before deployment. |
| **Implementation** | CI/CD pipeline executes unit tests, benchmark tests, and end-to-end (e2e) tests on every change. Pipeline must pass all test stages before a release artifact is produced. Test coverage is tracked and regressions block deployment. |
| **Evidence** | CI/CD pipeline configuration; test execution logs and results; test coverage reports; pipeline failure records showing blocked deployments. |

## CC9 — Risk Mitigation

### CC9.1 — Network Segmentation

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC9.1 |
| **Description** | Network communication between services is restricted to only what is required. |
| **Implementation** | Kubernetes NetworkPolicies define explicit ingress and egress rules for each service. Only required communication paths are permitted (e.g., API Gateway → NATS, Shard Workers → NATS, Shard Workers → ScyllaDB). All other inter-pod traffic is denied by default. |
| **Evidence** | NetworkPolicy manifests; `kubectl describe networkpolicy` output; network traffic logs showing denied connections; architecture diagram of permitted communication paths. |

### CC9.2 — Secret Management

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC9.2 |
| **Description** | Secrets are managed securely with no plaintext storage. |
| **Implementation** | OpenBao (a fork of HashiCorp Vault) manages all secrets including database credentials, API keys, encryption keys, and TLS certificates. Secrets are injected into pods at runtime via the OpenBao agent. No plaintext secrets exist in source code, configuration files, or environment variables. |
| **Evidence** | OpenBao configuration and policies; Kubernetes pod specs showing secret injection; source code scan results confirming no hardcoded secrets; OpenBao audit logs showing secret access patterns. |

### CC9.3 — Encryption at Rest

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC9.3 |
| **Description** | Data at rest is encrypted using strong cryptographic algorithms. |
| **Implementation** | AES-GCM-256 envelope encryption protects all data at rest. The Key Management Service (KMS) derives Data Encryption Keys (DEKs) which are used to encrypt data. The DEK itself is encrypted by the KMS master key and stored alongside the ciphertext. Per-dataClass key derivation ensures different data classifications use different encryption keys. |
| **Evidence** | Encryption implementation source code; KMS configuration; encrypted data samples showing ciphertext format; key derivation logic; dataClass-to-key mapping configuration. |

### CC9.4 — Tamper Detection

| Attribute | Detail |
|-----------|--------|
| **Control ID** | CC9.4 |
| **Description** | Data integrity is protected through cryptographic mechanisms that detect unauthorized modification. |
| **Implementation** | Every transaction extends a SHA-256 audit hash chain per account. Each hash incorporates the previous hash, transaction ID, account ID, amount, type, and timestamp — creating an append-only log where any tampering breaks the chain. Additionally, every processed transaction receives an ECDSA P-256 signed receipt that independently verifies integrity. |
| **Evidence** | Hash chain implementation source code; hash chain verification tool output; signed receipt samples with signature verification; chain integrity audit reports. |
