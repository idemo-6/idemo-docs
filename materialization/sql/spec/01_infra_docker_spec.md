# Spec 01: Infra Docker Generation (from DSL)

Status: draft for discussion
Owner: `AiSqlAgent`

## Goal

Generate infrastructure artifacts from `infra DSL` without executing Docker:
1. `Dockerfile` (optional, only when app-side DB client/runtime config is requested).
2. `docker-compose.yml` (required for containerized DB engines).
3. `.env.example` (required, project-global template).
4. `artifacts.json` manifest (required).

## Scope

1. Supported DB engines:
- `pgsql`
- `mysql`
- `sqlite` (no DB container; file-mode only)

2. Non-goals:
- No `docker compose up`.
- No secret generation.
- No runtime mutation outside artifact files.

Mode alignment:
1. In `dev`, generation and sandbox checks are allowed.
2. In `prod`, this spec is execution-forbidden; only pre-approved stored artifacts may be consumed by orchestrators.
3. See mode policy: [[idemo-docs/materialization/sql/spec/02_execution_modes_dev_prod|Spec 02]].

## Input Contract (`infra DSL` v1)

Required fields:
1. `db_engine` (`pgsql|mysql|sqlite`)
2. `db_version` (engine-specific valid major/minor tag)
3. `db_name`
4. `db_user`
5. `db_password_var` (name of env var, default `DB_PASSWORD`)
6. `db_port`
7. `volume_name`

Optional fields:
1. `service_name` (default `db`)
2. `container_name` (default `<service_name>`)
3. `init_scripts_path` (default `./docker/db/init`)
4. `healthcheck` (`true|false`, default `true`)
5. `mysql_root_password_var` (default `DB_ROOT_PASSWORD`, mysql only)
6. `charset` / `collation` (mysql only)
7. `extensions` (pgsql only; informational unless Dockerfile template is enabled)

## DSL Examples (YAML)

### PostgreSQL

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
  service_name: db
  container_name: db
  init_scripts_path: ./docker/db/init
  healthcheck: true
```

### MySQL

```yaml
dsl_version: v1
kind: infra_docker
spec:
  db_engine: mysql
  db_version: "8.0"
  db_name: app
  db_user: app
  db_password_var: DB_PASSWORD
  mysql_root_password_var: DB_ROOT_PASSWORD
  db_port: 3306
  volume_name: mysqldata
  service_name: db
  container_name: db
  init_scripts_path: ./docker/db/init
  healthcheck: true
  charset: utf8mb4
  collation: utf8mb4_unicode_ci
```

### SQLite

```yaml
dsl_version: v1
kind: infra_docker
spec:
  db_engine: sqlite
  db_version: "3"
  db_name: app
  db_user: app
  db_password_var: DB_PASSWORD
  db_port: 0
  volume_name: sqlite_data
  service_name: db
  container_name: db
  healthcheck: false
```

## Environment Strategy

`.env.example` is project-global and DB config follows Laravel-style keys:
1. `DB_CONNECTION`
2. `DB_HOST`
3. `DB_PORT`
4. `DB_DATABASE`
5. `DB_USERNAME`
6. `DB_PASSWORD`

Optional for MySQL:
1. `DB_ROOT_PASSWORD`

Rules:
1. No literal secrets in generated files.
2. `docker-compose.yml` must reference `${...}` variables.
3. Real secrets are expected in local `.env`, not `.env.example`.

## Compose Mapping Rules

### `pgsql`

1. Image: `postgres:<db_version>`
2. Container env mapping:
- `POSTGRES_DB=${DB_DATABASE}`
- `POSTGRES_USER=${DB_USERNAME}`
- `POSTGRES_PASSWORD=${DB_PASSWORD}`
3. Port mapping:
- `${DB_PORT}:5432`
4. Volume:
- `<volume_name>:/var/lib/postgresql/data`
5. Healthcheck (if enabled):
- `pg_isready -U ${DB_USERNAME} -d ${DB_DATABASE}`

### `mysql`

1. Image: `mysql:<db_version>`
2. Container env mapping:
- `MYSQL_DATABASE=${DB_DATABASE}`
- `MYSQL_USER=${DB_USERNAME}`
- `MYSQL_PASSWORD=${DB_PASSWORD}`
- `MYSQL_ROOT_PASSWORD=${DB_ROOT_PASSWORD}`
3. Port mapping:
- `${DB_PORT}:3306`
4. Volume:
- `<volume_name>:/var/lib/mysql`
5. Healthcheck (if enabled):
- `mysqladmin ping -h 127.0.0.1 -u${DB_USERNAME} -p${DB_PASSWORD}`

### `sqlite`

1. No DB service in compose.
2. `.env.example` still generated with:
- `DB_CONNECTION=sqlite`
- `DB_DATABASE=database/database.sqlite`
3. Optional volume or bind-mount for app service is out of this spec.

## Deterministic Output Rules

1. Stable key order in YAML sections.
2. Stable service ordering (DB service first when present).
3. Stable env key ordering in `.env.example`.
4. Do not include timestamps in generated files.
5. `artifacts.json` must include SHA256 per generated file.

## Validation Rules

1. `db_engine` must be supported.
2. `db_version` must match per-engine allowlist pattern.
3. `db_port` must be numeric in range `1..65535`.
4. `db_password_var` and `mysql_root_password_var` must be valid env var names.
5. `volume_name` must be docker-volume-safe.
6. For mysql: require root password var (default allowed).
7. For sqlite: reject DB service-only fields that are engine-incompatible.

## Error Codes

1. `E_INFRA_PARSE`
2. `E_INFRA_UNSUPPORTED_ENGINE`
3. `E_INFRA_INVALID_VERSION`
4. `E_INFRA_INVALID_PORT`
5. `E_INFRA_INVALID_ENV_NAME`
6. `E_INFRA_INVALID_VOLUME`
7. `E_INFRA_MISSING_REQUIRED`
8. `E_INFRA_ENGINE_FIELD_CONFLICT`
9. `E_INFRA_RENDER_FAILED`

## Output Examples

### `.env.example` (pgsql)

```dotenv
DB_CONNECTION=pgsql
DB_HOST=db
DB_PORT=5432
DB_DATABASE=app
DB_USERNAME=app
DB_PASSWORD=change_me
```

### `.env.example` (mysql)

```dotenv
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=app
DB_USERNAME=app
DB_PASSWORD=change_me
DB_ROOT_PASSWORD=change_me_root
```

## Acceptance Criteria (for implementation)

1. Given valid DSL for `pgsql` or `mysql`, all required artifacts are generated.
2. Given valid DSL for `sqlite`, no DB service is generated but `.env.example` and manifest are generated.
3. Generated compose uses only env references for credentials.
4. Invalid inputs produce coded deterministic errors.
5. Re-running with same input yields byte-identical output files.
6. Any direct generation attempt in `prod` mode fails with mode-policy error.
