# IDEMO Context Schema Draft v0.1

Superseded by:
- `idemo-docs/runtime/IDEMO_Context_Schema_Draft_v0_2.md` (preferred for implementation).

## Назначение

Минимальная схема контекста для runtime-профиля IDEMO:
- разделить `Intent` и детали интеграции;
- включить policy-слой допуска;
- связать контекст с `experienceMemory` через policy/refs, без встраивания raw history.

Статус: draft profile.

## 1. Context model (минимум)

Контекст содержит:
1. `identity`: идентификатор и имя контекста.
2. `integration`: connection metadata (без секретов в открытом виде).
3. `policy`: что разрешено выполнять в этом контексте.
4. `operation_experience_policy`: правила использования прошлого опыта.
5. `experience_query_ref`: ссылка на фильтр/запрос к `experienceMemory`.

## 2. JSON draft

```json
{
  "contexts": [
    {
      "id": "9f0870fc-8b0e-48f5-8dfe-30ad7f8fce14",
      "name": "crm",
      "title": "CRM",
      "integration": {
        "type": "api",
        "base_url": "https://test.ru",
        "endpoints_map_ref": "registry://crm/v1/endpoints",
        "auth": {
          "method": "bearer",
          "secret_ref": "vault://prod/crm/token"
        }
      },
      "policy": {
        "contexts_used": ["crm.users", "crm.orders"],
        "operators": ["pt.crm_fetch", "pt.crm_write"],
        "functions": ["fn.plan_select", "fn.validate_shape"],
        "approvals_required": ["sec_review"]
      },
      "operation_experience_policy": {
        "allowed_result_classes": ["+1"],
        "max_age": "P90D",
        "min_confidence": 0.8,
        "require_attested": true
      },
      "experience_query_ref": "expq://crm/default-v1"
    },
    {
      "id": "2f58dcb6-0212-4bb3-b6fd-6c5b5b52c4c9",
      "name": "erp",
      "title": "ERP",
      "integration": {
        "type": "api",
        "base_url": "https://erp.test.ru",
        "endpoints_map_ref": "registry://erp/v2/endpoints",
        "auth": {
          "method": "oauth2",
          "secret_ref": "vault://prod/erp/client-secret"
        }
      },
      "policy": {
        "contexts_used": ["erp.staff", "erp.finance"],
        "operators": ["pt.erp_fetch"],
        "functions": ["fn.plan_join", "fn.score_match"],
        "approvals_required": ["fin_review"]
      },
      "operation_experience_policy": {
        "allowed_result_classes": ["+1"],
        "max_age": "P180D",
        "min_confidence": 0.75,
        "require_attested": false
      },
      "experience_query_ref": "expq://erp/default-v1"
    }
  ]
}
```

## 3. YAML draft

```yaml
contexts:
  - id: 9f0870fc-8b0e-48f5-8dfe-30ad7f8fce14
    name: crm
    title: CRM
    integration:
      type: api
      base_url: https://test.ru
      endpoints_map_ref: registry://crm/v1/endpoints
      auth:
        method: bearer
        secret_ref: vault://prod/crm/token
    policy:
      contexts_used: [crm.users, crm.orders]
      operators: [pt.crm_fetch, pt.crm_write]
      functions: [fn.plan_select, fn.validate_shape]
      approvals_required: [sec_review]
    operation_experience_policy:
      allowed_result_classes: [+1]
      max_age: P90D
      min_confidence: 0.8
      require_attested: true
    experience_query_ref: expq://crm/default-v1

  - id: 2f58dcb6-0212-4bb3-b6fd-6c5b5b52c4c9
    name: erp
    title: ERP
    integration:
      type: api
      base_url: https://erp.test.ru
      endpoints_map_ref: registry://erp/v2/endpoints
      auth:
        method: oauth2
        secret_ref: vault://prod/erp/client-secret
    policy:
      contexts_used: [erp.staff, erp.finance]
      operators: [pt.erp_fetch]
      functions: [fn.plan_join, fn.score_match]
      approvals_required: [fin_review]
    operation_experience_policy:
      allowed_result_classes: [+1]
      max_age: P180D
      min_confidence: 0.75
      require_attested: false
    experience_query_ref: expq://erp/default-v1
```

## 4. Invariants

1. Intent не содержит endpoint/base_url/token literals.
2. Секреты задаются только через `secret_ref`.
3. Контекст не хранит raw operation history; история в `experienceMemory`.
4. Решения используют опыт через `operation_experience_policy` и traceable refs.

## 5. v0.2 split model

Для исполнения flow рекомендуется разделять:

1. `context_catalog` (статическая часть)
- описание контекстов, интеграций, capability/profile-параметров;
- редко меняется;
- проходит отдельный governance review.

2. `runtime_context_binding` (операционный слой запуска)
- какие контексты и policy-оверрайды активны для конкретного flow/run;
- версия policy/registry на момент запуска;
- trace refs и execution mode.

## 6. v0.2 JSON draft (recommended)

```json
{
  "context_catalog": {
    "version": "ctx-catalog.v0.2",
    "contexts": [
      {
        "id": "9f0870fc-8b0e-48f5-8dfe-30ad7f8fce14",
        "name": "crm",
        "title": "CRM",
        "integration": {
          "type": "api",
          "base_url": "https://test.ru",
          "endpoints_map_ref": "registry://crm/v1/endpoints",
          "auth": {
            "method": "bearer",
            "secret_ref": "vault://prod/crm/token"
          }
        },
        "default_policy": {
          "contexts_used": ["crm.users", "crm.orders"],
          "operators": ["pt.crm_fetch", "pt.crm_write"],
          "functions": ["fn.plan_select", "fn.validate_shape"],
          "approvals_required": ["sec_review"]
        },
        "default_operation_experience_policy": {
          "allowed_result_classes": ["+1"],
          "max_age": "P90D",
          "min_confidence": 0.8,
          "require_attested": true
        },
        "default_experience_query_ref": "expq://crm/default-v1"
      },
      {
        "id": "2f58dcb6-0212-4bb3-b6fd-6c5b5b52c4c9",
        "name": "erp",
        "title": "ERP",
        "integration": {
          "type": "api",
          "base_url": "https://erp.test.ru",
          "endpoints_map_ref": "registry://erp/v2/endpoints",
          "auth": {
            "method": "oauth2",
            "secret_ref": "vault://prod/erp/client-secret"
          }
        },
        "default_policy": {
          "contexts_used": ["erp.staff", "erp.finance"],
          "operators": ["pt.erp_fetch"],
          "functions": ["fn.plan_join", "fn.score_match"],
          "approvals_required": ["fin_review"]
        },
        "default_operation_experience_policy": {
          "allowed_result_classes": ["+1"],
          "max_age": "P180D",
          "min_confidence": 0.75,
          "require_attested": false
        },
        "default_experience_query_ref": "expq://erp/default-v1"
      }
    ]
  },
  "runtime_context_binding": {
    "flow_id": "flow-2026-03-04-001",
    "active_contexts": [
      {
        "context_name": "crm",
        "policy_override_ref": "policy://crm/run/strict-v2",
        "operation_experience_policy_override": {
          "allowed_result_classes": ["+1"],
          "max_age": "P30D",
          "min_confidence": 0.85,
          "require_attested": true
        },
        "experience_query_ref": "expq://crm/run-2026-03-04"
      },
      {
        "context_name": "erp",
        "policy_override_ref": null,
        "operation_experience_policy_override": null,
        "experience_query_ref": "expq://erp/default-v1"
      }
    ],
    "execution_mode": "prod",
    "policy_version_ref": "policy-bundle://2026-03-04.1"
  }
}
```

## 7. v0.2 YAML draft (recommended)

```yaml
context_catalog:
  version: ctx-catalog.v0.2
  contexts:
    - id: 9f0870fc-8b0e-48f5-8dfe-30ad7f8fce14
      name: crm
      title: CRM
      integration:
        type: api
        base_url: https://test.ru
        endpoints_map_ref: registry://crm/v1/endpoints
        auth:
          method: bearer
          secret_ref: vault://prod/crm/token
      default_policy:
        contexts_used: [crm.users, crm.orders]
        operators: [pt.crm_fetch, pt.crm_write]
        functions: [fn.plan_select, fn.validate_shape]
        approvals_required: [sec_review]
      default_operation_experience_policy:
        allowed_result_classes: [+1]
        max_age: P90D
        min_confidence: 0.8
        require_attested: true
      default_experience_query_ref: expq://crm/default-v1

    - id: 2f58dcb6-0212-4bb3-b6fd-6c5b5b52c4c9
      name: erp
      title: ERP
      integration:
        type: api
        base_url: https://erp.test.ru
        endpoints_map_ref: registry://erp/v2/endpoints
        auth:
          method: oauth2
          secret_ref: vault://prod/erp/client-secret
      default_policy:
        contexts_used: [erp.staff, erp.finance]
        operators: [pt.erp_fetch]
        functions: [fn.plan_join, fn.score_match]
        approvals_required: [fin_review]
      default_operation_experience_policy:
        allowed_result_classes: [+1]
        max_age: P180D
        min_confidence: 0.75
        require_attested: false
      default_experience_query_ref: expq://erp/default-v1

runtime_context_binding:
  flow_id: flow-2026-03-04-001
  active_contexts:
    - context_name: crm
      policy_override_ref: policy://crm/run/strict-v2
      operation_experience_policy_override:
        allowed_result_classes: [+1]
        max_age: P30D
        min_confidence: 0.85
        require_attested: true
      experience_query_ref: expq://crm/run-2026-03-04

    - context_name: erp
      policy_override_ref: null
      operation_experience_policy_override: null
      experience_query_ref: expq://erp/default-v1

  execution_mode: prod
  policy_version_ref: policy-bundle://2026-03-04.1
```

## 8. v0.2 runtime checks

1. `runtime_context_binding.active_contexts[*].context_name` должен существовать в `context_catalog`.
2. Любой override должен быть traceable (`*_ref` обязателен).
3. `execution_mode=prod` требует валидного approval-policy bundle.
4. В runtime trace фиксируются:
- `flow_id`
- `active_contexts`
- `policy_version_ref`
- `experience_query_ref` (per active context)
