# Secret Remediation Agent

## Detailed Design Specification

Architecture, contracts, operations, and verification

| **Document attribute** | **Value** |
| --- | --- |
| **Version** | 1.1 |
| **Status** | Synchronized with REM-001 through REM-125 |
| **Date** | 2026-10-03 |
| **Baseline** | Implementation-ready product baseline |
| **Trace model** | REM-### -> DD-## -> test_rem_###_* |

**Release decision**

Approved as a build baseline, subject to the explicit assumptions, open decisions, and release gates in this specification. No plaintext secret may enter durable product state.

## Document control and design authority

This design is synchronized to Product Requirements v1.1. DD-01 through DD-16 preserve and expand the prior conceptual design areas; DD-17 through DD-34 add product-grade identity, persistence, APIs, operations, testing, packaging, and readiness. When prose conflicts with a requirement, the requirement and its acceptance criteria govern until an approved change updates both artifacts.

| **Concern** | **Design decision** |
| --- | --- |
| Architecture | Durable control plane plus isolated, ephemeral execution workers; privileged effects only through typed gateways. |
| Persistence | Relational source of truth, object evidence store, transactional outbox, immutable snapshots and audit chain. |
| Secrets | No durable plaintext. Retrieval and use are transient; all boundaries enforce redaction and egress control. |
| Consistency | At-least-once delivery with idempotent consumers/effects, optimistic versions, uniqueness constraints, leases and reconciliation. |
| Tenant security | Server-derived tenant context at every storage, cache, queue, object, metric, and cryptographic boundary. |
| Agent safety | Repository/scanner/model content is untrusted data. Deterministic policy validates every proposed tool action. |

## Architecture context

External actors and systems include scanners, Git hosting, identity provider, HashiCorp Vault, CyberArk, runtime platforms, CI/CD, notification/ticket systems, KMS, and telemetry backends. The product receives only locations/metadata from scanners, obtains plaintext briefly from the pinned source revision when authorized, proposes and validates a secure reference change, and opens a PR. Deployment and credential rotation are separately verified lifecycle phases.

### Logical component flow

- API/UI -> identity and authorization -> ingestion/case services.

- Case service -> repository analyzer + fingerprint service -> immutable evidence snapshot.

- Planner -> effective config + policy + signed playbook + provider capability gateway -> immutable plan revision.

- Workflow orchestrator -> approval gate -> isolated executor -> provider effect ledger -> Git mutation/PR service.

- Deployment evidence -> rotation coordinator -> provider rotation/revocation -> case closure.

- Every transition -> audit/outbox -> events, notifications, metrics, traces and support diagnostics.

### Primary end-to-end sequence

1. Ingest and validate a scanner record; persist canonical metadata and an idempotent source key.

1. Resolve repository/ref to a commit, retrieve the exact region transiently, compute tenant-scoped HMAC, redact context, and discard plaintext.

1. Correlate findings into a case; enumerate refs; analyze ancestry and first introduction with bounded history; return repository, branch, or ambiguous origin plus evidence.

1. Resolve effective config and most-specific signed playbook; discover provider capabilities; construct an immutable plan and risk score.

1. In review mode obtain required approvals; in autonomous mode verify policy eligibility. Re-check drift immediately before effects.

1. Provision/reuse the provider entry, generate a reference-based code change in a sandbox, rescan/build/test, and leak-check every outbound artifact.

1. Create/reconcile a remediation branch and PR, then monitor merge and deployment evidence.

1. After verified deployment, create/execute the approved rotation handoff, validate consumers, revoke old credentials, rescan and close the case.

## DD-01 - Scope, principles, and context

Defines product boundary: remediation of hardcoded secrets found by external scanners; durable state never stores plaintext; PR merge is a human gate; branch deletion is recommendation-only; Git origin is evidence-based and may be ambiguous.

- Interfaces are versioned and typed; unsafe or unsupported states fail closed with a machine-readable reason.

- All durable data is tenant-owned, access-controlled, classified, retained, and free of plaintext secret material.

- Every external effect is policy-authorized, idempotent, observable, auditable, and reconcilable after interruption.

- Implementation evidence is mapped back to the relevant REM requirements and release gates.

## DD-02 - System architecture

Control plane services: API gateway, identity/authorization, ingestion, repository analysis, fingerprint service, policy/config, playbook registry, planning, workflow orchestrator, provider gateway, execution sandbox, Git/PR service, UI, notification, audit/telemetry, and persistence.

- Interfaces are versioned and typed; unsafe or unsupported states fail closed with a machine-readable reason.

- All durable data is tenant-owned, access-controlled, classified, retained, and free of plaintext secret material.

- Every external effect is policy-authorized, idempotent, observable, auditable, and reconcilable after interruption.

- Implementation evidence is mapped back to the relevant REM requirements and release gates.

## DD-03 - Canonical data model

Defines canonical Finding and durable entities with tenant ownership, immutable snapshots, optimistic versions, retention class, and prohibition on plaintext-secret columns.

### Entity model

| **Entity** | **Key fields and invariants** |
| --- | --- |
| Tenant | tenant_id, region, status, policy root; source for all authorization and crypto context. |
| Repository | repo_id, tenant_id, provider, immutable remote identity, default branch, installation binding. |
| Finding | finding_id, source key, repo/ref/SHA/path/coordinates, rule/category/severity/confidence, scan time; no plaintext. |
| FingerprintRef | fingerprint_id, tenant_id, HMAC bytes, algorithm, key_version; access restricted to correlation service. |
| Case/Evidence | case_id, correlation scope, origin result, affected refs, sanitized evidence, algorithm version, immutable evidence digest. |
| Plan/Revision/Step | plan_id, revision, config/playbook/evidence digests, risk, editable fields, expected effects, validations. |
| Approval | actor, role snapshot, decision, scope, revision, expiry, reason; append-only. |
| Execution/Effect | attempts, idempotency key, lease, provider target, request/result digest, compensation state. |
| PR/Deployment/Rotation | provider IDs, target/base/head SHAs, environment evidence, task owner/deadline, status. |
| Audit/Outbox | ordered correlation, actor/workload, action/object/result, safe details, previous/event hashes, publish state. |

## DD-04 - Scanner adapter framework

Versioned adapter SPI: detect schema, validate, redact, normalize, preserve safe source metadata, produce typed errors, and provide fixture-based certification for GitGuardian, Gitleaks, and TruffleHog.

### Adapter interface

```text
interface ScannerAdapter {
  detect(payload, content_type) -> SchemaMatch
  validate(payload, schema_version) -> ValidationReport
  redact(payload) -> SafePayload
  normalize(payload, tenant_context) -> CanonicalFinding[]
}
// Adapter output MUST NOT contain plaintext secret fields.
```

Adapters run before general logging. The ingress proxy caps payload size and assigns a correlation ID; adapter redaction occurs before quarantine or trace enrichment. A registry maps adapter/version to supported native schema versions and fixture sets.

## DD-05 - Secret retrieval and correlation

Just-in-time retrieval from immutable revisions, bounded redacted context, HMAC-SHA-256 fingerprinting via tenant-scoped KMS key, correlation algorithm, and key-version migration.

### Correlation algorithm

- Authorize repository/ref/path/coordinates; resolve ref to SHA and fetch only required objects with bounded expansion.

- Extract matched bytes in the isolated worker, pass over an authenticated memory channel to the fingerprint service, compute HMAC-SHA-256 using tenant key version, and return only the fingerprint ID.

- Persist a redacted typed context window and repository evidence digest; zero/discard buffers and destroy the workspace after the step.

- Correlate only within tenant and repository identity. Fingerprint equality is necessary but origin/location compatibility is evaluated to prevent unrelated reuse from collapsing incorrectly.

## DD-06 - Configuration and policy

Layered effective configuration; immutable platform security controls; JSON Schema; dry-run/explain; signed snapshots; provider preference and feature rollout.

### Effective configuration

```yaml
schemaVersion: remediation.openai.example/v1
metadata: {tenant: acme, revision: 42}
security: {plaintextPersistence: forbidden, autonomousMaxRisk: medium}
providers:
  strategy: [managed, generic_vault, platform_native, runtime_injection, manual]
  hashicorp: {enabled: true, namespace: team-a, authRef: workload:remediator}
  cyberark: {enabled: true, safeTemplate: APP-{application}, authRef: workload:remediator}
playbooks: {registry: signed://org-playbooks, tieBehavior: require_review}
rotation: {defaultMode: manual_post_deploy, stabilizationMinutes: 30}
git: {historyObjectLimit: 250000, staleDays: 180, branchDeletion: recommend_only}
validation: {rescan: required, build: policy_defined, leakGuard: required}
```

Precedence: immutable platform controls > tenant controls > organization > application > repository > authorized run override. Merge is field-specific, not arbitrary YAML overlay. The explain endpoint returns value, source layer, revision, and whether the field is editable.

## DD-07 - Git provenance algorithm

Repository identity normalization, ref discovery, bounded fetch expansion, blob verification, merge-base/ancestry analysis, earliest supported introduction candidates, confidence rules, and explicit AMBIGUOUS/INCOMPLETE_HISTORY outcomes.

### Provenance output

```json
{
  "classification": "REPOSITORY|BRANCH|AMBIGUOUS",
  "candidateRef": "refs/heads/main",
  "candidateCommit": "<sha>",
  "firstObservedCommit": "<sha>",
  "mergeBases": [{"ref": "...", "sha": "..."}],
  "confidence": 0.0,
  "algorithmVersion": "git-origin/1",
  "historyComplete": true,
  "reasonCodes": [],
  "evidenceDigest": "sha256:..."
}
```

- Normalize remote identity; get default branch and complete ref pages from provider API; resolve each ref to SHA.

- Verify the exact fingerprint at each candidate revision. Walk reachable commits with date/generation ordering but decide using graph ancestry, not timestamps alone.

- Compute merge bases and determine whether introduction is reachable from default before divergence. Rank only candidates supported by content and ancestry evidence.

- Return AMBIGUOUS for equal candidates, missing/rewritten/shallow history, object-limit exhaustion, submodule/LFS content unavailable, or confidence below policy. Never infer branch creation metadata that Git does not store.

## DD-08 - Branch targeting and stale policy

One-origin/one-PR selection, branch state inventory, stale metadata, recommendation rules, no automatic deletion, and propagation reporting.

- Interfaces are versioned and typed; unsafe or unsupported states fail closed with a machine-readable reason.

- All durable data is tenant-owned, access-controlled, classified, retained, and free of plaintext secret material.

- Every external effect is policy-authorized, idempotent, observable, auditable, and reconcilable after interruption.

- Implementation evidence is mapped back to the relevant REM requirements and release gates.

## DD-09 - Workflow state machines

Case, plan, execution, PR, deployment, and rotation states; guarded transitions; cancellation; timeouts; compensation; reconciliation and resumability.

### Case states

NEW -> NORMALIZED -> CORRELATED -> ANALYZING -> {PLANNED | NEEDS_REVIEW | MANUAL_REMEDIATION_REQUIRED | FAILED}. PLANNED -> AWAITING_APPROVAL -> APPROVED -> EXECUTING -> PR_OPEN -> MERGED -> DEPLOYMENT_PENDING -> ROTATION_PENDING -> VERIFYING -> CLOSED. Any active state may move to PAUSED, CANCEL_REQUESTED, or FAILED only through guarded transitions. FAILED records retryability and may transition to the last safe predecessor after reconciliation.

### Step states

PENDING -> CLAIMED -> RUNNING -> {SUCCEEDED | RETRY_WAIT | COMPENSATING | FAILED | CANCELLED}. The database owns the transition transaction and outbox write. Workers never infer success solely from a timeout; they reconcile by idempotency key.

## DD-10 - Operator UI and accessibility

Case dashboard/detail, plan editor, diff/evidence, approvals, timelines, saved filters, stale indicators, notifications, accessibility and secure session behavior.

### Primary screens

| **Screen** | **Required content/actions** |
| --- | --- |
| Dashboard | Facets, saved views, SLA/assignment, origin/status/risk/provider/stale fields, accessible status cues. |
| Case detail | Sanitized evidence, provenance graph/table, affected refs, freshness, effective policy and activity timeline. |
| Plan review | Pinned evidence/config/playbook, deterministic/adaptive steps, provider side effects, diff, validations, rollback and editable fields. |
| Approvals | Required roles/quorum, decisions, expiry, SoD, reauthentication and revision invalidation. |
| Lifecycle | PR, merge, deployment per environment, rotation task, health/revoke results and final rescan. |
| Admin | Tenants, roles, integrations, schemas, playbooks, policies, kill switches and audit exports. |

## DD-11 - Playbook and agent engine

Signed playbook schema; deterministic applicability/specificity; pinned versions; adaptive code proposal boundary; typed tools; validation and prompt-injection resistance.

### Playbook contract

```yaml
id: oracle-db-password
version: 3.2.0
digest: sha256:...
applicability: {category: database_password, vendor: oracle}
priority: 800
provider: {capability: managed_database, fallback: generic_vault}
steps: [preflight, provision, transform, validate, open_pr, deployment_gate, rotation_handoff]
adaptiveScope: {files: matched_and_bootstrap_only, tools: [ast_edit], network: none}
approvals: {minimumRisk: medium, roles: [application_operator, security_approver]}
rollback: {preserveOldCredentialUntil: health_verified}
signature: cosign://org-playbooks/oracle-db-password@sha256:...
```

Specificity tuple: explicit repository/application override, exact vendor/type, exact framework/runtime, environment, then numeric priority. Equal top tuples produce AMBIGUOUS_PLAYBOOK. Model output is a proposal; a deterministic validator constrains files, operations, provider, destination, egress, commands and outbound text.

## DD-12 - Secret-provider architecture

Provider capability contract and adapters for HashiCorp Vault and CyberArk; workload auth; naming, collision, ownership, metadata, managed/generic/platform fallbacks, conformance suite.

### Provider SPI

```text
capabilities(context) -> CapabilitySet
resolveName(plan) -> ProviderLocator
lookup(locator, ownership) -> EntryMetadata?
provision(request, idempotencyKey) -> ProvisionResult
reference(entry, consumer) -> RuntimeReference
requestRotation(entry, mode, idempotencyKey) -> RotationJob
rotationStatus(job) -> RotationStatus
revoke(entry/version, idempotencyKey) -> RevokeResult
compensate(effect, ownershipProof) -> CompensationResult
```

| **Adapter** | **Initial supported behaviors** |
| --- | --- |
| HashiCorp | Namespace/mount capability discovery; KV v2 metadata/value write; dynamic/database capability where configured; workload auth; lease renew/revoke; path ownership tags. |
| CyberArk | Safe/account lookup/create metadata; approved platform assignment; workload/app identity; rotation/CPM request and status; ownership metadata; no secret in control-plane responses. |

## DD-13 - Risk and approvals

Risk model, autonomous eligibility, approver role/quorum, separation of duties, plan edit boundaries, approval expiry/revocation and drift invalidation.

### Risk inputs and approval

Risk derives from environment, secret category/privilege, number of consumers/branches, origin confidence, transformation support, provider operation, pre-deploy rotation, protected repository, validation gaps, and playbook provenance. Policy returns autonomous eligibility, approver roles/quorum, expiry, reauthentication, and separation-of-duties constraints. Any diff, target, evidence, config, playbook, capability, or repository-head drift can invalidate approvals.

## DD-14 - Git mutation and PR workflow

Remediation branch creation from pinned base SHA, syntax-aware patch, signed identity, provider reconciliation, branch protection, safe PR template and outbound leak guard.

### PR contract

- Head branch: remediation/<case-short>/<plan-revision>, created from approved base SHA after permission/protection preflight.

- Commit contains only validated changes and generated metadata allowed by policy; identity is the remediation bot and signature/attestation is recorded.

- PR body contains case link/ID, safe finding summary, origin evidence summary, affected refs/count, playbook/provider reference, validation, deployment/rotation/rollback checklist, and owners.

- Outbound leak guard scans branch, patch, commit, title/body/comments, validation logs and provider request payloads before each publish action.

## DD-15 - Consistency and concurrency

Transactional outbox, uniqueness constraints, leases, idempotency records, optimistic locking, drift checks and tenant/repository/case conflict scopes.

- Interfaces are versioned and typed; unsafe or unsupported states fail closed with a machine-readable reason.

- All durable data is tenant-owned, access-controlled, classified, retained, and free of plaintext secret material.

- Every external effect is policy-authorized, idempotent, observable, auditable, and reconcilable after interruption.

- Implementation evidence is mapped back to the relevant REM requirements and release gates.

## DD-16 - Security and data protection

Threat model, data classification/flows, zero-plaintext invariant, encryption, redaction, egress, telemetry leak prevention, retention/privacy, key custody, and incident containment.

### Trust boundaries and invariants

| **Boundary** | **Control** |
| --- | --- |
| Scanner ingress | Size/schema limits, early redaction, tenant authentication, quarantine. |
| Repository content | Untrusted data; sandbox, no implicit instructions, bounded context, deny-by-default network. |
| Plaintext processing | Ephemeral memory only; isolated worker; no durable field; canary leak tests; crash/log safeguards. |
| Model/agent | Proposal only; typed tools; deterministic authorization/validation; no direct credentials. |
| Provider/Git effects | Workload identity, least privilege, idempotency, ownership checks, effect ledger, outbound leak guard. |
| Tenant data | Server-derived scope, row/object/queue/cache/metric isolation, crypto context, negative tests. |

## DD-17 - Identity, RBAC, and tenancy

OIDC/SAML, workload identity, tenant binding, resource authorization, roles/permissions, repository installation scopes, separation of duties, kill-switch authority.

### Role matrix

| **Role** | **Key permissions** |
| --- | --- |
| Viewer | Read accessible cases and sanitized evidence. |
| Analyst | Triage/category/assign; propose policy-safe plan changes. |
| Operator | Run analysis/validation, edit permitted plan fields, request execution. |
| Approver | Approve assigned risk scopes; cannot satisfy prohibited self-approval. |
| Repository admin | Manage Git installation bindings and repository allowlist. |
| Security admin | Manage providers/playbooks/policy/key references and security kill switches. |
| Auditor | Read/export immutable audit and trace evidence. |
| Platform admin | Operate deployment/tenants without default access to customer evidence. |

## DD-18 - Transformation and runtime injection

Framework-aware property mapping, startup injection, AST transforms, restricted command execution, rescan/build/test validation, and unsupported-language review gates.

### Runtime-injection pattern

Prefer preserving the existing configuration key while changing its source: application reads DATABASE_PASSWORD as before; deployment/bootstrap obtains the value through a workload-authenticated provider integration and places it in an in-memory framework configuration source or environment block. Never write a populated properties/.env file. Ensure values do not appear in command arguments, shell tracing, crash reports, diagnostics, or child processes unless explicitly required.

Transformation pipeline: parse supported syntax -> identify exact value node -> insert approved reference/bootstrap -> preserve formatting -> compile/type/lint -> unit/integration -> rescan/leak guard -> compare allowed-file and semantic-diff bounds. Unsupported parsing or ambiguous use forces review.

## DD-19 - Deployment and rotation lifecycle

Provision/reference/PR/merge/deploy/verify/rotate/revoke sequence, manual default, dual-credential options, evidence, ownership, deadlines, rollback and partial-deployment handling.

### Lifecycle gates

| **Phase** | **Entry/exit gate** |
| --- | --- |
| Provision/reference | Owned provider entry exists; code references it; old credential remains valid. |
| PR/merge | Rescan/build/tests pass; approvals valid; PR merged at known SHA. |
| Deploy | Exact artifact/commit is healthy in every declared consumer environment. |
| Rotate | Manual default task approved, or coordinated playbook approved; rollback credential/path available. |
| Stabilize/revoke | Consumer health succeeds for configured window; old credential revoked; post-rotation rescan/verification passes. |

## DD-20 - Reliability and external effects

Timeouts/retries/circuit breakers, effect ledger, idempotency keys, provider reconciliation, compensation, high availability, chaos and crash-point recovery.

### Effect ledger

Before each external mutation, the orchestrator commits an intent with idempotency key and expected ownership. After the call it stores safe request/result digests and external identifiers in the same workflow step transaction where possible. On timeout, it queries by idempotency key/name/ownership before retry. Compensation is permitted only for objects proven to have been created by that plan and not referenced by another plan.

## DD-21 - Audit, observability, and notifications

Tamper-evident audit events, structured logs/metrics/traces, SLOs, safe notification events, diagnostic bundles, alerting and redaction.

### Audit envelope

```json
{
  "eventId": "uuid", "occurredAt": "RFC3339",
  "tenantId": "server-derived", "actor": {"type": "user|workload", "id": "..."},
  "action": "plan.approved", "object": {"type": "plan", "id": "...", "version": 4},
  "result": "success|denied|failure", "correlationId": "...",
  "policyDigest": "...", "safeDetails": {}, "previousHash": "...", "eventHash": "..."
}
```

SLO dashboards cover API/UI availability and latency, ingestion/analysis/plan latency, queue age, workflow success/retries, provider/Git errors and throttling, PR/deployment/rotation aging, and redaction/leak-guard failures. High-cardinality labels exclude repository paths and user content.

## DD-22 - Persistence, migrations, and lifecycle

Relational model, object evidence store, transactional outbox, indexes/partitions, migration strategy, backup/restore, retention/legal hold/deletion.

### Storage and migration

- PostgreSQL-compatible relational store is the system of record; row-level tenant guard plus application authorization and separate service roles.

- Encrypted object storage holds sanitized large evidence, diffs, and diagnostic bundles with tenant prefixes, short-lived signed access and retention lifecycle.

- Transactional outbox publishes domain events after commit. Search/cache are derived and rebuildable, never authorization sources.

- Migrations use expand/migrate/contract. Old and new versions coexist during rolling deployment; backfills are resumable, throttled and observable.

## DD-23 - REST API contracts

OpenAPI 3.1 resources, auth, idempotency, ETags, cursor pagination, typed errors, asynchronous job pattern, filtering and bulk limits.

### Core REST resources

| **Method/path** | **Contract summary** |
| --- | --- |
| POST /api/v1/ingestions | Idempotency-Key required; validates adapter/source; returns 202 Job. |
| GET /api/v1/jobs/{id} | Status, progress, safe errors, created resources, cancellation eligibility. |
| GET /api/v1/cases | Cursor pagination; server-side filters/sort; tenant/repository authorization. |
| GET /api/v1/cases/{id} | Case, sanitized evidence, current plan/lifecycle links; ETag. |
| POST /api/v1/cases/{id}/plans | Create/replan asynchronously from current evidence/config/playbook. |
| PATCH /api/v1/plans/{id} | If-Match required; editable fields only; returns new immutable revision. |
| POST /api/v1/plans/{id}/approvals | Decision applies to exact revision; reauth/SoD enforced. |
| POST /api/v1/plans/{id}/executions | Idempotency-Key; policy and drift checks; returns 202 Job. |
| POST /api/v1/cases/{id}/deployment-evidence | Authenticated CI/CD evidence for exact revision/environment. |
| POST /api/v1/rotation-tasks/{id}/complete | Evidence/result only; provider operation may be separate typed action. |

### Error schema

```json
{"error":{"code":"PLAN_DRIFTED","message":"Plan evidence changed; re-plan required.","correlationId":"...","retryable":false,"fieldErrors":[],"details":{"currentVersion":5}}}
```

## DD-24 - Events, queues, and scale

Versioned event envelope, durable queues, fairness/quotas, leases, dead letters, backpressure, provider rate limiting, horizontal scaling and capacity tests.

### Event envelope and scaling

```json
{"specversion":"1.0","type":"remediation.plan.approved.v1","source":"/tenants/<id>/plans","id":"uuid","time":"RFC3339","subject":"plan/<id>/revision/4","datacontenttype":"application/json","tenant":"bound-claim","data":{"planId":"...","revision":4}}
```

Partition work by tenant and repository hash while enforcing tenant quotas and provider-specific rate limiters. Autoscaling signals include queue age/depth and active sandbox slots. Payloads contain identifiers and safe status only; consumers fetch authorized detail. Dead-letter replay requires an operator permission and reuses the original event ID/idempotency scope.

## DD-25 - Execution sandbox and network

Ephemeral per-tenant job sandboxes, read-only images, resource limits, credential brokering, deny-by-default egress, SSRF defenses, artifact controls.

### Sandbox profile

- Fresh namespace/VM/container per execution; no cross-tenant reuse; non-root; dropped capabilities; seccomp/AppArmor/SELinux policy; read-only root; bounded tmpfs/workspace.

- CPU/memory/PID/disk/time limits; no Docker socket/host mounts; metadata service blocked; DNS and egress through an allowlisting proxy.

- Credential broker issues step-scoped, short-lived tokens only after policy authorization. Secrets are never mounted for untrusted build commands unless explicitly required by a reviewed test profile.

- Outputs pass type/size/path checks, malware and secret scans, and ownership labeling before leaving the boundary.

## DD-26 - Test architecture

Requirement markers, unit/property/schema/contract/integration/E2E/security/chaos/performance/migration/visual suites, synthetic secrets and provider certification.

### Verification matrix

| **Suite** | **Scope/release role** |
| --- | --- |
| Unit/property | Pure normalization, policy, taxonomy, provenance graph, state and naming invariants. |
| Schema/contract | OpenAPI, events, scanner adapters, provider SPI, playbook/config schemas. |
| Integration/E2E | Database/outbox/queue/object store, Git provider, Vault/CyberArk test environments, full PR-to-rotation path. |
| Security | RBAC/tenant negatives, leak canaries, prompt injection, SSRF/egress, sandbox escape, auth/session, supply chain. |
| Reliability/chaos | Crash points, timeouts, duplicate/out-of-order events, provider outages, lease loss, DR/restore. |
| Performance | Large repos/branches, batch/burst/soak, UI/API p95/p99, provider throttling and queue backpressure. |
| Migration/upgrade | N-1 rolling compatibility, backfill resume, rollback/recovery. |
| UI/accessibility | Component/E2E, visual regression, axe-style automation, keyboard and screen-reader manual gate. |

## DD-27 - Ingestion operations

Batch/job status, quarantine, replay, source checkpoints, error reports and operational limits.

- Interfaces are versioned and typed; unsafe or unsupported states fail closed with a machine-readable reason.

- All durable data is tenant-owned, access-controlled, classified, retained, and free of plaintext secret material.

- Every external effect is policy-authorized, idempotent, observable, auditable, and reconcilable after interruption.

- Implementation evidence is mapped back to the relevant REM requirements and release gates.

## DD-28 - Deployment and platform operations

Kubernetes/container topology, managed dependencies, health, autoscaling, HA/DR, Helm/reference manifests, upgrade/rollback, product credential inventory, artifact verification.

### Reference deployment

Supported initial topology is Kubernetes-compatible: stateless API/UI/backend deployments; workflow workers; isolated sandbox executor pool; managed PostgreSQL, durable queue, object storage, KMS; ingress/WAF; private provider/Git connectivity where available; centralized telemetry. Environments use separate identities, keys, data stores and provider targets. Helm/reference manifests encode network policies, pod security, disruption budgets, autoscaling, health probes and resource requests.

## DD-29 - Runbooks and incident response

Global/tenant/provider kill switches; degraded modes; secret-exposure, provider outage, Git outage, queue backlog, data recovery and support escalation runbooks.

### Required runbooks

- Suspected plaintext leak: global mutation stop, key/token revocation, evidence preservation, scope search, tenant communication and mandatory rotation.

- Provider/Git outage: open circuit, pause affected steps, preserve leases/effects, safe retry/reconciliation and backlog recovery.

- Queue/worker incident: throttle ingress, priority/fairness controls, DLQ inspection/replay, duplicate-effect verification.

- Database/object-store recovery: restore, integrity checks, outbox reconciliation, external-effect inventory, controlled resume.

- Kill switch operation and rollback, including authorization, propagation verification and audit review.

## DD-30 - Accessibility and UX validation

WCAG 2.2 AA design tokens, keyboard/screen reader flows, responsive behavior, non-color cues, confirmation patterns and visual regression.

- Interfaces are versioned and typed; unsafe or unsupported states fail closed with a machine-readable reason.

- All durable data is tenant-owned, access-controlled, classified, retained, and free of plaintext secret material.

- Every external effect is policy-authorized, idempotent, observable, auditable, and reconcilable after interruption.

- Implementation evidence is mapped back to the relevant REM requirements and release gates.

## DD-31 - Packaging, supply chain, and licensing

Signed images/charts/tools, SBOM/provenance, dependency and license policy, NOTICE, compatibility matrix and release artifacts.

### Release package

- Signed OCI images and digest-pinned Helm chart/reference manifests.

- Config/playbook/OpenAPI/event schemas, database migration tooling and compatibility matrix.

- SPDX/CycloneDX SBOM, provenance attestations, vulnerability scan results and signatures.

- Product license, third-party notices/attributions, export/compliance status and release notes.

- Administrator/operator/developer/security/DR documentation matching the version.

## DD-32 - Documentation, privacy, and support

Versioned user/admin/API/security/troubleshooting docs, data inventory/residency, analytics consent, support model and safe diagnostics.

- Interfaces are versioned and typed; unsafe or unsupported states fail closed with a machine-readable reason.

- All durable data is tenant-owned, access-controlled, classified, retained, and free of plaintext secret material.

- Every external effect is policy-authorized, idempotent, observable, auditable, and reconcilable after interruption.

- Implementation evidence is mapped back to the relevant REM requirements and release gates.

## DD-33 - Product readiness and rollout

Preview/pilot/GA gates, design-partner constraints, autonomous-action rollout, operational readiness review, risk register and ownership.

### Stage gates

| **Stage** | **Minimum exit criteria** |
| --- | --- |
| Development preview | Core flow works on synthetic repos; no production credentials; REM/DD/test trace skeleton complete. |
| Pilot | Named design partners, scoped scanners/Git/providers/languages, mandatory review mode for unsupported risk, threat model and pen-test plan, support/on-call, rollback and data controls. |
| GA | REM-124 evidence complete; supported matrix/documentation/licensing published; SLO/load/DR results accepted; no critical open risk; upgrade/support lifecycle proven. |

## DD-34 - Traceability governance

Machine-readable REM-DD-test matrix, CI orphan checks, change control, requirement status and verification evidence.

### Trace controls

Store the canonical matrix as machine-readable YAML/JSON in the implementation repository. Each row contains requirement ID, status, design IDs, test IDs, verification type, owner, last evidence URL/digest, and exception. CI rejects unknown/orphan IDs, missing required pytest markers, design references that do not exist, and release candidates with unsatisfied Must requirements unless an authorized exception is linked.

## Requirements-to-design traceability

| **Requirement** | **Title** | **Priority** | **Design sections** |
| --- | --- | --- | --- |
| REM-001 | Supported scanner ingestion | P0 | DD-04, DD-27 |
| REM-002 | Canonical finding schema | P0 | DD-04, DD-27 |
| REM-003 | Input validation and quarantine | P0 | DD-04, DD-27 |
| REM-004 | Batch and multi-branch input | P0 | DD-04, DD-27 |
| REM-005 | Least-privilege repository access | P0 | DD-14, DD-17, DD-25 |
| REM-006 | Transient plaintext extraction | P0 | DD-05, DD-16 |
| REM-007 | HMAC-SHA-256 fingerprint | P0 | DD-05, DD-16 |
| REM-008 | Cross-branch deduplication | P0 | DD-05, DD-16 |
| REM-009 | Git history origin analysis | P0 | DD-07, DD-08 |
| REM-010 | Origin classification | P0 | DD-07, DD-08 |
| REM-011 | Origin URL and evidence | P0 | DD-07, DD-08 |
| REM-012 | One-origin remediation | P0 | DD-07, DD-08 |
| REM-013 | Branch metadata | P1 | DD-08, DD-10 |
| REM-014 | Stale branch recommendation | P1 | DD-08, DD-10 |
| REM-015 | Protected branch safety | P0 | DD-08, DD-10 |
| REM-016 | Context analysis | P0 | DD-11, DD-18 |
| REM-017 | Secret taxonomy | P1 | DD-11, DD-18 |
| REM-018 | Playbook repository | P0 | DD-11, DD-18 |
| REM-019 | Most-specific playbook wins | P0 | DD-11, DD-18 |
| REM-020 | Hybrid deterministic/adaptive behavior | P0 | DD-11, DD-18 |
| REM-021 | Organization YAML configuration | P0 | DD-06 |
| REM-022 | Configuration precedence | P0 | DD-06 |
| REM-023 | Provider abstraction | P0 | DD-12 |
| REM-024 | HashiCorp template | P0 | DD-12 |
| REM-025 | CyberArk template | P0 | DD-12 |
| REM-026 | Provider routing | P0 | DD-12 |
| REM-027 | Managed vault preference | P0 | DD-12 |
| REM-028 | Unmanaged vault fallback | P0 | DD-12 |
| REM-029 | Platform-native fallback | P1 | DD-12 |
| REM-030 | Environment/runtime injection fallback | P1 | DD-12 |
| REM-031 | Prohibit plaintext .env remediation | P0 | DD-12 |
| REM-032 | Centralized property/config externalization | P0 | DD-18 |
| REM-033 | Runtime vault bootstrap/injection | P0 | DD-18 |
| REM-034 | Ephemeral file exception | P1 | DD-18 |
| REM-035 | Action plan generation | P0 | DD-10, DD-13 |
| REM-036 | Review/UI mode | P0 | DD-10, DD-13 |
| REM-037 | Autonomous bot mode | P0 | DD-10, DD-13 |
| REM-038 | Risk-based approval rules | P1 | DD-10, DD-13 |
| REM-039 | Dashboard case view | P1 | DD-10, DD-13 |
| REM-040 | Dashboard plan editing | P1 | DD-10, DD-13 |
| REM-041 | Dedicated remediation branch | P0 | DD-14 |
| REM-042 | One PR per remediation unit | P0 | DD-14 |
| REM-043 | PR safety and content | P0 | DD-14 |
| REM-044 | No default auto-merge | P0 | DD-14 |
| REM-045 | Existing-secret rotation sequencing | P0 | DD-19 |
| REM-046 | Rotation handoff | P0 | DD-19 |
| REM-047 | Pre-PR validation | P0 | DD-18, DD-26 |
| REM-048 | Regression secret scan | P0 | DD-18, DD-26 |
| REM-049 | Dry-run mode | P0 | DD-18, DD-26 |
| REM-050 | Explicit state machine | P0 | DD-09 |
| REM-051 | Idempotent execution | P0 | DD-15, DD-20 |
| REM-052 | Partial-failure recovery | P0 | DD-09, DD-20 |
| REM-053 | Deployment status integration | P1 | DD-19, DD-24 |
| REM-054 | Audit trail | P0 | DD-21 |
| REM-055 | Logging/telemetry redaction | P0 | DD-16, DD-21 |
| REM-056 | LLM context boundary | P0 | DD-11, DD-16, DD-25 |
| REM-057 | Least privilege and separation of duties | P0 | DD-17 |
| REM-058 | Workspace isolation and cleanup | P0 | DD-25 |
| REM-059 | Network egress policy | P1 | DD-25 |
| REM-060 | Config/playbook versioning | P1 | DD-06, DD-34 |
| REM-061 | Provider contract tests | P0 | DD-12, DD-26 |
| REM-062 | Git origin fixture suite | P0 | DD-07, DD-26 |
| REM-063 | Requirement-to-pytest traceability | P0 | DD-26, DD-34 |
| REM-064 | Security-negative test suite | P0 | DD-16, DD-25, DD-26 |
| REM-065 | Extensible plugin architecture | P1 | DD-02, DD-26 |
| REM-066 | Case/report export | P1 | DD-10, DD-21 |
| REM-067 | Secret location drift handling | P0 | DD-05, DD-15 |
| REM-068 | Binary/unsupported file handling | P1 | DD-18 |
| REM-069 | History rewriting out of scope by default | P0 | DD-07, DD-16 |
| REM-070 | Concurrency control | P0 | DD-15 |
| REM-071 | Public API versioning | Must | DD-02, DD-23 |
| REM-072 | OpenAPI contract | Must | DD-23, DD-26 |
| REM-073 | Asynchronous job contract | Must | DD-09, DD-23 |
| REM-074 | Error model | Must | DD-23, DD-16 |
| REM-075 | Event contract | Must | DD-24, DD-26 |
| REM-076 | Pagination and bulk limits | Must | DD-23, DD-24 |
| REM-077 | Core persistence model | Must | DD-03, DD-22 |
| REM-078 | Optimistic concurrency | Must | DD-15, DD-23 |
| REM-079 | Database migrations | Must | DD-22, DD-28 |
| REM-080 | Backup and restore | Must | DD-22, DD-28 |
| REM-081 | Retention and deletion | Must | DD-22, DD-16 |
| REM-082 | Multi-tenant isolation | Must | DD-17, DD-25 |
| REM-083 | Authentication | Must | DD-17 |
| REM-084 | RBAC and resource authorization | Must | DD-17, DD-26 |
| REM-085 | Separation of duties | Must | DD-13, DD-17 |
| REM-086 | Repository authorization | Must | DD-14, DD-17 |
| REM-087 | Session and CSRF security | Must | DD-10, DD-17 |
| REM-088 | UI case dashboard | Must | DD-10 |
| REM-089 | UI case detail | Must | DD-10 |
| REM-090 | UI accessible workflow | Must | DD-10, DD-30 |
| REM-091 | Notifications | Should | DD-21, DD-24 |
| REM-092 | Observability | Must | DD-21, DD-16 |
| REM-093 | Service-level objectives | Must | DD-21, DD-28 |
| REM-094 | Health and readiness | Must | DD-28 |
| REM-095 | Retries and circuit breakers | Must | DD-20, DD-24 |
| REM-096 | Queueing and backpressure | Must | DD-24 |
| REM-097 | Scalability targets | Must | DD-24, DD-28 |
| REM-098 | Repository execution sandbox | Must | DD-25 |
| REM-099 | Egress control | Must | DD-25, DD-16 |
| REM-100 | Artifact and cache safety | Must | DD-16, DD-25 |
| REM-101 | Supply-chain security | Must | DD-28, DD-31 |
| REM-102 | Encryption | Must | DD-16, DD-28 |
| REM-103 | Security threat model | Must | DD-16, DD-25 |
| REM-104 | AI/agent control boundary | Must | DD-11, DD-25 |
| REM-105 | Privacy and data residency | Should | DD-16, DD-32 |
| REM-106 | High availability | Must | DD-20, DD-28 |
| REM-107 | Disaster recovery | Must | DD-28, DD-29 |
| REM-108 | Operational kill switches | Must | DD-17, DD-29 |
| REM-109 | Deployment topology | Must | DD-28 |
| REM-110 | Installation and upgrade | Must | DD-22, DD-28 |
| REM-111 | Configuration and feature rollout | Must | DD-06, DD-28 |
| REM-112 | Secrets for the product itself | Must | DD-16, DD-28 |
| REM-113 | Testing pyramid | Must | DD-26 |
| REM-114 | Provider certification | Must | DD-12, DD-26 |
| REM-115 | Security testing | Must | DD-25, DD-26 |
| REM-116 | Performance testing | Must | DD-24, DD-26 |
| REM-117 | Test data safety | Must | DD-16, DD-26 |
| REM-118 | Packaging and distribution | Must | DD-28, DD-31 |
| REM-119 | Licensing and third-party notices | Must | DD-31 |
| REM-120 | Version support policy | Must | DD-31, DD-32 |
| REM-121 | Documentation set | Must | DD-32 |
| REM-122 | Product telemetry consent | Should | DD-16, DD-32 |
| REM-123 | Supportability | Must | DD-21, DD-29 |
| REM-124 | Release gates | Must | DD-31, DD-33, DD-34 |
| REM-125 | Pilot and general availability criteria | Must | DD-33 |

## Implementation start recommendation

Proceed with implementation in vertical slices: (1) contracts/data/identity foundation, (2) ingestion and synthetic-secret correlation, (3) Git provenance and review-only planning, (4) provider adapters and sandboxed transformation, (5) PR workflow, (6) deployment/rotation lifecycle, and (7) autonomous eligibility after security/reliability evidence. Keep high-risk mutations and rotation behind disabled-by-default feature policy until pilot exit criteria are met.
