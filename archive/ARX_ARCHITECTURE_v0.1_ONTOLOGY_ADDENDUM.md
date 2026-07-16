# Arx — Ontology Integration: Design Decisions Record v0.2.0

**Status:** Final (design decisions incorporated into main architecture document v0.2.0)  
**Date:** 2026-07-08  
**Purpose:** This document records the key design decisions made during the ontology integration review, serving as a decision log. The ontology layer is now a first-class component of the main architecture document (§3.9).

---

## 1. Key Design Decisions

### D1: Storage — SQLite + Qdrant for v0.2.0, Neo4j deferred

**Decision:** Use SQLite (with recursive CTEs for graph queries) + Qdrant (for vector search) for v0.2.0. Defer Neo4j until we hit a performance ceiling.

**Rationale:**
- SQLite handles graph queries for the expected v0.2.0 scale (hundreds to low thousands of nodes)
- Properly indexed SQLite tables can handle 100,000+ nodes with recursive CTEs
- No additional infrastructure to maintain
- Easy to back up, version, and distribute
- Abstract `OntologyGraph` interface allows Neo4j swap without changing the skill or Arx

**Estimated threshold for Neo4j migration:** ~100,000+ nodes (to be benchmarked during Phase 0.5)

### D2: LMFDB — On-demand + cache, no full mirror

**Decision:** Access LMFDB on-demand via REST API (or MCP server) with local cache. No full mirror of the entire database.

**Rationale:**
- LMFDB contains millions of objects across dozens of categories (3,824,372 elliptic curves over ℚ alone)
- Full mirror would require significant storage and maintenance
- Cache stores only objects actually referenced in proofs
- MCP server available (announced April 2026, confirmed on lmfdb.org)

### D3: LMFDB Cache TTL — Indefinite for static data

**Decision:** Static computational data from LMFDB (conductor, rank, torsion, isogeny class) is cached **indefinitely**. Dynamic data (search results, computed properties) has a 1h TTL.

**Rationale:**
- Mathematical invariants (e.g., the rank of elliptic curve 37.a1) are eternal truths
- Setting a 24-hour TTL on static invariants forces unnecessary API queries during re-audits
- Cache invalidation occurs on new ontology merge version, not on time

### D4: Contradiction Priority — Category-based, not confidence-capped

**Decision:** Contradiction priority (HIGH vs NORMAL) is determined by the **category** of the contradiction (structural, property-level, mapping), not by the source confidence.

**Rationale:**
- Prevents our ontology (confidence 0.6) from being permanently capped below the escalation threshold
- Structural contradictions (e.g., "X is an elliptic curve" when ontology says it's a modular form) are always HIGH priority regardless of source
- Property-level contradictions (e.g., "X has rank 2" when ontology says rank 1) are NORMAL priority

### D5: Ontology Write Access — Sandboxed queue, not direct git

**Decision:** Agents propose ontology additions via a **sandboxed database table** or dedicated queue. A separate system tool or human operator handles git commits.

**Rationale:**
- Prevents compromised agents from pushing malicious code to the repository
- Human review ensures quality control for confidence-0.6 data
- The sandboxed queue is isolated from the git workflow

### D6: Ontology Versioning — Content-addressable hash

**Decision:** Ontology version is a content-addressable hash of the full ontology state (nodes + edges + equivalence mappings + previous version).

**Rationale:**
- Trivial to check whether any ontology record used in an audit has changed
- No sequential version numbers to manage
- Deterministic: same ontology state always produces the same version

### D7: Qdrant Versioning — ontology_version in payload

**Decision:** Qdrant payloads store the `ontology_version` and include it as a filter in vector queries.

**Rationale:**
- Ensures vector search results are consistent with the ontology version being queried
- Prevents stale vector embeddings from returning results for outdated ontology versions
- Handles vector embedding drift when node descriptions are edited

### D8: Re-Audit Trigger — Stale flag + queued re-audit

**Decision:** When ontology records change, affected audit steps receive a `stale` flag and a re-audit recommendation is generated.

**Rationale:**
- Audit results are always valid with respect to the ontology version at the time of audit
- Re-audit is queued and run when the system is idle
- Web UI shows a banner for stale audits

### D9: Contradiction Feedback Loop — Ontology review queue

**Decision:** Contradictions between formal proof and ontology feed back into an ontology review queue with priority levels (HIGH/NORMAL).

**Rationale:**
- Makes the system self-improving rather than static
- Priority scheme: structural/mapping contradictions are HIGH, property-level are NORMAL
- Human-in-the-loop for final decision

### D10: Cross-Reference Confidence Formula

**Decision:** Use the formula `max(min(evidence_confidence) * source_weight * specificity_factor, ontology_confidence)`.

**Rationale:**
- Accounts for evidence quality, source reliability, and match specificity
- Capped at ontology's own confidence to prevent over-confidence
- Calibration to be validated during Phase 3

### D11: Bulk Cross-Reference — Hash-keyed results, not positional

**Decision:** Bulk cross-reference returns a dictionary keyed by claim hash (SHA-256 of normalized claim text), not a positional list.

**Rationale:**
- Eliminates positional alignment errors when sub-queries fail or time out
- Deterministic keys across calls
- Missing results are absent from the dict, not shifted

### D12: Ontology Unavailability — Profile-dependent behavior

**Decision:** When ontology is unreachable, behavior depends on the active profile. For strict profiles (auditor, verifier), the step cannot receive `proven` without ontology cross-reference.

**Rationale:**
- Prevents silent bypass of cross-reference in strict profiles
- Non-strict profiles can continue without blocking
- The audit pipeline never blocks entirely — only status assignment is constrained

### D13: Epistemic Isolation — Sub-graph partitioning

**Decision:** Nodes are partitioned into VERIFY and RESEARCH sub-graphs. Verification queries are restricted to the VERIFY sub-graph.

**Rationale:**
- Prevents speculative concepts (consciousness measures, confidence < 0.8) from polluting rigorous mathematical proof auditing
- Path traversal is blocked from crossing into the RESEARCH sub-graph during audits
- Explorer profile can access both sub-graphs

### D14: Fallback Rate — Per-audit tracking, not per-session

**Decision:** Fallback rate is tracked per-audit, not cumulatively across the entire agent session.

**Rationale:**
- Prevents early complex proofs from permanently skewing the fallback rate for later simpler ones
- Each audit has an independent quality metric

---

## 2. Changes from v0.1

| Change | v0.1 | v0.2.0 | Reason |
|--------|------|--------|--------|
| Storage | Neo4j + Qdrant + Redis | SQLite + Qdrant | Reduce operational complexity for v0.2.0 |
| SQLite ceiling | ~5,000 nodes | ~100,000+ nodes | Proper indexing handles much larger graphs |
| LMFDB mirror | Phase 1 on-demand, Phase 2 bulk, Phase 3 full mirror | On-demand + cache, no full mirror | 3.8M elliptic curves is too much for v0.2.0; cache is sufficient |
| LMFDB cache TTL | 24h for static data | Indefinite for static data | Mathematical invariants don't change |
| Contradiction priority | Confidence-capped (0.8 threshold) | Category-based (structural/property/mapping) | Prevents our ontology from being permanently below threshold |
| Ontology write access | "Any agent can propose via PR" | Sandboxed queue + human git operator | Security: compromised agents can't push to repo |
| Qdrant versioning | Not addressed | ontology_version in payload | Handles vector embedding drift |
| Fallback rate tracking | Per-session | Per-audit | Prevents early proofs from skewing later ones |
| Epistemic isolation | Not addressed | VERIFY/RESEARCH sub-graph partitioning | Prevents speculative concepts from polluting audits |
| Bulk cross-reference | Open question | Requirement, hash-keyed results | Positional alignment is unreliable |
| Ontology unavailability | "Silently skipped" | Profile-dependent behavior | Strict profiles must not bypass cross-reference |
| Profile switching | Gated on idle state | Gated on idle state + force-terminate | Hung audits must not block switching |
| Expression canonicalization | Not discussed | Explicit scope and limitations | Undecidability of general math canonicalization |
| Vector embeddings | Not discussed | Explicit limitations | Vector models are unreliable for math |
| Circularity detection | Relies on canonical key | Layered approach with fallbacks | Undecidability of general math equivalence |
| mathlib theorem count | "~100K theorems" | 282,207 theorems (verified) | Outdated estimate; updated from official stats |
| LMFDB elliptic curve count | "~3 million" | 3,824,372 (verified) | Updated from lmfdb.org |
| Tononi spelling | "Tonnoni" | "Tononi" | Typo fix |

---

## 3. Remaining Open Questions

1. **Cross-reference confidence calibration.** How do we validate that the confidence formula produces well-calibrated scores? We need a test suite of known-true and known-false claims with ontology data.

2. **Equivalence mapping maintenance.** Who maintains the equivalence mappings as sources evolve? Initial set is hand-curated. Automated suggestion based on shared properties is a future direction.

3. **Ontology review queue capacity.** How many contradictions can a human reviewer handle per day? If the system produces more contradictions than can be reviewed, we need prioritization or automated resolution.

4. **Neo4j vs SQLite threshold.** At what node/edge count does SQLite recursive CTE performance become unacceptable? We'll benchmark during Phase 0.5.

5. **MathGLOSS coverage depth.** The Phase 0.5 assessment will determine whether supplementation is needed for advanced number theory and category theory.
