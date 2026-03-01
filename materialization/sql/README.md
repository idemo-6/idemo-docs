# SQL PT @->?? Profile (IDEMO / IntentFlow)

## Назначение

Этот профиль фиксирует содержательный контракт для перехода `@(...).collect -> [pt.*] -> ??(analyze)` в информационных системах.

Ключевой принцип:
- `collect` планирует запросы.
- `pt_hook` между `@` и `??` материализует данные.
- `analyze` анализирует уже полученный снимок без нового внешнего DB I/O.

## Канонический поток

1. `@(...).collect`:
- объявление контекстов;
- сборка планов (`fn.plan_*`, включая SQL-ориентированные планы);
- без исполнения SQL.

2. `=> [pt.*]` между `@` и `??`:
- исполнение approved SQL artifact или разрешенного fetch-оператора;
- materialization в `data_snapshot`;
- фиксация `execution_id`, `artifact_id/version/checksum`, `duration`, trace/audit метаданных.

3. `??(analyze)`:
- только `fn.*`/`guard` анализ по `data_snapshot`;
- без прямых SQL/DB вызовов.

## Интеграция с SQL-спеками

- DSL и нормализация: [[idemo-docs/materialization/sql/spec/05_simplified_dsl_spec|Spec 05: Simplified DSL]]
- Исполнение артефактов: [[idemo-docs/materialization/sql/spec/06_sql_executor_spec|Spec 06: SQL Executor Agent]]
- Контекстный envelope: [[idemo-docs/materialization/sql/spec/07_sql_agents_context_contract|Spec 07: SQL Agents Context Contract]]

## Политика `Result`/`transition_code` (CDM/FROR согласование)

- `0` зарезервирован для `ApplicabilityFailure` на уровне CDM runtime.
- SQL no-op/zero-cost исход не должен маппиться в `Result=0`.
- Для no-op/no-cost используется аналитическая метка (`fror_zero_class=no_cost_transition`) при `Result in {+1,-1}`.

## Минимальный PT-контракт для SQL materialization

Вход `pt.sql_execute_artifact(...)`:
- `artifact_id`, `version`
- `runtime_params`
- `context_envelope`
- `request_id`, `trace_id`

Выход:
- `data_snapshot` (или `rows` + typed projection)
- `execution_meta`:
  - `execution_id`
  - `artifact_id`, `version`, `checksum`
  - `duration_ms`
  - `policy_decision`
- `transition_meta`:
  - `transition_result`
  - `transition_code` (`+1|-1`, `0` только для ApplicabilityFailure)
  - `fror_zero_class` (`none|no_cost_transition`)
  - `tau_ms`
  - `experience_event_ref`

## Границы

- Прямой SQL в phase body (`??`, `~`, `^`, `>`, `_`) не допускается.
- Генерация SQL и исполнение SQL разделены по агентам (design vs execute).
- В `prod` — только approved artifacts.

## How to choose syntax

Use `Natural First` by default:
1. For business users, start with simple formulas (`filter`, `merge`, `find_many`, `update`, etc.).
2. Let parser normalize to canonical CRUD DSL and IR.
3. Use symbolic ICSS forms (`/(cond)`, `*`) only for advanced/explicit control semantics.

Normative mapping details:
- [[idemo-docs/materialization/sql/spec/08_sql_icss_mapping_v1_core|Spec 08: ICSS -> SQL Mapping v1 Core]]
