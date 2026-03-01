# AiSql Agents Roadmap

## Final Agent Model

1. `AiSqlAgent` — design, validation, and generation.
2. `AiSqlExecutorAgent` — controlled and secure execution.

## 1) AiSqlAgent Scope

1. DSL parsing and validation:
- Syntax validation (`E_PARSE` and related errors).
- Semantic validation against schema context (tables, columns, aliases, joins).
- Data rules validation (`rules DSL` with Laravel-like syntax + internal IR model).

2. SQL generation:
- `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `UPSERT`.
- Clauses: `WHERE`, `JOIN`, `GROUP/HAVING`, `ORDER`, `LIMIT/OFFSET`.
- Deterministic SQL rendering by dialect (`pgsql`, `mysql`, `sqlite`).

3. Migrations (SQL-first):
- Generate `up/down` migrations.
- Schema diff between current and target state.
- Safety policy for destructive operations (`drop`, risky alters).

4. Infrastructure artifacts (generation only):
- `Dockerfile`, `docker-compose.yml`, `.env.example`.
- DB config, healthcheck, volume/init scripts.

5. Contracts and outputs:
- Stable error codes and message format.
- Machine-readable output artifacts for executor handoff.

## 2) AiSqlExecutorAgent Scope

1. Query execution:
- DB connections and pooling.
- Transactions, timeouts, retry policy.

2. Safety and policy:
- `read-only` and `read-write` modes.
- Dangerous command restrictions via policy.
- Strict parameter binding (no string concatenation execution path).

3. Execution controls:
- `dry-run`, `EXPLAIN`, row limits.
- Heavy query safeguards.

4. Observability:
- Audit log (`who`, `what`, `when`, `result`).
- Metrics (`latency`, `error rate`, `success rate`).
- Trace/correlation support.

## 3) What We Explicitly Do Not Split Into Separate Agents

1. Validation stays inside `AiSqlAgent` as a module, not an independent agent.
2. Docker provisioning is artifact generation in `AiSqlAgent`, not runtime execution.
3. ORM adapter is optional and postponed; SQL-first is the baseline.

## 4) Delivery Priority

### MVP

1. `AiSqlAgent`:
- CRUD SQL generation.
- Schema validation.
- Rules validation v1.
- Stable error codes.

2. `AiSqlExecutorAgent`:
- Safe execution baseline.
- Transactions.
- Audit logging.
- Dry-run mode.

### V1.5

1. Migration engine (SQL-first).
2. `EXPLAIN` guardrails and performance checks.

### V2

1. Docker artifact generation.
2. Optional ORM adapters.

## 5) Rules Validation Strategy (v1)

1. External syntax: Laravel-like rules for developer ergonomics.
2. Internal representation: canonical IR for deterministic processing.
3. Initial rule families:
- Types: `string`, `int`, `numeric`, `bool`, `date`.
- Presence: `required`, `nullable`.
- Limits: `min`, `max`, `between`.
- Sets: `in`, `not_in`.
- DB-linked: `exists`, `unique`.
- Cross-field: `required_if`, `same`, `different`.

## 6) Exit Criteria Per Stage

1. MVP exit:
- Golden tests for DSL-to-SQL pass for CRUD set.
- Validation errors are deterministic and coded.
- Executor blocks unsafe commands in `read-only`.

2. V1.5 exit:
- Migration generation includes reversible `up/down`.
- Destructive migration requires explicit opt-in.

3. V2 exit:
- Generated Docker artifacts run with a clean local bootstrap.
- Artifact templates are dialect-aware and versioned.

## 7) Detailed Specs

1. [[idemo-docs/materialization/sql/spec/00_dsl_format_policy|Spec 00: DSL Format Policy]]
2. [[idemo-docs/materialization/sql/spec/01_infra_docker_spec|Spec 01: Infra Docker Generation]]
3. [[idemo-docs/materialization/sql/spec/02_execution_modes_dev_prod|Spec 02: Execution Modes (dev/prod)]]
4. [[idemo-docs/materialization/sql/spec/03_migrations_sql_first_spec|Spec 03: Migrations (SQL-first)]]
5. [[idemo-docs/materialization/sql/spec/04_crud_sql_spec|Spec 04: CRUD SQL (SQL-first)]]
6. [[idemo-docs/materialization/sql/spec/05_simplified_dsl_spec|Spec 05: Simplified DSL (v1)]]
7. [[idemo-docs/materialization/sql/spec/06_sql_executor_spec|Spec 06: SQL Executor Agent]]
8. [[idemo-docs/materialization/sql/spec/07_sql_agents_context_contract|Spec 07: SQL Agents Context Contract]]

## 8) TODO (Post-v1)

1. Non-PK relationship references (explicit override only, currently out of v1 contract).
2. Expanded subquery DSL coverage beyond v1 core:
- derived-table primitives,
- top-N per group shortcuts,
- reusable named subqueries.
3. Window-function DSL helpers.
4. CTE as user-facing primitive.
5. Advanced optimizer/plan-guided rewrites.
