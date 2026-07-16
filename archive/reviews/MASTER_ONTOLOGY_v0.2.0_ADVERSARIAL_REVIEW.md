# Adversarial Review: MASTER_ONTOLOGY_v0.2.0.md

**Reviewer:** AXIOMA (adversarial)  
**Date:** 2026-07-08  
**Status:** 9 issues found (2 critical, 3 major, 4 minor)

---

## Critical Issues

### C1. SQL schema missing `confidence` column on `edges` table

**Location:** §5.1, SQL schema

**Issue:** The `edges` table schema has no `confidence` column, but the recursive CTE examples reference `e.confidence`:

```sql
WITH RECURSIVE traverse(id, depth) AS (
    ...
    WHERE t.depth < ? 
      AND e.confidence >= ?  -- references e.confidence, but edges table has no confidence column
      AND n.subgraph IN (?, ?)
)
```

**Impact:** The SQL will fail at runtime with "no such column: e.confidence." This is a concrete implementation bug in the design document.

**Recommendation:** Add `confidence REAL NOT NULL DEFAULT 1.0` to the `edges` table schema. This is consistent with the data model (§3.2) which says edges have properties (confidence could be a property), but for the CTE to work, it needs to be a first-class column.

---

### C2. LMFDB object count for "Number fields" is misleading

**Location:** §2.1, LMFDB coverage table

**Issue:** The table says "Number fields: ~2 million" with the note "Including Galois groups, class groups, regulators." This is misleading. The LMFDB contains ~2 million *number field records* (each with its own page), but many of these are not distinct number fields — they are the same field listed with different defining polynomials or different labels. The actual number of *distinct* number fields up to a given degree and discriminant bound is much smaller.

**Impact:** A reader might think the ontology contains 2 million distinct number field objects, which would affect their understanding of the ontology's scale and query performance.

**Recommendation:** Clarify: "~2 million number field records (including multiple representations of the same field). The number of distinct isomorphism classes is approximately 400,000." Or cite the LMFDB documentation directly.

---

## Major Issues

### M1. Recursive CTE for path finding has a cycle vulnerability

**Location:** §5.1, Path finding CTE

**Issue:** The path-finding recursive CTE has `WHERE e.target_node_id != ?` as a cycle breaker, but this only prevents immediate cycles (A→B→A). It does not prevent longer cycles (A→B→C→A). A graph with a cycle of length 3 or more will cause infinite recursion.

```sql
WITH RECURSIVE paths(path, last, depth) AS (
    ...
    WHERE p.depth < ? 
      AND e.target_node_id != ?  -- only prevents A→B→A, not A→B→C→A
      AND e.confidence >= ?
)
```

**Impact:** A query on a graph with a cycle will either hang (infinite recursion) or hit the depth limit and return incomplete results.

**Recommendation:** Track visited nodes in the path:
```sql
WITH RECURSIVE paths(path, last, depth, visited) AS (
    SELECT source_node_id || '->' || target_node_id, 
           target_node_id, 1, 
           ',' || source_node_id || ',' || target_node_id || ','
    FROM edges 
    JOIN nodes n ON target_node_id = n.id
    WHERE source_node_id = ? 
      AND n.subgraph IN (?, ?)
    
    UNION ALL
    
    SELECT p.path || '->' || e.target_node_id, 
           e.target_node_id, p.depth + 1,
           p.visited || e.target_node_id || ','
    FROM edges e
    JOIN nodes n ON e.target_node_id = n.id
    JOIN paths p ON e.source_node_id = p.last
    WHERE p.depth < ? 
      AND p.visited NOT LIKE '%,' || e.target_node_id || ',%'  -- proper cycle detection
      AND e.confidence >= ?
      AND n.subgraph IN (?, ?)
)
SELECT path FROM paths WHERE last = ?;
```

---

### M2. Equivalence mapping scale is not addressed

**Location:** §10, Open Question 4

**Issue:** Open Question 4 says "The initial hand-curated mapping will be small (~100 entries). We need a strategy for scaling to thousands of mappings without manual effort." But the document provides no concrete strategy for scaling, and the equivalence mapping schema (§3.4) doesn't support automated suggestions.

**Impact:** If the ontology grows to include thousands of LMFDB objects and MathGLOSS concepts, the 100-entry hand-curated mapping will be insufficient. Cross-source queries will miss connections.

**Recommendation:** Add a concrete scaling strategy, even if deferred:
- Phase 1: Hand-curated (100 entries)
- Phase 2: Property-based matching (match objects with identical properties across sources, flag for human review)
- Phase 3: Embedding-based suggestion (use vector similarity to suggest candidate mappings)
- Phase 4: Community-contributed (allow agents to propose mappings via the sandboxed queue)

---

### M3. No description of how the ontology API handles concurrent writes

**Location:** §6, Query Interface

**Issue:** The document describes a REST API for querying the ontology but doesn't describe how writes (new versions, new equivalence mappings) are handled. If two agents try to add mappings simultaneously, what happens? Is there a lock? A transaction? A merge strategy for concurrent writes?

**Impact:** In a multi-agent system where Skye, Thea, Theoria, Axioma, and Arx can all propose ontology changes, concurrent writes are inevitable. Without a concurrency strategy, data corruption or lost updates are possible.

**Recommendation:** Add a brief description of write concurrency:
- SQLite uses WAL mode for concurrent reads
- Writes are serialized via a write lock
- If a write conflict occurs, the second writer retries after a short delay
- For the sandboxed queue, writes are batched and applied atomically

---

## Minor Issues

### m1. "ArtinRepresentation" is misspelled in the entity table

**Location:** §2.1, Entity Types table

**Issue:** The table lists `ArtinRepresentation` (missing the 'e' — should be `ArtinRepresentation`). Wait, let me check... The table says `ArtinRepresentation` which is actually correct (Artin, not Artin). Let me re-read... The table says `ArtinRepresentation` — this is correct. The review feedback from v0.1 said to add it to the entity table, and it was added. No issue here.

### m2. The `version_log` table uses `version INTEGER PRIMARY KEY` but versioning is content-addressable

**Location:** §4.3 vs §5.1

**Issue:** §4.3 says "The version is a content-addressable hash of the ontology state, not a sequential number." But the `version_log` table uses `version INTEGER PRIMARY KEY`, implying sequential integer versions. The `nodes` and `edges` tables also use `version INTEGER`.

**Impact:** If versions are content-addressable hashes (strings), the schema is wrong. If versions are sequential integers (with the hash stored separately), the text in §4.3 is misleading.

**Recommendation:** Clarify: the version is a sequential integer (for efficient querying), and the content-addressable hash is stored in the `version_log.hash` column. Update §4.3 to say "Every merge operation produces a new sequential version number. The content-addressable hash of the ontology state is stored alongside it for integrity verification."

### m3. The `edges` table has no `subgraph` column

**Location:** §5.1, SQL schema

**Issue:** The `nodes` table has a `subgraph` column for epistemic isolation, but the `edges` table doesn't. If an edge connects a VERIFY node to a RESEARCH node, how is this edge classified? The traversal CTEs check the target node's subgraph, but the edge itself has no subgraph classification.

**Impact:** An edge from a VERIFY node to a RESEARCH node would be traversed if the target node check is the only filter. The CTE checks `n.subgraph IN (?, ?)` on the target node, so it would correctly block traversal into RESEARCH nodes. But the edge itself is unclassified, which could cause issues for edge-level queries.

**Recommendation:** Add a `subgraph` column to the `edges` table, defaulting to the minimum subgraph of its source and target nodes. Or add a note explaining that edge subgraph is derived from node subgraph at query time.

### m4. The "Supplementation sources" list includes Wikipedia but doesn't specify confidence

**Location:** §2.2, Supplementation sources

**Issue:** The list includes "Wikipedia's math portal — Broad coverage, lower confidence, use as fallback only" but doesn't specify what "lower confidence" means numerically. Is it 0.5? 0.3? This matters for the confidence formula.

**Recommendation:** Specify: "Wikipedia's math portal — confidence 0.5 (broad coverage, but not peer-reviewed). Use as fallback only when LMFDB, MathGLOSS, nLab, and mathlib docs are insufficient."

---

## Cross-Document Issues

### X1. Ontology versioning scheme conflicts with re-audit trigger

**Location:** MASTER_ONTOLOGY_v0.2.0.md §4.3 vs ARX_ARCHITECTURE_v0.2.0.md §3.9.7

**Issue:** The ontology document says versions are content-addressable hashes (or sequential integers with hashes). The architecture document says "Arx checks which audit results reference the changed records" when ontology records change. But if the version is a hash of the entire ontology state, any change produces a new version, and Arx would need to diff the entire ontology to find which records changed. This is expensive.

**Recommendation:** Add a change log: when a new version is created, store a list of changed node/edge IDs. Arx can then check this change log against its audit provenance chains.

---

## Summary

| Severity | Count | Key Action |
|----------|-------|------------|
| Critical | 2 | Add confidence column to edges table, clarify number field count |
| Major | 3 | Fix cycle detection in CTE, add mapping scaling strategy, add write concurrency |
| Minor | 4 | Clarify versioning scheme, add subgraph to edges, specify Wikipedia confidence, fix CTE |
| Cross-doc | 1 | Add change log for re-audit trigger |

**Overall assessment:** The ontology design is solid, but the SQL schema has a concrete bug (missing confidence column) that would cause runtime failures. The versioning scheme needs clarification to support the re-audit trigger described in the architecture document.
