# Master Ontology — Design v0.1

**Status:** Final (incorporating independent review feedback)  
**Date:** 2026-07-08  
**Project root:** `/home/ubuntu/arx/`  
**Related documents:**
  - [ARX_ARCHITECTURE_v0.1.md](ARX_ARCHITECTURE_v0.1.md) — Arx agent architecture (§3.9)
  - [NEUROCORE_SKILL_ONTOLOGY_v0.1.md](NEUROCORE_SKILL_ONTOLOGY_v0.1.md) — Neurocore skill for ontology access
  - AXIOMA v1.9.1 / v2.0 — substrate and cognition layer
  - NeuroCore v0.3.0 — framework for skill-based agent capabilities

---

## 0. Executive Summary

The Master Ontology is a **unified, reconciled, versioned graph** that merges mathematical knowledge from three distinct sources:

1. **LMFDB** — verified computational data on L-functions, modular forms, elliptic curves, number fields, and related objects (confidence 1.0)
2. **MathGLOSS** — a Wikidata-aligned glossary of mathematical terms with subclass and relationship hierarchies (confidence 0.8)
3. **Our Ontology** — our own conceptual structure: the geometry↔energy↔information framework, consciousness measures, proof audit concepts, and the Riemann Hypothesis research program (confidence 0.6)

The Master Ontology serves two purposes:

- **System-level (Neurocore skill):** Any agent (Skye, Thea, Theoria, Axioma, Arx) can query the ontology to resolve mathematical terms, look up object properties, and traverse structural relationships
- **Arx-specific (proof audit):** Arx uses the ontology to classify proof structures, verify cross-references against LMFDB data, and detect inconsistencies between theoretical claims and known mathematical relationships

The ontology is stored as a **property graph** (SQLite + Qdrant for v0.1, Neo4j when needed), with every node and edge carrying **provenance metadata** indicating which source(s) contributed it and at what confidence level.

---

## 1. Design Principles

| # | Principle | What it constrains |
|---|---|---|
| O1 | **Provenance is first-class.** | Every node and edge carries a `provenance` field: a list of `(source, source_id, confidence, timestamp)` tuples. No anonymous data. |
| O2 | **Sources are independent, the merge is additive.** | No source is ever modified by the merge. Conflicts are resolved by explicit policy (see §4), never by overwriting source data. |
| O3 | **The ontology is queryable, not just storable.** | The ontology must support graph traversal (subclass, instance-of, depends-on), property lookup, and semantic similarity search. |
| O4 | **Versioned and auditable.** | Every merge operation produces a new version. The full history is preserved. Any agent can ask "what did the ontology say about X at time T?" |
| O5 | **Extensible by design.** | New sources can be added without modifying the merge logic. Each source provides a conformant adapter. |
| O6 | **Confidence is explicit, not hidden.** | A claim from LMFDB (verified computational data) has confidence 1.0. A claim from MathGLOSS (Wikidata alignment) has confidence 0.8. A claim from our ontology (theoretical) has confidence 0.6. These are defaults, adjustable per-claim. |
| O7 | **The ontology serves agents, not humans directly.** | The primary interface is programmatic (Neurocore skill API). Human-readable views are secondary. |
| O8 | **Equivalence is explicit, not inferred.** | Cross-source concept equivalence is asserted via versioned, auditable mapping rules. No automatic alignment. Mapping confidence is capped at the minimum confidence of the two source objects it connects. |
| O9 | **Storage is abstracted.** | The query interface is designed as an abstraction so the backend (SQLite, Neo4j, etc.) can be swapped without changing the skill or consuming agents. |

---

## 2. Source Ontologies

### 2.1 LMFDB (L-Functions and Modular Forms Database)

**Nature:** Verified computational database of number-theoretic objects.
**Access:** REST API (JSON/YAML), MCP server available (announced April 2026, confirmed on lmfdb.org).
**Coverage:** The LMFDB contains objects across dozens of categories. Key counts (as of 2026):

| Category | Count | Notes |
|----------|-------|-------|
| Elliptic curves over ℚ | 3,824,372 | In 2,917,287 isogeny classes, conductor up to ~300 million |
| Elliptic curves over number fields | 767,518 | In 367,933 isogeny classes, over 894 number fields |
| Number fields | ~2 million | Including Galois groups, class groups, regulators |
| Modular forms (classical) | ~1 million | Levels up to ~10,000 |
| L-functions | ~5 million | Including those attached to the above objects |
| Artin representations | ~100,000 | Dimensions 1–4 |
| Dirichlet characters | ~500,000 | Moduli up to ~100,000 |
| Sato-Tate groups | ~100 | All 52 Sato-Tate groups for genus 1 |
| Abstract groups | ~500,000 | Up to order ~10^7 |
| Genus 2 curves | ~100,000 | Over ℚ |
| Abelian varieties over finite fields | ~100,000 | |

Total object count is in the **low millions** across all categories.

**What we extract:**

| Entity Type | Properties | Relationships |
|---|---|---|
| `LFunction` | conductor, degree, sign, Euler factors, functional equation, zeros | `is_lfunction_of` → EllipticCurve, ModularForm, NumberField, ArtinRepresentation |
| `EllipticCurve` | conductor, rank, torsion, L-function label, isogeny class | `has_lfunction` → LFunction, `is_isogenous_to` → EllipticCurve |
| `ModularForm` | level, weight, character, dimension, Hecke eigenvalues | `has_lfunction` → LFunction, `is_of_level` → Integer |
| `NumberField` | degree, discriminant, Galois group, signature | `has_lfunction` → LFunction (Dedekind zeta), `has_subfield` → NumberField |
| `ArtinRepresentation` | dimension, conductor, Frobenius traces | `has_lfunction` → LFunction |
| `DirichletCharacter` | modulus, conductor, order | `is_primitive` → DirichletCharacter |
| `SatoTateGroup` | name, measure, moments | `is_sato_tate_of` → EllipticCurve |

**Confidence baseline:** 1.0 (verified computational data, peer-reviewed).

**Ingestion strategy:**
- **On-demand lookup** (primary): Query-by-label during proof audit, cache results locally
- **No full mirror.** The LMFDB is too large to mirror in v0.1. Cache referenced objects over time.
- **MCP server** (preferred): Use the MCP server if available. Fall back to REST API.

### 2.2 MathGLOSS (Mathematical Global Ontology of Scientific Structures)

**Nature:** Wikidata-aligned glossary of mathematical terms with relationship extraction.
**Access:** GitHub repository (pipeline toolkit), Neo4j export available, REST API possible.
**Coverage:** Undergraduate to graduate-level mathematics; subclass and extends-hierarchy extraction from textbooks, nLab, Mathlib, Wikipedia.

**What we extract:**

| Entity Type | Properties | Relationships |
|---|---|---|
| `MathConcept` | name, Wikidata QID, description, aliases | `subclass_of` → MathConcept, `related_to` → MathConcept |
| `MathStructure` | name, QID, definition, field | `has_substructure` → MathStructure, `generalizes` → MathStructure |
| `MathTheorem` | name, QID, statement, field | `involves` → MathConcept, `proved_by` → MathTheorem |
| `MathField` | name, QID, description | `contains` → MathConcept |

**Confidence baseline:** 0.8 (Wikidata alignment is generally reliable but not verified).

**Ingestion strategy:**
- Run the MathGLOSS pipeline to produce a Neo4j dump
- Import into the Master Ontology graph (SQLite)
- Re-run periodically to pick up updates

**Coverage assessment (Phase 0.5):**
Before full integration, assess MathGLOSS coverage of:
- Advanced number theory (L-functions, modular forms, automorphic representations)
- Algebraic geometry (elliptic curves, abelian varieties, motives)
- Category theory and higher structures (for geometry↔energy↔information framework)
- Proof theory and formal verification concepts

**Supplementation sources (if gaps found):**
- **nLab** — Excellent coverage of advanced category theory, homotopy theory, higher structures
- **mathlib docs** — Formalized theorem statements with precise type signatures
- **LMFDB's own taxonomy** — Well-organized classification of L-functions, modular forms, Sato-Tate groups
- **Wikipedia's math portal** — Broad coverage, lower confidence, use as fallback only

### 2.3 Our Ontology

**Nature:** Our own conceptual structure — the geometry↔energy↔information framework, consciousness measures, proof audit concepts, RH research program.
**Source:** Hand-authored YAML files, version-controlled alongside research documents.

**Write access:**
- **Lark:** Approves ontology structure changes (new categories, new cross-source mappings)
- **Skye, Thea, Theoria, Axioma, Arx:** Can add individual concept additions within existing categories
- **All additions require:** A provenance statement (why this concept exists, what source it came from, what confidence we assign)

**What we include:**

| Entity Type | Properties | Relationships |
|---|---|---|
| `ResearchConcept` | name, description, domain, status | `specializes` → ResearchConcept, `contradicts` → ResearchConcept |
| `ConsciousnessMeasure` | name, symbol, definition, range | `measures` → ResearchConcept, `related_to` → ConsciousnessMeasure |
| `ProofConcept` | name, definition, audit status | `used_in` → ProofStep, `depends_on` → ProofConcept |
| `RHConcept` | name, description, approach | `related_to` → LFunction, `approach` → RHApproach |
| `RHApproach` | name, description, status, key_insight | `uses` → ResearchConcept, `references` → LFunction |
| `GeometryEnergyInfo` | name, description, mapping | `maps_from` → ResearchConcept, `maps_to` → ResearchConcept |

**Confidence baseline:** 0.6 (theoretical, subject to revision).

**Ingestion strategy:**
- Define a YAML schema for our ontology entries
- Write a Python script that reads the YAML and produces graph nodes/edges
- Version-controlled alongside the research documents
- PR-based workflow for additions and modifications

**Schema:**
```yaml
concepts:
  - id: "our:consciousness-phi"
    label: "Φ (Integrated Information)"
    description: "Tononi et al.'s measure of integrated information"
    domain: consciousness
    confidence: 0.6
    relationships:
      - type: measures
        target: "our:consciousness"
      - type: related_to
        target: "our:psi-integration"
    provenance:
      source: "our_ontology"
      author: "skye"
      timestamp: "2026-07-08"
```

---

## 3. Data Model

### 3.1 Node Types

Every node in the Master Ontology has:

```
{
  "id": "uuid",                    // canonical, stable identifier
  "type": "LFunction | EllipticCurve | MathConcept | ResearchConcept | ...",
  "source_type": "lmfdb | mathgloss | our_ontology",
  "source_id": "string",          // original ID in the source (e.g., LMFDB label)
  "label": "string",              // human-readable name
  "properties": { ... },          // type-specific properties
  "provenance": [
    {
      "source": "lmfdb",
      "source_id": "37.a1",
      "confidence": 1.0,
      "ingested_at": "2026-07-08T12:00:00Z"
    }
  ],
  "created_at": "ISO datetime",
  "updated_at": "ISO datetime",
  "version": 1
}
```

### 3.2 Edge Types

Every edge has:

```
{
  "id": "uuid",
  "type": "has_lfunction | subclass_of | specializes | depends_on | ...",
  "source_node_id": "uuid",
  "target_node_id": "uuid",
  "properties": { ... },          // edge-specific properties
  "provenance": [ ... ],          // same structure as node provenance
  "created_at": "ISO datetime",
  "version": 1
}
```

### 3.3 Canonical Edge Types

| Edge Type | Source → Target | Description | Source |
|---|---|---|---|
| `has_lfunction` | EllipticCurve → LFunction | The object's associated L-function | LMFDB |
| `is_isogenous_to` | EllipticCurve → EllipticCurve | Isogeny relationship | LMFDB |
| `subclass_of` | MathConcept → MathConcept | Taxonomic hierarchy | MathGLOSS |
| `generalizes` | MathStructure → MathStructure | Abstract → concrete | MathGLOSS |
| `involves` | MathTheorem → MathConcept | Theorem mentions concept | MathGLOSS |
| `specializes` | ResearchConcept → ResearchConcept | Our conceptual hierarchy | Our Ontology |
| `contradicts` | ResearchConcept → ResearchConcept | Known tension | Our Ontology |
| `depends_on` | ProofConcept → ProofConcept | Logical dependency | Our Ontology |
| `references` | RHApproach → LFunction | Approach uses L-function data | Our Ontology |
| `equivalent_to` | Any → Any | Cross-source equivalence | Merge process |
| `related_to` | Any → Any | Semantic relatedness | Any source |

### 3.4 Cross-Source Equivalence

When the merge process determines that two nodes from different sources refer to the same real-world entity, it creates an `equivalent_to` edge between them. For example:

- An LMFDB `EllipticCurve` with label `37.a1` may be equivalent to a MathGLOSS `MathConcept` with QID `Q12345` (the concept of "elliptic curve of conductor 37")
- A MathGLOSS `MathConcept` "L-function" may be equivalent to our `ResearchConcept` "L-function"

Equivalence is **never asserted automatically** without a clear mapping rule. The initial set of equivalence rules is hand-authored.

Mapping confidence is capped at `min(conf_a, conf_b)` where `conf_a` and `conf_b` are the confidences of the two source objects. This prevents a mapping from being more reliable than its least reliable endpoint.

These mappings are themselves auditable artifacts. If a mapping is wrong, it propagates errors through the entire ontology.

### 3.5 Epistemic Isolation and Sub-graph Filtering

To prevent speculative concepts (e.g., consciousness measures, baseline confidence 0.6) from polluting rigorous mathematical proof auditing, the Master Ontology implements **strict sub-graph isolation and path traversal filters**.

1. **Sub-graph Partitioning:**
   Nodes are partitioned into logical sub-graphs based on their `source_type` and an explicit `domain` parameter:
   * **Verification Sub-graph (`VERIFY`):** Contains nodes from `lmfdb` and `mathgloss`, plus any `our_ontology` nodes explicitly marked with domain `proof_theory` or `mathematics` (where confidence ≥ 0.8).
   * **Research Sub-graph (`RESEARCH`):** Contains speculative research nodes from `our_ontology` (e.g., `geometry_energy_info`, `consciousness`, where confidence < 0.8).

2. **Traversal Path Filters:**
   When performing graph traversals (`traverse` or `path` queries) during proof auditing or status verification:
   * The query defaults to the `VERIFY` sub-graph.
   * Path traversal is restricted to nodes and edges with `confidence >= threshold` (default threshold is 0.8 for the auditor/verifier profiles).
   * Any path that attempts to cross an `equivalent_to` or `related_to` edge pointing to a node in the `RESEARCH` sub-graph is blocked.
   * Speculative nodes are completely pruned from the search space during formal auditing, ensuring the agent's logic is grounded only in verified mathematical facts.

---

## 4. Merge Strategy

### 4.1 Ingestion Pipeline

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│   LMFDB     │   │  MathGLOSS  │   │Our Ontology │
│   Adapter   │   │   Adapter   │   │   Adapter   │
└──────┬──────┘   └──────┬──────┘   └──────┬──────┘
       │                 │                 │
       ▼                 ▼                 ▼
┌─────────────────────────────────────────────────┐
│           Staging Area (normalized JSON)         │
│  - All nodes in canonical format                │
│  - All edges in canonical format                │
│  - Provenance attached                          │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│           Reconciliation Engine                   │
│  - Deduplication (by equivalence rules)           │
│  - Conflict resolution (by policy)               │
│  - Cross-source edge generation                  │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│           Graph Database (SQLite + Qdrant)        │
│  - Versioned snapshots                           │
│  - Full provenance on every node/edge           │
│  - Indexed for fast traversal                    │
└─────────────────────────────────────────────────┘
```

### 4.2 Reconciliation Rules

**Rule 1: Same-source deduplication.**
If two nodes from the same source have the same `source_id`, they are the same node. Merge properties, keeping the most recent timestamp.

**Rule 2: Cross-source equivalence by explicit mapping.**
Equivalence is only asserted when a mapping rule exists. Initial mapping rules:

| Source A | Source B | Mapping Rule |
|---|---|---|
| LMFDB `EllipticCurve` (label) | MathGLOSS `MathConcept` (QID) | Hand-curated list of label↔QID pairs |
| LMFDB `LFunction` (label) | MathGLOSS `MathConcept` (QID) | Hand-curated list |
| MathGLOSS `MathConcept` (QID) | Our `ResearchConcept` (name) | Name-based matching with manual review |

**Rule 3: Property conflict resolution.**
When two sources provide different values for the same property of the same entity:

| Scenario | Resolution |
|---|---|
| One source has the property, the other doesn't | Accept the provided value |
| Both sources have the same value | Accept (no conflict) |
| Both sources have different values | Keep both as `proposed_value` with provenance; mark as `conflict`; flag for human review |
| LMFDB vs any other source | LMFDB wins (verified data) |

**Rule 4: Edge conflict resolution.**
Same as property conflicts, with the addition that edges from LMFDB are considered ground truth for computational relationships.

### 4.3 Versioning

Every merge operation produces a new **ontology version**. The version is a **content-addressable hash** of the ontology state, not a sequential number:

```python
ontology_version = hash(concat(
    hash(all_nodes),
    hash(all_edges),
    hash(all_equivalence_mappings),
    hash(previous_version)
))
```

The graph database stores:
- A `version` property on every node and edge
- A `version_log` collection mapping version numbers to: timestamp, sources ingested, reconciliation rules applied, number of nodes/edges added/modified

Querying the ontology at a specific version is supported: `SELECT ... WHERE version <= $version`.

---

## 5. Storage

### 5.1 Primary Store: SQLite (v0.1)

**Rationale:** SQLite handles graph queries via recursive CTEs for the scale we expect in v0.1 (hundreds to low thousands of nodes). No additional infrastructure to maintain.

**Schema:**
```sql
CREATE TABLE nodes (
    id TEXT PRIMARY KEY,
    type TEXT NOT NULL,
    source_type TEXT NOT NULL,
    source_id TEXT NOT NULL,
    label TEXT NOT NULL,
    properties TEXT,  -- JSON
    provenance TEXT,  -- JSON array
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    UNIQUE(source_type, source_id)
);

CREATE TABLE edges (
    id TEXT PRIMARY KEY,
    type TEXT NOT NULL,
    source_node_id TEXT NOT NULL REFERENCES nodes(id),
    target_node_id TEXT NOT NULL REFERENCES nodes(id),
    properties TEXT,  -- JSON
    provenance TEXT,  -- JSON array
    created_at TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE equivalence_mappings (
    id TEXT PRIMARY KEY,
    source_a TEXT NOT NULL,
    object_a TEXT NOT NULL,
    source_b TEXT NOT NULL,
    object_b TEXT NOT NULL,
    confidence REAL NOT NULL,
    rule TEXT NOT NULL,
    verified_by TEXT NOT NULL,
    ontology_version INTEGER NOT NULL
);

CREATE TABLE version_log (
    version INTEGER PRIMARY KEY,
    hash TEXT NOT NULL,
    created_at TEXT NOT NULL,
    description TEXT,
    sources_ingested TEXT,  -- JSON array
    node_count INTEGER,
    edge_count INTEGER
);

-- Indexes
CREATE INDEX idx_nodes_type ON nodes(type);
CREATE INDEX idx_nodes_source ON nodes(source_type, source_id);
CREATE INDEX idx_nodes_label ON nodes(label);
CREATE INDEX idx_edges_type ON edges(type);
CREATE INDEX idx_edges_source ON edges(source_node_id);
CREATE INDEX idx_edges_target ON edges(target_node_id);
```

**Graph queries via recursive CTEs:**
```sql
-- Traverse edges with sub-graph isolation and confidence threshold filtering
WITH RECURSIVE traverse(id, depth) AS (
    SELECT source_node_id, 0 FROM edges 
    JOIN nodes n ON source_node_id = n.id
    WHERE source_node_id = ? 
      AND n.source_type IN (?, ?)  -- restrict to allowed sub-graphs (e.g. lmfdb, mathgloss)
    
    UNION ALL
    
    SELECT e.target_node_id, t.depth + 1
    FROM edges e
    JOIN nodes n ON e.target_node_id = n.id
    JOIN traverse t ON e.source_node_id = t.id
    WHERE t.depth < ? 
      AND e.confidence >= ?  -- minimum confidence threshold (e.g. 0.8)
      AND n.source_type IN (?, ?)  -- block traversal into speculative source sub-graphs
)
SELECT * FROM nodes WHERE id IN (SELECT id FROM traverse);

-- Find paths with sub-graph isolation and confidence threshold filtering
WITH RECURSIVE paths(path, last, depth) AS (
    SELECT source_node_id || '->' || target_node_id, target_node_id, 1 FROM edges 
    JOIN nodes n ON target_node_id = n.id
    WHERE source_node_id = ? 
      AND n.source_type IN (?, ?)  -- restrict target to allowed sub-graphs
    
    UNION ALL
    
    SELECT p.path || '->' || e.target_node_id, e.target_node_id, p.depth + 1
    FROM edges e
    JOIN nodes n ON e.target_node_id = n.id
    JOIN paths p ON e.source_node_id = p.last
    WHERE p.depth < ? 
      AND e.target_node_id != ?
      AND e.confidence >= ?  -- minimum confidence threshold
      AND n.source_type IN (?, ?)  -- block traversal into speculative source sub-graphs
)
SELECT path FROM paths WHERE last = ?;
```

### 5.2 Secondary Store: Qdrant (vector index)

**Rationale:** Semantic similarity search across concept descriptions. When an agent asks "find concepts related to X," we can do both graph traversal and vector similarity.

**Schema:**
- Collection: `ontology_concepts`
- Vector: embedding of `label + description` (using a math-aware embedding model)
- Payload: `id`, `type`, `label`, `source_type`

### 5.3 Cache Layer: SQLite (built-in)

**Rationale:** No need for a separate Redis instance. The LMFDB query cache lives in the same SQLite database.

**Cache policy:**
- TTL: 24h for static data (conductor, rank, torsion), 1h for dynamic data
- Invalidation: on new merge version

### 5.4 Future: Neo4j

When the node/edge count exceeds SQLite's performance ceiling (estimated: ~5000 nodes, ~20000 edges), migrate to Neo4j. The abstract `OntologyGraph` interface makes this a drop-in replacement:

```python
class OntologyGraph:
    def query(self, cypher_or_sql: str, params: dict) -> list[dict]: ...
    def traverse(self, node_id: str, edge_type: str, direction: str) -> list[Edge]: ...
    def path(self, start: str, end: str, max_depth: int) -> list[list[Edge]]: ...

class SQLiteOntologyGraph(OntologyGraph): ...
class Neo4jOntologyGraph(OntologyGraph): ...
```

---

## 6. Query Interface

### 6.1 Core Query Types

The Master Ontology supports these query types:

| Query Type | Description | Example |
|---|---|---|
| `lookup(id)` | Get a node by its canonical ID | `lookup("lmfdb:37.a1")` |
| `lookup_by_source(source, source_id)` | Get a node by its original source ID | `lookup_by_source("lmfdb", "37.a1")` |
| `search(label_pattern)` | Full-text search on labels | `search("elliptic curve")` |
| `traverse(node_id, edge_type, direction)` | Follow edges of a given type | `traverse("lmfdb:37.a1", "has_lfunction", OUT)` |
| `path(start_id, end_id, max_depth)` | Find paths between two nodes | `path("lmfdb:37.a1", "mathgloss:Q12345", 3)` |
| `neighbors(node_id, max_depth)` | Get all nodes within N hops | `neighbors("lmfdb:37.a1", 2)` |
| `semantic_search(text, limit)` | Vector similarity search | `semantic_search("modular forms with complex multiplication", 10)` |
| `version_query(version)` | Query the ontology at a specific version | `version_query(5)` |
| `cross_reference(claim, context)` | Verify a claim against ontology data | `cross_reference("37.a1 has conductor 37", {"strictness": "high"})` |
| `bulk_cross_reference(claims, context)` | Verify a list of claims in one call | `bulk_cross_reference([claim1, claim2, ...], ...)` |

### 6.2 Response Format

All queries return a standardized response:

```json
{
  "query": { "type": "lookup", "params": { "id": "lmfdb:37.a1" } },
  "version": 12,
  "results": [ ... nodes and/or edges ... ],
  "provenance_summary": {
    "sources_consulted": ["lmfdb", "mathgloss"],
    "confidence_range": [0.8, 1.0]
  },
  "query_time_ms": 45
}
```

### 6.3 Cross-Reference Confidence Formula

```python
confidence = max(
    min(evidence_confidence) * source_weight * specificity_factor,
    ontology_confidence
)
```

Where:

| Parameter | Values | Description |
|-----------|--------|-------------|
| `evidence_confidence` | 0.0–1.0 | Confidence of each piece of evidence supporting the claim |
| `source_weight` | 1.0 (LMFDB), 0.9 (MathGLOSS), 0.7 (our ontology) | Weight of the source providing the evidence |
| `specificity_factor` | 1.0 (direct match), 0.9 (direct property), 0.7 (subclass inference), 0.5 (structural analogy), 0.4 (definitional overlap) | How specific the match is |
| `ontology_confidence` | 0.0–1.0 | Baseline confidence from the ontology itself |

---

## 7. Integration with Arx

### 7.1 Ontology-Aware Audit Pipeline

Arx uses the Master Ontology at three points in the audit pipeline:

**1. Proof Decomposition (R3):**
Before decomposing a proof into steps, Arx queries the ontology to:
- Resolve mathematical terms to canonical concepts (via MathGLOSS)
- Identify known theorems and lemmas referenced in the proof
- Classify the proof's mathematical domain

**2. Step Verification (R4):**
For each step, Arx queries the ontology to:
- Cross-reference computational claims against LMFDB data
- Verify that structure classifications are consistent with known relationships
- Check that referenced theorems exist and have the claimed properties

**3. Provenance Construction (J-Scope):**
The provenance chain for each step includes:
- Ontology queries made during verification
- The ontology version at the time of query
- Confidence scores from the ontology

### 7.2 Contradiction Feedback Loop

When Arx detects a contradiction (formal proof vs. ontology), the contradiction feeds back into the ontology:

1. A structured feedback record is generated:
   ```json
   {
     "contradiction_id": "uuid",
     "proof_step_id": "...",
     "formal_claim": "...",
     "ontology_claim": "...",
     "ontology_source": "lmfdb | mathgloss | our_ontology",
     "confidence": 0.95,
     "priority": "high | normal | low"
   }
   ```
2. The feedback record goes into an **ontology review queue**
3. Priority scheme:
   - **HIGH:** Contradictions involving formal proofs (most urgent)
   - **NORMAL:** Contradictions between two curated sources (LMFDB vs MathGLOSS)
   - **LOW:** Contradictions involving LLM-extracted data (least urgent)
4. A human (Lark, or one of us) reviews the queue and decides whether to update the ontology or flag the proof

### 7.3 Re-Audit Trigger

When ontology records change:
1. Arx checks which audit results reference the changed records
2. Affected steps receive a `stale` flag
3. A re-audit recommendation is generated: re-run cross-reference for stale steps, re-verify if cross-reference was the primary verification method
4. The re-audit is queued and run when the system is idle

---

## 8. Integration with Neurocore-Skill-Ontology

The Master Ontology is the **single source of truth** that the Neurocore skill wraps. The skill does not store its own copy — it queries the Master Ontology's API.

```
Agent (Skye, Thea, Theoria, Axioma, Arx)
    │
    ▼
Neurocore-Skill-Ontology (query interface)
    │
    ▼
Master Ontology API (REST, port TBD)
    │
    ├──► SQLite (graph queries)
    ├──► Qdrant (semantic search)
    └──► SQLite cache (LMFDB cache)
```

The skill provides:
- A unified query interface (all query types from §6.1)
- Result formatting (standardized JSON)
- Error handling (ontology unavailable, version mismatch, query timeout)
- Caching (agent-side, for repeated queries within a session)

---

## 9. Implementation Plan

### Phase 0 — Foundation (W1)

- [ ] Set up SQLite database with schema
- [ ] Set up Qdrant instance (Docker, port TBD)
- [ ] Define canonical node/edge schemas (Python dataclasses)
- [ ] Write abstract `OntologyGraph` interface
- [ ] Write `SQLiteOntologyGraph` implementation

### Phase 1 — Our Ontology Ingestion (W2)

- [ ] Define YAML schema for our ontology entries
- [ ] Write ingestion script (YAML → SQLite)
- [ ] Seed initial concepts: geometry↔energy↔information, consciousness measures, RH approaches, proof audit concepts
- [ ] Write `lookup`, `search`, `traverse` query functions

### Phase 2 — MathGLOSS Ingestion (W3)

- [ ] Run MathGLOSS pipeline to produce Neo4j dump
- [ ] Write MathGLOSS adapter (dump → canonical format)
- [ ] Import into Master Ontology
- [ ] Write equivalence mapping rules (MathGLOSS ↔ Our Ontology)
- [ ] Test cross-source queries

### Phase 3 — LMFDB Ingestion (W4)

- [ ] Write LMFDB adapter (REST API → canonical format)
- [ ] Implement on-demand lookup (query-by-label)
- [ ] Implement local cache (SQLite)
- [ ] Write equivalence mapping rules (LMFDB ↔ MathGLOSS)
- [ ] Test cross-source queries

### Phase 4 — Reconciliation Engine (W5)

- [ ] Implement deduplication (same-source)
- [ ] Implement equivalence detection (cross-source)
- [ ] Implement conflict resolution
- [ ] Implement versioning
- [ ] Write `version_query` function

### Phase 5 — Arx Integration (W6)

- [ ] Add `ontology` backend type to Arx compute kernel
- [ ] Integrate ontology queries into proof decomposition (R3)
- [ ] Integrate ontology queries into step verification (R4)
- [ ] Add ontology parameters to profile system
- [ ] Test end-to-end: audit a proof that references LMFDB objects

### Phase 6 — Neurocore Skill Integration (W7)

- [ ] Write Neurocore-Skill-Ontology (see separate design document)
- [ ] Test from Skye, Thea, Theoria, Axioma
- [ ] Write agent-facing documentation

---

## 10. Open Questions

1. **LMFDB MCP server details.** The MCP server was announced April 2026 and confirmed on lmfdb.org. We need to test its actual interface before committing to the integration strategy.

2. **MathGLOSS coverage depth.** The README says "mostly concerned with undergraduate mathematics." We need to assess whether it covers the advanced number theory and modular forms territory that the RH work needs.

3. **Neo4j vs SQLite threshold.** At what node/edge count does SQLite recursive CTE performance become unacceptable? Estimate: ~5000 nodes, ~20000 edges. We'll benchmark during Phase 0.5.

4. **Equivalence mapping scale.** The initial hand-curated mapping will be small (~100 entries). We need a strategy for scaling to thousands of mappings without manual effort.

5. **Cross-reference confidence calibration.** How do we validate that the confidence formula produces well-calibrated scores? We need a test suite of known-true and known-false claims with ontology data.

---

## 11. Acceptance Gates

| Gate | Criterion |
|---|---|
| G1 | Our ontology: 50+ concepts ingested, queryable via `lookup` and `search` |
| G2 | MathGLOSS: 1000+ concepts ingested, subclass hierarchy queryable |
| G3 | LMFDB: on-demand lookup works for elliptic curves, L-functions, modular forms |
| G4 | Cross-source query: "find all L-functions related to elliptic curves of conductor 37" returns results from both LMFDB and MathGLOSS |
| G5 | Versioning: query at version N returns different results than version N+1 after a merge |
| G6 | Arx integration: a proof referencing an LMFDB object is audited with ontology cross-reference in the provenance chain |
| G7 | Neurocore skill: all query types work from a test agent |
| G8 | Performance: lookup < 100ms, search < 500ms, traverse < 1s (for depth ≤ 3) |
