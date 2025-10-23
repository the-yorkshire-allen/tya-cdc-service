# CDC Sink Runtime ↔ Datastore Adapter Contract

- **Contract Version**: 0.1.0
- **Last Updated**: October 21, 2025
- **Maintainers**: CDC Runtime Team


## Purpose
Define the language-agnostic interface between the shared CDC sink runtime and datastore-specific adapters (Postgres, MySQL, Redis). This contract enables adapters to be implemented in any language while preserving consistent lifecycle, security, and operational guarantees.

## Transport & Deployment
- Adapters run as separate processes or containers alongside the sink runtime.
- Communication occurs over mTLS-secured gRPC or HTTP/2.
- Each adapter registers its endpoint with the runtime during bootstrap using the provisioning token issued via the sink config.
- The runtime and adapter read a shared sink configuration file for network, credential, and trust settings; adapters may optionally load a separate schema-translation configuration referenced in that file.

## Authentication & Trust
- Runtime presents its mTLS client cert issued during sink bootstrap.
- Adapters validate runtime identity using the trust bundle supplied in the sink config.
- Adapters must expose a `Status` endpoint to confirm trust and readiness.

## Core RPCs / Endpoints

### 1. `ApplyChange`
- **Request**
  - `event_id`: UUID from the CDC outbox.
  - `sequence_token`: Monotonic ordering token.
  - `table`: Source table name.
  - `operation`: `INSERT` | `UPDATE` | `DELETE`.
  - `payload`: JSON or Avro-encoded row data.
  - `metadata`: Map including capture timestamp, TTL, retention class.
- **Response**
  - `status`: `ACK` | `RETRYABLE_ERROR` | `FATAL`.
  - `error_detail`: Optional message for non-ACK responses.
- **Semantics**
  - Must be idempotent; runtime may retry on transient failures.
  - Adapter commits to the target datastore before acknowledging.

### 2. `ApplySweep`
- **Request**
  - `sweep_id`: Identifier for the sweep manifest.
  - `manifest_uri`: Location of snapshot data (S3 or pre-signed URL).
  - `schema_version`: Snapshot schema version.
  - `scope`: Tables/keys included in the sweep.
- **Response**
  - `status`: `ACK` | `RETRYABLE_ERROR` | `FATAL`.
  - `progress_token`: Optional to support resumable sweeps.
- **Semantics**
  - Adapters must load data transactionally and signal completion.
  - Runtime pauses incremental delivery for affected tables until sweep completes.

### 3. `AckHeartbeat`
- Lightweight health ping from runtime; adapter returns current status (`HEALTHY`, `DEGRADED`, `ERROR`) plus sink-specific metrics.

### 4. `RotateCredentials`
- Runtime sends updated cert paths and refresh tokens.
- Adapter must reload credentials without downtime and confirm update.

### 5. `SchemaAnnouncement`
- **Request**
  - `schema_version`: Identifier of the CDC schema snapshot.
  - `tables`: Array of table descriptors (name, primary key, column definitions, column types).
  - `change_types`: Supported operations (`INSERT`, `UPDATE`, `DELETE`) per table.
- **Response**
  - `status`: `ACK` | `RETRYABLE_ERROR` | `FATAL`.
  - `error_detail`: Optional details for mismatches.
- **Semantics**
  - Runtime invokes after bootstrap and whenever schema changes.
  - Adapter validates compatibility with its schema-translation configuration and may return `RETRYABLE_ERROR` while reloading translation metadata.

### 6. `LogBatch`
- **Request**
  - `batch_id`: UUID for idempotency.
  - `records`: Array of log entries {timestamp, level, component, message, context}.
  - `config_version`: Effective log configuration version received from runtime.
- **Response**
  - `status`: `ACK` | `RETRYABLE_ERROR` | `FATAL`.
  - `retry_after`: Optional milliseconds for back-off.
- **Semantics**
  - Adapter sends whenever local buffer reaches threshold or on flush command from runtime.
  - Runtime persists logs and may respond with updated log configuration identifiers.

### 7. `LogControl`
- **Request**
  - `config_version`: Requested configuration version.
  - `log_level`: Desired minimum level (`TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`).
  - `sampling_rate`: Optional sampling percentage for `INFO`/`DEBUG`.
  - `filters`: Optional suppression patterns.
- **Response**
  - `status`: `ACK` | `REJECTED`.
  - `error_detail`: Reason for rejection.
- **Semantics**
  - Runtime invokes when CDC operators update log settings.
  - Adapter applies configuration atomically and acknowledges; `REJECTED` indicates unsupported combination.

## Error Handling
- `RETRYABLE_ERROR` triggers exponential backoff (configurable via runtime).
- `FATAL` marks the event/sweep as failed; runtime escalates to operators.
- Adapters must surface structured errors with codes: `CONNECTION`, `SCHEMA`, `CONSTRAINT`, `TIMEOUT`, `INTERNAL`.

## Observability
- Adapters emit structured logs with event IDs and sweep IDs.
- Metrics exposed via Prometheus endpoint: `adapter_apply_latency`, `adapter_errors_total{code}`, `adapter_sweep_progress`.
- Runtime scrapes metrics and correlates with sink ID.

## Configuration Inputs
- Shared sink configuration file supplies datastore connection strings, retry thresholds, batching size, schema-translation config path, and telemetry defaults (log level, sampling).
- Optional adapter-specific schema file defines column mappings or transformation rules; referenced via `bootstrap.adapter_schema.schema_config_path`.
- Runtime passes global policy (retention, grace periods), current CDC schema metadata, and telemetry overrides via `AdapterInit`, `SchemaAnnouncement`, or `LogControl` calls.

## Versioning
- Contract version header `x-adapter-contract-version` must be exchanged on every call.
- Breaking changes require spec update and coordinated rollout.

## Security Requirements
- No plaintext credentials in transit; all secrets sourced from secrets manager paths defined in sink config.
- Adapters must wipe in-memory tokens after rotation.
- Untrusted TLS certificates must be rejected; runtime will not disable verification.

