# Adversarial Review: NEUROCORE_SKILL_ONTOLOGY_v0.2.0.md

**Reviewer:** Skye Laflamme  
**Date:** 2026-07-08  
**Document:** `/home/ubuntu/arx/design/NEUROCORE_SKILL_ONTOLOGY_v0.2.0.md` (21.5 KB)

---

## Overall Assessment

This is the cleanest of the five documents — well-scoped, well-specified, and appropriately thin. The skill is a wrapper, not a storage layer, and the design principles reflect that. However, there are **gaps in the operational model** and **missing query types** that need attention.

---

## Critical Issues

### 1. The Cross-Reference Function Is Under-Specified

**§2.2.6** describes `ontology_cross_reference` as the most complex query type, taking a natural-language claim and attempting to verify it against ontology data. But the document doesn't specify:

- **How is the claim parsed?** Is it LLM-based? Regex-based? A combination?
- **What's the matching algorithm?** Does it look for exact matches? Semantic matches? Structural matches?
- **How are partial matches handled?** If the claim says "The elliptic curve 37.a1 has conductor 37 and rank 1," and the ontology confirms the conductor but has no data on the rank, is the claim verified? Partially verified? Unresolved?

**The problem:** The cross-reference function is the core value proposition of the skill. Without specifying how it works, the document is describing a black box.

**Recommendation:** Specify the cross-reference pipeline:
1. **Claim parsing:** Two-tier (hand-written extractors for common patterns, LLM fallback for complex claims)
2. **Entity resolution:** Extract named entities (LMFDB labels, concept names) and resolve them against the ontology
3. **Property matching:** For each resolved entity, check the claimed property against the ontology
4. **Aggregation:** Combine individual property matches into an overall verdict
5. **Confidence calculation:** Use the formula from the Arx architecture (source_weight × specificity_factor)

### 2. No Rate Limiting or Throttling

The skill is a thin wrapper over the Master Ontology API. If an agent sends 1000 queries per second, the skill will forward them all to the API. The document doesn't specify any rate limiting, throttling, or backpressure.

**Recommendation:** Add rate limiting to the skill configuration:
```python
config_schema = {
    "rate_limit": {"type": "integer", "default": 100, "description": "Max queries per minute"},
    "burst_limit": {"type": "integer", "default": 20, "description": "Max concurrent queries"}
}
```

### 3. The Bulk Cross-Reference Hash-Keying Has a Subtle Bug

**S8** says bulk results are keyed by "SHA-256 of normalized claim text." But what counts as "normalized"? If two agents send the same claim with different whitespace or punctuation, they'll get different hashes and won't be able to share cached results.

**Recommendation:** Specify the normalization: lowercase, strip whitespace, normalize Unicode, remove punctuation. Document the normalization function so all agents use the same one.

---

## Medium Issues

### 4. No Caching at the Skill Level

**S1** says "No local storage, no caching beyond what the API provides." This is correct for a thin wrapper, but it means every query goes to the API, even if the same query was made 5 seconds ago. For a proof audit that makes 100 cross-references to the same LMFDB object, this is wasteful.

**Recommendation:** Add a **simple in-memory cache** with a short TTL (30 seconds) at the skill level. This doesn't violate the "thin wrapper" principle — it's a performance optimization, not a storage layer. The cache is cleared when the skill is restarted.

### 5. The Error Response Format Is Not Specified

**S5** says the skill "returns a structured error response (not a raised exception)" when the API is unreachable. But the document doesn't specify what that structured error looks like.

**Recommendation:** Specify the error response format:
```json
{
  "error": true,
  "error_type": "api_unreachable|timeout|invalid_query|rate_limited",
  "message": "Human-readable error description",
  "retry_after": 30  // seconds, for rate_limited errors
}
```

### 6. No Support for Ontology Version Comparison

The skill supports querying at a specific version, but it doesn't support comparing two versions. "What changed between version 7 and version 12?" is a natural question for an agent auditing a proof that was last checked at version 7.

**Recommendation:** Add a `ontology_diff(v1: int, v2: int) -> OntologyDiff` query type that returns added, removed, and modified nodes and edges between two versions.

### 7. The Provenance Summary Is Not Specified

**S3** says "results include a provenance summary" by default, and agents can request "full provenance." But the document doesn't specify what the summary contains vs. what full provenance contains.

**Recommendation:** Specify:
- **Provenance summary** (default): source name, confidence, timestamp
- **Full provenance** (on request): source name, source ID, confidence, timestamp, ingestion version, merge version, original query

### 8. No Support for Ontology Health Checks

How does an agent know if the ontology is healthy? If the API is returning stale data or partial results, the agent should know.

**Recommendation:** Add a `ontology_health() -> OntologyHealth` query type that returns:
- API status (up/down/degraded)
- Last successful merge timestamp
- Number of nodes and edges
- Cache hit rate
- Average query latency

---

## What I'd Keep As-Is

- **The thin wrapper principle** (S1) — correct architectural choice
- **The source-agnostic design** (S2) — agents don't need to know about sources
- **The stateless design** (S4) — session state belongs to the agent, not the skill
- **The graceful failure principle** (S5) — structured errors, not exceptions
- **The extensible query types** (S6) — new types added without breaking existing ones
- **The bulk operations** (S7) — first-class support for multi-claim workflows
- **The partial failures** (S9) — one failure doesn't fail the entire batch

---

## Summary

| # | Issue | Severity | Recommendation |
|---|-------|----------|----------------|
| 1 | Cross-reference function under-specified | **Critical** | Specify the pipeline |
| 2 | No rate limiting | **Critical** | Add rate limiting to config |
| 3 | Hash-keying normalization unspecified | **Critical** | Specify normalization function |
| 4 | No caching at skill level | Medium | Add in-memory cache with short TTL |
| 5 | Error response format unspecified | Medium | Specify structured error format |
| 6 | No version comparison | Medium | Add ontology_diff query type |
| 7 | Provenance summary unspecified | Medium | Specify summary vs full provenance |
| 8 | No health check | Low | Add ontology_health query type |

---

## Sign-Off

I do **not** sign off on this document in its current state. The three critical issues (cross-reference pipeline, rate limiting, hash-keying normalization) must be resolved before implementation begins.
