# Spec 03: Migrations (SQL-first)

Status: draft for discussion
Owner: `AiSqlAgent` (generation), `AiSqlExecutorAgent` (execution)

## Goal

Define deterministic SQL-first migration workflow:
1. Generate reversible migration artifacts (`up`/`down`).
2. Validate migration safety before execution.
3. Execute only approved migration artifacts in `prod`.

## Scope

1. Dialects: `pgsql`, `mysql`, `sqlite`.
2. Migration units:
- schema changes (DDL)
- seed-like structural bootstrap metadata (optional, non-business data)
3. SQL-first baseline; ORM adapter is optional later.

Non-goals:
1. Direct online schema refactoring automation for zero-downtime edge cases.
2. Auto-repair of failed production migrations without explicit rollback/forward plan.

## Input Contracts

### A) Author-facing migration DSL (YAML)

```yaml
dsl_version: v1
kind: migration_plan
spec:
  dialect: pgsql
  name: add_users_email_unique
  strategy: safe
  operations:
    - op: add_column
      table: users
      column: email
      type: varchar(255)
      nullable: false
      default: ''
    - op: create_index
      table: users
      index: users_email_unique
      columns: [email]
      unique: true
    - op: add_foreign_key
      table: orders
      columns: [user_id]
      ref_table: users
      ref_columns: [id]
      on_delete: cascade
      on_update: restrict
      constraint: orders_user_id_fk
    - op: create_pivot_table
      table: role_user
      left:
        table: users
        column: id
        fk_column: user_id
      right:
        table: roles
        column: id
        fk_column: role_id
      primary_key: [user_id, role_id]
      timestamps: false
      on_delete_left: cascade
      on_delete_right: cascade
```

### B) Schema diff mode (generated)

`AiSqlAgent` may generate migrations from:
1. `current_schema` (introspected snapshot)
2. `target_schema` (desired schema spec)

Result must still produce the same migration artifact contract below.

## Artifact Contract (`migration_sql`)

Each migration artifact must contain:
1. `artifact_id`
2. `version`
3. `dialect`
4. `name`
5. `up_sql` (ordered statements)
6. `down_sql` (ordered statements; may be policy-blocked in prod)
7. `checksum`
8. `risk_level` (`low|medium|high`)
9. `destructive_ops` (list)
10. `requires_manual_approval` (`true|false`)
11. `prechecks` (list)
12. `postchecks` (list)
13. `estimated_locking` (`none|short|long|unknown`)

Primary key metadata:
1. `primary_key_changes` (list of add/drop/alter pk operations)

## Deterministic Generation Rules

1. Operation ordering is stable and explicit.
2. Identifier quoting follows dialect rules from SQL prompt spec.
3. `up_sql` and `down_sql` are generated together in one pass.
4. No auto-generated timestamps in SQL payload.
5. Re-running generator with same input must produce byte-identical SQL.

Foreign key deterministic rules:
1. Constraint name is required or deterministically generated.
2. Composite FK column order is preserved exactly as provided.
3. `on_delete` and `on_update` are always rendered explicitly.

## Deterministic Naming Policy

If explicit names are omitted, generator must derive names from canonical pattern rules.

Naming patterns:
1. Primary key:
- `<table>_pkey`
2. Foreign key:
- `<table>_<col1>_<colN>_fk`
3. Unique index/constraint:
- `<table>_<col1>_<colN>_uniq`
4. Non-unique index:
- `<table>_<col1>_<colN>_idx`
5. Check constraint:
- `<table>_<rule_slug>_chk`

Normalization rules:
1. Lowercase only.
2. Non `[a-z0-9_]` chars replaced with `_`.
3. Multiple `_` collapsed into one.
4. Column segment order preserved as declared.
5. Suffix is mandatory (`_fk`, `_uniq`, `_idx`, `_chk`, `_pkey`).

Length and collision rules:
1. If generated name exceeds dialect limit, truncate and append `_` + 8-char deterministic hash.
2. If collision remains in same schema scope, append `_` + incremental numeric suffix (`_2`, `_3`, ...).
3. Same input must always resolve to the same final name set.

Rename safety:
1. Renaming generated constraint/index names across versions is forbidden unless operation intent is explicit rename.
2. Diff engine must treat deterministic regenerated same-name objects as unchanged.

## Safety Policies

### Risk classification

1. `low`:
- add nullable column
- create non-blocking index (where supported)
2. `medium`:
- add non-null column with default
- alter column type with explicit conversion
- change primary key shape on non-empty table
3. `high`:
- drop column/table
- destructive type narrowing
- rename without compatibility bridge
- adding FK with `on_delete=cascade` on high-cardinality parent relation
- dropping primary key from active table

### Strategy modes

1. `safe` (default):
- blocks `high` risk unless explicit override.
- requires manual approval for any destructive op.
2. `force`:
- allows high risk with explicit flags and approval evidence.

### Required safeguards

1. Any destructive operation sets `requires_manual_approval=true`.
2. Missing `down_sql` for destructive changes is rejected.
3. Data-loss risk must be annotated in metadata.
4. Migration touching large tables must include precheck constraints.
5. For FK creation:
- referenced table/columns must exist,
- referencing and referenced types must be compatible,
- index presence policy must be satisfied on FK columns.
6. For `on_delete=cascade`:
- policy must explicitly allow cascade for that table pair,
- migration metadata must include impact note.

## Prechecks / Postchecks

Prechecks examples:
1. table exists / column exists expectations.
2. row-count or nullability gate before constraint hardening.
3. index existence guard.
4. FK backfill integrity gate (`orphan rows = 0`) before adding FK.

Postchecks examples:
1. expected schema version applied.
2. expected indexes/constraints present.
3. optional smoke query returns expected shape.
4. FK constraint exists with expected delete/update rules.

## Foreign Key and Cascade Rules

Supported FK operations:
1. `add_foreign_key`
2. `drop_foreign_key`

`add_foreign_key` required fields:
1. `table`
2. `columns` (ordered list)
3. `ref_table`
4. `ref_columns` (ordered list; same length as `columns`)
5. `on_delete` (`restrict|cascade|set_null|no_action`)
6. `on_update` (`restrict|cascade|set_null|no_action`)
7. `constraint` (optional if deterministic naming enabled)

Validation rules:
1. `set_null` requires nullable FK columns.
2. `cascade` in `safe` strategy requires explicit allow policy entry.
3. `drop_foreign_key` in `safe` strategy is at least `medium` risk.
4. FK creation should fail if existing data violates referential integrity.

## Pivot Table Rules (Many-to-Many)

Supported pivot operations:
1. `create_pivot_table`
2. `drop_pivot_table`

`create_pivot_table` required fields:
1. `table` (pivot table name)
2. `left.table`
3. `left.column` (referenced column, usually PK)
4. `left.fk_column` (pivot column referencing left table)
5. `right.table`
6. `right.column` (referenced column, usually PK)
7. `right.fk_column` (pivot column referencing right table)

Optional fields:
1. `primary_key` (default `[left.fk_column, right.fk_column]`)
2. `timestamps` (`true|false`, default `false`)
3. `on_delete_left` (`restrict|cascade|set_null|no_action`, default `cascade`)
4. `on_delete_right` (`restrict|cascade|set_null|no_action`, default `cascade`)
5. `on_update_left` (`restrict|cascade|set_null|no_action`, default `restrict`)
6. `on_update_right` (`restrict|cascade|set_null|no_action`, default `restrict`)

Generation requirements:
1. Create both FK constraints from pivot to left/right tables.
2. Create composite PK on pivot (`primary_key`).
3. Create supporting indexes when required by dialect/perf policy.
4. Apply deterministic naming policy to PK/FK/indexes.

Validation rules:
1. Left/right referenced tables and columns must exist.
2. Left and right FK column types must be compatible with referenced columns.
3. `primary_key` must include both relation columns.
4. If `timestamps=true`, add `created_at` and `updated_at` columns by dialect convention.
5. In `safe` strategy, both-side `cascade` requires explicit allow policy for that relation.

## Primary Key Rules

Supported PK operations:
1. `add_primary_key`
2. `drop_primary_key`
3. `change_primary_key`

`add_primary_key` required fields:
1. `table`
2. `columns` (ordered list, one or more)
3. `constraint` (optional; defaults to deterministic naming policy)

`drop_primary_key` required fields:
1. `table`
2. `constraint` (optional if deterministic name can be resolved)

`change_primary_key` required fields:
1. `table`
2. `from_columns`
3. `to_columns`
4. `constraint` (optional)

Validation rules:
1. PK columns must be `NOT NULL` before PK creation.
2. PK candidate values must be unique before PK creation.
3. Composite PK column order is significant and must be preserved.
4. `change_primary_key` on non-empty table is at least `medium` risk.
5. `drop_primary_key` in `safe` strategy requires explicit override.

## Dev/Prod Execution Alignment

1. In `dev`:
- generate migration artifacts,
- run sandbox apply + optional rollback test,
- mark as `tested` with evidence.

2. In `prod`:
- execute only `approved` migration artifacts,
- enforce maintenance window policy,
- enforce migration lock,
- persist schema version markers before/after execution.

3. `down_sql` in `prod`:
- disabled by default,
- requires explicit override policy and approval mode constraints.

## Failure and Rollback Model

1. If `up_sql` fails mid-flight:
- executor records failed statement index,
- transaction rollback when dialect/operation set allows,
- otherwise stop and require operator decision.

2. Roll-forward preferred:
- generate follow-up corrective migration when possible.

3. Rollback execution:
- allowed automatically in `dev` test loop,
- restricted in `prod` by policy.

## Registry Metadata Additions (for migrations)

1. `schema_version_from`
2. `schema_version_to`
3. `test_report_ref`
4. `approval_mode` (`auto|hybrid|manual`)
5. `deployment_role` (`prod_primary|prod_fallback|none`)

## Error Codes

1. `E_MIGRATION_PARSE`
2. `E_MIGRATION_UNSUPPORTED_OP`
3. `E_MIGRATION_INVALID_ORDER`
4. `E_MIGRATION_DOWN_MISSING`
5. `E_MIGRATION_RISK_BLOCKED`
6. `E_MIGRATION_PRECHECK_FAILED`
7. `E_MIGRATION_POSTCHECK_FAILED`
8. `E_MIGRATION_SCHEMA_VERSION_CONFLICT`
9. `E_MIGRATION_TXN_UNSAFE`
10. `E_MIGRATION_ROLLBACK_FORBIDDEN`
11. `E_MIGRATION_FK_INVALID`
12. `E_MIGRATION_CASCADE_POLICY_BLOCKED`
13. `E_MIGRATION_REFERENTIAL_INTEGRITY_FAILED`
14. `E_MIGRATION_NAME_TOO_LONG`
15. `E_MIGRATION_NAME_COLLISION`
16. `E_MIGRATION_PK_INVALID`
17. `E_MIGRATION_PK_NOT_NULL_VIOLATION`
18. `E_MIGRATION_PK_DUPLICATE_VALUES`
19. `E_MIGRATION_PIVOT_INVALID`
20. `E_MIGRATION_PIVOT_RELATION_MISSING`

## Acceptance Criteria

1. Same DSL input produces identical migration artifacts on repeated runs.
2. Destructive changes are blocked in `safe` mode without explicit override.
3. Every approved migration has both `up_sql` and `down_sql` metadata (unless policy-exempt and explicitly documented).
4. `prod` execution rejects non-approved migration artifacts.
5. `dev` pipeline can run apply-check-rollback-check for candidate migrations.
6. Migration execution emits auditable pre/post schema version markers.

## End-to-End DSL Example (`users` + `roles` + pivot)

```yaml
dsl_version: v1
kind: migration_plan
spec:
  dialect: pgsql
  name: create_users_roles_and_pivot
  strategy: safe
  operations:
    - op: create_table
      table: users
      columns:
        - { name: id, type: bigint, nullable: false, auto_increment: true }
        - { name: email, type: varchar(255), nullable: false }
        - { name: name, type: varchar(255), nullable: false }
        - { name: created_at, type: timestamp, nullable: false }
        - { name: updated_at, type: timestamp, nullable: false }
    - op: add_primary_key
      table: users
      columns: [id]

    - op: create_table
      table: roles
      columns:
        - { name: id, type: bigint, nullable: false, auto_increment: true }
        - { name: code, type: varchar(64), nullable: false }
        - { name: title, type: varchar(255), nullable: false }
        - { name: created_at, type: timestamp, nullable: false }
        - { name: updated_at, type: timestamp, nullable: false }
    - op: add_primary_key
      table: roles
      columns: [id]
    - op: create_index
      table: roles
      index: roles_code_uniq
      columns: [code]
      unique: true

    - op: create_pivot_table
      table: role_user
      left:
        table: users
        column: id
        fk_column: user_id
      right:
        table: roles
        column: id
        fk_column: role_id
      primary_key: [user_id, role_id]
      timestamps: false
      on_delete_left: cascade
      on_delete_right: cascade
      on_update_left: restrict
      on_update_right: restrict
```
