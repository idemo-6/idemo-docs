# Spec 07: SQL Agents Context Contract

Status: draft for discussion
Owner: `AiSqlAgent`, `AiSqlExecutorAgent`

## Goal

Define the context envelope contract required by SQL agents only.

Out of scope:
1. Orchestrator internals.
2. Pre-SQL routing logic.
3. Multi-agent planning before SQL operator invocation.

## Context Envelope (Input)

Every SQL-agent call must receive `context_envelope`:

```yaml
context_envelope:
  context_version: v1
  request_context:
    request_id: req_...
    trace_id: trace_...
    timestamp: "2026-02-25T20:00:00Z"
    caller:
      id: user_or_service
      type: user|service
  environment_context:
    mode: dev|prod
    dialect: pgsql|mysql|sqlite
    target: sandbox|production
  schema_context:
    tables: {}
  policy_context:
    safe_mode: true
    allow_mass_update: false
    allow_mass_delete: false
    max_rows: 1000
    timeout_ms: 10000
  connection_context:
    dsn_ref: secret://...
    host: db.internal
    port: 5432
    database: app
    user_ref: secret://...
    password_ref: secret://...
  system_state:
    maintenance_window: false
    db_health: healthy|degraded|down
    load_level: low|medium|high
  artifact_context:
    artifact_id: art_...
    version: 3
    checksum: sha256:...
```

## Required Blocks by Agent

### `AiSqlAgent`

Required:
1. `request_context`
2. `environment_context.mode`
3. `environment_context.dialect`
4. `schema_context`
5. `policy_context`

Optional:
1. `system_state`
2. `artifact_context` (when resolving existing artifact revisions)
3. `connection_context` (not required for pure generation)

### `AiSqlExecutorAgent`

Required:
1. `request_context`
2. `environment_context`
3. `policy_context`
4. `connection_context`
5. `artifact_context`
6. `schema_context` (required when runtime validation depends on schema)

Optional:
1. `system_state` (recommended for runtime gating)

## Invariants

1. Context is immutable during a single agent execution.
2. `request_id` and `trace_id` must be propagated to all logs/events.
3. `mode=prod` requires strict policy enforcement and approved artifact usage.
4. `dialect` in context must match artifact dialect when artifact execution is requested.
5. Missing required context block fails fast before parsing/generation/execution.
6. Transition envelope fields must be propagated to logging/audit context for terminal SQL operations.

## Schema Context Rules

1. Schema context must contain all tables/columns referenced by requested operation.
2. FK metadata is required for simplified path DSL resolution.
3. v1 FK contract: references are resolved to target `primaryKey`.
4. Stale or partial schema context must be rejected for operations that depend on missing entities.

## Policy Context Rules

Minimum policy keys:
1. `safe_mode`
2. `allow_mass_update`
3. `allow_mass_delete`
4. `max_rows`
5. `timeout_ms`

Extended keys (optional):
1. `approval_mode` (`auto|hybrid|manual`)
2. `fallback_policy` (`none|on_error|on_timeout`)
3. `allow_down_migration_in_prod`

## Connection Context Rules (Executor)

1. Use secret references, not raw secrets in payload where possible.
2. DSN/host/port/database must be consistent with `environment_context.target`.
3. Connection context must never be persisted in clear text in audit logs.

## System State Usage

`system_state` may gate execution:
1. If `db_health=down` -> reject runtime execution.
2. If `maintenance_window=false` and migration artifact -> reject in `prod`.
3. If `load_level=high`, optional policy may force `read-only` or tighter limits.

## Context Normalization

Before use, SQL agents normalize context:
1. Apply defaults for optional policy keys.
2. Normalize enum casing to lowercase.
3. Validate required keys and primitive types.
4. Produce deterministic internal `normalized_context` object.

## Security and Redaction

1. Never log raw secrets or resolved secret values.
2. Redact secret refs in user-facing errors where needed.
3. Include only minimal connection metadata in audit (`host hash`, `db name`, `port`).

## Error Codes

1. `E_CONTEXT_MISSING`
2. `E_CONTEXT_INVALID_TYPE`
3. `E_CONTEXT_VERSION_UNSUPPORTED`
4. `E_CONTEXT_DIALECT_MISMATCH`
5. `E_CONTEXT_SCHEMA_INCOMPLETE`
6. `E_CONTEXT_POLICY_INVALID`
7. `E_CONTEXT_CONNECTION_INVALID`
8. `E_CONTEXT_SYSTEM_STATE_BLOCKED`
9. `E_CONTEXT_IMMUTABILITY_VIOLATION`

## Acceptance Criteria

1. SQL agents fail fast with coded error when required context blocks are absent.
2. Same `context_envelope` yields same `normalized_context` deterministically.
3. Executor blocks runtime when context policy/state forbids execution.
4. No sensitive connection data appears in logs or returned errors.
5. Traceability is preserved via `request_id` and `trace_id` in all execution records.
6. Transition envelope linkage (`transition_code`, `fror_zero_class`, `experience_event_ref`) is present in execution records.
