
## Назначение

Зафиксировать policy для принципа:
реализация является вторичной (производной) сущностью относительно Intent,
контекста и опыта, а не самостоятельной онтологической основой системы.

Статус: draft runtime policy profile (refined baseline).

## 1. Core rule

1. Intent первичен.
2. Реализация производна от:
- активного контекста;
- разрешенных операторов;
- накопленного опыта.
3. Реализация не обладает самостоятельным каноническим статусом.
4. Ephemeral в текущей версии трактуется как `secondary`, а не как `transient`.

## 2. Source-of-Truth Policy

1. Source-of-truth для эволюции системы:
- Intent;
- policy/governance constraints;
- experience records.
2. Исполняемый код не является source-of-truth.
3. Долгоживущий код допускается, но его поддержка не является первоочередной задачей человека.

## 3. Human/LLM Responsibility Split

1. Человек в первую очередь поддерживает:
- Intent;
- ограничения контекста;
- правила допуска/ответственности.
2. LLM в первую очередь поддерживает:
- генерацию реализаций;
- исполнение в рамках phase/policy boundaries;
- улучшение реализаций по опыту.
3. Human-in-the-loop включается по risk/policy trigger.

## 4. Execution Model Boundaries

1. Инфраструктурная сложность переносится в execution/context layer (не в бизнес-функции).
2. Intent содержит план оркестрации контекстов/инструментов/системных правил/окружения.
3. Фазовые тела ориентированы на небольшие целевые функции с одной ответственностью.
4. Взаимодействие с внешним миром концентрируется в фазовых переходах.
5. LLM не должен формировать произвольную "общую архитектуру" внутри phase-local логики.

## 5. Conformance Signals

1. При конфликте `Intent/policy` vs existing implementation приоритет у `Intent/policy`.
2. Phase-local функции сохраняют single-responsibility профиль.
3. Побочные эффекты не внедряются в pure-фазы.
4. Production execution without approval -> reject/error.

## 6. Open Questions (for refinement)

1. Переход к fully transient runtime-code остается целевым вариантом будущих версий, но не текущей нормой.
2. Нужен ли отдельный профиль для machine-verifiable quality gates phase-local функций.

## 7. Boundary: Ephemeral vs Approval

### 7.1 Scope split (normative)

1. `Approval policy` отвечает на вопрос: "Можно ли исполнять эту реализацию в production сейчас?"
2. `Ephemeral policy` отвечает на вопрос: "Является ли эта реализация первичной сущностью системы и каков режим ее сопровождения?"

### 7.2 Ownership split

1. Approval ownership:
- authority agents;
- governance/risk/compliance контур.
2. Ephemeral ownership:
- архитектурный runtime-контур Intent/policy/experience;
- инженерный режим сопровождения реализации.

### 7.3 Decision precedence

1. Если `approval=denied`, исполнение запрещено независимо от состояния реализации.
2. Если `approval=granted`, исполнение допускается только при соблюдении phase/policy границ.
3. При конфликте `Intent/policy` и существующей реализации приоритет всегда у `Intent/policy`.

### 7.4 Non-overlap rule

1. Approval не определяет онтологический статус кода.
2. Ephemeral не заменяет approval-gate и не дает права обхода допуска.
