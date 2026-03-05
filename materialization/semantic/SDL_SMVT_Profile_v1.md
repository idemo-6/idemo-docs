# SDL/SMVT Profile v1

## Назначение

Описать профиль semantic materialization для IDEMO:
- `SDL` (Semantic Data Layer);
- `SchemaMapping` и semantic bindings;
- `SMVT` как токен управляемого доступа к смысловой проекции данных.

Статус: operational profile.

## 1. SDL layers

1. `Physical Data`
- физические источники (SQL/NoSQL/API), без требований к semantic именованию.

2. `SchemaMapping Registry`
- соответствие `physical_path -> semantic_key`;
- версия binding обязательна.

3. `Semantic Context Projection`
- нормализованная безопасная проекция для phase-local обработки.

4. `SMVT Guard`
- проверка прав, контекста и времени жизни токена перед materialization.

## 1.1 Governance mode

SMVT фиксируется как поддерживаемый механизм платформы, но режим применения задается владельцем системы:

1. `smvt_mode=required`
- materialization допускается только с валидным SMVT.

2. `smvt_mode=optional`
- SMVT может не использоваться в конкретном контуре;
- владелец системы обязан зафиксировать альтернативный access-control policy.

3. `smvt_mode=disabled`
- допускается только в явно обозначенных low-risk/internal контурах;
- требуется явное архитектурное решение и аудит-след.

## 2. Binding model (минимум)

Каждый binding содержит:
- `binding_id`, `version`;
- `physical_path`;
- `semantic_key`;
- `rule` (например, mask/pseudonymize/allow_plain);
- `contexts_allowed`.

Правила:
1. Верхние слои (intent/functions) оперируют только `semantic_key`.
2. Прямой доступ к `physical_path` вне materialization-профиля запрещен.
3. Неоднозначный binding -> hard-fail.

## 3. SMVT lifecycle

1. `Issue`
- токен создается на этапе `collect`/PT-authorize;
- фиксирует `intent_ref`, `context`, `ttl`, `scope_id`.

2. `Use`
- materialization вызовы принимают только валидный `SMVT`.

3. `Transform`
- SDL применяет rules и выдает semantic projection.

4. `Revoke`
- после commit/evaluate токен деактивируется;
- повторное использование отклоняется.

## 4. Phase access policy

1. `collect`: разрешено планирование materialization и выдача SMVT.
2. `analyze/forecast/decide`: разрешено чтение semantic projection (без direct DB I/O).
3. `implement`: разрешены только утвержденные execution paths.
4. `evaluate`: запись опыта/метрик, без re-open SMVT.

## 5. Conformance Signals

1. Missing/expired SMVT -> reject/error.
2. Physical access from phase body -> reject/error.
3. Ambiguous mapping -> reject/error.
4. Reused revoked token -> reject/error.

Примечание по conformance:
- проверки `1` и `4` обязательны при `smvt_mode=required`;
- при `smvt_mode=optional|disabled` обязательна проверка наличия альтернативного access-policy профиля.

## 6. Context Integration Abstraction (normative)

Внешние интеграции абстрагируются в контекстах и исполнителях.
Intent работает только с контекстными/семантическими ссылками.

### 6.1 Context connection contract (пример структуры)

Контекст может включать:
- `name`, `id`, `title`;
- `connection.type`;
- `connection.base_url`;
- `connection.endpoints_map_ref`;
- `connection.auth.secret_ref`.

### 6.2 Invariants

1. `no endpoint in intent`
- в Intent запрещены прямые endpoint/base_url/service URL;
- разрешены только context/semantic references.

2. `secret_ref only`
- секреты/токены не хранятся в Intent и не передаются как literal;
- допускаются только ссылки на секреты (`secret_ref`) через защищенный store.

3. `resolver-only access`
- разрешение физических деталей интеграции выполняет только context resolver/executor;
- phase-local логика не получает прямого доступа к physical connection details.

### 6.3 Conformance checks

1. Endpoint literal в intent -> reject/error.
2. Secret/token literal в intent -> reject/error.
3. Direct connector call из phase body в обход resolver -> reject/error.
