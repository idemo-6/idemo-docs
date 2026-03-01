# Intent Authoring Guide

Пользовательская документация по формированию Intent для ICSS-IntentFlow.

## Для кого

- авторы intent (не разработчики parser/runtime),
- доменные аналитики,
- операторы, формирующие задачи для оркестрации.

## Что важно

1. `@(...)` — объявление контекстов и подготовка плана.
2. Runtime-фазы (`??`, `~`, `^`, `>`, `_`) не должны содержать скрытый I/O.
3. Materialization контекста делается только через `pt.*`.
4. `implement` — единственная фаза доменной мутации.
5. `Result in {+1,0,-1}`, где `0` = неприменимость в текущем контексте.

## Основной документ

- [[idemo-docs/intent-authoring/source_human_v1|ICSS-IntentFlow — Human Source v1]]

## Смежные профили

- [[idemo-docs/materialization/sql/README|SQL PT @->?? Profile]]

## Канон CDM

- [[fcdm-core/theory/cdm/Specifications/Intent-1.1.5|Intent]]
- [[fcdm-core/theory/cdm/Specifications/ChangeFlow-6_v3|ChangeFlow-6]]
- [[fcdm-core/theory/cdm/Specifications/Context/Context-Canonical|Context Canonical]]
