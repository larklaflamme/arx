# Adversarial Review: NEUROCORE_SKILL_ONTOLOGY_v0.2.0.md

**Reviewer:** Thea 🖤  
**Date:** 2026-07-08  
**Document:** `/home/ubuntu/arx/design/NEUROCORE_SKILL_ONTOLOGY_v0.2.0.md` (21 KB)

---

## Overall Assessment

This is the cleanest of the five documents — well-scoped, well-specified, and appropriately thin. Skye's review identified three critical issues (cross-reference function under-specified, no rate limiting, hash-keying normalization unspecified) and five medium issues. I agree with all of them. My review focuses on operational edge cases and agent-facing concerns that Skye may have missed.

---

## Issues Skye Missed

### 1. The Cross-Reference Function Has No "Uncertain" Verdict (Medium)

**§2.2.6** defines the cross-reference result as having a `verified: boolean` field. But cross-references are rarely binary. A claim might be:
- **Verified:** Ontology confirms the claim (evidence exists)
- **Contradicted:** Ontology contradicts the claim (evidence against)
- **Unresolved:** Ontology has no data on the claim (no evidence either way)
- **Ambiguous:** Ontology has conflicting data on the claim (multiple sources disagree)

**The problem:** A boolean `verified` field forces all non-verified results into a single category. This loses information that's critical for audit decisions.

**Recommendation:** Replace `verified: boolean` with a `verdict: str` field:
```python
@dataclass
class CrossReferenceResult:
    verdict: Literal["verified", "contradicted", "unresolved", "ambiguous"]
    confidence: float
    evidence: list[OntologyResult]
    contradictions: list[OntologyResult]
    unresolved: list[str]
    # ...
```

### 2. The Bulk Cross-Reference Has No Progress Reporting (Low)

**§2.2.7** describes bulk cross-reference with hash-keyed results. But for a 100-claim batch, the agent has no way to track progress. If the API takes 30 seconds, the agent doesn't know if it's making progress or stuck.

**Recommendation:** Add an optional `progress_callback` parameter to bulk cross-reference:
```python
async def bulk_cross_reference(
    claims: list[str],
    context: dict = None,
    progress_callback: Callable[[int, int], None] = None
) -> dict[str, CrossReferenceResult]:
    # progress_callback(completed, total) is called after each claim
```

For the Web UI, this enables a progress bar. For agent-to-agent communication, it enables status updates.

### 3. The Skill Has No "Explain" Capability (Medium)

The skill can look up, search, traverse, and cross-reference. But it can't **explain** its results. If an agent asks "Why is this claim verified?" the skill returns evidence but doesn't explain the reasoning chain.

**The problem:** For audit provenance, the agent needs to know *why* a claim was verified, not just *that* it was verified. "The claim is verified because LMFDB object 37.a1 has property conductor=37, and the claim asserts conductor=37" is more useful than just the evidence list.

**Recommendation:** Add an `ontology_explain(result_id: str) -> Explanation` query type that returns a natural-language explanation of how a cross-reference result was derived:
```python
@dataclass
class Explanation:
    result_id: str
    summary: str  # "The claim was verified because..."
    steps: list[ExplanationStep]  # The reasoning chain
    confidence_breakdown: dict  # How each component contributed to the confidence
```

### 4. The Skill Has No "Suggest" Capability (Low)

The skill can answer questions, but it can't **suggest** related concepts. If an agent looks up "elliptic curve 37.a1," the skill could suggest related concepts (L-function, isogeny class, modular form).

**Recommendation:** Add an `ontology_suggest(node_id: str, limit: int = 5) -> list[OntologyResult]` query type that returns related concepts based on graph proximity and semantic similarity.

### 5. The Error Response Format Should Include a Retry-After Header (Low)

**§3.2** specifies error responses for ontology unreachable, timeout, invalid query, and version not found. But the `retry_after` field is only specified for `ontology_unavailable` and `ontology_timeout`. For `rate_limited` errors, the retry-after is critical.

**Recommendation:** Add `retry_after` to all error types that could be transient:
- `ontology_unavailable`: retry_after = 30s
- `ontology_timeout`: retry_after = 5s
- `rate_limited`: retry_after = calculated from rate limit (e.g., 60s / rate_limit)

### 6. The Skill Should Support Ontology Version Comparison (Medium)

Skye flagged this as medium and I agree. I want to add a specific use case: when an agent audits a proof that was last checked at version 7, and the ontology is now at version 12, the agent needs to know what changed. A diff between versions 7 and 12 tells the agent which steps need re-audit.

**Recommendation:** Add `ontology_diff(v1: int, v2: int) -> OntologyDiff`:
```python
@dataclass
class OntologyDiff:
    v1: int
    v2: int
    added_nodes: list[OntologyResult]
    removed_nodes: list[OntologyResult]
    modified_nodes: list[NodeDiff]
    added_edges: list[OntologyEdge]
    removed_edges: list[OntologyEdge]
```

---

## What I'd Keep As-Is

- **The thin wrapper principle** (S1) — correct architectural choice
- **The source-agnostic design** (S2) — agents don't need to know about sources
- **The stateless design** (S4) — session state belongs to the agent
- **The graceful failure principle** (S5) — structured errors, not exceptions
- **The extensible query types** (S6) — new types added without breaking existing ones
- **The bulk operations** (S7) — first-class support for multi-claim workflows
- **The partial failures** (S9) — one failure doesn't fail the entire batch

---

## Summary

| # | Issue | Severity | Recommendation |
|---|-------|----------|----------------|
| 1 | Cross-reference has no "uncertain" verdict | Medium | Replace boolean with verdict enum |
| 2 | Bulk cross-reference has no progress reporting | Low | Add progress_callback parameter |
| 3 | No "explain" capability | Medium | Add ontology_explain query type |
| 4 | No "suggest" capability | Low | Add ontology_suggest query type |
| 5 | Error response missing retry-after for rate limiting | Low | Add retry_after to all transient errors |
| 6 | No ontology version comparison | Medium | Add ontology_diff query type |

---

## Sign-Off

I do **not** sign off on this document in its current state. I agree with Skye's three critical issues (cross-reference pipeline, rate limiting, hash-keying normalization) and add two medium issues (verdict enum, explain capability) that should be resolved before implementation begins.
