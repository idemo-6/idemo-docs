# ICSS-IntentFlow — Human Source v1

Этот документ описывает смысл языка для людей.
Формальная версия для парсеров/валидаторов: `idemo-machine/docs/parser/spec_unified_v1.md`.

## 1. Зачем нужен язык

IntentFlow нужен, чтобы задавать процесс ChangeFlow-6 детерминированно:
- что мы объявляем как возможные зависимости,
- что реально используем по условиям,
- где именно разрешены внешние взаимодействия,
- где разрешены изменения доменного состояния.

Главный принцип: не смешивать декларацию, вычисление и эффекты.

## 2. Три разных операции (их нельзя смешивать)

1. `declare(ctx)`
- Объявление доступных контекстов.
- Разрешено только в `@(...)`.

2. `use(ctx)`
- Чистое (pure) условное использование контекста в логике.
- Обычно через план (`FetchPlan`) и ветвления.
- Не делает I/O само по себе.

3. `materialize(ctx)`
- Реальное обращение к внешним данным/системам.
- Только через `pt.*`.
- Разрешено только в PT hook и transition effects.

## 3. Каноническая модель v1

- Ровно один корневой блок intent: `(...)`.
- Внутри него ровно один collect/declaration блок: `@(...)`.
- Все fetch-планы проектируются только в `@(...).collect`.
- Intent без фаз допустим: `@(...)`-only (пустой flow).
- В обычных фазах нет I/O.
- В `implement` разрешены `imp.*` (мутация домена) и `fn.*` (чистая подготовка).
- Контексты объявляются только в `@(...)`.
- Контексты могут использоваться условно (pure).
- Материализация контекста только через PT.

## 4. Контексты: declared vs used

- `DeclaredCtxSet` = все `ctx`, объявленные в `@(...)`.
- `UsedCtxSet` = то, что реально дошло до materialization через `pt.*` (напрямую или через план).

Правило:
- Любой контекст, который материализуется, обязан быть заранее объявлен.
- Иначе: `E_CTX_USED_BUT_NOT_DECLARED`.

## 5. LC-политика

LC — жесткий allowlist-фильтр.

- Если LC не разрешает контекст, функцию или оператор — это сразу ошибка.
- Для контекстов используется единый код: `E_LC_CTX_FORBIDDEN`.
- Отдельный код для "used forbidden" не вводится.

## 6. Attest (делегированные core-фазы)

`attest` — это доказательство/свидетельство делегированных шагов.

Строгое правило (принято):
- Если есть `implement`, и при этом отсутствует хотя бы одна из core-фаз
  (`analyze`, `forecast`, `decide`), `attest` обязателен.
- Это требование не отключается policy override.

`attest` не объявляет контексты и не заменяет `ctx`-декларации.

## 7. Почему это важно для ИИ-агентов

Этот язык специально устроен так, чтобы агент:
- не делал скрытых внешних вызовов в phase body,
- не подменял declared/use/materialize,
- проходил предсказуемую валидацию,
- воспроизводимо исполнялся в разных средах.

## 7.1 Семантика результата evaluate (+1 / 0 / -1)

- `+1` — результат применим к текущему контексту и успешен.
- `-1` — результат применим к текущему контексту, но ведет к деградации/неуспеху.
- `0` — результат неприменим к текущему контексту.
- Неопределенность рассматривается как частный случай неприменимости (`0`), а не как отдельный runtime-класс.

## 8. Минимальный ментальный шаблон

1. В `@(...)` объявить universe зависимостей (`ctx`, `io`, `env`, при необходимости `attest`).
2. В `@(...).collect` проектировать планы (`fn.plan_*`) и выполнять чистую подготовку.
3. В runtime-фазах считать pure-логику поверх уже подготовленных данных и планов.
4. Между фазами в PT hook выполнять materialization (`pt.*`).
5. В `implement` делать доменную мутацию через `imp.*`.
6. В `evaluate` фиксировать результат и публикацию через разрешенные transition effects.

## 9. Статус

`source_human_v1.md` — человекочитаемый источник смысла (source-first).

Норматив для компилятора/парсера:
- `idemo-machine/docs/parser/spec_unified_v1.md`

Содержательный SQL-слой для ветки `@(...).collect -> [pt.*] -> ??(analyze)`:
- [[idemo-docs/materialization/sql/README|SQL PT @->?? Profile]]
- [[idemo-docs/materialization/sql/spec/08_sql_icss_mapping_v1_core|Spec 08: ICSS -> SQL Mapping v1 Core]]

## 10. Синтаксис и примеры (legacy source + v1 адаптация)

Ниже перенесен синтаксический материал из `idemo-machine/docs/parser/legacy/patch1.md` для людей и обсуждений.
Для строгой машинной проверки используйте нормативный файл `idemo-machine/docs/parser/spec_unified_v1.md`.

### 10.1 Grammar patch (как в `idemo-machine/docs/parser/legacy/patch1.md`)

```ebnf
collect_item += collect_decl ;

collect_decl     = "collect", ":", ws, collect_body ;
collect_body     = collect_stmt, { ws, collect_stmt } ;
collect_stmt     = binding, ws, "::", ws, fn_call, ws, [";"] ;
```

Правило:
- `collect_stmt` допускает только `fn.*` (без bang, без `op/pt/imp`).

```ebnf
value += ctx_lit ;
ctx_lit          = "$ctx", "(", ws, ctx_key, ws, ")" ;
```

Пример:
```txt
$ctx(ecom.cart)
```

`FetchPlan` задается как абстрактное значение, производимое функциями:
- `fn.plan_fetch(...)`
- `fn.plan_add(...)`
- `fn.plan_remove(...)`
- `fn.plan_empty()`

### 10.2 Семантические правила (из `idemo-machine/docs/parser/legacy/patch1.md`, кратко)

1. `ctx` объявляются только в `@(...)`.
2. Планирование (`fn.plan_*`) делается только в `@(...).collect`.
3. В фазах разрешено только pure-использование уже объявленных контекстов и уже подготовленных планов.
4. Materialization только через `pt.*` в PT-местах.
5. `pt.fetch(plan)!` валиден только если все `ctx` из плана объявлены.
6. При нарушении: `E_CTX_USED_BUT_NOT_DECLARED`.
7. LC ограничивает и declared, и used/materialized контексты; для нарушений используем `E_LC_CTX_FORBIDDEN`.

### 10.3 Канонические примеры (перенос из `idemo-machine/docs/parser/legacy/patch1.md`)

#### D1) VALID (v1): single @(...), plans in @.collect, materialize in PT

```txt
(
  @(
    ctx: io.user
    ctx: crm.user
    ctx: erp.user
    ctx: ecom.cart

    env: local.mess:
      success: 'Успех'
      error: 'Ошибка'

    collect:
      need_cart :: fn.need_cart(io.user);
      plan1 :: fn.plan_fetch(
        $ctx(crm.user),
        $ctx(erp.user),
        /(need_cart == true : $ctx(ecom.cart) | null)
      );
  )

  => [ pt.fetch(plan1)! ] ??(
    analysis :: fn.analyze()
  ) #analyze

  => ~( ... ) #forecast
  => ^( ... ) #decide
  => >( ... ) #implement
  => _( ... ) #evaluate
)
```

#### D2) VALID (v1): attest + declared contexts in one @(...)

```txt
(
  @(
    ctx: crm.users
    ctx: io.user

    attest:
      context_hash: 'sha256:ctx...'
      decision_ref: 'artifact://decisions/dec-00158.json'
      decision_hash: 'sha256:dec...'
      policy_hash: 'sha256:pol...'

    collect:
      plan :: fn.plan_fetch($ctx(crm.users));
  )

  => [ pt.fetch(plan)! ] ??(
    analysis :: fn.analyze()
  ) #analyze

  => >( ... ) #implement
  => _( ... ) #evaluate
)
```

#### D3) INVALID: undeclared context in plan

```txt
(
  @(
    ctx: io.user
    collect:
      plan :: fn.plan_fetch($ctx(ecom.cart));
  )

  => [ pt.fetch(plan)! ] ??( ... ) #analyze
)
```

```txt
ERROR: E_CTX_USED_BUT_NOT_DECLARED: ecom.cart is referenced in FetchPlan but not declared in @(...)
```

#### D4) INVALID: context declaration inside analyze

```txt
(
  @(ctx: io.user)
  => ??(
    ctx: ecom.cart
  ) #analyze
)
```

```txt
ERROR: E_CTX_DECLARATION_OUTSIDE_COLLECT: ctx declarations are only allowed in @(...)
```

### 10.4 Нормализация (delta из `idemo-machine/docs/parser/legacy/patch1.md`)

- Нормализовать ctx-ссылки к форме `$ctx(<ctx_key>)`.
- Нормализовать fetch-запросы к `pt.fetch(plan)!`; прямой fetch по контексту: `pt.fetch_ctx($ctx(x))!`.
- Проверять, что каждый `ctx_key` в:
  - `$ctx(...)`,
  - plan-объектах,
  - `pt.fetch_ctx(...)`
  присутствует в `DeclaredCtxSet`.

### 10.5 Важная пометка совместимости с v1

Примеры выше переписаны в v1-форму. Исторические версии могли выглядеть иначе.
В текущем каноне v1:
- один `@(...)` блок внутри root intent;
- допустим collect-only intent без фаз;
- `fn.plan_*` проектируются в `@(...).collect`;
- делегированный `implement` требует `attest` без override.
