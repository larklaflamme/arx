# Arx — Math Proof Audit Agent: Implementation Plan v0.5.0

This document outlines the detailed, step-by-step implementation plan for the **Arx Math Proof Audit Agent (v0.5.0)**, based on [ARX_ARCHITECTURE_v0.3.0.md](file:///home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.3.0.md). This v0.5.0 revision addresses and resolves all feedback from the adversarial reviews by AXIOMA, Skye, Thea, and Theoria (47 issues total: 2 critical, 9 medium, 10 minor resolved; 1 critical, 5 medium, 5 low newly addressed).

---

## 1. Executive Summary & Scope

The v0.5.0 release establishes the end-to-end proof audit capability by grounding agent reasoning in the **AXIOMA Substrate**, utilizing **J-Space** for cryptographic reproducibility, routing events via the **Multi-Agent Event Loop**, enforcing safety constraints via the **$\theta$-Rule Engine**, and referencing the **Master Ontology**.

### Backend Coverage Scope & Deferral
The architecture lists 9 verification backends. To manage complexity and external dependencies, the core compute kernel will implement **4 backends**, while the remaining **5 backends** are deferred to v0.5.0:
* **Implemented in v0.4.0:**
  * **Lean4:** Primary interactive theorem prover for formal checkouts.
  * **Z3:** SMT solver for automated first-order logic and algebraic checkouts.
  * **SymPy:** Computer algebra engine for symbolic verification.
  * **mpmath:** High-precision floating-point arithmetic library (running in conjunction with SymPy).
* **Deferred to v0.5.0 (Rationale: Requires specialized interactive theorem prover integrations and heavy computer algebra dependencies):**
  * **Coq, Vampire, SageMath, PARI/GP, GAP.**

---

## 2. Database & Storage Schema Definitions

The Master Ontology uses SQLite as its primary graph store and cache database. It is located at `ontology/data/master_ontology.db`.

```sql
CREATE TABLE nodes (
    id TEXT PRIMARY KEY,
    type TEXT NOT NULL,
    source_type TEXT NOT NULL,
    source_id TEXT NOT NULL,
    label TEXT NOT NULL,
    properties TEXT,  -- JSON string
    provenance TEXT,  -- JSON array of source records
    subgraph TEXT NOT NULL CHECK (subgraph IN ('verify', 'research')),
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    version INTEGER NOT NULL
);

CREATE TABLE edges (
    id TEXT PRIMARY KEY,
    type TEXT NOT NULL,
    source_node_id TEXT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
    target_node_id TEXT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
    confidence REAL NOT NULL CHECK (confidence BETWEEN 0.0 AND 1.0),
    properties TEXT,  -- JSON string
    provenance TEXT,  -- JSON array
    subgraph TEXT NOT NULL CHECK (subgraph IN ('verify', 'research')),
    created_at TEXT NOT NULL,
    version INTEGER NOT NULL,
    -- Epistemic isolation: edge subgraph must match both endpoint subgraphs
    CHECK (
        EXISTS (SELECT 1 FROM nodes n1 WHERE n1.id = source_node_id AND n1.subgraph = subgraph)
        AND EXISTS (SELECT 1 FROM nodes n2 WHERE n2.id = target_node_id AND n2.subgraph = subgraph)
    )
);

-- Merging rules and deduplication mappings between LMFDB, MathGLOSS, and Our Ontology
CREATE TABLE equivalence_mappings (
    id TEXT PRIMARY KEY,
    source_a TEXT NOT NULL,
    object_a TEXT NOT NULL,
    source_b TEXT NOT NULL,
    object_b TEXT NOT NULL,
    confidence REAL NOT NULL CHECK (confidence BETWEEN 0.0 AND 1.0),
    rule TEXT NOT NULL,
    verified_by TEXT NOT NULL,
    ontology_version INTEGER NOT NULL,
    -- Prevent self-mappings and duplicates
    CHECK (source_a != source_b OR object_a != object_b),
    UNIQUE (source_a, object_a, source_b, object_b)
);

-- Version tracking table for content-addressed snapshot hashes
CREATE TABLE version_log (
    version INTEGER PRIMARY KEY AUTOINCREMENT,
    hash TEXT NOT NULL UNIQUE,
    created_at TEXT NOT NULL,
    description TEXT,
    sources_ingested TEXT,  -- JSON array
    node_count INTEGER NOT NULL,
    edge_count INTEGER NOT NULL
);

-- In-memory and local SQLite query cache for LMFDB and external REST queries
CREATE TABLE lmfdb_cache (
    query_key TEXT PRIMARY KEY,
    response_body TEXT NOT NULL,
    is_static INTEGER NOT NULL CHECK (is_static IN (0, 1)),
    created_at TEXT NOT NULL,
    expires_at TEXT, -- Nullable for static data (infinite TTL uses NULL)
    -- Static entries must have NULL expires_at; non-static entries must have a value
    CHECK ((is_static = 1 AND expires_at IS NULL) OR (is_static = 0 AND expires_at IS NOT NULL))
);

-- Indexes for efficient recursive CTE path traversals
CREATE INDEX idx_nodes_type ON nodes(type);
CREATE INDEX idx_nodes_source ON nodes(source_type, source_id);
CREATE INDEX idx_nodes_subgraph ON nodes(subgraph);
CREATE INDEX idx_edges_source_type ON edges(source_node_id, type);
CREATE INDEX idx_edges_target ON edges(target_node_id);
CREATE INDEX idx_edges_subgraph ON edges(subgraph);
-- Partial index for cache eviction queries (non-static entries only)
CREATE INDEX idx_lmfdb_cache_expires ON lmfdb_cache(expires_at) WHERE is_static = 0;
```

### Qdrant Vector Collections
1. **`ontology_concepts`**: Semantic indexes for master ontology concept discovery.
   * Vector size: 384 (`all-MiniLM-L6-v2`)
   * Payload: `id`, `type`, `label`, `source_type`, `subgraph`, `ontology_version`
2. **`theta_rules`**: Rule vectors for real-time consultation checks.
   * Vector size: 384 (`all-MiniLM-L6-v2`)
   * Payload: `rule_id`, `category`, `priority`, `action`, `nl_text`

### Ontology Snapshot Storage
Snapshots of Our Ontology are written as complete content-addressed files.
* **Storage Location:** `/home/ubuntu/arx/ontology/data/snapshots/v{version}_{hash}.sql.gz`
* **Format:** Gzip-compressed SQLite dump output (`.sql.gz`).
* **Loading Mechanism:** Re-audits (Level 3 verification) restore historical graphs using the native SQLite restore command.

---

## 3. Directory & Module Specifications

All code is situated under the project root `/home/ubuntu/arx/`.

### 3.1 Master Ontology Layer (`ontology/` & `neurocore-skill-ontology/`)

#### `ontology/graph/abstract.py`
Defines the database-agnostic interface for graph operations.
```python
class OntologyGraph(ABC):
    @abstractmethod
    async def lookup(self, node_id: str, version: int = 0) -> dict: ...
    @abstractmethod
    async def traverse(self, node_id: str, edge_type: str = None, direction: str = "out", max_depth: int = 1, version: int = 0) -> list[dict]: ...
    @abstractmethod
    async def path(self, start_id: str, end_id: str, max_depth: int = 5) -> list[list[dict]]: ...
    @abstractmethod
    async def search(self, query: str, limit: int = 10, source_filter: list[str] = None) -> list[dict]: ...
```

#### `ontology/graph/sqlite_graph.py`
Concrete implementation of the `OntologyGraph` interface using SQLite and recursive CTEs for traversals.

#### `ontology/api/server.py`
FastAPI REST API server separating mathematical validation (`/api/v1/verify/...`) from speculative research concepts (`/api/v1/research/...`).
* Enforces sub-graph isolation: verification routes reject queries crossing equivalence mappings to the `RESEARCH` sub-graph.

#### `neurocore-skill-ontology/skill.py`
Wraps REST API endpoints into standard NeuroCore skill functions.
* **`ontology_cross_reference(claim: str, context: dict)`**: Evaluates natural language claims. First executes pattern extractors. Falls back to LLM JSON schemas if extractors miss.
* **`ontology_bulk_cross_reference(claims: list[str], context: dict)`**: Processes batches. To eliminate positional alignment failure modes, returns a **dictionary keyed by claim hash** (SHA-256 of normalized claim text):
  ```python
  def ontology_bulk_cross_reference(claims: list[str], context: dict) -> dict[str, dict]: ...
  ```

---

### 3.2 Adapted Cognition Layer (`arx/cognition/`)

#### `arx/cognition/r1_verification.py`
Manages the step status state machine and propagation of statuses: `audited`, `gap-detected`, `circular`, `provenance-broken`, `corroborated`, `ontology-unavailable`, `proven-with-contradiction`, `proven-with-note`, `formalization-uncertain`, `stale`.

#### `arx/cognition/r2_mwm.py`
Manages the Mathematical Working Memory (MWM).
* **Pre-population Script (`arx/cognition/scripts/import_mathlib.py`):** Pre-populated via one-time import script from the latest stable mathlib release at implementation time (targeting `v4.15.0` or later). The tag is parameterized via a configuration constant, not hard-coded.
  * **Pre-import validation:** Checks tag existence, JSON schema version compatibility, and available disk space before beginning import.
  * **Per-record error handling:** Malformed JSON records are skipped with a warning; a single bad record does not abort the entire import of ~282,207 theorems.
  * **Post-import verification:** Compares imported count to expected count; logs any discrepancy.
  * **Resume capability:** If the import fails partway (e.g., at 200K theorems), it resumes from the last successfully imported record rather than restarting from zero.
* **Daily Sync:** Fetches daily tag diffs from the mathlib repository. Mathlib is treated as authoritative: updated theorem signatures overwrite older records, and deleted signatures are marked as deprecated.

#### `arx/cognition/r3_goal_dag.py`
Parses proofs into a dependency DAG. Tracks LLM fallback rates per-audit.

#### `arx/cognition/r4_compute_kernel.py`
Dispatches steps to verifier backends (Lean4, Z3, SymPy, mpmath).

#### `arx/cognition/backends/container_manager.py`
Manages container execution lifecycles. Pins Lean4 (`leanprover/lean4@sha256:...`) and Z3 (`z3prover/z3@sha256:...`) container images by cryptographic hash to guarantee bit-identical verification.

#### `arx/cognition/af_autoformalizer.py`
Translates natural-language proof steps into Lean/Z3 code.
* **Faithfulness Check:** Generates a round-trip natural language description using the LLM and computes semantic similarity using `all-MiniLM-L6-v2`.
  * **Threshold:** Cosine similarity $\ge 0.7$ passes.
  * **Self-Consistency Check:** Generates the description twice. If the cosine similarity between the two descriptions is $< 0.85$, flags the check as unreliable.
  * **Failure Handling:** Retries once with temperature $0.3$. If similarity is still $< 0.7$, stamps the step as `formalization-uncertain`.
  * **Fallback:** If the embedding model is unavailable, skips the similarity check and stamps the step as `formalization-uncertain` with a detailed note.

---

### 3.3 Meta-Cognition Layer (`arx/meta_cognition/`)

#### `arx/meta_cognition/rule_based.py`
Tracks substrate vitals ($\theta$, $\Delta\Phi$, $\psi$, fragmentation) and applies threshold-based resource coupling:
* **$\theta < 2.0$:** Normal operation.
* **$2.0 \le \theta < 4.0$:** Reduces verification effort (skips heuristic backends).
* **$\theta \ge 4.0$:** Pauses new audit tasks.
* **$\psi < 0.3$:** Disables parallelization (runs backends sequentially).

#### `arx/meta_cognition/emergency_stop.py`
Monitors substrate fragmentation ($\Delta\Phi$) at a 1-second polling frequency.
* **ΔΦ > 0.7 Trigger:** Emergency stop signals are broadcast to all active verification processes, the compute kernel, and the event loop.
* **Sequence:** Suspends all active backends, halts Lean/Z3 containers, writes a J-Space checkpoint node, and sets the system state to `paused`.
* **Recovery:** Re-enables operations manually after substrate stabilizes, or automatically after $\Delta\Phi$ drops below 0.5 for 60 consecutive seconds. All emergency stop events are appended to J-Space audit logs.

---

### 3.4 J-Space Context & Reproducibility (`arx/jspace/`)

#### `arx/jspace/scope_stack.py`
Manages nesting scopes (Global $\rightarrow$ Theorem $\rightarrow$ Lemma $\rightarrow$ Step) and performs garbage collection on variables when exiting scope.

#### `arx/jspace/dependency_closure.py`
Computes transitive closures of proof step dependencies and circularity.

#### `arx/jspace/provenance_chain.py`
Structures and appends audit logs, tool executions, and ontology versions to build the verifiable, non-repudiable provenance record.

#### `arx/jspace/reproducibility.py`
Strips comments, canonicalizes whitespaces, applies Unicode NFC normalization, and LaTeX symbol replacements.

#### `arx/jspace/audit_dag.py`
Builds nodes in canonical JSON (RFC 8785). Timestamps and operational parameters are excluded from hashes. Captures the cognitive vitals snapshot (`{theta, delta_phi, psi, fragmentation}`) **before each node generation** by querying the substrate directly when J-Space compiles the node. Checkpoint nodes are written every $N=10$ steps (configurable via `ARX_CHECKPOINT_INTERVAL` environment variable, default 10).

#### `arx/jspace/keystore.py`
Manages the cryptographic signing keys.
* **Hardware Enclave:** Connects to TPM/HSM devices using library wrappers. Private keys never leave the hardware boundary.
* **Passphrase Fallback:** For v0.1 environments, reads the passphrase from a file with `0600` permissions, whose path is configured via the `ARX_KEYSTORE_PASSPHRASE_FILE` environment variable. The passphrase itself is never stored in an environment variable. The keystore reads the file, decrypts the local PKCS#8 PEM file into memory for signing.
* **DAG Signatures:** Computes an Ed25519 signature over the root DAG hash and appends it as a `signature` field to the root node.

---

### 3.5 Multi-Agent Event Loop & Comms (`arx/comms/` & `arx/main.py`)

#### `arx/comms/neurogossip.py`
Bounded thread-safe queue communication (max capacity 1000). Drops ambient broadcasts if queue $> 80\%$ capacity.

#### `arx/comms/agora.py`
Agora transport is available as a fallback communication channel when Neurogossip is unavailable. Deferred to v0.5.0 for full implementation; a stub module is provided in v0.4.0 that logs a warning and returns `connected: false`.

#### `arx/main.py` (Event Loop Core)
Manages scheduling, disengagement, and message filtering:
* **Productivity Scorer:** Computes score:
  $$\text{score} = (w_1 \cdot \text{info\_gain} + w_2 \cdot \text{progress} + w_3 \cdot \text{relevance} + w_4 \cdot \text{depth}) \cdot (1 - \text{penalties})$$
  Weights: $w_1 = 0.4$, $w_2 = 0.3$, $w_3 = 0.2$, $w_4 = 0.1$. Penalties are scaled logarithmic-turn penalties. Relevance searches for $\ge 3$ reasoning keywords.
* **Agent Disconnection Handling:** Expects a heartbeat from registered agents every 30 seconds. If 3 heartbeats are missed, marks the agent as disconnected. If disconnected for $> 5$ minutes, pending requests and conversation threads are flagged as `orphaned`.
* **Vitals WebSocket Endpoint:** Exposes `/api/v1/vitals/stream` for real-time vitals updates. Data format: `{"theta": float, "delta_phi": float, "psi": float, "fragmentation": float, "timestamp": "ISO8601"}`. Update frequency: 1 second (debounced).

---

### 3.6 $\theta$-Rule Engine (`arx/theta_rule/`)

#### `arx/theta_rule/compiler.py`
Compiles plain-text English rules into Rule vectors. Verification similarity check must be $\ge 0.8$.

#### `arx/theta_rule/matcher.py`
Evaluates cosine similarity (MiniLM) and enforces the Anti-Negation Guard.

#### `arx/theta_rule/resolver.py`
Resolves priority conflicts and chaining limits (depth 3 limit, fail-secure DENY).

#### `arx/theta_rule/emergency_stop.py`
* **Kill Switch API:**
  * **Per-rule:** Disables a specific rule by its ID.
  * **Per-category:** Disables all rules in a specific category (e.g. `safety`).
  * **Global:** Disables the rule engine entirely, falling back to a hard-coded safety rule set.
* **Audit Trail:** Appends all kill switch actions, authorization credentials, and reasons to J-Space logs.
* **Authorization:** Only authorized administrative profiles or verified HSM signatures can trigger a kill switch.

---

### 3.7 Profile Configurations (`arx/profiles/`)

#### `arx/profiles/loader.py`
Loads profiles from config files.
* **`auditor` / `verifier` profile:** Stalled steps remain at `corroborated` or lower if the ontology API is offline.
* **`prover` / `explorer` profile:** Audits proceed with warnings if the ontology API is offline.

---

### 3.8 Web UI (`arx/web_ui/`)

Single-page Svelte application on port 8803.
* **DAG Visualizer MVP:** Renders up to 100 proof nodes using canvas-based progressive loading.
  * **Scaling for >100 nodes:** Shows a summary view of the DAG with search/filter capability (e.g., "show only `proven-with-contradiction` nodes", "show steps 50-100") and a "load full DAG" button.
* **Vitals Dashboard:** Real-time updates via WebSockets (connecting to `/api/v1/vitals/stream` on the event loop). Debounces updates at 1 second.

---

### 3.9 Logging & Observability (`arx/observability/`)

#### `arx/observability/logger.py`
Structured JSON logging with per-audit correlation IDs.
* **Log format:** `{"timestamp": "ISO8601", "level": "INFO|WARN|ERROR", "audit_id": "uuid", "component": "theta_rule|event_loop|jspace|...", "message": "...", "context": {...}}`
* **Health check endpoints:** Each component exposes `/health` returning `{"status": "ok|degraded|down", "uptime_seconds": int, "last_error": "..."}`.
* **Metrics to monitor:** Rule engine latency (p50/p95/p99), ontology query latency, DAG build time, queue depth, active audit count, emergency stop count.

---

## 4. Integration Contracts & Protocols

### 4.1 $\theta$-Rule Engine / Event Loop Contract
* **Query Schema (Request):**
  ```json
  {
    "event_type": "conversation.engage",
    "context": {"topic": "Riemann Hypothesis", "confidence": 0.85},
    "agent_id": "arx-01",
    "profile": "auditor",
    "proposed_action": "engage",
    "thread_id": "thread_abc123"
  }
  ```
* **Response Schema:**
  ```json
  {
    "verdict": "ALLOW", -- ALLOW | DENY | OVERRIDE_STATUS | FLAG | ESCALATE | PAUSE | LOG
    "matched_rules": [{"rule_id": "rule-001", "priority": "critical", "action": "ALLOW"}],
    "confidence": 0.95,
    "reason": "Safety check passed."
  }
  ```
* **Crash Fallback (FAIL-SECURE):** Timeout of `500ms`. If the rule engine is unavailable, the caller enforces a **fail-secure DENY**, adds a `rule_engine_unavailable` flag, and halts validation. No unsafe actions are allowed.

---

### 4.2 Event Loop / DAG Builder Contract
* **Event Schema:**
  ```json
  {
    "event_type": "verification.stamp",
    "thread_id": "thread_abc123",
    "turn_number": 12,
    "timestamp": "2026-07-09T01:00:00Z",
    "payload": {
      "step_id": "step-005",
      "status": "proven"
    },
    "agent_id": "arx-01",
    "profile": "auditor",
    "idempotency_key": "step-005_turn-12_arx-01" -- Composite key preventing duplicate entries
  }
  ```
* **Retry Queue:** The event loop maintains a local retry queue. If J-Space is down, it retries sending the event 3 times with exponential backoff before logging a critical error.

---

### 4.3 Ontology Merge Transaction Rollback
All merge operations run inside an SQLite transaction block. Any failure (timeout, validation error, mapping discrepancy) rolls back the transaction.

---

### 4.4 Anti-Negation Guard Mathematical Regex Patterns
* **Natural Negations:** `\b(never|no|not|without|except|unless)\b`
* **Mathematical Prefix Negations:** `\b(non-|dis|un)[a-z]+` (e.g. non-isomorphic, disjoint, unequal)
* **Symbolic Negations:** `[≠¬~]`
* **Verbal Mathematical Negations:** `\b(fails? to|does? not|cannot|can not|there (is|are) no)\b`

---

### 4.5 Contradiction Detection & Priority Derivation

The system detects contradictions by comparing formal proof outputs against ontology cross-reference results. The detection mechanism is specified per priority level:

* **Structural Contradiction (HIGH Priority):** Compares the ontology `type` and structural properties against the formal proof output. If the formal proof proves `rank(E) = 0` and the ontology says `rank(E) = 1`, that is a property contradiction. If the formal proof says `E` is an elliptic curve and the ontology says `E` is a modular form, that is a structural contradiction. Detection: exact field comparison between ontology node properties and formal proof output properties. Propagates `proven-with-contradiction` status.

* **Property Contradiction (NORMAL Priority):** Mismatches in numerical properties where the formal proof and ontology disagree on a computed value. Detection: numerical comparison with configurable tolerance (default: exact match for integer properties, $10^{-6}$ relative tolerance for floating-point properties). Propagates `proven-with-note`.

* **Logical Contradiction (CRITICAL Priority):** Direct logical contradictions ($A \land \neg A$) detected by comparing formal statements: does the proof assert both `P` and `¬P`? Detection: syntactic comparison of formal statements for complementary pairs. Halts verification and routes to human reviewer.

---

### 4.6 Edge Confidence Derivation
$$\text{edge\_confidence} = \min(C_a, C_b) \cdot w_{\text{edge\_type}}$$
Where $w_{\text{edge\_type}}$ is: `1.0` for `equivalent_to` identity edges; `0.9` for structural equivalence mappings; `0.7` for `related_to` associations.

---

### 4.7 Level 4 Consensus Verification Protocol

This protocol implements the technical steps to fulfill the architecture's 2-step escalation protocol (automated resolution → human escalation):

1. **Discovery & Routing:** The event loop maintains a registry of active agents. Neurogossip requests are routed to the least-loaded available agent exposing the `Level 4 audit` capability.
   * **Agent availability check:** Before dispatching, the system checks if a secondary agent is available. If no secondary agent is available, the system logs a warning and skips Level 4 (the audit proceeds at Level 3). The audit is flagged as `partial-level-4`.

2. **Registry:** Progress is tracked in `shared_audit_registry`:
   ```sql
   CREATE TABLE shared_audit_registry (
       audit_id TEXT PRIMARY KEY,
       primary_agent_id TEXT NOT NULL,
       secondary_agent_id TEXT,
       status TEXT NOT NULL CHECK (status IN ('pending', 'processing', 'completed', 'disputed', 'partial-level-4')),
       primary_dag_hash TEXT,
       secondary_dag_hash TEXT,
       last_updated TEXT NOT NULL
   );
   ```

3. **Neurogossip Messaging:**
   * **Request:** `{type: "audit_request", proof_id: "p_100", proof_text: "...", original_dag_hash: "sha256:...", requested_level: 4}`
   * **Response:** `{type: "audit_response", proof_id: "p_100", dag_hash: "sha256:...", status_summary: "corroborated", confidence: 0.90}`
   * **Timeout:** If the secondary agent fails to respond within 1 hour, the primary agent logs a timeout and escalates to human review.
   * **Partial result handling:** If the secondary agent starts the audit but crashes partway through, the primary agent uses whatever DAG nodes were completed and flags the audit as `partial-level-4`.
   * **DAG validation on receipt:** The primary agent verifies the secondary agent's DAG is structurally valid before comparison.

4. **Automated Resolution (Architecture Step 1):** Before escalating to human review, the system attempts automated resolution:
   * **More backends wins:** If one agent used more verification backends (e.g., Lean4 + Z3 + SymPy vs. Lean4 only), the more thorough audit is preferred.
   * **Lower confidence wins (conservative principle):** If both agents used the same backends but produced different confidence scores, the lower confidence score is selected.
   * **One-page summary:** Each agent produces a one-page summary: `{agent_id, verdict, confidence, backends_used, divergence_points: [list of step_ids]}`.
   * **If automated resolution succeeds:** The system writes a signed `consensus_resolution` node with the selected DAG hash and the resolution rule applied.
   * **If automated resolution fails** (agents disagree on fundamental structure): Escalate to human review.

5. **Comparison Algorithm (α-equivalence):** The `dag_comparator` parses formal statements into Lean 4 ASTs and performs a de Bruijn-index-based α-equivalence comparison ($O(N)$ complexity).
   * **Granularity:** Nodes are matched individually, producing a list of node-level mismatches.
   * **Timeout:** If comparison exceeds 5s per node, it falls back to a normalized syntactic string comparison.
   * **Disputes:** If agents disagree on the comparison itself, the dispute is escalated to human review.

6. **Human Escalation (Architecture Step 2):** Side-by-side AST mismatches are presented to the human reviewer via Svelte UI, along with the one-page summary from each agent. The human reviewer may:
   * **Select canonical DAG:** Choose one agent's DAG as authoritative.
   * **Reject all:** Mark the audit as `disputed` and halt.
   * **Trigger re-audit with modified parameters:** Re-run the audit with different backends or parameters.
   * If not resolved within 24 hours, the audit is marked as `disputed` and halted.

#### 4.7.1 Proof Irrelevance Handling
To prevent false alarms from valid alternative proof strategies:
* **Structural Equivalence:** Node-level ASTs match up to variable renaming (α-equivalence).
* **Semantic Equivalence:** Root statements match, and the set of leaf statements (axioms/assumptions) match, ignoring internal decomposition step differences.
* **Protocol:** First checks structural equivalence. If it fails, checks semantic equivalence. If semantic equivalence passes, it reports the audit as "semantically equivalent, different proof strategy" without flagging a warning. If both fail, it escalates to human review.

---

### 4.8 Stale Status Detection & Re-Audit Trigger
When the Master Ontology is updated, the system scans the `version_log` table. Affected steps are flagged as `stale` in J-Space, and the Web UI alerts the operator to trigger a re-audit.

---

### 4.9 System Data Flows

#### 4.9.1 End-to-End Proof Audit Flow
1. **Input Normalization:** `reproducibility.py` strips comments, canonicalizes whitespaces, and normalizes LaTeX.
2. **Goal Decomposition:** `r3_goal_dag.py` parses the normalized proof into individual `ProofStep` nodes.
3. **MWM Lookup:** `r2_mwm.py` checks mathlib signatures for exact matches.
4. **Backend Dispatch:** `r4_compute_kernel.py` dispatches verification tasks to containerized Lean4, Z3, or SymPy backends.
5. **Ontology Cross-Reference:** Queries the Master Ontology API (`server.py`) and cached `lmfdb_cache` results.
6. **Safety Gate Check:** `matcher.py` queries the $\theta$-Rule engine to verify the proposed verification actions.
7. **Status Assignment:** `r1_verification.py` assigns step statuses based on verifier outputs and contradiction priorities.
8. **Report Assembly:** Assembles the final report package: DAG, recipe (audit parameters, backends used, profile), LLM cache tokens, API replay logs, and ontology version snapshot reference.
9. **DAG Compilation:** `audit_dag.py` gathers step results and vitals to build the content-addressed DAG.
10. **Signing:** `keystore.py` signs the root DAG hash via TPM/HSM or passphrase fallback.
11. **Persistence:** Saves the signed DAG, report package, LLM tokens, and API replay logs to disk.
12. **Web UI Rendering:** The Svelte Web UI renders the final report for human review (Phase 5 deliverable).

#### 4.9.2 Rule Enforcement Flow
1. **Action Event:** Component detects proposed action and context.
2. **Rule Query:** Queries the $\theta$-Rule engine with the proposed action schema.
3. **Negation Guarding:** `matcher.py` scans for negation modifiers. If a mismatch is found, it overrides the rule and returns a mismatch.
4. **Semantic Matching:** Searches Qdrant for matching rules using `all-MiniLM-L6-v2` embeddings.
5. **Conflict Resolution:** `resolver.py` resolves priority conflicts and specificity tie-breakers, applying a fail-secure `DENY` for ties.
6. **Depth Limiter:** Stops rule chaining if depth $> 3$, returning `DENY`.
7. **Enforcement:** Enforces the resolver's verdict (`ALLOW` or `DENY`) on the calling component.

---

## 5. Verification Levels

The system supports four distinct verification levels:
* **Level 1 (Structural):** Validates J-Space DAG well-formedness and recalculates node hashes (excluding timestamps and operational variables).
* **Level 2 (Spot-Check):** Re-runs a random 10% sample of proof steps against cached LLM outputs and API replay logs. Triggered on-demand or on suspicion.
* **Level 3 (Full Re-Audit):** Complete DAG re-execution using cached API/LLM logs. The resulting root hash must match the original. Triggered by stale status flags or operator requests.
* **Level 4 (Independent Re-Audit):** Secondary agent performs a live re-audit. The resulting DAG is compared to the original for $\alpha$-equivalence and semantic equivalence. Triggered on final checkouts.

---

## 6. Phase-by-Phase Timeline & Deliverables

The implementation schedule is aligned with the architecture's 14-week (98 days) plan. **Note:** The week boundaries have shifted from the architecture's original allocation: Phase 4 is extended by 7 days (from 14 to 21) to accommodate the Level 4 consensus protocol, with Phase 5 reduced by a corresponding 7 days (from 28 to 21). Total timeline remains 98 days.

* **Phase 0: Foundations, Ingestion & Normalization (W1–W2: 14 days)**
  * SQLite schemas (including mapping and version log tables), Our Ontology ingestion, sub-graph partitioning.
  * MWM database pre-population (`import_mathlib.py`) from the latest stable mathlib release.
  * Input normalization pipeline (`reproducibility.py`).
  * **Integration Checkpoint:** SQLite schema validation, normalization text mappings, and MWM lookup checks.

* **Phase 1: Event Loop, Bounded Queues & DAG Builder (W3–W4: 14 days)**
  * Neurogossip-v3 bounded queue (max capacity 1000) and backpressure dropping rules.
  * Meaningless message filters, context-aware affirmations, and agent disconnection handling.
  * Event loop / DAG integration events, composited idempotency keys, and retry queues.
  * DAG node schemas, parent chaining, and Level 1 structural verification.
  * **Integration Checkpoint:** Heartbeat timeouts, queue backpressure dropping, and Level 1 structural verification.

* **Phase 2: $\theta$-Rule Engine & Safety Gates (W5–W6: 14 days)**
  * Qdrant rule store integration with MiniLM embeddings.
  * Rule compiler with semantic round-trip validation similarity checks.
  * Anti-Negation Guard mathematical regex patterns and resolver priority logic.
  * Rule engine emergency stop kill switches, authorization checks, and J-Space logging.
  * **Integration Checkpoint:** Rule compiler round-trip checks, negation guard overrides, and kill switch propagation.

* **Phase 3: Ontology Skill, Parser & AF Faithfulness Check (W7–W8: 14 days)**
  * Bulk cross-reference returning hash-keyed dicts.
  * Two-tier claim parsing and fallback rate monitoring.
  * Autoformalizer faithfulness check, embedding models, and fallback paths.
  * **Integration Checkpoint:** Hash-keyed dict return formats, two-tier parse fallback rates, and AF faithfulness checks.

* **Phase 4: Reproducibility logs, Checkpoints & Consensus (W9–W11: 21 days)**
  * **Phase 4a (W9: 7 days):** API mock replay logs, checkpoint nodes crash recovery, stale status detection and triggers.
  * **Phase 4b (W10: 7 days):** Cryptographic signing keys (keystore, passphrase fallback) and shared audit registry schema.
  * **Phase 4c (W11: 7 days):** TPM/HSM integration, agent discovery/routing, Level 4 comparison, de Bruijn-index AST matching, semantic equivalence, and proof irrelevance handling.
  * **Integration Checkpoint:** Checkpoint recovery, private key passphrase decoding, α-equivalence tests, and semantic equivalence bypasses.

* **Phase 5: Web UI, Testing & Hardening (W12–W14: 21 days)**
  * Svelte visualizer MVP (progressive canvas loading, search/filter capability), vitals WebSocket updates, and human review handoff UI.
  * Container manager backend lifecycle.
  * Performance benchmarking, regression test suite execution, and CI/CD pipelines.
  * **Integration Checkpoint:** End-to-end simulated proof audit test suite execution.

---

## 7. Performance Benchmarks & Testing Strategy

### Performance SLAs
* **Single ontology lookup:** $< 50\text{ms}$ (p95).
* **Ontology traversal (depth 5):** $< 200\text{ms}$ (p95).
* **Bulk cross-reference (100 claims):** $< 5\text{s}$.
* **Full audit (100 steps, 3 backends per step):** $< 10\text{ minutes}$.
* **DAG structural verification (Level 1):** $< 1\text{s}$ for 1000 nodes.
* **Maximum Memory Footprint:** $< 2\text{GB}$ for a 1000-node DAG.

### Testing Strategy
* **Unit Tests:** Minimum 80% code coverage.
* **CI/CD Integration:** Automated tests on branch push.
* **Regression Suite:** 10 known-valid and 10 known-invalid mathematical proofs.

---

## 8. Verification & Acceptance Gates

To certify the v0.5.0 release, the system must pass these concrete acceptance gates:

| ID | Component | Acceptance Test Criteria |
|---|---|---|
| **A-ONT-1** | Master Ontology | Ingest 100 entries of Our Ontology and MathGLOSS. Verify a traversal from `VERIFY` to `RESEARCH` is blocked. Verify that an edge connecting a `VERIFY` node to a `RESEARCH` node is rejected by the database CHECK constraint. |
| **A-XREF-2**| Skill Ontology | Bulk cross-reference of 10 claims returns a hash-keyed dict mapping hash to result. Simulating a timeout on 1 query fails only that claim, keeping other 9 keys intact. |
| **A-EVL-3** | Event Loop | Bare "yes" answering a pending request is allowed. Ambient broadcasts dropped if inbox has > 800 items. |
| **A-EVL-4** | Event Loop | Disengages from conversation after 3 turns with MiniLM score $<0.15$. Heartbeat failure marks agent disconnected after 90s. |
| **A-RULE-5**| Rule Engine | Mismatched safety rule negation modifier (e.g. rule: `never stamp`, event: `stamp`) triggers `DENY` despite cosine similarity $> 0.95$. |
| **A-RULE-6**| Rule Engine | Infinite rule loop (depth $>3$) results in fail-secure `DENY` and escalates to human review. |
| **A-RULE-7**| Rule Engine | Rule compiler fails deployment if a vector fails to decode back to natural language with similarity $< 0.8$. |
| **A-RULE-8**| Rule Engine | Triggering rule engine kill switch successfully disables target rules (per-rule, per-category, or global) and writes credentials to J-Space logs. |
| **A-JSPA-9**| J-Space | Level 1 structural verification of a 10-step audit passes in under 1s. Re-running validation with a modified timestamp yields identical node hashes. |
| **A-JSPA-10**| J-Space | Simulating an agent crash at step 13 recovers state from checkpoint 1 (step 10) and completes audit without repeating steps 1–10. |
| **A-JSPA-11**| J-Space | Detects transitive circular dependency (e.g. A $\rightarrow$ B $\rightarrow$ C $\rightarrow$ A) and assigns `circular` status. |
| **A-COG-12**| Cognition | Autoformalizer fails faithfulness check and stamps `formalization-uncertain` if the generated Lean code's round-trip description similarity is $< 0.7$. |
| **A-VER-13**| Verification | Level 4 verification successfully identifies two α-equivalent formal statements differing in variable names. Correctly classifies **50 α-equivalent and 50 non-α-equivalent pairs** (100% accuracy) containing edge cases (mutually recursive definitions, shadowed binders, dependent types, universe polymorphism — at least 20% of the test set). |
| **A-VER-14**| Verification | Level 4 comparison identifies 3 structurally different but semantically equivalent proofs and reports them as "semantically equivalent, different proof strategy" without triggering human review flags. |
| **A-VER-15**| Verification | Consensus protocol escalates structural disagreement to human reviewer, and writes a GPG/Ed25519 signed `consensus_resolution` node upon decision. Automated resolution step (more backends wins, lower confidence wins) is tested with 5 scenarios. |
| **A-WUI-16**| Web UI | Web dashboard renders vitals and color-codes a `proven-with-contradiction` node in magenta. |
| **A-SYS-17**| System | High-fragmentation vitals ($\Delta\Phi > 0.7$) trigger an immediate emergency stop, halting all active verifications, writing a checkpoint DAG node, and logs recovery status. |
| **A-SYS-18**| System | Updating the Master Ontology successfully triggers `stale` flags on all historical audits matching the previous version hash. |
| **A-SYS-19**| System | Keystore correctly signs the root DAG hash via passphrase fallback (`ARX_KEYSTORE_PASSPHRASE_FILE`). The signature is verifiable with the corresponding public key. |
| **A-SYS-20**| System | Substrate vitals coupling: $\theta \ge 4.0$ pauses new audit tasks. $\psi < 0.3$ disables parallelization. |
| **A-SYS-21**| System | Level 2 verification re-runs a random 10% sample of proof steps against cached LLM outputs. Level 3 verification re-executes the full DAG and the root hash matches the original. |
| **A-SYS-22**| System | LLM caching: re-verification uses cached tokens, not live LLM calls. API replay logs are correctly captured and replayed. |
| **A-SYS-23**| System | Transitive timeout extension: parent request timeouts are paused when child requests are pending on human review. |
| **A-SYS-24**| System | Equivalence mappings are correctly ingested and applied. Merge failures roll back correctly. Ontology snapshots are correctly saved and loaded. |

---

## 9. Open Questions & Risks

The following open questions from Architecture §11 are addressed as follows:
* **LMFDB MCP Server Maturity:** Evaluated during Phase 0; if unstable, the compute kernel falls back to querying the cached SQLite/LMFDB queries.
* **Qdrant Vector Drift:** Addressed by scheduling monthly vector re-indexing scripts and running semantic similarity calibration checks in Phase 5.
* **Review Handoff UI:** Fully specified as a side-by-side AST comparison pane with diff highlights, to be delivered in Phase 5.
* **Key Revocation Propagation:** Keystore client handles signed revocation lists appended directly to the database version log records.
