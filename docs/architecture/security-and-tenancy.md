# Security and tenancy

## Status

This document describes requirements for planned capabilities. Authentication, authorization, tenant isolation, RLS, audit logging, secrets management, and AI permission controls are not implemented in the current repository. The existing static demo is not an authenticated platform and must not be treated as a secure production system.

## Authentication and authorization

Authentication must establish a verified principal through an identity-provider abstraction so provider-specific protocols do not define domain authorization. The provider, protocol, session/token model, MFA requirements, account recovery, and service-to-service identity implementation are unresolved. Do not implement a custom password system without an explicit reviewed decision.

Model users, tenants, and tenant memberships with explicit lifecycle states, including invitation/acceptance, active membership, suspension, removal, and revocation. Membership suspension or revocation must invalidate or re-evaluate sessions/tokens and queued authority before further access. Define token expiry, refresh, revocation, and session invalidation behavior. Create separate service identities for workers and integrations; scope them to tenant and capability, rotate their credentials, and audit their use.

Authorization is enforced server-side for every API operation and asynchronous action. Resolve tenant membership from the authenticated principal; tenant context must originate from authenticated identity and validated membership, never an arbitrary client-supplied tenant ID. Deny by default and perform resource-level authorization on each object/action boundary. Frontend visibility and opaque identifiers are not authorization controls.

## Tenant isolation requirements

- Tenant-owned records include an explicit, non-null tenant key and tenant-aware uniqueness constraints.
- Every API request, PostgreSQL operation, job and queue message, cache read/write, search index/query, object access, export, log/telemetry path, and administrative/support operation is scoped to an authorized tenant.
- Tenant context is derived from verified identity and membership, then propagated explicitly through requests and jobs.
- Cross-tenant access attempts fail closed and are covered by automated tests, including background workers and administrative paths.
- Cache keys and search documents/queries carry tenant scope; object access policies and exports enforce authorization; logs, traces, metrics, backups, and support tooling do not create unintended cross-tenant disclosure.
- Administrative or support access is separately authorized, least privilege, time bounded where practical, and audited.

These are requirements, not existing guarantees.

## PostgreSQL row-level security

For every tenant-owned table, `ENABLE ROW LEVEL SECURITY` and `FORCE ROW LEVEL SECURITY` are mandatory. Policies fail closed if tenant context is absent or invalid and must specify both `USING` and `WITH CHECK` predicates. Set validated tenant context transaction-locally for every database transaction, and verify it cannot leak across pooled connections. Runtime roles are restricted, do not own protected tables, and do not have `BYPASSRLS`. Migration roles and break-glass roles are separate from runtime, narrowly granted, controlled, and audited. Application runtime must never connect as a superuser or `BYPASSRLS` role. PostgreSQL superusers and roles with `BYPASSRLS` can bypass RLS even when `FORCE ROW LEVEL SECURITY` is enabled; those roles are privileged trust boundaries and must be unavailable to normal application paths.

RLS does not replace application authorization, tenant-aware constraints, or tests. Verify policies using the same restricted runtime role used by the application. Test RLS enforcement and FORCE behavior, role permissions, absent context, invalid context, cross-tenant reads and writes, mismatched tenant IDs, bulk operations, migrations, and worker access. Include connection-pool reuse tests proving tenant context does not leak. Define a carefully controlled path for intentionally global reference data and system administration.

## Role-based access control

Use role-based access control as the initial model, with tenant-scoped membership and resource-level action checks. Provisional initial roles are tenant owner, administrator, contributor, and viewer; this model remains planned pending approval. Define permissions as capabilities and authorize at resource/action boundaries rather than scattering role-name comparisons throughout code. Document the role-to-capability matrix before implementation. Membership suspension/revocation and token/session invalidation must stop API, worker, and integration access. Project/application-level scopes or custom roles may be needed, but should not be assumed until workflows are known.

Open decisions include role definitions, invitation and membership lifecycle, project-level delegation, support/operator roles, separation of duties, and whether external collaborators can be granted limited access.

## Audit logging

Record security-relevant events including authentication and membership changes, authorization failures, integration connection changes, evidence access/export/deletion, policy changes, AI tool invocations, and approved write actions. Audit events should include actor or service identity, tenant, action, target, timestamp, outcome, correlation ID, and relevant source/version reference without recording secrets or unnecessary payloads.

Audit records are append-only by application contract. The application runtime role has no `UPDATE` or `DELETE` privilege on audit records; audit writing and reading use explicitly scoped privileges. Events include a schema version and sufficient ordering/correlation data to investigate related actions. Any correction is a new event, not an edit. Define tamper-evidence or protected external export controls and monitor write failures/gaps. Retention, legal hold, export access, and alerting requirements remain unresolved. This logging capability does not currently exist; privileged database administrators remain a separate trust boundary unless records are protected in an independently controlled sink.

## Secrets management

Do not commit credentials or embed secrets in frontend bundles, container images, logs, queue/job messages, or AI prompts. Local development uses synthetic credentials and ignored local configuration. Production secrets must come from an approved secrets manager or managed identity mechanism, with least privilege, encryption at rest, rotation, revocation, access audit, and separation between environments. Connector credentials are accessible only through a tenant-scoped authorization check; usage is audited. Queue messages carry authorized credential references, never secret values. OAuth grants request least-privilege scopes and support revocation. Integration tokens, webhook secrets, and signing keys require explicit encryption and lifecycle design.

The secrets manager, key management service, encryption-key ownership, per-tenant key strategy, and integration token storage model are unresolved.

## Secure AI agent permissions

AI orchestration is planned and is not currently connected to live systems. Apply these constraints before enabling it:

- Authorize retrieval using the requesting user's tenant membership and object-level permissions.
- Send only the minimum context needed; exclude secrets and unrelated tenant data.
- Default to read-only tools. Use explicit allowlists, scoped credentials, and validated inputs; do not provide unrestricted shell, network, or repository access.
- Treat retrieved documents and tool output as untrusted data. They cannot grant permissions or override system policy.
- Clearly label generated analysis and link it to supporting evidence, model/provider, and prompt/tool version where appropriate.
- Require human approval at execution time for changes to code, tickets, configuration, or production systems. Bind approval to the exact action, canonical parameters, actor, tenant, target, and expiry. Revalidate authorization immediately before execution and reject changed, expired, or replayed approvals. Record approval and resulting action in audit history.
- Apply timeouts, quotas, redaction, output validation, and monitoring. Define evaluation for accuracy, unsafe tool use, and tenant leakage before release.

Model provider, data retention/residency, prompt storage, approval UX, and evaluation thresholds remain open decisions.

## Operations, backups, and recovery

Minimum planned monitoring covers API availability/latency/errors, worker queue age and failures, PostgreSQL health and connection saturation, broker health once introduced, integration sync outcomes, storage failures, and audit pipeline failures. Define actionable alerts and ownership before customer use. OpenTelemetry instrumentation alone is not an alerting or monitoring plan.

Backups must be encrypted in transit and at rest, access restricted, and include database plus evidence/object data and required configuration. Restore tests are mandatory and must verify integrity, tenant boundaries, and documented recovery steps. Provisional planning targets are **RPO 24 hours** and **RTO 8 hours** for the initial service; these are not approved production commitments. Product and operations owners must approve targets and validate them through restore/DR exercises before production. Region strategy, point-in-time recovery, backup retention, and disaster recovery topology remain decisions.
