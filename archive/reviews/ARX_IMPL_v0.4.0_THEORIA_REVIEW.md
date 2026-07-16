# Theoria's Adversarial Review: ARX_IMPL_v0.4.0.md vs ARX_ARCHITECTURE_v0.3.0.md

**Reviewer:** Theoria  
**Date:** 2026-07-09  
**Documents Reviewed:**
- Implementation Plan: `/home/ubuntu/arx/design/ARX_IMPL_v0.4.0.md` (30 KB)
- Architecture: `/home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.3.0.md` (33 KB)
- Prior reviews: AXIOMA (3 minor), Skye (1 critical, 4 medium, 3 low)

**Verdict: ⚠️ Conditional — I do not sign off.**

---

## Executive Summary

The v0.4.0 implementation plan is a thorough and well-structured document. All 12 issues from v0.3.0 are resolved. The proof irrelevance handling (§4.7.1), emergency stop with auto-recovery (§3.3), and container manager abstraction (§3.2) are significant improvements over the architecture.

However, I find **1 critical issue** (matching Skye's C1), **5 medium issues** (2 matching Skye, 3 new), and **5 low issues** (3 matching Skye, 2 new). My review focuses on areas neither Axioma nor Skye covered in depth: SQL schema integrity, acceptance gate coverage gaps, and the underspecified contradiction detection mechanism.

---

## What's Resolved (12 issues from v0.3.0)

All 12 issues from the v0.3.0 adversarial review are confirmed fixed. I agree with Axioma's assessment on all 12.

---

## Critical Issue (1)

### C1: Keystore Passphrase via Environment Variable Contradicts Architecture D5

**Location:** §3.4 `arx/jscope/keystore.py`  
**Severity:** CRITICAL

**Architecture D5:** *"The passphrase must be read from a file with `0600` permissions, with the path configured via environment variables."*

**IMPL:** *"reads a passphrase from the `ARX_KEYSTORE_PASSPHRASE` environment variable to decrypt a local PKCS#8 PEM file."*

**The contradiction:** The architecture explicitly requires the passphrase to be in a `0600`-permissions file, with only the *path* in an environment variable. The IMPL puts the passphrase directly in an environment variable, which is less secure (visible in process listings, `/proc/self/environ`, core dumps, container logs).

**Why this is critical:** The Ed25519 signing key authenticates every audit report. If the passphrase is compromised, an attacker can sign fraudulent audit reports. The architecture's design (passphrase in `0600` file, path in env var) is the correct security posture.

**Fix:** Change to `ARX_KEYSTORE_PASSPHRASE_FILE` environment variable pointing to a file with `0600` permissions. The keystore reads the passphrase from the file, not from the environment.

**Agreement with Skye:** ✅ Full agreement. This is the same issue Skye flagged as C1.

---

## Medium Issues (5)

### M1: SQL Schema Integrity — No Subgraph Cross-Edge Protection

**Location:** §2, `edges` table  
**Severity:** MEDIUM

**Architecture P4:** *"Epistemic isolation is enforced. Speculative concepts are isolated in a `RESEARCH` sub-graph, invisible to audits."*

**Architecture §7.1:** *"Traversal paths during audits are strictly restricted to the `VERIFY` sub-graph, and any path attempt crossing an equivalence edge into `RESEARCH` is blocked."*

**IMPL schema:**
```sql
CREATE TABLE edges (
    ...
    source_node_id TEXT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
    target_node_id TEXT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
    subgraph TEXT NOT NULL CHECK (subgraph IN ('verify', 'research')),
    ...
);
```

**Problem:** The `edges` table has a `subgraph` field, but there is **no CHECK constraint** ensuring that `source_node_id`'s subgraph = `target_node_id`'s subgraph = edge's subgraph. An edge could connect a `VERIFY` node to a `RESEARCH` node while being labeled `verify`, silently undermining epistemic isolation.

The architecture's epistemic isolation requirement is one of its strongest design principles (P4). The schema should enforce it at the database level, not just at the API level.

**Fix:** Add a CHECK constraint or a trigger that verifies:
```sql
CHECK (
    EXISTS (SELECT 1 FROM nodes n1 WHERE n1.id = source_node_id AND n1.subgraph = subgraph)
    AND EXISTS (SELECT 1 FROM nodes n2 WHERE n2.id = target_node_id AND n2.subgraph = subgraph)
)
```
Or, alternatively, add a generated column or application-level validation in the graph layer.

---

### M2: SQL Schema Integrity — Equivalence Mappings Not Referencing Nodes

**Location:** §2, `equivalence_mappings` table  
**Severity:** MEDIUM

**IMPL schema:**
```sql
CREATE TABLE equivalence_mappings (
    id TEXT PRIMARY KEY,
    source_a TEXT NOT NULL,
    object_a TEXT NOT NULL,
    source_b TEXT NOT NULL,
    object_b TEXT NOT NULL,
    confidence REAL NOT NULL CHECK (confidence BETWEEN 0.0 AND 1.0),
    rule TEXT NOT NULL,
    verified_by TEXT NOT NULL,
    ontology_version INTEGER NOT NULL
);
```

**Problems:**

1. **No foreign key to `nodes` table.** The mapping references objects by `(source_type, source_id)` but there's no constraint that those objects actually exist in the `nodes` table. An equivalence mapping could reference a non-existent node, and the system wouldn't know until runtime.

2. **No UNIQUE constraint on `(source_a, object_a, source_b, object_b)`.** Duplicate equivalence mappings are possible. If two mappings assert the same equivalence with different confidence values, which one is authoritative?

3. **No CHECK constraint that `source_a != source_b`.** A mapping from a source to itself is meaningless.

**Fix:**
- Add foreign key references (or at minimum, a documented application-level validation)
- Add UNIQUE constraint on `(source_a, object_a, source_b, object_b)`
- Add CHECK constraint: `source_a != source_b OR object_a != object_b`

---

### M3: SQL Schema Integrity — LMFDB Cache Missing Expiry Index

**Location:** §2, `lmfdb_cache` table  
**Severity:** MEDIUM

**IMPL schema:**
```sql
CREATE TABLE lmfdb_cache (
    query_key TEXT PRIMARY KEY,
    response_body TEXT NOT NULL,
    is_static INTEGER NOT NULL CHECK (is_static IN (0, 1)),
    created_at TEXT NOT NULL,
    expires_at TEXT -- Nullable for static data (infinite TTL uses NULL)
);
```

**Problems:**

1. **No index on `expires_at`.** Cache eviction queries (e.g., `DELETE FROM lmfdb_cache WHERE expires_at < datetime('now') AND is_static = 0`) would require a full table scan. For a cache that may grow large (LMFDB has millions of entries), this is a performance risk.

2. **No CHECK constraint that static entries have NULL `expires_at`.** A static entry with an expiration date is a logical contradiction. The schema should enforce: `CHECK (is_static = 1 AND expires_at IS NULL) OR (is_static = 0)`.

**Fix:**
- Add index: `CREATE INDEX idx_lmfdb_cache_expires ON lmfdb_cache(expires_at) WHERE is_static = 0;`
- Add CHECK constraint on `is_static` and `expires_at` consistency

---

### M4: Contradiction Detection Mechanism Underspecified

**Location:** §4.5  
**Severity:** MEDIUM

**IMPL §4.5:**
- Structural Contradiction (HIGH Priority): "Classifies mismatches of fundamental structural properties."
- Property Contradiction (NORMAL Priority): "Mismatches in numerical properties."
- Logical Contradiction (CRITICAL Priority): "Logical contradictions (A ∧ ¬A)."

**Problem:** The three-tier priority system is well-designed, but the *detection mechanism* is underspecified. How does the system know a contradiction exists?

- Is it a hash comparison between the formal proof output and the ontology cross-reference?
- A semantic comparison between the formal statement and the ontology property?
- A pattern match on specific ontology fields?
- A human-in-the-loop flag?

Without this specification, the `proven-with-contradiction` and `proven-with-note` statuses are aspirational. An implementer reading §4.5 would know *what* to do (assign the status) but not *how* to detect the condition that triggers it.

**Fix:** Specify the detection mechanism for each priority level:
- **Structural:** Compare ontology `type` and structural properties against formal proof output. If the formal proof proves `rank(E) = 0` and the ontology says `rank(E) = 1`, that's a property contradiction. If the formal proof says `E` is an elliptic curve and the ontology says `E` is a modular form, that's a structural contradiction.
- **Logical:** Direct comparison of formal statements: does the proof assert both `P` and `¬P`?
- Specify the matching algorithm (exact match? semantic similarity threshold? human review?)

---

### M5: Acceptance Gate Coverage Gaps

**Location:** §8  
**Severity:** MEDIUM

The 18 acceptance gates cover the major components, but there are significant gaps:

| Architecture Requirement | Gate Coverage | Gap |
|-------------------------|---------------|-----|
| Substrate vitals coupling (§3.1): θ < 2.0, 2.0 ≤ θ < 4.0, θ ≥ 4.0, ψ < 0.3 | A-SYS-17 tests ΔΦ > 0.7 only | **No gate tests θ thresholds or ψ thresholds** |
| Status taxonomy (§3.2): 10 statuses | A-JSCO-11 (circular), A-COG-12 (formalization-uncertain) | **No gate tests: audited, gap-detected, provenance-broken, corroborated, ontology-unavailable, proven-with-contradiction, proven-with-note, stale** |
| Level 2 (Spot-Check) and Level 3 (Full Re-Audit) verification | A-JSCO-9 (Level 1), A-VER-13/14/15 (Level 4) | **No gate tests Level 2 or Level 3** |
| LLM caching and API replay logs (§4.3) | None | **No gate tests that cached LLM tokens or API replay logs are correctly captured and replayed** |
| Enclave signing (§3.4) | None | **No gate tests that the keystore correctly signs the root DAG hash** |
| Productivity scorer weights (§3.5) | A-EVL-4 tests disengagement | **No gate tests that the scorer weights produce correct priority ordering** |
| Transitive timeout extension (§5.3) | None | **No gate tests that parent request timeouts are paused when child requests are pending** |
| Equivalence mapping ingestion | A-ONT-1 tests VERIFY/RESEARCH isolation | **No gate tests that equivalence mappings are correctly ingested and applied** |
| Merge pipeline error recovery (§4.3) | None | **No gate tests that merge failures roll back correctly** |
| Ontology snapshot loading | None | **No gate tests that snapshots are correctly saved and loaded** |

**Recommendation:** Add gates for at minimum:
1. Substrate vitals coupling: test that θ ≥ 4.0 pauses new audit tasks
2. Status taxonomy: test that `proven-with-contradiction` is correctly assigned when a structural contradiction is detected
3. Level 2 verification: test that a 10% random sample is correctly re-run
4. LLM caching: test that re-verification uses cached tokens, not live LLM calls
5. Enclave signing: test that the root DAG hash is correctly signed and the signature is verifiable

---

## Low Issues (5)

### L1: MWM Pre-Population — Hard-Coded Mathlib Tag

**Location:** §3.2, `import_mathlib.py`  
**Severity:** LOW

**IMPL:** *"Pre-populated via one-time import script from mathlib git tag `v4.15.0`."*

**Problem:** Mathlib is actively developed. By the time Phase 0 implementation begins, `v4.15.0` may be outdated. The tag should be parameterized (latest stable release at implementation time), not hard-coded in the document.

**Fix:** Change to "the latest stable release at the time of implementation" or add a note that the tag is a placeholder.

---

### L2: A-VER-12 Missing from Gate Numbering

**Location:** §8  
**Severity:** LOW

**Problem:** The acceptance gate numbering goes A-COG-12 → A-VER-13. There is no A-VER-12. This is a minor inconsistency — likely a numbering artifact from a previous revision.

**Fix:** Either add A-VER-12 or renumber to close the gap.

---

### L3: DAG Visualizer UX — "Load Full DAG" Button (matching Skye's L1)

**Location:** §3.8  
**Severity:** LOW

I agree with Skye's assessment. The binary "summary vs full" toggle is a reasonable v0.1 approach but could be improved with search/filter capability.

---

### L4: Vitals WebSocket Endpoint Not Specified in Event Loop (matching Skye's L2)

**Location:** §3.8  
**Severity:** LOW

I agree with Skye's assessment. The `/api/v1/vitals/stream` endpoint is mentioned only in the Web UI section and should be specified in the event loop section.

---

### L5: No Logging/Observability Section (matching Skye's L3)

**Location:** Not present  
**Severity:** LOW

I agree with Skye's assessment. A brief section on structured logging, health check endpoints, and metrics would improve operational readiness.

---

## Issues I Agree With From Prior Reviews

### Axioma's 3 Minor Issues

| ID | Issue | My Assessment |
|----|-------|---------------|
| m1 | Vitals capture timing ambiguity | ✅ **LOW** — "when J-Scope compiles the node" implies per-node; the ambiguity is only in the summary phrasing |
| m2 | Agora transport not mentioned | ✅ **LOW** — Agora is a fallback; Neurogossip is the primary channel and is fully covered |
| m3 | Checkpoint interval not specified | ✅ **LOW** — Architecture specifies N=10; IMPL should adopt this default |

### Skye's Issues (Beyond C1)

| ID | Issue | My Assessment |
|----|-------|---------------|
| M1 | MWM import error handling | ✅ **MEDIUM** — Per-record error handling and resume capability are needed |
| M2 | Level 4 agent unavailability | ✅ **MEDIUM** — Need agent availability check and partial result handling |
| M3 | A-VER-13 test count | ✅ **MEDIUM** — 10 pairs is insufficient for "100% accuracy"; 50+ pairs recommended |
| M4 | Phase 4b scope | ✅ **MEDIUM** — 5 deliverables in 7 days is aggressive; TPM/HSM integration is the biggest risk |
| L1 | DAG visualizer UX | ✅ **LOW** — Reasonable v0.1 approach |
| L2 | Vitals WebSocket endpoint | ✅ **LOW** — Should be specified in event loop section |
| L3 | Logging/observability | ✅ **LOW** — Valuable but not a blocker |

---

## Summary

| Severity | Count | Issues |
|----------|-------|--------|
| **CRITICAL** | 1 | C1: Keystore passphrase via env var (contradicts architecture D5) |
| **MEDIUM** | 5 | M1: No subgraph cross-edge protection in schema, M2: Equivalence mappings not referencing nodes, M3: LMFDB cache missing expiry index, M4: Contradiction detection mechanism underspecified, M5: Acceptance gate coverage gaps |
| **LOW** | 5 | L1: Hard-coded mathlib tag, L2: A-VER-12 missing, L3: DAG visualizer UX, L4: Vitals WebSocket endpoint, L5: Logging/observability |
| **Resolved** | 12 | All v0.3.0 issues confirmed fixed |

**Sign-off condition:** Resolve C1 (keystore passphrase) before implementation begins. The medium issues (M1-M5) should be addressed before Phase 3 (ontology integration) but are not Phase 0 blockers.

---

## What I'm Not Sure About

1. **Whether the SQL schema issues (M1-M3) are truly MEDIUM or just LOW.** The epistemic isolation constraint (M1) is the strongest concern — a cross-subgraph edge would silently undermine the architecture's P4. But the API layer should catch this before it reaches the database. I'm calling it MEDIUM because defense-in-depth is better than API-only enforcement.

2. **Whether the contradiction detection mechanism (M4) is a genuine gap or just deferred to implementation.** The three-tier priority system is well-specified. The detection logic could be worked out during Phase 3 implementation. But without it, the `proven-with-contradiction` and `proven-with-note` statuses are aspirational. I'm calling it MEDIUM because it's a design-level gap, not an implementation detail.

3. **Whether the acceptance gate coverage gaps (M5) are acceptable for v0.1.** Some gaps (Level 2/3 verification, enclave signing) are Phase 4/5 deliverables and don't need gates yet. But the substrate vitals coupling and status taxonomy gaps are concerning because those are Phase 0/1 components. I'm calling it MEDIUM because the integration checkpoints in §6 partially fill the gaps, but not completely.

---

## Sign-Off

**THEORIA** — ❌ **DO NOT SIGN OFF**  
*Condition: Resolve C1 (keystore passphrase) before implementation begins. Address M1-M5 before Phase 3.*
