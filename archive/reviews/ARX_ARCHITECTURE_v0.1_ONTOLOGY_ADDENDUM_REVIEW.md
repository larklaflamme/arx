# Design Review: ARX_ARCHITECTURE_v0.1_ONTOLOGY_ADDENDUM.md

**Date of Review:** 2026-07-08  
**Document Reviewed:** [ARX_ARCHITECTURE_v0.1_ONTOLOGY_ADDENDUM.md](file:///home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.1_ONTOLOGY_ADDENDUM.md)  
**Review Status:** Secondary Feedback  

---

## 1. Executive Summary

The [Addendum document](file:///home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.1_ONTOLOGY_ADDENDUM.md) summarizes key design decisions (D1–D10) regarding the Master Ontology integration. Because the core content has been folded into the main architecture document, the primary focus of this review is on the specific rationale, configuration values, and operational gaps identified in the decisions.

---

## 2. Gaps and Inconsistencies

### 2.1 Ineffectiveness of `proven-with-contradiction` for Our Ontology
* **Gap:** Decision **D4** establishes that if ontology confidence $\ge 0.8$, a contradiction escalates to `proven-with-contradiction` (HIGH priority), else it receives `proven-with-note` (NORMAL priority).
* **Inconsistency:** Decision **D5** states that claims backed by our ontology default to ~0.42 confidence (baseline 0.6 multiplied by weights). 
* **Impact:** Since $0.42 < 0.8$, any contradiction between a formal proof and our ontology will *always* default to `proven-with-note` (NORMAL priority). Even if our ontology contains a critical structural mapping error or a major conceptual mismatch, it will never trigger a high-priority escalation because the confidence score is mathematically capped below the 0.8 threshold.
* **Recommendation:** Separate the contradiction priority threshold from the raw source confidence baseline, or use a distinct category-based escalation rule for structural contradictions.

### 2.2 Short Cache TTL for Invariant Computational Data
* **Gap:** Decision **D9** sets a TTL of 24 hours for static LMFDB data like conductor, rank, and torsion.
* **Issue:** Mathematical invariants (e.g., the rank of elliptic curve 37.a1) are eternal truths. They do not change. Setting a 24-hour TTL on static invariants forces unnecessary API queries and network overhead during re-audits.
* **Recommendation:** Static computational data from LMFDB should be cached indefinitely. Introduce database-level version tracking rather than time-based expiration (TTL).

---

## 3. Reliability and Security Concerns

### 3.1 Security Gaps in Agent-Authored PRs for Ontology Updates
* **Concern:** Decision **D8** states that "Our Ontology" is updated via YAML PRs in the arx repository, and *"Any agent can propose additions via PR."*
* **Risk:** This implies that automated agents have write access/credentials to push branches or submit Pull Requests directly to the Git repository. If an agent is compromised (e.g., via prompt injection or dependency hijacking), the attacker can use the agent's credentials to write malicious code or inject compromised files into the repository.
* **Recommendation:** Agents should write proposals to an isolated, sandboxed database table or a dedicated queue. A separate system tool or human operator should handle git commits, rather than giving raw git write credentials directly to the agents.

### 3.2 Threshold Benchmarking for SQLite Graph Traversal
* **Concern:** Decision **D1** proposes SQLite for graph storage, with a plan to benchmark and define a migration threshold at ~5000 nodes / ~20000 edges.
* **Risk:** The suggested threshold of 5000 nodes is extremely low and will be reached very quickly. Once we begin caching elliptic curves and modular forms from LMFDB, the node count will exceed 5000 within days of active audit work. If SQLite CTE performance degrades at this scale, it represents a structural bottleneck.
* **Recommendation:** Ensure index optimization is verified. Properly indexed SQLite tables can easily handle hundreds of thousands of nodes/edges with recursive CTEs. Migrating to Neo4j adds massive infrastructure overhead (Docker, JVM memory, configuration) that should be avoided unless absolutely necessary.
