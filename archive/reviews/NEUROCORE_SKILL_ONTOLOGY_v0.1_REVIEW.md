# Design Review: NEUROCORE_SKILL_ONTOLOGY_v0.1.md

**Date of Review:** 2026-07-08  
**Document Reviewed:** [NEUROCORE_SKILL_ONTOLOGY_v0.1.md](file:///home/ubuntu/arx/design/NEUROCORE_SKILL_ONTOLOGY_v0.1.md)  
**Review Status:** Secondary Feedback  

---

## 1. Executive Summary

[NEUROCORE_SKILL_ONTOLOGY_v0.1.md](file:///home/ubuntu/arx/design/NEUROCORE_SKILL_ONTOLOGY_v0.1.md) details the NeuroCore skill wrapping the Master Ontology API. It exposes lookup, search, traversal, pathfinding, and cross-referencing capabilities to all agents in the ecosystem.

Key findings include issues with local memory cache synchronization, session-based bias in fallback rate tracking, and potential agent crash hazards due to raw exception handling.

---

## 2. Gaps and Architectural Concerns

### 2.1 Non-Shared Local Memory Cache
* **Gap:** Section 3.3 specifies: *"Cache store: in-memory (Python dict) — no Redis dependency for the skill itself."*
* **Risk:** The skill runs within the process space of each individual agent instance. If multiple agents (e.g., Skye, Arx, Thea) query the same ontology, they will maintain separate, redundant caches. This increases overall memory footprint and leads to cache inconsistency if the ontology changes and one agent's cache has not expired.
* **Recommendation:** Since the Master Ontology API itself has a local SQLite cache layer (Section 5.3 of Master Ontology), the skill should rely on the server's cache or share a lightweight local cache if running on the same host, rather than keeping independent Python dicts in each agent.

### 2.2 Session-wide Bias in Fallback Rate Tracking
* **Gap:** The cross-reference response includes a `fallback_rate` representing the *"cumulative fallback rate for this session"*.
* **Issue:** Fallback rate is used by Arx to discount overall audit confidence (reducing it if the fallback rate > 0.5). If this metric is cumulative over the entire session, early complex proofs (which require LLM fallback parsing) will permanently skew the fallback rate of later, simpler proofs that are verified via regex extractors.
* **Recommendation:** Track and report the fallback rate per-audit or per-proof, rather than cumulatively across the entire agent conversation session.

---

## 3. Reliability and Security Concerns

### 3.1 Crash Hazard on Raw API Exceptions
* **Concern:** Section 3.2 specifies that the skill raises Python exceptions (like `OntologyUnavailableError`, `OntologyTimeoutError`) during API failures.
* **Risk:** A raised exception that is not caught by the calling agent will crash the agent's main thread. In [ARX_ARCHITECTURE_v0.1.md](file:///home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.1.md), it is stated that ontology unavailability should *not* block the audit. If the skill raises a raw exception, it forces the agent to crash unless every consumer wraps every call in try-except blocks.
* **Recommendation:** The skill should return a standard response object with `success: false` and error details in the metadata, letting the agent gracefully degrade without throwing hard errors.

### 3.2 Lack of Partial Failure Reporting in Bulk API
* **Concern:** The `ontology_bulk_cross_reference` query does not define how partial failures are handled.
* **Risk:** If a batch of 50 claims is sent, and the 24th claim causes a parsing timeout, does the entire bulk call fail? Re-running the entire batch wastes API credits and LLM resources.
* **Recommendation:** Ensure the response payload maps individual results to their status, allowing partial success returns where successful verifications are stored and failed ones are flagged.
