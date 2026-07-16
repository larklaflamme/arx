# Arx Implementation Schedule & Progress Tracker

This document tracks the implementation progress of the **Arx Math Proof Audit Agent (v0.5.0)** based on [ARX_IMPL_v0.5.0.md](file:///home/ubuntu/arx/design/ARX_IMPL_v0.5.0.md).

---

## Overall Status
* **Current Phase:** Phase 0 (Foundations)
* **Overall Progress:** 5%
* **Last Updated:** 2026-07-09

---

## 1. Project Initialization & Structure

- [x] Create project subdirectories
- [x] Initialize Python package files (`__init__.py`)
- [x] Configure base environment and metadata
- [x] Create initial `ARX_SCHEDULE.md` progress tracker

---

## 2. Phase-by-Phase Checklist

### Phase 0: Foundations, Ingestion & Normalization (W1–W2: 14 days)
* **Status:** In Progress (10%)
- [ ] Create Master Ontology SQLite Schema & Indexes (`ontology/data/master_ontology.db`)
- [ ] Implement Abstract Graph Interface (`ontology/graph/abstract.py`)
- [ ] Implement SQLite Graph CTE traversals (`ontology/graph/sqlite_graph.py`)
- [ ] Write YAML/Wikidata importer for Our Ontology & MathGLOSS
- [ ] Implement MWM Pre-population Script (`arx/cognition/scripts/import_mathlib.py`) with transactional imports & rollback
- [ ] Implement MWM Daily Sync & Tag diffing (`arx/cognition/r2_mwm.py`)
- [ ] Build J-Scope Input Normalization pipeline (`arx/jscope/reproducibility.py`)
- [ ] Set up Phase 0 Integration Checkpoint (SQLite CRUD, normalization mappings, and MWM mock queries)

### Phase 1: Event Loop, Bounded Queues & DAG Builder (W3–W4: 14 days)
* **Status:** Pending (0%)
- [ ] Build Neurogossip-v3 bounded queue (capacity 1000) and backpressure dropping (`arx/comms/neurogossip.py`)
- [ ] Implement Event Loop Core Scheduler & heartbeats (`arx/main.py`)
- [ ] Build Meaningless Message Filter & Context-Aware Affirmations
- [ ] Implement Agent Disconnection / heartbeat timeouts
- [ ] Build Event Loop / DAG Builder contract & composite idempotency keys (`arx/jscope/audit_dag.py`)
- [ ] Implement DAG node schemas, parent chaining, and Level 1 structural verifier
- [ ] Set up Phase 1 Integration Checkpoint (Heartbeats, backpressure drops, Level 1 DAG verification)

### Phase 2: $\theta$-Rule Engine & Safety Gates (W5–W6: 14 days)
* **Status:** Pending (0%)
- [ ] Set up Qdrant vector collections (`ontology_concepts`, `theta_rules`)
- [ ] Implement Rule Compiler with round-trip similarity checks (`arx/theta_rule/compiler.py`)
- [ ] Implement resolver priority,Specificity tie-breakers, and chain depth limiter
- [ ] Build Anti-Negation Guard mathematical regex patterns (`arx/theta_rule/matcher.py`)
- [ ] Build Rule Engine Emergency Stop kill switches, authorization rules, and J-Scope logs
- [ ] Set up Phase 2 Integration Checkpoint (Compiler checks, negation guard overrides, kill switch propagation)

### Phase 3: Ontology Skill, Parser & AF Faithfulness Check (W7–W8: 14 days)
* **Status:** Pending (0%)
- [ ] Build FastAPI REST server with verify/research partition limits (`ontology/api/server.py`)
- [ ] Implement NeuroCore bulk cross-reference returning hash-keyed dicts (`neurocore-skill-ontology/skill.py`)
- [ ] Build R3 Two-tier Goal DAG parser (`arx/cognition/r3_goal_dag.py`) and fallback rate tracker
- [ ] Implement R4 verifier backends (Lean4, Z3, SymPy, mpmath) dispatches (`arx/cognition/r4_compute_kernel.py`)
- [ ] Implement Autoformalizer faithfulness check, description retries, and fallback (`arx/cognition/af_autoformalizer.py`)
- [ ] Set up Phase 3 Integration Checkpoint (Hash-keyed dict returns, fallback rates, AF similarity checks)

### Phase 4: Reproducibility logs, Checkpoints & Consensus (W9–W11: 21 days)
* **Status:** Pending (0%)
- [ ] **Phase 4a (W9: 7 days):** Implement API mock replay server, checkpoint node recovery, stale status detection & re-audit triggers
- [ ] **Phase 4b (W10: 7 days):** Build Cryptographic Key Client (`arx/jscope/keystore.py`), TPM/HSM bindings, local passphrase fallback, and shared registry schema
- [ ] **Phase 4c (W11: 7 days):** Build Level 4 comparison, de Bruijn-index AST matching, semantic equivalence, and proof irrelevance handling
- [ ] Set up Phase 4 Integration Checkpoint (Checkpoint recovery, passphrase decoding, α-equivalence tests, semantic bypasses)

### Phase 5: Web UI, Testing & Hardening (W12–W14: 21 days)
* **Status:** Pending (0%)
- [ ] Renders Web UI DAG visualizer with canvas-based progressive loading and vitals websocket
- [ ] Build container lifecycle manager (`arx/cognition/backends/container_manager.py`)
- [ ] Execute Performance SLA benchmarks & memory footprints
- [ ] Run regression test suite (10 valid and 10 invalid proofs) & configure CI/CD pipeline
- [ ] Set up Phase 5 Integration Checkpoint (End-to-end simulated proof audit test suite execution)
