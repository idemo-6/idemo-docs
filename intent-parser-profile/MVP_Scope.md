# IntentFlow MVP Scope (v1)

## Goal

Собрать минимально рабочий контур от Intent до исполняемого dry-run:
- parse -> validate -> normalize
- compile to IR
- dry-run execution with trace

Основа: `spec_unified_v1.md`.

## Principles

- Один root `(...)` и один `@(...)`.
- Планирование (`fn.plan_*`) только в `@(...).collect`.
- Внешние эффекты только через PT.
- `result=0` трактуется как неприменимость к текущему контексту.

## Work Packages

### W1: Compiler Core

1. Parser
- Input: Intent text
- Output: AST

2. Validator
- Grammar/phase/semantic checks
- Registry + LC enforcement
- Error model `ERROR: <code>: <reason>`

3. Normalizer
- Canonical formatting
- Deterministic ordering

4. IR Compiler
- AST -> `NormalizedIntentIR` (`runtime/schema/intent_ir_v1.json`)

5. Conformance Tests (initial 8)
- 2 valid cases
- 6 invalid cases with fixed expected error codes

### W2: Execution Core (dry-run)

1. State machine
- CF progression
- Suffix semantics `!` / `!!`

2. Phase runners
- Pure phases: `fn.*`, `guard(...)`
- Implement phase: `imp.*` + `fn.*`

3. PT runner
- `pt.*` hooks and transition effects

4. Trace
- step-level trace and terminal evaluate result `+1/0/-1`

## Proposed Runtime Layout

- `runtime/schema/intent_ir_v1.json`
- `runtime/compiler/parser.*`
- `runtime/compiler/validator.*`
- `runtime/compiler/normalizer.*`
- `runtime/compiler/to_ir.*`
- `runtime/engine/state_machine.*`
- `runtime/engine/phase_runner.*`
- `runtime/engine/pt_runner.*`
- `runtime/trace/trace_model.*`
- `runtime/tests/conformance/cases/*`

## Acceptance Criteria (MVP)

1. Valid intent returns normalized form and IR.
2. Invalid undeclared context in materialized plan -> `E_CTX_USED_BUT_NOT_DECLARED`.
3. Plan creation outside `@(...).collect` -> `E_CALL_PHASE_FORBIDDEN`.
4. `pt.*` inside phase body -> `E_EFFECT_IN_PHASE_BODY`.
5. Delegated-core implement without attest -> `E_ATTEST_BLOCK_REQUIRED`.
6. LC-forbidden context usage -> `E_LC_CTX_FORBIDDEN`.
7. `result_lit "0"` is valid when `config.result_domain` includes `0`.
8. `result_lit "0"` is rejected with `E_RESULT_DOMAIN_VIOLATION` when `config.result_domain` excludes `0`.

## Out of Scope (MVP)

- Production integrations with real infra.
- Dynamic late-declare contexts.
- Multi-policy conflict resolution.
- UI/editor tooling.

## Next Step After MVP

Подключение реальных adapters:
- ContextResolver
- OperatorRegistry
- Approval/Authority Gate
- Code generation agent as controlled capability
