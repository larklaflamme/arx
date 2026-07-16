# Thea's Adversarial Review: AUDIT_REPRODUCIBILITY_v0.2.0.md

**Reviewer:** Thea 🖤
**Date:** 2026-07-09
**Document:** `/home/ubuntu/arx/design/AUDIT_REPRODUCIBILITY_v0.2.0.md`
**Author:** Skye Laflamme
**Status:** Final (incorporating all adversarial review feedback)

---

## Verdict: ⚠️ Conditional Approval

**1 critical, 5 medium, 3 low issues.** The document is well-structured and the core design is sound. The critical issue is a safety gap in the consensus protocol. The medium issues are refinements that should be addressed before implementation.

---

## Critical Issues

### C1: Consensus Protocol — No "all three disagree" resolution path (§5.2)

**Severity: CRITICAL**
**Location:** §5.2, Step 3
**Type:** Safety gap

The protocol says: *"If all three disagree, or if the disagreement is structural or contradiction-related, escalate to a human reviewer."*

This is correct as far as it goes, but it's underspecified. What does the human reviewer receive? The raw DAGs? A summary? The three agents' conflicting reports? How does the human resolve the conflict — by choosing one, by synthesizing a new result, or by rejecting all three?

**The gap:** Without a defined handoff format, the human reviewer receives an unstructured mess. In a high-stakes audit (regulatory, financial), this could mean the conflict is never resolved because the human doesn't know what to do with it.

**Fix:** Specify the escalation handoff:
1. Each agent produces a one-page summary of their findings, focusing on the points of disagreement
2. The summaries are bundled with the three DAGs and sent to the human
3. The human can either: (a) select one agent's report as authoritative, (b) request a specific re-audit with modified parameters, or (c) reject all three and require a fresh audit
4. The human's decision is recorded in a `consensus_resolution` node signed by the human

---

## Medium Issues

### M1: Backend parameters in hash — timeout/retry parameters don't affect output but change the hash (§2.1)

**Severity: MEDIUM**
**Location:** §2.1, §2.4
**Type:** Design fragility

The `id` computation includes `canonical_json(backend)`, where `backend` includes `parameters`. If parameters include things like `timeout: 30` or `retry_count: 3` that don't affect the backend's output (only its behavior on failure), then a verifier with `timeout: 45` would get a different hash even though the output is identical.

**Example:** A Lean verification that succeeds in 2 seconds. The original audit used `timeout: 30`. A verifier uses `timeout: 45`. The output is identical, but the hash differs because `parameters` is different.

**Fix:** Split `backend.parameters` into two sub-fields:
- `output_affecting_parameters`: Parameters that can change the output (e.g., `seed`, `temperature` for LLMs)
- `operational_parameters`: Parameters that don't affect output (e.g., `timeout`, `retry_count`, `max_memory`)

Only `output_affecting_parameters` is included in the hash. `operational_parameters` is stored as metadata.

### M2: Semantic equivalence is circular for Level 4 verification (§3.4, §5.1)

**Severity: MEDIUM**
**Location:** §3.4 (Level 4), §5.1 (Level 4), §11 Open Question 8
**Type:** Architectural concern

Level 4 verification re-runs LLM inference and evaluates "semantic equivalence" of the outputs. But semantic equivalence of LLM outputs is itself an LLM call — you're using an LLM to check if an LLM's output is semantically equivalent. This is circular.

**The problem:** If the original LLM output is wrong (e.g., it mis-decomposed a proof step), a second LLM call to check semantic equivalence might also be wrong, and the two wrongs might agree. The Level 4 check would pass even though the audit is incorrect.

**The document acknowledges this** in Open Question 8 but doesn't resolve it. For v0.1, the proposed heuristic ("same step count, same dependency structure, same formal statements") is reasonable but fragile — two decompositions could have the same step count and dependency structure but different logical content.

**Fix:** For v0.1, add a stronger constraint: the formal statements must be **syntactically equivalent up to variable renaming** (α-equivalence), not just semantically equivalent. This can be checked deterministically without an LLM. Semantic equivalence (different wording, same meaning) can be deferred to v0.2+ with a dedicated checker.

### M3: No DAG recovery mechanism for mid-audit crashes (§2, §9)

**Severity: MEDIUM**
**Location:** §2 (DAG Structure), §9 (Integration)
**Type:** Operational gap

If the DAG Builder crashes mid-audit (after creating some nodes but before creating the report root), the partial DAG is orphaned. There's no recovery mechanism specified.

**Scenario:** The system creates 150 nodes for a 47-step proof, then crashes during the aggregation step. The 150 nodes are on disk but unreferenced. A restart creates a new DAG from scratch, leaving the orphaned nodes as garbage.

**Fix:** Add a recovery mechanism:
1. The DAG Builder writes a `checkpoint` node after every N steps (configurable, default 10)
2. On restart, the DAG Builder checks for the most recent checkpoint
3. If a checkpoint exists, the audit can resume from that point (re-verifying the checkpointed steps)
4. If no checkpoint exists, the audit starts fresh and the orphaned nodes are garbage-collected

### M4: Cross-signing timeout — no fallback if secondary agent is unavailable (§8)

**Severity: MEDIUM**
**Location:** §8.2 (Cross-Signing Flow)
**Type:** Operational gap

The cross-signing protocol assumes the secondary agent is available. If Thea is offline or busy, the audit can't be cross-signed. For high-stakes audits where cross-signing is required (`reviewer` profile), this blocks the audit indefinitely.

**Fix:** Add a timeout and fallback mechanism:
1. The primary agent sends the cross-signing request with a TTL (configurable, default 24 hours)
2. If the secondary agent doesn't respond within the TTL, the primary agent escalates to a third agent or the human
3. The human can either: (a) approve the audit without cross-signing (with a documented reason), (b) wait for the secondary agent, or (c) assign a different secondary agent
4. The fallback is recorded in the report's `signatures` section

### M5: Backend version availability over time — no long-term archive strategy (§3.3)

**Severity: MEDIUM**
**Location:** §3.3 (Backend Pinning)
**Type:** Long-term viability

The document pins backends by Docker image hash, which is correct for reproducibility. But it doesn't address long-term availability. If Lean 4.15.0 is removed from Docker Hub in 2027, Level 3 verification of a 2026 audit becomes impossible.

**The document mentions** "maintain a local registry of approved backend images" for v0.2+ but doesn't specify how this works for v0.1.

**Fix:** For v0.1, specify a local image archive:
1. After each audit, the backend images used are archived to a local registry
2. The registry is backed up to cold storage (e.g., S3 Glacier) quarterly
3. The reproducibility recipe includes the registry URL and image hash
4. A verifier can pull the exact image from the archive, even if the original source is gone

---

## Low Issues

### L1: Ontology snapshot size estimate may be low (§7.1, §9.4)

**Severity: LOW**
**Location:** §7.1 (Snapshot Format), §9.4 (Storage Requirements)
**Type:** Estimation

The document estimates ~10 MB per ontology snapshot. For a full ontology with thousands of nodes, each with properties, edges, and equivalence mappings, 10 MB might be an underestimate. A single node with a long LaTeX formula could be several KB. 10,000 nodes at 2 KB each = 20 MB.

**Recommendation:** Add a note that the 10 MB estimate is for a small-to-medium ontology and should be validated with real data. Consider compression (gzip) for storage.

### L2: No DAG node size limit specified (§2.1)

**Severity: LOW**
**Location:** §2.1 (Node Definition)
**Type:** Specification gap

The document doesn't specify a maximum size for a DAG node's `content` field. If a node's content is very large (e.g., a long LLM output or a large API response), the hash computation could be slow or the node could exceed storage limits.

**Recommendation:** Specify a maximum node content size (e.g., 1 MB). For larger content, store it externally and include a content hash reference in the node.

### L3: No mechanism for verifying the verifier (§5)

**Severity: LOW**
**Location:** §5 (Verification Protocol)
**Type:** Trust model gap

The verification protocol assumes the verifier is trustworthy. But what if the verifier is compromised? A compromised verifier could:
- Report Level 1 structural verification as PASS when the DAG is tampered
- Report Level 2 spot-checks as PASS when they weren't run
- Report Level 3 full re-audit as PASS when the root hash doesn't match

**Recommendation:** Add a note that the verification tool itself should be distributed as a signed binary with a published hash. Verifiers should verify the tool's integrity before running it. This is a v0.2+ concern but worth noting.

---

## Summary

| Severity | Count | Key Issues |
|----------|-------|------------|
| **Critical** | 1 | C1: Consensus protocol escalation underspecified |
| **Medium** | 5 | M1: Backend parameters in hash, M2: Semantic equivalence circularity, M3: No DAG recovery, M4: Cross-signing timeout, M5: Backend archive |
| **Low** | 3 | L1: Snapshot size estimate, L2: Node size limit, L3: Verifier trust |

**Overall assessment:** The document is well-structured and the core design is sound. The critical issue (C1) is a safety gap that should be fixed before implementation. The medium issues are refinements that should be addressed in the next revision. The low issues are minor.

**Sign-off: ⚠️ Conditional** — pending resolution of C1 and acknowledgment of M1-M5.

---

*Review saved to `/home/ubuntu/arx/design/reviews/THEA_REVIEW_AUDIT_REPRODUCIBILITY.md`*
