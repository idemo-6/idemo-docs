# AiSqlAgent DSL Checklist (v2)

This checklist is intended for:
- spec validation,
- regression testing,
- documentation-by-examples.

## Baseline Schema

```yaml
tables:
  events:
    primaryKey: id
    columns: [id, user_id, product_id, occurred_at, amount, status]
  users:
    primaryKey: id
    columns: [id, name, email, country_id, created_at]
  countries:
    primaryKey: id
    columns: [id, name, code]
  orders:
    primaryKey: id
    columns: [id, user_id, total, created_at, status]
  products:
    primaryKey: id
    columns: [id, name, category, price]
```

## Baseline Config

```yaml
format_keys: [months, M.YYYY, YYYY]
allowed_functions_extra: [coalesce_safe]
```

## Cases

### 01. Minimal valid query (default dialect pgsql)

DSL:
```txt
from(events)
select(main.id, main.status)
```

Expected SQL:
```sql
SELECT "main"."id", "main"."status" FROM "events" AS "main"
```

### 02. WHERE comma means AND

DSL:
```txt
from(events)
select(main.id)
where(main.status = 'paid', main.amount > 100)
```

Expected SQL:
```sql
SELECT "main"."id" FROM "events" AS "main" WHERE "main"."status" = 'paid' AND "main"."amount" > 100
```

### 03. Nested logical groups

DSL:
```txt
from(events)
select(main.id)
where(or(main.status = 'paid', and(main.amount > 100, main.amount < 500)))
```

Expected SQL:
```sql
SELECT "main"."id" FROM "events" AS "main" WHERE ("main"."status" = 'paid' OR ("main"."amount" > 100 AND "main"."amount" < 500))
```

### 04. `select(*)` expansion (main table only)

DSL:
```txt
from(events)
select(*)
```

Expected SQL:
```sql
SELECT "main"."id", "main"."user_id", "main"."product_id", "main"."occurred_at", "main"."amount", "main"."status" FROM "events" AS "main"
```

### 05. DISTINCT + ORDER default ASC

DSL:
```txt
from(events)
distinct()
select(main.user_id)
order(main.user_id)
```

Expected SQL:
```sql
SELECT DISTINCT "main"."user_id" FROM "events" AS "main" ORDER BY "main"."user_id" ASC
```

### 06. Auto join (`::ref.fk`)

DSL:
```txt
from(events)
select(main.id, u.name)
left_join(main:users::id.user_id, as u)
```

Expected SQL:
```sql
SELECT "main"."id", "u"."name" FROM "events" AS "main" LEFT JOIN "users" AS "u" ON "u"."id" = "main"."user_id"
```

### 07. Manual join (`on(...)`)

DSL:
```txt
from(events)
select(main.id, users.name)
inner_join(users, on(main.user_id = users.id))
```

Expected SQL:
```sql
SELECT "main"."id", "users"."name" FROM "events" AS "main" INNER JOIN "users" AS "users" ON "main"."user_id" = "users"."id"
```

### 08. Join strategy conflict

DSL:
```txt
from(events)
select(main.id)
left_join(main:users::id.user_id, on(main.user_id = users.id))
```

Expected error:
```txt
ERROR: E_JOIN_STRATEGY_CONFLICT: ...
```

### 09. Join strategy missing

DSL:
```txt
from(events)
select(main.id)
left_join(users)
```

Expected error:
```txt
ERROR: E_JOIN_STRATEGY_MISSING: ...
```

### 10. Unknown table

DSL:
```txt
from(unknown_table)
select(main.id)
```

Expected error:
```txt
ERROR: E_UNKNOWN_TABLE: ...
```

### 11. Unknown column

DSL:
```txt
from(events)
select(main.not_exists)
```

Expected error:
```txt
ERROR: E_UNKNOWN_COLUMN: ...
```

### 12. Unknown scope

DSL:
```txt
from(events)
select(e.id)
```

Expected error:
```txt
ERROR: E_UNKNOWN_SCOPE: ...
```

### 13. Duplicate alias

DSL:
```txt
from(events, as x)
select(x.id)
left_join(users, on(x.user_id = users.id), as x)
```

Expected error:
```txt
ERROR: E_DUPLICATE_ALIAS: ...
```

### 14. Aggregate + GROUP + HAVING

DSL:
```txt
from(events)
select(main.status)
sum(main.amount, as total_amount)
group(main.status)
having(sum(main.amount) > 100)
```

Expected SQL:
```sql
SELECT "main"."status", SUM("main"."amount") AS "total_amount" FROM "events" AS "main" GROUP BY "main"."status" HAVING SUM("main"."amount") > 100
```

### 15. Aggregate alias auto-generation

DSL:
```txt
from(events)
select(main.status)
sum(main.amount)
group(main.status)
```

Expected SQL:
```sql
SELECT "main"."status", SUM("main"."amount") AS "agg_1" FROM "events" AS "main" GROUP BY "main"."status"
```

### 16. `group_joins` chain (last ref projected and grouped)

DSL:
```txt
from(events)
select(main.status)
group_joins(main:users::id.user_id, countries::name.country_id, as country_name)
sum(main.amount, as total_amount)
group(main.status)
```

Expected SQL:
```sql
SELECT "main"."status", SUM("main"."amount") AS "total_amount", "countries"."name" AS "country_name" FROM "events" AS "main" INNER JOIN "users" AS "users" ON "users"."id" = "main"."user_id" INNER JOIN "countries" AS "countries" ON "countries"."id" = "users"."country_id" GROUP BY "main"."status", "countries"."name"
```

### 17. `format_key` without preceding alias

DSL:
```txt
from(events)
select(main.id)
group('YYYY')
```

Expected error:
```txt
ERROR: E_FORMAT_NO_ALIAS: ...
```

### 18. EXISTS subquery

DSL:
```txt
from(users)
select(main.id, main.email)
where(exists(sub_query(orders, as o) sub_select(o.id) sub_where(o.user_id = main.id)))
```

Expected SQL:
```sql
SELECT "main"."id", "main"."email" FROM "users" AS "main" WHERE EXISTS (SELECT "o"."id" FROM "orders" AS "o" WHERE "o"."user_id" = "main"."id")
```

### 19. EXISTS with forbidden order/limit/offset (parse-level reject)

DSL:
```txt
from(users)
select(main.id)
where(exists(sub_query(orders, as o) sub_select(o.id) order(o.id)))
```

Expected error:
```txt
ERROR: E_PARSE: ...
```

### 20. Unknown function

DSL:
```txt
from(events)
select(bad_func(main.amount))
```

Expected error:
```txt
ERROR: E_UNKNOWN_FUNCTION: ...
```

### 21. Extra function allowed by config

DSL:
```txt
from(events)
select(coalesce_safe(main.amount, 0))
```

Expected:
```txt
Valid query (function name allowed by config.allowed_functions_extra).
```

### 22. Unqualified column reference (grammar reject)

DSL:
```txt
from(events)
select(id)
```

Expected error:
```txt
ERROR: E_PARSE: ...
```

### 23. Missing schema for `select(*)`

Input conditions:
```txt
Schema context is missing or empty.
```

DSL:
```txt
from(events)
select(*)
```

Expected error:
```txt
ERROR: E_SCHEMA_REQUIRED_FOR_STAR: ...
```

### 24. MySQL dialect quoting

DSL:
```txt
db(mysql)
from(events)
select(main.id, main.status)
limit(10)
offset(20)
```

Expected SQL:
```sql
SELECT `main`.`id`, `main`.`status` FROM `events` AS `main` LIMIT 10 OFFSET 20
```

