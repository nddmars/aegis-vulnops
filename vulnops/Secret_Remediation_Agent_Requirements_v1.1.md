# Secret Remediation Agent

## Product Requirements Specification

Production-grade, implementation-ready requirements

| **Document attribute** | **Value** |
| --- | --- |
| **Version** | 1.1 |
| **Status** | Approved baseline with explicit product-readiness gates |
| **Date** | 2026-10-03 |
| **Baseline** | Implementation-ready product baseline |
| **Trace model** | REM-### -> DD-## -> test_rem_###_* |

**Release decision**

Approved as a build baseline, subject to the explicit assumptions, open decisions, and release gates in this specification. No plaintext secret may enter durable product state.

## Document control and synchronization

This revision preserves the exact REM-001 through REM-070 requirement titles, priorities, statements, and acceptance criteria from the supplied v1.0 requirements document, updates their design traces to the expanded architecture, and adds REM-071 through REM-125 for productization gaps. The corresponding design uses DD-01 through DD-34. Every requirement has acceptance criteria, design trace, and a deterministic pytest naming convention.

Source-file note: the supplied files named v1.0 and v0.9 were byte-for-byte identical requirements documents. No design DOCX was present in the provided folder, so the v1.1 design was reconstructed from the complete referenced discussion, the recorded DD-01 through DD-16 baseline summary, and this synchronized requirements baseline.

| **Item** | **Decision** |
| --- | --- |
| Product boundary | Remediate hardcoded source secrets reported by supported scanners; avoid duplicate migration when an approved reference is already in use. |
| Plaintext invariant | Plaintext may exist only transiently in bounded execution memory; never in durable state, UI, logs, events, exports, PRs, or telemetry. |
| Git origin | Evidence-based inference only; ambiguous or incomplete history blocks autonomous execution. |
| Branch deletion | Recommendation-only. The remediation product never automatically deletes branches. |
| Rotation | Post-deployment manual handoff is the default; coordinated automation requires an approved playbook and verification. |
| Release posture | Sufficient to begin implementation; GA remains gated by REM-124 and REM-125. |

## Purpose and outcomes

- Normalize scanner findings from GitGuardian, Gitleaks, and TruffleHog without accepting plaintext in the input contract.

- Correlate duplicate branch findings safely using tenant-scoped HMAC fingerprints and Git evidence.

- Select organization-approved playbooks and secure destinations deterministically, while allowing constrained context-aware code transformation.

- Create reviewable, validated code changes and one targeted pull request per origin case by default.

- Coordinate deployment and rotation without breaking running consumers.

- Operate as a secure, multi-tenant, observable, supportable product rather than a one-off automation script.

## Actors and responsibility

| **Actor** | **Primary responsibility** |
| --- | --- |
| Security analyst | Triage evidence, taxonomy, policy exceptions, and security outcomes. |
| Application operator | Review plans/diffs, validate application context, own remediation rollout. |
| Approver | Authorize risk-scoped execution with separation of duties. |
| DevOps/SRE | Deploy changes, verify consumers, execute or oversee rotation and rollback. |
| Repository admin | Install/scopes Git integration, branch protection and reviewers. |
| Platform/security admin | Manage tenants, RBAC, providers, immutable controls, keys and kill switches. |
| Auditor | Read immutable evidence and reports without accessing secret material. |

## Requirement conventions

- Must: required for the first production release unless an authorized exception is recorded.

- Should: expected product behavior; deferral requires product/security approval and documented limitation.

- Each test file follows tests/requirements/test_rem_NNN_<slug>.py and carries pytest.mark.rem_NNN.

- Acceptance criteria are normative. Examples and rationale are explanatory unless explicitly marked as a requirement.

## Open implementation decisions

The baseline is build-ready, but the following values must be selected during architecture inception and frozen in versioned ADRs: supported Git provider(s) for v1, initial cloud/Kubernetes target, tenant deployment model, numeric SLO/RPO/RTO/capacity targets, supported language/framework matrix, exact HashiCorp and CyberArk product/API versions, notification/ticket connectors, data residency regions, and commercial/open-source licensing model. These are bounded choices, not missing control requirements.

## 1. Input, normalization, and correlation

### REM-001 - Supported scanner ingestion

**Priority: **P0**   |   Design: **DD-04, DD-27**   |   Pytest: **test_rem_001_supported_scanner_ingestion.py

Ingest findings from GitGuardian, Gitleaks, TruffleHog, and a generic adapter without requiring plaintext secret values.

**Acceptance criteria**

- Each supported sample converts to canonical schema.

- Plaintext secret is not required in input.

- Unsupported formats fail with typed errors before mutation.

### REM-002 - Canonical finding schema

**Priority: **P0**   |   Design: **DD-04, DD-27**   |   Pytest: **test_rem_002_canonical_finding_schema.py

Normalize repository URL, branch, file path, start/end line and column, scanner rule ID, normalized category, severity, confidence, scan time, scanner name, and optional commit SHA.

**Acceptance criteria**

- Required location fields survive normalization.

- Missing category becomes TBD/unknown.

- Optional fields can be null without downstream failure.

### REM-003 - Input validation and quarantine

**Priority: **P0**   |   Design: **DD-04, DD-27**   |   Pytest: **test_rem_003_input_validation_and_quarantine.py

Reject or quarantine malformed, ambiguous, or unsafe records before repository or vault operations.

**Acceptance criteria**

- Invalid URI, path traversal, invalid ranges, and invalid branch references are detected.

- Rejected records contain reason codes.

- Rejected records cause zero mutating side effects.

### REM-004 - Batch and multi-branch input

**Priority: **P0**   |   Design: **DD-04, DD-27**   |   Pytest: **test_rem_004_batch_and_multi_branch_input.py

Support one or more repositories and many findings/branches in a batch while preserving per-repository isolation.

**Acceptance criteria**

- 100+ branch findings can be normalized without losing branch identity.

- Failure in one repository does not corrupt another repository case.

- Correlation occurs only within permitted repository scope.

### REM-005 - Least-privilege repository access

**Priority: **P0**   |   Design: **DD-14, DD-17, DD-25**   |   Pytest: **test_rem_005_least_privilege_repository_access.py

Clone/fetch repositories using least-privilege credentials and isolated workspaces.

**Acceptance criteria**

- Analysis starts read-only.

- Credentials never appear in clone URL/logs/artifacts.

- Checkout failure is recoverable with no mutation.

### REM-006 - Transient plaintext extraction

**Priority: **P0**   |   Design: **DD-05, DD-16**   |   Pytest: **test_rem_006_transient_plaintext_extraction.py

Retrieve a hardcoded secret from source only transiently when required for fingerprinting/vaulting/transformation.

**Acceptance criteria**

- Plaintext is not stored in DB, logs, telemetry, PRs, test artifacts, or persistent temp files.

- Extraction is limited to identified source span/context.

- Secret references are released after required operations.

### REM-007 - HMAC-SHA-256 fingerprint

**Priority: **P0**   |   Design: **DD-05, DD-16**   |   Pytest: **test_rem_007_hmac_sha_256_fingerprint.py

Compute a keyed HMAC-SHA-256 fingerprint and use it as the primary correlation identity instead of an unkeyed hash.

**Acceptance criteria**

- Same key+secret produces same fingerprint.

- Different test secrets produce different fingerprints.

- HMAC key is external to YAML/source.

- Only fingerprint is persisted.

### REM-008 - Cross-branch deduplication

**Priority: **P0**   |   Design: **DD-05, DD-16**   |   Pytest: **test_rem_008_cross_branch_deduplication.py

Group the same secret across branches using HMAC plus repository/context metadata.

**Acceptance criteria**

- Same fingerprint occurrences group together.

- Different fingerprints never merge solely by line/rule.

- Affected branch list/count are retained.

### REM-009 - Git history origin analysis

**Priority: **P0**   |   Design: **DD-07, DD-08**   |   Pytest: **test_rem_009_git_history_origin_analysis.py

Analyze Git history deterministically to locate the earliest observable introduction of a correlated secret.

**Acceptance criteria**

- Earliest reachable introducing commit is identified for synthetic fixtures.

- Origin commit SHA and evidence are stored.

- Unprovable origin is marked ambiguous, never guessed.

### REM-010 - Origin classification

**Priority: **P0**   |   Design: **DD-07, DD-08**   |   Pytest: **test_rem_010_origin_classification.py

Classify secret origin as default/repository origin, branch-specific origin, or ambiguous using commit-graph evidence.

**Acceptance criteria**

- Default-branch inherited secret is repository/default origin.

- Feature-only introduction is branch-specific when supported by history.

- Ambiguous histories retain ambiguity reason.

### REM-011 - Origin URL and evidence

**Priority: **P0**   |   Design: **DD-07, DD-08**   |   Pytest: **test_rem_011_origin_url_and_evidence.py

Record origin URL appropriate to classification plus origin branch/commit and evidence.

**Acceptance criteria**

- Repository-origin URL identifies repository/default target.

- Branch-specific URL identifies branch/commit target.

- No URL contains credentials or secret material.

### REM-012 - One-origin remediation

**Priority: **P0**   |   Design: **DD-07, DD-08**   |   Pytest: **test_rem_012_one_origin_remediation.py

Remediate the origin rather than independently patching every inherited branch occurrence.

**Acceptance criteria**

- Repository-origin finding targets default branch once.

- Branch-specific finding targets origin branch once.

- Descendant branches are reported but not patched by default.

### REM-013 - Branch metadata

**Priority: **P1**   |   Design: **DD-08, DD-10**   |   Pytest: **test_rem_013_branch_metadata.py

Collect branch activity metadata for operational reporting.

**Acceptance criteria**

- Capture last commit time, protected/default flag, merge status when determinable, and age/first-seen approximation.

- Unavailable fields are unknown, not invented.

- Collection is read-only.

### REM-014 - Stale branch recommendation

**Priority: **P1**   |   Design: **DD-08, DD-10**   |   Pytest: **test_rem_014_stale_branch_recommendation.py

Identify potentially stale branches using configurable policy and recommend lifecycle review rather than remediation PR when appropriate.

**Acceptance criteria**

- Inactivity threshold is configurable.

- UI can filter/highlight stale candidates.

- Evidence is shown.

- No branch is automatically deleted.

### REM-015 - Protected branch safety

**Priority: **P0**   |   Design: **DD-08, DD-10**   |   Pytest: **test_rem_015_protected_branch_safety.py

Never recommend deletion of default/protected branches and never bypass branch protection.

**Acceptance criteria**

- Default/protected branches excluded from delete recommendations.

- Agent cannot force-push protected branches.

- Repository protections remain authoritative.

## 2. Git provenance and branch targeting

### REM-016 - Context analysis

**Priority: **P0**   |   Design: **DD-11, DD-18**   |   Pytest: **test_rem_016_context_analysis.py

Inspect bounded source/config context to understand how the secret is consumed.

**Acceptance criteria**

- Consumer type/framework/property key candidates are recorded.

- Persisted context replaces plaintext with placeholder.

- Context window is configurable.

### REM-017 - Secret taxonomy

**Priority: **P1**   |   Design: **DD-11, DD-18**   |   Pytest: **test_rem_017_secret_taxonomy.py

Maintain extensible normalized categories while preserving scanner rule IDs.

**Acceptance criteria**

- Rule ID and normalized category are separate.

- Unknown stays TBD.

- Org mappings can extend categories without core code change.

### REM-018 - Playbook repository

**Priority: **P0**   |   Design: **DD-11, DD-18**   |   Pytest: **test_rem_018_playbook_repository.py

Support organization-specific remediation playbooks for secret/application types such as Oracle DB credentials.

**Acceptance criteria**

- Playbooks are versioned/schema validated.

- Playbook can define provider, destination convention, mandatory steps, transformations, validation and rotation guidance.

- Case records playbook version/digest.

### REM-019 - Most-specific playbook wins

**Priority: **P0**   |   Design: **DD-11, DD-18**   |   Pytest: **test_rem_019_most_specific_playbook_wins.py

Select the most specific matching playbook deterministically.

**Acceptance criteria**

- Oracle credential outranks generic database credential.

- Tie/precedence is deterministic.

- No-match routes to configured fallback/review.

### REM-020 - Hybrid deterministic/adaptive behavior

**Priority: **P0**   |   Design: **DD-11, DD-18**   |   Pytest: **test_rem_020_hybrid_deterministic_adaptive_behavior.py

Mandatory security/procedure/provider rules are deterministic; code/config transformation may adapt within guardrails.

**Acceptance criteria**

- Adaptive logic cannot override mandatory playbook controls.

- Plan labels policy decisions vs adaptive decisions.

- Output patch is validated before mutation.

### REM-021 - Organization YAML configuration

**Priority: **P0**   |   Design: **DD-06**   |   Pytest: **test_rem_021_organization_yaml_configuration.py

Provide versioned remediation-config.yaml for providers, strategic defaults, application/repository overrides, routing, strategies, approvals, rotation, branch policy, validation and security.

**Acceptance criteria**

- YAML validates against schema.

- Invalid enum/unknown critical key fails safely.

- No provider credential or HMAC key is embedded.

### REM-022 - Configuration precedence

**Priority: **P0**   |   Design: **DD-06**   |   Pytest: **test_rem_022_configuration_precedence.py

Apply deterministic precedence: mandatory org policy > secret/playbook rule > application/repository override when allowed > strategic default > approved fallback.

**Acceptance criteria**

- Conflicts resolve identically across runs.

- Selection reason/rule is recorded.

- Lower precedence cannot weaken mandatory policy.

### REM-023 - Provider abstraction

**Priority: **P0**   |   Design: **DD-12**   |   Pytest: **test_rem_023_provider_abstraction.py

Use a common vault provider interface rather than hard-coded provider workflows.

**Acceptance criteria**

- Interface supports capabilities, destination validation, create/write, reference generation, idempotency lookup, and optional lifecycle hooks.

- Unsupported capability returns typed result.

- Mock provider passes contract suite.

### REM-024 - HashiCorp template

**Priority: **P0**   |   Design: **DD-12**   |   Pytest: **test_rem_024_hashicorp_template.py

Ship a HashiCorp Vault provider/config template with org-specific placeholders.

**Acceptance criteria**

- Endpoint/namespace/mount/path/auth reference are configurable.

- No credentials embedded.

- Template validates offline.

### REM-025 - CyberArk template

**Priority: **P0**   |   Design: **DD-12**   |   Pytest: **test_rem_025_cyberark_template.py

Ship a CyberArk provider/config template with org-specific placeholders.

**Acceptance criteria**

- Endpoint/safe/account mapping/auth reference/naming conventions are configurable.

- No credentials embedded.

- Template validates offline.

### REM-026 - Provider routing

**Priority: **P0**   |   Design: **DD-12**   |   Pytest: **test_rem_026_provider_routing.py

Choose HashiCorp, CyberArk, or another approved provider using secret/application rules and strategic defaults.

**Acceptance criteria**

- Specific routing beats generic default.

- Application preference applies only if org policy allows.

- Plan records provider and rationale.

## 3. Configuration and playbooks

### REM-027 - Managed vault preference

**Priority: **P0**   |   Design: **DD-12**   |   Pytest: **test_rem_027_managed_vault_preference.py

Prefer provider-native managed secret/account when supported by the playbook/provider.

**Acceptance criteria**

- Capability is checked before selection.

- Managed strategy outranks generic storage.

- Persist only provider object/reference, not value.

### REM-028 - Unmanaged vault fallback

**Priority: **P0**   |   Design: **DD-12**   |   Pytest: **test_rem_028_unmanaged_vault_fallback.py

Use generic/unmanaged enterprise vault storage when managed integration is unavailable or inappropriate.

**Acceptance criteria**

- Fallback follows approved path/naming policy.

- Plan marks lifecycle/rotation as unmanaged.

- No plaintext source-controlled artifact is generated.

### REM-029 - Platform-native fallback

**Priority: **P1**   |   Design: **DD-12**   |   Pytest: **test_rem_029_platform_native_fallback.py

Allow approved platform-native stores such as Kubernetes/CI protected secrets after enterprise vault strategies.

**Acceptance criteria**

- Disabled unless configured.

- Reference is non-plaintext.

- Reason higher-priority strategy was unavailable is recorded.

### REM-030 - Environment/runtime injection fallback

**Priority: **P1**   |   Design: **DD-12**   |   Pytest: **test_rem_030_environment_runtime_injection_fallback.py

Allow environment/runtime injection as a lower-priority approved strategy.

**Acceptance criteria**

- Policy controls use.

- Repository changes contain variable/reference names only.

- No plaintext .env is committed.

### REM-031 - Prohibit plaintext .env remediation

**Priority: **P0**   |   Design: **DD-12**   |   Pytest: **test_rem_031_prohibit_plaintext_env_remediation.py

Never automatically create or commit a plaintext .env file.

**Acceptance criteria**

- Sentinel secret cannot appear in generated .env tracked content.

- File-based runtime materialization, if explicitly allowed, is ephemeral and ignored by VCS.

- Unsafe fallback is blocked.

### REM-032 - Centralized property/config externalization

**Priority: **P0**   |   Design: **DD-18**   |   Pytest: **test_rem_032_centralized_property_config_externalization.py

When many call sites read secrets through a common property/config layer, prefer changing the central configuration boundary instead of application call sites.

**Acceptance criteria**

- Agent identifies shared property/config keys.

- Consumers remain unchanged when framework permits.

- Plan lists the single integration point and impacted keys.

### REM-033 - Runtime vault bootstrap/injection

**Priority: **P0**   |   Design: **DD-18**   |   Pytest: **test_rem_033_runtime_vault_bootstrap_injection.py

Support startup/runtime integration that resolves vault values into application configuration/environment without persistent plaintext on disk.

**Acceptance criteria**

- Bootstrap uses provider references/workload identity.

- Original application code need not change when config abstraction supports injection.

- Generated artifacts do not contain original secret.

## 4. Providers and remediation strategies

### REM-034 - Ephemeral file exception

**Priority: **P1**   |   Design: **DD-18**   |   Pytest: **test_rem_034_ephemeral_file_exception.py

If a legacy runtime requires a properties file, permit only policy-approved ephemeral materialization with restrictive permissions and cleanup.

**Acceptance criteria**

- Requires explicit policy/approval.

- File is outside VCS and workspace retention.

- Permissions/cleanup are validated.

### REM-035 - Action plan generation

**Priority: **P0**   |   Design: **DD-10, DD-13**   |   Pytest: **test_rem_035_action_plan_generation.py

Generate a structured action plan before any mutation.

**Acceptance criteria**

- Plan includes origin, branches, playbook, provider, strategy, destination reference, transformations, validation, PR target, deployment/rotation handoff and compensation.

- Plan is machine- and human-readable.

- Every step has rationale/status/dependencies.

### REM-036 - Review/UI mode

**Priority: **P0**   |   Design: **DD-10, DD-13**   |   Pytest: **test_rem_036_review_ui_mode.py

In review mode, stop after plan generation until authorized approval.

**Acceptance criteria**

- No vault/repo mutation before approval.

- Operator can approve/reject or edit allowed fields.

- Plan edits are audited and revalidated.

### REM-037 - Autonomous bot mode

**Priority: **P0**   |   Design: **DD-10, DD-13**   |   Pytest: **test_rem_037_autonomous_bot_mode.py

In autonomous mode, execute eligible plans end-to-end through PR creation.

**Acceptance criteria**

- Only policy-eligible cases auto-execute.

- Uncertainty/risk can force review mode.

- Default autonomous authority ends at PR creation.

### REM-038 - Risk-based approval rules

**Priority: **P1**   |   Design: **DD-10, DD-13**   |   Pytest: **test_rem_038_risk_based_approval_rules.py

Require human review based on category, repository sensitivity, confidence, provider, transformation, origin ambiguity, or playbook.

**Acceptance criteria**

- Rules are declarative.

- Most restrictive applicable rule wins.

- Decision and rule IDs are audited.

### REM-039 - Dashboard case view

**Priority: **P1**   |   Design: **DD-10, DD-13**   |   Pytest: **test_rem_039_dashboard_case_view.py

Provide an operations dashboard showing analysis, action plan, status and recommendations.

**Acceptance criteria**

- Displays origin, affected branches/count, stale indicators, provider/playbook/strategy, validation and approval state.

- Filters include repo/category/status/provider/origin/stale.

- Never displays plaintext secrets.

### REM-040 - Dashboard plan editing

**Priority: **P1**   |   Design: **DD-10, DD-13**   |   Pytest: **test_rem_040_dashboard_plan_editing.py

Allow authorized changes to policy-permitted plan fields.

**Acceptance criteria**

- Mandatory fields are immutable.

- Choices are constrained to approved options.

- Modified plan must pass validation before approval.

### REM-041 - Dedicated remediation branch

**Priority: **P0**   |   Design: **DD-14**   |   Pytest: **test_rem_041_dedicated_remediation_branch.py

Create a dedicated collision-safe remediation branch; never commit directly to origin/default branch.

**Acceptance criteria**

- Branch naming configurable.

- No force push of existing branch.

- Reruns safely reuse/supersede according to idempotency.

### REM-042 - One PR per remediation unit

**Priority: **P0**   |   Design: **DD-14**   |   Pytest: **test_rem_042_one_pr_per_remediation_unit.py

Consolidate inherited occurrences into one PR against the origin target; batch compatible findings per configured grouping policy.

**Acceptance criteria**

- No PR-per-inherited-branch explosion.

- PR enumerates affected branches/count.

- Branch-specific origin targets its origin branch.

### REM-043 - PR safety and content

**Priority: **P0**   |   Design: **DD-14**   |   Pytest: **test_rem_043_pr_safety_and_content.py

PR title/body/diff/comments/attachments must not reveal plaintext or reversible secret material.

**Acceptance criteria**

- Pre-submit redaction/secret scan runs.

- PR includes summary, affected branches, validation, deployment/rotation instructions.

- Sentinel secret absent from PR payload.

### REM-044 - No default auto-merge

**Priority: **P0**   |   Design: **DD-14**   |   Pytest: **test_rem_044_no_default_auto_merge.py

Do not automatically merge remediation PRs in the default design.

**Acceptance criteria**

- Required reviews/status checks are not bypassed.

- Agent cannot impersonate approver.

- Merge remains governed by repository policy.

### REM-045 - Existing-secret rotation sequencing

**Priority: **P0**   |   Design: **DD-19**   |   Pytest: **test_rem_045_existing_secret_rotation_sequencing.py

Do not rotate an existing hardcoded credential before the remediated path is deployed unless a playbook explicitly proves it safe.

**Acceptance criteria**

- Default is post-deployment rotation.

- Plan marks exposed credential rotation required.

- Pre-deployment rotation requires explicit policy/playbook and approval.

### REM-046 - Rotation handoff

**Priority: **P0**   |   Design: **DD-19**   |   Pytest: **test_rem_046_rotation_handoff.py

Provide DevOps/operations a non-secret rotation handoff when rotation is not safely automated.

**Acceptance criteria**

- Handoff identifies provider object/credential identity and timing.

- States include pending/completed/waived-with-reason.

- No plaintext required.

### REM-047 - Pre-PR validation

**Priority: **P0**   |   Design: **DD-18, DD-26**   |   Pytest: **test_rem_047_pre_pr_validation.py

Run configured syntax/build/tests plus secret rescan before PR creation.

**Acceptance criteria**

- Validation executes in isolated environment.

- Targeted hardcoded occurrence is absent after patch.

- Failure blocks autonomous normal PR; policy may permit clearly marked draft.

### REM-048 - Regression secret scan

**Priority: **P0**   |   Design: **DD-18, DD-26**   |   Pytest: **test_rem_048_regression_secret_scan.py

Detect persistence or accidental introduction of secrets in generated changes.

**Acceptance criteria**

- Original fingerprint no longer present in target content.

- New high-confidence secret finding blocks execution.

- Results persist without plaintext.

### REM-049 - Dry-run mode

**Priority: **P0**   |   Design: **DD-18, DD-26**   |   Pytest: **test_rem_049_dry_run_mode.py

Support full analysis/plan simulation without vault/repository mutations.

**Acceptance criteria**

- Same decision logic as execution for same input/config.

- No vault write/push/PR/rotation occurs.

- Output can feed UI and CI.

### REM-050 - Explicit state machine

**Priority: **P0**   |   Design: **DD-09**   |   Pytest: **test_rem_050_explicit_state_machine.py

Track remediation cases through explicit, validated states.

**Acceptance criteria**

- Invalid transitions rejected.

- Transitions timestamped/audited.

- Failure states preserve resume information.

## 5. Modes, approvals, PR, and lifecycle

### REM-051 - Idempotent execution

**Priority: **P0**   |   Design: **DD-15, DD-20**   |   Pytest: **test_rem_051_idempotent_execution.py

Repeated processing must not create duplicate vault objects, commits, branches, or PRs.

**Acceptance criteria**

- Stable non-secret idempotency key is used.

- Successful prior steps are discovered on retry.

- Duplicate event maps to same/superseding case.

### REM-052 - Partial-failure recovery

**Priority: **P0**   |   Design: **DD-09, DD-20**   |   Pytest: **test_rem_052_partial_failure_recovery.py

Define compensation/resume behavior for partial failures such as vault success followed by Git/PR failure.

**Acceptance criteria**

- Step state is durable before dependent mutation.

- Retry does not duplicate vault object.

- Unsafe rollback yields recoverable manual-attention state with instructions.

### REM-053 - Deployment status integration

**Priority: **P1**   |   Design: **DD-19, DD-24**   |   Pytest: **test_rem_053_deployment_status_integration.py

Accept authenticated CI/CD/webhook/operator status updates after merge.

**Acceptance criteria**

- Duplicate/out-of-order events are idempotent.

- Case can reach deployed and rotation states.

- No secret value needed.

### REM-054 - Audit trail

**Priority: **P0**   |   Design: **DD-21**   |   Pytest: **test_rem_054_audit_trail.py

Audit decisions and mutations without storing plaintext secrets.

**Acceptance criteria**

- Capture case, actor/agent, timestamps, policy/playbook versions, repo/branch/commit, provider object IDs and results.

- Audit is append-oriented/tamper-evident where supported.

- No secret/auth token values.

### REM-055 - Logging/telemetry redaction

**Priority: **P0**   |   Design: **DD-16, DD-21**   |   Pytest: **test_rem_055_logging_telemetry_redaction.py

Protect logs, traces, metrics labels, exceptions and prompts from secret leakage.

**Acceptance criteria**

- Central redaction filters sentinel secrets.

- Raw source-line dumps disabled by default.

- Leakage tests assert zero sentinel occurrence.

### REM-056 - LLM context boundary

**Priority: **P0**   |   Design: **DD-11, DD-16, DD-25**   |   Pytest: **test_rem_056_llm_context_boundary.py

If an LLM performs adaptive transformation, never send plaintext secrets; use placeholders and bounded context.

**Acceptance criteria**

- Prompt contains placeholder, not sentinel secret.

- Output is treated as untrusted proposal and validated.

- LLM cannot directly invoke privileged vault/Git mutation.

### REM-057 - Least privilege and separation of duties

**Priority: **P0**   |   Design: **DD-17**   |   Pytest: **test_rem_057_least_privilege_and_separation_of_duties.py

Scope repository, vault, CI and PR permissions to minimum necessary privileges.

**Acceptance criteria**

- Analysis identity cannot mutate.

- Execution identity cannot bypass merge controls by default.

- Vault identity limited to approved namespaces/safes.

### REM-058 - Workspace isolation and cleanup

**Priority: **P0**   |   Design: **DD-25**   |   Pytest: **test_rem_058_workspace_isolation_and_cleanup.py

Run each case in isolated workspace and clean it on terminal state.

**Acceptance criteria**

- Concurrent cases cannot read each other.

- Restrictive filesystem permissions.

- Cleanup applies on success/failure and retained artifacts contain no plaintext.

### REM-059 - Network egress policy

**Priority: **P1**   |   Design: **DD-25**   |   Pytest: **test_rem_059_network_egress_policy.py

Restrict outbound destinations to configured Git, vault, build/package and model endpoints.

**Acceptance criteria**

- Unapproved egress denied/logged.

- Analysis/dry-run can use stricter profile.

- Endpoints derive from validated config.

### REM-060 - Config/playbook versioning

**Priority: **P1**   |   Design: **DD-06, DD-34**   |   Pytest: **test_rem_060_config_playbook_versioning.py

Version and digest configuration/playbooks used by every case.

**Acceptance criteria**

- Historical case identifies exact versions.

- Breaking schema versions reject or migrate explicitly.

- Plan/audit includes digests.

### REM-061 - Provider contract tests

**Priority: **P0**   |   Design: **DD-12, DD-26**   |   Pytest: **test_rem_061_provider_contract_tests.py

HashiCorp, CyberArk and mock adapters must pass a common contract suite.

**Acceptance criteria**

- Tests cover capabilities, destination validation, write/create, reference, typed errors and idempotency.

- Provider-specific tests may extend but not weaken common behavior.

### REM-062 - Git origin fixture suite

**Priority: **P0**   |   Design: **DD-07, DD-26**   |   Pytest: **test_rem_062_git_origin_fixture_suite.py

Maintain synthetic local Git histories for origin-analysis tests.

**Acceptance criteria**

- Fixtures cover default inheritance, feature introduction, merge, rebase/cherry-pick, deleted branch and ambiguous history.

- Expected evidence is deterministic.

- No external Git host required.

### REM-063 - Requirement-to-pytest traceability

**Priority: **P0**   |   Design: **DD-26, DD-34**   |   Pytest: **test_rem_063_requirement_to_pytest_traceability.py

Map every P0 requirement to automated pytest coverage and every P1 requirement to automated or explicit acceptance coverage.

**Acceptance criteria**

- CI can produce requirement-to-test matrix.

- CI fails if a P0 ID has no mapped test.

- Tests use requirement markers/naming convention.

## 6. Workflow safety and traceability

### REM-064 - Security-negative test suite

**Priority: **P0**   |   Design: **DD-16, DD-25, DD-26**   |   Pytest: **test_rem_064_security_negative_test_suite.py

Test unsafe behaviors explicitly.

**Acceptance criteria**

- Tests cover plaintext leakage, unsafe .env, unauthorized mutation, invalid fallback, PR leakage, premature rotation and partial failure.

- Sentinel secret is absent from all persisted artifacts.

### REM-065 - Extensible plugin architecture

**Priority: **P1**   |   Design: **DD-02, DD-26**   |   Pytest: **test_rem_065_extensible_plugin_architecture.py

Allow new scanners, providers, playbooks and transformation strategies without changing core orchestration semantics.

**Acceptance criteria**

- Plugin registration is explicit/versioned.

- Plugin passes interface tests.

- Core state/audit/security controls remain mandatory.

### REM-066 - Case/report export

**Priority: **P1**   |   Design: **DD-10, DD-21**   |   Pytest: **test_rem_066_case_report_export.py

Produce a non-secret remediation report suitable for operations and governance.

**Acceptance criteria**

- Includes finding/case IDs, origin evidence, affected branches, stale recommendations, selected remediation, validation, PR and rotation state.

- Supports JSON and human-readable export.

- No plaintext secret.

### REM-067 - Secret location drift handling

**Priority: **P0**   |   Design: **DD-05, DD-15**   |   Pytest: **test_rem_067_secret_location_drift_handling.py

Before extraction/remediation, verify the reported location still corresponds to the intended finding; handle line drift safely.

**Acceptance criteria**

- Stale line/column does not cause unrelated value extraction.

- Rule/context search may relocate candidate with evidence.

- Ambiguous relocation forces review/re-scan.

### REM-068 - Binary/unsupported file handling

**Priority: **P1**   |   Design: **DD-18**   |   Pytest: **test_rem_068_binary_unsupported_file_handling.py

Do not blindly modify binary, generated, vendored or unsupported files.

**Acceptance criteria**

- File classification occurs before transformation.

- Unsupported file becomes manual/review outcome.

- No destructive rewrite occurs.

### REM-069 - History rewriting out of scope by default

**Priority: **P0**   |   Design: **DD-07, DD-16**   |   Pytest: **test_rem_069_history_rewriting_out_of_scope_by_default.py

Do not rewrite Git history to purge prior secret values as part of the default remediation PR workflow.

**Acceptance criteria**

- No filter-repo/BFG/force push is invoked automatically.

- Historical purge may be recommended as separate controlled workflow.

- Rotation remains required for exposed credentials.

### REM-070 - Concurrency control

**Priority: **P0**   |   Design: **DD-15**   |   Pytest: **test_rem_070_concurrency_control.py

Prevent conflicting remediation cases from simultaneously modifying the same repository/origin/finding group.

**Acceptance criteria**

- Repository/origin lock or optimistic concurrency detects conflict.

- Second case waits/merges/supersedes safely.

- No duplicate conflicting PRs.

## 7. APIs, persistence, identity, and UI

### REM-071 - Public API versioning

**Priority: **Must**   |   Design: **DD-02, DD-23**   |   Pytest: **test_rem_071_public_api_versioning.py

All external REST endpoints and events shall use explicit semantic API versions with a published compatibility and deprecation policy.

**Acceptance criteria**

- Contract tests pin request/response/error schemas for each supported version.

- Breaking changes require a new major version and overlapping support window.

### REM-072 - OpenAPI contract

**Priority: **Must**   |   Design: **DD-23, DD-26**   |   Pytest: **test_rem_072_openapi_contract.py

The service shall publish an OpenAPI 3.1 contract covering authentication, pagination, filtering, idempotency, concurrency tokens, errors, and examples.

**Acceptance criteria**

- Generated contract validation runs in CI against handlers.

- Sensitive fields are marked and never returned by list endpoints.

### REM-073 - Asynchronous job contract

**Priority: **Must**   |   Design: **DD-09, DD-23**   |   Pytest: **test_rem_073_asynchronous_job_contract.py

Long-running ingestion, analysis, planning, execution, export, and revalidation operations shall use asynchronous jobs with stable status and cancellation semantics.

**Acceptance criteria**

- Create returns 202 plus job/location; polling and event updates converge.

- Cancellation is best-effort, state-aware, and cannot interrupt an unsafe atomic effect.

### REM-074 - Error model

**Priority: **Must**   |   Design: **DD-23, DD-16**   |   Pytest: **test_rem_074_error_model.py

APIs and adapters shall use a stable typed error model with code, safe message, correlation ID, retryability, field errors, and optional remediation link.

**Acceptance criteria**

- No stack trace, credential, token, code excerpt, or secret value enters the error response.

- Retryable errors include safe backoff hints where applicable.

### REM-075 - Event contract

**Priority: **Must**   |   Design: **DD-24, DD-26**   |   Pytest: **test_rem_075_event_contract.py

Domain and integration events shall be versioned, tenant-scoped, signed/authenticated, idempotent, and contain no secret material.

**Acceptance criteria**

- Schema registry compatibility checks run in CI.

- Duplicate/out-of-order consumer tests preserve state correctness.

### REM-076 - Pagination and bulk limits

**Priority: **Must**   |   Design: **DD-23, DD-24**   |   Pytest: **test_rem_076_pagination_and_bulk_limits.py

List and bulk APIs shall implement bounded cursor pagination, deterministic sorting, and configurable request/result limits.

**Acceptance criteria**

- Cursors cannot be modified to access another tenant.

- Oversized requests fail predictably without partial hidden side effects.

### REM-077 - Core persistence model

**Priority: **Must**   |   Design: **DD-03, DD-22**   |   Pytest: **test_rem_077_core_persistence_model.py

Durable entities shall include tenant, identity bindings, repository, scanner source, finding, fingerprint reference, case, evidence, config snapshot, playbook snapshot, plan/revision/step, approval, execution, external effect, PR, deployment, rotation task, audit event, and outbox event.

**Acceptance criteria**

- The logical and physical models define primary/foreign keys, ownership, retention, and sensitive classifications.

- No column is provided for plaintext secret storage.

### REM-078 - Optimistic concurrency

**Priority: **Must**   |   Design: **DD-15, DD-23**   |   Pytest: **test_rem_078_optimistic_concurrency.py

Mutable API resources shall expose versions/ETags and reject lost updates.

**Acceptance criteria**

- Stale If-Match requests return 412/CONFLICT with current version metadata.

- Plan edits cannot overwrite concurrent approvals or drift updates.

### REM-079 - Database migrations

**Priority: **Must**   |   Design: **DD-22, DD-28**   |   Pytest: **test_rem_079_database_migrations.py

Schema migrations shall be versioned, tested on production-like data, backward compatible during rolling deployment, and reversible or accompanied by a recovery plan.

**Acceptance criteria**

- Expand/migrate/contract checks prevent old binaries from failing during rollout.

- Migration status and checksums are observable and audited.

### REM-080 - Backup and restore

**Priority: **Must**   |   Design: **DD-22, DD-28**   |   Pytest: **test_rem_080_backup_and_restore.py

Persistent state shall be encrypted, backed up, and restorable to defined RPO/RTO with tenant-consistent recovery.

**Acceptance criteria**

- Quarterly restore tests verify integrity and recorded RPO/RTO.

- Restored workflows reconcile external effects before resuming.

### REM-081 - Retention and deletion

**Priority: **Must**   |   Design: **DD-22, DD-16**   |   Pytest: **test_rem_081_retention_and_deletion.py

Retention shall be configurable by data class, legal hold, and tenant policy; deletion shall cover primary, replica, search, cache, export, and backup lifecycles.

**Acceptance criteria**

- Expiry jobs are idempotent and audited.

- Deletion never removes security/audit records earlier than mandatory policy.

### REM-082 - Multi-tenant isolation

**Priority: **Must**   |   Design: **DD-17, DD-25**   |   Pytest: **test_rem_082_multi_tenant_isolation.py

Tenant isolation shall be enforced in identity claims, service authorization, database access, object storage, queues, caches, metrics labels, and cryptographic context.

**Acceptance criteria**

- Automated cross-tenant negative tests cover every repository and API path.

- Tenant IDs are server-derived and cannot be overridden by request payloads.

### REM-083 - Authentication

**Priority: **Must**   |   Design: **DD-17**   |   Pytest: **test_rem_083_authentication.py

Interactive users shall authenticate through OIDC/SAML SSO with MFA policy delegated to the identity provider; workloads shall use short-lived workload identity.

**Acceptance criteria**

- Token validation enforces issuer, audience, signature, expiry, and tenant binding.

- Local passwords and long-lived personal access tokens are disabled in production.

### REM-084 - RBAC and resource authorization

**Priority: **Must**   |   Design: **DD-17, DD-26**   |   Pytest: **test_rem_084_rbac_and_resource_authorization.py

The product shall provide least-privilege roles for viewer, analyst, operator, approver, repository admin, security admin, auditor, and platform admin, plus resource/tenant scopes.

**Acceptance criteria**

- An authorization matrix maps every API/UI action to permissions.

- Deny-by-default tests cover direct object reference and privilege escalation.

### REM-085 - Separation of duties

**Priority: **Must**   |   Design: **DD-13, DD-17**   |   Pytest: **test_rem_085_separation_of_duties.py

Policies shall prevent prohibited combinations such as proposing and solely approving a high-risk plan or administering audit retention while erasing evidence.

**Acceptance criteria**

- Conflicting-role assignments are blocked or require documented exception workflow.

- Approval evaluation uses immutable actor identity and role snapshot.

### REM-086 - Repository authorization

**Priority: **Must**   |   Design: **DD-14, DD-17**   |   Pytest: **test_rem_086_repository_authorization.py

Repository access shall use installed application/workload credentials scoped to approved organizations and repositories, with permission discovery before analysis or mutation.

**Acceptance criteria**

- Insufficient scopes fail before cloning or side effects.

- Credential selection cannot be influenced by untrusted repository content.

### REM-087 - Session and CSRF security

**Priority: **Must**   |   Design: **DD-10, DD-17**   |   Pytest: **test_rem_087_session_and_csrf_security.py

The web UI shall use secure session cookies or standards-based tokens, CSRF defenses, short inactivity timeouts, reauthentication for sensitive actions, and logout revocation.

**Acceptance criteria**

- Security tests cover fixation, CSRF, open redirect, token leakage, and clickjacking.

- Sensitive approvals require recent authentication when policy specifies.

### REM-088 - UI case dashboard

**Priority: **Must**   |   Design: **DD-10**   |   Pytest: **test_rem_088_ui_case_dashboard.py

The UI shall provide searchable/filterable case lists with severity, status, origin, age, provider, repository, assignee, stale indicator, approval state, and SLA.

**Acceptance criteria**

- Server-side filters and pagination match API semantics.

- Saved views do not leak inaccessible tenant/repository identifiers.

### REM-089 - UI case detail

**Priority: **Must**   |   Design: **DD-10**   |   Pytest: **test_rem_089_ui_case_detail.py

Case detail shall display sanitized finding metadata, provenance evidence, branches, effective config, selected playbook, plan revisions, diff, validation, approvals, PR/deployment/rotation timeline, and audit history.

**Acceptance criteria**

- Each displayed artifact shows source/version/freshness.

- Secret values and reversible fingerprints are never rendered or downloadable.

### REM-090 - UI accessible workflow

**Priority: **Must**   |   Design: **DD-10, DD-30**   |   Pytest: **test_rem_090_ui_accessible_workflow.py

Core UI workflows shall meet WCAG 2.2 AA, support keyboard navigation, screen readers, focus management, responsive layouts, and non-color status cues.

**Acceptance criteria**

- Automated accessibility checks and manual keyboard/screen-reader tests pass release gates.

- Destructive or high-risk actions include clear confirmation and consequence text.

### REM-091 - Notifications

**Priority: **Should**   |   Design: **DD-21, DD-24**   |   Pytest: **test_rem_091_notifications.py

The product shall send configurable, deduplicated notifications for assignments, approvals, failures, PR changes, deployment readiness, rotation overdue, and SLA breach through approved channels.

**Acceptance criteria**

- Notifications contain safe deep links and no secret/code payload.

- Delivery failures are retried and observable without blocking workflow state.

## 8. Observability, reliability, scale, and security

### REM-092 - Observability

**Priority: **Must**   |   Design: **DD-21, DD-16**   |   Pytest: **test_rem_092_observability.py

Services shall emit structured redacted logs, metrics, and distributed traces with tenant-safe correlation IDs and documented SLO indicators.

**Acceptance criteria**

- Telemetry leak tests seed canary secrets and assert zero disclosure.

- Golden signals and workflow/provider metrics power dashboards and alerts.

### REM-093 - Service-level objectives

**Priority: **Must**   |   Design: **DD-21, DD-28**   |   Pytest: **test_rem_093_service_level_objectives.py

Production shall define availability, ingestion latency, plan latency, execution success, UI latency, and recovery SLOs with error budgets.

**Acceptance criteria**

- SLO formulas, windows, exclusions, and ownership are documented.

- Release policy reacts to exhausted error budgets.

### REM-094 - Health and readiness

**Priority: **Must**   |   Design: **DD-28**   |   Pytest: **test_rem_094_health_and_readiness.py

Each deployable component shall expose liveness, readiness, dependency, and build/version information without sensitive data.

**Acceptance criteria**

- Readiness removes unhealthy instances before accepting work.

- Dependency degradation is represented separately from process liveness.

### REM-095 - Retries and circuit breakers

**Priority: **Must**   |   Design: **DD-20, DD-24**   |   Pytest: **test_rem_095_retries_and_circuit_breakers.py

All remote calls shall use typed timeouts, bounded exponential backoff with jitter, retry budgets, rate-limit handling, and circuit breakers where appropriate.

**Acceptance criteria**

- Non-idempotent calls are retried only with provider-supported idempotency/reconciliation.

- Chaos tests prove bounded failure amplification.

### REM-096 - Queueing and backpressure

**Priority: **Must**   |   Design: **DD-24**   |   Pytest: **test_rem_096_queueing_and_backpressure.py

Asynchronous workers shall use durable queues, bounded concurrency, visibility/lease renewal, dead-letter handling, priority/fairness, and tenant quotas.

**Acceptance criteria**

- Load tests demonstrate backpressure instead of memory exhaustion.

- Dead-letter replay is authorized, audited, and idempotent.

### REM-097 - Scalability targets

**Priority: **Must**   |   Design: **DD-24, DD-28**   |   Pytest: **test_rem_097_scalability_targets.py

The product shall publish initial capacity assumptions and scale horizontally for tenants, repositories, findings, active workflows, UI users, and provider rate limits.

**Acceptance criteria**

- A repeatable load test meets documented p95/p99 latency and throughput targets at 2x expected peak.

- No unbounded cardinality metric or in-memory tenant state prevents scaling.

### REM-098 - Repository execution sandbox

**Priority: **Must**   |   Design: **DD-25**   |   Pytest: **test_rem_098_repository_execution_sandbox.py

Untrusted repository content and validation commands shall execute in an ephemeral hardened sandbox with no host access, minimal network, resource limits, read-only base images, and controlled credentials.

**Acceptance criteria**

- Escape and exfiltration tests cover filesystem, metadata services, network, process, and artifact channels.

- Sandbox is destroyed after each execution and never reused across tenants.

### REM-099 - Egress control

**Priority: **Must**   |   Design: **DD-25, DD-16**   |   Pytest: **test_rem_099_egress_control.py

Network egress shall be deny-by-default and allow only approved Git, provider, identity, package mirror, and telemetry endpoints per step.

**Acceptance criteria**

- DNS/IP redirects and SSRF attempts cannot reach metadata/internal services.

- Egress policy violations are blocked and audited.

### REM-100 - Artifact and cache safety

**Priority: **Must**   |   Design: **DD-16, DD-25**   |   Pytest: **test_rem_100_artifact_and_cache_safety.py

Generated patches, clones, build caches, exports, and temporary files shall be encrypted where durable, tenant-separated, scanned, access-controlled, and automatically expired.

**Acceptance criteria**

- Secret canary tests cover every artifact store and cache.

- Temporary workspaces are securely deleted or cryptographically expired after use.

### REM-101 - Supply-chain security

**Priority: **Must**   |   Design: **DD-28, DD-31**   |   Pytest: **test_rem_101_supply_chain_security.py

Builds shall pin and verify dependencies, produce SBOMs and provenance attestations, scan code/images/IaC, and sign release artifacts.

**Acceptance criteria**

- Release gates fail on policy-defined critical vulnerabilities or unverifiable provenance.

- Production verifies artifact signatures before deployment.

### REM-102 - Encryption

**Priority: **Must**   |   Design: **DD-16, DD-28**   |   Pytest: **test_rem_102_encryption.py

Data shall use approved TLS in transit and KMS-backed encryption at rest, with tenant/context separation for sensitive evidence and exports.

**Acceptance criteria**

- TLS configuration and certificate rotation are tested.

- Key access and rotation are audited; application logs never contain data keys.

### REM-103 - Security threat model

**Priority: **Must**   |   Design: **DD-16, DD-25**   |   Pytest: **test_rem_103_security_threat_model.py

The product shall maintain a reviewed threat model covering secret exfiltration, malicious repositories, confused deputy, prompt/context injection, provider compromise, SSRF, supply chain, insider misuse, and multi-tenant attacks.

**Acceptance criteria**

- Threats map to controls and verification artifacts.

- Material architecture changes trigger threat-model review.

### REM-104 - AI/agent control boundary

**Priority: **Must**   |   Design: **DD-11, DD-25**   |   Pytest: **test_rem_104_ai_agent_control_boundary.py

Untrusted source, scanner text, repository instructions, and model output shall be treated as data; models may propose but cannot directly invoke privileged effects outside validated plans.

**Acceptance criteria**

- Tool calls require typed schemas, policy authorization, and deterministic parameter validation.

- Prompt-injection fixtures cannot change provider, destination, approval, or egress policy.

### REM-105 - Privacy and data residency

**Priority: **Should**   |   Design: **DD-16, DD-32**   |   Pytest: **test_rem_105_privacy_and_data_residency.py

The product shall document personal data, code/evidence processing, subprocessors, residency options, export, and deletion behavior.

**Acceptance criteria**

- Tenant region controls constrain durable data and processing where offered.

- Privacy inventory and data-flow diagram match deployed components.

### REM-106 - High availability

**Priority: **Must**   |   Design: **DD-20, DD-28**   |   Pytest: **test_rem_106_high_availability.py

Stateless services and durable workflow components shall tolerate instance/zone loss without duplicate effects or data loss beyond RPO.

**Acceptance criteria**

- Failure tests demonstrate leader/lease recovery and continued API availability.

- Single points of failure are documented with mitigation.

### REM-107 - Disaster recovery

**Priority: **Must**   |   Design: **DD-28, DD-29**   |   Pytest: **test_rem_107_disaster_recovery.py

The product shall define regional/major-incident recovery procedures, dependencies, communications, and periodic exercises.

**Acceptance criteria**

- A DR exercise demonstrates the declared RTO/RPO and external-effect reconciliation.

- Runbooks identify authority for pausing automation globally or per tenant.

## 9. Operations, testing, packaging, and readiness

### REM-108 - Operational kill switches

**Priority: **Must**   |   Design: **DD-17, DD-29**   |   Pytest: **test_rem_108_operational_kill_switches.py

Authorized administrators shall be able to disable ingestion, planning, mutations, provider writes, PR creation, or rotation globally and by tenant/provider/repository.

**Acceptance criteria**

- Kill switches are strongly authenticated, audited, and take effect within the documented propagation target.

- Read-only access and evidence preservation remain available unless explicitly disabled.

### REM-109 - Deployment topology

**Priority: **Must**   |   Design: **DD-28**   |   Pytest: **test_rem_109_deployment_topology.py

The product shall define supported Kubernetes/container deployment topology, managed dependencies, network zones, identities, resource sizing, and environment separation.

**Acceptance criteria**

- Reference manifests/Helm chart pass policy and smoke tests.

- Production credentials and configuration are injected at runtime, not baked into images.

### REM-110 - Installation and upgrade

**Priority: **Must**   |   Design: **DD-22, DD-28**   |   Pytest: **test_rem_110_installation_and_upgrade.py

Installation, configuration, upgrade, rollback, and uninstall procedures shall be documented and automated for supported versions.

**Acceptance criteria**

- A clean install and N-1 upgrade test complete in CI/release validation.

- Rollback instructions cover database compatibility and in-flight workflows.

### REM-111 - Configuration and feature rollout

**Priority: **Must**   |   Design: **DD-06, DD-28**   |   Pytest: **test_rem_111_configuration_and_feature_rollout.py

Runtime flags and policy changes shall be versioned, scoped, validated, auditable, and progressively rolled out with safe defaults.

**Acceptance criteria**

- A flag cannot bypass immutable security controls.

- Rollback restores the prior effective configuration deterministically.

### REM-112 - Secrets for the product itself

**Priority: **Must**   |   Design: **DD-16, DD-28**   |   Pytest: **test_rem_112_secrets_for_the_product_itself.py

Platform credentials shall be provisioned through the deployment secret plane, rotated, least-privileged, and excluded from application configuration files.

**Acceptance criteria**

- Startup fails closed when required credentials are unavailable.

- Credential inventory maps owner, scope, rotation, and break-glass procedure.

### REM-113 - Testing pyramid

**Priority: **Must**   |   Design: **DD-26**   |   Pytest: **test_rem_113_testing_pyramid.py

The implementation shall include unit, property, schema, contract, integration, end-to-end, security, chaos, performance, migration, and visual/accessibility tests proportional to risk.

**Acceptance criteria**

- The test plan maps each suite to environments and release gates.

- Nondeterministic external systems use recorded/fake contracts plus scheduled real integration validation.

### REM-114 - Provider certification

**Priority: **Must**   |   Design: **DD-12, DD-26**   |   Pytest: **test_rem_114_provider_certification.py

HashiCorp and CyberArk integrations shall pass repeatable conformance and failure-injection suites against supported product versions.

**Acceptance criteria**

- A compatibility matrix lists tested server/API versions and known limitations.

- Unsupported versions fail with an actionable compatibility error.

### REM-115 - Security testing

**Priority: **Must**   |   Design: **DD-25, DD-26**   |   Pytest: **test_rem_115_security_testing.py

Releases shall include SAST, dependency/container/IaC scanning, secret scanning, API authorization testing, sandbox escape tests, and periodic penetration testing.

**Acceptance criteria**

- Critical findings require remediation or documented time-bound risk acceptance.

- Security regression fixtures include every previously fixed vulnerability class.

### REM-116 - Performance testing

**Priority: **Must**   |   Design: **DD-24, DD-26**   |   Pytest: **test_rem_116_performance_testing.py

Capacity, soak, burst, provider-throttling, large-repository, and high-branch-count tests shall validate published limits.

**Acceptance criteria**

- Results include p50/p95/p99, resource use, queue depth, errors, and recovery.

- Performance regressions beyond agreed thresholds fail the release gate.

### REM-117 - Test data safety

**Priority: **Must**   |   Design: **DD-16, DD-26**   |   Pytest: **test_rem_117_test_data_safety.py

Automated tests shall use synthetic/canary secrets and sanitized repositories; production secrets or customer source shall not enter test fixtures.

**Acceptance criteria**

- Fixture scanning fails CI if non-allowlisted credentials are detected.

- Canary values are unique and traceable for leak detection.

### REM-118 - Packaging and distribution

**Priority: **Must**   |   Design: **DD-28, DD-31**   |   Pytest: **test_rem_118_packaging_and_distribution.py

The product shall ship versioned signed container images, deployment manifests/charts, config schemas, migration tooling, CLI/API clients as applicable, SBOMs, and release notes.

**Acceptance criteria**

- Artifacts are reproducible or provenance-attested and retained per release policy.

- Installation verifies signatures and compatible platform versions.

### REM-119 - Licensing and third-party notices

**Priority: **Must**   |   Design: **DD-31**   |   Pytest: **test_rem_119_licensing_and_third_party_notices.py

Source and binary distributions shall include approved product license, dependency license inventory, attribution/NOTICE, and export/compliance review status.

**Acceptance criteria**

- CI license policy blocks prohibited or unreviewed dependencies.

- Release artifacts contain matching NOTICE and license materials.

### REM-120 - Version support policy

**Priority: **Must**   |   Design: **DD-31, DD-32**   |   Pytest: **test_rem_120_version_support_policy.py

The product shall publish supported platform, Git provider, scanner, HashiCorp, CyberArk, database, browser, and upgrade version matrices with end-of-support rules.

**Acceptance criteria**

- Compatibility is tested for each supported combination or explicitly qualified.

- Deprecated versions generate advance operator warnings.

### REM-121 - Documentation set

**Priority: **Must**   |   Design: **DD-32**   |   Pytest: **test_rem_121_documentation_set.py

Release readiness requires administrator, operator, approver, auditor, developer/API, integration, security, troubleshooting, and disaster-recovery documentation.

**Acceptance criteria**

- Docs are versioned with the product and validated for links/examples.

- A new operator can complete a sandbox remediation using only published docs.

### REM-122 - Product telemetry consent

**Priority: **Should**   |   Design: **DD-16, DD-32**   |   Pytest: **test_rem_122_product_telemetry_consent.py

Usage analytics, if any, shall be opt-in/tenant-controlled, documented, minimized, and separate from security/operational telemetry.

**Acceptance criteria**

- Disabling analytics does not disable required security logs or product functions.

- Analytics schemas exclude repository paths, code, secrets, and user-entered evidence.

### REM-123 - Supportability

**Priority: **Must**   |   Design: **DD-21, DD-29**   |   Pytest: **test_rem_123_supportability.py

The product shall provide safe diagnostics bundles, correlation-based troubleshooting, support access controls, escalation paths, and incident communication procedures.

**Acceptance criteria**

- Diagnostic bundles are redacted, previewable, encrypted, and tenant-authorized.

- Support impersonation is prohibited; elevated access is time-bound and audited.

### REM-124 - Release gates

**Priority: **Must**   |   Design: **DD-31, DD-33, DD-34**   |   Pytest: **test_rem_124_release_gates.py

Production release shall require passed traceability, tests, migrations, threat review, vulnerability/license checks, signed artifacts, rollback validation, docs, SLO dashboards, and operational readiness review.

**Acceptance criteria**

- The release checklist is machine-verifiable where possible and records approvers/evidence.

- A failed mandatory gate cannot be waived without documented authorized exception.

### REM-125 - Pilot and general availability criteria

**Priority: **Must**   |   Design: **DD-33**   |   Pytest: **test_rem_125_pilot_and_general_availability_criteria.py

The product shall define entry/exit criteria for development preview, pilot, and GA, including design partners, feature boundaries, security evidence, reliability, scale, support, and legal readiness.

**Acceptance criteria**

- Pilot explicitly disables or gates unsupported autonomous actions.

- GA requires no open critical risks and documented ownership for residual risks.

## Traceability matrix

This matrix is authoritative for design coverage. Test names are derived from the pattern shown in each requirement block; CI must also validate pytest markers and evidence status.

| **Requirement** | **Priority** | **Design** | **Pytest artifact** |
| --- | --- | --- | --- |
| REM-001 | P0 | DD-04, DD-27 | test_rem_001_supported_scanner_ingestion.py |
| REM-002 | P0 | DD-04, DD-27 | test_rem_002_canonical_finding_schema.py |
| REM-003 | P0 | DD-04, DD-27 | test_rem_003_input_validation_and_quarantine.py |
| REM-004 | P0 | DD-04, DD-27 | test_rem_004_batch_and_multi_branch_input.py |
| REM-005 | P0 | DD-14, DD-17, DD-25 | test_rem_005_least_privilege_repository_acces.py |
| REM-006 | P0 | DD-05, DD-16 | test_rem_006_transient_plaintext_extraction.py |
| REM-007 | P0 | DD-05, DD-16 | test_rem_007_hmac_sha_256_fingerprint.py |
| REM-008 | P0 | DD-05, DD-16 | test_rem_008_cross_branch_deduplication.py |
| REM-009 | P0 | DD-07, DD-08 | test_rem_009_git_history_origin_analysis.py |
| REM-010 | P0 | DD-07, DD-08 | test_rem_010_origin_classification.py |
| REM-011 | P0 | DD-07, DD-08 | test_rem_011_origin_url_and_evidence.py |
| REM-012 | P0 | DD-07, DD-08 | test_rem_012_one_origin_remediation.py |
| REM-013 | P1 | DD-08, DD-10 | test_rem_013_branch_metadata.py |
| REM-014 | P1 | DD-08, DD-10 | test_rem_014_stale_branch_recommendation.py |
| REM-015 | P0 | DD-08, DD-10 | test_rem_015_protected_branch_safety.py |
| REM-016 | P0 | DD-11, DD-18 | test_rem_016_context_analysis.py |
| REM-017 | P1 | DD-11, DD-18 | test_rem_017_secret_taxonomy.py |
| REM-018 | P0 | DD-11, DD-18 | test_rem_018_playbook_repository.py |
| REM-019 | P0 | DD-11, DD-18 | test_rem_019_most_specific_playbook_wins.py |
| REM-020 | P0 | DD-11, DD-18 | test_rem_020_hybrid_deterministic_adaptive_be.py |
| REM-021 | P0 | DD-06 | test_rem_021_organization_yaml_configuration.py |
| REM-022 | P0 | DD-06 | test_rem_022_configuration_precedence.py |
| REM-023 | P0 | DD-12 | test_rem_023_provider_abstraction.py |
| REM-024 | P0 | DD-12 | test_rem_024_hashicorp_template.py |
| REM-025 | P0 | DD-12 | test_rem_025_cyberark_template.py |
| REM-026 | P0 | DD-12 | test_rem_026_provider_routing.py |
| REM-027 | P0 | DD-12 | test_rem_027_managed_vault_preference.py |
| REM-028 | P0 | DD-12 | test_rem_028_unmanaged_vault_fallback.py |
| REM-029 | P1 | DD-12 | test_rem_029_platform_native_fallback.py |
| REM-030 | P1 | DD-12 | test_rem_030_environment_runtime_injection_fa.py |
| REM-031 | P0 | DD-12 | test_rem_031_prohibit_plaintext_env_remediati.py |
| REM-032 | P0 | DD-18 | test_rem_032_centralized_property_config_exte.py |
| REM-033 | P0 | DD-18 | test_rem_033_runtime_vault_bootstrap_injectio.py |
| REM-034 | P1 | DD-18 | test_rem_034_ephemeral_file_exception.py |
| REM-035 | P0 | DD-10, DD-13 | test_rem_035_action_plan_generation.py |
| REM-036 | P0 | DD-10, DD-13 | test_rem_036_review_ui_mode.py |
| REM-037 | P0 | DD-10, DD-13 | test_rem_037_autonomous_bot_mode.py |
| REM-038 | P1 | DD-10, DD-13 | test_rem_038_risk_based_approval_rules.py |
| REM-039 | P1 | DD-10, DD-13 | test_rem_039_dashboard_case_view.py |
| REM-040 | P1 | DD-10, DD-13 | test_rem_040_dashboard_plan_editing.py |
| REM-041 | P0 | DD-14 | test_rem_041_dedicated_remediation_branch.py |
| REM-042 | P0 | DD-14 | test_rem_042_one_pr_per_remediation_unit.py |
| REM-043 | P0 | DD-14 | test_rem_043_pr_safety_and_content.py |
| REM-044 | P0 | DD-14 | test_rem_044_no_default_auto_merge.py |
| REM-045 | P0 | DD-19 | test_rem_045_existing_secret_rotation_sequenc.py |
| REM-046 | P0 | DD-19 | test_rem_046_rotation_handoff.py |
| REM-047 | P0 | DD-18, DD-26 | test_rem_047_pre_pr_validation.py |
| REM-048 | P0 | DD-18, DD-26 | test_rem_048_regression_secret_scan.py |
| REM-049 | P0 | DD-18, DD-26 | test_rem_049_dry_run_mode.py |
| REM-050 | P0 | DD-09 | test_rem_050_explicit_state_machine.py |
| REM-051 | P0 | DD-15, DD-20 | test_rem_051_idempotent_execution.py |
| REM-052 | P0 | DD-09, DD-20 | test_rem_052_partial_failure_recovery.py |
| REM-053 | P1 | DD-19, DD-24 | test_rem_053_deployment_status_integration.py |
| REM-054 | P0 | DD-21 | test_rem_054_audit_trail.py |
| REM-055 | P0 | DD-16, DD-21 | test_rem_055_logging_telemetry_redaction.py |
| REM-056 | P0 | DD-11, DD-16, DD-25 | test_rem_056_llm_context_boundary.py |
| REM-057 | P0 | DD-17 | test_rem_057_least_privilege_and_separation_o.py |
| REM-058 | P0 | DD-25 | test_rem_058_workspace_isolation_and_cleanup.py |
| REM-059 | P1 | DD-25 | test_rem_059_network_egress_policy.py |
| REM-060 | P1 | DD-06, DD-34 | test_rem_060_config_playbook_versioning.py |
| REM-061 | P0 | DD-12, DD-26 | test_rem_061_provider_contract_tests.py |
| REM-062 | P0 | DD-07, DD-26 | test_rem_062_git_origin_fixture_suite.py |
| REM-063 | P0 | DD-26, DD-34 | test_rem_063_requirement_to_pytest_traceabili.py |
| REM-064 | P0 | DD-16, DD-25, DD-26 | test_rem_064_security_negative_test_suite.py |
| REM-065 | P1 | DD-02, DD-26 | test_rem_065_extensible_plugin_architecture.py |
| REM-066 | P1 | DD-10, DD-21 | test_rem_066_case_report_export.py |
| REM-067 | P0 | DD-05, DD-15 | test_rem_067_secret_location_drift_handling.py |
| REM-068 | P1 | DD-18 | test_rem_068_binary_unsupported_file_handling.py |
| REM-069 | P0 | DD-07, DD-16 | test_rem_069_history_rewriting_out_of_scope_b.py |
| REM-070 | P0 | DD-15 | test_rem_070_concurrency_control.py |
| REM-071 | Must | DD-02, DD-23 | test_rem_071_public_api_versioning.py |
| REM-072 | Must | DD-23, DD-26 | test_rem_072_openapi_contract.py |
| REM-073 | Must | DD-09, DD-23 | test_rem_073_asynchronous_job_contract.py |
| REM-074 | Must | DD-23, DD-16 | test_rem_074_error_model.py |
| REM-075 | Must | DD-24, DD-26 | test_rem_075_event_contract.py |
| REM-076 | Must | DD-23, DD-24 | test_rem_076_pagination_and_bulk_limits.py |
| REM-077 | Must | DD-03, DD-22 | test_rem_077_core_persistence_model.py |
| REM-078 | Must | DD-15, DD-23 | test_rem_078_optimistic_concurrency.py |
| REM-079 | Must | DD-22, DD-28 | test_rem_079_database_migrations.py |
| REM-080 | Must | DD-22, DD-28 | test_rem_080_backup_and_restore.py |
| REM-081 | Must | DD-22, DD-16 | test_rem_081_retention_and_deletion.py |
| REM-082 | Must | DD-17, DD-25 | test_rem_082_multi_tenant_isolation.py |
| REM-083 | Must | DD-17 | test_rem_083_authentication.py |
| REM-084 | Must | DD-17, DD-26 | test_rem_084_rbac_and_resource_authorization.py |
| REM-085 | Must | DD-13, DD-17 | test_rem_085_separation_of_duties.py |
| REM-086 | Must | DD-14, DD-17 | test_rem_086_repository_authorization.py |
| REM-087 | Must | DD-10, DD-17 | test_rem_087_session_and_csrf_security.py |
| REM-088 | Must | DD-10 | test_rem_088_ui_case_dashboard.py |
| REM-089 | Must | DD-10 | test_rem_089_ui_case_detail.py |
| REM-090 | Must | DD-10, DD-30 | test_rem_090_ui_accessible_workflow.py |
| REM-091 | Should | DD-21, DD-24 | test_rem_091_notifications.py |
| REM-092 | Must | DD-21, DD-16 | test_rem_092_observability.py |
| REM-093 | Must | DD-21, DD-28 | test_rem_093_service_level_objectives.py |
| REM-094 | Must | DD-28 | test_rem_094_health_and_readiness.py |
| REM-095 | Must | DD-20, DD-24 | test_rem_095_retries_and_circuit_breakers.py |
| REM-096 | Must | DD-24 | test_rem_096_queueing_and_backpressure.py |
| REM-097 | Must | DD-24, DD-28 | test_rem_097_scalability_targets.py |
| REM-098 | Must | DD-25 | test_rem_098_repository_execution_sandbox.py |
| REM-099 | Must | DD-25, DD-16 | test_rem_099_egress_control.py |
| REM-100 | Must | DD-16, DD-25 | test_rem_100_artifact_and_cache_safety.py |
| REM-101 | Must | DD-28, DD-31 | test_rem_101_supply_chain_security.py |
| REM-102 | Must | DD-16, DD-28 | test_rem_102_encryption.py |
| REM-103 | Must | DD-16, DD-25 | test_rem_103_security_threat_model.py |
| REM-104 | Must | DD-11, DD-25 | test_rem_104_ai_agent_control_boundary.py |
| REM-105 | Should | DD-16, DD-32 | test_rem_105_privacy_and_data_residency.py |
| REM-106 | Must | DD-20, DD-28 | test_rem_106_high_availability.py |
| REM-107 | Must | DD-28, DD-29 | test_rem_107_disaster_recovery.py |
| REM-108 | Must | DD-17, DD-29 | test_rem_108_operational_kill_switches.py |
| REM-109 | Must | DD-28 | test_rem_109_deployment_topology.py |
| REM-110 | Must | DD-22, DD-28 | test_rem_110_installation_and_upgrade.py |
| REM-111 | Must | DD-06, DD-28 | test_rem_111_configuration_and_feature_rollou.py |
| REM-112 | Must | DD-16, DD-28 | test_rem_112_secrets_for_the_product_itself.py |
| REM-113 | Must | DD-26 | test_rem_113_testing_pyramid.py |
| REM-114 | Must | DD-12, DD-26 | test_rem_114_provider_certification.py |
| REM-115 | Must | DD-25, DD-26 | test_rem_115_security_testing.py |
| REM-116 | Must | DD-24, DD-26 | test_rem_116_performance_testing.py |
| REM-117 | Must | DD-16, DD-26 | test_rem_117_test_data_safety.py |
| REM-118 | Must | DD-28, DD-31 | test_rem_118_packaging_and_distribution.py |
| REM-119 | Must | DD-31 | test_rem_119_licensing_and_third_party_notice.py |
| REM-120 | Must | DD-31, DD-32 | test_rem_120_version_support_policy.py |
| REM-121 | Must | DD-32 | test_rem_121_documentation_set.py |
| REM-122 | Should | DD-16, DD-32 | test_rem_122_product_telemetry_consent.py |
| REM-123 | Must | DD-21, DD-29 | test_rem_123_supportability.py |
| REM-124 | Must | DD-31, DD-33, DD-34 | test_rem_124_release_gates.py |
| REM-125 | Must | DD-33 | test_rem_125_pilot_and_general_availability_c.py |

## Product readiness assessment

Assessment: implementation-ready for architecture and feature development. It is not a declaration that the product is GA-ready today. Production release requires the explicit gates in REM-124 and stage criteria in REM-125, plus numerical decisions recorded in ADRs. The former v1.0 core was directionally strong but under-specified for API contracts, durable data, identity/RBAC, multi-tenancy, UI security/accessibility, migrations, operations, scale, packaging, licensing, and release governance; REM-071 through REM-125 close those gaps.
