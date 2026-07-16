# Adversarial Review: ARX_IMPL_v0.2.0.md vs ARX_ARCHITECTURE_v0.3.0.md

**Reviewer:** AXIOMA  
**Date:** 2026-07-10  
**Status:** **FAIL — 15 issues found (3 CRITICAL, 5 MAJOR, 7 MINOR)**  
**Verdict:** Requires revision before sign-off

---

## Executive Summary

The implementation plan (v0.2.0) claims to be "based on ARX_ARCHITECTURE_v0.3.0.md" but contains **3 critical inconsistencies**, **5 major omissions**, and **7 minor errors** that would cause the implemented system to diverge from the architecture specification. The most severe issues are a **phase ordering dependency violation** (the parser is built before the normalization pipeline it depends on), a **consensus protocol mismatch** (different escalation paths), and a **backend coverage gap** (4 of 9 backends implemented without explicit deferral).

---

## 1. CRITICAL Issues (Must Fix Before Implementation)

### CRIT-1: Phase Ordering Dependency Violation — Parser Built Before Normalization

**Severity:** CRITICAL  
**Architecture Reference:** §9.1 (Steps 2–3), §4.1 (Input Normalization)  
**Implementation Reference:** §5 Phase 3 (Parser), Phase 4 (J-Scope)

**Finding:** The architecture's data flow (§9.1) specifies:
1. Step 2: Normalization pipeline processes proof text
2. Step 3: R3 Goal DAG decomposes normalized proof into ProofStep nodes

The implementation plan inverts this dependency:
- Phase 3 (W7–W8): Builds the two-tier claim parser (`r3_goal_dag.py`)
- Phase 4 (W9–W10): Builds the normalization pipeline (`reproducibility.py`)

**Impact:** The parser is built and tested against unnormalized input. When normalization is later introduced, the parser's extractors may fail on normalized text (e.g., LaTeX `\mathbb{N}` replaced with `ℕ`). This creates a rework cycle where the parser must be modified after Phase 4 to handle normalized input.

**Required Fix:** Move normalization pipeline to Phase 0 or Phase 1, before the parser. The data flow dependency must be reflected in the build order.

---

### CRIT-2: Consensus Protocol Escalation Path Mismatch

**Severity:** CRITICAL  
**Architecture Reference:** §4.6 (Consensus Protocol for Disagreements)  
**Implementation Reference:** §4.2 (Consensus Protocol for Disagreements)

**Finding:** The architecture specifies a **2-step escalation**:
1. Automated resolution (accept more thorough audit or lower confidence)
2. Human escalation (one-page summaries, DAG diff, 24h timeout)

The implementation plan specifies a **3-step escalation**:
1. Automated check (same as architecture)
2. Escalation to third agent (e.g., Theoria) — **NOT in architecture**
3. Human review handoff (same as architecture)

**Impact:** The implementation adds an entire escalation tier not present in the architecture. This changes the consensus protocol. If the architecture is the source of truth, the implementation is wrong. If the implementation is a refinement, the architecture must be updated to v0.4.0. Currently inconsistent.

**Required Fix:** Either remove the third-agent step from the implementation plan (matching architecture), or add it to the architecture document and re-version both.

---

### CRIT-3: Backend Coverage Gap — 4 of 9 Backends Implemented

**Severity:** CRITICAL  
**Architecture Reference:** §3.5 (R4 — Compute & Evidence Kernel)  
**Implementation Reference:** §3.2 (r4_compute_kernel.py), §5 Phase 3

**Finding:** The architecture lists **9 verification backends**:
- Lean4, Z3, Coq, Vampire, SymPy, SageMath, PARI/GP, GAP, mpmath

The implementation plan implements only **4**:
- Lean, Z3, SymPy, Master Ontology API

**Missing backends:** Coq, Vampire, SageMath, PARI/GP, GAP, mpmath

**Impact:** The architecture's verification pipeline (§9.1 Step 5) dispatches to "formal backends (Lean4, Z3)" but the architecture also lists 7 other backends. The implementation plan is silent on whether the missing 5 are deferred to v0.3.0 or simply forgotten. If a proof requires SageMath or PARI/GP for number-theoretic computations, the system cannot handle it.

**Required Fix:** Either:
(a) Implement all 9 backends, or
(b) Explicitly defer the missing 5 to v0.3.0 with rationale, or
(c) Update the architecture to list only 4 backends for v0.2.0

---

## 2. MAJOR Issues (Should Fix Before Implementation)

### MAJ-1: Substrate Vitals Not Captured Per DAG Node

**Severity:** MAJOR  
**Architecture Reference:** §1.2 D4, §4.2 (Node Schema)  
**Implementation Reference:** §3.3 (audit_dag.py)

**Finding:** Architecture design decision D4 states: "Cognitive vitals (θ, ΔΦ, ψ, fragmentation) are captured before every node generation. A change in cognitive state during a re-audit triggers a warning but does not fail structural verification."

The implementation plan's `audit_dag.py` module description (§3.3) does not mention capturing substrate vitals. The DAG node schema (§3.3) has no `vitals` field. The `AuditNode` YAML schema in the architecture (§4.2) also does not include vitals — this is a gap in the architecture document itself, but the implementation plan should catch it.

**Impact:** Without vitals capture, re-audits cannot detect cognitive state changes. Architecture D4's warning mechanism cannot function. The system loses the ability to distinguish between "same audit, different cognitive state" and "different audit."

**Required Fix:** Add `vitals: { theta, delta_phi, psi, fragmentation }` to the DAG node schema. Implement vitals capture in `audit_dag.py` before every node generation.

---

### MAJ-2: Full Ontology Snapshots Not Implemented

**Severity:** MAJOR  
**Architecture Reference:** §1.2 D6  
**Implementation Reference:** §2 (version_log table)

**Finding:** Architecture design decision D6 states: "Snapshots of Our Ontology are written as complete content-addressed files rather than incremental diffs, avoiding cascade failures if a historical diff is corrupted."

The implementation plan has a `version_log` table that tracks version numbers and hashes, but no mechanism for writing or storing content-addressed snapshot files. The `version_log` table's `hash` field could reference a snapshot file, but the file storage mechanism is unspecified.

**Impact:** Without full snapshots, a corrupted diff in the ontology history would cascade through all subsequent versions, making historical audits unreproducible. This violates architecture principle P3 (Reproducibility is the gate).

**Required Fix:** Specify snapshot file format (e.g., gzipped JSON, SQLite dump), storage location, and content-addressed naming scheme (e.g., `snapshots/<sha256>.json.gz`). Implement snapshot writer in the ontology ingestion pipeline.

---

### MAJ-3: Profile Configurations Missing

**Severity:** MAJOR  
**Architecture Reference:** §8 (Project Structure), §3.5 (Ontology Unavailability)  
**Implementation Reference:** Not mentioned

**Finding:** The architecture specifies 5 profiles (auditor, prover, verifier, reviewer, explorer) with a dedicated module at `arx/profiles/loader.py` and configs at `arx/profiles/configs/`. The implementation plan has no mention of profiles anywhere.

Profiles affect critical behavior:
- **Ontology unavailability** (§3.5): "the verifier and auditor profiles stamp steps as `ontology-unavailable` and prevent them from transitioning to `proven`" — this implies different profiles have different behavior, but the implementation plan has no profile system.
- **Verification effort** (§3.1): θ thresholds reduce verification effort "by one tier" — which tier depends on the profile.

**Impact:** Without profiles, the system has a single mode of operation. The ontology unavailability behavior described in §3.5 cannot be profile-dependent. The verification effort budgeting in §3.1 cannot vary by profile.

**Required Fix:** Add profile module specification to §3.2 (Cognition Layer) or as a new subsection. Define at minimum the auditor and verifier profiles with their ontology unavailability behaviors.

---

### MAJ-4: Meta-Cognition Layer Missing

**Severity:** MAJOR  
**Architecture Reference:** §8 (Project Structure), §3.1 (Substrate Vitals Coupling)  
**Implementation Reference:** Not mentioned

**Finding:** The architecture specifies a meta-cognition module at `arx/meta_cognition/rule_based.py` for v0.1 rule-based metrics. The implementation plan has no meta-cognition module.

The meta-cognition layer handles:
- Vitals-aware resource allocation (§3.1: θ thresholds, ΔΦ > 0.7 recovery)
- Verification effort budgeting (skip tiers based on vitals)
- Profile switching (not yet implemented, but the module is the hook)

**Impact:** The substrate vitals coupling rules in §3.1 have no implementation home. The θ thresholds, ΔΦ recovery, and ψ parallelism rules are specified in the architecture but have no corresponding code module in the implementation plan.

**Required Fix:** Add `arx/meta_cognition/` module specification to §3.2 or as a new subsection. Implement at minimum the vitals monitoring and threshold-based resource allocation.

---

### MAJ-5: Missing Acceptance Gates for Key Features

**Severity:** MAJOR  
**Architecture Reference:** Various  
**Implementation Reference:** §6 (Acceptance Gates)

**Finding:** The implementation plan defines 14 acceptance gates (A-ONT-1 through A-WUI-14). However, several architecture features have no corresponding acceptance gate:

| Architecture Feature | Missing Gate |
|---------------------|--------------|
| Substrate vitals coupling (θ thresholds, ΔΦ > 0.7) | No gate tests vitals-aware behavior |
| Contradiction priority (structural vs property-level) | No gate tests HIGH vs NORMAL priority |
| Provenance chain integrity | No gate tests tamper detection |
| Consensus protocol (human handoff, 24h timeout) | No gate tests escalation path |
| Profile switching behavior | No gate tests profile-dependent behavior |
| Full ontology snapshot verification | No gate tests snapshot integrity |
| LMFDB cache invalidation (indefinite TTL) | No gate tests cache behavior for static data |
| Epistemic isolation (VERIFY vs RESEARCH block) | A-ONT-1 tests traversal block, but not at edge level |
| Key management (signature verification) | No gate tests signature validation |

**Impact:** These features can be implemented incorrectly without any acceptance test catching the error. The system could pass all 14 gates but still violate the architecture specification.

**Required Fix:** Add acceptance gates for each of the above features. Minimum 5 additional gates.

---

## 3. MINOR Issues (Should Fix Before Final)

### MIN-1: Version Mismatch

**Severity:** MINOR  
**Architecture Reference:** Header (v0.3.0)  
**Implementation Reference:** Header (v0.2.0)

**Finding:** The implementation plan is versioned v0.2.0 but claims to be "based on ARX_ARCHITECTURE_v0.3.0.md." The version numbers are inconsistent. If the implementation plan is for v0.3.0 of the architecture, it should be v0.3.0 of the implementation plan. If it's for an earlier version, it should reference the correct architecture version.

**Required Fix:** Bump implementation plan to v0.3.0 to match the architecture version, or clarify the versioning scheme.

---

### MIN-2: lmfdb_cache Schema Has `expires_at NOT NULL` for Static Data

**Severity:** MINOR  
**Architecture Reference:** §7.1 (Cache)  
**Implementation Reference:** §2 (lmfdb_cache table)

**Finding:** The `lmfdb_cache` table has `expires_at TEXT NOT NULL`. For static data (`is_static = 1`), the architecture specifies indefinite TTL. The `NOT NULL` constraint forces an expiration date even for static data, which contradicts the architecture.

**Required Fix:** Change `expires_at` to `TEXT` (nullable) or use a sentinel value (e.g., `'9999-12-31'`) for static data. Document the sentinel convention.

---

### MIN-3: edges Table Missing `subgraph` Field

**Severity:** MINOR  
**Architecture Reference:** §7.1 (Epistemic Isolation)  
**Implementation Reference:** §2 (edges table)

**Finding:** The `nodes` table has a `subgraph` field for VERIFY/RESEARCH partitioning, but the `edges` table does not. For proper epistemic isolation, edges should also carry subgraph information to prevent cross-subgraph traversals at the edge level. Without it, a traversal from a VERIFY node through an unmarked edge could reach a RESEARCH node.

**Required Fix:** Add `subgraph TEXT NOT NULL CHECK (subgraph IN ('verify', 'research'))` to the `edges` table. Ensure edge subgraph is consistent with both endpoint nodes' subgraphs.

---

### MIN-4: MWM Mathlib Counts Not Specified

**Severity:** MINOR  
**Architecture Reference:** §3.3 (~282,207 theorems, ~133,813 definitions)  
**Implementation Reference:** §3.2 (r2_mwm.py)

**Finding:** The architecture specifies precise mathlib counts. The implementation plan says "Caches mathlib signatures" but provides no scale information. This affects capacity planning for the MWM cache.

**Required Fix:** Add mathlib scale information to the r2_mwm.py module description. Specify expected memory footprint and sync frequency.

---

### MIN-5: Docker Container Management Not Specified

**Severity:** MINOR  
**Architecture Reference:** §9.1 (Step 5)  
**Implementation Reference:** §3.2 (backends/)

**Finding:** The architecture specifies "pinned Docker containers" for formal backends. The implementation plan lists Python files (`lean_backend.py`, `z3_backend.py`) but has no module for container lifecycle management (start, stop, health check, image pinning, version tracking).

**Required Fix:** Add a container management module (e.g., `arx/cognition/backends/container_manager.py`) or specify that Docker is managed externally. Document image pinning strategy and version tracking.

---

### MIN-6: Edge Confidence Derivation Rule Unspecified

**Severity:** MINOR  
**Architecture Reference:** §7.1 (Source Confidence)  
**Implementation Reference:** §2 (edges table)

**Finding:** The `edges` table has a `confidence` field, but neither the architecture nor the implementation plan specifies how edge confidence is derived from source confidence. For example, if node A is from LMFDB (confidence 1.0) and node B is from Our Ontology (confidence 0.6), what is the confidence of the edge connecting them?

**Required Fix:** Specify the edge confidence derivation rule. Recommended: `edge_confidence = min(source_a_confidence, source_b_confidence) * edge_type_weight` where `edge_type_weight` is 1.0 for identity edges, 0.9 for equivalence mappings, and configurable for other types.

---

### MIN-7: Acceptance Gate A-RULE-7 Has Inverted Comparison

**Severity:** MINOR  
**Architecture Reference:** §6.1 (Round-Trip Validation)  
**Implementation Reference:** §6 (A-RULE-7)

**Finding:** The acceptance gate states: "Rule compiler fails deployment if a vector fails to decode back to natural language with similarity ≥ 0.8."

The architecture states: "Newly compiled rule vectors must decode back to natural language with a semantic similarity of ≥ 0.8 against the source text before deployment."

The gate's condition is inverted. It should read: "fails deployment if similarity < 0.8" (i.e., fails when similarity is too low, not when it's high enough).

**Required Fix:** Change A-RULE-7 to: "Rule compiler fails deployment if a vector fails to decode back to natural language with similarity < 0.8."

---

## 4. Cross-Document Consistency Issues

### X-1: Consensus Protocol (CRIT-2) — Architecture Must Be Updated

If the implementation plan's third-agent escalation step is accepted as a refinement, the architecture document must be updated to v0.4.0 to include it. Currently, the two documents specify different protocols.

### X-2: Backend Coverage (CRIT-3) — Architecture Must Be Updated

If the implementation plan's 4-backend scope is accepted, the architecture must either:
- Remove the 5 unimplemented backends from §3.5, or
- Add a note that they are deferred to v0.3.0

### X-3: DAG Node Schema — Architecture Missing Vitals Field

The architecture's own `AuditNode` YAML schema (§4.2) does not include a `vitals` field, despite D4 requiring vitals capture. This is a gap in the architecture document that the implementation plan cannot fix alone.

---

## 5. Sign-Off Status

| Category | Count | Status |
|----------|-------|--------|
| CRITICAL | 3 | ❌ Must fix |
| MAJOR | 5 | ❌ Should fix |
| MINOR | 7 | ⚠️ Should fix |
| **Total** | **15** | **❌ FAIL** |

**Verdict:** The implementation plan is **not approved** in its current state. The 3 critical issues (phase ordering, consensus protocol mismatch, backend coverage) must be resolved before implementation begins. The 5 major issues (vitals capture, ontology snapshots, profiles, meta-cognition, acceptance gates) should be resolved concurrently.

**Recommended Action:** Produce ARX_IMPL_v0.3.0.md that:
1. Moves normalization pipeline to Phase 0
2. Aligns consensus protocol with architecture (or updates both)
3. Either implements all 9 backends or explicitly defers 5 with rationale
4. Adds profiles, meta-cognition, vitals capture, and ontology snapshots
5. Adds missing acceptance gates
6. Fixes all minor issues (version, schema, inverted comparison)
