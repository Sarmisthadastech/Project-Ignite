# Implementation roadmap

## Status

This is a proposed sequence, not a claim of delivered functionality. The repository currently contains only the static demo. The demo and its assets are to remain unchanged and separate from production application work.

## 1. Foundation

Keep this milestone limited to clean repository structure, backend application skeleton, PostgreSQL connectivity, database migrations, separate restricted runtime and migration roles, basic readiness and health checks, basic observability, CI with real PostgreSQL integration tests, local development instructions, and verification that the existing demo remains unchanged. Do not create tenant-owned application data in this milestone. RabbitMQ/outbox, worker processing, and object storage follow in a subsequent foundation increment. Avoid independent domain microservices.

**Exact acceptance criteria:**

1. Agreed platform directory structure and local startup guide exist; the demo and all assets are byte-for-byte unchanged.
2. Backend starts from a clean checkout and provides liveness plus readiness; readiness fails when PostgreSQL is unavailable.
3. CI provisions and tests against real PostgreSQL, applies baseline migrations to a clean database, and verifies connectivity and role assumptions.
4. Runtime and migration database roles are distinct; runtime privileges are restricted and runtime is neither table owner nor `BYPASSRLS`.
5. Configuration is validated; no secret is committed, logged, or embedded in an image.
6. Basic health, error, database connectivity, and request correlation signals are observable without logging sensitive payloads.
7. Automated CI checks pass, and the demo preservation check passes.

RLS-specific and cross-tenant tests are mandatory before accepting tenant-owned data, but tenant-owned application tables are outside this first milestone. Production migrations should favor forward-compatible changes; a downgrade is required only where explicitly designed and verified safe.

## 2. Identity and tenancy

Select an identity provider behind an abstraction and define session/token invalidation, user and tenant membership lifecycle, suspension/revocation, initial approved RBAC capabilities, resource-level authorization, and service identities. Implement principal resolution, validated membership, tenant-aware authorization, PostgreSQL RLS policies, and tenant-scoped audit events before storing tenant-owned application data. For every tenant-owned table require `ENABLE` and `FORCE ROW LEVEL SECURITY`, fail-closed `USING` and `WITH CHECK` policies, transaction-local context, and restricted runtime roles. Test under the actual runtime role.

**Exit criteria:** authorization is enforced server-side; real PostgreSQL CI tests cover RLS and FORCE, restricted runtime permissions, missing context, cross-tenant reads/writes, pooled-connection context leakage, migrations, and audit permissions; API paths fail closed; sessions/tokens are invalidated on membership suspension/revocation. Tenant-owned data remains prohibited until these pass.

## 3. Projects and applications

Implement tenant-scoped project and application registration, ownership/membership rules, lifecycle state, and audit history. Establish stable identifiers and domain interfaces for later integrations without embedding provider-specific behavior in core records.

**Exit criteria:** authorized users can create, read, update, and archive the agreed initial project/application entities; tenant boundaries and lifecycle rules have automated coverage.

## 4. Integrations and ingestion

Choose the first external systems based on user value and available permissions. Implement connector interfaces, tenant-scoped encrypted credential custody, least-privilege OAuth scopes, credential rotation/revocation and audited usage, consent, sync status, incremental ingestion, webhook validation if applicable, idempotency, rate-limit handling, retries, and dead-letter operations. Preserve provider identifiers and source context. Never include secret values in logs or queue messages.

**Exit criteria:** at least one approved integration can be connected and disconnected; ingestion is replayable and tenant-scoped; partial, stale, and failed sync states are distinguishable; sensitive credentials never appear in logs or job payloads.

## 5. Evidence and traceability

Define canonical evidence metadata and lineage. Hash clearly specified content bytes with SHA-256; keep versioned evidence history and create correction/supersession records instead of silent edits. Store large artifacts in selected S3-compatible storage and metadata in PostgreSQL. Do not promise universal WORM immutability until storage controls are verified. Link records only where supported by source evidence. Establish retention, deletion, and legal-hold policies before production use.

**Exit criteria:** an authorized user can inspect evidence source, version, timestamps, checksum, and transformation history; derived records link to inputs; object access is authorized and tenant scoped; deletion and retention behavior is verified.

## 6. Quality intelligence

Implement deterministic, explainable rules and traceability/coverage queries before introducing generative recommendations. Every finding must identify its inputs, rule/version, freshness, and known limitations. Distinguish missing evidence from evidence of failure.

**Exit criteria:** initial findings are reproducible from recorded evidence; supporting records can be inspected; stale or incomplete sources are visible; rule changes are versioned and reviewed.

## 7. Frontend

Build the enterprise Next.js experience against stable API contracts. Provide authentication, tenant/project selection based on authorized memberships, integration setup, evidence browsing, and quality views. Keep the existing direct-upload demo unchanged and do not silently convert it into the production frontend.

**Exit criteria:** primary workflows are usable with keyboard and accessible semantics; the UI handles loading, partial, stale, and error states; server authorization remains authoritative; frontend tests cover key flows.

## 8. Controlled AI engineering

Add AI assistance only after evidence, authorization, audit, and deterministic quality foundations exist. Begin with read-only, evidence-grounded summaries and test suggestions. Add write-capable tools only with scoped permissions and approvals bound to exact action, parameters, actor, tenant, target, and expiry; revalidate authorization before execution and reject replay. Require audit records, rollback/recovery design, and evaluation.

**Exit criteria:** retrieval respects tenant and object permissions; outputs cite supporting evidence and are labeled as generated; evaluations cover quality, unsafe actions, prompt injection, and tenant isolation; consequential actions require and record human approval.

## Cross-cutting decisions and gates

Before production handling of customer data, resolve identity provider, hosting/region, compliance obligations, data residency, retention/deletion, backup/restore objectives, object-store/key management, integration credential storage, audit retention, and AI provider data handling. Each roadmap phase requires security review and operational readiness appropriate to the data and actions it introduces. Timeline and staffing estimates are intentionally omitted until scope and service constraints are known.
