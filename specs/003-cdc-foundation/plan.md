# Implementation Plan — TYA CDC Service Foundation

**Branch**: `003-cdc-foundation`  
**Date**: October 20, 2025  
**Specification**: `specs/003-cdc-foundation/spec.md`

## Execution Flow Compliance
1. Load feature specification from input path — ✅ (`specs/003-cdc-foundation/spec.md`)
2. Extract technical context; cross-check with constitution guardrails — ✅ (see Technical Context)
3. Perform initial constitution check — ✅ No violations detected
4. Phase 0 planning (research agenda & owners) — outlined below
5. Phase 1 design (artefacts for contracts, data model, quickstart, agent context) — outlined below
6. Constitution check after Phase 1 outline — ✅ No new violations introduced
7. Phase 2 task generation strategy (pre-/tasks guidance) — defined below
8. Update progress tracking and prepare for `/tasks` command — see Progress Tracking

## Technical Context
- **Language / Runtime**: Java 21 (aligned with constitution simplicity & modern support)
- **Build Tooling**: Maven; multi-module structure limited to clearly justified modules (core, api, delivery, ops) documented in plan before implementation.
- **Key Libraries**: DataStax Cassandra driver (CDC), Netty/Helidon or minimal HTTP server (TBD after research), Jackson for JSON, Micrometer + Prometheus registry, SLF4J + Logback.
- **Persistence**: Cassandra control keyspace (registry/outbox/heartbeats/manifests); S3 (or equivalent) for replicated checkpoint snapshots.
- **Runtime Topology**: Dedicated CDC agent on every Cassandra node, running co-located with the database to minimise capture lag; deployment managed per-node (system service or lightweight container) per platform standards.
- **Testing Tooling**: JUnit 5, Testcontainers (Cassandra cluster + sink simulators), WireMock/MockWebServer for HTTP sink contract tests, Awaitility for async assertions.
- **Performance Envelope**: Tiered SLAs codified in `config/cdc-service.yaml` — Gold (lag ≤3s p95, sweep ≤45m per keyspace, backlog alert at 5k events, recovery RTO ≤15m, ack ≤60s, heartbeat 15s), Silver (lag ≤10s p95, sweep ≤3h, backlog alert 20k, RTO ≤30m, ack ≤120s, heartbeat 30s), Bronze (lag ≤60s p95, sweep ≤12h, backlog alert 50k, RTO ≤60m, ack ≤300s, heartbeat 60s).

## Constitution Gate Review
- **Simplicity**: Single deployable service; optional modules require documentation prior to creation. No message buses or additional services introduced.
- **Architecture Guardrails**: Admin APIs via HTTP only. CLI wrappers acceptable after Phase 1 design if needed.
- **Testing Discipline**: Contract tests precede integration/unit implementation; TDD checkpoints must appear in forthcoming tasks.
- **Observability**: Structured logging, Micrometer metrics, health/readiness endpoints, ServiceNow alert hooks are planned deliverables.
- **Governance**: All artefacts live under `specs/003-cdc-foundation/`; agent context updates scheduled in Phase 1; complexity tracking table currently empty.

Result: **Initial Constitution Check — PASS**

**Latest Alignment Review (Oct 21, 2025)**: Plan remains compliant with TYA CDC Service Constitution (no new deviations).

## Phase 0 — Research Plan
**Objectives**
- Confirm Cassandra CDC log access patterns (segment format, retention guarantees, multi-node coordination) and document required control tables.
- Validate sink delivery protocols (HTTPS push baseline; evaluate broker option) and authentication schemes.
- Clarify security/compliance expectations (PII handling, credential rotation cadence, audit logging retention).
- Document per-node service management approach (system service vs lightweight container orchestration) and secrets manager integration path.

**Deliverables**
- `specs/003-cdc-foundation/research.md` capturing findings, decision records, and open items.
- Updated `[NEEDS CLARIFICATION]` entries with owners/due dates.
- Links or notes for future ADRs if architectural trade-offs emerge.
- Credential compromise outline (halt, revoke, audit, replay) coordinated with Security for inclusion in operational runbooks.

**Owners**
- Platform Operations: CDC retention confirmation, deployment topology.
- Security: Credential compromise handling, secrets manager constraints.
- Data Platform Owner: SLA sign-off, governance escalation workflow.

## Phase 1 — Design & Contract Artefacts
**Objectives**
- Produce `data-model.md` defining Cassandra tables, partition/TTL/retention mapping, and S3 checkpoint schema (telemetry, scope, replay, credential, compromise-supporting tables).
- Define service APIs & payloads under `contracts/` (registration, operations, metrics, sweep trigger, replay API, schema/scope/telemetry endpoints) with JSON/YAML schemas and example interactions.
- Author failing contract tests aligned to FRs (registration validation, CDC ingestion/outbox write, delivery ack, sweep trigger, replay stub, metrics endpoint, schema change pause).
- Establish integration test scaffolding using Testcontainers (Cassandra cluster + sink mock) — failing until implementation.
- Draft `quickstart.md` describing local setup prerequisites, environment variables, secrets, and verification steps (run failing test suites initially).
- Check in scaffolded configuration files split by concern (`config/core.yaml`, `config/security.yaml`, `config/telemetry.yaml`, `config/replay.yaml`, `config/sinks/default.yaml`) with commented defaults.
- Update `/templates/CODEX.md` with stack/test expectations and constitution guardrails for collaborating agents.
- Document the SLA configuration schema and align runtime with modular configs (`config/core.yaml`, `config/security.yaml`, `config/telemetry.yaml`, `config/replay.yaml`).

**Constraints & Guardrails**
- Every contract must trace back to spec requirements (document mapping in `contracts/README.md`).
- Data model must annotate compliance requirements (audit fields, retention TTL, encryption flags).
- Quickstart emphasises TDD workflow (run contract tests first, expect failures) and observability verification commands.
- Post-design constitution check ensures new artefacts respect simplicity/testing rules. Result: **PASS** (no conflicts identified in outline).

## Phase 2 — Task Generation Strategy (Pre-/tasks)
- Tasks derived from Phase 1 artefacts only; no implementation coding until `/tasks` produces sequence.
- Order tasks logically: environment bootstrap → contract tests → integration scaffolding → domain modelling → delivery mechanisms → observability → hardening → documentation polish.
- Tag parallelisable efforts with `[P]` (e.g., metrics vs registration contracts) while ensuring dependencies recorded.
- Each task must cite the failing test it introduces or the artefact it finalises.
- Include explicit tasks for constitution compliance audit, ServiceNow integration validation, DR runbook rehearsal test, and configuration schema validation (lint + reload checks).
- Final tasks cover running full test suite, executing quickstart, and updating release/ops notes.

## Complexity Tracking
| Violation | Justification | Simpler Alternative Considered | Outcome |
|-----------|---------------|---------------------------------|---------|
| *(none)* | — | — | — |

## Clarifications & Owners
- `[RESOLVED] Deployment model: CDC agent runs directly on each Cassandra node (per-platform service management)` — Owner: Platform Operations — Resolution recorded in Technical Context.
- `[RESOLVED] Tiered SLA definitions ratified (Gold ≤3s/45m/5k/15m, Silver ≤10s/3h/20k/30m, Bronze ≤60s/12h/50k/60m; ack 60s/120s/300s; heartbeat 15s/30s/60s)` — Owner: Data Platform Owner — Documented in spec and plan.
- `[RESOLVED] Replay policy (bounded window, auth)` — Owner: Platform Operations & Security — Captured in specification (config/replay.yaml, FR-022).
- `[RESOLVED] Credential compromise response workflow` — Owner: Security — Captured in specification (Compromise response policy, FR-023).

## Progress Tracking
- [x] Phase 0 research scope defined
- [x] Phase 1 design artefacts planned
- [x] Phase 2 task strategy outlined
- [x] Initial constitution check pass recorded
- [x] Post-design constitution check pass recorded
- [x] All clarifications resolved
- [x] Ready to execute `/tasks`

*Plan prepared under TYA CDC Service Constitution v1.2.0*
