# TYA CDC Service Constitution

## Mission
Deliver and operate the TYA CDC platform—covering both the CDC control plane and the managed sink tooling—as a dependable data movement system that respects governance, observability, and operational excellence. Every change across either service must preserve stakeholder trust, support measurable quality, and maintain a clear audit trail that spec-kit automation can consume.

## Core Principles
1. **Spec-First Commitment** — No implementation proceeds without an approved spec-kit specification that centres on user value across both CDC control-plane and sink-bootstrap capabilities, with clarified scope and measurable outcomes.
2. **Plan Before Build** — Implementation plans are mandatory for work on either service. They decompose milestones aligned with spec-kit commands, highlight cross-service dependencies, and enforce constitution checks before tasks execute.
3. **Test-First Discipline (NON-NEGOTIABLE)** — Contract, integration, and unit tests are authored before implementation across both services. The Red→Green→Refactor loop is enforced for every task. Production code without a failing precursor test is prohibited.
4. **Operational Resilience** — Designs for the CDC service and sink tooling must accommodate node failures, back-pressure, and observability from inception. Health, metrics, auditing, and runbooks are first-class deliverables for both services.
5. **Simplicity with Traceability** — Favour straightforward architectures and direct dependencies across the platform. Every deviation or complexity must be justified within plan artefacts and linked back to specification requirements.
6. **Collaborative Transparency** — Codex and other agents must have up-to-date context files covering both services. Decisions, clarifications, and test evidence are documented where spec-kit expects them.

## Delivery Workflow
1. Draft or amend specification under `specs/[###-feature]/spec.md` using `/specify`.
2. Update this constitution if guardrails, tooling, or governance expectations change. Constitution amendments precede plans.
3. Produce the implementation plan (`/plan`) covering research, design artefacts, constitution checks, and task generation strategy.
4. Run `/tasks` to emit `tasks.md`; no coding starts until tasks exist and reference failing tests.
5. Execute tasks sequentially, maintaining TDD, logging deviations in complexity tracking tables, and documenting resolution of all [NEEDS CLARIFICATION] markers.
6. Validate work by running the full test suite, rehearsing the quickstart, and confirming observability baselines before requesting review or release.
7. Update release notes and operational documentation, ensuring metrics, alerts, and runbooks remain accurate.

## Operating Guardrails
### Architecture & Simplicity
- Maintain two governed services—the CDC control plane and sink bootstrap tooling—with shared guardrails. Internal modularisation is allowed when justified in the plan and tracked in complexity tables.
- Avoid introducing frameworks, message buses, or additional services without quantitative evidence and documented consent.
- Configuration, credentials, and infrastructure bindings must be externalised; no hard-coded secrets.

### Testing & Quality Gates
- Contract tests define external expectations (APIs, payloads, CDC schema) before implementation.
- Integration tests use real dependencies via Testcontainers. Mocks are limited to third-party systems that cannot be containerised.
- Unit tests focus on pure logic and edge handling after contracts/integration exist.
- CI must block merges lacking passing tests or violating formatting/quality policies defined in the plan.

### Observability & Operations
- Logs for both services are structured, correlation-friendly, and include sink ID, node ID, request trace, and operation type.
- Metrics follow Prometheus conventions and cover throughput, backlog, error rates, and sweep duration.
- Health/readiness endpoints and runbooks are mandatory deliverables before calling a feature complete.
- Alert policies for backlog growth, retention breaches, and node silence are defined in plans and validated in tasks.

### Documentation & Tooling
- Specification artefacts live under `specs/` with consistent numbering (e.g., `003-cdc-foundation`). Each service maintains the required files (`spec.md`, `plan.md`, `research.md`, `data-model.md`, `quickstart.md`, `contracts/`, `tasks.md`) and cross-service impacts are tagged.
- The sink runtime↔adapter contract (`specs/adapter-runtime-contract.md`) is the single source of truth for adapter interfaces; updates must be ratified before adapter or runtime changes ship.
- Agent context files (e.g., `/templates/CODEX.md`) are updated during Phase 1 to keep Codex aligned with stack choices and workflows.
- Clarifications, assumptions, and decisions are recorded with owners and due dates; unresolved items block advancement past planning.

## LLM Collaboration Rules
- Codex must cross-check the latest spec, plan, and constitution before making changes.
- When the environment blocks an action, Codex escalates with clear justification or proposes alternatives instead of proceeding silently.
- Generated content remains ASCII unless the repository already demands otherwise.
- Comments in code are concise and reserved for non-obvious logic or domain context.

## Governance & Amendments
- Amendments require: (1) written motivation, (2) impact analysis, (3) explicit approval recorded in version control, and (4) migration guidance for work in flight.
- Version bumps occur alongside amendment documentation; tag references in plans to keep spec-kit automation consistent.
- Persistent violations trigger work stoppage until remediated, with findings recorded in `complexity tracking` tables and communicated to stakeholders.

**Version**: 1.2.0  
**Ratified**: October 21, 2025  
**Supersedes**: Version 1.1.0 (October 20, 2025)
