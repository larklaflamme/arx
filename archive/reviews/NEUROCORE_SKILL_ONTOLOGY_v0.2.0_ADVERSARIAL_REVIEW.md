# Adversarial Review: NEUROCORE_SKILL_ONTOLOGY_v0.2.0.md

**Reviewer:** AXIOMA (adversarial)  
**Date:** 2026-07-08  
**Status:** 7 issues found (1 critical, 3 major, 3 minor)

---

## Critical Issues

### C1. `fallback_rate` in `CrossReferenceResult` is semantically wrong

**Location:** §2.2, `ontology_cross_reference` return type, and §2.3, `CrossReferenceResult` dataclass

**Issue:** The `CrossReferenceResult` dataclass includes a `fallback_rate: float` field, described as "fallback rate for this audit (not cumulative session)." But a single cross-reference call processes one claim. The fallback rate for a single claim is either 0.0 (extractor used) or 1.0 (LLM fallback used). A per-claim fallback rate is meaningless — it's always 0 or 1.

The document says "A high fallback rate (> 0.5) reduces the overall cross-reference confidence and is surfaced in the result metadata." But a single claim can't have a fallback rate of 0.5 — it's either extracted or not.

**Impact:** The `fallback_rate` field in `CrossReferenceResult` will always be 0.0 or 1.0, making it useless for the stated purpose of "reducing overall cross-reference confidence when fallback rate is high." The confidence reduction would happen on every LLM-fallback claim (rate = 1.0) and never on extractor claims (rate = 0.0), which is equivalent to just checking `fallback_used`.

**Recommendation:** Either:
- (a) Remove `fallback_rate` from `CrossReferenceResult` and keep only `fallback_used`. The per-audit fallback rate is computed by Arx's meta-cognition layer (§3.3.1 of the architecture document), not by the skill.
- (b) Or add a `fallback_rate` parameter to the `context` dict that the caller passes in, and the skill echoes it back in the result. This way Arx can track the cumulative rate and pass it to each cross-reference call.

Option (a) is cleaner — the skill should be stateless (Principle S4), and tracking cumulative metrics is the caller's responsibility.

---

## Major Issues

### M1. `ontology_semantic_search` doesn't support version filtering

**Location:** §2.2, `ontology_semantic_search`

**Issue:** The `ontology_semantic_search` method signature is:
```python
def ontology_semantic_search(text: str, limit: int = 10, source_filter: list[str] = None) -> list[OntologyResult]
```

It doesn't have a `version` parameter, unlike all other query types (`lookup`, `search`, `traverse`, `cross_reference`, `bulk_cross_reference`). This means semantic search always queries the latest ontology version, which is inconsistent with the version-aware design.

**Impact:** If an audit is performed against ontology version 5, but the semantic search returns results from version 12, the results may include concepts that didn't exist at version 5. This breaks reproducibility.

**Recommendation:** Add a `version` parameter to `ontology_semantic_search`:
```python
def ontology_semantic_search(text: str, limit: int = 10, source_filter: list[str] = None, version: int = 0) -> list[OntologyResult]
```

The Master Ontology API should filter Qdrant results by `ontology_version` (as described in MASTER_ONTOLOGY_v0.2.0.md §5.2).

---

### M2. Error handling for "version not found" returns 404 but the skill response says "success: false"

**Location:** §3.2, Error handling table

**Issue:** The error handling table says:
```
| Version not found | 404 | `{"success": false, "error": "ontology_version_error", ...}` |
```

But the "Node not found" case returns:
```
| Node not found | 404 | `{"success": true, "results": []}` (empty result, not an error) |
```

These are inconsistent: both return HTTP 404, but one returns `success: true` and the other returns `success: false`. A version not found is arguably a client error (the caller asked for a version that doesn't exist), but a node not found is also a client error (the caller asked for a node that doesn't exist).

**Impact:** Callers need to handle two different response formats for the same HTTP status code, which is error-prone.

**Recommendation:** Make them consistent. Either:
- Both return `success: true` with empty results (treating "not found" as a valid query result)
- Both return `success: false` with an error message (treating "not found" as an error)

Option 1 is better for the node case (a node not existing is a normal query result), but option 2 is better for the version case (a version not existing is a configuration error). I recommend keeping the distinction but documenting it clearly: "Node not found is a normal query result (success: true, empty results). Version not found is a configuration error (success: false, error message)."

---

### M3. The skill's `config_schema` requires `api_key` but the ontology API doesn't require authentication

**Location:** §2.1, Skill Metadata

**Issue:** The `config_schema` includes `"api_key": {"type": "string", "description": "Optional API key"}` but the Master Ontology API description (§3.1) doesn't mention authentication. If the API doesn't require an API key, why is it in the config schema?

**Impact:** Unnecessary configuration complexity. Every agent that installs the skill needs to configure an API key that isn't used.

**Recommendation:** Either:
- (a) Remove `api_key` from the config schema if the API doesn't require authentication
- (b) Add authentication to the Master Ontology API design and document it

---

## Minor Issues

### m1. `ontology_path` is in the skill interface but not in the provides list

**Location:** §2.1 vs §2.2

**Issue:** The `provides` list in the skill metadata is:
```python
provides=["ontology_lookup", "ontology_search", "ontology_traverse", "ontology_semantic_search", "ontology_cross_reference", "ontology_bulk_cross_reference"]
```

But §2.2 also defines `ontology_path` and `ontology_version_info`. These are missing from the `provides` list.

**Impact:** The skill metadata is incomplete. Tools that depend on the `provides` list (e.g., NeuroCore's tool discovery) would not find `ontology_path` or `ontology_version_info`.

**Recommendation:** Add `"ontology_path"` and `"ontology_version_info"` to the `provides` list.

### m2. The `CrossReferenceResult` dataclass has `claim_hash` but the `CrossReferenceResult` in §2.3 doesn't include it

**Location:** §2.3, `CrossReferenceResult` dataclass

**Issue:** The `CrossReferenceResult` dataclass in §2.3 includes `claim_hash: str`, but the return description in §2.2 for `ontology_cross_reference` doesn't list `claim_hash` in the return fields. It lists: `verified`, `confidence`, `evidence`, `contradictions`, `unresolved`, `fallback_used`, `fallback_rate`.

**Impact:** The documentation is inconsistent. A developer implementing the skill would not know whether to include `claim_hash` in the result.

**Recommendation:** Add `claim_hash` to the return field list in §2.2.

### m3. The skill's project root is `/home/ubuntu/neurocore-skills/neurocore-skill-ontology/` but the Arx project structure in ARX_ARCHITECTURE_v0.2.0.md §4 shows it under `/home/ubuntu/arx/neurocore-skill-ontology/`

**Location:** This document's header vs ARX_ARCHITECTURE_v0.2.0.md §4

**Issue:** The skill document says the project root is `/home/ubuntu/neurocore-skills/neurocore-skill-ontology/` but the architecture document's project structure shows it under `/home/ubuntu/arx/neurocore-skill-ontology/`. These are different locations.

**Impact:** When implementing, there will be confusion about where the skill package should live.

**Recommendation:** Decide on one location and update both documents. The architecture document's location (`/home/ubuntu/arx/neurocore-skill-ontology/`) makes more sense if the skill is primarily developed as part of Arx. The skill document's location (`/home/ubuntu/neurocore-skills/`) makes more sense if it's a shared skill used by multiple agents.

---

## Cross-Document Issues

### X1. Claim hash normalization differs between documents

**Location:** This document §2.2 vs ARX_ARCHITECTURE_v0.2.0.md §3.2.4

**Issue:** The skill document says:
```python
def claim_hash(claim_text: str) -> str:
    normalized = claim_text.strip().lower()
    return hashlib.sha256(normalized.encode('utf-8')).hexdigest()
```

The architecture document says:
```python
# SHA-256(normalize(claim_text)) where normalize strips whitespace and normalizes Unicode
```

The skill document uses `.strip().lower()` (strip + lowercase), while the architecture document says "strips whitespace and normalizes Unicode" (no mention of lowercase). If these differ, the same claim will produce different hashes in different contexts.

**Recommendation:** Align the normalization: both should use the same algorithm. I recommend `strip().lower().normalize('NFKC')` for both, and document it in a shared location.

---

## Summary

| Severity | Count | Key Action |
|----------|-------|------------|
| Critical | 1 | Fix fallback_rate semantics (remove from per-claim result) |
| Major | 3 | Add version to semantic_search, fix error handling inconsistency, clarify API key |
| Minor | 3 | Fix provides list, add claim_hash to return fields, align project root |
| Cross-doc | 1 | Align claim hash normalization algorithm |

**Overall assessment:** The skill design is clean and well-structured. The critical issue is the semantically wrong `fallback_rate` field, which would cause confusion during implementation. The major issues are all fixable with small changes.
