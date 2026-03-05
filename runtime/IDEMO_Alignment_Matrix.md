# IDEMO Alignment Matrix

## Назначение

Единый контрольный слой синхронизации:
- канона Intent (CDM),
- исполнимой формализации ICSS/IntentFlow (`intent_parser`),
- онтологических инвариантов FROR/CDM.

## Источники канона

- [Intent-1.1.5](../../fcdm-core/theory/cdm/Specifications/Intent-1.1.5.md)
- [ChangeOperators-Canonical](../../fcdm-core/theory/cdm/Specifications/Operators/ChangeOperators-Canonical.md)
- [ChangeOperators-Access-Profile](../../fcdm-core/theory/cdm/Specifications/Operators/ChangeOperators-Access-Profile.md)
- [TIL-Implementation-Profile](../../fcdm-core/theory/cdm/Specifications/Operators/TIL-Implementation-Profile.md)
- [TIL-Lifecycle-Profile](../../fcdm-core/theory/cdm/Specifications/Operators/TIL-Lifecycle-Profile.md)
- [Operator-Novelty-Constraint-Profile](../../fcdm-core/theory/cdm/Specifications/Operators/Operator-Novelty-Constraint-Profile.md)
- [ChangeFlow-6_v3](../../fcdm-core/theory/cdm/Specifications/ChangeFlow-6_v3.md)
- [CtxL-Canonical](../../fcdm-core/theory/cdm/Specifications/CtxL/CtxL-Canonical.md)
- [FROR_CDM_bridge](../../fcdm-core/theory/cdm/bridge/FROR_CDM_bridge.md)
- [IDEMO_Runtime_Architecture_v1](./IDEMO_Runtime_Architecture_v1.md)
- [IDEMO_Runtime_Checklist_v1](./IDEMO_Runtime_Checklist_v1.md)
- [IDEMO_SystemMemory_and_Scope_Model](./IDEMO_SystemMemory_and_Scope_Model.md)
- [Ephemeral_Implementation_Policy](./Ephemeral_Implementation_Policy.md)
- [System-Canonical](../../fcdm-core/theory/cdm/Specifications/System/System-Canonical.md)
- [Identity-Canonical](../../fcdm-core/theory/cdm/Specifications/System/Identity-Canonical.md)
- [Identity-Scoring-Profile](../../fcdm-core/theory/cdm/Specifications/System/Identity-Scoring-Profile.md)
- [System-Classification-Profile](../../fcdm-core/theory/cdm/Specifications/System/System-Classification-Profile.md)
- [RLC-CC-Profile](../../fcdm-core/theory/cdm/Specifications/AppliedRules/RLC-CC-Profile.md)
- [ROS-Profile](../../fcdm-core/theory/cdm/Specifications/AppliedRules/ROS-Profile.md)
- [SEC-OC-Profile](../../fcdm-core/theory/cdm/Specifications/AppliedRules/SEC-OC-Profile.md)
- [Observer-Profile](../../fcdm-core/theory/cdm/Specifications/AppliedRules/Observer-Profile.md)
- [SQL PT @->?? Profile](../materialization/sql/README.md)
- [Spec 08: ICSS -> SQL Mapping v1 Core](../materialization/sql/spec/08_sql_icss_mapping_v1_core.md)
- [SDL/SMVT Profile v1](../materialization/semantic/SDL_SMVT_Profile_v1.md)
- [Post-Anthropocentric Computing Manifesto (vision)](../vision/Post-Anthropocentric_Computing_Manifesto.md)
- [Manifesto to Runtime Mapping](../vision/Manifesto_to_Runtime_Mapping.md)

## Матрица соответствий

| Канонический элемент | Intent-канон (CDM) | ICSS / runtime (`intent_parser`) | FROR/CDM bridge | Конформанс-критерий |
|---|---|---|---|---|
| Intent как первичная единица | Intent задает `what`, без `how` | Root intent + collect-first grammar | CDM = канонический операционный слой, не замена инвариантов | В phase-body нет hardcoded реализационной семантики Intent |
| Контекстуальность Intent | `Intent` определен только относительно `C_active` | `ctx` declaration в `@(...)`, `$ctx(...)` в use | Один активный контекст на ветку | Ошибка на materialized context вне declared/LC allowed |
| Тринарность результата | `Experience in {+1,0,-1}`, `0 <=> ApplicabilityFailure` | `Result/Experience` и `result_lit` поддерживаются | `{-,0,+}` сохраняется, но `0` разделяется: `0_structural -> Result=0`, `0_no_cost -> fror_zero_class` | Runtime не смешивает неприменимость и no-cost переход |
| Морфология ChangeFlow | `CF1..CF6`, CF5 как commit | Фазовый pipeline `analyze..evaluate` + hooks | `CF5` как необратимая точка commit | Нет state mutation вне `implement` |
| Необратимость | Коррекция только через новый ChangeFlow | rollback моделируется через новый flow | rollback = compensation_forward | Нет механизма "полной инверсии истории" как штатного паттерна |
| Цена фиксации | Изменение не бесплатно (по FROR-инвариантам) | Частично через imp/pt дисциплину, явной метрики нет | `c` и `τ` обязательны в bridge | Для commit есть измеримая цена или прокси |
| Накопленная цена (`τ`) | Подразумевается через необратимость/процесс | В IR явного `tau_acc` пока нет | `τ` как скаляр необратимости | Трасса должна поддерживать аккумулятор цены |
| Структурный детерминизм | Один Intent -> допустимое `ΔS`, много траекторий | Deterministic grammar + policy-constrained execution | Онтологический, не алгоритмический детерминизм | Для одного Intent допустимы разные планы при одинаковом целевом классе эффекта |
| Distributed realization | CF1-4/6 могут быть распределены, CF5 локален | PT hooks + authority constraints | Commit point локализован | Локальный authority-agent фиксируется в implement |
| Intent-Context-Policy coupling | Intent применим только в активном контексте с допустимой policy | `ctx` declaration/use + LC/policy allowlists | `Result=0 <=> ApplicabilityFailure` сохраняется | Нет активного контекста/допуска -> `Result=0`; context-only без intent не запускает change |
| Experience-policy coupling | Операционный опыт участвует в решениях через policy, а не как raw контекстные данные | `operation_experience_policy` + refs/query к `experienceMemory` | Сохраняется разделение: context conditions vs experience history | В контексте нет raw history; используется policy-filter + traceable references |
| Isolation-by-scope | Изменение состояния связано с фазовой дисциплиной и commit-boundary | `current_flow_scope`, merge only in implement | rollback = compensation-forward, no silent overwrite | Нет глобальной мутации до commit; trace содержит `scope_id`, `commit_result`, `conflict_class` |
| Semantic materialization access | Контекст определяет применимость и допустимость работы с данными | SDL/SMVT профиль + binding model (`smvt_mode` configurable) | Контекстная применимость + FROR/CDM safety constraints | Нет direct physical access из phase body; SMVT-проверки обязательны при `smvt_mode=required` |
| Вторичность реализации (Ephemeral as secondary) | Intent/policy/experience первичны, реализация производна | Ephemeral Implementation Policy v1 (secondary, not necessarily transient) | Сохраняется primacy Intent и управляемая эволюция реализации | При конфликте приоритет у Intent/policy; код не трактуется как source-of-truth |
| Граница Ephemeral vs Approval | Допуск к исполнению и онтологический статус реализации разведены | Ephemeral policy section 7 + runtime approval policies | Approval gate обязателен и не замещается lifecycle-политикой реализации | Нет обхода approval через ссылки на "вторичность" реализации |
| SQL materialization profile | Для информационных систем materialization выносится в `pt @->??` | `collect` планирует (`fn.plan_*`), `pt` исполняет SQL artifact, `analyze` остается pure | `Result=0` только для ApplicabilityFailure, no-cost -> `fror_zero_class` | Нет SQL/DB вызовов в phase body; SQL no-op не маппится в `Result=0` |

## Текущий статус (по факту документов)

### Полное совпадение

- Intent primacy и производность реализации.
- Контекстная семантика и ApplicabilityFailure.
- CF5 как commit-точка и отсутствие mutation вне implement.
- Тринарная модель результата (в каноне и bridge).

### Разрывы, которые стоит закрыть

1. Поля `fror_metrics.*`, `fror_zero_class` и `rollback_semantics` добавлены в контракт IR, но еще не подтверждена их обязательная эмиссия в runtime-трассах.
2. Нужна проверка, что SQL/executor и orchestrator везде сохраняют правило: `Result=0` только для `ApplicabilityFailure`.
3. Нужен сквозной автопрогон conformance-кейсов (включая новые SQL/Result-mapping сценарии) в CI.
4. Требуется финализировать правила governance для версионирования SQL-спеков внутри `IDEMO.DOCS/dsls/sql`.

## Минимальный пакет синхронизации (P1)

### Выполнено

1. Зафиксирован `result_domain` default как `[-1,0,+1]` в спецификации профиля.
2. В IR добавлены поля:
   - `fror_metrics.step_cost`
   - `fror_metrics.tau_acc`
   - `fror_zero_class` (`none|no_cost_transition`)
   - `rollback_semantics: compensation_forward`
3. Добавлены conformance-кейсы для проверки `result_domain` с `0`/без `0`.

### Осталось закрыть

1. Подтвердить обязательную эмиссию `fror_metrics.*` и `fror_zero_class` в runtime-трассах.
2. Добавить/прогнать conformance-сценарии:
   - reject direct full rollback primitive;
   - accumulate tau on nontrivial implement;
   - enforce zero-class only via applicability failure mapping;
   - enforce no-cost transition mapping via `fror_zero_class` при `Result in {+1,-1}`.

## Канонический тезис синхронизации

IDEMO корректен как вычислительная парадигма тогда и только тогда, когда:
- `Intent` остается онтологически первичным,
- `ICSS` остается исполнимой нормализацией,
- `FROR` инварианты (цена, необратимость, компенсационность rollback) сохраняются в runtime.
