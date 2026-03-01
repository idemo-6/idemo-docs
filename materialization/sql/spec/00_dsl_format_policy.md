# Spec 00: DSL Format Policy

Status: accepted baseline
Owner: `AiSqlAgent`

## Decision

1. External DSL format: `YAML`.
2. Canonical internal format: normalized `JSON IR`.
3. `TOML` is allowed only for tool/runtime configuration, not for business/query/infrastructure DSL payloads.

## Rationale

1. YAML is human-friendly for authoring and documentation.
2. JSON IR is deterministic for validation, hashing, diffing, testing, and cross-agent handoff.
3. TOML is strong for flat config, but not ideal as primary DSL for nested domain payloads.

## Processing Pipeline

1. Parse external YAML.
2. Validate YAML safety constraints.
3. Normalize into canonical JSON IR.
4. Validate JSON IR against schema/rules.
5. Run generation/execution logic only on JSON IR.

## YAML Safety Constraints

1. Disallow anchors and aliases.
2. Disallow merge keys (`<<`).
3. Disallow custom tags.
4. Disallow duplicate keys.
5. Use UTF-8 text.
6. Use spaces only (no tabs).

## Canonical JSON IR Rules

1. Stable object key ordering.
2. Explicit defaults applied during normalization.
3. Remove fields with `null` when spec marks them as optional and omitted-equivalent.
4. Normalize enums to lowercase.
5. Preserve list order where semantics are order-sensitive.

## Versioning

1. Every DSL payload must include `dsl_version`.
2. Current baseline: `v1`.
3. Backward-incompatible changes require `v2+`.

## Error Codes

1. `E_DSL_PARSE`
2. `E_DSL_UNSAFE_YAML`
3. `E_DSL_DUPLICATE_KEY`
4. `E_DSL_VERSION_UNSUPPORTED`
5. `E_DSL_NORMALIZATION_FAILED`

## Example (author-facing YAML)

```yaml
dsl_version: v1
kind: infra_docker
spec:
  db_engine: pgsql
  db_version: "16"
  db_name: app
  db_user: app
  db_password_var: DB_PASSWORD
  db_port: 5432
  volume_name: pgdata
```

## Example (canonical JSON IR)

```json
{
  "dsl_version": "v1",
  "kind": "infra_docker",
  "spec": {
    "db_engine": "pgsql",
    "db_version": "16",
    "db_name": "app",
    "db_user": "app",
    "db_password_var": "DB_PASSWORD",
    "db_port": 5432,
    "volume_name": "pgdata",
    "service_name": "db",
    "container_name": "db",
    "init_scripts_path": "./docker/db/init",
    "healthcheck": true
  }
}
```
