# Tasks — TYA CDC Service Foundation

_All tasks must respect TYA CDC Service Constitution v1.2.0. Tests are authored before production code; each feature task cites the failing test(s) it introduces._

## Phase 0 — Research & Clarifications
- [x] **TSK-001** Gather Cassandra CDC retention & segment format details; document findings in `research.md` Section 1. _Owner: Platform Operations liaison._
- [x] **TSK-002** Confirm sink delivery protocol expectations (HTTPS vs broker) and authentication schemes; record decision + rationale in `research.md`. _Owner: Integration lead._
- [x] **TSK-003** Document resolved security policies (replay bounds, credential compromise workflow) in `research.md` and ensure specification references are cross-linked. _Owner: Security liaison._
- [x] **TSK-004** Obtain SLA sign-off for tiered freshness/backlog targets and deployment scheduler preference; update spec + research docs accordingly. _Owner: Data Platform Owner liaison._
- [x] **TSK-004A** Draft credential compromise incident runbook outline aligned with spec workflow; store under `research.md` and reference in plan deliverables. _Owner: Security liaison._

## Phase 1 — Design Artefacts & Contracts
- [ ] **TSK-005** Draft `data-model.md` with Cassandra table schemas, partition keys, TTL/retention, audit fields, and S3 checkpoint manifest structure.
- [ ] **TSK-006** Create `contracts/README.md` mapping FR/NFR items to contract files; outline validation approach.
- [ ] **TSK-007** Author failing contract test `RegistrationContractTest.requiresMandatoryMetadata()` defining sink registration API behaviour; add corresponding JSON schema under `contracts/registration/`.
- [ ] **TSK-008** Author failing contract test `CredentialLifecycleContractTest.enforcesRotationAndRevocation()` covering FR-002; add API contract files.
- [ ] **TSK-009** Author failing contract test `CdcCaptureContractTest.persistChangesWithOrdering()` for FR-003 + FR-006; include CDC payload schema examples.
- [ ] **TSK-010** Author failing contract test `DeliveryAckContractTest.retriesUntilRetentionDeadline()` for FR-004 + policy alignment; document delivery acknowledgement payload.
- [ ] **TSK-011** Author failing contract test `SweepContractTest.producesConsistentSnapshot()` for FR-005; define sweep trigger/manifest contract.
- [ ] **TSK-012** Author failing contract test `MetricsContractTest.exposesPrometheusSeries()` for FR-008/FR-009; capture sample metric exposition.
- [ ] **TSK-013** Author failing contract test `SchemaChangeContractTest.blocksUntilMetadataAligned()` for FR-010; specify notification payload.
- [ ] **TSK-014** Author failing contract test `ReplayContractTest.replaysWithinAuthorizedWindow()` for FR-013; codify replay submission/approval payload and expected validation errors.
- [ ] **TSK-014A** Capture Replay API spec under `contracts/replay/` (POST/GET/approve/cancel schemas) with examples mapped to config policy fields.
- [ ] **TSK-015** Set up integration test scaffolding `CdcCaptureIntegrationTest` using Testcontainers Cassandra; ensure test fails pending implementation.
- [ ] **TSK-016** Add integration test `DeliveryFanoutIntegrationTest` simulating multi-sink delivery with mock endpoints (fails initially).
- [ ] **TSK-017** Add integration test `OperationalApiIntegrationTest` validating pause/resume/deactivate flows via Testcontainers (fails initially).
- [ ] **TSK-018** Create observability integration test `ObservabilityIntegrationTest` asserting metrics + health endpoints behaviour via Micrometer test registry (fails initially).
- [ ] **TSK-019** Draft `quickstart.md` guiding local setup (Testcontainers, env vars, secrets) and pointing to failing test suites for first run.
- [ ] **TSK-020** Update `/templates/CODEX.md` with stack overview, testing workflow, constitution reminders, and artefact paths.
- [ ] **TSK-021** Re-run constitution checklist; append findings (if any) to `plan.md` complexity tracking and note pass/fail status.

## Phase 2 — Implementation Prep & Governance
- [ ] **TSK-022** Derive Red→Green→Refactor workflow notes for each contract/integration test, embedding instructions in task annotations or comments.
- [ ] **TSK-023** Produce dependency graph for upcoming coding tasks (core → delivery → observability) and store in `plan.md` appendix.
- [ ] **TSK-024** Define ServiceNow integration validation plan (test stubs, incident payloads) and document in `research.md` & `contracts/alerting/`.
- [ ] **TSK-025** Draft DR rehearsal checklist (restore from S3 snapshots) in `quickstart.md` or separate runbook reference.
- [ ] **TSK-026** Prepare release note outline and operational handoff checklist under `specs/003-cdc-foundation/quickstart.md` appendix or new `ops-notes.md`.
- [ ] **TSK-026A** Implement configuration schema lint/validation tool (covering `config/*.yaml`) and document usage in `quickstart.md`.
- [ ] **TSK-027** Final readiness review: verify all clarifications resolved, artefacts committed, and failing tests collected; mark progress checklist as ready for implementation phase.

## Validation & Compliance
- [ ] **TSK-028** Execute full test suite (`mvn test`) expecting failures introduced above; capture output summary in `research.md` to anchor Red phase.
- [ ] **TSK-029** Run quickstart dry-run using documentation to ensure instructions reproduce failing state; adjust docs if discrepancies found.
- [ ] **TSK-030** Log constitution compliance confirmation in `plan.md` and prepare changelog entry for upcoming implementation.

*Proceed to implementation only after all Phase 0–2 tasks are complete and `/tasks` checklist is satisfied.*
