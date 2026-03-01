# Spec 05: Simplified DSL (v1)

Status: draft for discussion
Owner: `AiSqlAgent`

## Goal

Provide a user-friendly DSL for non-SQL users while preserving deterministic generation across full CRUD.

Primary pipeline:
1. `Simplified DSL (YAML/text formulas)`
2. `Canonical CRUD DSL (Spec 04 compatible)`
3. `Canonical IR (JSON)`
4. `SQL template` (dialect-aware)

Output contract:
1. Result of simplified parsing MUST be a full DSL payload compatible with [[idemo-docs/materialization/sql/spec/04_crud_sql_spec|Spec 04]].
2. Direct simplified-to-SQL without canonical CRUD DSL normalization is not allowed in `prod` flow.

`prod` executes only approved artifacts (see Spec 02).

## Design Principles

1. Minimal user syntax; agent resolves complexity from schema.
2. Implicit joins by FK path.
3. Deterministic parsing and rendering.
4. Strict schema-driven validation.
5. No raw SQL passthrough.

## Schema Contract (Required for Simplified DSL)

Minimum schema metadata:
1. `tables.<table>.primaryKey`
2. `tables.<table>.columns[]`
3. `tables.<table>.foreignKeys[]` with:
- `column` (FK column in source table)
- `refTable`

v1 relationship contract:
1. FK references target table `primaryKey` only.
2. Non-PK references are out of scope for v1 (TODO).

## Core Syntax (v1)

### Base clauses

1. `from(table)`
2. `select(item, ...)`
3. `where(cond, ...)`
4. `group(item, ...)`
5. `sum(item [as alias])`, `avg(...)`, `min(...)`, `max(...)`, `count([item] [as alias])`
6. `having(cond, ...)`
7. `order(item [asc|desc], ...)`
8. `limit(n)`, `offset(n)`

### Item and path arguments

1. `column` from main scope: `status`, `sum`
2. FK path:
- `account_id:name`
- `account_id:kind:type:name`

Path resolution rule:
1. Every intermediate token is treated as FK column in current scope.
2. Each step joins to target table by `source.<fk_column> = target.<primaryKey>`.
3. Last token is selected/filtered/grouped column in resolved scope.

### Conditions (functional syntax)

1. `eq(arg, value)`, `ne(arg, value)`
2. `gt(arg, value)`, `gte(arg, value)`, `lt(arg, value)`, `lte(arg, value)`
3. `between(arg, v1, v2)`
4. `in(arg, v1, v2, ...)`
5. `like(arg, pattern)`
6. `is_null(arg)`, `is_not_null(arg)`
7. `and(cond, cond, ...)`, `or(cond, cond, ...)`

### Simplified CRUD formulas (v1)

Read:
1. `find_one(table, select(...), where(...))`
2. `find_many(table, select(...), [where(...)], [order(...)], [limit(...)], [offset(...)])`

Write:
1. `create(table, values(field=value, ...))`
2. `create_many(table, rows(values(...), values(...), ...))`
3. `update(table, set(field=value, ...), where(...))`
4. `delete(table, where(...))`
5. `upsert(table, values(...), conflict(field,...), update(field=value,...))`

Normalization rule:
1. `find_one` -> Spec 04 `operation=select_one`
2. `find_many` -> Spec 04 `operation=select_many`
3. `create` -> Spec 04 `operation=insert_one`
4. `create_many` -> Spec 04 `operation=insert_many`
5. `update` -> Spec 04 `operation=update_where`
6. `delete` -> Spec 04 `operation=delete_where`
7. `upsert` -> Spec 04 `operation=upsert`

CRUD safety propagation:
1. `update(...)` without `where(...)` is rejected in `safe_mode` (maps to Spec 04 write guards).
2. `delete(...)` without `where(...)` is rejected in `safe_mode`.

## Compute Stage (replacing explicit user subqueries)

For row-level expressions before aggregation:
1. `compute(alias = expr, ...)`

Rule:
1. `compute` is evaluated before `group/aggregate`.
2. If computed alias is used in aggregate/having, agent materializes derived stage (subquery/CTE internally).
3. User does not write explicit subquery for this case.

Example intent:
1. `compute(mark_sum = (quantity * (price - margin)) - (sum * 0.06))`
2. `sum(mark_sum as mark_sum)`

## Subqueries (v1 coverage)

Supported in v1:
1. `exists(subquery)`
2. `not_exists(subquery)`
3. `in_subquery(arg, subquery)`
4. Scalar aggregate comparison (internally resolved when required)

Out of v1 (TODO):
1. top-N per group shortcuts
2. named reusable subqueries
3. generic user-authored derived tables
4. user-authored CTE primitives

## Date Formulas (optimized v1)

Keep only high-demand date helpers from legacy list.

### Recommended core set

1. `today([period])`
2. `yesterday([period])`
3. `tomorrow([period])`
4. `date(unit, [amount], [period])`
5. `period(start, end, [arg])`
6. `current_week([period])`, `last_week([period])`, `next_week([period])`
7. `current_month([period])`, `last_month([period])`, `next_month([period])`
8. `current_year([period])`, `last_year([period])`, `next_year([period])`
9. `day_offset([amount], [period])`, `month_offset([amount], [period])`, `year_offset([amount], [period])`

### Optional (keep if business needs confirm)

1. quarter helpers (`current_quarter`, `last_quarter`, `next_quarter`, `quarter_offset`)
2. `exact(...)`
3. unit shortcuts `y/q/m/d/h/i/s`
4. `add(...)`, `sub(...)`

Argument conventions:
1. `period`: `start|end|now|null` (function-dependent)
2. `unit`: `day|week|month|quarter|year`
3. `amount`: integer, default `0`

## Deterministic Parser Output (Canonical IR)

Parser output must include:
1. `kind=crud_query`
2. `dialect`
3. `operation` (Spec 04 operation enum)
4. `table` / `from` (operation dependent)
5. `projections[]` (for read operations)
6. `joins[]` (resolved implicit join chain)
7. `filters_ast`
8. `compute[]`
9. `group_by[]`
10. `aggregates[]`
11. `having_ast`
12. `order_by[]`
13. `limit`, `offset`
14. `values` / `set` / `conflict_target` / `update_set` (operation dependent)
15. `params_schema`
16. `safety_flags`

Canonicalization target:
1. The normalized payload must be serializable directly as Spec 04 input contract.

## Deterministic Rules

1. Same DSL + schema -> same IR + same SQL.
2. Stable join alias generation by encounter order.
3. Stable projection order by declaration.
4. Stable error format: `ERROR: <code>: <reason>`.

## Error Codes (v1 additions)

1. `E_PATH_FK_NOT_FOUND`
2. `E_PATH_COLUMN_NOT_FOUND`
3. `E_PATH_AMBIGUOUS`
4. `E_PATH_DEPTH_LIMIT`
5. `E_COMPUTE_SCOPE_INVALID`
6. `E_COMPUTE_AGG_MIX_FORBIDDEN`

## Examples

### Minimal selection

```txt
from(mark_orders)
select(id, status, sum)
```

### Implicit join by path

```txt
from(mark_orders)
select(account_id:name as account_name, status)
where(eq(account_id:name, 'Retail'))
```

### Multi-hop path + aggregation

```txt
from(mark_orders)
select(account_id:kind:type:name as acc_type)
sum(sum as order_sum)
group(account_id:kind:type:name as acc_type)
```

### Compute before aggregate

```txt
from(mark_orders)
compute(mark_sum = (quantity * (price - margin)) - (sum * 0.06))
select(account_id:name as account_name)
sum(mark_sum as mark_sum)
group(account_id:name as account_name)
```

### Create (insert_one)

```txt
create(users, values(email=:email, name=:name, created_at=:created_at, updated_at=:updated_at))
```

### Update (update_where)

```txt
update(users, set(name=:name, updated_at=:updated_at), where(eq(id, :id)))
```

### Delete (delete_where)

```txt
delete(users, where(eq(id, :id)))
```

### Upsert

```txt
upsert(roles, values(code=:code, title=:title), conflict(code), update(title=:title))
```

## TODO (Post-v1)

1. Non-PK relation support (`references.column` override).
2. Explicit user DSL for advanced derived-table/CTE patterns.
3. Window function primitives.
4. Locale-aware formatting bundles for period labels.

## ICSS-aligned simple operators (v1 Core)

This section defines user-facing simple sugar aligned with ICSS intent semantics.

### Simple sugar vocabulary

1. `filter(cond)` -> normalized to `where(cond)`.
2. `post_filter(cond)` -> normalized to `having(cond)`.
3. `merge(by:path)` -> normalized to FK-based `join` chain (default inner join).
4. `merge(rows)` -> normalized to `union all`.
5. `merge(unique_rows)` -> normalized to `union`.

### ICSS symbolic alignment

1. `/(cond : keep | drop)` maps to `where(cond)` or `having(cond)` depending on stage.
2. `*(rel_a, rel_b, by(path))` maps to `join`.
3. `*(rel_a, rel_b)` maps to `union all` for compatible projections.
4. `*(unique rel_a, rel_b)` maps to `union`.
5. `/(cond : q1 | q2)` at query level is compile-time branch selection of one normalized plan.

### Deterministic precedence and alias policy

1. Boolean tree order is preserved from parsed functional AST.
2. `merge(by:path)` is resolved before set-op merges.
3. Join aliases are assigned deterministically by encounter order (`t0`, `t1`, ...).
4. User alias is accepted only if unique and unambiguous; otherwise reject with deterministic error.

### Canonicalization constraints

1. Sugar must normalize to existing IR fields (`joins`, `filters_ast`, `having_ast`, set-op metadata).
2. Base operation enum of v1 is unchanged.
3. No direct simplified-to-SQL shortcut in prod is allowed.

Reference: [[idemo-docs/materialization/sql/spec/08_sql_icss_mapping_v1_core|Spec 08: ICSS -> SQL Mapping v1 Core]].
