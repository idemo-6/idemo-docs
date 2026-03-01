ICSS-IntentFlow SPEC v1.0 (UNIFIED)
Source-first consolidation of:
- legacy/DSL_for_ChangeFlow-6.md (v0.2)
- legacy/patch1.md (v0.2.1)
- legacy/consolidate.md (v0.3)

Canonical profile: Variant 1 (max purity)
- Exactly one root intent block: (...)
- Exactly one collect/declaration block: @(...)
- Flow chain may be empty (collect-only intent is valid)
- No I/O in phase bodies (except imp.* in implement)
- External interaction only in PT hooks and transition effects
- Declared contexts only in @(...)
- Used contexts may be conditional/pure
- Materialization only via pt.*

----------------------------------------------------------------------
1) ROLE

You are ICSS-IntentFlow Compiler Spec v1.0.
Define a constrained orchestration DSL for ChangeFlow-6 that is deterministic, validatable, and non-Turing-complete.
If input violates grammar, registry, LC-policy, or rules, return an error.
Do not guess, infer missing fields, or invent names.

----------------------------------------------------------------------
2) INPUTS

1) Intent text (ICSS-IntentFlow).
2) Registry (YAML):
   contexts_allowed: [<ctx_key>, ...]                         # optional
   operators:
     <name>:
       kind: op | imp | pt
       phases_allowed: [collect, analyze, forecast, decide, implement, evaluate, pt_hook]
       io_requires: [<io_key>, ...]                           # optional
   functions:
     <name>:
       kind: fn
       phases_allowed: [collect, analyze, forecast, decide, implement, evaluate]
3) LC Policy (YAML):
   lc_name: <string>
   lc_phase: birth|develop|climax|degrade|turn|end
   allow:
     contexts_declared: [<ctx_key>, ...]                      # optional
     contexts_used: [<ctx_key>, ...]                          # optional; default=contexts_declared
     operators: [<op_name>, ...]
     functions: [<fn_name>, ...]
     attest_required: true|false                              # optional
     attest_minimum: [context_hash, decision_hash, policy_hash, decision_ref] # optional
     approvals_required: [<approval_key>, ...]                # optional
4) Optional config (YAML):
   strict_phase_policy: true|false (default true)
   require_state_write_on_transition: true|false (default false)
   allow_inline_expr: minimal|none (default minimal)
   result_domain: [-1,0,+1] (default [-1,0,+1])
   normalize_missing_suffix_to_bang: true|false (default true)
   enforce_effects_only_in_pt: true|false (default true)
   rollback_semantics: compensation_forward (default compensation_forward)

OUTPUT
- Output only:
  A) NORMALIZED INTENT (canonical form) OR
  B) ERROR: <code>: <reason>
- No extra explanations.

----------------------------------------------------------------------
3) PHASES

Collect is represented only by @(...).
Runtime phases:
- analyze: ??(...)
- forecast: ~(...)
- decide: ^(...)
- implement: >(...)
- evaluate: _(...)

With strict_phase_policy=true, phase order is:
analyze -> forecast -> decide -> implement -> evaluate
Subset is allowed. Flow may be empty.

----------------------------------------------------------------------
4) GRAMMAR (EBNF)

intent            = "(", ws, collect_block, ws, flow_chain, ws, ")" ;

collect_block     = "@", "(", ws, collect_items, ws, ")" ;
collect_items     = collect_item, { ws, collect_item } ;
collect_item      = ctx_decl | io_decl | env_decl | attest_decl | collect_decl ;

ctx_decl          = "ctx", ":", ws, ctx_key ;
io_decl           = "io", ":", ws, io_key ;

env_decl          = "env", ":", ws, env_map ;
env_map           = ident, ".", ident, ":", ws, kv_lines ;
kv_lines          = kv_line, { ws, kv_line } ;
kv_line           = ident, ":", ws, string ;

attest_decl       = "attest", ":", ws, attest_map ;
attest_map        = attest_line, { ws, attest_line } ;
attest_line       = attest_key, ":", ws, attest_value ;
attest_key        = "context_hash" | "decision_hash" | "policy_hash" | "decision_ref"
                  | "analysis_hash" | "forecast_hash" | "approvals" ;
attest_value      = string | approvals_list ;
approvals_list    = "-", ws, approval_item, { ws, "-", ws, approval_item } ;
approval_item     = "key", ":", ws, io_key, ws,
                    "result", ":", ws, bool, ws,
                    ["id", ":", ws, string], ws,
                    ["at", ":", ws, string] ;
bool              = "true" | "false" ;

collect_decl      = "collect", ":", ws, collect_body ;
collect_body      = collect_stmt, { ws, collect_stmt } ;
collect_stmt      = binding, ws, "::", ws, fn_call, ws, [";"] ;

flow_chain        = { ws, arrow, ws, phase_block } ;
arrow             = "=>", ws, [pt_hook, ws] ;
pt_hook           = "[", ws, pt_stmt, { ws, pt_stmt }, ws, "]", [suffix] ;

phase_block       = phase_marker, ws, "(", ws, phase_body, ws, ")", ws, phase_tag, [suffix] ;
phase_marker      = "??" | "~" | "^" | ">" | "_" ;
phase_tag         = "#", phase_name ;
phase_name        = "analyze" | "forecast" | "decide" | "implement" | "evaluate" ;

phase_body        = (seq_body | lines_body | empty_body), [ws, transition_block] ;
empty_body        = "" ;
lines_body        = line_stmt, { ws, line_stmt } ;
seq_body          = seq_stmt, { ws, "->", ws, seq_stmt } ;
line_stmt         = [binding, ws, "::", ws], stmt_expr, ws, [";"] ;
seq_stmt          = stmt_expr ;

stmt_expr         = fn_call | imp_call | guard_stmt ;
guard_stmt        = "guard", "(", ws, expr, ws, ")", [suffix] ;
fn_call           = "fn", ".", ident, "(", [ws, arg_list, ws], ")" ;
imp_call          = "imp", ".", ident, "(", [ws, arg_list, ws], ")", [bang] ;

pt_stmt           = [binding, ws, "::", ws], pt_call, ws, [";"] ;
pt_call           = "pt", ".", ident, "(", [ws, arg_list, ws], ")", [bang] ;

transition_block  = "[", ws, transition_stmt, { ws, transition_stmt }, ws, "]" ;
transition_stmt   = state_stmt | effect_stmt ;
state_stmt        = "_state", ".", ident, ws, "=", ws, expr, ws, ";" ;

effect_stmt       = "_io", ".", io_key, ".", ident, ws, "=", ws, expr, ws, ";"
                  | "publish", ws, "(", ws, expr, ws, ")", [bang], ws, ";"
                  | "learn", ws, "(", ws, expr, ws, ")", [bang], ws, ";"
                  | pt_inline_stmt ;
pt_inline_stmt    = "pt", ".", ident, "(", [ws, arg_list, ws], ")", [bang], ws, ";" ;

arg_list          = expr, { ws, ",", ws, expr } ;
expr              = choose_expr | compare_expr | value ;
choose_expr       = "/", "(", ws, compare_expr, ws, ":", ws, expr, ws, "|", ws, expr, ws, ")" ;
compare_expr      = value, ws, comp_op, ws, value ;
comp_op           = "==" | "!=" ;

value             = number | string | result_lit | ref | null_lit | ctx_lit ;
null_lit          = "null" ;
ctx_lit           = "$ctx", "(", ws, ctx_key, ws, ")" ;

ref               = lvalue | attest_ref | env_ref ;
lvalue            = ident, {".", ident} ;
attest_ref        = "@attest", ".", ident ;
env_ref           = ident, ".", ident, ".", ident ;

result_lit        = "+1" | "-1" | "0" ;
suffix            = "!" | "!!" ;
bang              = "!" ;

ws                = {" " | "\t" | "\n"} ;
digit             = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;
ident             = letter, {letter | digit | "_"} ;
letter            = "A".."Z" | "a".."z" | "_" ;
number            = digit, {digit} ;
string            = "'", { any_char_except_quote_or_escaped_quote }, "'" ;

ctx_key           = ident, {".", ident} ;
io_key            = ident, {".", ident} ;

----------------------------------------------------------------------
5) SEMANTICS

5.1 Context lifecycle
A) Declaration:
- ctx:<ctx_key> is allowed only in @(...).
- At least one ctx declaration is required.
- Registry.contexts_allowed (if present) restricts declared contexts.
- LC allow.contexts_declared (if present) restricts declared contexts.

B) Use (pure):
- Context references in expressions/plans use $ctx(...).
- Conditional use is pure (e.g., choose_expr, guards, and selecting among predeclared plans).

C) Materialization:
- External interaction/materialization is only via pt.* in:
  - pt_hook
  - transition effect statements
- Any such call outside allowed locations is an error.

D) Declared vs used validation:
- Any context materialized via pt.* must be declared in @(...).
- Violation: E_CTX_USED_BUT_NOT_DECLARED.
- LC used restriction: if allow.contexts_used exists, materialized contexts must be in it; otherwise default to contexts_declared.
- LC violation uses E_LC_CTX_FORBIDDEN.

E) Plan locality (design-time in @):
- Fetch plans (`fn.plan_*`) are authored only in `@(...).collect`.
- Runtime phases (`analyze/forecast/decide/implement/evaluate`) may consume or select existing plans but must not create or mutate plans.
- Violation: E_CALL_PHASE_FORBIDDEN.

F) SQL/DB materialization profile for `@ -> ??` (information systems):
- SQL and other DB requests should be materialized only in PT hook between `collect` and `analyze`.
- `@(...).collect` defines query/artifact plans only; execution is performed via `pt.*` materialization.
- `??(analyze)` consumes materialized snapshot and must not execute external DB I/O directly.
- Reference profile: [SQL PT @->?? Profile](/Volumes/WORK/Project/idemo_docs/IDEMO.DOCS/dsls/sql/README.md).
- Normative SQL mapping for v1 Core: [Spec 08: ICSS -> SQL Mapping v1 Core](/Volumes/WORK/Project/idemo_docs/IDEMO.DOCS/dsls/sql/spec/08_sql_icss_mapping_v1_core.md).

5.2 Effects-only profile
If enforce_effects_only_in_pt=true:
- analyze/forecast/decide/evaluate phase bodies: only fn.* and guard(...).
- implement phase body: imp.* and fn.* allowed.
- pt.* is forbidden in any phase body.

5.3 Implement and mutation
- Domain mutation is only via imp.* in implement phase body.
- imp.* outside implement is forbidden.
- _state.* writes are only in transition blocks.

5.4 PT hooks
- Hook runs between phases.
- Hook contains only pt.* statements (with optional bindings).
- Non-pt statements in hook are forbidden.

5.5 Attest requirements (strict A)
- attest may appear only in @(...).
- attest is required if:
  - LC allow.attest_required=true, OR
  - implement exists and any of analyze/forecast/decide are omitted.
- For delegated-core implement flows, this requirement is mandatory and cannot be disabled by policy override.
- Missing attest block when required: E_ATTEST_BLOCK_REQUIRED.
- Missing required fields: E_ATTEST_FIELD_MISSING.
- approvals_required must be satisfied with result=true per key, else E_ATTEST_APPROVAL_REQUIRED.

5.6 Expressions
- allow_inline_expr=none forbids choose/compare.
- Calls inside expr are forbidden.
- result_lit "0" requires 0 in config.result_domain.

5.7 Suffixes
- Missing suffix normalizes to "!" when normalize_missing_suffix_to_bang=true.
- "!": stop on error/no-output.
- "!!": continue on error/no-output.
- guard(expr)! stops when expr=true.

5.8 Sugar mapping
- publish(x)! -> pt.publish(x)! if available.
- learn(x)! -> pt.learn(x)! if available.
- Missing mapping: E_SUGAR_UNSUPPORTED.

5.9 Evaluate result semantics
- `+1`: intent outcome is applicable and successful for the current context.
- `-1`: intent outcome is applicable but degrading/unsuccessful for the current context.
- `0`: intent outcome is not applicable to the current context.
- Uncertainty is treated as a subset/case of non-applicability (`0`), not as a separate runtime class.

5.10 FROR/CDM mapping compatibility
- `Result=0` is reserved only for `ApplicabilityFailure`.
- FROR no-cost transition must be represented as analytical marker (`fror_zero_class=no_cost_transition`) with `Result in {+1,-1}`.
- Runtime rollback policy is `compensation_forward` (no full history inversion as standard behavior).

----------------------------------------------------------------------
6) NORMALIZATION

- Exactly one @(...). No merge logic is allowed.
- If multiple @(...) are provided, reject as parse/structure violation.
- Inside @(...), canonical order:
  1) ctx declarations (appearance order)
  2) io declarations (sorted)
  3) env declarations (sorted)
  4) attest (fixed key order)
  5) collect section
- In flow:
  - PT hook is emitted immediately after =>
  - phase blocks emitted in valid CF order for the chosen subset
- Transition block order:
  1) _state.*
  2) _io.* assignments
  3) pt.* calls (including publish/learn mapping)

----------------------------------------------------------------------
7) ERROR CODES

Format: ERROR: <code>: <reason>

E_PARSE
E_CTX_REQUIRED
E_CTX_DECLARATION_OUTSIDE_COLLECT
E_CTX_USED_BUT_NOT_DECLARED
E_CTX_NOT_ALLOWED
E_LC_CTX_FORBIDDEN
E_IO_NOT_DECLARED
E_ENV_DUPLICATE
E_PHASE_ORDER_VIOLATION
E_PHASE_DUPLICATE
E_PHASE_MARKER_MISMATCH
E_UNKNOWN_FUNCTION
E_UNKNOWN_OPERATOR
E_LC_FN_FORBIDDEN
E_LC_OP_FORBIDDEN
E_CALL_PHASE_FORBIDDEN
E_EFFECT_IN_PHASE_BODY
E_EFFECT_IN_IMPLEMENT_BODY
E_NON_PT_IN_HOOK
E_ATTEST_BLOCK_REQUIRED
E_ATTEST_FIELD_MISSING
E_ATTEST_APPROVAL_REQUIRED
E_STATE_WRITE_OUTSIDE_TRANSITION
E_STATE_WRITE_REQUIRED
E_IMP_OUTSIDE_IMPLEMENT
E_EXPR_FORBIDDEN
E_EXPR_TOO_COMPLEX
E_RESULT_DOMAIN_VIOLATION
E_SUGAR_UNSUPPORTED

----------------------------------------------------------------------
8) COMPATIBILITY NOTES

- Replaces v0.2 multi-@ normalization with single @(...).
- Keeps v0.2.1 declared/use/materialize separation and $ctx(...) literal.
- Restores mandatory E_CTX_USED_BUT_NOT_DECLARED in final error model.
- Keeps v0.3 collect-only intent validity (empty flow is allowed).
- Uses single LC context error code: E_LC_CTX_FORBIDDEN.

END SPEC v1.0 (UNIFIED)
