# Project Ignite architecture

## Status and scope

This directory records the initial target architecture for Project Ignite. It is a design baseline, not evidence that the described platform has been implemented. At this baseline, the repository contains only the static demonstration in `project_ignite_direct_upload/`; the platform services, production frontend, database, workers, integrations, and security controls described here are planned.

The existing demonstration is preserved as-is and is not part of the production application boundary. This foundation task creates documentation only: it does not add application code, deployment resources, database schema, or operational configuration.

## Architecture overview

Ignite will be a tenant-aware engineering quality and evidence platform. It will connect project and application context with external engineering systems, retain attributable evidence, and provide explainable quality intelligence. Human users remain accountable for consequential engineering actions.

The initial application is a modular monolith: one versioned backend codebase organized into explicit domain modules, with an HTTP API and a separately runnable worker process. The API and worker share domain code and PostgreSQL; they are not independent microservices. A Next.js web application is a separate frontend. PostgreSQL is the system of record for structured platform data, while S3-compatible object storage holds larger immutable or versioned evidence artifacts. RabbitMQ carries asynchronous work. See [Platform architecture](platform-architecture.md) and [Technology decisions](technology-decisions.md).

## Platform vision

The planned platform should make it possible to:

- Register projects and applications and connect approved engineering systems.
- Ingest and normalize requirements, changes, tests, defects, and delivery evidence with source and version context.
- Link evidence across the engineering lifecycle and identify coverage gaps or change risk.
- Present findings with traceable supporting evidence, freshness, and limitations.
- Add AI-assisted analysis behind explicit authorization, bounded tools, and human review.

The public demo illustrates selected concepts with fictional data. Its scripted behavior is not a live service, and demo content must not be represented as production evidence.

## Architectural principles

1. **Preserve the demo boundary.** Do not modify or depend on `project_ignite_direct_upload/` as part of the platform runtime.
2. **Start with a modular monolith.** Keep domain ownership explicit and use internal interfaces. Split a component into a separately deployed service only when a measured scaling, reliability, security, or ownership need justifies the cost.
3. **Tenant context is mandatory.** Tenant context originates from authenticated identity and validated membership, never from an arbitrary client-supplied tenant ID. Enforce it across API requests, PostgreSQL, jobs/queues, cache, search, object storage, exports, logs/telemetry, and administrative/support operations. PostgreSQL RLS is mandatory defense in depth.
4. **Evidence must be attributable.** Preserve source identity, tenant, ingestion time, source version or revision when available, processing history, and links between derived findings and their inputs.
5. **Asynchronous work is explicit and recoverable.** Use durable jobs for long-running ingestion and analysis; make handlers idempotent, observable, and safe to retry.
6. **AI is an orchestrated capability, not an authority.** AI actions use narrowly scoped tools and permissions. Generated conclusions remain distinguishable from source evidence and require human approval for consequential changes.
7. **Secure defaults and observable operations.** Apply least privilege, centralized secrets handling, structured audit events, and OpenTelemetry instrumentation from the foundation stage.
8. **Document uncertainty.** Treat technology choices in this baseline as proposed decisions until validated against deployment, scale, compliance, and user requirements.

## Initial implementation scope

The first foundation milestone is deliberately narrow and contains no application workflow or tenant-owned application data:

- Clean repository structure for the agreed platform boundaries, preserving `project_ignite_direct_upload/` and all its assets.
- Backend application skeleton, PostgreSQL connectivity, and baseline migrations.
- Distinct restricted runtime and migration database roles; tenant tables and customer data are not introduced in this milestone.
- Basic liveness and readiness checks, with readiness verifying required database connectivity.
- Basic logs, metrics, and traces with correlation context and secret-safe fields.
- CI pipeline that runs automated checks and real PostgreSQL integration tests.
- Local development instructions that work from a clean checkout.
- Verification that the existing demo is unchanged.

RabbitMQ, transactional outbox, worker execution, and object storage follow in a subsequent foundation increment. These are planned outcomes, not current capabilities. Do not accept tenant-owned application data until authentication, validated membership, RLS enforcement, and cross-tenant tests pass.

### Foundation milestone acceptance criteria

- The repository includes the agreed directory boundaries and a root setup guide; the demo file and its assets are unchanged.
- The agreed clean repository structure exists, with all demo files and assets byte-for-byte unchanged.
- A developer can follow local instructions from a clean checkout and start the backend and PostgreSQL.
- The backend exposes liveness and readiness checks; readiness verifies the database dependency.
- Baseline migrations apply to a clean real PostgreSQL database and are exercised by CI. Production rollback is not presumed safe; use forward-compatible migrations and document any tested downgrade separately.
- Runtime and migration roles are distinct; runtime privileges are restricted and the role is not a table owner or `BYPASSRLS` role.
- CI runs real PostgreSQL integration tests, not SQLite substitutes, for connectivity, migrations, and database role assumptions. Tenant/RLS-specific tests are mandatory before tenant-owned data is accepted.
- Configuration is validated at startup; secrets are not committed, logged, or embedded in images.
- Basic logs, metrics, traces, and correlation context are available without recording secrets or unnecessary sensitive payloads.
- Health and readiness behavior, migrations, and demo preservation are verified by automated checks.
- No capability is described as deployed or secure until implementation and verification support that claim.

## Assumptions and unresolved decisions

Assumptions for this baseline: the first deployment is a single product operated by one team; the first release is a web application with an API; tenant-owned records require strict cross-tenant isolation; and managed SaaS may begin with a shared PostgreSQL cluster using enforced tenant controls. Shared RabbitMQ is a later decision, not part of the first milestone.

Decisions still open include identity provider and authentication flow; tenant lifecycle and membership model; initial RBAC roles and permission granularity; hosting region and deployment platform; database and object-store backup/retention requirements; object-store provider and encryption-key model; compliance obligations and data residency; integration credential custody and OAuth support; evidence retention/deletion policy; and AI model/provider, data handling, and evaluation policy. These decisions are tracked in [Technology decisions](technology-decisions.md), [Security and tenancy](security-and-tenancy.md), and [Implementation roadmap](implementation-roadmap.md).
