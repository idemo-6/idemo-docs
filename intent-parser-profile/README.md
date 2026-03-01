# intent_parser

`intent_parser` — машинный профиль формализации Intent/ChangeFlow (ICSS-IntentFlow v1) для валидации, нормализации и конформанс-проверок.

## Статус и границы

- Этот каталог описывает **исполняемую DSL-форму**.
- Он **не заменяет** каноническую онтологию CDM.
- Все решения по смыслу, ролям фаз и контекстной семантике наследуются из канона CDM.

## Каноническая база (обязательно)

Все, что касается парсера, базируется на канонах из `CDM`:

- [ChangeFlow-6](/Volumes/WORK/Project/idemo_docs/IDEMO.DOCS/CDM/Specifications/ChangeFlow-6_v3.md)
- [Intent](/Volumes/WORK/Project/idemo_docs/IDEMO.DOCS/CDM/Specifications/Intent-1.1.5.md)
- [Context Canonical](/Volumes/WORK/Project/idemo_docs/IDEMO.DOCS/CDM/Specifications/Context/Context-Canonical.md)
- [CDL Canonical](/Volumes/WORK/Project/idemo_docs/IDEMO.DOCS/CDM/Specifications/CDL/CDL-Canonical.md)
- [Governance: Canon <-> DSL Synchronization](/Volumes/WORK/Project/idemo_docs/IDEMO.DOCS/CDM/Specifications/Governance/Canonical-DSL-Governance.md)

При конфликте канона и DSL-профиля приоритет всегда у канона CDM.

## Что внутри intent_parser

- `spec_unified_v1.md` — нормативная спецификация DSL/валидатора (source-first для профиля).
- `source_human_v1.md` — человекочитаемое описание смысла языка.
- `runtime/schema/intent_ir_v1.json` — схема нормализованного IR.
- `runtime/tests/conformance/` — минимальный конформанс-набор (валидные/невалидные кейсы).
- `MVP_Scope.md` — границы MVP.
- `legacy/` — исторические материалы эволюции спецификации.

## Инварианты синхронизации с CDM

1. Канонические фазы: `CF1..CF6`.
2. DSL-маркеры (`@`, `??`, `~`, `^`, `>`, `_`) — только синтаксические алиасы представления.
3. `Result/Experience in {+1,0,-1}`, где `0` = неприменимость (неопределенность — частный случай).
4. `FROR.0_no_cost_transition` не маппится в `Result=0`; для этого используется аналитическая метка `fror_zero_class`.
5. `rollback_semantics` фиксируется как `compensation_forward`.
6. Один активный контекст на ветку выполнения.
7. `LC` задает доступность контекстов, `C_coord` решает конфликты внутри доступного множества, `C_meta` управляет активацией/приоритетами контекстов.

## PT @->?? для информационных систем

- Для SQL/DB домена материализация внешних данных выполняется в `pt_hook` между `@(...).collect` и `??(analyze)`.
- В `collect` допускается только планирование (`fn.plan_*`), без прямого внешнего I/O.
- `analyze` работает только с уже материализованным снимком данных.
- Профиль и контракты SQL-агентов: [SQL PT @->?? Profile](/Volumes/WORK/Project/idemo_docs/IDEMO.DOCS/dsls/sql/README.md).

## Как вносить изменения

Перед изменениями проверять governance-документ:

- [Canonical-DSL-Governance](/Volumes/WORK/Project/idemo_docs/IDEMO.DOCS/CDM/Specifications/Governance/Canonical-DSL-Governance.md)

Типы изменений:

- `C` — изменения канона (CDM),
- `P` — изменения профиля (`intent_parser`),
- `CP` — сквозные изменения (канон + профиль + conformance).

Для `P/CP`-изменений должны быть синхронно обновлены:

- `spec_unified_v1.md`,
- `runtime/schema/intent_ir_v1.json` (если меняется структура),
- `runtime/tests/conformance/*` (если меняется валидация/поведение).
