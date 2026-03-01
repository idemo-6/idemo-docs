# IDEMO Runtime Architecture v1

## Назначение

Фиксация практической архитектуры IDEMO-runtime, где:
- `Intent` первичен,
- `ChangeFlow` задает морфологию реализации,
- исполняемая оркестрация выражается через ICSS DSL,
- генерация и исполнение разделены по ролям.

## Связанные документы

- [[IDEMO_Alignment_Matrix]]
- [[IDEMO_Runtime_Checklist_v1]]
- [[CDM/Specifications/Intent-1.1.5]]
- [[FROR/FROR_CDL_bridge]]
- [[intent_parser/README]]

## 1. Базовый runtime-паттерн

Каноническая форма orchestration DSL:

```text
$cf_name(
  @(...),          # collect
  => ??(...)       # analyze
  => ~(...)        # forecast
  => ^(...)        # decide
  => >(...)        # implement
  => _(...)        # evaluate
)
```

Смысл:
- это не «код бизнес-логики»;
- это исполнимая морфология изменения;
- фазы задают границы ответственности.

## 2. Разделение ролей

### 2.1 LLM Agent (phase-local synthesis)

LLM генерирует небольшие целенаправленные функции для конкретной фазы:
- одна функция -> одна ответственность;
- без глобальной оркестрации;
- без права нарушать фазовые ограничения.

LLM не является владельцем commit-последствий.

### 2.2 AiOrcAgent (orchestration authority)

AiOrcAgent:
- собирает композицию фазовых функций;
- маршрутизирует исполнителей;
- управляет PT hooks и контекстной материализацией;
- обеспечивает policy/gov checks;
- фиксирует commit-границу (`implement`) и closure в `evaluate`.

## 3. Фазовая дисциплина

1. `collect`:
- декларация контекстов/ограничений;
- без мутаций состояния.

2. `analyze/forecast/decide`:
- смысловые преобразования и выбор;
- допускаются только pure-операции.

3. `implement`:
- единственная фаза, где разрешена мутация доменного состояния.

4. `evaluate`:
- фиксация результата `Experience in {+1,0,-1}`;
- обновление обучающего контура.

## 4. Инварианты runtime (обязательные)

1. `Intent-first`:
- реализация всегда производна от Intent.

2. `Single-context-per-branch`:
- на ветку исполнения активен один контекст.

3. `Commit-boundary`:
- commit только в implement.

4. `Compensation-not-rewind`:
- rollback после commit трактуется как новый flow-компенсатор.

5. `Experience-closure`:
- каждый завершенный flow обязан иметь evaluate-след.

## 5. Operational contract (минимум)

1. LLM возвращает phase-local артефакты.
2. AiOrcAgent валидирует их против LC/CDL/policy.
3. Композиция выполняется по CF-порядку.
4. После implement накапливается операционная стоимость.
5. evaluate записывает класс результата и обновляет опыт.

## 6. Почему это не «просто очередной workflow»

- Workflow обычно задает процесс исполнения задач.
- IDEMO runtime задает онтологию необратимого изменения:
  - первичен смысл (`Intent`),
  - путь морфологически структурирован (`CF6`),
  - commit имеет причинностный статус,
  - опыт входит в семантику, а не в побочный telemetry.

## 7. Короткий канонический тезис

IDEMO runtime = phase-local synthesis (LLM) + authority orchestration (AiOrcAgent) + irreversible commit discipline (CF5) + experience closure (CF6).
