# Neurocore-Skill-Ontology — Design v0.2.0

**Status:** Final (incorporating all independent review feedback)  
**Date:** 2026-07-08  
**Project root:** `/home/ubuntu/neurocore-skills/neurocore-skill-ontology/`  
**Related documents:**
  - [MASTER_ONTOLOGY_v0.2.0.md](../MASTER_ONTOLOGY_v0.2.0.md) — the Master Ontology this skill wraps
  - [ARX_ARCHITECTURE_v0.2.0.md](../ARX_ARCHITECTURE_v0.2.0.md) — Arx agent architecture (consumer, §3.9)
  - NeuroCore v0.3.0 — framework for skill-based agent capabilities
  - NeuroCore-Skills README — skill package conventions

---

## 0. Executive Summary

`neurocore-skill-ontology` is a **NeuroCore skill** that provides every agent in the ecosystem (Skye, Thea, Theoria, Axioma, Arx) with a unified query interface to the **Master Ontology** — the merged, reconciled, versioned graph of mathematical knowledge from LMFDB, MathGLOSS, and our own ontology.

The skill wraps the Master Ontology's API and exposes it as a standard NeuroCore skill, following the same conventions as `neurocore-skill-tavily`, `neurocore-skill-wolfram`, and other skills in the monorepo. Any agent that has the skill installed can:

- Look up mathematical objects by ID or name
- Search for concepts by label or semantic similarity
- Traverse relationships between objects
- Resolve ambiguous terminology to canonical concepts
- Cross-reference claims against verified LMFDB data
- Bulk cross-reference multiple claims in a single call (results keyed by claim hash)
- Query the ontology at a specific version

The skill is **source-agnostic**: the agent doesn't need to know whether a result came from LMFDB, MathGLOSS, or our ontology — the skill returns unified results with provenance metadata.

---

## 1. Design Principles

| # | Principle | What it constrains |
|---|---|---|
| S1 | **The skill is a thin wrapper over the Master Ontology API.** | No local storage, no caching beyond what the API provides. The skill translates agent queries into API calls and formats the results. |
| S2 | **Agents don't need to know about sources.** | The skill abstracts away LMFDB, MathGLOSS, and our ontology. Agents ask "what is the conductor of L-function 37.a1?" and get the answer, regardless of which source provides it. |
| S3 | **Provenance is transparent on request.** | By default, results include a provenance summary. Agents can request full provenance (which source, what confidence, when ingested) if they need it. |
| S4 | **The skill is stateless.** | Each query is independent. Session state (e.g., "the agent is working on an elliptic curve problem") is managed by the agent, not the skill. |
| S5 | **Fail gracefully.** | If the Master Ontology is unreachable, the skill returns a structured error response (not a raised exception). It never fabricates data. |
| S6 | **Extensible query types.** | New query types can be added without breaking existing ones. The skill uses a `query_type` parameter to dispatch. |
| S7 | **Bulk operations are first-class.** | For multi-claim workflows (e.g., auditing a 100-step proof), the skill supports bulk cross-reference to avoid N sequential API calls. |
| S8 | **Bulk results are hash-keyed, not positional.** | Bulk cross-reference returns a dictionary keyed by claim hash (SHA-256 of normalized claim text), not a positional list. This eliminates positional alignment errors when sub-queries fail or time out. |
| S9 | **Partial failures are reported, not hidden.** | Bulk operations return individual results per claim. A failure for one claim does not fail the entire batch. |

---

## 2. Skill Interface

### 2.1 Skill Metadata

```python
skill_meta = SkillMeta(
    name="ontology",
    description="Query the Master Ontology — a unified graph of mathematical knowledge from LMFDB, MathGLOSS, and our own ontology. Look up objects, search concepts, traverse relationships, cross-reference claims.",
    provides=["ontology_lookup", "ontology_search", "ontology_traverse", "ontology_semantic_search", "ontology_cross_reference", "ontology_bulk_cross_reference"],
    consumes=[],
    config_schema={
        "type": "object",
        "properties": {
            "api_url": {"type": "string", "description": "Master Ontology API base URL"},
            "api_key": {"type": "string", "description": "Optional API key"},
            "default_version": {"type": "integer", "description": "Default ontology version to query (0 = latest)", "default": 0},
            "timeout_seconds": {"type": "integer", "default": 10}
        },
        "required": ["api_url"]
    },
    tags=["ontology", "mathematics", "lmfdb", "mathgloss", "knowledge-graph"]
)
```

### 2.2 Query Types

The skill exposes the following query types. Each is a method on the skill class.

#### `ontology_lookup(id: str, version: int = 0) -> OntologyResult`

Look up a node by its canonical ID.

- `id`: Canonical ID (e.g., `"lmfdb:37.a1"`, `"mathgloss:Q12345"`, `"our:consciousness-phi"`)
- `version`: Ontology version to query (0 = latest)
- Returns: The node with all properties, edges (optionally), and provenance

#### `ontology_lookup_by_source(source: str, source_id: str, version: int = 0) -> OntologyResult`

Look up a node by its original source ID.

- `source`: Source name (`"lmfdb"`, `"mathgloss"`, `"our"`)
- `source_id`: ID in the source's namespace
- `version`: Ontology version to query
- Returns: The node with all properties and provenance

#### `ontology_search(query: str, limit: int = 10, source_filter: list[str] = None, version: int = 0) -> list[OntologyResult]`

Full-text search on node labels and descriptions.

- `query`: Search string
- `limit`: Max results
- `source_filter`: Optional list of sources to restrict to (e.g., `["lmfdb"]`)
- `version`: Ontology version to query
- Returns: List of matching nodes with relevance scores

#### `ontology_traverse(node_id: str, edge_type: str = None, direction: str = "out", max_depth: int = 1, version: int = 0) -> list[OntologyEdge]`

Follow edges from a node.

- `node_id`: Starting node ID
- `edge_type`: Optional edge type filter (e.g., `"has_lfunction"`, `"subclass_of"`)
- `direction`: `"out"`, `"in"`, or `"both"`
- `max_depth`: How many hops to traverse
- `version`: Ontology version to query
- Returns: List of edges with source, target, type, and properties

#### `ontology_semantic_search(text: str, limit: int = 10, source_filter: list[str] = None) -> list[OntologyResult]`

Semantic similarity search using vector embeddings.

- `text`: Natural language description
- `limit`: Max results
- `source_filter`: Optional source filter
- Returns: List of matching nodes with similarity scores

#### `ontology_cross_reference(claim: str, context: dict = None) -> CrossReferenceResult`

Cross-reference a single claim against the ontology. This is the most complex query type — it takes a natural-language claim and attempts to verify it against ontology data.

- `claim`: A natural-language claim (e.g., "The elliptic curve 37.a1 has conductor 37")
- `context`: Optional context (e.g., proof step ID, current profile settings, strictness level)
- Returns: A structured cross-reference result with:
  - `verified`: boolean (whether the claim is consistent with ontology data)
  - `confidence`: float (how confident the system is in this assessment)
  - `evidence`: list of ontology records that support or contradict the claim
  - `contradictions`: list of ontology records that contradict the claim
  - `unresolved`: list of parts of the claim that couldn't be checked
  - `fallback_used`: whether LLM fallback was used for claim parsing
  - `fallback_rate`: fallback rate for this audit (not cumulative session)

**Claim parsing:** Uses a two-tier approach:

1. **Hand-written extractors** (tier 1): Pattern-matched extractors for high-frequency claim types:
   - L-function conductor, degree, sign, Euler factors
   - Elliptic curve rank, torsion, conductor
   - Number field degree, discriminant, Galois group
   - Modular form level, weight, character, dimension
   - Dirichlet character modulus, conductor, order
   - Sato-Tate group name, measure
   
   Each extractor is a function that takes a claim string and returns a structured `Claim` object (or None if the pattern doesn't match). Extractors are registered in a priority-ordered list.

2. **LLM fallback** (tier 2): For complex or novel claim types not covered by extractors. The LLM is prompted with a structured output schema to produce a machine-parseable `Claim` object.

The fallback rate is tracked **per-audit** (not per-session), to prevent early complex proofs from skewing the metric for later simpler ones:
```python
FallbackMetrics {
    total_claims: int,
    extractor_claims: int,
    llm_fallback_claims: int,
    fallback_rate: float,  # llm_fallback / total
}
```

A high fallback rate (> 0.5) reduces the overall cross-reference confidence and is surfaced in the result metadata.

#### `ontology_bulk_cross_reference(claims: list[str], context: dict = None) -> dict[str, CrossReferenceResult]`

Cross-reference a list of claims in a single call. Results are returned as a **dictionary keyed by claim hash** (SHA-256 of the normalized claim text), not a positional list.

- `claims`: A list of natural-language claims
- `context`: Optional context (shared across all claims)
- Returns: A dictionary mapping claim hashes to `CrossReferenceResult` objects

**Rationale for hash-keyed results:**
- Eliminates positional alignment errors when sub-queries fail or time out
- If a sub-query fails, only that claim's result is missing from the dict, not shifted
- Deterministic keys across calls (same claim text → same hash)
- Consumers can match results to input claims by hash, not by position

**Partial failure handling:**
- Each claim is processed independently
- A failure for one claim (timeout, parsing error) does not fail the entire batch
- Failed claims are absent from the result dict (or present with `success: false`)
- The caller can retry only the failed claims

**Claim hash computation:**
```python
import hashlib
def claim_hash(claim_text: str) -> str:
    normalized = claim_text.strip().lower()
    return hashlib.sha256(normalized.encode('utf-8')).hexdigest()
```

#### `ontology_version_info() -> VersionInfo`

Get information about the current ontology version.

- Returns: version number, timestamp, sources ingested, node/edge counts

#### `ontology_path(start_id: str, end_id: str, max_depth: int = 5) -> list[list[OntologyEdge]]`

Find paths between two nodes.

- `start_id`: Starting node ID
- `end_id`: Target node ID
- `max_depth`: Maximum path length
- Returns: List of paths (each path is a list of edges)

### 2.3 Response Types

```python
@dataclass
class OntologyResult:
    id: str
    type: str
    label: str
    properties: dict
    edges: list[OntologyEdge]  # optional, included on request
    provenance: list[ProvenanceRecord]
    confidence: float
    ontology_version: int

@dataclass
class OntologyEdge:
    id: str
    type: str
    source_id: str
    target_id: str
    properties: dict
    provenance: list[ProvenanceRecord]
    confidence: float

@dataclass
class ProvenanceRecord:
    source: str          # "lmfdb", "mathgloss", "our"
    source_id: str       # original ID in the source
    confidence: float    # 0.0-1.0
    ingested_at: str     # ISO datetime

@dataclass
class CrossReferenceResult:
    claim: str
    claim_hash: str      # SHA-256 of normalized claim text
    verified: bool
    confidence: float
    evidence: list[OntologyResult]
    contradictions: list[OntologyResult]
    unresolved: list[str]
    ontology_version: int
    fallback_used: bool  # whether LLM fallback was used for claim parsing
    fallback_rate: float # fallback rate for this audit (not cumulative session)

@dataclass
class VersionInfo:
    version: int
    created_at: str
    sources: list[str]
    node_count: int
    edge_count: int
    description: str
```

---

## 3. Integration with the Master Ontology

### 3.1 API Communication

The skill communicates with the Master Ontology via a REST API:

```
GET    /api/v1/lookup/{id}?version={version}
GET    /api/v1/lookup-by-source?source={source}&source_id={source_id}&version={version}
GET    /api/v1/search?q={query}&limit={limit}&source={source}&version={version}
GET    /api/v1/traverse/{node_id}?edge_type={type}&direction={dir}&depth={depth}&version={version}
GET    /api/v1/semantic-search?q={text}&limit={limit}&source={source}
POST   /api/v1/cross-reference   body: { claim, context }
POST   /api/v1/bulk-cross-reference   body: { claims: [...], context }
GET    /api/v1/version
GET    /api/v1/path?start={start_id}&end={end_id}&max_depth={depth}
```

### 3.2 Error Handling

The skill returns **structured error responses** rather than raising raw exceptions. This ensures that calling agents can gracefully degrade without wrapping every call in try-except blocks.

| Error | HTTP Status | Skill Response |
|---|---|---|
| Ontology unreachable | Connection error | `{"success": false, "error": "ontology_unavailable", "message": "Master Ontology API is unreachable", "retry_after": 30}` |
| Node not found | 404 | `{"success": true, "results": []}` (empty result, not an error) |
| Query timeout | 408 | `{"success": false, "error": "ontology_timeout", "message": "Query timed out after 10s", "retry_after": 5}` |
| Invalid query | 400 | `{"success": false, "error": "ontology_query_error", "message": "Invalid query: ..."}` |
| Version not found | 404 | `{"success": false, "error": "ontology_version_error", "message": "Version 42 not found. Available versions: 1-15"}` |

### 3.3 Caching

The skill relies on the **Master Ontology API's server-side cache** (SQLite-based LMFDB cache, indefinite TTL for static data). The skill itself does not maintain an independent in-memory cache, to avoid:

- Redundant memory footprint across multiple agents
- Cache inconsistency when the ontology changes
- Stale data from expired TTLs

If agent-side caching is needed in the future, it should be implemented as a shared Redis cache (not per-agent Python dicts) to ensure consistency across all agents.

---

## 4. Agent-Facing Usage

### 4.1 From a Blueprint

```yaml
steps:
  - skill: ontology
    action: lookup
    params:
      id: "lmfdb:37.a1"
  - skill: ontology
    action: traverse
    params:
      node_id: "{{ steps.lookup.result.id }}"
      edge_type: "has_lfunction"
      direction: "out"
  - skill: ontology
    action: cross_reference
    params:
      claim: "The elliptic curve 37.a1 has conductor 37 and rank 1"
```

### 4.2 From an Agent (Programmatic)

```python
# Skye or any agent with the skill installed
result = await ontology.lookup("lmfdb:37.a1")
print(f"Label: {result.label}")
print(f"Conductor: {result.properties['conductor']}")
print(f"Provenance: {result.provenance}")

# Cross-reference a claim
xref = await ontology.cross_reference(
    claim="The L-function of 37.a1 has conductor 37",
    context={"strictness": "high"}
)
if xref.verified:
    print("Claim verified against ontology data")
else:
    print(f"Contradictions: {xref.contradictions}")
    print(f"Unresolved: {xref.unresolved}")

# Bulk cross-reference for a multi-step proof
claims = [
    "The elliptic curve 37.a1 has conductor 37",
    "The elliptic curve 37.a1 has rank 1",
    "The L-function of 37.a1 has analytic rank 1"
]
results = await ontology.bulk_cross_reference(claims, context={"strictness": "high"})
# results is a dict keyed by claim hash
for claim_hash, result in results.items():
    print(f"Claim hash {claim_hash[:8]}...: verified={result.verified}, confidence={result.confidence}")
```

### 4.3 From Arx (Proof Audit Context)

Arx uses the skill during proof audit. The integration is deeper than other agents — Arx calls `cross_reference` or `bulk_cross_reference` for each proof step and includes the results in the provenance chain.

```python
# Inside Arx's verification pipeline (R4)
step_claim = "By the Modularity Theorem, the L-function of 37.a1 is modular"
xref = await ontology.cross_reference(
    claim=step_claim,
    context={
        "proof_id": "proof-2026-07-08-001",
        "step_id": "step-4",
        "profile": "auditor",
        "strictness": "high"
    }
)
# xref.verified = True if the ontology confirms the claim
# xref.evidence = [LMFDB record for 37.a1, MathGLOSS record for Modularity Theorem]
# xref.fallback_used = True if LLM fallback was needed
```

---

## 5. Arx-Specific Extensions

### 5.1 Ontology-Aware Proof Decomposition

When Arx decomposes a proof (R3), it calls `ontology_search` and `ontology_lookup` to:

1. **Resolve terminology:** "elliptic curve 37.a1" → canonical LMFDB object
2. **Identify theorems:** "Modularity Theorem" → MathGLOSS concept with QID
3. **Classify domain:** "number theory" → MathGLOSS field classification
4. **Check structure:** "X is a modular form" → verify against ontology

### 5.2 Ontology-Backed Verification

When Arx verifies a step (R4), it calls `ontology_cross_reference` or `ontology_bulk_cross_reference` to:

1. **Check computational claims:** "conductor 37" → verify against LMFDB
2. **Check structural claims:** "X is an elliptic curve" → verify against ontology
3. **Check relationship claims:** "X is isogenous to Y" → verify against LMFDB
4. **Flag contradictions:** "X has rank 2" but LMFDB says rank 1 → flag as `proven-with-contradiction` (if formally proven) or `provenance-broken` (if not)

### 5.3 Ontology in Provenance Chains

Every ontology query made during audit is recorded in the step's provenance chain:

```json
{
  "step_id": "step-4",
  "status": "proven",
  "provenance": {
    "backends_used": ["lean", "ontology"],
    "ontology_queries": [
      {
        "type": "cross_reference",
        "claim": "The elliptic curve 37.a1 has conductor 37",
        "claim_hash": "a1b2c3d4e5f6...",
        "result": { "verified": true, "confidence": 1.0 },
        "ontology_version": 12,
        "timestamp": "2026-07-08T15:30:00Z",
        "fallback_used": false
      }
    ]
  }
}
```

---

## 6. Implementation Plan

### Phase 0 — Foundation (W1)

- [ ] Create skill package structure (`neurocore-skill-ontology/`)
- [ ] Set up `pyproject.toml` with entry point
- [ ] Implement `SkillMeta` and base class
- [ ] Implement HTTP client for Master Ontology API
- [ ] Implement error handling (structured responses, not exceptions)

### Phase 1 — Core Query Types (W2)

- [ ] Implement `ontology_lookup`
- [ ] Implement `ontology_lookup_by_source`
- [ ] Implement `ontology_search`
- [ ] Implement `ontology_traverse`
- [ ] Implement `ontology_version_info`
- [ ] Write unit tests

### Phase 2 — Advanced Query Types (W3)

- [ ] Implement `ontology_semantic_search`
- [ ] Implement `ontology_path`
- [ ] Write unit tests

### Phase 3 — Cross-Reference (W4)

- [ ] Implement `ontology_cross_reference`
- [ ] Implement two-tier claim parsing (hand-written extractors + LLM fallback)
- [ ] Implement fallback rate tracking (per-audit)
- [ ] Implement verification logic (check claim against ontology data)
- [ ] Implement contradiction detection
- [ ] Implement `ontology_bulk_cross_reference` with hash-keyed results and partial failure handling
- [ ] Write unit tests

### Phase 4 — Arx Integration (W5)

- [ ] Integrate into Arx's proof decomposition (R3)
- [ ] Integrate into Arx's step verification (R4)
- [ ] Integrate into Arx's provenance chain (J-Scope)
- [ ] Test end-to-end with a real proof

### Phase 5 — Documentation & Release (W6)

- [ ] Write agent-facing documentation
- [ ] Write example blueprints
- [ ] Write integration tests
- [ ] Release as installable package

---

## 7. Acceptance Gates

| Gate | Criterion |
|---|---|
| G1 | All core query types return correct results against a running Master Ontology |
| G2 | `ontology_cross_reference` correctly verifies a known-true claim and flags a known-false claim |
| G3 | `ontology_bulk_cross_reference` returns results as a dict keyed by claim hash, not a positional list |
| G4 | Two-tier claim parsing: extractor rate ≥ 70% on a test suite of 100 number theory claims |
| G5 | Fallback rate is tracked per-audit and returned in cross-reference results |
| G6 | Error handling: ontology unreachable returns structured error response, not a raised exception |
| G7 | Arx integration: a proof audit with ontology cross-reference produces a complete provenance chain |
| G8 | Performance: lookup < 200ms (including network), search < 1s, cross-reference < 3s, bulk cross-reference < 5s for 10 claims |
| G9 | Partial failure: bulk cross-reference with one failing claim still returns results for the other claims |

---

## 8. Open Questions

1. **Claim parsing for cross-reference.** The two-tier approach (extractors → LLM fallback) is the right design, but the initial extractor set needs to cover the most common claim types in number theory proofs. We'll build the extractor set iteratively based on real proof data.

2. **Cross-reference confidence calibration.** How do we validate that the confidence formula produces well-calibrated scores? We need a test suite of known-true and known-false claims with ontology data.

3. **Version-aware cross-reference.** If a claim was verified against ontology version 5, and the ontology has since been updated to version 12, is the claim still valid? The skill should support re-verification against the latest version (handled by the re-audit trigger in Arx, not the skill itself).
