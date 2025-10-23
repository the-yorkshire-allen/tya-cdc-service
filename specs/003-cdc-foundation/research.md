# TYA CDC Service Foundation — Research Log

_Phase 0 findings supporting specification `specs/003-cdc-foundation/spec.md`._

## 1. Cassandra CDC Retention & Segment Format (TSK-001)
- **Capture source**: Cassandra 4.x change data capture commit logs (`cdc_raw`) per node. Segments rotate when full (default 32 MB).
- **Retention expectation**: minimum 48 hours to satisfy Bronze replay windows; policy set to 72 hours for Gold/Silver consistency. Requires cluster config `cdc_total_space_in_mb` ≥ 2× peak daily change volume per node.
- **Processing model**: co-located CDC agent tails segments sequentially, writing checkpoints (`checkpoints` table) after every segment flush.
- **Operational note**: disk-headroom alerts triggered at 80% of `cdc_total_space_in_mb`; if breached, pipeline pauses ingestion and escalates via ServiceNow.

## 2. Sink Delivery Protocol & Authentication (TSK-002)
- **Transport**: HTTPS push from CDC runtime to sink-hosted ingest endpoint; optional mTLS.
- **Authentication**: short-lived access tokens minted from refresh tokens issued during registration; runtime also presents client cert.
- **Acknowledgement**: sinks respond with `202 Accepted` including ack token. Retry policy defined in spec FR-004 (exponential with jitter).
- **Broker alternative**: deferred; HTTP push is baseline for foundation release. Broker integration earmarked as future ADR if latency requirements increase.

## 3. Security Policies Resolution (TSK-003)
- **Replay bounds**: codified in `config/replay.yaml` (`max_window_seconds`, `max_history_days`). Default 7 days for PII, 30 days non-PII; dual approval required beyond defaults.
- **Compromise workflow**: detailed in spec (policy + FR-023). Upon compromise state, deliveries halt, tokens/certs rotated, telemetry preserved, and controlled replay required before resumption.
- **Audit logging**: all replay/compromise actions stored in Cassandra tables (`replay_requests`, `compromise_events`) with TTL = 180 days.

## 4. SLA & Deployment Confirmation (TSK-004)
- **Tier SLAs**: Gold/Silver/Bronze lag and sweep metrics confirmed with Data Platform Owner; recorded in spec Policy Decisions.
- **Deployment scheduler**: per-node systemd service on Cassandra hosts with optional container packaging for future orchestration. SRE owns service template.
- **Escalation**: SLA breaches generate ServiceNow incidents routed via platform operations queue (“CDC-L2”).

## 5. Credential Compromise Runbook Outline (TSK-004A)
1. **Detect & Flag** — automated signal (telemetry anomaly or SOC alert) posts to compromise queue; operations invokes `POST /sinks/{id}/state` → `compromise`.
2. **Freeze Delivery** — runtime pauses outbox replay; scope/replay APIs disabled for sink.
3. **Rotate Secrets** — security issues new refresh/ingest tokens via secrets manager; runtime refreshes trust bundle; adaptor restarts if required.
4. **Preserve Evidence** — extend TTL on telemetry/outbox/token tables for incident window, export to forensic storage.
5. **Communicate** — notify sink operators with incident ticket; share rotation steps.
6. **Validate Scope** — operations reviews `scope` definition; adjust if necessary.
7. **Replay & Resume** — after controls re-established, run targeted replay per `config/replay.yaml`; once successful, set sink state to `active` and close incident with postmortem link.

## 6. Outstanding Questions
- None for Phase 0; clarifications resolved or documented above. Future ADRs may revisit broker delivery and multi-region replication strategy.
