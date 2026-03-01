# LC-Intent SPEC v0.1.1 (MINIMAL, FORMAL)

GOAL
Describe an LC system as:
- declared context/IO universe
- ordered LC phases with per-phase allowlists and constraints
- explicit LC phase transitions as autonomous processes with metrics (tau, resource, OP, result, absorbing)

STATUS / NORMATIVE BOUNDARY
- Status: profile (domain policy layer), not canonical replacement.
- Canonical LC source: `[[fcdm-core/theory/cdm/Specifications/LifeCycle-6_v2]]`
- Canonical Intent source: `[[fcdm-core/theory/cdm/Specifications/Intent-1.1.5]]`
- Canonical CtxL source: `[[fcdm-core/theory/cdm/Specifications/CtxL/CtxL-Canonical]]`
- This document specifies implementation policy over canonical constraints.
- Companion template for authoring profiles: `[[idemo-docs/icss/LC-Intent Profile Template]]`

CANONICAL PHASE MAPPING
- `+`  (birth)   -> `F1`
- `+=` (develop) -> `F2`
- `&&` (climax)  -> `F3`
- `-=` (degrade) -> `F4`
- `+-` (turn)    -> `F5`
- `-`  (end)     -> `F6`
- Storage/validation should use canonical `F1..F6`; symbols are presentation aliases.

----------------------------------------------------------------------
A) CANONICAL FORM

(
  $lc_name = "CRM.LC"

  @(
    ctx: crm
    ctx: crm.users
    ctx: crm.orders
    io: kafka
    io: http
  )

  => +(
    phase: birth
    policy: @(
      allow.ctx:
        - crm.users
      allow.cf:
        - CF.UserOnboarding
      allow.ops:
        - op.users_get
        - op.users_create
      constraints:
        ds_max: 0.2
    )
    pt_lc: @(
      # LC-PhaseTransition policy for birth -> develop
      to: develop
      conditional: true
      metrics:
        tau:
          max: 30d
        resource:
          budget: 1000
        op:
          name: "crm_stability"
          threshold: 0.65
      on_fail:
        absorbing: true
        state: "birth_blocked"
      result:
        transition_result: bool
        transition_code: [+1,0,-1]
        zero_only_for: ApplicabilityFailure
    )
  ) #birth

  => +=(
    phase: develop
    policy: @(
      allow.ctx:
        - crm.users
        - crm.orders
      allow.cf:
        - CF.UserUpdate
        - CF.OrderCreate
      allow.ops:
        - op.users_get
        - op.users_set
        - op.orders_create
      constraints:
        ds_max: 0.4
    )
    pt_lc: @(
      # LC-PhaseTransition policy for develop -> climax
      to: climax
      conditional: true
      metrics:
        tau:
          max: 90d
        resource:
          budget: 5000
        op:
          name: "crm_adoption"
          threshold: 0.75
      on_fail:
        absorbing: false
        state: "develop_retry"
      result:
        transition_result: bool
        transition_code: [+1,0,-1]
        zero_only_for: ApplicabilityFailure
    )
  ) #develop

  => &&(
    phase: climax
    policy: @(
      allow.ctx:
        - crm.users
        - crm.orders
      allow.cf:
        - CF.UserUpdate
        - CF.OrderCreate
        - CF.OrderUpdate
      allow.ops:
        - op.users_get
        - op.users_set
        - op.orders_get
        - op.orders_set
      constraints:
        ds_max: 0.6
    )
    pt_lc: @(
      to: degrade
      conditional: true
      metrics:
        tau:
          max: 180d
        resource:
          budget: 10000
        op:
          name: "error_pressure"
          threshold: 0.35
      on_fail:
        absorbing: false
        state: "climax_stays"
      result:
        transition_result: bool
        transition_code: [+1,0,-1]
        zero_only_for: ApplicabilityFailure
    )
  ) #climax

  => -=(
    phase: degrade
    policy: @(
      allow.ctx:
        - crm.users
        - crm.orders
      allow.cf:
        - CF.MigrationPrep
        - CF.DataCleanup
      allow.ops:
        - op.users_get
        - op.orders_get
        - op.archive
      constraints:
        ds_max: 0.85
    )
    pt_lc: @(
      to: turn
      conditional: true
      metrics:
        tau:
          max: 60d
        resource:
          budget: 8000
        op:
          name: "refactor_readiness"
          threshold: 0.55
      on_fail:
        absorbing: true
        state: "degrade_locked"
      result:
        transition_result: bool
        transition_code: [+1,0,-1]
        zero_only_for: ApplicabilityFailure
    )
  ) #degrade

  => +-(
    phase: turn
    policy: @(
      allow.ctx:
        - crm.users
        - crm.orders
      allow.cf:
        - CF.Cutover
        - CF.Migration
      allow.ops:
        - op.export
        - op.import
        - op.freeze_writes
      constraints:
        ds_max: 0.95
    )
    pt_lc: @(
      to: end
      conditional: false
      metrics:
        tau:
          max: 14d
        resource:
          budget: 20000
        op:
          name: "cutover_complete"
          threshold: 0.99
      on_fail:
        absorbing: true
        state: "turn_failed"
      result:
        transition_result: bool
        transition_code: [+1,0,-1]
        zero_only_for: ApplicabilityFailure
    )
  ) #turn

  => -(
    phase: end
    policy: @(
      allow.ctx:
        - crm
      allow.cf: []
      allow.ops:
        - op.shutdown
      constraints:
        ds_max: 1.0
    )
    end: @(
      one_of:
        - death: dissipate
        - transform: new "CRM.LC.vNext"
    )
  ) #end
)

----------------------------------------------------------------------
B) VALIDATION INVARIANTS

Scope:
- Invariants in this section are profile-level validation rules and must not contradict canonical CDM constraints.

1) Declared context closure
- `DeclaredCtxSet` is defined only by root `@(...)` block.
- For every phase `p`: `allow.ctx(p) subseteq DeclaredCtxSet`.
- Violation: reject profile as inconsistent (undeclared context leak).

2) Operator registry closure
- For every phase `p`: `allow.ops(p) subseteq OperatorRegistry`.
- Every operator in `allow.ops(p)` must have executable contract in the target environment.
- Violation: reject profile as inconsistent (unknown or non-executable operator).

3) CF profile registry closure
- For every phase `p`: `allow.cf(p) subseteq CFRegistry`.
- Each referenced CF profile must declare compatible phase contract and authority constraints.
- Violation: reject profile as inconsistent (unknown or incompatible CF profile).

4) Transition gate rule
- Phase transition `p -> p_next` is allowed only if:
  - phase constraints for `p` are satisfied;
  - transition metrics gate is satisfied (`tau/resource/op` thresholds).
- If gate fails:
  - apply `on_fail` policy;
  - set transition outcome according to policy and canonical result semantics.

5) Result semantics rule
- `transition_code in {+1,0,-1}`.
- `transition_code=0` is allowed only for `ApplicabilityFailure`.
- No-op/no-cost transition in operational profiles must be represented analytically (for example, via dedicated analytical marker), not by overloading `0`.

6) End-state exclusivity
- In `end.one_of`, exactly one branch is selected:
  - `death: dissipate`, or
  - `transform: new <Tag>`.
- Simultaneous activation of both branches is invalid.

7) Monotonic policy compatibility
- Per-phase policy must not grant capabilities outside parent/system global policy.
- Formally (profile-level): `allow.ctx/ops/cf(phase) subseteq GlobalAllowSet`.
- Violation: reject profile as policy-escalation attempt.
