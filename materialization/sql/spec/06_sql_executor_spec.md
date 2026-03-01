# Spec 06: SQL Executor Agent (Execution Runtime)

Status: draft for discussion
Owner: `AiSqlExecutorAgent`

## Goal

Execute approved SQL artifacts safely and deterministically in runtime environments.

Scope:
1. Execute by `artifact_id + version`.
2. Enforce environment/mode policy.
3. Validate parameters and checksum.
4. Provide audit trail and execution metrics.

## Role in Architecture

1. `AiSqlAgent` generates/validates artifacts.
2. `AiSqlExecutorAgent` executes artifacts only.
3. No DSL parsing/generation in executor.

## Modes and Environment Policy

1. `dev` mode:
- allows execution of `tested` and `approved` artifacts in sandbox DSNs.
- supports `dry_run`, `explain`, and test loops.

2. `prod` mode:
- allows execution of `approved` artifacts only.
- rejects direct SQL strings and non-approved artifacts.
- enforces strict policy and full audit.

## Execution Inputs

Required:
1. `environment` (`dev|prod`)
2. `artifact_id`
3. `version`
4. `runtime_params` (map)
5. `request_id` (idempotency key)
6. `caller_identity`

Optional:
1. `execution_mode` (`execute|dry_run|explain`)
2. `timeout_ms`
3. `max_rows`
4. `transaction` (`auto|required|forbidden`)
5. `fallback_policy` (`none|on_error|on_timeout`)

## Execution Pipeline

1. Resolve artifact from registry.
2. Validate `status`, environment eligibility, and deployment role mapping.
3. Verify checksum against stored canonical payload.
4. Validate caller authorization for artifact kind and target environment.
5. Validate `runtime_params` against artifact `params_schema`.
6. Build prepared statement (parameter binding only).
7. Apply policy guards (timeout, row caps, write constraints).
8. Execute and collect result metadata.
9. Persist audit event and metrics.
10. Return structured response.

## Registry Integration

Executor must use these read contracts:
1. `get_artifact(artifact_id, version)`
2. `get_deployment_roles(logical_key)` or equivalent mapping
3. `get_policy(environment, artifact_kind)`

Eligibility rules:
1. In `prod`, only `approved` artifacts are executable.
2. Artifact checksum mismatch blocks execution.
3. Deprecated artifacts require explicit policy override.

## Parameter Validation

1. Reject unknown parameters.
2. Reject missing required parameters.
3. Enforce type checks and nullable rules.
4. Enforce optional business rules attached in `params_schema.rules`.
5. Normalize date/time parameters to canonical format before DB call.

## Security Controls

1. Prepared statements only (no string interpolation execution path).
2. Environment DSN isolation (`dev` and `prod` credentials separated).
3. Policy-based statement class restrictions by environment:
- example: `prod/read-only` blocks write operations.
4. Maximum execution time enforcement.
5. Maximum returned/affected rows enforcement.
6. No secret material in logs or audit payloads.

## Transactions and Retries

1. Transaction policy:
- `auto`: executor decides based on artifact metadata.
- `required`: fail if transaction cannot be opened.
- `forbidden`: fail if artifact requires transaction.

2. Retry policy:
- retry only for transient classes (network hiccup, lock timeout where policy allows).
- no retry for semantic SQL errors or policy denials.
- retries must preserve idempotency constraints.

3. Idempotency:
- same `request_id` + same input returns same terminal result envelope.
- duplicate in-flight requests may be deduplicated by lock.

## Fallback Strategy

When deployment roles are configured (`prod_primary`, optional `prod_fallback`):
1. Execute `prod_primary` first.
2. Switch to `prod_fallback` only when policy permits and trigger matches:
- `on_error` for retryable runtime failures,
- `on_timeout` for timeout class failures.
3. Never fallback on policy/authorization errors.
4. Always record fallback usage in audit.

## Result Contract

Return envelope:
1. `execution_id`
2. `environment`
3. `artifact_id`, `version`, `checksum`
4. `request_id`
5. `mode` (`execute|dry_run|explain`)
6. `result` (`success|error`)
7. `rows_returned`
8. `rows_affected`
9. `duration_ms`
10. `fallback_used` (`true|false`)
11. `error_code` / `error_message` (if failed)
12. `result_preview` (policy-limited, optional)

## Phase Transition Envelope (IDEMO alignment)

Every execution response MUST include transition-level fields:
1. `transition_result` (`true|false`)
2. `transition_code` (`+1|-1`; `0` only for ApplicabilityFailure)
3. `tau_ms` (transition latency; equal to or derived from `duration_ms`)
4. `order_parameter` (domain-specific convergence metric, numeric `0..1`)
5. `in_transition` (`true|false`)
6. `experience_event_ref` (reference to recorded experience/audit event)

Mapping rules:
1. Runtime success -> `transition_result=true`, `transition_code=+1`.
2. Neutral/no-op terminal outcome -> `transition_result=true`, `transition_code=+1`, `fror_zero_class=no_cost_transition`.
3. Failure/absorbing-state terminal outcome -> `transition_result=false`, `transition_code=-1`.
4. `order_parameter` must be deterministic for the same execution class and policy.
5. `experience_event_ref` is mandatory for `prod` executions.

## Audit and Observability

Per execution record:
1. `execution_id`, `request_id`
2. actor/context (`caller_identity`, service, environment)
3. artifact identity and checksum
4. operation class (`read|write|migration`)
5. timing and row counters
6. final status and error code
7. fallback metadata when applied

Metrics (minimum):
1. success/error count
2. p50/p95 duration
3. timeout count
4. fallback rate
5. policy-denied count
6. transition code distribution (`+1|-1`) with analytical no-cost marker
7. order parameter trend

## Error Codes

1. `E_EXEC_INVALID_INPUT`
2. `E_EXEC_ARTIFACT_NOT_FOUND`
3. `E_EXEC_ARTIFACT_STATUS_INVALID`
4. `E_EXEC_CHECKSUM_MISMATCH`
5. `E_EXEC_AUTHZ_DENIED`
6. `E_EXEC_ENVIRONMENT_MISMATCH`
7. `E_EXEC_PARAMS_INVALID`
8. `E_EXEC_POLICY_DENIED`
9. `E_EXEC_TIMEOUT`
10. `E_EXEC_RETRY_EXHAUSTED`
11. `E_EXEC_TXN_REQUIRED`
12. `E_EXEC_TXN_FORBIDDEN`
13. `E_EXEC_IDEMPOTENCY_CONFLICT`
14. `E_EXEC_FALLBACK_NOT_AVAILABLE`
15. `E_EXEC_RUNTIME_FAILURE`

## Acceptance Criteria

1. Executor never executes artifacts that fail status/policy checks.
2. Every production execution is auditable by artifact/version/checksum/request_id.
3. Parameter validation failures happen before DB call.
4. Timeout and row-limit guards are enforceable by policy.
5. Fallback can activate only under allowed triggers and is observable.
6. Repeated identical `request_id` calls are idempotent.
7. Every response contains a valid phase transition envelope.

## TODO (Post-v1)

1. Circuit-breaker policy per artifact family.
2. Adaptive timeout based on historical metrics.
3. Automatic rollback orchestration hooks for migration failures.
4. Multi-region execution routing.
