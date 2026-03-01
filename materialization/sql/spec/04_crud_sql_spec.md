# Spec 04: CRUD SQL (SQL-first)

Status: draft for discussion
Owner: `AiSqlAgent` (generation/validation), `AiSqlExecutorAgent` (execution)

## Goal

Define deterministic CRUD generation and validation:
1. Generate parameterized SQL templates for CRUD operations.
2. Validate schema and rules before artifact creation.
3. Align execution with `dev/prod` artifact workflow.

## Scope

CRUD operations v1:
1. `select_one`
2. `select_many`
3. `insert_one`
4. `insert_many`
5. `update_where`
6. `delete_where`
7. `upsert`

Dialects:
1. `pgsql`
2. `mysql`
3. `sqlite`

## Input Contract (YAML DSL)

```yaml
dsl_version: v1
kind: crud_query
spec:
  dialect: pgsql
  operation: update_where
  table: users
  set:
    name: :name
    updated_at: :updated_at
  where:
    - field: id
      op: '='
      value: :id
  options:
    safe_mode: true
    limit: 1
```

Common required fields:
1. `dialect`
2. `operation`
3. `table`

Operation-specific fields:
1. `select_*`: `select`, optional `where`, `order`, `limit`, `offset`
2. `insert_*`: `values` (map or list of maps)
3. `update_where`: `set`, `where`
4. `delete_where`: `where`
5. `upsert`: `values`, `conflict_target`, `update_set` (dialect-dependent)

## Validation Rules

1. Table and columns must exist in schema context.
2. Parameter placeholders must be declared in `params_schema`.
3. Unsafe write guards:
- `update_where` without `where` is blocked in `safe_mode`.
- `delete_where` without `where` is blocked in `safe_mode`.
4. `limit` and `offset` must be non-negative integers.
5. `insert_many` rows must share the same column set.
6. `upsert` requires dialect-compatible conflict syntax.
7. FK and unique constraints can be prechecked in `dev` sandbox.

## Parameter Contract

Each artifact must define `params_schema`:
1. `name`
2. `type` (`string|int|numeric|bool|date|datetime|json|null`)
3. `required`
4. `nullable`
5. optional `rules` (aligned with rules DSL)

Runtime rules:
1. Unknown runtime params are rejected.
2. Missing required params fail fast before execution.
3. Nullability and basic type checks run before DB call.

## Deterministic SQL Rendering

1. SQL templates must be parameterized (no literal interpolation).
2. Stable column ordering:
- `insert`: schema order intersected with provided fields.
- `update`: lexicographic column order unless explicit deterministic order is provided.
3. Stable placeholder ordering by first appearance.
4. Identifier quoting by dialect:
- `pgsql/sqlite`: `"identifier"`
- `mysql`: `` `identifier` ``
5. No trailing semicolon.

## Operation Semantics

### `select_one`

1. Behaves like `select_many` plus enforced row cap (`LIMIT 1`).
2. Returns shape metadata for one row or null.

### `select_many`

1. Supports filtering, ordering, pagination.
2. Default limit policy may be applied by environment.

### `insert_one` / `insert_many`

1. Require explicit writable column set.
2. Optional returning policy by dialect:
- `pgsql`: `RETURNING`
- `mysql/sqlite`: follow executor post-insert fetch policy.

### `update_where`

1. Requires non-empty `set`.
2. `where` required in `safe_mode`.
3. Optional write limit where dialect supports it (or executor-enforced row cap).

### `delete_where`

1. `where` required in `safe_mode`.
2. Hard delete only in v1.
3. Soft delete is modeled via `update_where` (for example `deleted_at` set).

### `upsert`

1. `pgsql`: `INSERT ... ON CONFLICT (...) DO UPDATE`.
2. `mysql`: `INSERT ... ON DUPLICATE KEY UPDATE`.
3. `sqlite`: `INSERT ... ON CONFLICT (...) DO UPDATE`.
4. Conflict target validation required for engines that need explicit target.

## Safety Policies

1. Mode alignment:
- `dev`: generate + test + persist tested artifacts.
- `prod`: execute approved artifacts only.
2. Write operation policy flags:
- `safe_mode=true` default.
- `allow_mass_update=false` default.
- `allow_mass_delete=false` default.
3. For mass operations, explicit override and approval are required.

## Artifact Contract (`crud_sql`)

Required metadata:
1. `artifact_id`
2. `version`
3. `kind=crud_sql`
4. `dialect`
5. `operation`
6. `table`
7. `sql_template`
8. `params_schema`
9. `checksum`
10. `read_write` (`read|write`)
11. `safe_flags`

Optional metadata:
1. `expected_result_shape`
2. `row_limit_policy`
3. `requires_manual_approval`

## Audit Requirements

For execution records:
1. `execution_id`
2. `artifact_id`, `version`, `checksum`
3. `operation`, `table`
4. `rows_returned` / `rows_affected`
5. `duration_ms`
6. `result`
7. `error_code` (if failed)

## Error Codes

1. `E_CRUD_PARSE`
2. `E_CRUD_UNSUPPORTED_OPERATION`
3. `E_CRUD_SCHEMA_INVALID`
4. `E_CRUD_PARAM_INVALID`
5. `E_CRUD_UNSAFE_WRITE`
6. `E_CRUD_WHERE_REQUIRED`
7. `E_CRUD_EMPTY_SET`
8. `E_CRUD_UPSERT_CONFLICT_INVALID`
9. `E_CRUD_DIALECT_UNSUPPORTED_FEATURE`
10. `E_CRUD_RENDER_FAILED`

## Acceptance Criteria

1. Same input DSL produces byte-identical SQL template and metadata.
2. Unsafe update/delete without `where` are blocked in `safe_mode`.
3. All generated write templates are parameterized.
4. `upsert` generation is valid for each supported dialect.
5. Runtime param validation fails before DB call on schema mismatch.
6. `prod` execution rejects non-approved CRUD artifacts.

## TODO (Post-v1)

1. Non-PK relationship support (explicit reference override) for edge legacy schemas.
2. Advanced subquery patterns:
- derived tables as first-class user DSL constructs,
- top-N per group helpers,
- reusable named subqueries.
3. Window-function helpers (`row_number`, `rank`, partition analytics).
4. CTE authoring primitives in user-facing DSL.
5. Correlated scalar subqueries in `SELECT` as first-class shortcut.
6. Automatic plan-based query rewrites (optimizer hints/policy).

## End-to-End DSL Examples

### `insert_one` (`users`)

```yaml
dsl_version: v1
kind: crud_query
spec:
  dialect: pgsql
  operation: insert_one
  table: users
  values:
    email: :email
    name: :name
    created_at: :created_at
    updated_at: :updated_at
```

### `select_many` with pagination (`users`)

```yaml
dsl_version: v1
kind: crud_query
spec:
  dialect: pgsql
  operation: select_many
  table: users
  select: [id, email, name, created_at]
  where:
    - field: email
      op: like
      value: :email_like
  order:
    - field: id
      dir: desc
  limit: 50
  offset: 0
```

### `upsert` (`roles`)

```yaml
dsl_version: v1
kind: crud_query
spec:
  dialect: pgsql
  operation: upsert
  table: roles
  values:
    code: :code
    title: :title
  conflict_target: [code]
  update_set:
    title: :title
```
