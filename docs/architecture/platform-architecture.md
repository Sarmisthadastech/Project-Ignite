# Platform architecture

## Status

This document defines the planned architecture. None of the components below are implemented in this repository today. The only current application artifact is the static demo under `project_ignite_direct_upload/`, which remains separate and unchanged.

## System shape: modular monolith

The initial platform will use a modular monolith for backend domain logic. One backend repository and release train will contain domain modules with explicit ownership and internal interfaces. The HTTP API and background worker are separate runtime processes so their concurrency and failure handling can be managed independently, but they share the same application/domain code and database model. This is not a microservice-per-domain design.

Candidate backend modules include identity and tenancy, project/application registry, integration connections, ingestion, evidence and traceability, quality intelligence, audit, and AI orchestration. Modules should not reach into another module's tables or internals without an explicit interface. A future service extraction requires a demonstrated operational or organizational need and a migration plan for data ownership.

## Component boundaries

### Frontend: `apps/web`

Planned Next.js and TypeScript application for user interaction. It calls the backend API and renders only data authorized by that API. It must not hold integration secrets, connect directly to PostgreSQL, or make browser-side tenant selection an authorization decision. UI visibility is not a substitute for server authorization.

### Backend API: `apps/api`

Planned FastAPI application. It authenticates requests, resolves the actor and tenant membership, applies authorization, validates input, invokes domain services, and returns tenant-scoped responses. It owns synchronous request orchestration and enqueues durable jobs for longer operations. It should avoid embedding provider-specific integration code in route handlers.

### Database: `database/`

Planned PostgreSQL schema and migration history, managed using SQLAlchemy and Alembic. PostgreSQL is the source of truth for tenant, project, integration metadata, job references, evidence metadata, relationships, and audit records. Large payloads and files are stored in object storage; database rows retain their metadata, checksums, versions, and object references.

### Worker: `services/workers`

Planned separately runnable process in the same modular monolith. RabbitMQ and workers follow the first foundation milestone. Work-producing database transactions use a transactional outbox so committed state and event intent cannot diverge. The publisher uses broker publisher confirmations; workers acknowledge only after durable handling, are idempotent, handle duplicate messages, use bounded retries with backoff, and route exhausted work to dead-letter queues. Redrive is an audited operation with duplicate-safe semantics. Jobs carry tenant and actor/service identity context, correlation identifiers, idempotency keys, and explicit input references. Workers revalidate authorization or use a narrowly scoped service identity; a queued message alone is not authorization. Queue messages contain secret references only where necessary, never secret values.

### AI orchestration: `services/ai-orchestration`

Planned orchestration boundary for model calls, retrieval, prompt/version management, tool selection, policy checks, and evaluation. It consumes approved context and returns proposed outputs with citations to evidence and model/run metadata. It must not bypass domain authorization, directly mutate production systems, or treat model output as verified fact. Tools that change code, tickets, or configuration require explicit human approval and an auditable execution path.

### Shared packages and integrations

`packages/` will hold small, stable shared contracts and utilities; it must not become a dumping ground for cross-domain business logic. `integrations/` will contain provider adapters behind common interfaces for authentication, pagination, rate limits, webhook verification, normalization, and error handling. Provider-specific payloads should be retained or referenced as source evidence where policy permits.

## Control plane and data plane

These are logical responsibilities, not a commitment to separate deployments in the first release.

- **Control plane:** tenant and membership lifecycle, authentication configuration, roles and permissions, project/application registration, integration connection configuration, policy settings, and audit administration.
- **Data plane:** tenant-scoped ingestion, normalized engineering records, evidence objects and links, quality analysis, search/retrieval, and asynchronous processing.

The initial modular monolith may host both planes. Control-plane operations must be explicitly authorized and auditable. Data-plane jobs must carry tenant scope and must not gain broader access merely because they run asynchronously. A future physical separation would require explicit service identities and a reviewed control/data interface.

## Tenant isolation

Tenant context originates from authenticated identity and validated tenant membership, never an arbitrary client-supplied tenant ID. A client may select among memberships only after server-side validation. Tenant enforcement is mandatory across API requests, PostgreSQL, jobs/queues, cache, search, object storage, exports, logs/telemetry, and administrative/support operations. Tenant-owned records and artifacts carry a tenant key; deliberate global data is explicitly classified and handled separately.

For every tenant-owned PostgreSQL table, enable and force RLS. Policies fail closed when tenant context is absent or invalid and define both `USING` and `WITH CHECK`. The application sets validated tenant context transaction-locally for each transaction. Runtime roles are restricted, do not own protected tables, and do not have `BYPASSRLS`; migration and break-glass roles are separate and controlled. Superusers and roles with `BYPASSRLS` can bypass RLS, including forced RLS, and must not be used by application paths. See [Security and tenancy](security-and-tenancy.md).

Apply the same tenant boundary outside PostgreSQL: tenant-aware cache keys and authorization; mandatory tenant filters in search indexing and querying; tenant-scoped object authorization and short-lived signed URLs only after access checks; exports authorized and labeled for the requesting tenant; and tenant-safe support workflows. Logs and telemetry must not contain secrets or unnecessary payloads, and tenant context must not enable cross-tenant browsing. Backups, restore procedures, and operational tools must preserve these boundaries.

## Evidence and provenance

Evidence is an attributable source record or derived artifact, not simply a displayed finding. Each evidence version records tenant, source integration and external identifier, source revision/version or retrieval cursor where available, observed and ingested timestamps, media type, retention classification, and transformation lineage. For imported evidence, hash the exact unmodified source payload bytes as received and retained for that version, before parsing, normalization, or reserialization; compute SHA-256 and record the algorithm and digest with the version. If a source cannot supply raw bytes, define and record the canonical serialization used before accepting that evidence type. Raw evidence and normalized representations are distinguishable. Derived findings link to inputs and transformation/version. Corrections create new versioned evidence or auditable supersession records, never silent rewrites.

Large artifacts are planned for S3-compatible storage with database metadata. Version history and checksum verification are required. Do not promise universal WORM immutability until provider controls, retention lock, administrator access, and deletion behavior are selected and verified. Retention periods, deletion, legal hold, and exception approval remain unresolved decisions.

## Integration architecture

Each external connector will implement a common adapter contract while keeping provider-specific behavior isolated. The connection lifecycle includes explicit tenant ownership, credential reference, least-privilege OAuth scopes, status, consent/authorization metadata, and last-success/error information. Credentials are encrypted at rest, accessed only through tenant-scoped authorization, rotated and revoked through defined lifecycle operations, and usage is audited. Secret values never appear in logs or queue messages; queue work references authorized credential handles where needed. The secrets manager/key custody choice remains unresolved.

Ingestion should use cursor-based or timestamp-based incremental reads where possible, validate webhooks, observe provider rate limits, and support idempotent replay. Normalize provider data into canonical records without discarding source identifiers or provenance. The system should distinguish unavailable, stale, partial, and failed data from a complete successful sync. Initial supported providers and whether to prioritize polling or webhooks are undecided.

## AI orchestration boundaries

The orchestration module may retrieve only context the requesting actor is authorized to access. It should minimize sent data, record model/provider and prompt/tool versions, and return references to supporting evidence. Model output is a proposal with uncertainty and provenance, not an authorization decision or a source of truth.

Tool permissions are allowlisted and scoped per operation. Read-only analysis is the default. Any write-capable tool requires approval bound to the exact action, canonical parameters, actor, tenant, target, and expiry. Immediately before execution, revalidate authorization and approval scope; reject expired, changed, or replayed approvals. Use idempotent requests and audit approval and execution. Secrets, unrelated tenants, and unrestricted shell/repository access are not placed in model context. Model provider, data residency, retention settings, sandbox design, and evaluation approach remain unresolved.
