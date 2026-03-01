# IDEMO Runtime Checklist v1

## Назначение

Краткий pre-release / pre-prod чеклист для проверки, что IDEMO runtime
сохраняет канон Intent, фазовую дисциплину CF6 и FROR-инварианты.

## Связанные документы

- [[IDEMO_Runtime_Architecture_v1]]
- [[IDEMO_Alignment_Matrix]]
- [[FROR/FROR_CDL_bridge]]
- [[intent_parser/README]]

## A. Канон и спецификации

1. Актуальная версия `Intent`-канона зафиксирована для релиза.
2. Версия `intent_parser/spec_unified_v1.md` зафиксирована и тегирована.
3. Схема `intent_ir_v1.json` синхронизирована со spec.
4. Governance-правила Canon <-> DSL проверены для текущих изменений.

## B. Фазовая дисциплина CF6

1. В runtime соблюдается порядок фаз (`analyze->forecast->decide->implement->evaluate`) или валидный поднабор.
2. `collect` используется только для деклараций/контекста.
3. Мутация состояния разрешена только в `implement`.
4. `evaluate` обязателен для любого завершенного flow.

## C. Контекст и применимость

1. На ветку исполнения активен один контекст.
2. Все materialized contexts предварительно объявлены в collect.
3. LC-policy ограничивает contexts/operators/functions.
4. `Result=0` используется только как неприменимость в активном контексте.

## D. Commit и rollback

1. Commit-boundary формально зафиксирован в implement.
2. Нет примитивов «полной инверсии истории» как штатной операции.
3. rollback после commit реализован как `compensation_forward`.
4. Есть audit-trace причин и эффектов компенсации.

## E. FROR-инварианты в runtime

1. Для нетривиального перехода есть положительная цена (или согласованная прокси-метрика).
2. Накопленная цена (`tau`) трассируется по flow.
3. Тринарная классификация исходов маппится однозначно.
4. Нарушения инвариантов переводятся в reject/error, а не молчаливо игнорируются.

## F. LLM и Orchestrator границы

1. LLM генерирует только phase-local артефакты (single responsibility).
2. LLM не имеет прямого права на state mutation.
3. AiOrcAgent владеет композицией, маршрутизацией и policy-checks.
4. Human override и authority boundaries документированы.

## G. Наблюдаемость и конформанс

1. Включены conformance-тесты на grammar/policy/phase rules.
2. Есть тест-кейсы на `Result in {+1,0,-1}`.
3. Есть тест на запрет mutation вне implement.
4. Есть тест на компенсационный rollback.
5. Метрики runtime (latency, failure class, tau/cost proxy) публикуются.

## H. Go/No-Go

Go, если:
- критические пункты B3, B4, D1, D3, E1, E2, F2 выполнены;
- conformance-suite проходит без критических ошибок.

No-Go, если:
- обнаружены обходы commit-boundary;
- отсутствует трассировка цены или evaluate-closure;
- rollback реализован как скрытая инверсия без компенсационного следа.

