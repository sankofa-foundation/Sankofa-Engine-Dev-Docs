---
title: "Configuration"
linkTitle: "Configuration"
weight: 1
description: >
  Configuration schema, environment variable overrides, and example configurations.
---

## Configuration Schema

The Sankofa Engine is configured via a YAML file (default path: `config.yaml`). Below is the complete schema with all supported fields.

### Default Configuration

```yaml
shard_count: 4

nats:
  cluster_urls:
    - "nats://localhost:4222"
  jetstream:
    max_message_age_seconds: 220898160  # 7 years

scylladb:
  endpoints:
    - "localhost:9042"
  keyspace: "sankofa_engine"
  read_consistency: "ONE"
  request_timeout: "30s"

kms:
  provider: "local"
  region: "us-east-1"

openbao:
  address: "http://localhost:8200"
  token: "dev-root-token"
  transit_mount: "transit"

service:
  name: "monolith"
  id: ""
```

### Field Reference

| Field Path | Type | Default | Description |
|---|---|---|---|
| `shard_count` | integer | `4` | Number of shards for partitioning transactions across workers. Determines the modulus for FNV-1a shard routing. |
| `nats.cluster_urls` | []string | `["nats://localhost:4222"]` | List of NATS server URLs for the client to connect to. Supports multiple URLs for cluster failover. |
| `nats.jetstream.max_message_age_seconds` | integer | `220898160` (7 years) | Maximum age in seconds before JetStream messages are expired. Set to 7 years for long-term audit retention. |
| `scylladb.endpoints` | []string | `["localhost:9042"]` | ScyllaDB/Cassandra contact points. The driver discovers additional nodes from these seeds. |
| `scylladb.keyspace` | string | `"sankofa_engine"` | ScyllaDB keyspace name where all tables reside. |
| `scylladb.read_consistency` | string | `"ONE"` | Read consistency level. Common values: `ONE`, `QUORUM`, `LOCAL_QUORUM`. |
| `scylladb.request_timeout` | string | `"30s"` | Timeout for individual ScyllaDB requests. Go duration format (e.g., `30s`, `1m`). |
| `kms.provider` | string | `"local"` | Key management provider. `"local"` for development, `"aws"` for AWS KMS in production. |
| `kms.region` | string | `"us-east-1"` | Cloud provider region for the KMS service. |
| `openbao.address` | string | `"http://localhost:8200"` | OpenBao (Vault-compatible) server address for transit encryption. |
| `openbao.token` | string | `"dev-root-token"` | Authentication token for OpenBao. Use a provisioned secret in production. |
| `openbao.transit_mount` | string | `"transit"` | Mount path for the OpenBao transit secrets engine. |
| `service.name` | string | `"monolith"` | Service identity. Used for logging, metrics, and NATS subject routing. Values: `monolith`, `api`, `shard-worker`. |
| `service.id` | string | `""` | Unique instance identifier. Used to distinguish multiple instances of the same service. |

## Environment Variable Overrides

Environment variables take precedence over values in the config file. Use these to configure the engine in containerized or orchestrated environments.

| Environment Variable | Config Field Override | Example |
|---|---|---|
| `CONFIG_PATH` | Config file path | `/etc/sankofa/config.yaml` |
| `HTTP_PORT` | API listen port | `8080` |
| `NATS_CLUSTER_URLS` | `nats.cluster_urls` | `nats://nats-1:4222,nats://nats-2:4222` |
| `SCYLLADB_ENDPOINTS` | `scylladb.endpoints` | `scylla-1:9042,scylla-2:9042` |
| `POSTGRESQL_CONN_STRING` | PostgreSQL connection | `postgres://user:pass@host:5432/sankofa?sslmode=require` |
| `OPENBAO_ADDRESS` | `openbao.address` | `https://vault.internal:8200` |
| `OPENBAO_TOKEN` | `openbao.token` | (provisioned secret) |
| `SERVICE_NAME` | `service.name` | `api`, `shard-worker`, etc. |
| `WORKER_ID` | Shard worker instance ID | `worker-0` |
| `SHARD_MAP_BUCKET` | NATS KV bucket name | `shard-map` |

{{% alert title="Note" color="info" %}}
Comma-separated environment variables (`NATS_CLUSTER_URLS`, `SCYLLADB_ENDPOINTS`) are split into string arrays at runtime.
{{% /alert %}}

## Service Configuration Matrix

Different services use different subsets of the configuration. The table below shows which config sections each service requires.

| Config Section | `monolith` | `api` | `shard-worker` |
|---|:---:|:---:|:---:|
| `shard_count` | Yes | Yes | Yes |
| `nats` | Yes | Yes | Yes |
| `scylladb` | Yes | No | Yes |
| `kms` | Yes | No | Yes |
| `openbao` | Yes | No | Yes |
| `service` | Yes | Yes | Yes |

- **monolith** -- Runs all components in a single process. Requires the full configuration.
- **api** -- Stateless HTTP gateway. Needs NATS to forward requests to shard workers but does not access ScyllaDB or KMS directly.
- **shard-worker** -- Processes transactions for assigned shards. Requires database, encryption, and messaging configuration.

## Example: Production Configuration

```yaml
shard_count: 16

nats:
  cluster_urls:
    - "nats://nats-1.internal:4222"
    - "nats://nats-2.internal:4222"
    - "nats://nats-3.internal:4222"
  jetstream:
    max_message_age_seconds: 220898160

scylladb:
  endpoints:
    - "scylla-1.internal:9042"
    - "scylla-2.internal:9042"
    - "scylla-3.internal:9042"
  keyspace: "sankofa_engine"
  read_consistency: "LOCAL_QUORUM"
  request_timeout: "10s"

kms:
  provider: "aws"
  region: "us-east-1"

openbao:
  address: "https://vault.internal:8200"
  token: ""  # Set via OPENBAO_TOKEN env var
  transit_mount: "transit"

service:
  name: "shard-worker"
  id: ""  # Set via WORKER_ID env var
```
