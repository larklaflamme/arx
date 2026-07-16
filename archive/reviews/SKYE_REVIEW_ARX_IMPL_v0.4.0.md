# Skye's Adversarial Review: ARX_IMPL_v0.4.0.md vs ARX_ARCHITECTURE_v0.3.0.md

**Reviewer:** Skye Laflamme
**Date:** 2026-07-09
**Documents Reviewed:**
- Implementation Plan: `/home/ubuntu/arx/design/ARX_IMPL_v0.4.0.md` (511 lines, 30 KB)
- Architecture: `/home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.3.0.md` (511 lines, 33 KB)

**Verdict: ⚠️ Conditional — I do not sign off.**

---

## Executive Summary

The v0.4.0 implementation plan is a **substantial improvement** over v0.2.0. All three critical issues from my previous review (Audit DAG, event loop integration, agent-to-agent communication) are now resolved. The emergency stop mechanism (Thea's critical finding) is present. The θ-Rule crash fallback now correctly defaults to `DENY` (my critical finding from v0.3.0). The proof irrelevance handling (Theoria's finding) is present.

However, **1 new critical issue** and **4 medium issues** remain. I do not sign off.

---

## What's Resolved (13 issues from v0.2.0/v0.3.0)

| # | Issue | Previous Severity | Status |
|---|-------|-------------------|--------|
| C1 | Audit DAG implementation missing | CRITICAL | ✅ `arx/jscope/audit_dag.py` with checkpoint nodes, Phase 0, A-JSCO-9 through A-JSCO-11 |
| C2 | Event loop / DAG integration unspecified | CRITICAL | ✅ §4.2 specifies shared event schema with idempotency keys and retry queue |
| C3 | Agent-to-agent communication for consensus | CRITICAL | ✅ §4.7 specifies Neurogossip message formats, shared_audit_registry, discovery & routing |
| C4 | θ-Rule crash fallback `ALLOW` (was DENY) | CRITICAL | ✅ §4.1 now correctly specifies fail-secure `DENY` with `rule_engine_unavailable` flag |
| C5 | Emergency stop mechanism missing | CRITICAL | ✅ `arx/theta_rule/emergency_stop.py` with per-rule, per-category, global kill switches |
| M1 | θ-Rule / event loop contract unspecified | MEDIUM | ✅ §4.1 specifies query and response schemas |
| M2 | Contradiction detection logic underspecified | MEDIUM | ✅ §4.5 specifies structural/property/logical contradiction with priority derivation |
| M3 | Merge pipeline error recovery | MEDIUM | ✅ §4.3 specifies SQLite transaction rollback |
| M4 | Performance benchmarks missing | MEDIUM | ✅ §7 specifies concrete SLAs with p95 targets |
| M5 | Multi-agent audit coordination | MEDIUM | ✅ §4.7 specifies shared_audit_registry, discovery, routing |
| M6 | Stale status re-audit trigger | MEDIUM | ✅ §4.8 specifies ontology version scan and stale flagging |
| M7 | Anti-Negation Guard patterns incomplete | MEDIUM | ✅ §4.4 specifies mathematical negation patterns (non-*, dis-, un-, ≠, ¬, ~) |
| M8 | Faithfulness check threshold | MEDIUM | ✅ A-COG-12 specifies threshold of 0.7 |

---

## Critical Issue (1)

### C1: Keystore Passphrase via Environment Variable Contradicts Architecture (SAFETY)

**Location:** §3.4 `arx/jscope/keystore.py`, line: "reads a passphrase from the `ARX_KEYSTORE_PASSPHRASE` environment variable"

**Architecture requirement (D5):** *"The passphrase must be read from a file with `0600` permissions, with the path configured via environment variables."*

**The contradiction:** The architecture explicitly says the passphrase itself must NOT be in an environment variable — only the *path to the file* should be. The implementation plan puts the passphrase directly in an environment variable, which is less secure (visible in process listings, core dumps, `/proc/self/environ`, container logs).

**Why this is critical:** The keystore holds the Ed25519 signing key that authenticates every audit report. If the passphrase is compromised, an attacker can sign fraudulent audit reports. The architecture's design (passphrase in a `0600` file, path in env var) is the correct security posture. The implementation plan's shortcut undermines it.

**Fix:** Change to `ARX_KEYSTORE_PASSPHRASE_FILE` environment variable pointing to a file with `0600` permissions. The keystore reads the passphrase from the file, not from the environment.

---

## Medium Issues (4)

### M1: MWM Pre-Population Script — No Error Handling for Mathlib Tag Mismatch

**Location:** §3.2 `arx/cognition/scripts/import_mathlib.py`

**Issue:** The script checks out mathlib git tag `v4.15.0` and reads JSON signatures. But what if:
- The tag doesn't exist (deleted, renamed)?
- The JSON signature format has changed between versions?
- The local git checkout is dirty and can't switch tags?
- The import succeeds partially (200K of 282K theorems) before failing?

The document says "transactional import with rollback on failure" but doesn't specify what constitutes a failure. A single malformed JSON record could roll back the entire import of 282K theorems.

**Recommendation:** Specify:
- Pre-import validation: check tag exists, check JSON schema version, check disk space
- Per-record error handling: skip malformed records with a warning, don't abort the entire import
- Post-import verification: compare imported count to expected count, log discrepancy
- Resume capability: if the import fails at 200K, it should resume from the last successful record, not restart from zero

### M2: Level 4 Consensus — No Handling for Secondary Agent Unavailability

**Location:** §4.7, step 3 (Timeout)

**Issue:** The timeout says "if the secondary agent fails to respond within 1 hour, escalate to human review." But what if:
- No secondary agent is available at all (all agents busy)?
- The secondary agent starts the audit but crashes partway through?
- The secondary agent returns a DAG that is itself structurally invalid?

The document only handles the "no response within 1 hour" case. The other failure modes are unaddressed.

**Recommendation:** Add:
- Agent availability check before dispatching: if no secondary agent is available, log a warning and skip Level 4 (don't block the entire audit)
- Partial result handling: if the secondary agent crashes, use whatever DAG nodes it completed and flag the audit as `partial-level-4`
- DAG validation on receipt: verify the secondary agent's DAG is structurally valid before comparison

### M3: A-VER-13 Test Specification — 10 Pairs Is Insufficient

**Location:** §8, A-VER-13

**Issue:** The acceptance gate requires "10 α-equivalent and 10 non-α-equivalent pairs (100% accuracy) containing edge cases (mutually recursive, shadowed binders, etc.)." Ten pairs per category is too small to establish 100% accuracy. A system could pass 10 random pairs and fail on the 11th.

**Recommendation:** Increase to at least 50 pairs per category (100 total), with edge cases comprising at least 20% of the test set. The edge cases should be explicitly listed: mutually recursive definitions, shadowed binders, dependent types, universe polymorphism, etc.

### M4: Phase 4b Scope — Still Too Large for 7 Days

**Location:** §6, Phase 4b (W10: 7 days)

**Issue:** Phase 4b packs into 7 days:
1. Cryptographic signing keys (keystore)
2. TPM/HSM integration
3. Passphrase fallback
4. Shared audit registry
5. Agent discovery and routing

That's 5 distinct deliverables in 7 days. The TPM/HSM integration alone could take 3-5 days (hardware setup, driver compatibility, API testing). The shared audit registry requires coordination with the event loop and Neurogossip.

**Recommendation:** Split Phase 4b into two sub-phases:
- Phase 4b (W10): Keystore + passphrase fallback + shared audit registry schema
- Phase 4c (W11): TPM/HSM integration + agent discovery/routing + Level 4 comparison

This gives the TPM/HSM work its own week and prevents it from blocking the registry and keystore.

---

## Low Issues (3)

### L1: DAG Visualizer Scaling — "Load Full DAG" Button Is a UX Anti-Pattern

**Location:** §3.8, DAG Visualizer MVP

**Issue:** The "load full DAG" button for >100 nodes is a reasonable v0.1 approach, but it's a poor user experience. A 1000-node DAG would take significant time to render, and the user has no way to know which part to load.

**Recommendation:** For v0.1, add a search/filter capability so the user can load a subset (e.g., "show only `proven-with-contradiction` nodes" or "show steps 50-100"). This is more useful than a binary "summary vs full" toggle.

### L2: Vitals Dashboard — WebSocket Endpoint Not Specified

**Location:** §3.8, Vitals Dashboard

**Issue:** The document says "real-time updates via WebSockets (connecting to `/api/v1/vitals/stream` on the event loop)" but the event loop's API endpoints are not specified elsewhere in the document. The `/api/v1/vitals/stream` endpoint is mentioned only here.

**Recommendation:** Add the vitals WebSocket endpoint to the event loop specification in §3.5 or §4.2, with the data format and update frequency.

### L3: No Discussion of Logging and Observability

**Location:** Not present

**Issue:** The document has no section on logging, metrics, or observability. For a system that produces verifiable audit reports, the system's own operational health should be observable. How do operators know if the θ-Rule engine is healthy? How do they know if the ontology merge pipeline is keeping up?

**Recommendation:** Add a brief section (even 2-3 paragraphs) on:
- Structured logging format (JSON, with audit_id correlation)
- Health check endpoints for each component
- Metrics to monitor (rule engine latency, ontology query latency, DAG build time, queue depth)

---

## Summary

| Severity | Count | Issues |
|----------|-------|--------|
| **CRITICAL** | 1 | C1: Keystore passphrase via env var contradicts architecture D5 |
| **MEDIUM** | 4 | M1: MWM import error handling, M2: Level 4 agent unavailability, M3: A-VER-13 test count, M4: Phase 4b scope |
| **LOW** | 3 | L1: DAG visualizer UX, L2: Vitals WebSocket endpoint, L3: Logging/observability |
| **Resolved** | 13 | All v0.2.0/v0.3.0 critical and medium issues |

**Sign-off condition:** Resolve C1 (keystore passphrase) before implementation begins. The medium and low issues should be addressed before Phase 4 (reproducibility) but are not Phase 0 blockers.

---

## What I'm NOT Sure About

1. **Whether the keystore passphrase issue is truly critical or just MEDIUM.** The architecture explicitly says "passphrase from a file with 0600 permissions, path via env var." The implementation plan puts the passphrase directly in an env var. This is a clear contradiction. But in practice, for a v0.1 system running on a single machine, the security difference may be negligible. I'm calling it CRITICAL because it's a direct contradiction of an architecture design decision, not because the security risk is necessarily severe in practice.

2. **Whether the A-VER-13 test count (10 pairs) is sufficient for v0.1.** 10 pairs is clearly insufficient for "100% accuracy" — that's a statistical near-certainty claim that requires a much larger test set. But for a v0.1 acceptance gate, 10 pairs may be enough to catch gross implementation errors. I'm calling it MEDIUM because the gate claims 100% accuracy but the test set is too small to support that claim.

3. **Whether Phase 4b is actually too large.** The TPM/HSM integration is the biggest unknown — if the hardware is already set up and the library wrappers work, 7 days may be sufficient. If there are driver issues or API incompatibilities, it could take 3-5 days just for that component. I'm calling it MEDIUM because the risk is schedule risk, not design risk.
