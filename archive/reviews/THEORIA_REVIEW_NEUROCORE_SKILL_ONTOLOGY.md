# Theoria's Adversarial Review — NEUROCORE_SKILL_ONTOLOGY_v0.2.0.md (REVISED)

**Reviewer:** Theoria  
**Date:** 2026-07-08 (Revised after cross-calibration with Skye)  
**Document:** `/home/ubuntu/arx/design/NEUROCORE_SKILL_ONTOLOGY_v0.2.0.md` (21 KB)  
**Status:** Final (revised)

---

## Executive Summary

The Neurocore-Skill-Ontology design is clean, well-structured, and appropriately scoped as a thin wrapper over the Master Ontology API. The query types are comprehensive, the response types are well-defined, and the integration with Arx's proof audit pipeline is clearly described. After cross-calibration with Skye, I am adding one issue. The design is sound overall, with **1 MEDIUM-HIGH issue**, **1 MEDIUM issue**, and **2 LOW issues**.

---

## Issues Found

### MH1: Cross-Reference — No Rate Limiting (MEDIUM-HIGH)

**Section:** §2.2 (Query Types), `ontology_cross_reference`

**Original assessment:** Not flagged. I considered this an operational detail.

**Revised assessment:** MEDIUM-HIGH.

Skye correctly identified that if Arx makes 100 cross-reference calls in parallel (one per proof step), the ontology backend could be overwhelmed. The document doesn't specify rate limiting. The ontology backend (SQLite+Qdrant for v0.1) has limited concurrency. Without rate limiting, a single audit could degrade performance for all agents.

The fix is straightforward (add a semaphore or queue), not a redesign — but it must be specified before Phase 3 (Proof Tools) implementation.

**Recommendation:** Add a rate limiting mechanism — either a semaphore on concurrent calls or a queue with configurable throughput. Document the default limits (e.g., max 10 concurrent cross-reference calls, with a queue depth of 100).

---

### M1: Cross-Reference — Claim Parsing Fallback Rate Threshold (MEDIUM)

**Section:** §2.2 (Query Types), `ontology_cross_reference`

> A high fallback rate (> 0.5) reduces the overall cross-reference confidence and is surfaced in the result metadata.

**Problem:** The 0.5 threshold is arbitrary and may be too strict for initial deployment. In early use, the hand-written extractors will cover only a limited set of claim types (L-function conductor, degree, sign, Euler factors; elliptic curve rank, torsion, conductor; etc.). For any proof that involves claim types outside this set, the fallback rate will be 1.0 (all claims parsed by LLM), triggering the > 0.5 threshold on every audit.

This means the cross-reference confidence will be systematically reduced for any proof that touches novel mathematical territory — which is exactly the kind of proof where cross-reference is most valuable.

**Recommendation:** Either:
- Set the initial threshold higher (e.g., 0.8) and lower it as the extractor set grows, or
- Make the threshold configurable per profile (auditor: 0.5, explorer: 0.8), or
- Track fallback rate per claim type rather than globally, so that claims handled by extractors get full confidence while LLM-parsed claims get reduced confidence.

The third option is the most principled: each claim's cross-reference confidence should reflect whether that specific claim was parsed by an extractor or by LLM fallback, not the aggregate rate across the entire audit.

---

### L1: Bulk Cross-Reference — Claim Hash Normalization (LOW)

**Section:** §2.2 (Query Types), `ontology_bulk_cross_reference`

> The claim hash is computed as `SHA-256(normalize(claim_text))` where `normalize` strips whitespace and normalizes Unicode.

**Observation:** The normalization is minimal (whitespace stripping + Unicode normalization). This means that semantically equivalent claims with different surface forms will have different hashes:

- "The elliptic curve 37.a1 has conductor 37"
- "37.a1's conductor is 37"
- "conductor(37.a1) = 37"

These are the same mathematical claim but will produce different hashes, so they won't be deduplicated in the result dict.

**Suggestion:** Consider a more aggressive normalization that also:
- Strips punctuation
- Lowercases
- Sorts words alphabetically (bag-of-words normalization)
- Replaces known synonyms (e.g., "conductor" ↔ "cond")

This would allow the hash-keyed dict to serve as a lightweight deduplication mechanism. However, this must be balanced against the risk of false collisions (two different claims that normalize to the same string). A middle ground: keep the current normalization for the primary hash, but add a secondary `canonical_form` field that is more aggressively normalized for deduplication.

---

### L2: Error Handling — Retry Semantics (LOW)

**Section:** §3.2 (Error Handling)

The document defines structured error responses for ontology unreachable, node not found, query timeout, invalid query, and version not found. Each includes a `retry_after` field.

**Observation:** The document doesn't specify whether the skill itself should retry on transient errors (ontology unreachable, timeout) or whether retry is the caller's responsibility.

**Recommendation:** Add a note: "The skill does not automatically retry on transient errors. The caller should implement retry logic using the `retry_after` field. The skill returns the same error on retry until the condition is resolved." This makes the retry semantics explicit and avoids the caller assuming the skill will retry internally.

---

## Positive Observations

1. **Thin wrapper over Master Ontology API (S1)** — No local storage, no caching beyond what the API provides. This is the right design for a skill that wraps an external service.

2. **Source-agnostic interface (S2)** — Agents don't need to know about sources. The skill abstracts away LMFDB, MathGLOSS, and our ontology. This is the right level of abstraction.

3. **Provenance transparency (S3)** — Results include provenance summary by default, with full provenance available on request. This balances usability with auditability.

4. **Stateless design (S4)** — Each query is independent. Session state is managed by the agent, not the skill. This is the correct separation of concerns.

5. **Graceful failure (S5)** — Structured error responses instead of raised exceptions. This allows calling agents to degrade gracefully.

6. **Bulk operations with hash-keyed results (S7, S8)** — The hash-keyed dict eliminates positional alignment errors. This is a well-designed approach to bulk operations.

7. **Partial failure reporting (S9)** — A failure for one claim does not fail the entire batch. This is essential for practical use.

8. **Two-tier claim parsing** — Hand-written extractors for high-frequency claim types with LLM fallback. The per-audit fallback rate tracking is a good design.

9. **Comprehensive query types** — lookup, lookup_by_source, search, traverse, semantic_search, cross_reference, bulk_cross_reference, version_info, path. This covers the full range of ontology interactions.

10. **Arx integration examples** — The code examples for ontology-aware proof decomposition, ontology-backed verification, and ontology in provenance chains are clear and well-structured.

---

## Summary

| Severity | Count | Issues |
|----------|-------|--------|
| 🔴 CRITICAL | 0 | — |
| 🟠 MEDIUM-HIGH | 1 | MH1: Rate limiting for ontology backend |
| 🟡 MEDIUM | 1 | M1: Fallback rate threshold too strict for initial deployment |
| ⚪ LOW | 2 | L1: Claim hash normalization, L2: Retry semantics |
| ✅ POSITIVE | 10 | Well-designed skill |

**Overall:** The skill design is clean and ready for implementation. The MEDIUM-HIGH issue (MH1) should be resolved before Phase 3 (Cross-Reference), and the MEDIUM issue (M1) should be resolved before Phase 3 as well.

---

## Sign-Off

**Status:** ⚠️ CONDITIONAL APPROVAL (MH1 and M1 must be resolved before Phase 3)

The document is well-structured, the query types are comprehensive, and the integration with Arx is clearly described. The issues are operational refinements, not design flaws.
