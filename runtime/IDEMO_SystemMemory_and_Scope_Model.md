# IDEMO SystemMemory and Scope Model v1

## Назначение

Зафиксировать минимальную runtime-модель:
- `SystemMemory` как операционное пространство состояния;
- `Isolation-by-Scope` как обязательную дисциплину до commit;
- правила merge/конфликтов в `implement`.

Статус: operational profile (не заменяет канон CDM/FROR).

## 1. SystemMemory (операционный контракт)

1. `stateMemory`
- актуальное состояние системы-носителя;
- обновляется только в `implement` после прохождения policy/authority-gates.

2. `dataMemory`
- фактические данные и материализованные снимки;
- чтение возможно в `collect/analyze/...`, запись доменных изменений — только в `implement`.

3. `experienceMemory`
- журнал исходов (`+1/0/-1`), трассы, ссылки на `Approved(+1)` реализации;
- пополняется в `evaluate` и может использоваться в анализе/прогнозе.

## 2. Isolation-by-Scope

Каждый flow исполняется в `current_flow_scope`.

Правила:
1. До `implement` изменения остаются в локальном scope и не мутируют глобальное состояние.
2. `collect/analyze/forecast/decide` работают с теневой проекцией (`SystemMemory + local deltas`).
3. Глобальный merge разрешен только в commit-boundary (`implement`).

## 2.1 Intent-Context-Policy Coupling

Базовый runtime-инвариант:

1. Intent без контекста не имеет операционной применимости.
2. Контекст без Intent не инициирует управляемое изменение.
3. Policy является частью активного контекста исполнения и определяет допустимость действий.

Формально (operational):
- `Applicable(Intent) <=> exists C_active : Policy(C_active) allows Intent`.
- `not exists C_active -> ApplicabilityFailure -> Result=0`.

## 2.2 Context and Operation Experience

Разделение ответственности:

1. `operation_experience_result` хранится в `experienceMemory` как источник истории/обучения.
2. Контекст хранит не историю, а правила и ссылки на релевантный опыт:
- `operation_experience_policy` (что допустимо использовать);
- `experience_refs`/`experience_query_ref` (как получить релевантный срез опыта).

Минимальная policy-проекция опыта в контексте:
- `allowed_result_classes` (например, только `+1`);
- `max_age`;
- `min_confidence`;
- `require_attested`.

## 3. Commit and Conflict Policy

1. Единственная точка commit: `implement`.
2. Merge выполняется атомарно.
3. Если обнаружен конфликт актуальности/политики:
- commit отклоняется;
- flow переводится в компенсационный или re-analyze путь;
- полная инверсия истории не допускается.

## 4. Trace Requirements (минимум)

Для каждого flow обязательны (ближайшая версия runtime-профиля):
- `flow_id`, `scope_id`;
- `commit_attempted: bool`;
- `commit_result: success|rejected`;
- `conflict_class` (если есть);
- `rollback_semantics: compensation_forward`;
- `experience_event_ref`.

## 5. Conformance Signals

1. Mutation outside implement -> reject/error.
2. Commit conflict -> no silent overwrite.
3. Rejected commit -> no global state change.
4. Completed flow -> evaluate closure is present.
