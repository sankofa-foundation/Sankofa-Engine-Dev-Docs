---
title: "Libraries & Dependencies"
linkTitle: "Libraries"
weight: 2
description: >
  Go module dependencies, versions, and their purposes.
---

The Sankofa Engine is written in Go and requires **Go 1.25 or later**. Below are all direct dependencies from `go.mod`, grouped by category.

## HTTP Framework

| Package | Version | Purpose |
|---|---|---|
| `github.com/gofiber/fiber/v3` | v3.1.0 | HTTP framework for the REST API |

## Database

| Package | Version | Purpose |
|---|---|---|
| `github.com/gocql/gocql` | v1.7.0 | ScyllaDB/Cassandra driver |
| `github.com/jackc/pgx/v5` | v5.9.1 | PostgreSQL driver with connection pooling |

## Messaging

| Package | Version | Purpose |
|---|---|---|
| `github.com/nats-io/nats.go` | v1.49.0 | NATS client for pub/sub and RPC |
| `github.com/nats-io/nats-server/v2` | v2.12.6 | Embedded NATS server for testing |

## Authentication and Authorization

| Package | Version | Purpose |
|---|---|---|
| `github.com/golang-jwt/jwt/v5` | v5.3.1 | JWT token generation and validation |
| `github.com/casbin/casbin/v2` | v2.135.0 | Role-based access control (RBAC) |

## Cryptography

| Package | Version | Purpose |
|---|---|---|
| `golang.org/x/crypto` | v0.49.0 | Cryptographic primitives |

## Cloud Services

| Package | Version | Purpose |
|---|---|---|
| `github.com/aws/aws-sdk-go-v2/config` | v1.29.5 | AWS SDK configuration |
| `github.com/aws/aws-sdk-go-v2/service/kms` | v1.43.0 | AWS KMS for key management |

## Utilities

| Package | Version | Purpose |
|---|---|---|
| `github.com/google/uuid` | v1.6.0 | UUID generation |
| `github.com/shirou/gopsutil/v3` | v3.24.5 | System metrics collection |
| `golang.org/x/time` | v0.15.0 | Rate limiting |
| `gopkg.in/yaml.v3` | v3.0.1 | YAML configuration parsing |

## Testing

| Package | Version | Purpose |
|---|---|---|
| `github.com/stretchr/testify` | v1.11.1 | Testing assertions and mocks |

## Dependency Management

The project uses Go modules (`go.mod` / `go.sum`) for dependency management. To update dependencies:

```bash
# Update all dependencies
go get -u ./...
go mod tidy

# Update a specific dependency
go get -u github.com/gofiber/fiber/v3@latest
go mod tidy
```

{{% alert title="Note" color="info" %}}
The embedded NATS server (`nats-server/v2`) is used exclusively in integration tests to provide a self-contained test environment. It is not used in production deployments.
{{% /alert %}}
