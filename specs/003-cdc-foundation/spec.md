# TYA CDC Service Foundation Specification

- **Spec Version**: 0.3.0
- **Last Updated**: October 21, 2025
- **Maintainers**: CDC Platform Guild


## Context & Goals
- Deliver a resilient change-data-capture (CDC) platform that replicates Cassandra keyspace changes to authorised downstream sinks.
- Provide a governed onboarding process for sinks, including credentialing, access scope, and retention policies.
- Guarantee at-least-once delivery semantics with observability that allows operators to verify freshness, backlog, and health.
- Support both incremental CDC streaming and operator-triggered full sweeps for reconciliation.
- Success looks like: ≥95% sinks meeting class-specific lag targets, zero unacknowledged events aging past retention without incident, and disaster recovery from latest checkpoint within 30 minutes.

## Out of Scope
- Building bespoke sink-specific transformation pipelines beyond generic payload shaping outside of the standard Postgres, MySQL, and Redis onboarding kits.
- Managing downstream sink infrastructure or data warehousing processes.
- Providing a user-facing UI dashboard in the initial release (CLI and API only).
- Supporting databases other than Apache Cassandra as the source of truth.

## Stakeholders & Personas
- Data Platform Owner — sponsors roadmap, approves governance decisions.
- Sink Operators — register sinks, manage credentials, consume data.
- Platform Operations & SRE — ensure Cassandra availability, respond to incidents.
- Security & Compliance — oversee credential governance, audit trails, data classifications.
- QA & Analytics Consumers — validate datasets, ensure downstream adoption readiness.

## Domain Concepts
- Actors: sink operator, platform operator, security officer, observability system.
- Data Objects: sink profile, CDC event, outbox entry, sweep manifest, checkpoint, audit trail.
- Operations: registration, credential issuance/rotation, CDC capture, delivery fan-out, acknowledgment tracking, sweep initiation, replay, pause/resume.
- Policies: retention (PII vs non-PII), tiered freshness classes, ServiceNow alert routing, retry/maintenance windows.

## User Scenarios & Acceptance Testing
1. **Sink Registration** — Given a sink operator submits the required metadata, when the service validates the request, then the sink receives credentials and scoped access confirmation.
2. **CDC Capture & Delivery** — Given row-level changes occur in a tracked table, when the CDC capture process runs, then the change is persisted in the outbox and delivered to each eligible sink at least once.
3. **Operator Sweep** — Given an authorised operator triggers a full sweep, when the sweep completes, then the sink receives a transactionally consistent snapshot for its authorised keyspaces.
4. **Retention Breach Handling** — Given a sink fails to acknowledge delivered events before the retention window expires, when the window is exceeded, then the service flags the sink for operator remediation and preserves the data until action is taken.
5. **Observability Metrics** — Given observability tooling queries the metrics endpoint, when the service is healthy, then throughput, backlog, and error rates are reported accurately per sink.
6. **Schema Change Response** — Given a tracked table schema changes, when the service detects the change, then operators are notified and sinks receive compatible metadata before delivery resumes.

### Edge Cases & Open Questions
- What governance process is required when a sink requests broader access scope post-registration? `[Owner: Data Platform Owner]`
- Should sinks be able to request replays for a bounded time window without full sweeps? `[Owner: Platform Operations + Security]`
- How are malicious or compromised credentials handled mid-flight to avoid partial deliveries? `[Owner: Security]`

### Policy Decisions
- **Capture retry window**: Exponential retries for up to 10 minutes when Cassandra nodes are unreachable; after that the service enters maintenance hold pending operator acknowledgment.
- **CDC freshness, sweep latency & backlog thresholds**: Tiered sink classes — Gold (CDC lag ≤3s p95, sweep ≤45m, backlog alert ≥5k events, recovery RTO ≤15m), Silver (lag ≤10s p95, sweep ≤3h, backlog alert ≥20k events, recovery RTO ≤30m), Bronze (lag ≤60s p95, sweep ≤12h, backlog alert ≥50k events, recovery RTO ≤60m); class assignment occurs at registration and drives alert thresholds.
- **Delivery acknowledgements & heartbeats**: Gold sinks must acknowledge deliveries within 60s with 15s heartbeat cadence; Silver within 120s with 30s heartbeat cadence; Bronze within 300s with 60s heartbeat cadence. Missing three consecutive heartbeats for any tier triggers a ServiceNow incident.
- **SLA configuration management**: SLA parameters and retention windows are externalised in `config/core.yaml`; updates are audited and surfaced through metrics/health endpoints.
- **Undelivered event retention**: Retain PII-bearing datasets for 72 hours and non-PII datasets for 14 days, enforced per sink.
- **Operator alerting**: Integrate with enterprise ServiceNow event bus for backlog, retention, and node outage incidents.
- **Sink runtime architecture**: Provide a shared CDC sink runtime responsible for registration, credential exchange, trust uploads, sweep orchestration, acknowledgments, and metrics, with datastore-specific adapters (Postgres/MySQL/Redis) delegated to a defined interface for change application and sweep reconciliation. When no datastore adapter is configured, the runtime emits CDC events and sweeps to local files using settings from `config/sinks/default.yaml` (output path, naming template, rotation thresholds).
- **Sink configuration governance**: The sink runtime and datastore adapters consume the same sink configuration file for security, networking, and trust material; adapters may load an additional schema-translation config specific to their datastore.
- **Schema synchronisation**: The sink runtime retrieves expected CDC schemas from the CDC service before delivery and on schema change notifications, distributing metadata to adapters or the file fallback so transformations remain consistent.
- **Data residency & reconciliation**: All sink telemetry logs, credential artifacts (certs, trust bundles), refresh/access tokens, and sink registrations persist in the CDC Cassandra keyspace using dedicated tables with configured TTLs (per dataset). CDC service nodes periodically reconcile the keyspace for new sinks or credential updates so any node can assume delivery duties.
- **Telemetry governance**: Sinks push structured logs and diagnostics to the CDC service through an authenticated telemetry API; CDC operators can remotely adjust sink log levels and sampling policies without redeploying sinks.
- **Sink scope management**: Authorised APIs maintain per-sink data scope definitions (keyspaces, tables, filters) stored in Cassandra so delivery responsibilities are inspectable and editable without redeploying sinks.
- **Compromise response**: When credentials are suspected compromised, the sink enters `compromise` state, deliveries halt, ServiceNow alerts fire, tokens and certs are revoked/rotated, forensic data is preserved, scopes reviewed, and remediation steps audited before resuming traffic.
- **Sink credential lifecycle**: Each sink maintains a configuration file that declares certificate locations; if mTLS material is absent, bootstrap tooling provisions a new cert/key pair from the CDC service-managed CA and persists paths back into the config. Sinks can optionally supply enterprise PKI or locally generated certificates so long as the trust bundle is shared. After obtaining a short-lived provisioning token—lifetime and scope defined in `config/sinks/default.yaml`—the bootstrap tooling POSTs the trust bundle to the CDC trust-registration API, which keeps outbound delivery in a `not_trusted` state until the bundle is accepted; status is returned in the API response so sinks avoid setting insecure TLS flags. Sinks also receive an mTLS client identity delivered through the secrets-manager handoff at registration, a revocable refresh token for calling CDC APIs, and must supply the CDC service with a dedicated ingest refresh token and mTLS trust bundle for outbound deliveries. The ingest refresh token is issued during registration via the secrets manager handoff workflow and re-issued on every coordinated rotation. Refresh tokens mint signed access tokens with ≤5 minute TTL; rotating either refresh token requires rotating the paired direction so the CDC↔sink credentials remain in lockstep, invalidates outstanding tokens within 60 seconds, and records every issuance in the audit log.
- **Checkpoint storage & recovery**: Persist offsets and sweep manifests in Cassandra control keyspace and replicate encrypted snapshots to S3 for disaster recovery.
- **Configuration topology**: Service configuration is split into `config/core.yaml` (cluster scheduling, reconciliation cadence, shard counts), `config/security.yaml` (credential retention, secrets-manager paths, certificate TTLs), `config/telemetry.yaml` (log defaults, sampling, ingestion limits, TTLs), `config/replay.yaml` (replay windows, approval roles, throttle budgets, retention), and optional per-sink overrides in `config/sinks/{sink_id}.yaml` extending the shared sink configuration defaults.
- **Replay governance**: Replay windows, approval requirements, concurrency limits, throttling budgets, and audit retention are centrally defined in `config/replay.yaml`; the CDC service enforces these limits before issuing replays and records every decision in Cassandra.
- **Bootstrap configuration schema**: `config/sinks/default.yaml` defines `bootstrap.provisioning_token` (ttl_seconds, max_uses, scope, issuer), `bootstrap.trust_registration` (endpoint, retry_backoff, status_poll_interval), `bootstrap.ca` (default_ca, allow_external_ca, approved_external_cas), `bootstrap.sink_config_defaults` (cert_path, key_path, trust_bundle_path, file_mode, owner_group), `secrets_manager` (sink_refresh_secret_path_pattern, cdc_ingest_secret_path_pattern, rotation_window_seconds), `security.audit` (enabled, sink_event_topic, retention_days), and `delivery.block_on_untrusted` with optional grace_period_seconds; per-sink overrides live under `sinks.<sink_id>.bootstrap.*`, and `bootstrap.local_fallback` (enabled, filepath, filename_pattern, rotation_bytes, rotation_interval) plus `bootstrap.adapter_schema` (schema_config_path, translation_mode) and `bootstrap.telemetry` (initial_log_level, sampling_rate, endpoint_override, ttl_seconds).

## Functional Requirements
- **FR-001** Authenticated API for sink registration including identity, contacts, scope, retention, and secrets-manager exchange for the CDC ingest refresh token set.
- **FR-002** Issue/manage unique credentials per sink with revocation/rotation without downtime, covering both CDC API access and sink ingest delivery via refresh-token backed short-lived access tokens.
- **FR-003** Ingest Cassandra CDC logs for configured tables and persist events with ordering metadata suitable for re-delivery.
- **FR-004** Deliver CDC events to each authorised sink, tracking acknowledgments and retries until confirmation or retention expiry, with reference implementations for Postgres, MySQL, and Redis sinks.
- **FR-005** Support operator-triggered full sweeps that produce consistent snapshots aligned with sink scope and label them with sweep identifiers.
- **FR-006** Maintain auditable outbox recording event status (pending, delivered, failed, expired) per sink.
- **FR-007** Expose operational APIs for pausing, resuming, or deactivating sinks with reason codes and audit trails.
- **FR-008** Surface health/readiness endpoints with node status, last successful capture timestamp, and delivery backlog overview.
- **FR-009** Provide structured metrics (Prometheus) for capture lag, backlog size, throughput, error counts, sweep duration per sink.
- **FR-010** Detect incompatible schema changes and halt deliveries until resolution.
- **FR-011** Log lifecycle events (registration updates, credential issues, sweeps, delivery failures) with traceable identifiers.
- **FR-012** Support configurable retry policies for capture and delivery stages with jittered backoff and escalation hooks.
- **FR-013** Provide replay mechanism for time-bounded re-delivery without full sweep, respecting authorisation and replay policies.
- **FR-014** Integrate with ServiceNow event bus for backlog thresholds, retention breaches, node outages.
- **FR-015** Provide sink bootstrap tooling that writes the local configuration file, provisions or validates mTLS material, uploads trust bundles via the CDC trust-registration API, persists refresh-token credentials via the secrets-manager handoff, and ships adapters for Postgres, MySQL, and Redis targets.
- **FR-016** Expose a CDC trust-registration API that authenticates sinks, accepts trust bundles, returns trust status (`pending`, `trusted`, `rejected`), enforces provisioning token policies defined in `config/sinks/default.yaml`, and blocks outbound deliveries until trust is established.
- **FR-017** Define a language-agnostic adapter interface (change events, sweep manifests, health callbacks) so datastore adapters can be implemented in any language and hosted alongside the shared runtime via gRPC/HTTP contracts. The runtime must provide a default file-sink implementation honouring the local fallback configuration when no adapter is registered.
- **FR-018** Allow the sink runtime to request expected CDC schemas from the CDC service via an authenticated API and propagate schema metadata to adapters alongside their schema-translation configuration.
- **FR-019** Provide a telemetry API that ingests sink log batches, honours CDC-controlled log levels, and exposes operator endpoints to query recent sink diagnostics.
- **FR-020** Persist sink telemetry, credentials, certificates, trust bundles, and token artifacts in dedicated Cassandra tables within the CDC keyspace, and run reconciliation jobs that detect new sinks or credential changes to keep all CDC nodes consistent.
- **FR-021** Offer sink scope management APIs that allow operators to read, update, and version the data scope each sink should receive, with changes persisted in Cassandra and propagated to runtimes.
- **FR-022** Enforce replay policies defined in configuration files, requiring approvals, verifying capacity limits, and auditing replay executions in Cassandra.
- **FR-023** Provide a credential compromise workflow that freezes deliveries, revokes tokens/certs, preserves observability data, requires scope revalidation, and supports controlled replays prior to resuming service.

### Sink Telemetry API
- **Endpoint**: `POST /api/v1/sinks/{sink_id}/logs` accepts structured log batches (JSON lines) signed by the sink runtime; payload includes log level, timestamp, event_id, adapter component, and optional stack trace.
- **Endpoint**: `GET /api/v1/sinks/{sink_id}/log-config` returns the current effective log level, sampling policy, and suppression filters.
- **Endpoint**: `POST /api/v1/sinks/{sink_id}/log-config` (operator-only) updates sink log level, sampling rate, or filter rules; responses include versioned config identifiers propagated to sinks and replicated across CDC nodes.
- **Authentication**: POST log batches require sink access tokens; config updates require operator credentials with `log_admin` scope.
- **Delivery Guarantees**: Logs are accepted with at-least-once semantics; CDC service de-duplicates via batch and record identifiers. Responses include back-pressure hints (`retry_after`) when ingestion is throttled.
- **Retention & Querying**: CDC service writes sink logs into the `cdc.telemetry_logs` Cassandra table with configurable TTL (default 30 days) and streams them to the observability lake; operators query recent logs via observability tooling, not this API.
- **Error Handling**: `413` for oversized batches, `429` for throttling, `409` when config version conflicts, `403` for insufficient scope.

### Sink Scope API
- **Endpoint**: `GET /api/v1/sinks/{sink_id}/scope` returns the current scope definition (keyspaces, tables, filters, sweep eligibility) stored in Cassandra.
- **Endpoint**: `PUT /api/v1/sinks/{sink_id}/scope` updates the scope with optimistic concurrency via `If-Match` header on the scope version; accepts JSON describing keyspace/table patterns and optional column filters.
- **Endpoint**: `GET /api/v1/sinks/{sink_id}/scope/history` lists prior versions for audit with pagination.
- **Authentication**: Operator credentials with `scope_admin` role; sinks receive read-only access tokens to fetch their scope for diagnostics.
- **Propagation**: Updates emit change events so sink runtimes reload scope and reconcile Cassandra tables before next delivery.
- **Error Handling**: `409` on version conflict, `422` for invalid scope definitions, `423` when sink paused, `404` if sink not found.

### Replay API
- **Endpoint**: `POST /api/v1/sinks/{sink_id}/replays` submits a replay request with payload `{start_ts, end_ts, tables, justification}`; validates against `config/replay.yaml`, requires approval claims, and returns a `replay_id`.
- **Endpoint**: `GET /api/v1/sinks/{sink_id}/replays/{replay_id}` returns replay status (`pending_approval`, `queued`, `running`, `completed`, `denied`) plus metrics (events queued, remaining).
- **Endpoint**: `POST /api/v1/sinks/{sink_id}/replays/{replay_id}/approve` allows authorised approvers to approve/deny requests; dual approval enforced when replay exceeds default window or touches PII datasets.
- **Endpoint**: `DELETE /api/v1/sinks/{sink_id}/replays/{replay_id}` cancels queued/running replays, ensuring reconciliation with outbox state.
- **Authentication**: Replay submission requires operator tokens with `replay_request` scope; approvals demand `replay_approver` role; audit records captured for all actions.
- **Execution Semantics**: Approved replays re-publish events to the shared runtime using the same ordering/dedup guarantees, respecting throughput budgets and injecting identifiers so sinks can reconcile.
- **Observability**: Metrics (`replay_events_total`, `replay_duration_seconds`, `replay_denied_total`) and ServiceNow notifications are emitted for start, completion, denial.
- **Error Handling**: `409` on overlapping replay windows, `429` when capacity threshold exceeded, `403` without proper roles, `410` when replay already completed/expired.

### Credential Compromise Workflow
- **Detection**: Security signals or runtime anomalies transition sink state to `compromise`, raising ServiceNow alerts and pausing deliveries/replays automatically.
- **Revocation**: CDC service revokes refresh/access tokens and ingest credentials, issues new ones via secrets manager, and rotates mTLS certificates/trust bundles stored in Cassandra.
- **Forensics**: Extend TTL on relevant telemetry/outbox/token tables, export evidence to investigation storage, and capture incident metadata in Cassandra audit tables.
- **Scope Validation**: Operators must confirm or adjust sink scope via the scope API; deliveries remain paused until revalidated.
- **Recovery**: After remediation, controlled replays (subject to `config/replay.yaml`) rebuild sink state; resume deliveries only when new credentials validated and audit log updated.
- **Runbook Integration**: Update incident runbooks and configuration (`config/security.yaml`) with lessons learned and rotation cadence adjustments.

### CDC Schema Metadata API
- **Endpoint**: `GET /api/v1/sinks/{sink_id}/schema` returns the current authorised CDC schema for the sink. Optional query params `version` (exact schema version) and `format` (`json`, `avro`) allow historical lookup and encoding preferences.
- **Endpoint**: `GET /api/v1/sinks/{sink_id}/schema/changes` returns incremental schema updates since a supplied `since_version` token, enabling adapters to prefetch diffs before rotation.
- **Endpoint**: `POST /api/v1/sinks/{sink_id}/schema/refresh` allows authorised operators to trigger regeneration of schema manifests (restricted to operator credentials).
- **Authentication**: Requires sink access token issued via refresh-token flow; operator-triggered refresh requires elevated role claims. All responses signed with `x-schema-version` and `etag` headers.
- **Payload**: Schemas include table descriptors (name, primary key, columns with type, nullable, default, codecs) plus change-types metadata and compatibility flags.
- **Consistency**: CDC service snapshots schema at capture checkpoints; queries honour the latest committed checkpoint ensuring runtime and adapters read self-consistent definitions.
- **Caching & Versioning**: Responses cacheable for `max-age=60s`; schema versions are monotonically increasing integers with semantic metadata stored in Cassandra control keyspace. Breaking changes require sink acknowledgement via adapter `SchemaAnnouncement` ACK.
- **Error Handling**: `404` when sink lacks schema, `412` when requested version is ahead of current, `409` when schema refresh is already running, `423` when sink is paused. Structured errors include `code`, `message`, `retry_after`.

## Non-Functional Requirements
- **NFR-001** Guarantee at-least-once delivery semantics even across restarts or failovers.
- **NFR-002** Scale to 100 sinks, 5 TB daily change volume, 5k events/sec sustained throughput while meeting tiered latency targets.
- **NFR-003** Operate across multiple Cassandra nodes without single point of failure; tolerate transient partitions.
- **NFR-004** Customer-facing APIs respond within 2 seconds under normal load (excluding sweep operations).
- **NFR-005** Observability data is structured and correlation-friendly, tagging sink ID and Cassandra node ID.
- **NFR-006** Security controls comply with internal auth standards and encrypt sensitive data in transit/at rest.
- **NFR-007** Disaster recovery restores from latest checkpoint within 30 minutes using Cassandra control keyspace snapshots replicated to S3 plus runbook-driven restoration.

## Key Entities
- **Sink Profile**: Metadata (identifier, credentials, contact, access scope, retention policy, state, audit timestamps, adapter type such as Postgres/MySQL/Redis).
- **CDC Event**: Captured change context (table, primary key, payload, version hash, capture timestamp, sequence token).
- **Adapter Contract**: Runtime-to-adapter API describing change application, sweep ingestion, and health reporting semantics, language-neutral.
- **Outbox Entry**: Delivery tracking record per sink (event ref, status, attempt count, last attempt timestamp, retention deadline).
- **Sweep Job**: Full data capture run (sink, scope, initiation context, progress checkpoints, snapshot manifest, completion state).
- **Node Heartbeat**: CDC service instance health (node ID, heartbeat timestamp, last processed log segment, backlog indicator).
- **Audit Log Entry**: Immutable record of actions affecting sinks, sweeps, credentials, configuration.

## Dependencies & Assumptions
- Cassandra clusters expose CDC logs with sufficient retention for the service to consume. `[Owner: Platform Operations]`
- CDC Cassandra keyspace hosts sink telemetry, credential, and token tables with cluster-wide replication. `[Owner: Platform Engineering]`
- Downstream sinks provide authenticated network endpoints (HTTPS or broker) and support standard adapter contracts for Postgres, MySQL, or Redis. `[Owner: Sink Operators]`
- ServiceNow event bus integration is available and linked to on-call workflows. `[Owner: SRE]`
- Central secrets manager is accessible for credential storage/rotation. `[Owner: Security]`
- CDC service-managed certificate authority is provisioned for issuing sink certificates, with optional enterprise PKI integration supported. `[Owner: Security]`
- Persistent storage for outbox/sweep manifests available (Cassandra or managed storage). `[Owner: Platform Engineering]`
- Encrypted S3 buckets provisioned for checkpoint replication. `[Owner: Platform Engineering]`
- Staffing allocated: 2 Java engineers, 1 SRE, 1 QA for initial release. `[Owner: Leadership]`


## Review & Acceptance Checklist
- [x] All mandatory sections populated and internally consistent.
- [x] Stakeholder personas identified and addressed.
- [x] Each requirement maps to scenarios or goals (see scenario mapping entries).
- [x] All `[NEEDS CLARIFICATION]` items assigned owners.
- [x] Scope boundaries captured in Out of Scope section.
- [x] No implementation-specific jargon biasing design decisions.

## Execution Status
- [x] Input processed
- [x] Concepts extracted
- [x] Scenarios drafted
- [x] Requirements authored
- [x] Entities identified
- [x] Dependencies recorded
- [x] Review checklist completed

*Specification ratified under TYA CDC Service Constitution v1.2.0*
