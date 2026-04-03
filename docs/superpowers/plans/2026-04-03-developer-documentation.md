# Sankofa Engine Developer Documentation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a production-ready Hugo + Docsy documentation site for the Sankofa Engine, serving enterprise API consumers and SOC 2 auditors, deployable to GitHub Pages.

**Architecture:** Hugo static site with Docsy theme, hand-authored OpenAPI 3.0 spec with embedded Swagger UI, security documentation mapped to SOC 2 Trust Service Criteria. All content lives in this repo (`Sankofa-Engine-Dev-Docs`), built and deployed via GitHub Actions.

**Tech Stack:** Hugo (static site generator), Docsy (theme via Hugo module), OpenAPI 3.0 (hand-authored), Swagger UI (embedded), GitHub Actions (CI/CD), GitHub Pages (hosting)

**Source Engine Repo:** `C:\Users\Belle\sankofa-engine\` — read-only reference for API routes, domain types, config schema, dependencies.

---

## File Structure

```
Sankofa-Engine-Dev-Docs/
├── hugo.toml                                    # Hugo config (site title, menu, params)
├── go.mod                                       # Go module for Hugo/Docsy
├── go.sum                                       # Go module checksums
├── package.json                                 # PostCSS dependencies (Docsy requirement)
├── assets/
│   └── scss/
│       └── _variables_project.scss              # Light-mode theme overrides
├── layouts/
│   ├── shortcodes/
│   │   └── swaggerui.html                       # Swagger UI embed shortcode
│   └── partials/
│       └── hooks/
│           └── head-end.html                    # Swagger UI CSS injection
├── static/
│   ├── openapi/
│   │   └── openapi.yaml                         # Hand-authored OpenAPI 3.0 spec
│   └── swagger-ui/
│       ├── swagger-ui-bundle.js                 # Swagger UI dist
│       ├── swagger-ui-standalone-preset.js      # Swagger UI preset
│       └── swagger-ui.css                       # Swagger UI styles
├── content/
│   ├── _index.md                                # Landing page
│   ├── overview/
│   │   ├── _index.md                            # Overview section landing
│   │   ├── introduction.md                      # Product intro, value prop, licensing
│   │   ├── architecture.md                      # Hexagonal arch, service topology
│   │   └── services.md                          # Microservice reference
│   ├── api-reference/
│   │   ├── _index.md                            # API reference section landing
│   │   ├── authentication.md                    # Auth flows (JWT + tx signing)
│   │   ├── swagger.md                           # Embedded Swagger UI page
│   │   ├── transactions.md                      # Transaction endpoints
│   │   ├── accounts.md                          # Account endpoints
│   │   ├── tokens.md                            # Fungible token + NFT endpoints
│   │   └── compliance.md                        # Compliance/assertion endpoints
│   ├── guides/
│   │   ├── _index.md                            # Guides section landing
│   │   ├── quickstart.md                        # First API call
│   │   ├── transaction-flow.md                  # End-to-end lifecycle
│   │   ├── self-custody.md                      # Self-custody signing
│   │   ├── nft-lifecycle.md                     # NFT operations
│   │   └── compliance-proofs.md                 # ZKP proof generation
│   ├── reference/
│   │   ├── _index.md                            # Reference section landing
│   │   ├── configuration.md                     # config.yaml + env vars
│   │   ├── libraries.md                         # Go dependencies
│   │   ├── data-model.md                        # Domain entities
│   │   └── error-codes.md                       # Error code reference
│   ├── security/
│   │   ├── _index.md                            # Security section landing
│   │   ├── controls.md                          # SOC 2 controls inventory
│   │   ├── encryption.md                        # Encryption at rest/in transit
│   │   ├── access-control.md                    # RBAC, JWT, API keys
│   │   ├── audit-logging.md                     # Hash chains, receipts
│   │   └── data-residency.md                    # Storage tiers, retention
│   ├── changelog/
│   │   ├── _index.md                            # Changelog section landing
│   │   └── v0.1.0-alpha.md                      # Initial release notes
│   └── support/
│       └── _index.md                            # Support & contact
├── .github/
│   └── workflows/
│       └── deploy-docs.yml                      # GitHub Pages deployment
└── .gitignore                                   # Hugo build artifacts
```

---

## Task 1: Hugo + Docsy Project Scaffold

**Files:**
- Create: `hugo.toml`
- Create: `go.mod`
- Create: `package.json`
- Create: `.gitignore`
- Create: `assets/scss/_variables_project.scss`

- [ ] **Step 1: Initialize Hugo module**

Run:
```bash
cd "C:/Users/Belle/Sankofa-Engine-Dev-Docs"
hugo new site . --force
```

This creates the base Hugo structure. The `--force` flag allows init in a non-empty directory (we already have `README.md` and `docs/`).

- [ ] **Step 2: Initialize Go module and add Docsy**

Run:
```bash
cd "C:/Users/Belle/Sankofa-Engine-Dev-Docs"
hugo mod init github.com/sankofa-labs/sankofa-engine-dev-docs
hugo mod get github.com/google/docsy@latest
```

- [ ] **Step 3: Write hugo.toml**

Replace the generated `hugo.toml` with:

```toml
baseURL = "https://sankofa-labs.github.io/sankofa-engine-dev-docs/"
title = "Sankofa Engine Documentation"
contentDir = "content"
defaultContentLanguage = "en"
defaultContentLanguageInSubdir = false

# Docsy theme module
[module]
  proxy = "direct"
  [module.hugoVersion]
    extended = true
    min = "0.110.0"
  [[module.imports]]
    path = "github.com/google/docsy"
    disable = false

# Language config
[languages]
  [languages.en]
    languageName = "English"
    weight = 1
    [languages.en.params]
      title = "Sankofa Engine Documentation"
      description = "Developer documentation for the Sankofa Engine — a sharded, privacy-preserving financial ledger for digital assets."

# Markup config
[markup]
  [markup.goldmark]
    [markup.goldmark.parser.attribute]
      block = true
    [markup.goldmark.renderer]
      unsafe = true
  [markup.highlight]
    style = "github"

# Site params
[params]
  copyright = "Sankofa Labs Inc."
  version = "v0.1.0-alpha"
  archived_version = false
  version_menu = "Versions"
  url_latest_version = "https://sankofa-labs.github.io/sankofa-engine-dev-docs/"

  # Docsy UI params
  offlineSearch = true
  prism_syntax_highlighting = false

  [params.ui]
    sidebar_menu_compact = true
    sidebar_menu_foldable = true
    sidebar_search_disable = false
    navbar_logo = false
    footer_about_disable = true
    breadcrumb_disable = false

  [params.ui.feedback]
    enable = false

  [params.ui.readingtime]
    enable = false

  [params.links]
    [[params.links.developer]]
      name = "GitHub"
      url = "https://github.com/sankofa-labs/sankofa-engine"
      icon = "fab fa-github"
      desc = "Source code (Enterprise customers)"

# Top-level menu
[menu]
  [[menu.main]]
    name = "Overview"
    weight = 10
    url = "/overview/"
  [[menu.main]]
    name = "API Reference"
    weight = 20
    url = "/api-reference/"
  [[menu.main]]
    name = "Guides"
    weight = 30
    url = "/guides/"
  [[menu.main]]
    name = "Reference"
    weight = 40
    url = "/reference/"
  [[menu.main]]
    name = "Security"
    weight = 50
    url = "/security/"
  [[menu.main]]
    name = "Changelog"
    weight = 60
    url = "/changelog/"
```

- [ ] **Step 4: Create package.json for PostCSS (Docsy requirement)**

```json
{
  "name": "sankofa-engine-dev-docs",
  "version": "0.1.0",
  "description": "Sankofa Engine developer documentation",
  "private": true,
  "devDependencies": {
    "autoprefixer": "^10.4.20",
    "postcss": "^8.5.3",
    "postcss-cli": "^11.0.1"
  }
}
```

- [ ] **Step 5: Create light-mode theme overrides**

Create `assets/scss/_variables_project.scss`:

```scss
// Professional light-mode theme for Sankofa Engine docs
// Dark mode will be added in a future release

// Brand colors — clean, professional, fintech-appropriate
$primary: #1a56db;       // Deep blue — trust, finance
$secondary: #6b7280;     // Neutral gray
$success: #059669;       // Green — confirmations
$warning: #d97706;       // Amber — cautions
$danger: #dc2626;        // Red — errors

// Typography
$font-family-sans-serif: "Inter", -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
$font-family-monospace: "JetBrains Mono", "Fira Code", "Cascadia Code", SFMono-Regular, Menlo, Monaco, Consolas, monospace;

// Navbar
$navbar-bg: #ffffff;
$navbar-text-color: #1f2937;

// Sidebar
$sidebar-bg: #f9fafb;

// Code blocks
$code-bg: #f3f4f6;
```

- [ ] **Step 6: Update .gitignore**

```gitignore
# Hugo
/public/
/resources/_gen/
/.hugo_build.lock
/node_modules/

# OS
.DS_Store
Thumbs.db

# IDE
.idea/
.vscode/
*.swp
```

- [ ] **Step 7: Install npm dependencies and verify build**

Run:
```bash
cd "C:/Users/Belle/Sankofa-Engine-Dev-Docs"
npm install
hugo --gc --minify 2>&1 | head -20
```

Expected: Hugo builds successfully (may warn about missing content, that's fine).

- [ ] **Step 8: Commit**

```bash
git add hugo.toml go.mod go.sum package.json package-lock.json .gitignore assets/
git commit -m "feat: scaffold Hugo + Docsy documentation site

Initialize Hugo project with Docsy theme module, professional light-mode
theme overrides, PostCSS toolchain, and site configuration."
```

---

## Task 2: Landing Page and Content Skeleton

**Files:**
- Create: `content/_index.md`
- Create: `content/overview/_index.md`
- Create: `content/api-reference/_index.md`
- Create: `content/guides/_index.md`
- Create: `content/reference/_index.md`
- Create: `content/security/_index.md`
- Create: `content/changelog/_index.md`
- Create: `content/support/_index.md`

- [ ] **Step 1: Create landing page**

Create `content/_index.md`:

```markdown
---
title: "Sankofa Engine"
linkTitle: "Sankofa Engine"
type: docs
---

{{% blocks/cover title="Sankofa Engine" image_anchor="top" height="med" color="primary" %}}
<p class="lead mt-4">Developer Documentation</p>
<p>A sharded, privacy-preserving financial ledger engine for digital assets.</p>
<p>
  <a class="btn btn-lg btn-primary me-3 mb-4" href="{{< relref "/overview/introduction" >}}">
    Get Started
  </a>
  <a class="btn btn-lg btn-secondary me-3 mb-4" href="{{< relref "/api-reference" >}}">
    API Reference
  </a>
</p>
<p class="lead mt-2">
  <span class="badge bg-info">v0.1.0-alpha</span>
</p>
{{% /blocks/cover %}}

{{% blocks/section color="white" type="row" %}}

{{% blocks/feature icon="fa-book" title="Overview" url="/overview/" %}}
Architecture, service topology, and security model of the Sankofa Engine.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-code" title="API Reference" url="/api-reference/" %}}
Complete REST API documentation with interactive Swagger UI and curl examples.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-rocket" title="Guides" url="/guides/" %}}
Step-by-step integration guides: quickstart, transaction flow, self-custody signing, NFTs, and compliance proofs.
{{% /blocks/feature %}}

{{% /blocks/section %}}

{{% blocks/section color="light" type="row" %}}

{{% blocks/feature icon="fa-shield-alt" title="Security & Compliance" url="/security/" %}}
Security controls inventory mapped to SOC 2 Trust Service Criteria. Encryption, access control, audit logging, and data residency.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-cogs" title="Technical Reference" url="/reference/" %}}
Configuration schema, libraries and dependencies, data model, and error codes.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-list" title="Changelog" url="/changelog/" %}}
Version history, release notes, and breaking changes.
{{% /blocks/feature %}}

{{% /blocks/section %}}

---

**Sankofa Engine** is developed by [Sankofa Labs Inc.](https://sankofalabs.io) Enterprise customers receive full source code access for review. All installation, deployment, and upgrades are performed by Sankofa Labs. A free non-commercial (learning) version is planned for early 2027.
```

- [ ] **Step 2: Create section index pages**

Create `content/overview/_index.md`:

```markdown
---
title: "Overview"
linkTitle: "Overview"
weight: 10
description: >
  Product overview, architecture, services, and security model.
---
```

Create `content/api-reference/_index.md`:

```markdown
---
title: "API Reference"
linkTitle: "API Reference"
weight: 20
description: >
  Complete REST API documentation with interactive Swagger UI, authentication guides, and endpoint references.
---

The Sankofa Engine exposes a REST API via the API Gateway service on port 8080. All endpoints use JSON request/response bodies and return [RFC 7807](https://datatracker.ietf.org/doc/html/rfc7807) Problem Details for errors.

**Explore the API:**

- [Authentication](/api-reference/authentication/) — How to authenticate (JWT and transaction signing)
- [Interactive API Explorer (Swagger UI)](/api-reference/swagger/) — Try endpoints interactively
- [Transactions](/api-reference/transactions/) — Submit and query transactions
- [Accounts](/api-reference/accounts/) — Account state and balances
- [Tokens](/api-reference/tokens/) — Fungible tokens and NFTs
- [Compliance](/api-reference/compliance/) — Zero-knowledge proofs and attestations
```

Create `content/guides/_index.md`:

```markdown
---
title: "Guides"
linkTitle: "Guides"
weight: 30
description: >
  Step-by-step integration guides for common tasks.
---
```

Create `content/reference/_index.md`:

```markdown
---
title: "Technical Reference"
linkTitle: "Reference"
weight: 40
description: >
  Configuration, libraries, data model, and error codes.
---
```

Create `content/security/_index.md`:

```markdown
---
title: "Security & Compliance"
linkTitle: "Security"
weight: 50
description: >
  Security controls, encryption, access control, audit logging, and data residency. Mapped to SOC 2 Trust Service Criteria.
---

This section documents the security architecture of the Sankofa Engine for enterprise security reviews, vendor risk assessments, and SOC 2 audit readiness.

| Section | Description |
|---------|-------------|
| [Security Controls](/security/controls/) | Controls inventory mapped to SOC 2 Trust Service Categories |
| [Encryption](/security/encryption/) | Encryption at rest (AES-GCM-256), in transit (mTLS), and key management |
| [Access Control](/security/access-control/) | RBAC, JWT authentication, API key provisioning |
| [Audit Logging](/security/audit-logging/) | Cryptographic audit hash chains, signed receipts, provenance |
| [Data Residency](/security/data-residency/) | Storage tiers, retention policies, archival |
```

Create `content/changelog/_index.md`:

```markdown
---
title: "Changelog"
linkTitle: "Changelog"
weight: 60
description: >
  Version history and release notes.
---
```

Create `content/support/_index.md`:

```markdown
---
title: "Support"
linkTitle: "Support"
weight: 70
description: >
  Contact Sankofa Labs for support, issues, and questions.
---

## Contact

For technical support, questions, or to report issues with the Sankofa Engine:

- **Email:** support@sankofalabs.io
- **Enterprise Support Portal:** Available to customers with active support agreements

## Licensing

- **Enterprise:** Source code available for review. Sankofa Labs manages all deployment and upgrades.
- **Non-Commercial / Learning:** Free version planned for early 2027.

## Responsible Disclosure

For security vulnerabilities, please contact **security@sankofalabs.io**. Do not open public issues for security reports.
```

- [ ] **Step 3: Verify build with content**

Run:
```bash
cd "C:/Users/Belle/Sankofa-Engine-Dev-Docs"
hugo --gc --minify 2>&1 | tail -5
```

Expected: Successful build with pages generated.

- [ ] **Step 4: Commit**

```bash
git add content/
git commit -m "feat: add landing page and content section skeleton

Create landing page with navigation cards and all section index pages:
overview, API reference, guides, reference, security, changelog, support."
```

---

## Task 3: Overview Section — Introduction, Architecture, Services

**Files:**
- Create: `content/overview/introduction.md`
- Create: `content/overview/architecture.md`
- Create: `content/overview/services.md`

- [ ] **Step 1: Write introduction page**

Create `content/overview/introduction.md`:

```markdown
---
title: "Introduction"
linkTitle: "Introduction"
weight: 1
description: >
  What is the Sankofa Engine and what problems does it solve.
---

## What is the Sankofa Engine?

The Sankofa Engine is a **sharded, privacy-preserving financial ledger engine** designed for digital asset management. It provides cryptographically auditable transaction processing with zero-knowledge proof capabilities for regulated financial environments.

### Key Capabilities

| Capability | Description |
|------------|-------------|
| **Sharded Ledger** | Horizontally scalable transaction processing via FNV-1a hash-based shard routing |
| **Privacy-Preserving** | Encrypted balances with zero-knowledge proof assertions (proof-of-liabilities, proof-of-provenance, proof-of-compliance) |
| **Cryptographic Auditability** | SHA-256 hash chains per account, ECDSA P-256 signed receipts for tamper-evident processing |
| **Multi-Asset Support** | Fungible tokens and NFTs with full lifecycle management (mint, transfer, burn, provenance) |
| **Exactly-Once Processing** | NATS JetStream with idempotency keys ensures each transaction is processed exactly once |
| **Regulatory Ready** | SOC 2 aligned security controls, 7-year event retention, compliance proof generation |

### How It Works

1. **Submit** — Clients submit transactions via the REST API with JWT authentication or ECDSA transaction signing
2. **Route** — The API Gateway routes transactions to the correct shard using deterministic FNV-1a hashing
3. **Process** — Shard Workers batch transactions into blocks, validate balances, and update the ledger
4. **Sign** — Each processed transaction receives an ECDSA P-256 signed receipt with an audit hash chain entry
5. **Project** — The Projection Service updates read-optimized balance views in PostgreSQL
6. **Prove** — The Compliance Service generates zero-knowledge proofs over encrypted ledger state

### Deployment Model

The Sankofa Engine is deployed and managed by **Sankofa Labs Inc.** as a managed service:

- **Installation & upgrades** are performed by Sankofa Labs
- **Source code** is available for review by Enterprise customers
- **Kubernetes-native** — all services deploy as containers to Kubernetes clusters
- A **free non-commercial version** (for learning and evaluation) is planned for early 2027

### Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| HTTP Framework | Go + Fiber v3 | REST API serving |
| Ledger Storage | ScyllaDB 6.2 | High-throughput transaction storage |
| Balance Projections | PostgreSQL 16 | CQRS read model for account balances |
| Messaging | NATS 2.11+ with JetStream | Event-driven routing, exactly-once delivery |
| Key Management | OpenBao (Vault fork) | Encryption key management, transit encryption |
| Access Control | Casbin v2 | Role-based access control (RBAC) |
| Authentication | golang-jwt/v5 | JWT token generation and validation |
| Encryption | AES-GCM-256 + KMS | Envelope encryption for data at rest |
```

- [ ] **Step 2: Write architecture page**

Create `content/overview/architecture.md`:

```markdown
---
title: "Architecture"
linkTitle: "Architecture"
weight: 2
description: >
  Hexagonal architecture, service topology, and data flow.
---

## Architectural Style

The Sankofa Engine uses **Hexagonal Architecture** (Ports & Adapters) combined with **Domain-Driven Design** (DDD):

- **Domain Core** (`internal/core/`) — Pure business logic with zero infrastructure dependencies. Contains domain entities (Transaction, Account, NFT, FungibleToken) and port interfaces.
- **Ports** (`internal/core/port/`) — Interfaces that define contracts between the domain and the outside world. Input ports (what the domain offers) and output ports (what the domain needs).
- **Adapters** (`internal/adapter/`) — Implementations of ports that connect to real infrastructure (ScyllaDB, PostgreSQL, NATS, OpenBao).
- **Services** (`internal/service/`) — Business logic orchestration that composes domain entities and ports.

This separation ensures the domain logic can be tested without infrastructure and that storage, messaging, or encryption backends can be swapped without changing business rules.

## Service Topology

The engine runs as a set of cooperating microservices deployed to Kubernetes:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                       │
│                                                                 │
│  ┌──────────┐    ┌───────────────┐    ┌────────────────────┐   │
│  │   API     │───▶│     NATS      │◀───│  Shard Orchestrator│   │
│  │ Gateway   │    │  (JetStream)  │    │  (leader election) │   │
│  │ :8080     │    │               │    └────────────────────┘   │
│  └──────────┘    │               │                              │
│                  │               │    ┌────────────────────┐   │
│                  │               │───▶│   Shard Worker(s)  │   │
│                  │               │    │  (ledger processing)│   │
│                  │               │    └────────┬───────────┘   │
│                  │               │             │               │
│                  │               │    ┌────────▼───────────┐   │
│                  │               │◀───│   Projection       │   │
│                  │               │    │  (balance views)    │   │
│                  └───────────────┘    └────────────────────┘   │
│                                                                 │
│  ┌──────────┐  ┌──────────────┐  ┌─────────────────────────┐  │
│  │ScyllaDB  │  │ PostgreSQL   │  │  Compliance │ Settlement│  │
│  │(ledger)  │  │ (projections)│  │  (ZKP)      │ (finality)│  │
│  └──────────┘  └──────────────┘  └─────────────────────────┘  │
│                                                                 │
│  ┌──────────┐  ┌──────────────┐                                │
│  │ OpenBao  │  │  Archival    │                                │
│  │ (keys)   │  │  (cold tier) │                                │
│  └──────────┘  └──────────────┘                                │
└─────────────────────────────────────────────────────────────────┘
```

## Transaction Data Flow

A transaction flows through the system in the following stages:

### 1. Submission (API Gateway)

The client sends a `POST /v1/transactions` request to the API Gateway. The gateway validates the request schema, generates a transaction ID from the idempotency key, and publishes the transaction to NATS.

### 2. Shard Routing

Transactions are deterministically routed to shards using FNV-1a hashing:

```
shard_id = FNV-1a(account_id) % shard_count
```

This ensures all transactions for a given account are always processed by the same shard, maintaining consistency without cross-shard coordination.

### 3. Block Processing (Shard Worker)

Shard Workers consume transactions from their assigned NATS subjects, batch them into blocks, and process each block:

1. Read account balance from in-memory cache
2. Validate transaction (parse amount, check balance for debits)
3. Update balance
4. Extend the SHA-256 audit hash chain
5. Sign a receipt with ECDSA P-256
6. Batch INSERT to ScyllaDB
7. Publish receipt to NATS
8. Acknowledge the NATS message

### 4. Projection (Read Model)

The Projection Service subscribes to transaction receipts and updates the PostgreSQL balance projection. This provides a fast, denormalized read model for `GET /v1/accounts/{id}/balances` queries.

### 5. Archival (Cold Storage)

The Archival Service runs on a schedule (Kubernetes CronJob) and offloads aged transactions from ScyllaDB (hot tier) to S3 or local filesystem (cold tier) based on configurable retention policies.

## Exactly-Once Semantics

The engine guarantees exactly-once transaction processing through three mechanisms:

1. **Idempotency keys** — Each transaction includes a client-provided idempotency key. Duplicate submissions return the original receipt.
2. **NATS JetStream deduplication** — The `Nats-Msg-Id` header is set to the idempotency key, enabling server-side deduplication within the message retention window.
3. **Ack-after-commit** — Shard Workers only acknowledge NATS messages after the ScyllaDB write succeeds, ensuring no data loss on consumer crashes.

## Storage Architecture

| Tier | Technology | Purpose | Retention |
|------|-----------|---------|-----------|
| **Hot** | ScyllaDB | Ledger transactions, account state | Configurable (default 2 years) |
| **Projection** | PostgreSQL | Denormalized balance views (CQRS read model) | Current state |
| **Cold** | S3 / Local filesystem | Archived transactions | Long-term (configurable) |
| **Event Log** | NATS JetStream | Transaction events, receipts | 7 years |
```

- [ ] **Step 3: Write services page**

Create `content/overview/services.md`:

```markdown
---
title: "Services"
linkTitle: "Services"
weight: 3
description: >
  Microservices that comprise the Sankofa Engine.
---

The Sankofa Engine consists of the following microservices, each deployed as a separate container in Kubernetes.

## API Gateway

| Property | Value |
|----------|-------|
| **Port** | 8080 (HTTP), 9090 (health) |
| **Replicas** | 2 (default), autoscaled via HPA |
| **Dependencies** | NATS |

The public entry point for all REST API requests. Routes requests to backend services via NATS request-reply (RPC) or publishes transactions to NATS JetStream for async processing. Maintains an in-memory shard map cache for routing decisions.

**Key responsibilities:**
- Request validation and schema enforcement
- JWT authentication and RBAC authorization
- Rate limiting
- Transaction routing to correct shard via FNV-1a hash
- Proxying RPC calls to backend services (audit queries, token operations, compliance proofs)

## Shard Worker

| Property | Value |
|----------|-------|
| **Port** | 9090 (health only) |
| **Replicas** | 3 (default), autoscaled via HPA |
| **Dependencies** | NATS, ScyllaDB, OpenBao |

Processes transactions for assigned shards. Each worker instance is assigned one or more shards by the Shard Orchestrator and consumes transactions from the corresponding NATS subjects.

**Key responsibilities:**
- Block processing (batch transaction validation and ledger updates)
- In-memory balance cache for nanosecond-speed reads
- SHA-256 audit hash chain maintenance
- ECDSA P-256 receipt signing
- ScyllaDB batch writes

## Shard Orchestrator

| Property | Value |
|----------|-------|
| **Port** | 9090 (health only) |
| **Replicas** | 3 (for leader election quorum) |
| **Dependencies** | NATS |

Manages shard-to-worker assignment. Performs leader election and rebalances shards when workers join or leave the cluster.

**Key responsibilities:**
- Leader election via NATS
- Shard assignment and rebalancing
- Shard map publishing to NATS KV store
- Health monitoring of shard workers

## Projection Service

| Property | Value |
|----------|-------|
| **Port** | 9090 (health only) |
| **Replicas** | 2 (default), autoscaled via HPA |
| **Dependencies** | NATS, PostgreSQL |

Maintains the CQRS read model by subscribing to transaction receipts and updating denormalized balance views in PostgreSQL.

**Key responsibilities:**
- Block and settlement event consumption from NATS
- PostgreSQL balance projection updates
- Fast balance queries for `GET /v1/accounts/{id}/balances`

## Compliance Service

| Property | Value |
|----------|-------|
| **Port** | 9090 (health only) |
| **Replicas** | 2 (default), autoscaled via HPA |
| **Dependencies** | NATS |

Generates and verifies zero-knowledge proofs for compliance assertions.

**Key responsibilities:**
- Proof-of-liabilities generation (ZKP over encrypted balances)
- Proof-of-provenance generation (asset origin verification)
- Proof-of-compliance generation (regulatory rule compliance)
- Proof verification

## Settlement Service

| Property | Value |
|----------|-------|
| **Port** | 9090 (health only) |
| **Replicas** | 1 (default) |
| **Dependencies** | NATS |

Handles settlement finality and batch settlement operations.

**Key responsibilities:**
- Settlement batch processing
- Finality confirmation
- Settlement event publishing

## Archival Service

| Property | Value |
|----------|-------|
| **Deployment** | Kubernetes CronJob (nightly) |
| **Dependencies** | ScyllaDB, S3/local filesystem |

Offloads aged transactions from the hot tier (ScyllaDB) to the cold tier (S3 or local filesystem) based on retention policies.

**Key responsibilities:**
- Retention policy evaluation
- Transaction archival to cold storage
- Archival root reference maintenance
- Cyclic execution on schedule

## Health Checks

All services expose health endpoints on port 9090:

| Endpoint | Purpose |
|----------|---------|
| `GET /healthz/liveness` | Kubernetes liveness probe — is the process alive? |
| `GET /healthz/readiness` | Kubernetes readiness probe — are dependencies connected? |

Readiness probes check downstream dependencies (database connections, NATS connectivity) and return unhealthy if any dependency is unreachable.
```

- [ ] **Step 4: Verify build**

Run:
```bash
cd "C:/Users/Belle/Sankofa-Engine-Dev-Docs"
hugo --gc --minify 2>&1 | tail -5
```

Expected: Successful build.

- [ ] **Step 5: Commit**

```bash
git add content/overview/
git commit -m "feat: add overview section — introduction, architecture, services

Document product capabilities, hexagonal architecture, service topology,
transaction data flow, and all eight microservices with their roles,
ports, replicas, and dependencies."
```

---

## Task 4: Swagger UI Integration

**Files:**
- Create: `layouts/shortcodes/swaggerui.html`
- Create: `layouts/partials/hooks/head-end.html`
- Download: `static/swagger-ui/swagger-ui-bundle.js`
- Download: `static/swagger-ui/swagger-ui-standalone-preset.js`
- Download: `static/swagger-ui/swagger-ui.css`

- [ ] **Step 1: Download Swagger UI dist files**

Run:
```bash
cd "C:/Users/Belle/Sankofa-Engine-Dev-Docs"
mkdir -p static/swagger-ui
SWAGGER_VERSION="5.21.0"
curl -sL "https://unpkg.com/swagger-ui-dist@${SWAGGER_VERSION}/swagger-ui-bundle.js" -o static/swagger-ui/swagger-ui-bundle.js
curl -sL "https://unpkg.com/swagger-ui-dist@${SWAGGER_VERSION}/swagger-ui-standalone-preset.js" -o static/swagger-ui/swagger-ui-standalone-preset.js
curl -sL "https://unpkg.com/swagger-ui-dist@${SWAGGER_VERSION}/swagger-ui.css" -o static/swagger-ui/swagger-ui.css
```

- [ ] **Step 2: Create Swagger UI shortcode**

Create `layouts/shortcodes/swaggerui.html`:

```html
<div id="swagger-ui"></div>
<script src="{{ "swagger-ui/swagger-ui-bundle.js" | relURL }}"></script>
<script src="{{ "swagger-ui/swagger-ui-standalone-preset.js" | relURL }}"></script>
<script>
window.onload = function() {
  SwaggerUIBundle({
    url: "{{ .Get "url" | default ("/openapi/openapi.yaml" | relURL) }}",
    dom_id: '#swagger-ui',
    deepLinking: true,
    presets: [
      SwaggerUIBundle.presets.apis,
      SwaggerUIStandalonePreset
    ],
    plugins: [
      SwaggerUIBundle.plugins.DownloadUrl
    ],
    layout: "StandaloneLayout",
    defaultModelsExpandDepth: 2,
    defaultModelExpandDepth: 2,
    docExpansion: "list",
    filter: true,
    showExtensions: true,
    showCommonExtensions: true,
    tryItOutEnabled: false
  });
};
</script>
```

- [ ] **Step 3: Create head-end partial for Swagger CSS**

Create `layouts/partials/hooks/head-end.html`:

```html
{{ if .HasShortcode "swaggerui" }}
<link rel="stylesheet" href="{{ "swagger-ui/swagger-ui.css" | relURL }}">
<style>
  /* Override Swagger UI to fit within Docsy layout */
  #swagger-ui .topbar { display: none; }
  #swagger-ui .swagger-ui .info { margin: 20px 0; }
  #swagger-ui .swagger-ui .scheme-container { padding: 15px 0; }
</style>
{{ end }}
```

- [ ] **Step 4: Create the Swagger UI page**

Create `content/api-reference/swagger.md`:

```markdown
---
title: "API Explorer (Swagger UI)"
linkTitle: "API Explorer"
weight: 2
description: >
  Interactive API explorer powered by Swagger UI. Browse endpoints, view schemas, and see example requests.
---

{{< swaggerui >}}
```

- [ ] **Step 5: Commit**

```bash
git add layouts/ static/swagger-ui/ content/api-reference/swagger.md
git commit -m "feat: integrate Swagger UI into documentation site

Add Swagger UI dist files, Hugo shortcode for embedding, and API
Explorer page. Swagger UI renders the OpenAPI spec inline within the
Docsy navigation — no separate site."
```

---

## Task 5: OpenAPI 3.0 Specification

**Files:**
- Create: `static/openapi/openapi.yaml`

**Reference files (read-only):**
- `C:\Users\Belle\sankofa-engine\internal\adapter\input\httpapi\gateway_routes.go` — route definitions
- `C:\Users\Belle\sankofa-engine\internal\core\domain\domain.go` — domain entities
- `C:\Users\Belle\sankofa-engine\internal\core\domain\nft.go` — NFT entities
- `C:\Users\Belle\sankofa-engine\internal\core\domain\fungible_token.go` — token entities

- [ ] **Step 1: Read API route definitions from the engine source**

Read the following files from the engine repo to get exact endpoint paths, HTTP methods, and handler names:

```bash
# Read these files for reference (do not modify them):
cat "C:/Users/Belle/sankofa-engine/internal/adapter/input/httpapi/gateway_routes.go"
cat "C:/Users/Belle/sankofa-engine/internal/core/domain/domain.go"
cat "C:/Users/Belle/sankofa-engine/internal/core/domain/nft.go"
cat "C:/Users/Belle/sankofa-engine/internal/core/domain/fungible_token.go"
cat "C:/Users/Belle/sankofa-engine/internal/core/domain/receipt.go"
```

Use the exact types, field names, and endpoint paths from these files when writing the spec.

- [ ] **Step 2: Write the OpenAPI spec**

Create `static/openapi/openapi.yaml`. The spec must cover:

**Info section:**
- Title: Sankofa Engine API
- Version: v0.1.0-alpha
- Description referencing privacy-preserving financial ledger
- Contact: Sankofa Labs Inc.
- License: Proprietary

**Servers:**
- Default: `https://api.example.com` (placeholder, customer-specific)

**Security schemes:**
- `BearerAuth` — JWT Bearer token (`Authorization: Bearer <token>`)
- `TransactionSignature` — ECDSA P-256 signature in request body (for self-custody model)

**Paths — all endpoints from `gateway_routes.go`:**

Transaction endpoints:
- `POST /v1/transactions` — Submit transaction (202 async)
- `GET /v1/transactions/{id}` — Get transaction status
- `GET /v1/transactions` — Query transactions (with query params)
- `GET /v1/transactions/group/{groupID}` — Query by group ID

Account endpoints:
- `GET /v1/accounts/{id}/state` — Account state
- `GET /v1/accounts/{id}/balances` — All token balances
- `GET /v1/accounts/{id}/balances/{tokenId}` — Specific token balance
- `GET /v1/accounts/{id}/nfts` — Owned NFTs

Fungible token endpoints:
- `POST /v1/fungible-tokens` — Register token
- `GET /v1/fungible-tokens` — List tokens
- `GET /v1/fungible-tokens/{id}` — Get token details

NFT endpoints:
- `POST /v1/nft-classes` — Register NFT class
- `GET /v1/nft-classes` — List classes
- `GET /v1/nft-classes/{id}` — Get class details
- `GET /v1/nft-classes/{classId}/instances` — List instances
- `GET /v1/nft-classes/{classId}/instances/{instanceId}` — Get instance
- `GET /v1/nft-classes/{classId}/instances/{instanceId}/history` — Instance history
- `GET /v1/nft-classes/{classId}/instances/{instanceId}/provenance` — Provenance chain

Compliance/assertion endpoints:
- `POST /v1/assertions/proof-of-liabilities` — Generate ZKP
- `POST /v1/assertions/proof-of-provenance` — Generate ZKP
- `POST /v1/assertions/proof-of-compliance` — Generate ZKP
- `POST /v1/assertions/verify` — Verify proof

Attestation endpoints:
- `POST /v1/attestations` — Submit attestation
- `GET /v1/attestations/{id}` — Get attestation
- `GET /v1/attestations` — List attestations

Ledger/monitoring endpoints:
- `GET /v1/ledger/shards/{id}/status` — Shard status
- `GET /v1/storage/metrics` — Storage metrics

Health endpoints:
- `GET /healthz/liveness` — Liveness probe
- `GET /healthz/readiness` — Readiness probe

**Schemas — from domain entities:**
- `Transaction` — id, account_id, type (enum), amount (string), status (enum), signature, idempotency_key, timestamps
- `TransactionSubmission` — request body for POST /v1/transactions
- `TransactionResponse` — response with status, receipt
- `SignedReceipt` — transaction_id, shard_id, audit_hash, signature, timestamp
- `AccountState` — account_id, balances map
- `FungibleTokenType` — id, name, symbol, decimals, total_supply
- `NFTClass` — id, name, metadata_schema
- `NFTInstance` — id, class_id, owner, metadata, minted_at
- `ProofRequest` — type-specific proof parameters
- `ProofResponse` — proof bytes, verification_key, timestamp
- `ProblemDetail` — RFC 7807 error (type, title, status, detail, error_codes)

All amounts must be documented as `type: string` with a note about IEEE 754 avoidance.

This file will be large (~800-1200 lines of YAML). Write the complete spec — no placeholders, no "similar to above", no TODOs.

- [ ] **Step 3: Verify the spec renders in Swagger UI**

Run:
```bash
cd "C:/Users/Belle/Sankofa-Engine-Dev-Docs"
hugo server --port 1313 &
sleep 3
curl -s http://localhost:1313/api-reference/swagger/ | grep -c "swagger-ui"
kill %1
```

Expected: Page contains Swagger UI elements.

- [ ] **Step 4: Commit**

```bash
git add static/openapi/openapi.yaml
git commit -m "feat: add hand-authored OpenAPI 3.0 specification

Complete API spec covering all 25+ endpoints, two auth schemes (JWT +
ECDSA transaction signing), domain entity schemas, and RFC 7807 error
format. Amounts documented as strings per IEEE 754 avoidance policy."
```

---

## Task 6: API Reference Written Pages

**Files:**
- Create: `content/api-reference/authentication.md`
- Create: `content/api-reference/transactions.md`
- Create: `content/api-reference/accounts.md`
- Create: `content/api-reference/tokens.md`
- Create: `content/api-reference/compliance.md`

- [ ] **Step 1: Write authentication page**

Create `content/api-reference/authentication.md`:

Document both auth flows:

1. **JWT Authentication (default):**
   - Sankofa Labs provisions API credentials (client_id + client_secret)
   - Client exchanges credentials for a JWT token
   - JWT is passed as `Authorization: Bearer <token>` on all requests
   - Token expiry, refresh model
   - Include curl example for token exchange and authenticated request

2. **Transaction Signing (self-custody):**
   - Account generates ECDSA P-256 keypair
   - Public key registered with the engine
   - Transaction payload is signed with the private key
   - Signature included in `POST /v1/transactions` request body
   - Engine verifies signature against registered public key
   - Include example showing payload construction, signing, and submission

- [ ] **Step 2: Write transactions endpoint page**

Create `content/api-reference/transactions.md`:

For each endpoint, document:
- HTTP method and path
- Description of what it does
- Request headers, path params, query params, body schema
- Response schema with example JSON
- Idempotency behavior (duplicate keys return original receipt)
- Error responses with RFC 7807 examples
- curl examples with realistic data

Endpoints to cover:
- `POST /v1/transactions`
- `GET /v1/transactions/{id}`
- `GET /v1/transactions`
- `GET /v1/transactions/group/{groupID}`

- [ ] **Step 3: Write accounts endpoint page**

Create `content/api-reference/accounts.md`:

Cover:
- `GET /v1/accounts/{id}/state`
- `GET /v1/accounts/{id}/balances`
- `GET /v1/accounts/{id}/balances/{tokenId}`
- `GET /v1/accounts/{id}/nfts`

Include notes about balances being served from PostgreSQL projection (fast, eventually consistent with ledger).

- [ ] **Step 4: Write tokens endpoint page**

Create `content/api-reference/tokens.md`:

Cover fungible tokens and NFTs:
- `POST /v1/fungible-tokens`, `GET /v1/fungible-tokens`, `GET /v1/fungible-tokens/{id}`
- `POST /v1/nft-classes`, `GET /v1/nft-classes`, `GET /v1/nft-classes/{id}`
- `GET /v1/nft-classes/{classId}/instances`, `GET /v1/nft-classes/{classId}/instances/{instanceId}`
- `GET /v1/nft-classes/{classId}/instances/{instanceId}/history`
- `GET /v1/nft-classes/{classId}/instances/{instanceId}/provenance`

- [ ] **Step 5: Write compliance endpoint page**

Create `content/api-reference/compliance.md`:

Cover assertions and attestations:
- `POST /v1/assertions/proof-of-liabilities`
- `POST /v1/assertions/proof-of-provenance`
- `POST /v1/assertions/proof-of-compliance`
- `POST /v1/assertions/verify`
- `POST /v1/attestations`, `GET /v1/attestations/{id}`, `GET /v1/attestations`

Include high-level explanation of what each proof type demonstrates without requiring ZKP expertise from the reader.

- [ ] **Step 6: Commit**

```bash
git add content/api-reference/
git commit -m "feat: add written API reference pages with curl examples

Document authentication flows (JWT + ECDSA signing), transaction
endpoints, account queries, token/NFT operations, and compliance
proof endpoints with realistic examples and error scenarios."
```

---

## Task 7: Integration Guides

**Files:**
- Create: `content/guides/quickstart.md`
- Create: `content/guides/transaction-flow.md`
- Create: `content/guides/self-custody.md`
- Create: `content/guides/nft-lifecycle.md`
- Create: `content/guides/compliance-proofs.md`

- [ ] **Step 1: Write quickstart guide**

Create `content/guides/quickstart.md`:

Walk through a complete first integration in sequential steps:
1. Obtain API credentials from Sankofa Labs
2. Exchange credentials for JWT token (curl)
3. Submit a transaction (curl with full JSON body)
4. Poll for transaction status (curl)
5. Check account balance (curl)

Each step includes the exact curl command, expected response JSON, and what to look for.

- [ ] **Step 2: Write transaction flow guide**

Create `content/guides/transaction-flow.md`:

Detailed walkthrough of the 7-step transaction lifecycle with diagrams (using ASCII art or Mermaid if Docsy supports it):
1. Client Submit
2. API Gateway validation and routing
3. NATS JetStream publishing
4. Shard Worker block processing
5. Receipt signing
6. Projection update
7. Client poll

Explain idempotency, exactly-once semantics, and what happens on failures at each stage.

- [ ] **Step 3: Write self-custody signing guide**

Create `content/guides/self-custody.md`:

Step-by-step guide with code examples in **Go**, **Python**, and **JavaScript**:
1. Generate ECDSA P-256 keypair
2. Register public key with the engine
3. Construct transaction payload
4. Sign the payload
5. Submit the signed transaction to the public endpoint
6. Verify the receipt

Include complete, runnable code snippets for each language.

- [ ] **Step 4: Write NFT lifecycle guide**

Create `content/guides/nft-lifecycle.md`:

Walkthrough:
1. Register an NFT class (with metadata schema)
2. Mint an NFT instance
3. Query the instance
4. Transfer ownership
5. Query provenance chain
6. Burn an instance

Each step with curl examples and response JSON.

- [ ] **Step 5: Write compliance proofs guide**

Create `content/guides/compliance-proofs.md`:

Guide covering:
1. What are zero-knowledge proofs (2-paragraph accessible explanation)
2. Generate a proof-of-liabilities
3. Generate a proof-of-provenance
4. Generate a proof-of-compliance
5. Verify a proof
6. When to use each proof type

Include curl examples and explain what each proof demonstrates to auditors/regulators.

- [ ] **Step 6: Commit**

```bash
git add content/guides/
git commit -m "feat: add integration guides

Quickstart, transaction flow lifecycle, self-custody signing (Go/Python/JS),
NFT lifecycle, and compliance proof guides with worked examples."
```

---

## Task 8: Security & Compliance Section

**Files:**
- Create: `content/security/controls.md`
- Create: `content/security/encryption.md`
- Create: `content/security/access-control.md`
- Create: `content/security/audit-logging.md`
- Create: `content/security/data-residency.md`

**Reference files (read-only):**
- `C:\Users\Belle\sankofa-engine\internal\adapter\output\encryption\encryption.go`
- `C:\Users\Belle\sankofa-engine\internal\adapter\input\middleware\`
- `C:\Users\Belle\sankofa-engine\internal\service\block\block_processor.go`
- `C:\Users\Belle\sankofa-engine\deploy\k8s\`

- [ ] **Step 1: Write security controls inventory**

Create `content/security/controls.md`:

Organize by SOC 2 Trust Service Categories. For each control:
- **Control ID** (e.g., CC6.1)
- **Category name** (e.g., Logical and Physical Access Controls)
- **Control description** — what the control ensures
- **Implementation** — how it is implemented in the Sankofa Engine (specific technology, config, code reference)
- **Evidence** — what an auditor would examine to verify the control

Cover:
- **CC6 — Logical & Physical Access Controls:** JWT auth, mTLS, Casbin RBAC, API key provisioning, K8s RBAC
- **CC7 — System Operations:** Health probes (liveness/readiness), HPA autoscaling, NATS JetStream durability, monitoring
- **CC8 — Change Management:** Managed deployments by Sankofa Labs, versioned releases, changelog, CI/CD
- **CC9 — Risk Mitigation:** K8s network policies, namespace isolation, secret management (OpenBao), encryption

- [ ] **Step 2: Write encryption documentation**

Create `content/security/encryption.md`:

Document:
- **Encryption at rest:** AES-GCM-256 envelope encryption
  - KMS derives Data Encryption Keys (DEKs)
  - Per-dataClass key derivation for isolation
  - DEK caching strategy
  - Diagram showing envelope encryption flow
- **Encryption in transit:** mTLS between services, TLS for client-to-API
  - Certificate management
  - Cipher suites
- **Key management:** OpenBao transit backend
  - Key hierarchy
  - Key rotation model
  - AWS KMS integration option
  - Key access controls

- [ ] **Step 3: Write access control documentation**

Create `content/security/access-control.md`:

Document:
- **Authentication:**
  - API key provisioning by Sankofa Labs
  - JWT token exchange flow
  - Token lifetime and refresh
  - ECDSA transaction signing (self-custody alternative)
- **Authorization:**
  - Casbin RBAC model
  - Policy structure and role definitions
  - Permission enforcement points
- **Infrastructure access controls:**
  - K8s namespace isolation
  - Network policies
  - Secret scoping
  - Principle of least privilege

- [ ] **Step 4: Write audit logging documentation**

Create `content/security/audit-logging.md`:

Document:
- **SHA-256 audit hash chain:**
  - How the chain works (each transaction extends the hash)
  - Chain verification process
  - Checkpoint-based verification for performance
- **ECDSA P-256 signed receipts:**
  - What receipts contain
  - Signature verification
  - Tamper evidence
- **NFT provenance:**
  - Full ownership history tracking
  - Provenance queries
- **Event retention:**
  - NATS JetStream 7-year retention
  - Event replay capabilities

- [ ] **Step 5: Write data residency documentation**

Create `content/security/data-residency.md`:

Document:
- **Storage tiers:** Hot (ScyllaDB), Projection (PostgreSQL), Cold (S3/local)
- **Data classification:** What data lives where, why
- **Retention policies:** Configurable retention periods, defaults
- **Archival process:** CronJob schedule, archival root references
- **Data deletion:** Purge capabilities, compliance with data disposal requirements
- **Geographic considerations:** Deployment-dependent, customer-configurable

- [ ] **Step 6: Commit**

```bash
git add content/security/
git commit -m "feat: add security & compliance documentation

SOC 2 controls inventory mapped to Trust Service Criteria (CC6-CC9),
envelope encryption architecture, access control model, cryptographic
audit logging, and data residency documentation."
```

---

## Task 9: Technical Reference Section

**Files:**
- Create: `content/reference/configuration.md`
- Create: `content/reference/libraries.md`
- Create: `content/reference/data-model.md`
- Create: `content/reference/error-codes.md`

**Reference files (read-only):**
- `C:\Users\Belle\sankofa-engine\config.yaml`
- `C:\Users\Belle\sankofa-engine\go.mod`
- `C:\Users\Belle\sankofa-engine\internal\config\config.go`
- `C:\Users\Belle\sankofa-engine\internal\core\domain\domain.go`
- `C:\Users\Belle\sankofa-engine\internal\errors\`

- [ ] **Step 1: Write configuration reference**

Create `content/reference/configuration.md`:

Read the engine's `config.yaml` and `internal/config/config.go` to document every configuration field:

- **Table format:** Field path, Type, Default, Environment Variable Override, Description
- **Sections:** shard_count, nats (cluster_urls, jetstream), scylladb (endpoints, keyspace, consistency, timeout), postgresql (conn_string, pool), kms (provider, region), openbao (address, token, transit_mount), service (name, id)
- **Per-service differences:** Which config sections each microservice uses
- **Example configs:** Minimal config, production config with all options

- [ ] **Step 2: Write libraries reference**

Create `content/reference/libraries.md`:

Read `go.mod` and document all direct dependencies in a table:

| Package | Version | License | Category | Purpose |
|---------|---------|---------|----------|---------|

Group by category: HTTP, Database, Messaging, Authentication, Cryptography, Cloud, Monitoring, Testing, Utilities.

Include Go version requirement (1.25+).

- [ ] **Step 3: Write data model reference**

Create `content/reference/data-model.md`:

Read domain entity files and document:
- **Transaction:** All fields with types and descriptions. Emphasize `amount` as `string` (not float) with IEEE 754 rationale.
- **TransactionType enum:** debit, credit, transfer, exchange, mint, burn — with descriptions of when each is used
- **TransactionStatus enum:** pending, completed, failed — with lifecycle description
- **SignedReceipt:** Fields, what the signature covers
- **FungibleTokenType:** Registration fields, supply tracking
- **NFTClass / NFTInstance:** Class templates, instance fields, metadata schema
- **ShardHealth:** Health state fields
- **Key design decisions:** Amounts as strings, FNV-1a shard routing formula, idempotency key semantics

- [ ] **Step 4: Write error codes reference**

Create `content/reference/error-codes.md`:

Read error definitions from the engine source and document:
- **RFC 7807 format:** Explain the Problem Details structure (type, title, status, detail, error_codes)
- **Error code table:** Code, HTTP Status, Description, Resolution
- **Example error responses:** For common failures (validation error, auth failure, insufficient balance, duplicate idempotency key, not found)
- **Error handling best practices:** How clients should handle different error categories

- [ ] **Step 5: Commit**

```bash
git add content/reference/
git commit -m "feat: add technical reference section

Configuration schema with env var overrides, Go dependency inventory,
domain entity data model, and error code reference with RFC 7807 format."
```

---

## Task 10: Changelog

**Files:**
- Create: `content/changelog/v0.1.0-alpha.md`

- [ ] **Step 1: Write initial release notes**

Create `content/changelog/v0.1.0-alpha.md`:

```markdown
---
title: "v0.1.0-alpha"
linkTitle: "v0.1.0-alpha"
weight: 100
description: >
  Initial alpha release of the Sankofa Engine.
---

## v0.1.0-alpha

**Release date:** 2026-04-03
**Status:** Pre-release (Alpha)

### Summary

Initial alpha release of the Sankofa Engine — a sharded, privacy-preserving financial ledger engine for digital assets.

### Features

#### Core Ledger
- Sharded transaction processing with FNV-1a hash-based routing
- Block-based transaction batching with in-memory balance cache
- SHA-256 audit hash chain per account
- ECDSA P-256 signed transaction receipts
- Exactly-once processing via NATS JetStream deduplication and idempotency keys

#### Multi-Asset Support
- Fungible token registration, minting, burning, and transfer
- NFT class registration, instance minting, transfer, burn, and provenance tracking

#### Privacy & Compliance
- AES-GCM-256 envelope encryption with KMS-derived keys
- Zero-knowledge proof generation: proof-of-liabilities, proof-of-provenance, proof-of-compliance
- Proof verification endpoint
- Asset attestation support

#### API
- REST API via Fiber v3 with JWT authentication
- ECDSA P-256 transaction signing for self-custody model
- RFC 7807 Problem Details error format
- Casbin v2 RBAC authorization

#### Infrastructure
- 8 microservices: API Gateway, Shard Worker, Shard Orchestrator, Projection, Compliance, Settlement, Archival, DevCtl
- Kubernetes-native deployment with HPA, health probes, network policies
- ScyllaDB (ledger), PostgreSQL (projections), NATS JetStream (messaging), OpenBao (key management)
- Hot/cold tier archival with configurable retention policies

### Known Issues

- Settlement service is skeleton implementation — finality operations are not yet complete
- OpenBao KMS service integration is in progress
- ZKP backend is placeholder — proof generation logic is stubbed

### Dependencies

- Go 1.25+
- ScyllaDB 6.2
- PostgreSQL 16
- NATS 2.11+ with JetStream
- OpenBao (Vault fork)

### Breaking Changes

None — initial release.
```

- [ ] **Step 2: Commit**

```bash
git add content/changelog/
git commit -m "feat: add v0.1.0-alpha release notes

Initial changelog entry documenting all features, known issues,
dependencies, and breaking changes for the alpha release."
```

---

## Task 11: GitHub Actions Deployment

**Files:**
- Create: `.github/workflows/deploy-docs.yml`

- [ ] **Step 1: Create GitHub Pages deployment workflow**

Create `.github/workflows/deploy-docs.yml`:

```yaml
name: Deploy Documentation

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: "0.147.5"
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb
          sudo dpkg -i ${{ runner.temp }}/hugo.deb

      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install Node dependencies
        run: npm ci

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: "stable"

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Build with Hugo
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: production
          TZ: America/New_York
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

- [ ] **Step 2: Commit**

```bash
git add .github/
git commit -m "feat: add GitHub Actions workflow for Pages deployment

Builds Hugo site with Docsy theme and deploys to GitHub Pages on push
to main. Uses Hugo extended, Node.js for PostCSS, and Go for Hugo modules."
```

---

## Task 12: Final Build Verification and Cleanup

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Read the current README**

```bash
cat "C:/Users/Belle/Sankofa-Engine-Dev-Docs/README.md"
```

- [ ] **Step 2: Update README with project description and local dev instructions**

Update `README.md` to include:
- What this repo is (Sankofa Engine developer documentation)
- Prerequisites (Hugo extended, Go 1.25+, Node.js 20+)
- Local development: `npm install && hugo server`
- Build: `hugo --gc --minify`
- Deployment: automatic via GitHub Actions to GitHub Pages

- [ ] **Step 3: Full build verification**

Run:
```bash
cd "C:/Users/Belle/Sankofa-Engine-Dev-Docs"
npm install
hugo --gc --minify 2>&1
```

Expected: Successful build with all pages generated, no errors. Warnings about unused shortcodes are acceptable.

- [ ] **Step 4: Local server test**

Run:
```bash
cd "C:/Users/Belle/Sankofa-Engine-Dev-Docs"
hugo server --port 1313 &
sleep 3
# Verify key pages exist
curl -s -o /dev/null -w "%{http_code}" http://localhost:1313/
curl -s -o /dev/null -w "%{http_code}" http://localhost:1313/overview/
curl -s -o /dev/null -w "%{http_code}" http://localhost:1313/api-reference/
curl -s -o /dev/null -w "%{http_code}" http://localhost:1313/api-reference/swagger/
curl -s -o /dev/null -w "%{http_code}" http://localhost:1313/security/
curl -s -o /dev/null -w "%{http_code}" http://localhost:1313/changelog/
kill %1
```

Expected: All return 200.

- [ ] **Step 5: Commit**

```bash
git add README.md
git commit -m "docs: update README with local development instructions"
```

- [ ] **Step 6: Final commit — remove any Hugo-generated scaffolding files not needed**

Check for and remove any auto-generated files from `hugo new site` that aren't needed (e.g., default `archetypes/default.md` if it exists):

```bash
cd "C:/Users/Belle/Sankofa-Engine-Dev-Docs"
ls archetypes/ 2>/dev/null && git rm archetypes/default.md 2>/dev/null
git status
```

If there are files to clean up:
```bash
git commit -m "chore: remove unused Hugo scaffolding files"
```
