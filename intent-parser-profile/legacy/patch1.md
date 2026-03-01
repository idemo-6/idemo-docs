ICSS-IntentFlow SPEC v0.2.1 (PATCH)
Focus: Context declaration vs conditional use, and materialization only in PT hooks.

STATUS
- v0.2.1 is a normative patch over v0.2 (does not break the core philosophy).
- Canon: Variant 1 (max purity): no I/O in phases; all external interaction only in PhaseTransition hooks.

KEY IDEAS
1) DECLARED != USED
   - DECLARED contexts define the “universe” of possible dependencies for this Intent.
   - USED contexts are a conditional subset, computed inside phases (purely).
   - MATERIALIZATION (fetch/probe/approval/transport) is ONLY in PT hooks.

2) Three operations (must not be conflated)
   A) declare(ctx): only in @(...).
   B) use(ctx): may be computed in phases as pure values (e.g., building a plan).
   C) materialize(ctx): only in PT hook via pt.* operators.

3) New canonical artifact: FetchPlan
   - FetchPlan is a deterministic, declarative list of ctx keys (and optional query refs).
   - Produced inside phases via fn.* only.
   - Executed only in PT hooks via pt.fetch(plan).

----------------------------------------------------------------------
A) GRAMMAR PATCH (minimal and deterministic)

1) Allow ctx declarations only inside @(...):
   - Already present in v0.2. Keep as-is, but enforce semantics strictly.

2) Add (optional but recommended) collect section inside @(...):
ctx_item += collect_decl ;

collect_decl     = "collect", ":", ws, collect_body ;
collect_body     = collect_stmt, { ws, collect_stmt } ;
collect_stmt     = binding, ws, "::", ws, fn_call, ws, [";"] ;

RULE: collect_stmt allows ONLY fn.* calls (no bang, no op/pt/imp).

3) Add a new literal type for ctx keys (so we can pass them as values safely):
value += ctx_lit ;
ctx_lit          = "$ctx", "(", ws, ctx_key, ws, ")" ;

Example:
  $ctx(ecom.cart)

This avoids ambiguous “string vs ctx” and makes validation simpler.

4) Define FetchPlan as an abstract value (not a new syntax node), produced by functions:
- fn.plan_fetch(...)
- fn.plan_add(...)
- fn.plan_remove(...)
- fn.plan_empty()

No new grammar needed beyond existing fn_call + arg_list.

----------------------------------------------------------------------
B) SEMANTICS PATCH

B1) Context Declaration Rule (hard)
1) ctx declarations (ctx: <ctx_key>) are allowed ONLY inside @(...).
   - Any attempt to declare a new ctx in phase bodies or hooks -> ERROR: E_CTX_DECLARATION_OUTSIDE_COLLECT.

2) Registry/LC validation applies to DECLARED contexts:
   - if Registry.contexts_allowed exists: each declared ctx must be allowed
	 -> E_CTX_NOT_ALLOWED
   - if LC policy allow.contexts exists: each declared ctx must be allowed
	 -> E_LC_CTX_FORBIDDEN

3) Declared contexts may be “unused”; that is valid.
   Declared != used.

B2) Context Use Rule (pure usage)
1) A phase may compute which contexts are needed, but only as pure values.
   - This “use” happens through constructing a FetchPlan or through pure branching.
   - You MUST NOT “activate” or “attach” contexts via I/O inside phases.

2) A ctx is considered USED when it appears in:
   - a FetchPlan (directly or indirectly)
   - a pt.* call argument in a PT hook (direct $ctx(...) or via plan execution)

B3) Materialization Rule (I/O only in PT hooks; Variant 1 canon)
1) Any pt.* operator call is allowed ONLY in pt_hook blocks (between phases)
   and in transition_block effect statements (end-of-phase), as in v0.2.

2) In phase bodies (collect/analyze/forecast/decide/evaluate):
   - Only fn.* calls are allowed.
   - Any op/pt/imp call -> ERROR: E_EFFECT_IN_PHASE_BODY.

3) In implement phase body:
   - fn.* and imp.* allowed (as in v0.2), still no pt.* and no op.*.

B4) FetchPlan validity (new)
Definitions:
- DeclaredCtxSet = set of ctx keys declared in @(...)
- PlanCtxSet(plan) = set of ctx keys referenced by plan (including nested)

Rules:
1) Any function producing or modifying a plan MUST be fn.* and pure.
2) When executing materialization:
   - pt.fetch(plan)! is valid only if PlanCtxSet(plan) ⊆ DeclaredCtxSet
   - Otherwise -> ERROR: E_CTX_USED_BUT_NOT_DECLARED.

3) Direct materialization without a plan is allowed but must still respect declaration:
   - pt.fetch_ctx($ctx(ecom.cart))! allowed only if ecom.cart declared.
   - Otherwise -> E_CTX_USED_BUT_NOT_DECLARED.

B5) LC policy enforcement on USED contexts (important)
Even if a ctx is declared, LC may restrict what can be used/materialized in the current LC phase.

Two enforcement points (both deterministic):
1) At declaration time (strict): LC.allow.contexts limits DeclaredCtxSet.
2) At materialization time (optional but recommended): LC.allow.contexts_used limits USED contexts.
   - If not provided, default = LC.allow.contexts.
   - Violation -> E_LC_CTX_FORBIDDEN (or E_LC_CTX_USED_FORBIDDEN if you want a separate code).

B6) Attest interacts with context but does not replace declaration
- @attest provides evidence for delegated phases.
- It does NOT declare new contexts.
- If decision payload references a ctx not declared, treat as:
  - ERROR: E_CTX_USED_BUT_NOT_DECLARED (because the intent did not declare dependency).

----------------------------------------------------------------------
C) REQUIRED NEW ERROR CODES (v0.2.1)

E_CTX_DECLARATION_OUTSIDE_COLLECT
E_CTX_USED_BUT_NOT_DECLARED

(Everything else remains from v0.2.)

----------------------------------------------------------------------
D) CANONICAL EXAMPLES

D1) Example: declare all potential contexts in @, use conditionally in phases, materialize in PT

(
  @(ctx: io.user)
  @(ctx: crm.user)
  @(ctx: erp.user)
  @(ctx: ecom.cart)
  @(
	env: local.mess:
	  success: 'Успех'
	  error: 'Ошибка'
  )

  => ?(
	need_cart :: fn.need_cart(io.user)
	plan1 :: fn.plan_fetch(
	  $ctx(crm.user),
	  $ctx(erp.user),
	  /(need_cart == true : $ctx(ecom.cart) | null)
	)
  ) #collect

  => [ pt.fetch(plan1)! ] ??(
	analysis :: fn.analyze()
  ) #analyze

  => ~( ... ) #forecast
  => ^( ... ) #decide
  => >( ... ) #implement
  => _( ... ) #evaluate
)

Notes:
- ecom.cart is DECLARED in @.
- It is USED conditionally inside plan1 (pure).
- The actual fetch happens only in PT hook.

D2) Example: use decision/attest but still must declare contexts

(
  @(ctx: crm.users)
  @(ctx: io.user)
  @(
	attest:
	  context_hash: 'sha256:ctx...'
	  decision_ref: 'artifact://decisions/dec-00158.json'
	  decision_hash: 'sha256:dec...'
	  policy_hash: 'sha256:pol...'
  )

  => ?(
	plan :: fn.plan_fetch($ctx(crm.users))
  ) #collect

  => [ pt.fetch(plan)! ] ??(
	analysis :: fn.analyze()
  ) #analyze

  => >( ... ) #implement
  => _( ... ) #evaluate
)

Rule illustrated:
- Even if decision_ref will cause an imp.* mutation, any external ctx dependency must be declared.

D3) INVALID: using an undeclared context in a plan

(
  @(ctx: io.user)
  => ?(
	plan :: fn.plan_fetch($ctx(ecom.cart))
  ) #collect
  => [ pt.fetch(plan)! ] ??( ... ) #analyze
)

ERROR: E_CTX_USED_BUT_NOT_DECLARED: ecom.cart is referenced in FetchPlan but not declared in @(...)

D4) INVALID: attempting to declare ctx inside analyze

(
  @(ctx: io.user)
  => ??(
	ctx: ecom.cart
  ) #analyze
)

ERROR: E_CTX_DECLARATION_OUTSIDE_COLLECT: ctx declarations are only allowed in @(...)

----------------------------------------------------------------------
E) NORMALIZATION RULES (delta)

- Normalize any ctx references used as values into $ctx(<ctx_key>) form.
- Normalize fetch requests to pt.fetch(plan)!; direct ctx fetch uses pt.fetch_ctx($ctx(x))!.
- Enforce that every ctx_key appearing in:
  - $ctx(...)
  - plan objects
  - pt.fetch_ctx(...)
  must exist in DeclaredCtxSet.

END PATCH v0.2.1