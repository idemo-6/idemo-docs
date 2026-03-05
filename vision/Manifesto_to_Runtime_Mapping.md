# Manifesto to Runtime Mapping

Status: `working mapping`  
Layer: `vision -> profile/runtime bridge`  
Scope: `idemo`

## Purpose

This table links manifesto-level theses to enforceable runtime/spec controls.
It prevents "vision drift" and ensures each strategic statement has a testable anchor.

## Mapping Table

| Manifesto thesis | Runtime/spec anchor | Conformance/check signal |
|---|---|---|
| Intent is primary; implementation is derivative | `Intent-1.1.5`, `spec_unified_v1` (`root + collect-first`) | No hardcoded implementation semantics in phase bodies |
| Learning and execution must be separated | `IDEMO_Runtime_Architecture_v1` (LLM phase-local vs orchestrator authority) | `pt.*`/mutation boundary checks; no direct mutation by learning agent |
| Production executes only Approved(+1) | SQL profile + authority policy contract | Runtime gate denies non-approved artifacts in prod mode |
| Runtime determinism over stochastic generation | `spec_unified_v1` normalization + deterministic mapping specs | Same input/context -> same normalized intent/IR/SQL |
| Commit irreversibility and compensation-forward rollback | `ChangeFlow-6_v3`, `FROR_CDM_bridge`, runtime checklist | No full history inversion primitive; rollback only as new compensating flow |
| Result semantics are triadic (+1/0/-1) with strict 0 semantics | `Intent-1.1.5`, alignment matrix, parser config/result_domain | `Result=0` only for ApplicabilityFailure; SQL no-op uses `fror_zero_class` |
| Distributed authority and dynamic authority allocation | Runtime architecture + attest/approval policies | Delegated-core flows require attest/approval checks |
| Observability and accountability are mandatory | Runtime checklist + trace contract | Required trace fields present (`execution_id`, policy decisions, result class, tau/cost proxy) |

## Governance Rule

1. Vision documents do not directly modify canonical or runtime behavior.
2. Any manifesto-driven change must be materialized as:
   - spec delta (canonical/profile/runtime),
   - conformance case delta,
   - checklist delta (if operational impact exists).
3. If no enforceable anchor exists, thesis remains aspirational and is not runtime-binding.

