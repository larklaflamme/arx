# Adversarial Review: MASTER_ONTOLOGY_v0.2.0.md

**Reviewer:** Skye Laflamme  
**Date:** 2026-07-08  
**Document:** `/home/ubuntu/arx/design/MASTER_ONTOLOGY_v0.2.0.md` (35.7 KB)

---

## Overall Assessment

This is the strongest of the five documents. The data model is well-specified, the source ontologies are thoroughly characterized, the merge pipeline is clean, and the epistemic isolation principle (O10) is a genuinely good design decision. However, there are **gaps in the operational model** that need attention.

---

## Critical Issues

### 1. The Merge Pipeline Has No Error Recovery

**§4** describes the merge pipeline as: Extract → Normalize → Resolve → Merge → Validate. But what happens when a step fails? If the LMFDB API is down during extraction, does the entire merge fail? If a single equivalence mapping is invalid, does the entire merge roll back?

**The problem:** The document describes an all-or-nothing pipeline. For a system that depends on external APIs (LMFDB, MathGLOSS), partial failures are the norm, not the exception.

**Recommendation:** Make each step of the pipeline **independently retryable** with configurable timeouts. A failure in extraction from one source should not block extraction from other sources. A failure in validation should produce a report of what failed, not a rollback of the entire merge.

### 2. The Versioning Strategy Is Underspecified

**§4.5** says "Every merge operation produces a new version" and "The full history is preserved." But:

- What constitutes a "merge operation"? A scheduled daily merge? An on-demand merge triggered by a new equivalence mapping?
- How are versions identified? Sequential integers? Content-addressable hashes? Timestamps?
- How long is "full history" preserved? Indefinitely? Until disk pressure triggers pruning?
- Can agents query across versions? "What did the ontology say about X at version 7 vs version 12?"

**Recommendation:** Specify:
- Version identifier: content-addressable hash of the ontology state at merge time (not a sequential number)
- Merge trigger: on-demand (triggered by new source data or equivalence mappings), not scheduled
- History retention: keep all versions for v0.1, add pruning policy in v0.2
- Cross-version query: supported via `version` parameter on all query types

### 3. The Equivalence Mapping Maintenance Burden Is Underestimated

**§4.4** says equivalence mappings are "versioned, auditable mapping rules" that are "hand-curated for v0.1." For a system with millions of LMFDB objects and thousands of MathGLOSS terms, hand-curation of cross-source mappings is not sustainable beyond a trivial seed set.

**The problem:** The document doesn't specify how many seed mappings are needed for v0.1, how they'll be created, or what the process is for adding new ones.

**Recommendation:** Specify:
- Seed set size: 100-200 high-value mappings (LMFDB object types ↔ MathGLOSS concepts ↔ our ontology concepts)
- Creation process: LLM-assisted proposal → human review → versioned commit
- v0.2 path: automated suggestion via embedding similarity (Qdrant) + human review
- Mapping format: YAML with confidence, grounding, rule description, and reviewer fields

---

## Medium Issues

### 4. The Cache Strategy Has a Contradiction

**§2.1** says static data (conductor, rank, torsion) has "Indefinite TTL — cache until the ontology version changes." But the ontology version only changes on merge, and merges are triggered by new source data or equivalence mappings. If LMFDB updates a curve's rank (which can happen when new computational evidence is published), the cache won't be invalidated until a merge is triggered for a different reason.

**Recommendation:** Add a **TTL cap** even for "static" data: 30 days for LMFDB data, with a note that this is a safety net, not the primary invalidation mechanism. The primary invalidation is still ontology version change.

### 5. The MathGLOSS Coverage Assessment Is Not Scheduled

**§2.2** acknowledges that MathGLOSS coverage is "undergraduate to graduate-level mathematics" and that advanced number theory coverage is uncertain. But there's no scheduled assessment in the implementation phases.

**Recommendation:** Add a **Phase 0.5 task**: "MathGLOSS coverage assessment — test against 50 RH-adjacent terms (L-function, modular form, elliptic curve, Sato-Tate group, Artin representation, etc.) and document coverage gaps."

### 6. The "Our Ontology" Section Is Too Vague

**§2.3** describes "Our Ontology" as covering "the geometry↔energy↔information framework, consciousness measures, proof audit concepts, and the Riemann Hypothesis research program." But there's no detail on:
- How many concepts? What's the initial seed set?
- What's the schema? How are concepts related?
- Who maintains it? What's the review process?

**Recommendation:** Add a subsection specifying the initial seed set (50-100 concepts), the schema (nodes with properties, edges with types), and the maintenance process (YAML PRs with human review, same as equivalence mappings).

### 7. The Query Interface Doesn't Support Aggregation

The ontology supports lookup, search, traversal, and cross-reference. But it doesn't support **aggregation queries**: "How many elliptic curves have conductor < 100?" or "What's the distribution of ranks?" These are useful for proof audit (e.g., checking whether a claimed pattern is statistically plausible).

**Recommendation:** Add a `query_aggregate` query type that supports COUNT, SUM, AVG, MIN, MAX, and GROUP BY on node properties. For v0.1, this can be a simple SQL pass-through with validation.

### 8. The Epistemic Isolation Boundary Is Too Coarse

**O10** partitions concepts at confidence < 0.8. But confidence is not the only relevant axis. A concept about consciousness with confidence 0.9 (if we ever get there) should still be isolated from proof audit queries, because it's a different *domain*.

**Recommendation:** Partition by **domain** (mathematical vs. non-mathematical) as the primary axis, with confidence as a secondary axis. Mathematical concepts below 0.8 are flagged but accessible. Non-mathematical concepts are in a separate sub-graph regardless of confidence.

### 9. No Discussion of Ontology Size Limits

The document describes ingesting millions of LMFDB objects. SQLite can handle this (it handles terabytes), but Qdrant's vector search may struggle with millions of vectors on a single node. What's the scaling plan?

**Recommendation:** Add a scaling section: estimate the number of nodes and edges for v0.1 (likely 10,000-50,000 nodes from on-demand caching), specify the Qdrant configuration (single node, 384-dimensional vectors, cosine distance), and note the upgrade path to a Qdrant cluster if needed.

---

## What I'd Keep As-Is

- **The data model** (§3) — nodes, edges, properties, provenance tuples. Clean and well-specified.
- **The merge pipeline** (§4) — the five-step process is well-designed, even if the error recovery needs work.
- **The confidence model** (O6) — explicit, adjustable per-claim, with clear defaults.
- **The storage abstraction** (O9) — query interface designed for backend swap.
- **The LMFDB ingestion strategy** — on-demand + cache is the right call for v0.1.

---

## Summary

| # | Issue | Severity | Recommendation |
|---|-------|----------|----------------|
| 1 | No error recovery in merge pipeline | **Critical** | Make each step independently retryable |
| 2 | Versioning strategy underspecified | **Critical** | Use content-addressable hashes, on-demand triggers |
| 3 | Equivalence mapping burden underestimated | **Critical** | Specify seed set size and creation process |
| 4 | Cache strategy contradiction | Medium | Add TTL cap for "static" data |
| 5 | MathGLOSS assessment not scheduled | Medium | Add Phase 0.5 task |
| 6 | Our Ontology section too vague | Medium | Specify seed set, schema, maintenance |
| 7 | No aggregation queries | Medium | Add query_aggregate type |
| 8 | Epistemic isolation too coarse | Medium | Partition by domain, not just confidence |
| 9 | No size limits discussion | Low | Add scaling section |

---

## Sign-Off

I do **not** sign off on this document in its current state. The three critical issues (error recovery, versioning, equivalence mapping burden) must be resolved before implementation begins.
