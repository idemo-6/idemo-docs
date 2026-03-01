# Spec 02: Execution Modes (`dev` and `prod`)

Status: draft for discussion
Owner: `AiSqlAgent`, `AiSqlExecutorAgent`

## Goal

Define a strict two-mode workflow for SQL agents:
1. `dev` for generation and sandbox verification.
2. `prod` for execution of pre-approved artifacts only.

## Core Principle

1. `dev` may create and test artifacts.
2. `prod` may execute only immutable approved artifacts.
3. No on-the-fly generation in `prod`.

Transition contract:
1. Any terminal execution in `dev` or `prod` must emit transition envelope fields:
- `transition_result` (`true|false`)
- `transition_code` (`+1|-1`; `0` only for `ApplicabilityFailure`)
- `tau_ms`
- `order_parameter`
- `experience_event_ref`

## Artifact Types

1. `query_sql` (read/write SQL templates with params)
2. `migration_sql` (`up/down`)
3. `infra_artifact` (compose/env/docker templates)
4. `validation_rules` (normalized rule sets)

## Artifact Identity

Each artifact must have:
1. `artifact_id` (stable logical ID)
2. `version` (monotonic, immutable after publish)
3. `checksum` (sha256 of canonical payload)
4. `kind`
5. `dialect`
6. `created_by`
7. `created_at`
8. `status`

## Artifact Status Lifecycle

1. `draft`
2. `tested`
3. `approved`
4. `deprecated`

Transitions:
1. `draft -> tested` (successful sandbox checks)
2. `tested -> approved` (human/system approval policy)
3. `approved -> deprecated` (superseded/retired)

Forbidden transitions:
1. direct `draft -> approved`
2. `deprecated -> approved`

## Mode: `dev` (Sandbox)

Allowed:
1. Generate new artifacts.
2. Execute sandbox tests.
3. Validate SQL/rules/migrations.
4. Persist successful outputs to registry/store.

Required checks before marking `tested`:
1. Parsing and semantic validation passed.
2. Parameter contract validation passed.
3. Execution test passed on sandbox DB (or approved dry-run strategy).
4. Output checksum and metadata recorded.

Not allowed:
1. Running against production DSNs.
2. Writing directly to prod registry namespace without workflow state.

## Mode: `prod`

Allowed:
1. Execute artifact by `artifact_id + version`.
2. Execute only if `status=approved`.
3. Record full audit trail.

Required preconditions:
1. Artifact exists in registry.
2. Checksum matches stored canonical payload.
3. Caller is authorized for artifact kind and environment.
4. Runtime parameters satisfy artifact parameter schema.

Not allowed:
1. New artifact generation.
2. DSL parsing/generation-to-execute shortcut.
3. Executing `draft` or `tested` artifacts.

## Registry / Store Contract (Minimal)

1. `put_artifact(payload, metadata) -> artifact_id, version`
2. `mark_tested(artifact_id, version, test_report_ref)`
3. `approve(artifact_id, version, approver)`
4. `deprecate(artifact_id, version, reason)`
5. `get_artifact(artifact_id, version)`
6. `list_artifacts(filters)`
7. `set_deployment_roles(primary_artifact_id, primary_version, fallback_artifact_id?, fallback_version?)`

Namespace rule:
1. Registry keeps logical separation for `dev` and `prod` environments.
2. Promotion to `prod` namespace is allowed only from `tested` to `approved`.

## Approval Policy

1. `approved` status must include:
- `approved_by`
- `approved_at`
- `approval_note` (optional)
2. The approver cannot be equal to `created_by` when strict approval mode is enabled.
3. Any payload change requires new version; re-approval of mutated version is forbidden.

Approval modes:
1. `auto`:
- Promotion to `approved` is allowed when sandbox thresholds are met.
- Example threshold: `min_success_runs=10`.
- Candidate ranking must be deterministic by configured metrics (for example: lower execution time, lower resource usage).
- Top 2 candidates become deployment targets:
  - rank 1 -> `prod_primary`
  - rank 2 -> `prod_fallback`
2. `hybrid`:
- Candidate generation and ranking follows `auto`.
- Final promotion requires explicit `manual_confirmation=true`.
3. `manual`:
- All successful sandbox candidates are saved as `tested`.
- Human approver selects `prod_primary` and optional `prod_fallback`.

Primary/fallback rules:
1. Exactly one active `prod_primary` per artifact family.
2. Zero or one active `prod_fallback`.
3. Fallback must match the same parameter schema and environment constraints.
4. Switching primary/fallback updates deployment mapping metadata only (artifact payload stays immutable).

## Runtime Parameter Contract

1. Each executable artifact stores parameter schema and defaults.
2. In `prod`, unknown runtime parameters are rejected.
3. In `prod`, missing required parameters fail fast before DB connection.
4. Parameter schema changes require a new artifact version.

## Concurrency and Idempotency

1. `prod` execution must support per-artifact execution lock for migration artifacts.
2. Same `execution_id` replay must be idempotent at orchestration layer.
3. Executor must record final state atomically (`success|error`) once.

## Migration-Specific Safety

1. `migration_sql` artifacts in `prod` execute only in explicit maintenance window policy.
2. `down` migration execution in `prod` is disabled by default and requires explicit override policy.
3. Each migration execution must persist pre/post schema version markers.

## Audit Requirements

Every `prod` execution record must include:
1. `execution_id`
2. `artifact_id`
3. `version`
4. `checksum`
5. `environment=prod`
6. `executed_by`
7. `started_at`, `finished_at`
8. `result` (`success|error`)
9. `rows_affected/returned`
10. `error_code` (if failed)

## Security Rules

1. Strict parameter binding.
2. Environment isolation (`dev` and `prod` credentials separated).
3. No secret values persisted in artifacts.
4. `prod` allowlist for executable artifact kinds.

## Error Codes

1. `E_MODE_INVALID`
2. `E_MODE_FORBIDDEN_OPERATION`
3. `E_ARTIFACT_NOT_FOUND`
4. `E_ARTIFACT_STATUS_INVALID`
5. `E_ARTIFACT_CHECKSUM_MISMATCH`
6. `E_ARTIFACT_VERSION_IMMUTABLE`
7. `E_AUTHZ_DENIED`
8. `E_ENVIRONMENT_MISMATCH`
9. `E_APPROVAL_POLICY_VIOLATION`
10. `E_RUNTIME_PARAMS_INVALID`
11. `E_EXECUTION_LOCKED`
12. `E_MIGRATION_POLICY_VIOLATION`
13. `E_APPROVAL_THRESHOLD_NOT_MET`
14. `E_DEPLOYMENT_ROLE_CONFLICT`
15. `E_MANUAL_CONFIRMATION_REQUIRED`

## Acceptance Criteria

1. Attempt to execute non-approved artifact in `prod` fails with coded error.
2. Attempt to generate in `prod` fails with coded error.
3. Successful `dev` test can be promoted to `tested` only with evidence.
4. Successful `prod` execution is fully auditable by artifact version and checksum.
5. Any payload mutation after approval is rejected and requires a new version.
6. `prod` migration execution respects lock and policy constraints.
7. In `auto` mode, approval fails if success threshold is not met.
8. In `hybrid` mode, approval fails until `manual_confirmation=true`.
9. Deployment mapping always resolves to one primary and optional fallback.
10. Terminal executions in both modes include transition envelope fields.
