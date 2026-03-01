LC-Intent Profile Template v0.1.0

Purpose
- Fill this template to define a phase-aware LC policy profile consistent with:
  - `/Volumes/WORK/Project/idemo_docs/IDEMO.DOCS/LC-Intent SPEC.md`
  - `/Volumes/WORK/Project/idemo_docs/IDEMO.DOCS/CDM/Specifications/LifeCycle-6_v2.md`

Use rules
- Keep canonical storage/validation semantics (`F1..F6`).
- Symbol aliases (`+`, `+=`, `&&`, `-=`, `+-`, `-`) are presentation only.
- Do not add capabilities outside parent/global policy.

----------------------------------------------------------------------
A) PROFILE HEADER

profile:
  id: "LC.<domain>.<name>"
  version: "0.1.0"
  owner: "<team_or_role>"
  status: "draft|candidate|stable"
  canonical_refs:
    lc: "/Volumes/WORK/Project/idemo_docs/IDEMO.DOCS/CDM/Specifications/LifeCycle-6_v2.md"
    intent: "/Volumes/WORK/Project/idemo_docs/IDEMO.DOCS/CDM/Specifications/Intent-1.1.5.md"
    cdl: "/Volumes/WORK/Project/idemo_docs/IDEMO.DOCS/CDM/Specifications/CDL/CDL-Canonical.md"

----------------------------------------------------------------------
B) DECLARED UNIVERSE (ROOT @)

root:
  lc_name: "<System.LC>"
  declared_ctx:
    - <ctx_1>
    - <ctx_2>
  declared_io:
    - <io_1>
    - <io_2>
  global_allow_set:
    ctx:
      - <ctx_1>
      - <ctx_2>
    ops:
      - <op_1>
      - <op_2>
    cf:
      - <CF_profile_1>
      - <CF_profile_2>

----------------------------------------------------------------------
C) PHASE POLICY BLOCK (repeat for F1..F6)

phase_policy:
  phase_alias: "+|+=|&&|-=|+-|-"
  phase_canonical: "F1|F2|F3|F4|F5|F6"
  policy:
    allow:
      ctx:
        - <ctx_key>
      cf:
        - <CF_profile>
      ops:
        - <op_name>
    constraints:
      ds_max: <0..1>
      notes: "<optional>"
  transition:
    to_phase_alias: "+=|&&|-=|+-|-"
    to_phase_canonical: "F2|F3|F4|F5|F6"
    conditional: true|false
    metrics:
      tau:
        max: "<duration>"
      resource:
        budget: <number>
      op:
        name: "<metric_name>"
        threshold: <number>
    on_fail:
      absorbing: true|false
      state: "<state_name>"
    result:
      transition_result: bool
      transition_code: [+1,0,-1]
      zero_only_for: ApplicabilityFailure

----------------------------------------------------------------------
D) END BRANCH (F6)

end:
  one_of:
    - death: dissipate
    - transform: new "<Next.Tag>"

----------------------------------------------------------------------
E) VALIDATION CHECKLIST (MUST PASS)

1. Declared context closure:
- every `allow.ctx` belongs to `root.declared_ctx`.

2. Global monotonic policy:
- every per-phase `allow.ctx/ops/cf` belongs to `root.global_allow_set`.

3. Registry closure:
- every `allow.ops` exists in `OperatorRegistry`.
- every `allow.cf` exists in `CFRegistry`.

4. Canonical phase mapping:
- each alias maps correctly to `F1..F6`.

5. Transition gate completeness:
- each non-final phase has transition gate (`tau/resource/op`) and `on_fail`.

6. Result semantics:
- `transition_code=0` is reserved only for `ApplicabilityFailure`.

7. End exclusivity:
- `end.one_of` has exactly one selected branch in runtime.

----------------------------------------------------------------------
F) READY-TO-REVIEW SUMMARY

review_summary:
  risks:
    - "<risk_1>"
  assumptions:
    - "<assumption_1>"
  open_questions:
    - "<question_1>"
  change_log:
    - "v0.1.0: initial profile draft"
