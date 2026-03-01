# Минимальная схема аттестаций для implement-only CF

Цель: локальная система исполняет **только CF₅ (implement)** (и почти всегда CF₆ evaluate), но **контракт CF₁–CF₆ сохраняется** через аттестации (доказательства/артефакты), что остальные роли были выполнены где-то ещё.

  

Ниже — минимально достаточная схема, которая остаётся детерминированной, валидируемой и хорошо ложится на Git/MMCF.

---

### **1) Канонический набор аттестаций (минимум)**

  

#### **A.** 

#### **attest.context**

####  **(CF₁: сформирован контекст)**

**Зачем:** implement не должен “догадываться”, что именно меняем и где.

**Форма:** стабильный контекстный объект + хэш.

- ctx_key — например crm.users
    
- scope — конкретизация: crm.users:id=123 или crm.users#123
    
- lc_phase — текущая LC-фаза (или её токен)
    
- constraints_ref — ссылка/хэш на policy (allowed ops, limits)
    
- ctx_hash — хэш канонического JSON/YAML контекста
    

  

**Минимум полей:** ctx_key, scope, ctx_hash

---

#### **B.** 

#### **attest.analysis**

####  **(CF₂: преобразование контекста / анализ)**

**Зачем:** отделить “данные” от “суждений/выводов”.

**Форма:** артефакт анализа, даже если это короткая сводка.

- analysis_ref — ссылка (URI/путь/ID)
    
- analysis_hash
    
- inputs_hash — что анализировали (чтобы не подменили)
    

  

**Минимум:** analysis_hash (и желательно inputs_hash)

---

#### **C.** 

#### **attest.forecast**

####  **(CF₃: прогноз)**

**Зачем:** CF₄ (решение) должно быть привязано к прогнозу, иначе решение неаудируемо.

**Форма:** прогнозный артефакт/модель.

- forecast_ref
    
- forecast_hash
    
- horizon (опционально)
    
- scenario_id (опционально)
    

  

**Минимум:** forecast_hash (или явное forecast_absent=true только если политика разрешает)

---

#### **D.** 

#### **attest.decision**

####  **(CF₄: выбор воздействия)**

**Зачем:** это критический токен: **что именно** разрешено исполнить.

**Форма:** “решение” как декларативная инструкция/patch/plan, подписанная.

- decision_id
    
- decision_payload (канонический patch/command)
    
- decision_hash
    
- policy_hash — на какую LC-политику опирались
    
- approvals — список approvals (если были)
    
- signature — подпись/аттестация источника решения (человек/сервис)
    

  

**Минимум:** decision_hash + decision_payload (или ссылка на него)

---

### **2) Что обязательно для запуска CF₅ (самый жёсткий минимум)**

  

Если вы хотите **реально минимально**, без лишнего:

1. attest.context.ctx_hash
    
2. attest.decision.decision_hash (+ доступ к decision_payload)
    
3. policy_hash (чтобы связать с LC-ограничениями)
    

  

То есть **три якоря**:

- _контекст_ (что/где)
    
- _решение_ (что сделать)
    
- _политика_ (можно ли это делать сейчас)
    

---

### **3) Аттестация CF₆ (evaluate) — локальная и обязательная**

  

Для “implement-only” систем CF₆ лучше считать **обязательным локально**, потому что иначе рвётся опыт.

  

Минимальный output CF₆:

- result ∈ {+1,0,-1}
    
- observations_hash (что наблюдали)
    
- exec_trace_hash (трасса implement и переходов)
    
- ds_delta (если применимо)
    
- experience_edge (flow→context→result)
    

---

### **4) Модель доверия: кто “аттестует”**

  

Минимальная типизация источника:

- issuer_type: human | service | committee
    
- issuer_id
    
- signature (опционально в v0, но крайне желательно)
    
- issued_at
    

  

Если подписи пока нет — хотя бы issuer_id + hash и фиксация в Git/ledger.

---

### **5) Нормализованный формат аттестаций (предлагаю YAML)**

`attest:`
  `context:`
    `ctx_key: "crm.users"`
    `scope: "crm.users:id=123"`
    `lc:`
      `name: "CRM.LC"`
      `phase: "climax"`
      `policy_hash: "sha256:..."`
    `ctx_hash: "sha256:..."`
    `issuer:`
      `type: "service"`
      `id: "mmcf-orchestrator"`
      `issued_at: "2026-02-27T10:12:00Z"`

  `decision:`
    `decision_id: "dec-2026-02-27-00158"`
    `decision_payload_ref: "artifact://decisions/dec-2026-02-27-00158.json"`
    `decision_hash: "sha256:..."`
    `policy_hash: "sha256:..."`
    `approvals:`
      `- channel: "kafka"`
        `topic: "director.approve_request"`
        `approval_id: "apr-..."`
        `result: true`
        `issued_at: "2026-02-27T10:15:10Z"`
    `issuer:`
      `type: "human"`
      `id: "director"`
      `issued_at: "2026-02-27T10:15:10Z"`

  `analysis:`
    `inputs_hash: "sha256:..."`
    `analysis_hash: "sha256:..."`
    `issuer:`
      `type: "service"`
      `id: "analytics-dept"`
      `issued_at: "2026-02-27T10:13:00Z"`

  `forecast:`
    `forecast_hash: "sha256:..."`
    `issuer:`
      `type: "service"`
      `id: "forecast-dept"`
      `issued_at: "2026-02-27T10:14:00Z"`

**Правило:** implement-узел может требовать только context + decision (+policy_hash); остальные — опциональны по LC-политике.

---

### **6) Валидация для implement-only CF (правила)**

  

Минимальный валидатор перед CF₅:

- ctx_hash присутствует
    
- decision_hash присутствует
    
- decision_payload доступен и его хэш совпадает
    
- policy_hash совпадает с текущей LC-политикой (или входит в allowlist)
    
- scope соответствует текущему @ctx(...) и доменному лексикону
    
- если decision требует approval — approval присутствует и result=true
    

  

Ошибки (минимум):

- E_ATTEST_CONTEXT_MISSING
    
- E_ATTEST_DECISION_MISSING
    
- E_ATTEST_POLICY_MISMATCH
    
- E_ATTEST_DECISION_TAMPERED
    
- E_ATTEST_APPROVAL_REQUIRED
    
- E_ATTEST_SCOPE_INVALID
    

---

## **Резюме всех обсуждений (сжато, но полно)**

  

### **A) Трёхслойная архитектура CDM → IDEMO → MMCF**

- **CDM** задаёт онтологию динамики изменений.
    
- **IDEMO** — вычислительная реализация: Intent/Context + Operators runtime + чистые функции.
    
- **MMCF** — методология и управление изменениями: контексты, координаты, опыт, трассировка.
    

  

### **B) Разделение ответственности: смысл vs реальность**

- **CF₁–CF₄ и CF₆** — “смысловые” преобразования (могут быть чистыми функциями).
    
- **CF₅ (implement)** — единственная фаза, где меняется state системы.
    
- **PhaseTransition** — отдельный подпроцесс между фазами со своим τ/Σ/failpoint и result ∈ {true,false}.
    

  

### **C) LifeCycle как родитель, ChangeFlow как дочерняя динамика**

- **LifeCycle** задаёт режимы и ограничения: какие CF/операторы допустимы в каждой фазе.
    
- **ChangeFlow не существует сам по себе**: он запускается как ответ на **Δ** в рамках текущей LC-фазы.
    
- PT — общий класс переходов; CF-PT и LC-PT имеют разную механику, но общий “тип процесса”.
    

  

### **D) Intent DSL как control plane**

- Intent описывает: контекст, поток, разрешённые операторы, переходы и политики.
    
- Исполняемый код — только чистые функции (минимальные, тестируемые), без инфраструктуры.
    
- Инфраструктура/DI/оркестрация не исчезают, а переносятся в Intent + Operators runtime.
    

  

### **E) 6 фаз LC и 6 фаз CF — контракт совместимости**

- “Жёсткость” — в структуре (интерфейс), гибкость — в:
    
    - DomainLexicon
        
    - разрешённых операторах
        
    - переходных механиках (условные/безусловные)
        
    - распределении фаз по подсистемам
        
    - критериях завершения и метриках (Δ/DS/OP)
        
    

  

### **F) Implement-only CF как делегирование фаз**

- Если локально есть только CF₅, остальные роли CF сохраняются через **аттестации**:
    
    - context-token
        
    - decision-token (+policy_hash)
        
    - (опционально) analysis/forecast artifacts
        
    
- CF₆ (evaluate) желательно делать локально всегда — для опыта и DS.



## **1) Встраиваем аттестации в ICSS-IntentFlow (пошагово)**

  

### **Шаг 1.1 — Вводим новый обязательный блок** 

### **@attest(...)**

  

Цель: сделать “implement-only” и вообще любой CF **формально полным** без выполнения всех фаз локально.

  

**Правило:** @attest(...) может присутствовать всегда, но **обязателен**, если:

- intent содержит #implement, и
    
- отсутствуют локальные фазы #analyze/#forecast/#decide (или LC-политика требует аттестации независимо).
    

  

**Минимальный синтаксис (канонический):**

`@(`
  `attest:`
    `context_hash: 'sha256:...'`
    `decision_hash: 'sha256:...'`
    `policy_hash: 'sha256:...'`
    `decision_ref: 'artifact://decisions/dec-...json'`
`)`

Где:

- context_hash = якорь CF₁ (контекст)
    
- decision_hash + decision_ref = якорь CF₄ (решение)
    
- policy_hash = привязка к LC (допустимость)

### **Шаг 1.2 — Добавляем опциональные аттестации** 


analysis_hash
forecast_hash
approvals


Это нужно для аудита/совместимости, но не заставляет усложнять минимальный путь.

`@(`
  `attest:`
    `analysis_hash: 'sha256:...'`
    `forecast_hash: 'sha256:...'`
    `approvals:`
      `- channel: kafka`
        `topic: director.approve_request`
        `approval_id: 'apr-...'`
        `result: true`
`)`

**Правило:** если decision или LC-политика требуют approval — отсутствие approvals = ошибка.

---

### **Шаг 1.3 — Определяем “где живут” аттестации в исполнении**

  

В runtime (IDEMO machine / MMCF) аттестации становятся:

- частью **execution context**
    
- входом для валидатора (до запуска CF₅)
    
- частью trace и опыта (CF₆)
    

  

Технически:

- runtime грузит decision_ref
    
- проверяет decision_hash
    
- проверяет policy_hash с текущей LC-политикой
    
- проверяет scope (из ctx) против ctx_hash (или делает канонизацию и пересчитывает)
    

---

### **Шаг 1.4 — Встраиваем “implement-only” как валидный частный случай**

  

Теперь intent может быть минимальным:

`$(`
`@(ctx: crm.users)`
`@(io: kafka)`
`@(`
  `attest:`
    `context_hash: 'sha256:...'`
    `decision_ref: 'artifact://decisions/dec-00158.json'`
    `decision_hash: 'sha256:...'`
    `policy_hash: 'sha256:...'`
`)`

`=> >(`
  `result :: op.users_set_from_decision(@attest.decision_ref) !`
  `[ _state.result = result; ]`
`) #implement`

`=> _(`
  `[`
    `_kafka.topic = /((result == +1) : 'ok' | 'fail');`
    `publish(_kafka.topic) !;`
    `learn(result) !;`
  `]`
`) #evaluate`
`)`

**Смысл:** CF₁–CF₄ представлены аттестациями, локально выполняем CF₅ и CF₆.

---

### **Шаг 1.5 — Добавляем правила доступа к аттестациям из DSL**

  

Чтобы использовать это в выражениях/операторах, вводим стандартные ссылки:

- @attest.context_hash
    
- @attest.decision_ref
    
- @attest.decision_hash
    
- @attest.policy_hash
    
- @attest.approvals[*]...
    

  

Это просто ещё один namespace, как local.mess.*.

---

### **Шаг 1.6 — Добавляем минимальные ошибки (обязательные)**

  

Нужно расширить error model:

- E_ATTEST_BLOCK_REQUIRED
    
- E_ATTEST_CONTEXT_MISSING
    
- E_ATTEST_DECISION_MISSING
    
- E_ATTEST_POLICY_MISSING
    
- E_ATTEST_DECISION_TAMPERED
    
- E_ATTEST_POLICY_MISMATCH
    
- E_ATTEST_APPROVAL_REQUIRED
    

  

И правило: ошибки аттестаций **останавливают** запуск CF₅.

---

## **2) Нормализация/рендеринг + генерация шаблонов и валидаторов Codex (пошагово)**

  

### **Шаг 2.1 — Определяем каноническую форму (Canonical Intent)**

  

Codex сможет генерировать парсер/валидатор только если есть **одна** каноническая форма.

  

**Правила канонизации:**

1. Все @(...) блоки сортируются и печатаются в порядке:
    
    - @ctx (в порядке появления)
        
    - @env (лексикографически по ключу)
        
    - @io (лексикографически)
        
    - @attest (фиксировано последним)
        
    
2. Внутри @attest поля печатаются в фиксированном порядке:
    
    - context_hash, decision_ref, decision_hash, policy_hash,
        
    - затем analysis_hash, forecast_hash,
        
    - затем approvals (в порядке появления)
        
    
3. Каждый phase block печатается на отдельной “секции” с тегом #phase.
    
4. Transition blocks [...] внутри phase печатаются:
    
    - state writes (_state.*) первыми
        
    - затем I/O effects
        
    - затем learn/publish
        
    
5. Выражения /(cond : a | b) нормализуются:
    
    - скобки обязательны
        
    - == и != только
        
    - литералы всегда в single quotes (кроме +1/-1/0)
        
    

---

### **Шаг 2.2 — Минимальный “шаблон файла” Intent и “шаблон аттестаций”**

  

Чтобы Codex мог создавать папки/файлы и обновлять changelog, нужно стандартизировать пути.

  

Рекомендованный минимум:

- intents/<ctx_key>/<intent_id>.icss
    
- attestations/<ctx_key>/<decision_id>.yaml
    
- artifacts/decisions/<decision_id>.json
    
- logs/<ctx_key>/<cf_run_id>.jsonl (опционально)
    

---

### **Шаг 2.3 — Генерация валидатора: 3 слоя проверки**

  

Codex будет генерировать код проще, если валидатор разделён:

1. **Parse validation** (грамматика)
    
2. **Registry validation** (известные ops/fns/contexts)
    
3. **Policy & attest validation** (LC + attest + approvals)
    

  

Минимальный интерфейс валидатора (псевдо):

- validate_intent(text, registry, lc_policy, attest_store) -> ok|error
    
- normalize_intent(ast) -> canonical_text
    

---

### **Шаг 2.4 — RENDERING CONTRACT для diffs и Git**

  

Чтобы diff был стабильным:

- запрет “свободного” форматирования
    
- ровно один стиль
    
- фиксированные отступы (например 2 пробела)
    
- пустая строка между фазами
    
- ключи env/attest в фиксированном порядке
    

  

Это делает автоматическое обновление changelog/версий предсказуемым.

---

### **Шаг 2.5 — Автогенерация папок/шаблонов Codex: что именно он делает**

  

Codex может:

1. создать директории по шаблону
    
2. создать:
    
    - registry.yaml (operators/functions/contexts)
        
    - lc_policy.yaml (allowed ops by LC phase)
        
    - intents/*/*.icss (пустые шаблоны)
        
    - attestations/*/*.yaml (пустые шаблоны)
        
    - CHANGELOG.md (канонический формат)
        
    
3. добавить pre-commit hook:
    
    - icss fmt (нормализация)
        
    - icss validate (валидация)
        
    - icss changelog (обновление)
        
    

---

### **Шаг 2.6 — Минимальные правила для changelog и CF-index как commit-counter**

  

Так как вы хотите CF-index ~ commit:

- каждый merge/commit, который меняет intent/registry/policy, увеличивает cf_index
    
- cf_index пишется:
    
    - в changelog
        
    - в метаданные intent (опционально)
        
    - в MMCF trace
        
    

  

Канонический trailer в Git commit message:

- CF-CTX: crm.users
    
- CF-IDX: 00158
    
- LC: CRM.LC#climax
    
- INTENT: users_update_00158