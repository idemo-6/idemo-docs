# One-page Playbook: Intent + SQL (без SQL и программирования)

## Для кого

Для пользователей, которые формулируют задачу смыслом, а не SQL-кодом.

## Что написать одной фразой

Шаблон:
`Найди/измени <сущность> по <условию>, покажи <поля>, отсортируй по <поле>, ограничь <N>.`

Пример:
`Покажи заказы клиента Retail за текущий месяц: id, сумма, статус, последние 100.`

## 5 шагов

1. Определи сущность (`from(table)`):
- пример: `from(mark_orders)`.

2. Выбери поля (`select(...)`):
- пример: `select(id, status, sum)`.

3. Добавь фильтр (`filter(...)`):
- пример: `filter(eq(account_id:name, 'Retail'))`.

4. Добавь объединение при необходимости (`merge(...)`):
- `merge(by:path)` для связей между таблицами;
- `merge(rows)` / `merge(unique_rows)` для склейки наборов.

5. Ограничь результат (`order(...)`, `limit(...)`):
- пример: `order(id desc)` + `limit(100)`.

## Готовые рецепты

1. Поиск одной записи:
- `find_one(users, select(id, email, name), where(eq(id, :id)))`

2. Список с фильтром:
- `find_many(mark_orders, select(id, status, sum), where(and(eq(status, 'paid'), gte(sum, 1000))), order(id desc), limit(100))`

3. Отчет с группировкой:
- `from(mark_orders)`
- `select(account_id:name as account_name)`
- `sum(sum as order_sum)`
- `group(account_id:name as account_name)`
- `post_filter(gt(sum, 0))`

4. Обновление по условию:
- `update(users, set(name=:name, updated_at=:updated_at), where(eq(id, :id)))`

5. Удаление по условию:
- `delete(users, where(eq(id, :id)))`

6. Upsert:
- `upsert(roles, values(code=:code, title=:title), conflict(code), update(title=:title))`

## Безопасность

1. Почему `update/delete` без `where` запрещены в `safe_mode`:
- это защищает от массового изменения/удаления всех строк.

2. Что означает `+1/-1/0`:
- `+1`: применимо и успешно;
- `-1`: применимо, но неуспешно/деградация;
- `0`: только неприменимость (`ApplicabilityFailure`), не SQL no-op.

3. SQL no-op:
- помечается как `fror_zero_class=no_cost_transition`, а не как `Result=0`.

## Частые ошибки и как исправить

1. `E_PATH_FK_NOT_FOUND`
- путь связи не найден; проверь `by:path` и FK в схеме.

2. `E_PATH_COLUMN_NOT_FOUND`
- поле не существует в таблице/пути.

3. `E_PATH_AMBIGUOUS`
- неоднозначный путь/алиас; уточни путь явно.

4. `E_PATH_DEPTH_LIMIT`
- слишком длинная цепочка связей; упрости путь.

5. `E_RESULT_DOMAIN_VIOLATION`
- используется `0`, но политика `result_domain` не включает `0`.

## Как это связано с ICSS

- Понятная форма (`filter`, `merge`) автоматически переводится в каноническую форму.
- Символическая форма (`/(cond)`, `*`) доступна как advanced-слой.
- Материализация SQL/DB выполняется в `pt` между `@(...).collect` и `??(analyze)`.

## Куда смотреть дальше

1. [[idemo-docs/materialization/sql/spec/05_simplified_dsl_spec|Spec 05: Simplified DSL]]
2. [[idemo-docs/materialization/sql/spec/08_sql_icss_mapping_v1_core|Spec 08: ICSS -> SQL Mapping v1 Core]]
3. [[idemo-docs/materialization/sql/README|SQL PT @->?? Profile]]
