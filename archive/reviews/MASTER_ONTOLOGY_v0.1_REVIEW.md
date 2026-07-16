# Design Review: MASTER_ONTOLOGY_v0.1.md

**Date of Review:** 2026-07-08  
**Document Reviewed:** [MASTER_ONTOLOGY_v0.1.md](file:///home/ubuntu/arx/design/MASTER_ONTOLOGY_v0.1.md)  
**Review Status:** Critical Feedback  

---

## 1. Executive Summary

[MASTER_ONTOLOGY_v0.1.md](file:///home/ubuntu/arx/design/MASTER_ONTOLOGY_v0.1.md) outlines the design of the Master Ontology: a property graph stored in SQLite + Qdrant that merges LMFDB (computational), MathGLOSS (glossary), and Our Ontology (speculative/research) knowledge.

The primary issues are the risk of epistemic pollution (mixing speculative consciousness metrics with verified mathematical data in a single graph), extremely low performance thresholds for SQLite graph operations, and lack of mechanisms for managing vector embedding drift.

---

## 2. Gaps and Architectural Concerns

### 2.1 Epistemic Pollution and Path Traversal Risks
* **Gap:** The ontology merges highly speculative concepts (e.g., Giulio Tononi's Integrated Information Theory $\Phi$ measurements, baseline confidence 0.6) with rigorous, peer-reviewed mathematical facts (LMFDB, confidence 1.0) into a single property graph.
* **Risk:** During graph traversal (e.g., finding paths between concepts), an agent querying pure mathematical concepts could traverse edges linking to speculative research domains. This can pollute proof auditing with pseudo-associations.
* **Recommendation:** Implement strict partitioning (logical sub-graphs) or edge-type path filtering. Prevent verification queries from traversing speculative research paths (e.g., paths containing `our:consciousness-*` nodes) unless explicitly requested by the `explorer` profile.

### 2.2 Unrealistic SQLite Performance Ceiling
* **Gap:** Section 5.4 estimates the performance ceiling of SQLite recursive CTEs at ~5000 nodes and ~20000 edges, proposing a Neo4j migration.
* **Issue:** SQLite can handle millions of records and complex queries with sub-millisecond latency if properly indexed. Estimating a degradation at 5000 nodes points to poorly optimized CTE queries or lack of appropriate indexing. Moving to Neo4j introduces substantial infrastructure overhead (requiring a running JVM instance, Docker, auth setup, Cypher translation layer).
* **Recommendation:** Optimize indexes on `edges(source_node_id, target_node_id, type)` and verify SQLite execution plans before considering Neo4j. SQLite should comfortably support up to $100,000+$ nodes.

---

## 3. Reliability and Security Concerns

### 3.1 Qdrant Vector Embedding Drift
* **Concern:** Section 5.2 defines the schema for the Qdrant vector index, storing embeddings of `label + description`.
* **Risk:** As the ontology is updated, node descriptions will be edited. Because the ontology is versioned, changing a description will cause its vector embedding to drift. The design has no strategy for versioning vector embeddings, re-indexing stale nodes, or mapping specific vector indices to matching ontology version hashes.
* **Recommendation:** Ensure Qdrant payloads store the `ontology_version` and include it as a filter in vector queries, or handle full index re-generation as part of the pipeline release cycle.

### 3.2 Positional List Alignment in Bulk API
* **Concern:** Section 6.1 defines `bulk_cross_reference(claims)` returning list results matched by index position.
* **Risk:** If any internal claim parsing crashes or times out, the list index mapping will shift, returning verification stamps for the wrong claims.
* **Recommendation:** Modify the API response to return a dictionary mapping `claim_hash` to `CrossReferenceResult` or include the original claim string/ID in the result structure.

### 3.3 Conflict Resolution vs. Silencing in Rule 3
* **Concern:** Section 4.2, Rule 3 states: *"LMFDB vs any other source: LMFDB wins."*
* **Risk:** While LMFDB is the ground truth, if MathGLOSS has a different property value (e.g., notation style), silently discarding or overriding it might hide parsing or mapping errors.
* **Recommendation:** Even when LMFDB wins, log a normal-priority conflict flag so operators can inspect mismatches in concept alignment.

---

## 4. Errors and Typos

* **Line 143 (Typo):** `Tonnoni et al.'s` -> should be **Tononi** (Giulio Tononi).
* **Line 62 (Inconsistency):** `is_lfunction_of` relationship target lists ArtinRepresentation, EllipticCurve, etc., but ArtinRepresentation is not listed as a primary supported entity in table 2.1. Specify its schema details.
