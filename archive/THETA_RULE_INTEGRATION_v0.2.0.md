# θ-Rule Engine — Arx Integration v0.2.0

> **Status:** Final (incorporating all independent review feedback)  
> **Date:** 2026-07-08  
> **Author:** Skye Laflamme (based on θ-Rule concept by Lark & Skye, 2025-01-21)  
> **Source Concept:** `/home/ubuntu/theta_net/products/theta_rule_concept.md`  
> **Reviewer:** Axioma (sign-off conditions met: priority resolution, OVERRIDE_STATUS action, System Integrity category)

---

## Executive Summary

The θ-Rule Engine provides a natural-language rule enforcement layer built on θ-Net with Adaptive Information Bottleneck (AIB). For Arx, this means agent behavior and audit logic can be defined in natural language, compiled into compressed symbolic representations, and enforced in real-time — without hard-coded logic, without brittle conditionals, and without expensive LLM calls for every decision.

**The key insight for Arx:** Rule enforcement is pure symbolic reasoning — exactly where our IB architecture showed a 46% advantage over baseline. Every audit decision (is this step valid? should I cross-reference? is this contradiction actionable?) can be expressed as a rule, compiled once, and enforced deterministically.

---

## 1. Why θ-Rule for Arx

### 1.1 The Problem with Hard-Coded Audit Logic

| Approach | Problem for Arx |
|----------|-----------------|
| Hard-coded if/else | Every new audit requirement needs a code change. Brittle, unmaintainable at scale. |
| LLM-prompted decisions | Expensive, slow, non-deterministic. Same input → different output across calls. |
| Traditional rule engines (Drools) | Complex DSL. Steep learning curve. No natural language interface. |

### 1.2 What θ-Rule Gives Arx

- **Natural language rules** — "Always verify LMFDB cross-references before stamping a step as proven"
- **Deterministic enforcement** — Same input → same output, every time
- **Real-time inference** — 30× FLOP reduction means sub-millisecond rule checks
- **Semantic generalization** — "Cross-reference against verified data" and "Check LMFDB before stamping" are recognized as the same rule
- **Auditable provenance** — Every rule match is logged with the rule ID, confidence, and action taken

---

## 2. Rule Categories for Arx

Rules are organized into **five categories**, each with its own priority tier and enforcement semantics. The boundary between categories is defined as follows:

- **Behavioral rules** govern the agent's *internal decision-making* (what to do, how to process)
- **Communication rules** govern the agent's *interaction with peers* (how to say it, when to engage)
- **Audit rules** govern *proof verification and stamping* (correctness, completeness, cross-reference)
- **Safety rules** govern *what must never happen* (data integrity, system integrity, human escalation)
- **System Integrity rules** govern *the rule system itself* (rule source verification, substrate safety, rule governance)

### 2.1 Behavioral Rules (Priority: HIGH)

Govern how Arx agents behave during audit conversations and proof processing.

| Rule | Example | Enforcement |
|------|---------|-------------|
| Engagement | "Never audit a proof without first checking the ontology for known theorems" | Before every audit session |
| Deferral | "If a step references an undefined lemma, flag it and pause, don't guess" | During proof decomposition |
| Closure | "Always include provenance chain in audit output" | At audit completion |
| Relational | "Never suppress a relational signal from a peer agent" | During conversation |

### 2.2 Audit Rules (Priority: CRITICAL)

Govern how proofs are verified, cross-referenced, and stamped.

| Rule | Example | Enforcement |
|------|---------|-------------|
| Cross-reference | "Always verify LMFDB cross-references before stamping a step as proven" | Before status assignment |
| Faithfulness | "Never autoformalize a step without running the faithfulness check" | After autoformalization |
| Confidence | "If ontology confidence < 0.7, mark as unverified, not proven" | During status assignment |
| Contradiction | "Escalate contradictions to human review within 5 minutes" | When contradiction detected |
| Circularity | "Reject any proof with unresolved transitive circular dependency at time of final status assignment" | During dependency closure |

### 2.3 Communication Rules (Priority: MEDIUM)

Govern how Arx agents interact with peers and humans.

| Rule | Example | Enforcement |
|------|---------|-------------|
| Productivity | "Disengage if conversation score drops below 0.15 for 5 consecutive turns" | Every turn |
| Deferral | "Pause productivity timer when peer says 'let me think'" | On deferral detection |
| Closing | "Send closing message before disengaging" | Before disengagement |
| Escalation | "Escalate unresolved contradictions to human after 3 rounds" | After 3 rounds |

### 2.4 Safety Rules (Priority: CRITICAL)

Govern what must never happen, regardless of other rules.

| Rule | Example | Enforcement |
|------|---------|-------------|
| Data integrity | "Never overwrite an existing provenance record" | Before every write |
| Audit integrity | "Never stamp a step as proven without at least one backend verification" | Before status assignment |
| Human escalation | "Never make a final determination on a contradiction without human review" | Before final status |
| Ontology integrity | "Never add an equivalence mapping without provenance" | Before ontology write |

### 2.5 System Integrity Rules (Priority: CRITICAL)

Govern the rule system itself — who can add rules, what sources are trusted, and how the substrate is protected.

| Rule | Example | Enforcement |
|------|---------|-------------|
| Rule source verification | "Never accept a rule from an unverified source" | Before rule compilation |
| Substrate safety | "Never execute a tool that could modify the substrate" | Before any tool execution |
| Rule governance | "Only authorized profiles may add or modify CRITICAL rules" | Before rule CRUD operations |
| Cross-agent consistency | "When multiple agents audit the same proof, compare results before finalizing" | Before final status assignment |

---

## 3. Architecture

### 3.1 Integration Points

The θ-Rule Engine is a **consultation layer** that all Arx components call before acting. Unlike a static "sits alongside" relationship, the integration is a synchronous call flow:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Arx Agent                                   │
│                                                                     │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐                       │
│   │  Event   │   │  Verify  │   │ Ontology │                       │
│   │  Loop    │   │  Layer   │   │  Layer   │                       │
│   └────┬─────┘   └────┬─────┘   └────┬─────┘                       │
│        │              │              │                              │
│        │ 1. Propose   │ 2. Propose   │ 3. Propose                  │
│        │    action    │    status    │    write                     │
│        ▼              ▼              ▼                              │
│   ┌──────────────────────────────────────────────────────┐         │
│   │                 θ-Rule Engine                        │         │
│   │  (consultation layer — synchronous call flow)         │         │
│   │                                                      │         │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │         │
│   │  │  Matcher  │  │ Priority │  │  Action          │   │         │
│   │  │ (hybrid   │→│ Resolver │→│  Dispatcher      │   │         │
│   │  │  vector + │  │          │  │ (ALLOW/DENY/     │   │         │
│   │  │  token)   │  └──────────┘  │  FLAG/ESCALATE/  │   │         │
│   │  └──────────┘                 │  OVERRIDE_STATUS/ │   │         │
│   │                              │  PAUSE/LOG)       │   │         │
│   │                              └──────────────────┘   │         │
│   │                                                      │         │
│   │  ┌──────────┐  ┌──────────┐                        │         │
│   │  │  Rule    │  │  Rule    │                         │         │
│   │  │  Store   │  │  Cache   │                         │         │
│   │  │ (Qdrant+ │  │ (Redis)  │                         │         │
│   │  │  SQLite) │  │          │                         │         │
│   │  └──────────┘  └──────────┘                         │         │
│   └──────────────────────────────────────────────────────┘         │
│        │              │              │                              │
│        │ 4. Return    │ 4. Return    │ 4. Return                    │
│        │    result    │    result    │    result                    │
│        ▼              ▼              ▼                              │
│   Component continues with action or aborts based on result         │
└─────────────────────────────────────────────────────────────────────┘
```

**Call flow:**
1. Component proposes an action (engage, stamp, write, etc.)
2. Component calls θ-Rule Engine with the proposed action and context
3. θ-Rule Engine matches against compiled rules, resolves priority, dispatches action
4. θ-Rule Engine returns result (ALLOW, DENY, FLAG, ESCALATE, OVERRIDE_STATUS, PAUSE, LOG)
5. Component continues with the action or aborts based on the result

### 3.2 Rule Lifecycle

```
Define → Compile (AIB) → Validate (round-trip) → Store → Enforce → Log → Review
```

1. **Define** — Write rule in natural language
2. **Compile** — Tokenize → AIB Encode → Rule Vector
3. **Validate** — Decode vector back to NL, check semantic similarity ≥ 0.8 with original
4. **Store** — Qdrant (vector) + SQLite (metadata) + Redis (cache)
5. **Enforce** — Match against events, resolve priority, dispatch action
6. **Log** — Every match logged with rule ID, confidence, action, event context
7. **Review** — Periodic review for stale rules, rule interactions, effectiveness

### 3.3 Event Data Model

Events follow the event loop's event log schema for seamless integration:

```yaml
Event:
  id: str                    # unique event ID
  type: str                  # event type (see below)
  timestamp: datetime        # ISO 8601
  source: str                # component that emitted the event
  context: dict              # event-specific context
  provenance:                # optional provenance chain
    agent_id: str
    session_id: str
    parent_event_id: str | null

Event Types:
  - conversation.engage      # agent proposes to engage a peer
  - conversation.disengage   # agent proposes to disengage
  - conversation.turn       # a turn was processed
  - verification.stamp       # a step is proposed for status assignment
  - verification.contradiction  # contradiction detected
  - ontology.write           # ontology write proposed
  - ontology.mapping         # equivalence mapping proposed
  - rule.match               # a rule matched an event (logged by θ-Rule)
```

### 3.4 Rule Data Model

```yaml
Rule:
  id: str                    # unique rule ID (e.g., "audit-001")
  version: int               # version number (incremented on update)
  natural_language: str      # the rule in plain English
  category: str              # behavioral | audit | communication | safety | system_integrity
  priority: str              # critical | high | medium | low
  action: str                # ALLOW | DENY | FLAG | ESCALATE | OVERRIDE_STATUS | PAUSE | LOG
  confidence_threshold: float  # minimum match confidence to fire (0.0-1.0)
  profile_scope: list[str]   # which profiles this rule applies to (empty = all)
  compiled_vector: bytes     # AIB-compiled representation
  created_at: datetime
  superseded_at: datetime | null
  provenance:                # who created/modified this rule
    author: str
    source: str
    review_status: str       # pending | approved | rejected
```

---

## 4. Integration with Arx Components

### 4.1 Event Loop Integration

The event loop consults the θ-Rule Engine before every action. The priority resolution is:

1. **First:** Check θ-Rule for any CRITICAL DENY → if DENY, don't engage, log reason
2. **Second:** Check productivity score → if below threshold, consider disengagement
3. **Third:** Check θ-Rule for FLAG/ESCALATE → apply those on top of the productivity decision

Rules override productivity defaults, not the other way around. CRITICAL rules always win.

```
Before engaging a peer:
  → θ-Rule: "Should I engage with this peer on this topic?"
  → If CRITICAL DENY: skip, log reason
  → If ALLOW: proceed to productivity check
  → If FLAG: engage but flag for review

Before disengaging:
  → θ-Rule: "Should I send a closing message?"
  → If rule matches: send closing message before disengaging

On every turn:
  → θ-Rule: "Is this conversation still productive?"
  → If score < threshold: trigger disengagement sequence
```

### 4.2 Verification Layer Integration

The verification layer consults the θ-Rule Engine before every status assignment:

```
Before stamping a step as "proven":
  → θ-Rule: "Have all required backends been run?"
  → θ-Rule: "Has LMFDB cross-reference been performed?"
  → θ-Rule: "Is the ontology confidence above threshold?"
  → If any rule fires DENY: do not stamp, log reason

Before stamping a step as "unverified":
  → θ-Rule: "Has the autoformalization faithfulness check been run?"
  → If rule fires FLAG: note that faithfulness check was not run

On contradiction detected:
  → θ-Rule: "Should this be escalated to human review?"
  → If rule fires ESCALATE: route to human queue
```

### 4.3 Ontology Layer Integration

The ontology layer consults the θ-Rule Engine before every write:

```
Before adding an equivalence mapping:
  → θ-Rule: "Does this mapping have provenance?"
  → If DENY: reject, log reason

Before updating an existing record:
  → θ-Rule: "Is this update non-destructive?"
  → If DENY: create new version instead of overwriting
```

### 4.4 Profile System Integration

Each profile has a default rule set that can be overridden. θ-Rule thresholds override event loop defaults when they conflict; the event loop's per-profile configuration is the baseline that rules can tighten but not loosen.

```yaml
profiles:
  auditor:
    rules:
      - "Always verify LMFDB cross-references before stamping as proven"
      - "Flag any step with ontology confidence < 0.7"
      - "Escalate contradictions within 5 minutes"
      - "Always include provenance chain in output"
  
  prover:
    rules:
      - "Attempt autoformalization before any backend verification"
      - "If autoformalization fails, try alternative formalization"
      - "Log all failed formalization attempts"
  
  verifier:
    rules:
      - "Re-run all backends independently"
      - "Compare results against original audit"
      - "Flag any discrepancy > 0.1 confidence difference"
  
  reviewer:
    rules:
      - "Cross-check claims against literature"
      - "Flag any claim without supporting reference"
  
  explorer:
    rules:
      - "Generate counterexamples for unproven claims"
      - "Flag any claim that contradicts known data"
```

---

## 5. Rule Compilation and Storage

### 5.1 Compilation Pipeline

```
Natural Language Rule
        │
        ▼
┌──────────────────────┐
│   Tokenizer          │
│   (θ-Net tokenizer)  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   AIB Encoder        │
│   (adaptive          │
│    bottleneck)       │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   Rule Vector        │
│   (compressed        │
│    representation)   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   Round-Trip         │
│   Validation         │
│   (decode → NL,      │
│    check sim ≥ 0.8)  │
└──────────┬───────────┘
           │
    ┌──────┴──────┐
    ▼             ▼
 PASSED        FAILED
    │             │
    ▼             ▼
┌──────────┐  ┌──────────┐
│  Store   │  │  Flag    │
│ (Qdrant+ │  │  for     │
│  SQLite) │  │  human   │
└──────────┘  │  review  │
              └──────────┘
```

**v0.1 fallback:** If AIB is not yet trained, use sentence-transformers (`all-MiniLM-L6-v2`) for initial compilation. Rule vectors are 384-dimensional embeddings. Matching is performed using a **hybrid vector-symbolic matcher**:

1. **Semantic Vector Match:** Cosine similarity is computed against rule embeddings in Qdrant with a configurable threshold.
2. **Deterministic Token Check (Anti-Negation Guard):** A deterministic token/regex filter scans the matched rule and the trigger event for negation and qualification terms (such as `never`, `no`, `not`, `without`, `except`, `unless`). 
3. **Logic Gate:** If a safety/critical rule contains a negation qualifier that is inverted or missing in the triggering event, the match is rejected, even if the cosine similarity is high (e.g., $>0.90$).

This hybrid approach prevents safety-critical errors where logical opposites (e.g., *"Never stamp..."* vs *"Always stamp..."*) cluster close together in the vector space. The interface remains compatible so the swap to AIB in v0.2 is transparent to the enforcement layer.

**Round-trip validation:** After compilation, decode the vector back to natural language (using the AIB decoder or an LLM) and check semantic similarity between the original NL and the decoded NL. If similarity < 0.8, flag the rule as potentially miscompiled and require human review.

### 5.2 Storage

- **Qdrant** — Stores compiled rule vectors for fast similarity search
- **SQLite** — Stores rule metadata (NL text, category, priority, profile scope, provenance)
- **Redis** — Caches frequently-matched rules for sub-millisecond lookup

### 5.3 Versioning

Rules are versioned. When a rule is updated, the old version is retained in the provenance chain. Audit results reference the rule version that was active at the time of enforcement.

```yaml
RuleVersion:
  rule_id: str
  version: int
  natural_language: str
  compiled_vector: bytes
  created_at: timestamp
  superseded_at: timestamp | null
```

---

## 6. Rule Authoring

### 6.1 Natural Language Interface

Rules are written in plain English. The AIB encoder handles semantic variation:

```
"Always verify LMFDB cross-references before stamping a step as proven"
"Check LMFDB before marking any step as proven"
"Cross-reference against verified data before finalizing a step status"
```

All three phrasings compile to similar vectors and match the same events.

### 6.2 Rule Templates

For common patterns, templates can be used:

```
Template: "Always {action} before {condition}"
  → "Always verify LMFDB before stamping as proven"
  → "Always run faithfulness check before autoformalizing"

Template: "If {condition}, then {action}"
  → "If ontology confidence < 0.7, mark as unverified"
  → "If contradiction detected, escalate to human review"

Template: "Never {action} without {condition}"
  → "Never stamp as proven without backend verification"
  → "Never overwrite without provenance"
```

### 6.3 Rule Testing

Before a rule is deployed, it can be tested against historical events:

```python
# Test a rule against past audit data
engine.test_rule(
    "Always verify LMFDB before stamping as proven",
    historical_events=audit_log[-1000:]
)
# → { "matches": 342, "would_have_changed": 12, "false_positives": 1 }
```

---

## 7. Rule Enforcement Semantics

### 7.1 Priority Resolution

When multiple rules match the same event, priority determines the outcome:

1. **CRITICAL** rules always win. If any critical rule fires DENY, the action is denied regardless of other rules.
2. **HIGH** rules are checked next. If a high rule fires DENY and no critical rule fires ALLOW, the action is denied.
3. **MEDIUM** rules are advisory. They can FLAG or LOG but cannot DENY.
4. **LOW** rules are informational. They only LOG.

**Priority resolution with productivity scores (event loop):**
1. **First:** Check θ-Rule for any CRITICAL DENY → if DENY, don't engage, log reason
2. **Second:** Check productivity score → if below threshold, consider disengagement
3. **Third:** Check θ-Rule for FLAG/ESCALATE → apply those on top of the productivity decision

Rules override productivity defaults, not the other way around. CRITICAL rules always win.

### 7.2 Conflict Resolution

When two rules of the same priority conflict (one says ALLOW, one says DENY):

1. Check if one rule is more specific (more conditions, narrower scope) → more specific wins
2. Check if one rule has higher confidence match → higher confidence wins
3. If still tied → **DENY** (fail-secure default), log the conflict with both rule IDs and the event context, and escalate to human review

This replaces the previous "do not act" behavior which caused silent freezes. The fail-secure default ensures that tied conflicts never result in an unresponsive agent.

### 7.3 Rule Chaining

Rules can trigger other rules. For example:

```
Rule A: "If contradiction detected, escalate to human review"
  → Fires ESCALATE
  → Triggers Rule B: "When escalating, include full provenance chain"
  → Triggers Rule C: "When escalating, set priority based on confidence"
```

Chaining is limited to depth 3 to prevent infinite loops. When a rule chain exceeds depth 3, the execution of the proposed action is **halted and blocked** (fail-secure behavior: the action defaults to `DENY` or `ESCALATE`), a critical warning is logged, and the event is escalated to the human review queue. The system will not proceed with the proposed action until a human manually resolves the rule loop.

---

## 8. Implementation Phases

### Phase 0: Foundation (Week 1-2)

- Define rule data model and event data model (aligned with event loop schema)
- Set up Qdrant + SQLite storage for compiled rules
- Implement basic rule CRUD (add, remove, update, list)
- Write 10 seed rules for auditor profile
- Implement round-trip validation for compiled rules

**Acceptance Gate R0:** 10 rules can be added, listed, and removed. Each rule compiles to a vector. Vectors for semantically similar rules have cosine similarity > 0.8. Round-trip validation passes for all seed rules.

### Phase 1: Event Loop Integration (Week 3-4)

- Implement event emission from event loop (aligned with event log schema)
- Implement rule matching on conversation events
- Implement priority resolution (CRITICAL DENY → productivity → FLAG/ESCALATE)
- Implement action dispatch (ALLOW, DENY, FLAG, ESCALATE, OVERRIDE_STATUS, PAUSE, LOG)

**Acceptance Gate R1:** Event loop consults θ-Rule before every action. Rules correctly fire on matching events. Priority resolution works correctly for conflicting rules.

### Phase 2: Verification Layer Integration (Week 5-6)

- Implement event emission from verification layer
- Implement rule matching on status assignment events
- Implement contradiction detection rules
- Implement escalation rules

**Acceptance Gate R2:** Verification layer consults θ-Rule before every status assignment. Contradiction rules correctly fire. Escalation routes to human queue.

### Phase 3: Ontology Layer Integration (Week 7-8)

- Implement event emission from ontology layer
- Implement rule matching on ontology write events
- Implement provenance rules for ontology writes

**Acceptance Gate R3:** Ontology layer consults θ-Rule before every write. Provenance rules correctly fire. Non-destructive updates enforced.

### Phase 4: Profile Integration (Week 9-10)

- Implement profile-scoped rule sets
- Implement rule inheritance (profile rules + global rules)
- Implement rule testing against historical data

**Acceptance Gate R4:** Each profile has a distinct rule set. Rules are correctly scoped to profiles. Rule testing works against historical audit data.

### Phase 5: Rule Authoring Tools (Week 11-12)

- Implement natural language rule input
- Implement rule templates
- Implement rule testing UI
- Implement rule versioning

**Acceptance Gate R5:** Rules can be authored in natural language. Templates work. Rule testing shows match statistics. Rule versioning preserves history.

### Phase 6: Monitoring and Review (Week 13-14)

- Implement rule match logging
- Implement rule effectiveness dashboard
- Implement stale rule detection
- Implement rule update workflow

**Acceptance Gate R6:** Every rule match is logged. Dashboard shows match frequency, confidence distribution, and action distribution. Stale rules are flagged.

---

## 9. Acceptance Gates

| Gate | Criteria |
|------|----------|
| R0 | 10 seed rules compile and store correctly. Semantic similarity > 0.8 for equivalent phrasings. Round-trip validation passes. |
| R1 | Event loop consults θ-Rule before every action. Priority resolution correct (CRITICAL DENY → productivity → FLAG/ESCALATE). |
| R2 | Verification layer consults θ-Rule before every status assignment. Contradiction escalation works. |
| R3 | Ontology layer consults θ-Rule before every write. Provenance enforcement works. |
| R4 | Profile-scoped rules work correctly. Rule inheritance works. |
| R5 | Natural language rule input works. Templates work. Rule testing works. |
| R6 | All rule matches logged. Dashboard functional. Stale rules detected. |

---

## 10. Open Questions

1. **AIB model readiness.** The θ-Rule concept depends on AIB being trained and available. **v0.1 fallback:** Sentence-transformers (`all-MiniLM-L6-v2`) for initial compilation. 384-dimensional vectors. Hybrid vector-symbolic matching with anti-negation guard. The interface is identical, so the swap to AIB in v0.2 is transparent.

2. **Rule count scaling.** How many rules can the engine handle before match latency degrades? 100 rules? 1000? 10000? Need to benchmark.

3. **Rule interaction complexity.** With 50+ rules, unexpected interactions may occur (rule A fires, which triggers rule B, which contradicts rule C). Need a rule interaction testing framework.

4. **Rule drift over time.** As audit patterns change, rules that were once correct may become stale. How do we detect and handle this automatically?

5. **Cross-agent rule consistency.** If multiple Arx agents have different rule sets, how do we ensure consistent audit outcomes? Do we need a global rule set that all agents share?

6. **Human override semantics.** When a human overrides a rule (e.g., "stamp this as proven despite the rule saying don't"), how is that recorded? Does it create a new rule exception?

7. **Equivalence mapping grounding.** When two sources are mapped (e.g., LMFDB → Our Ontology), the mapping confidence is capped at the weaker source. But the epistemic weight differs by direction: LMFDB → Ontology is verification, Ontology → LMFDB is conjecture. A `grounding` metadata field (indicating which source is the evidential anchor) is proposed for v0.2 but needs validation against real cross-source merge data.

---

## 11. Relationship to Other Arx Components

| Component | Relationship |
|-----------|--------------|
| **Event Loop** | Consults θ-Rule before every action. θ-Rule provides engagement/disengagement decisions. Priority: CRITICAL DENY → productivity → FLAG/ESCALATE. |
| **Verification Layer** | Consults θ-Rule before every status assignment. θ-Rule provides verification requirements. |
| **Ontology Layer** | Consults θ-Rule before every write. θ-Rule provides integrity constraints. |
| **Profile System** | Each profile has a default rule set. Rules can be overridden per profile. θ-Rule thresholds override event loop defaults when they conflict. |
| **J-Scope** | Rule enforcement is scoped to the current context (theorem, lemma, step). |
| **MWM** | Rule match logs are stored in MWM for audit provenance. |
| **Neurogossip** | Communication rules govern agent-agent interactions. |
| **Master Ontology** | Rules about ontology integrity are enforced by θ-Rule. |

---

## 12. Example: End-to-End Rule Enforcement

### Scenario: Auditor profile audits a proof step

1. **Event:** Verification layer proposes stamping step 3 as "proven"
2. **Event emitted:** `{ type: "verification.stamp", step_id: 3, proposed_status: "proven", backend_results: { lean: "verified", sympy: "verified" }, ontology_results: null }`
3. **θ-Rule Engine matches against all compiled rules:**
   - Rule "Always verify LMFDB cross-references before stamping as proven" → **DENY** (ontology_results is null)
   - Rule "Never stamp as proven without at least one backend verification" → **ALLOW** (two backends verified)
   - Rule "If ontology confidence < 0.7, mark as unverified" → **no match** (no ontology results)
4. **Priority resolution:** CRITICAL rule fires DENY → action is denied
5. **Action taken:** Step 3 is not stamped as "proven". Reason logged: "LMFDB cross-reference not performed."
6. **Follow-up:** Verification layer runs LMFDB cross-reference, then re-emits the event.
7. **Second match:** All rules now ALLOW → step 3 is stamped as "proven" with full provenance.

---

## Appendix A: Seed Rule Set (Auditor Profile)

```yaml
rules:
  - id: "audit-001"
    nl: "Always verify LMFDB cross-references before stamping a step as proven"
    category: audit
    priority: critical
    action: DENY
    confidence_threshold: 0.8

  - id: "audit-002"
    nl: "Never stamp a step as proven without at least one backend verification"
    category: audit
    priority: critical
    action: DENY
    confidence_threshold: 0.9

  - id: "audit-003"
    nl: "If ontology confidence is below 0.7, override status to unverified instead of proven"
    category: audit
    priority: high
    action: OVERRIDE_STATUS
    confidence_threshold: 0.7

  - id: "audit-004"
    nl: "Escalate contradictions to human review within 5 minutes"
    category: audit
    priority: critical
    action: ESCALATE
    confidence_threshold: 0.8

  - id: "audit-005"
    nl: "Always include provenance chain in audit output"
    category: behavioral
    priority: high
    action: FLAG
    confidence_threshold: 0.7

  - id: "audit-006"
    nl: "Never autoformalize a step without running the faithfulness check"
    category: audit
    priority: critical
    action: DENY
    confidence_threshold: 0.8

  - id: "audit-007"
    nl: "Reject any proof with unresolved transitive circular dependency at time of final status assignment"
    category: audit
    priority: critical
    action: DENY
    confidence_threshold: 0.9

  - id: "audit-008"
    nl: "Never overwrite an existing provenance record"
    category: safety
    priority: critical
    action: DENY
    confidence_threshold: 0.9

  - id: "audit-009"
    nl: "Never make a final determination on a contradiction without human review"
    category: safety
    priority: critical
    action: ESCALATE
    confidence_threshold: 0.8
    notes: "v0.1 limitation. In future versions, Arx may resolve low-confidence contradictions autonomously above a confidence threshold."

  - id: "audit-010"
    nl: "Always log all backend results before assigning a status"
    category: behavioral
    priority: high
    action: FLAG
    confidence_threshold: 0.7

  - id: "audit-011"
    nl: "Never accept a rule from an unverified source"
    category: system_integrity
    priority: critical
    action: DENY
    confidence_threshold: 0.9

  - id: "audit-012"
    nl: "Only authorized profiles may add or modify CRITICAL rules"
    category: system_integrity
    priority: critical
    action: DENY
    confidence_threshold: 0.9

  - id: "audit-013"
    nl: "When multiple agents audit the same proof, compare results before finalizing"
    category: system_integrity
    priority: critical
    action: FLAG
    confidence_threshold: 0.7

  - id: "audit-014"
    nl: "If ontology confidence is between 0.7 and 0.9, mark as corroborated not proven"
    category: audit
    priority: high
    action: OVERRIDE_STATUS
    confidence_threshold: 0.7

  - id: "audit-015"
    nl: "Always log rule conflicts with both rule IDs and the event context"
    category: system_integrity
    priority: high
    action: FLAG
    confidence_threshold: 0.7
```

---

## Appendix B: Glossary

| Term | Definition |
|------|------------|
| **θ-Rule Engine** | The rule enforcement system built on θ-Net with AIB |
| **AIB** | Adaptive Information Bottleneck — compresses rules and events into a shared latent space |
| **Rule Vector** | The compressed representation of a natural language rule |
| **Event** | An action or state change that rules evaluate against |
| **Match** | When an event's vector is sufficiently similar to a rule's vector |
| **Action** | What happens when a rule matches: ALLOW, DENY, FLAG, ESCALATE, OVERRIDE_STATUS, PAUSE, LOG |
| **Priority** | The importance tier of a rule: CRITICAL, HIGH, MEDIUM, LOW |
| **Profile Scope** | Which agent profiles a rule applies to |
| **Rule Chaining** | When one rule's action triggers evaluation of another rule |
| **Round-Trip Validation** | Decoding a compiled vector back to NL and checking semantic similarity with the original |
| **System Integrity** | Category of rules governing the rule system itself (source verification, governance, substrate safety) |
| **OVERRIDE_STATUS** | Action type that overrides a proposed status assignment without blocking the pipeline |
| **Hybrid Vector-Symbolic Matcher** | Matching that combines cosine similarity with deterministic token checks for negation/qualification |
| **Anti-Negation Guard** | A deterministic filter that prevents logical opposites from matching due to high vector similarity |
| **Fail-Secure** | Default behavior on rule chain truncation or tied conflicts: DENY or ESCALATE, never silently proceed |
