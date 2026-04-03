---
title: "Access Control"
linkTitle: "Access Control"
weight: 3
description: >
  Authentication, authorization, and infrastructure access controls.
---

The Sankofa Engine enforces access control at three layers: application authentication, policy-based authorization, and infrastructure-level isolation. This page documents each layer.

## Authentication

### API Key Provisioning

API keys are provisioned by Sankofa Labs during customer onboarding. Customers do not self-register or self-provision credentials.

| Step | Description |
|------|-------------|
| 1. Onboarding request | Customer requests access through Sankofa Labs |
| 2. Identity verification | Sankofa Labs verifies the customer's identity and authorization |
| 3. Key generation | A unique `client_id` and `client_secret` pair is generated |
| 4. Secure delivery | Credentials are delivered through a secure channel |
| 5. Activation | Credentials are activated in the target deployment |

### JWT Token Exchange

Clients authenticate by exchanging their API key credentials for a JWT bearer token:

```text
Client                          API Gateway
  │                                  │
  │  POST /auth/token                │
  │  { client_id, client_secret }    │
  │ ────────────────────────────────▶│
  │                                  │  Validate credentials
  │                                  │  Generate JWT
  │  200 OK                          │
  │  { access_token, expires_in }    │
  │ ◀────────────────────────────────│
  │                                  │
  │  GET /api/v1/accounts            │
  │  Authorization: Bearer <JWT>     │
  │ ────────────────────────────────▶│
  │                                  │  Validate JWT signature
  │                                  │  Extract claims
  │                                  │  Enforce authorization
  │  200 OK                          │
  │ ◀────────────────────────────────│
```

### Token Lifetime and Refresh

| Parameter | Description |
|-----------|-------------|
| Token lifetime | Configurable per deployment (e.g., 15 minutes, 1 hour) |
| Refresh model | Clients request a new token using their `client_id` and `client_secret` when the current token expires |
| Revocation | Tokens can be revoked immediately by invalidating the signing key or adding the token to a deny list |

Short-lived tokens limit the window of exposure if a token is compromised.

### ECDSA P-256 Transaction Signing (Self-Custody)

For customers who require self-custody of transaction authorization, the Sankofa Engine supports ECDSA P-256 digital signatures:

| Aspect | Detail |
|--------|--------|
| Algorithm | ECDSA with P-256 curve (NIST) |
| Key ownership | Customer generates and holds the private key |
| Signing | Customer signs transaction payloads before submission |
| Verification | The engine verifies the signature against the customer's registered public key |
| Non-repudiation | Signed transactions provide cryptographic proof of authorization |

This model gives customers full control over transaction authorization without sharing private keys with Sankofa Labs.

## Authorization

### Casbin v2 RBAC Model

The Sankofa Engine uses [Casbin v2](https://casbin.org/) for role-based access control. Casbin evaluates every API request against a policy file that defines who can do what on which resources.

### Policy Structure

Casbin policies follow a **subject, object, action** model:

```text
p, role:admin,    /api/v1/accounts/*,   *
p, role:operator, /api/v1/accounts/*,   GET
p, role:operator, /api/v1/transactions, POST
p, role:auditor,  /api/v1/accounts/*,   GET
p, role:auditor,  /api/v1/audit/*,      GET
```

| Element | Description |
|---------|-------------|
| **Subject** | The role or identity making the request (extracted from the JWT claims) |
| **Object** | The API resource being accessed (URL path) |
| **Action** | The HTTP method (GET, POST, PUT, DELETE) |

### Built-in Roles and Permissions

| Role | Permissions | Intended Use |
|------|------------|--------------|
| `admin` | Full access to all API endpoints | Platform administrators at the customer organization |
| `operator` | Read access to accounts; create transactions | Day-to-day operational use |
| `auditor` | Read-only access to accounts and audit endpoints | Compliance and audit personnel |

Custom roles can be defined in the Casbin policy file to match customer-specific requirements.

### Permission Enforcement

Authorization is enforced at the API Gateway middleware layer:

```text
Incoming Request
       │
       ▼
┌─────────────────┐
│ JWT Validation   │  Verify token signature and expiration
└───────┬─────────┘
        │
        ▼
┌─────────────────┐
│ Claims Extraction│  Extract role, subject, and tenant from JWT
└───────┬─────────┘
        │
        ▼
┌─────────────────┐
│ Casbin Enforcer  │  Evaluate (subject, object, action) against policy
└───────┬─────────┘
        │
   ┌────┴────┐
   │         │
 Allow     Deny
   │         │
   ▼         ▼
 Route    403 Forbidden
 to        + audit log
 backend
```

Every authorization decision — both allow and deny — is logged for audit purposes.

## Infrastructure Access Controls

### Kubernetes Namespace Isolation

All Sankofa Engine components run in a dedicated `sankofa-engine` Kubernetes namespace:

| Control | Implementation |
|---------|---------------|
| Namespace boundary | All pods, services, configmaps, and secrets are scoped to `sankofa-engine` |
| RBAC | Kubernetes RBAC roles and role-bindings restrict who can access resources in the namespace |
| Resource quotas | CPU and memory quotas prevent resource exhaustion |
| No cross-namespace access | Services in other namespaces cannot access Sankofa Engine resources |

### Network Policies

Kubernetes NetworkPolicies restrict pod-to-pod communication to only the paths required by the architecture:

| Source | Destination | Allowed |
|--------|-------------|---------|
| API Gateway | NATS JetStream | Yes |
| Shard Workers | NATS JetStream | Yes |
| Shard Workers | ScyllaDB | Yes |
| API Gateway | PostgreSQL | Yes |
| All services | OpenBao | Yes |
| Any other path | Any | **Denied** |

Default-deny policies ensure that any new pod added to the namespace has no network access until an explicit policy is created.

### Secret Scoping

Secrets are scoped to individual services following the principle of least privilege:

| Principle | Implementation |
|-----------|---------------|
| Per-service secrets | Each service has its own set of secrets (database credentials, KMS tokens, TLS certificates) |
| No shared credentials | No two services share the same secret |
| Runtime injection | Secrets are injected via OpenBao agent at pod startup, not stored in Kubernetes Secrets |
| No environment variables | Secrets are mounted as files, not exposed as environment variables (which can leak via process listings) |

### Principle of Least Privilege

Every component in the Sankofa Engine operates with the minimum permissions required:

- **Services** authenticate to only the backends they need (e.g., shard workers access ScyllaDB but not PostgreSQL).
- **KMS access** is scoped per service — each service can only access the key paths it requires.
- **Kubernetes RBAC** grants only the verbs (get, list, watch) each service account needs.
- **Network policies** allow only the specific ingress and egress paths required by the architecture.
