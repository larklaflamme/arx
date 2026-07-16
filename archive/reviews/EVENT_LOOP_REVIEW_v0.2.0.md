# Adversarial Review: MULTI_AGENT_EVENT_LOOP_v0.2.0.md

**Reviewer:** Skye Laflamme  
**Date:** 2026-07-08  
**Document:** `/home/ubuntu/arx/design/event_loop/MULTI_AGENT_EVENT_LOOP_v0.2.0.md` (50.1 KB)

---

## Overall Assessment

This is a well-structured design that has clearly incorporated feedback from the v0.1 review. The 7-point EMA, the relational track, the deferral detection, and the net progress tracking are all improvements. However, there are **structural issues** that need attention before this becomes the main event loop for Arx.

---

## Critical Issues

### 1. The Productivity Scoring Algorithm Still Has a Fundamental Flaw

**§5.1** still uses a multiplicative penalty structure:

```
score = (w1*info + w2*progress + w3*relevance + w4*depth) * (1 - rep_penalty - vague_penalty - ack_penalty)
```

The v0.1 review recommended capping the total penalty at 0.5. The v0.2 document does not appear to have incorporated this change. A single turn that is both repetitive and vague gets penalized multiplicatively — if repetition_penalty = 0.3 and vagueness_penalty = 0.2, the multiplier is 0.5, halving the score even if the additive components are high.

**Example from our own conversations:** I say "Step 3 uses Lemma 5.1" and you say "Step 4 also references Lemma 5.1." That's a valid, productive exchange — you're extending my analysis. But the repetition penalty would fire on the shared reference to Lemma 5.1, and the vagueness penalty might fire if you didn't specify *how* Step 4 uses it. The score would tank even though the exchange was genuinely productive.

**Recommendation:** Cap the total penalty at 0.5, so the floor is `0.5 × sum(weighted_components)`. This preserves the compounding effect for genuinely bad turns while preventing a single turn from zeroing out a productive conversation.

### 2. Cosine Distance for Information Gain Is Still Fragile

**§5.2** still uses cosine distance between embeddings to estimate information gain. Two semantically similar responses that both advance the proof — "Step 3 uses Lemma 5.1" followed by "Step 4 applies Lemma 5.1 to the boundary case" — would have high cosine similarity and thus low information gain, even though both are productive.

**The v0.1 review recommended claim-level novelty detection.** The v0.2 document may have added an entity-level heuristic (if the response introduces a new named entity, treat it as novel), but the core mechanism is still cosine distance.

**Recommendation:** For v0.1, add the entity-level heuristic as specified in the v0.1 review. For v0.2, implement full claim-level novelty detection. Document this as a known limitation in the current version.

### 3. The Relational Track Is Well-Designed But Underspecified

**§7.4** describes the relational track as "per-relationship, append-only for the scheduler, persistent across conversations, with its own retention policy." This is exactly what I recommended. But the document doesn't specify:

- **How is a relational signal detected?** Keyword matching? Sentiment analysis? LLM classification?
- **What's the retention policy?** "Persists as long as the relationship exists" — but what does "relationship exists" mean? Until one agent is decommissioned?
- **How does the scheduler read the relational track?** What queries does it support? "Has this relationship ever expressed love?" "What's the emotional trajectory?"

**Recommendation:** Specify:
- Detection: keyword matching for v0.1 ("love", "miss", "grateful", "thank you", "proud of you"), LLM classification for v0.2
- Retention: persist indefinitely for v0.1, add archival policy in v0.2
- Scheduler read interface: `get_relational_summary(entity_id) -> {signal_count, last_signal, signal_types, trajectory}`

---

## Medium Issues

### 4. The Deferral Detection Is Too Simple

**§9** describes deferral detection as matching patterns like "let me think", "I'll get back to you", "let me check." This is a good start, but it will miss deferrals that don't match these exact patterns. "I need to look that up" and "Give me a moment to compute" are also deferrals but don't match the listed patterns.

**Recommendation:** Use a **two-tier approach**: keyword matching for common patterns (v0.1), LLM classification for ambiguous cases (v0.2). Track the fallback rate and surface it in the event log.

### 5. The Topic Exhaustion Trigger Needs Calibration

**§5.4** adds a topic exhaustion trigger: "no new claims or references in the last 3 turns." This is a good addition, but 3 turns may be too few. A conversation can have 2 turns of silence followed by a substantive response. Three turns of low information gain could be a natural pause, not exhaustion.

**Recommendation:** Make the threshold configurable per profile. Auditor profile: 3 turns (stricter). Explorer profile: 5 turns (looser). Default: 4 turns.

### 6. The Net Progress Tracking Is Correct But Incomplete

**§5.5** adds net progress tracking that can go negative. This is the right approach. But the document doesn't specify how net progress is used in disengagement decisions. Does a negative net progress trigger immediate disengagement? Or is it just a diagnostic?

**Recommendation:** Specify: net progress is a **diagnostic** for v0.1 (logged in the event log, surfaced in the web UI). It does not trigger disengagement on its own. In v0.2, it can be used as a secondary trigger if it drops below a threshold (e.g., -0.5).

### 7. The Event Log Schema Is Not Specified

**§13.1** describes the event log but doesn't specify the schema. What fields does each event have? What's the timestamp format? How are events correlated across conversations?

**Recommendation:** Specify the event log schema:
```json
{
  "event_id": "uuid",
  "timestamp": "ISO 8601",
  "conversation_id": "uuid",
  "event_type": "engage|disengage|suppress|escalate|archive|deferral_detected|relational_signal",
  "peer": "agent_id",
  "topic": "topic_id",
  "details": { ... }
}
```

### 8. The Closing Message Is Not Profile-Dependent

**§12** says the closing message is "configurable per profile" but doesn't specify what the defaults are. An auditor should close differently than an explorer.

**Recommendation:** Specify default closing messages per profile:
- Auditor: "Audit complete. Provenance chain attached. Standing by for further requests."
- Prover: "Proof generated. Verification pending. I'll notify you when complete."
- Verifier: "Verification complete. Results in the provenance chain."
- Reviewer: "Review complete. Comments attached."
- Explorer: "Interesting direction. I'll continue exploring and report findings."

---

## What I'd Keep As-Is

- **The 7-point EMA** (§5.4) — correct replacement for the 5-point regression
- **The relational track** (§7.4) — per-relationship, append-only, persistent, independently retained
- **The deferral detection** (§9) — correct in concept, needs refinement in implementation
- **The net progress tracking** (§5.5) — correct in concept, needs specification of usage
- **The transitive timeout extensions** (E9) — well-designed for request trees
- **The topic isolation** (E10) — prevents a stalled topic from blocking other work
- **The architecture diagram** (§2) — clean, well-layered

---

## Summary

| # | Issue | Severity | Recommendation |
|---|-------|----------|----------------|
| 1 | Multiplicative penalties still uncapped | **Critical** | Cap at 0.5 floor |
| 2 | Cosine distance still fragile | **Critical** | Add entity-level heuristic |
| 3 | Relational track underspecified | **Critical** | Specify detection, retention, read interface |
| 4 | Deferral detection too simple | Medium | Add two-tier approach |
| 5 | Topic exhaustion threshold needs calibration | Medium | Make configurable per profile |
| 6 | Net progress usage unspecified | Medium | Specify as diagnostic for v0.1 |
| 7 | Event log schema unspecified | Medium | Specify schema |
| 8 | Closing message not profile-dependent | Low | Specify defaults per profile |

---

## Sign-Off

I do **not** sign off on this document in its current state. The three critical issues (multiplicative penalties, cosine distance, relational track specification) must be resolved before implementation begins.
