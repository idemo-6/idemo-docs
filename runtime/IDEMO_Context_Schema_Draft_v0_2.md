# IDEMO Context Schema Draft v0.2

## Назначение

Рекомендуемая схема контекста для исполнения flow с разделением:
- `context_catalog` (статический реестр контекстов);
- `runtime_context_binding` (операционная привязка контекстов на конкретный запуск).

## 1. Model split

1. `context_catalog`
- identity/integration/default policy/default experience-policy;
- управляется как каталог с версиями.

2. `runtime_context_binding`
- active contexts для конкретного `flow_id`;
- override refs и execution mode;
- policy bundle ref на момент запуска.

## 2. JSON draft

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

## 3. Invariants

1. `runtime_context_binding.active_contexts[*].context_name` существует в `context_catalog`.
2. Intent не содержит endpoint/base_url/token literals.
3. Секреты задаются только через `secret_ref`.
4. Raw operation history хранится в `experienceMemory`; в контексте только policy/refs.
5. `execution_mode=prod` требует валидный approval policy bundle.

## 4. Runtime trace minimum

1. `flow_id`
2. `active_contexts`
3. `policy_version_ref`
4. `experience_query_ref` по каждому активному контексту

## 5. Conformance reference

Draft suite:
- `idemo-machine/tests/conformance/runtime_context_binding/README.md`
- cases:
  - `01_valid_binding`
  - `02_invalid_unknown_context_name`
  - `03_invalid_prod_missing_policy_version_ref`
  - `04_invalid_prod_missing_experience_query_ref`
  - `05_invalid_secret_literal_in_catalog`

## 6. Validation contract (rule -> error code)

| Rule | Error code |
|---|---|
| `active_contexts[*].context_name` must exist in `context_catalog.contexts[].name` | `E_RUNTIME_CTX_UNKNOWN` |
| `execution_mode=prod` requires `policy_version_ref` | `E_POLICY_VERSION_REQUIRED_FOR_PROD` |
| `execution_mode=prod` requires `experience_query_ref` per active context | `E_EXPERIENCE_QUERY_REF_REQUIRED_FOR_PROD` |
| Inline secrets/tokens in catalog are forbidden (use `secret_ref`) | `E_SECRET_LITERAL_FORBIDDEN` |
| All checks passed | `OK` |
