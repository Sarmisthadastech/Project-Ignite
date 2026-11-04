# Technology decisions

## Status and decision method

The technologies below are the requested initial target choices for the platform foundation. They are planned and are not currently present in the repository. The existing demo is a standalone HTML/CSS/JavaScript artifact and remains unchanged. These choices should be revisited if deployment, compliance, scale, or team constraints invalidate their assumptions.

| Area | Initial choice | Intended use | Status |
|---|---|---|---|
| Backend | Python and FastAPI | Modular monolith HTTP API and shared domain layer | Proposed; not implemented |
| Database | PostgreSQL | System of record, tenant-scoped relational data, RLS | Proposed; not implemented |
| ORM and migrations | SQLAlchemy and Alembic | Persistence mapping and reviewed schema migrations | Proposed; not implemented |
| Frontend | Next.js and TypeScript | Enterprise web application | Proposed; not implemented |
| Background processing | RabbitMQ | Durable asynchronous job transport | Proposed; not implemented |
| Object storage | S3-compatible storage | Large evidence artifacts and versioned source material | Proposed; provider unresolved |
| Observability | OpenTelemetry | Traces, metrics, and context propagation | Proposed; not implemented |
| Local development | Docker-based development | Reproducible local dependencies and service startup | Proposed; not implemented |

## Backend: Python and FastAPI

Use Python for backend domain and integration code, with FastAPI for the HTTP boundary and generated API schema. Keep domain rules independent of route handlers and provider SDKs. Use typed request/response contracts, explicit dependency injection for tenant/actor context, and startup validation for required configuration. The API and worker should run as separate processes from the same application codebase at first.

Open decisions: supported Python version, package/build tool, API versioning policy, synchronous versus asynchronous database driver, and lint/type-check tooling.

## Database: PostgreSQL

Use PostgreSQL for structured system-of-record data and tenant-scoped relations. Use database constraints for invariants where appropriate, indexes based on observed query patterns, and RLS as a second isolation barrier. Avoid using JSON columns as a substitute for stable domain schemas; provider-specific source payloads may need a separate retention-aware representation.

Open decisions: managed database provider/version, hosting region, connection pooling approach, backup/restore objectives, high availability, and whether text search starts in PostgreSQL or requires a dedicated search service.

## SQLAlchemy and Alembic

Use SQLAlchemy for database access and Alembic for explicit, reviewed migrations. Migrations are the authoritative schema history; application startup must not silently mutate production schema. Migration execution uses a distinct privileged role from the application runtime. Tenant context for RLS is transaction-local and must be reset safely with pooled connections.

Open decisions: SQLAlchemy major version and sync/async mode; convention for generated versus hand-authored migrations; migration deployment and rollback policy.

## Frontend: Next.js and TypeScript

Use Next.js and TypeScript for the enterprise frontend. The frontend communicates with the API through documented contracts and does not connect directly to the database or hold integration credentials. Authentication session strategy, rendering mode, UI component system, and frontend hosting remain undecided.

## Background processing: RabbitMQ

Use RabbitMQ in a subsequent foundation increment to decouple longer ingestion and analysis tasks from request latency. The broker is transport, not the source of truth for job or evidence state. Persist job metadata and idempotency state in PostgreSQL. A transactional outbox is mandatory for database changes that enqueue work; require publisher confirmation, idempotent workers, duplicate message handling, bounded retry/backoff, dead-letter queues, and audited redrive. Define message schemas, ordering expectations, and tenant/actor service context before production use. Never put secrets in queue messages. The worker is an independently runnable process in the modular monolith, not a set of domain microservices.

Open decisions: managed versus self-hosted broker, topology and queue isolation, durability/availability configuration, and operational ownership.

## S3-compatible object storage

Use S3-compatible object storage in a subsequent foundation increment for large artifacts that do not belong in relational rows. Store metadata, SHA-256 hashes of clearly specified content bytes, tenant ownership, version history, and retention classification in PostgreSQL. Enforce authorization before object access and use encryption in transit and at rest. Do not claim universal WORM immutability until provider controls are selected and verified. Provider, bucket layout, per-tenant key strategy, key management, versioning/retention lock, lifecycle, legal hold, and deletion semantics are open decisions.

## OpenTelemetry observability

Instrument API, worker, database, and broker interactions with OpenTelemetry-compatible traces and metrics. Propagate correlation identifiers through HTTP requests and messages. Avoid placing secrets, raw evidence, or unnecessary personal data in telemetry; tenant identifiers should be handled according to a defined telemetry policy. Backend/exporter and retention decisions remain open.

## Docker-based local development

Use Docker-based local development to make PostgreSQL reproducible for the first milestone. Add RabbitMQ and selected dependencies in later increments. Provide non-production defaults and synthetic data only. Do not put real credentials in images or checked-in environment files. The exact orchestration mechanism, local S3-compatible emulator, and CI execution environment are not yet chosen.

## Deployment models

Local Docker development is distinct from production. The intended deployment environments are:

- **Managed SaaS:** planned initial deployment model, not currently implemented. The target is a centrally operated service with shared control plane and tenant-scoped application/data plane. Tenant boundaries, regional placement, noisy-neighbor limits, and operator access require explicit design.
- **Private cloud:** customer-controlled cloud account or network. Not a supported capability at this baseline; installation, upgrades, identity/secrets integrations, egress, monitoring, and support access are not yet defined.
- **Customer-managed on-premises:** not supported at this baseline. Packaging, upgrade compatibility, backup ownership, support access, and responsibility boundaries are unresolved.
- **Air-gapped:** no external network dependency at runtime. This is not currently a supported capability; model-provider access, license/update distribution, telemetry export, and connector availability would need separate decisions.

No production deployment profile is implemented or certified. Before claiming support for any environment, define deployment topology, TLS/network boundaries, secrets and key custody, patch/upgrade path, backup/restore responsibilities, observability, and support access. Initial support target and capability differences among these environments remain open decisions.

## Operations and recovery

Production operations require monitored service health, latency/error rates, database capacity, queue age/failure rates once RabbitMQ is introduced, connector health, storage errors, and audit delivery failures. OpenTelemetry provides instrumentation, not alert thresholds or response ownership. Require encrypted backups and tested restoration of database and evidence objects. Provisional planning targets are RPO 24 hours and RTO 8 hours, subject to product/operations approval before production; these are not service commitments. Backup retention, point-in-time recovery, DR topology, and restore-test cadence remain unresolved.

## Automated testing

CI must use a real PostgreSQL instance for database integration tests; SQLite or mocks cannot stand in for PostgreSQL RLS and role behavior. Before tenant-owned data is accepted, tests cover RLS and `FORCE ROW LEVEL SECURITY`, restricted runtime permissions, missing tenant context, cross-tenant reads and writes, connection-pool context leakage, migration execution, and audit write/update/delete permissions. Add broker/outbox and object-store tenant-isolation tests when those increments are implemented.

## Decision review triggers

Revisit these choices before committing to production operations if a required compliance regime, data residency constraint, availability objective, expected volume, integration model, or team capability materially conflicts with them. Record consequential changes with rationale and migration impact in this directory.
