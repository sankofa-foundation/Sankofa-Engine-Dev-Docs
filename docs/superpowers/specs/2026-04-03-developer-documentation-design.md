# Sankofa Engine Developer Documentation — Design Spec

**Date:** 2026-04-03
**Version:** v0.1.0-alpha
**Status:** Approved

---

## 1. Purpose

Build comprehensive developer documentation for the Sankofa Engine, a sharded privacy-preserving financial ledger for digital assets. The documentation serves two primary audiences:

1. **API consumers / integrators** — Enterprise developers building applications on top of the Sankofa Engine REST API
2. **Security reviewers / auditors** — Vendor risk assessment officers, SOC 2 auditors, and compliance teams evaluating the platform

The engine is deployed and managed by Sankofa Labs Inc. Source code is available for review by Enterprise customers. A free non-commercial (learning) version is planned for early 2027.

## 2. Technology Stack

- **Static site generator:** Hugo
- **Theme:** Docsy (Google's documentation theme, used by Kubernetes, gRPC, Istio)
- **API docs:** Hand-authored OpenAPI 3.0 spec + embedded Swagger UI
- **Deployment:** GitHub Pages via GitHub Actions
- **Styling:** Professional light-mode theme (dark mode planned for later)

### Why Hugo + Docsy

- Purpose-built for technical documentation in the Go/infrastructure ecosystem
- Sub-100ms builds, trivial GitHub Pages deployment
- Native Swagger UI embedding via shortcodes
- Enterprise-appropriate appearance for SOC 2 / banking audiences
- Markdown-based authoring

## 3. Site Structure

```
/                                    Landing page (hero + quick nav)
├── /overview/                       Product overview & architecture
│   ├── /overview/introduction/      What is Sankofa Engine, value prop, licensing
│   ├── /overview/architecture/      Hexagonal arch, service topology, data flow diagrams
│   ├── /overview/services/          Each microservice: role, ports, dependencies
│   └── /overview/security/          Security model overview (links to /security/)
│
├── /api-reference/                  API documentation
│   ├── /api-reference/overview/     Auth flows (JWT + tx signing), error format (RFC 7807)
│   ├── /api-reference/swagger/      Embedded Swagger UI (from openapi.yaml)
│   ├── /api-reference/transactions/ Endpoint details + curl examples
│   ├── /api-reference/accounts/     Account state, balances, NFTs
│   ├── /api-reference/tokens/       Fungible tokens + NFT classes/instances
│   ├── /api-reference/compliance/   ZKP proofs, assertions, attestations
│   └── /api-reference/webhooks/     Event notifications (placeholder/coming soon)
│
├── /guides/                         Integration guides
│   ├── /guides/quickstart/          First API call walkthrough
│   ├── /guides/transaction-flow/    End-to-end transaction lifecycle
│   ├── /guides/self-custody/        Self-custody signing model setup
│   ├── /guides/nft-lifecycle/       Minting, transferring, provenance
│   └── /guides/compliance-proofs/   Generating and verifying ZKP proofs
│
├── /reference/                      Technical reference
│   ├── /reference/configuration/    config.yaml schema, env vars, defaults
│   ├── /reference/libraries/        Go dependencies with versions and purposes
│   ├── /reference/data-model/       Domain entities, key types, amount handling
│   └── /reference/error-codes/      Complete error code reference
│
├── /security/                       Security & compliance (SOC 2 audience)
│   ├── /security/controls/          Security controls → SOC 2 Trust Service Criteria
│   ├── /security/encryption/        At rest (AES-GCM-256), in transit (mTLS), key mgmt
│   ├── /security/access-control/    RBAC (Casbin), JWT auth, API key provisioning
│   ├── /security/audit-logging/     Hash chains, receipt signing, provenance
│   └── /security/data-residency/    Storage tiers, retention policies, archival
│
├── /changelog/                      Version history
│   └── /changelog/v0.1.0-alpha/     Current release notes
│
└── /support/                        Support & contact
    └── /support/contact/            How to reach Sankofa Labs
```

### Design Rationale

- **Security is top-level** — vendor risk officers and SOC 2 auditors find it immediately, not buried under "reference"
- **Dual API docs** — Swagger UI for interactive exploration + written pages for context, best practices, edge cases
- **Guides are task-oriented** ("how do I submit a transaction") vs. reference being lookup-oriented ("what error codes exist")
- **Changelog scales** — supports per-release pages now, per-microservice version notes later

## 4. API Reference Design

### OpenAPI Spec

- **Location:** `static/openapi/openapi.yaml`
- **Format:** OpenAPI 3.0, hand-authored
- **Coverage:** All 25+ API Gateway endpoints
- **Auth schemes:**
  - `BearerAuth` — JWT token obtained via API key exchange
  - `TransactionSignature` — ECDSA P-256 signed payload for self-custody model
- **Error schema:** RFC 7807 Problem Details with all error codes
- **Migration path:** Move to code-generated spec from Go handler annotations when the API stabilizes

### Swagger UI

- Embedded at `/api-reference/swagger/` via Hugo shortcode
- Swagger UI dist bundle served from static assets (no external CDN)
- Same site navigation — not a separate application

### Written Endpoint Pages

Each endpoint group gets a dedicated page with:
- Narrative explanation of purpose and usage context
- `curl` examples with realistic payloads and responses
- Idempotency behavior, edge cases, error scenarios
- Cross-references to Swagger UI for interactive testing

## 5. Security & Compliance Documentation (SOC 2 Readiness)

### Security Controls Inventory

Organized by SOC 2 Trust Service Categories:

| Category | Coverage |
|----------|----------|
| **CC6 — Logical & Physical Access** | JWT auth, mTLS, RBAC (Casbin), API key provisioning |
| **CC7 — System Operations** | Health probes, HPA autoscaling, NATS JetStream durability |
| **CC8 — Change Management** | Sankofa Labs managed deployments, versioned releases, changelog |
| **CC9 — Risk Mitigation** | K8s network policies, namespace isolation, secret management (OpenBao) |

Each control documents: what it is, how it's implemented, where in the codebase/config it lives.

### Encryption Documentation

- **At rest:** AES-GCM-256 envelope encryption, KMS-derived DEKs, per-dataClass key isolation
- **In transit:** mTLS between services, TLS for client-to-API
- **Key management:** OpenBao transit backend, AWS KMS support, key rotation model
- Envelope encryption flow diagrams

### Access Control Documentation

- API key → JWT exchange authentication flow
- Casbin RBAC model: policy structure, role definitions
- ECDSA P-256 transaction signing for self-custody authorization
- Principle of least privilege in K8s deployment

### Audit Logging Documentation

- SHA-256 hash chain per account (every transaction extends the chain)
- ECDSA P-256 signed receipts (tamper-evident processing proof)
- Checkpoint-based hash chain verification
- NFT provenance tracking
- 7-year NATS JetStream retention for event replay

### Data Residency & Retention

- Hot tier: ScyllaDB (configurable, default 2 years)
- Projection tier: PostgreSQL (current state read model)
- Cold tier: S3/local filesystem (long-term archival)
- Retention policies, archival CronJob, data purge capabilities

## 6. Technical Reference

### Configuration Reference

- Full `config.yaml` schema: field-by-field with types, defaults, descriptions
- Environment variable override mapping table
- Per-service configuration differences
- Example configs for common scenarios

### Libraries & Dependencies

All direct Go dependencies from `go.mod`, grouped by category:

| Category | Packages |
|----------|----------|
| **HTTP** | `gofiber/fiber/v3` v3.1.0 |
| **Database** | `gocql` v1.7.0 (ScyllaDB), `pgx/v5` v5.9.1 (PostgreSQL) |
| **Messaging** | `nats.go` v1.49.0, `nats-server/v2` v2.12.6 |
| **Auth** | `golang-jwt/v5` v5.3.1, `casbin/v2` v2.135.0 |
| **Crypto** | `golang.org/x/crypto` v0.49.0 |
| **Cloud** | `aws-sdk-go-v2` (config v1.29.5, KMS v1.43.0) |
| **Utilities** | `google/uuid` v1.6.0, `shirou/gopsutil/v3` v3.24.5, `gopkg.in/yaml.v3` |
| **Testing** | `stretchr/testify` v1.11.1 |

Go version requirement: 1.25+

### Data Model Reference

- Domain entities with field definitions
- Amount handling: strings, never floats (IEEE 754 avoidance)
- FNV-1a shard routing algorithm
- Idempotency key semantics
- Transaction type and status enums

### Error Code Reference

- All domain error codes with descriptions and resolution guidance
- RFC 7807 response format with field explanations
- HTTP status code mapping
- Example error responses

## 7. Integration Guides

| Guide | Purpose | Key Content |
|-------|---------|-------------|
| **Quickstart** | First API call in 5 min | Auth → submit tx → poll status |
| **Transaction Flow** | Full lifecycle understanding | 7-step flow with diagrams |
| **Self-Custody** | Signature-based auth setup | Keypair generation, signing, submission (Go/Python/JS examples) |
| **NFT Lifecycle** | Token management | Register class → mint → transfer → provenance |
| **Compliance Proofs** | ZKP operations | Generate and verify proofs, high-level ZKP concepts |

## 8. Versioning & Changelog

- **Engine version:** `v0.1.0-alpha` (tagged in GitHub)
- **Versioning scheme:** Semver, API path prefix `v1` maps to major version
- **Per-microservice versions:** Tracked within release notes as the engine matures
- **Changelog format per release:** Summary, breaking changes, new features, fixes, known issues, dependency updates

## 9. Deployment

- **Target:** GitHub Pages
- **CI:** GitHub Actions workflow — on push to `main`, build Hugo site, deploy to `gh-pages` branch
- **Custom domain:** Configurable via CNAME (future)
- **Build time:** Expected <1 second for initial site size

## 10. Hugo Project Structure (in this repo)

```
Sankofa-Engine-Dev-Docs/
├── hugo.toml                        Hugo configuration
├── static/
│   └── openapi/
│       └── openapi.yaml             Hand-authored OpenAPI spec
├── assets/                          Custom CSS overrides
├── layouts/                         Custom layout overrides (Swagger shortcode)
├── content/
│   ├── _index.md                    Landing page
│   ├── overview/
│   ├── api-reference/
│   ├── guides/
│   ├── reference/
│   ├── security/
│   ├── changelog/
│   └── support/
├── .github/
│   └── workflows/
│       └── deploy.yml               GitHub Pages deployment
└── docs/                            Internal specs (not published)
```

## 11. Non-Goals (Explicit Exclusions)

- **No dark mode** in initial release (planned for later)
- **No code-generated OpenAPI spec** — hand-authored now, migrate later
- **No internal/contributor docs** — audience is external enterprise customers only
- **No interactive API playground** beyond Swagger UI
- **No deployment/installation guides** — Sankofa Labs handles all deployment
- **No pricing or commercial terms** — handled outside documentation
