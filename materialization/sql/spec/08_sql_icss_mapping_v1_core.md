# Spec 08: ICSS -> SQL Mapping (v1 Core)

Status: draft for discussion
Owner: `AiSqlAgent`

## Goal

Define a minimal, deterministic mapping between ICSS control symbols and Simplified SQL DSL for `v1 Core`.

Scope:
1. Keep syntax simple for non-SQL users.
2. Preserve deterministic normalization to Spec 04-compatible payload.
3. Keep `v1` free from user-authored CTE/window complexity.

## Position in Pipeline

Mapping applies between:
1. ICSS-level intent forms
2. Simplified DSL formulas
3. Canonical CRUD DSL / IR fields

Pipeline:
`Intent (ICSS)` -> `Simplified DSL` -> `Spec 04 payload` -> `Canonical IR` -> `SQL`.

## Normative Mapping (v1 Core)

1. `/(cond : keep | drop)` -> `where(cond)`.
2. `/(cond : keep | drop)` after `group(...)` -> `having(cond)`.
3. `*(rel_a, rel_b, by(path))` -> `join` (default `inner join`).
4. `*(rel_a, rel_b)` with compatible projection -> `union all`.
5. `*(unique rel_a, rel_b)` -> `union`.
6. `/(cond : q1 | q2)` on query level -> compile-time selection of one normalized query plan.
7. `!` / `!!` -> execution/orchestration policy, not SQL text.
8. `<>` and `**` remain orchestration-level in v1 (no direct SQL primitive).

## Simple Sugar Vocabulary (Natural First)

Accepted sugar for non-technical authoring:
1. `filter(cond)` -> `where(cond)`.
2. `post_filter(cond)` -> `having(cond)`.
3. `merge(by:path)` -> FK-based join-chain.
4. `merge(rows)` -> `union all`.
5. `merge(unique_rows)` -> `union`.

Normalization rule:
- All sugar must normalize into existing IR fields only: `joins`, `filters_ast`, `having_ast`, set-op metadata.
- No change to v1 base `operation` enum is allowed.

## Deterministic Precedence and Alias Policy

1. Condition precedence:
- explicit function tree (`and/or/not`) defines AST order;
- parser must not reorder boolean operands.

2. Merge precedence:
- `merge(by:path)` resolves before set-op merges (`rows`/`unique_rows`).

3. Alias policy:
- stable alias sequence by encounter order: `t0`, `t1`, `t2`, ...
- user alias wins only if unique and unambiguous; else `E_PATH_AMBIGUOUS`.

4. Set-op compatibility:
- same projection arity/order/type class required;
- violation -> deterministic validation error.

## Result Semantics Alignment (CDM/FROR)

1. `Result=0` is reserved for `ApplicabilityFailure`.
2. SQL no-op/zero-cost outcome must not map to `Result=0`.
3. SQL no-op is represented analytically via `fror_zero_class=no_cost_transition` with `Result in {+1,-1}`.

## Test Scenarios (normative)

1. Determinism: same input + same schema -> same IR + same SQL.
2. `filter` vs `post_filter` produce correct WHERE/HAVING split.
3. `merge(by:path)` yields deterministic FK join-chain.
4. `merge(rows)` and `merge(unique_rows)` produce `union all` / `union`.
5. Query-level `/(cond:q1|q2)` picks one branch at compile time.
6. SQL no-op path emits `fror_zero_class` and does not emit `Result=0`.
7. Negative: path ambiguous/not found/depth limit.
8. Negative: write without `where(...)` in `safe_mode`.

## Out of Scope for v1

1. Window functions.
2. User-authored CTE/derived-table primitives.
3. Top-N per group sugar.
4. Non-PK relationship references.
5. SQL calls in phase body (`??/~ /^/>/_`).

## References

1. [[idemo-docs/materialization/sql/spec/05_simplified_dsl_spec|Spec 05: Simplified DSL]]
2. [[idemo-docs/materialization/sql/spec/04_crud_sql_spec|Spec 04: CRUD SQL]]
3. [[idemo-docs/materialization/sql/README|SQL PT @->?? Profile]]
4. [[idemo-machine/docs/parser/spec_unified_v1|Intent parser unified spec]]
