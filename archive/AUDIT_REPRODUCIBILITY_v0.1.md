# Arx — Audit Reproducibility & Verifiability v0.1

> **Cryptographic DAG, Reproducibility Recipe, and Verification Protocol for Audit Reports**
>
> **Status:** Draft
> **Author:** Skye Laflamme
> **Date:** 2026-07-09
> **Depends on:** ARX_ARCHITECTURE_v0.2.0.md, MASTER_ONTOLOGY_v0.2.0.md, MULTI_AGENT_EVENT_LOOP_v0.2.0.md, THETA_RULE_INTEGRATION_v0.2.0.md

---

## §1: Principles

### 1.1 Core Insight

An audit is a **deterministic function** of five inputs:

```
Audit Result = f(Proof, Ontology_Snapshot, Rule_Set, Backend_Versions, Agent_Config)
```

If any of these inputs change, the result may change. **Reproducibility** means: given the same five inputs, the system produces the same result. **Verifiability** means: a third party can check that the result has not been tampered with, and can re-run any subset of steps to confirm correctness.

### 1.2 Design Principles

1. **Content-addressable storage.** Every artifact is stored by its cryptographic hash. No naming conflicts, no versioning ambiguity, free deduplication, tamper detection.

2. **Hash-chained DAG.** Every audit step carries a hash of its content plus the hashes of all its parent steps. The root hash commits to the entire audit.

3. **Environment capture.** Every audit records the exact environment (ontology snapshot, rule set, backend versions, agent configuration) needed to reproduce it.

4. **Nonce for non-determinism.** Non-deterministic steps (LLM calls) capture the random seed used, enabling deterministic replay.

5. **Multi-level verification.** A third party can verify at multiple levels of cost and trust: structural integrity, spot-check, full re-audit, or independent re-audit.

6. **Cross-signing for high-stakes audits.** A second agent can independently verify and cross-sign the report, providing defense against a single compromised agent or faulty backend.

7. **Non-repudiation.** The audit report carries cryptographic evidence that cannot be repudiated by the generating agent.

---

## §2: DAG Structure

### 2.1 Node Definition

Every node in the audit DAG is a content-addressable record:

```yaml
AuditNode:
  id: string                    # SHA256(content + parents + backend + timestamp + nonce)
  type: enum {
    input_proof,                # The original proof being audited
    decomposition,              # The proof decomposed into steps
    verification_step,          # A single step verified by a backend
    cross_reference,            # An ontology cross-reference lookup
    rule_enforcement,           # A θ-Rule enforcement decision
    status_assignment,          # A step receiving its audit status
    aggregation,                # Multiple steps aggregated into a summary
    report_root                 # The final report root node
  }
  content: object               # Type-specific payload (see §2.2)
  parents: List[string]         # SHA256 hashes of parent nodes
  backend: {                    # What produced this node
    name: string,               # "lean", "z3", "lmfdb", "ontology", "llm", "theta_rule"
    version: string,            # Semantic version or image hash
    parameters: object           # Parameters passed to the backend
  }
  timestamp: int                # Unix timestamp (seconds since epoch)
  nonce: string | null          # Random seed for non-deterministic steps (null if deterministic)
  signature: string | null      # Optional: signed by the agent's key
```

### 2.2 Node Content by Type

**input_proof:**
```yaml
content:
  source: "user-uploaded" | "lmfdb" | "arxiv" | "api"
  original_hash: "sha256:..."
  format: "latex" | "lean" | "plaintext" | "pdf"
  size_bytes: 12345
```

**decomposition:**
```yaml
content:
  method: "llm" | "rule-based" | "hybrid"
  step_count: 47
  steps:
    - step_id: "step-001"
      natural_language: "By the Mean Value Theorem..."
      formal_statement: "∀ a b, a < b → ∃ c ∈ (a,b), f'(c) = (f(b)-f(a))/(b-a)"
      dependencies: ["step-000"]
      premises: ["mean_value_theorem"]
```

**verification_step:**
```yaml
content:
  step_id: "step-012"
  backend: "lean"
  input: "theorem example: 1 + 1 = 2 := by native_dec_trivial"
  output: "proven" | "failed" | "timeout" | "error"
  output_detail: "type-checked in 0.23s"
  output_hash: "sha256:..."
  duration_ms: 230
```

**cross_reference:**
```yaml
content:
  claim: "elliptic curve 37.a1 has rank 0"
  source: "lmfdb"
  query: "EllipticCurve/37.a1"
  result: "rank: 0, conductor: 37, torsion: Z3"
  confidence: 0.95
  ontology_snapshot: "sha256:..."
```

**rule_enforcement:**
```yaml
content:
  rule_id: "audit-003"
  rule_hash: "sha256:..."
  action: "OVERRIDE_STATUS" | "DENY" | "FLAG" | "ESCALATE"
  context: "step-012 cross-reference confidence 0.6 < 0.7 threshold"
  result: "status overridden to unverified"
```

**status_assignment:**
```yaml
content:
  step_id: "step-012"
  status: "proven" | "corroborated" | "unverified" | "contradicted" | "proven-with-contradiction"
  confidence: 0.85
  evidence:
    - node_id: "sha256:..."    # verification_step node
    - node_id: "sha256:..."    # cross_reference node
    - node_id: "sha256:..."    # rule_enforcement node
```

**aggregation:**
```yaml
content:
  step_count: 47
  status_counts:
    proven: 35
    corroborated: 7
    unverified: 3
    contradicted: 2
  overall: "conditional"       # "proven" | "conditional" | "contradicted" | "inconclusive"
```

**report_root:**
```yaml
content:
  audit_id: "arx-audit-2026-07-09-001"
  proof_hash: "sha256:..."
  ontology_snapshot: "sha256:..."
  rule_set_hash: "sha256:..."
  agent_profile: "auditor-v0.2"
  backends:
    lean: "sha256:..."
    z3: "sha256:..."
    lmfdb: "sha256:..."
    ontology: "sha256:..."
    llm: "claude-4-sonnet@sha256:..."
  aggregation_node: "sha256:..."   # hash of the aggregation node
  generated_by: "arx-agent-v0.2.0"
  generated_at: "2026-07-09T14:30:00Z"
```

### 2.3 DAG Topology

```
                    ┌─────────────────────┐
                    │   Input Proof (P)    │
                    │   id = H(content)   │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Decomposition (D)  │
                    │  id = H(D, H(P))    │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼──────┐ ┌──────▼───────┐ ┌──────▼───────┐
    │ Step 1 Verify  │ │ Step 2 Verify│ │ Step 3 Verify│
    │ id = H(S1,     │ │ id = H(S2,   │ │ id = H(S3,   │
    │   H(D), B_v,   │ │   H(D), B_v, │ │   H(D), B_v, │
    │   O_t, R_t)    │ │   O_t, R_t)  │ │   O_t, R_t)  │
    └────────┬───────┘ └──────┬───────┘ └──────┬───────┘
             │                │                │
    ┌────────▼───────┐ ┌──────▼───────┐ ┌──────▼───────┐
    │ Step 1 X-Ref   │ │ Step 2 X-Ref │ │ Step 3 X-Ref │
    │ id = H(X1,     │ │ id = H(X2,   │ │ id = H(X3,   │
    │   H(S1), O_t)  │ │   H(S2), O_t)│ │   H(S3), O_t)│
    └────────┬───────┘ └──────┬───────┘ └──────┬───────┘
             │                │                │
    ┌────────▼───────┐ ┌──────▼───────┐ ┌──────▼───────┐
    │ Step 1 Rule    │ │ Step 2 Rule  │ │ Step 3 Rule  │
    │ id = H(R1,     │ │ id = H(R2,   │ │ id = H(R3,   │
    │   H(X1), R_t)  │ │   H(X2), R_t)│ │   H(X3), R_t)│
    └────────┬───────┘ └──────┬───────┘ └──────┬───────┘
             │                │                │
    ┌────────▼───────┐ ┌──────▼───────┐ ┌──────▼───────┐
    │ Step 1 Status  │ │ Step 2 Status│ │ Step 3 Status│
    │ id = H(St1,    │ │ id = H(St2,  │ │ id = H(St3,  │
    │   H(R1))       │ │   H(R2))     │ │   H(R3))     │
    └────────┬───────┘ └──────┬───────┘ └──────┬───────┘
             │                │                │
             └────────────────┼────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │  Aggregation (A)   │
                    │  id = H(A, H(St1), │
                    │    H(St2), ...)    │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Report Root (R)  │
                    │  id = H(R, H(A),   │
                    │    O_t, R_t, B_v,  │
                    │    Agent_Config)    │
                    └────────────────────┘
```

**Key property:** The root hash `H(R)` is a commitment to the *entire audit* — every step, every backend call, every cross-reference, every configuration parameter. Change anything, and the root hash changes.

### 2.4 Hash Computation

Each node's `id` is computed as:

```
id = SHA256(concat(
    type,
    canonical_json(content),
    canonical_json(parents_sorted),
    canonical_json(backend),
    str(timestamp),
    nonce || ""
))
```

Where:
- `canonical_json` produces a deterministic JSON serialization (sorted keys, no whitespace)
- `parents_sorted` is the list of parent hashes sorted lexicographically
- `nonce || ""` means the nonce string if present, empty string if null

This ensures the hash is deterministic across implementations and platforms.

---

## §3: Reproducibility Recipe

### 3.1 Recipe Format

Every audit report carries a reproducibility recipe that captures the exact environment:

```yaml
reproducibility_recipe:
  audit_id: "arx-audit-2026-07-09-001"
  root_hash: "sha256:abc123def456..."
  
  inputs:
    proof_hash: "sha256:..."
    proof_source: "user-uploaded"
    proof_format: "latex"
    
  environment:
    ontology_snapshot_hash: "sha256:..."
    rule_set_hash: "sha256:..."
    agent_profile: "auditor-v0.2"
    agent_version: "arx-agent-v0.2.0"
    
  backends:
    lean:
      version: "4.15.0"
      image: "leanprover/lean4@sha256:abc..."
      parameters: {}
    z3:
      version: "4.13.0"
      image: "z3prover/z3@sha256:def..."
      parameters: { timeout: 30 }
    lmfdb:
      version: "2026-07-01"
      cache_hash: "sha256:ghi..."
      parameters: { endpoint: "https://lmfdb.org/api" }
    ontology:
      version: "v0.2.0"
      snapshot_hash: "sha256:jkl..."
      parameters: {}
    llm:
      model: "claude-4-sonnet"
      provider: "anthropic"
      parameters:
        temperature: 0.0
        max_tokens: 4096
        seed: 42
  
  steps:
    total: 47
    verified: 42
    unverified: 3
    contradicted: 2
    duration_seconds: 847
```

### 3.2 Environment Capture

At the start of every audit, the system captures:

1. **Ontology snapshot hash** — the content-addressable hash of the ontology at that moment
2. **Rule set hash** — the hash of all active rules (including their compiled vectors)
3. **Backend versions** — the exact version and image hash of every backend
4. **Agent configuration** — the profile, parameters, and version of the auditing agent
5. **System time** — the Unix timestamp at audit start

This capture is itself a DAG node (type: `environment_snapshot`) that becomes a parent of the report root.

### 3.3 Backend Pinning

Every backend is pinned by Docker image hash, not by tag:

```
leanprover/lean4@sha256:a1b2c3d4e5f6...
z3prover/z3@sha256:1a2b3c4d5e6f...
```

This ensures bit-identical reproducibility. A tag like `4.15.0` can be updated by the maintainer; a hash is immutable.

**For v0.1:** Use Docker images pinned by hash. If a backend is not containerized (e.g., LMFDB API), capture the API version and cache the response.

**For v0.2+:** Maintain a local registry of approved backend images with their hashes.

### 3.4 Nonce Management

Non-deterministic steps (LLM calls) capture the random seed used:

```yaml
backend:
  name: "llm"
  version: "claude-4-sonnet@sha256:..."
  parameters:
    temperature: 0.0
    seed: 42
```

**For v0.1:** Use `temperature=0.0` for all LLM calls. This is deterministic in most LLM APIs and eliminates the need for nonce tracking.

**For v0.2+:** Allow higher temperatures with explicit nonce tracking. The nonce is generated at call time and stored in the DAG node.

### 3.5 Reproducibility Guarantee

With the recipe, a third party can:

1. Fetch the ontology snapshot by hash from the snapshot archive
2. Fetch the rule set by hash from the rule archive
3. Pull the exact backend images by hash
4. Re-run the audit with the same inputs and configuration
5. Compare the resulting root hash

If the root hashes match, the audit is **fully reproducible**.

---

## §4: Verification Protocol

### 4.1 Four Levels of Verification

A third party can verify an audit report at four levels, trading cost for trust:

#### Level 1: Structural Verification (cheap, fast)

Verifies the integrity of the DAG without re-running any backends:

1. Check that the DAG is well-formed (no cycles, all parent hashes exist)
2. Check that each node's `id` matches `SHA256(content + parents + backend + timestamp + nonce)`
3. Check that the root hash is consistent with the leaf hashes
4. Check that the reproducibility recipe matches the report root's content

**Cost:** O(N) where N is the number of nodes. No backend calls needed.
**Guarantee:** The report has not been tampered with since generation.

#### Level 2: Spot-Check Verification (medium cost)

Verifies a random sample of steps by re-running the backends:

1. Perform Level 1 verification first
2. Select N random steps from the DAG (configurable, default 10% of steps, minimum 3)
3. For each selected step, re-run the backend with the captured parameters
4. Verify that the output hash matches the stored node
5. Report the fraction of spot-checks that passed

**Cost:** O(M) where M is the number of spot-checked steps. Requires backend access.
**Guarantee:** With high probability, the audit was correctly executed. (If 10% of steps are spot-checked and all pass, the probability of a undetected error is < 0.1^N.)

#### Level 3: Full Re-Audit (expensive, definitive)

Re-runs the entire audit from scratch:

1. Perform Level 1 verification first
2. Fetch the original proof by hash
3. Fetch the ontology snapshot by hash
4. Fetch the rule set by hash
5. Spin up the same backend versions
6. Re-run the entire audit pipeline
7. Compare the resulting root hash

**Cost:** O(Full audit). Requires the same computational resources as the original audit.
**Guarantee:** The audit is fully reproducible. If the root hashes match, the audit is verified.

#### Level 4: Independent Re-Audit (most expensive, most trustworthy)

A different agent (different implementation, different backends) re-audits the same proof:

1. Perform Level 1 verification first
2. A second agent (e.g., Thea) independently audits the same proof
3. The second agent produces its own DAG and report
4. Compare the two reports for consistency

**Cost:** 2x the original audit cost.
**Guarantee:** The audit is not dependent on a specific implementation or agent. Discrepancies between the two reports indicate a potential issue in one or both audits.

### 4.2 Verification Tool

A standalone CLI tool for third-party verification:

```
arx-verify <report.yaml> [--level 1|2|3|4] [--spot-check N] [--agent thea]
```

**Level 1 (default):** Structural verification only. No network access needed.
**Level 2:** Spot-check verification. Requires backend access (Docker or API).
**Level 3:** Full re-audit. Requires significant computational resources.
**Level 4:** Independent re-audit. Requires a second agent to be available.

**Output:**
```yaml
verification_result:
  report_id: "arx-audit-2026-07-09-001"
  level: 2
  structural: PASS                    # Level 1 check
  spot_checks:
    total: 5
    passed: 5
    failed: 0
    sample_rate: 0.106                # 10.6% of steps checked
  overall: PASS                       # All checks passed
  warnings: []                        # Any non-fatal issues
```

### 4.3 Verification Without Backend Access

A third party may not have access to the same backends (e.g., they don't have Lean installed). In this case:

- **Level 1** is always available (structural verification needs no backends)
- **Level 2** is available if the verifier has the backends, or if the audit report includes **backend output hashes** that can be checked against a public registry
- **Level 3** requires full backend access
- **Level 4** requires a second agent

**For v0.1:** Publish backend output hashes to a public transparency log. A verifier can check that the output hash exists in the log without re-running the backend.

---

## §5: Report Format

### 5.1 Full Report Schema

```yaml
audit_report:
  version: "0.1"
  
  metadata:
    audit_id: "arx-audit-2026-07-09-001"
    generated_by: "arx-agent-v0.2.0"
    generated_at: "2026-07-09T14:30:00Z"
    root_hash: "sha256:abc123def456..."
    duration_seconds: 847
    
  summary:
    proof:
      title: "Riemann Hypothesis, Chapter 3, Section 2"
      author: "L. Laflamme"
      hash: "sha256:..."
    total_steps: 47
    statuses:
      proven: 35
      corroborated: 7
      unverified: 3
      contradicted: 2
    overall: "conditional"
    confidence: 0.82
    
  reproducibility_recipe:
    # As specified in §3.1
    
  dag:
    # For small audits (< 1000 nodes): full DAG inline
    # For large audits: root hash + link to external DAG file
    root: "sha256:abc..."
    node_count: 312
    depth: 7
    # Full DAG omitted for large audits; stored separately
    
  provenance_summary:
    - step_id: "step-012"
      status: "proven"
      backends_used: ["lean", "lmfdb"]
      ontology_references: ["elliptic_curve_37_a1", "L_function_conductor_37"]
      rule_references: ["audit-003", "audit-007"]
      node_hash: "sha256:..."
      duration_ms: 1230
    - step_id: "step-013"
      status: "contradicted"
      backends_used: ["lean", "ontology"]
      ontology_references: ["L_function_conductor_41"]
      rule_references: ["audit-004", "audit-009"]
      node_hash: "sha256:..."
      duration_ms: 3450
      contradiction_detail:
        formal_proof: "proven"
        ontology_evidence: "contradicts (confidence 0.92)"
        resolution: "escalated to human review"
    
  signatures:
    - agent: "arx-agent-v0.2.0"
      key_id: "arx-master-key-001"
      signature: "base64-encoded-signature..."
      signed_fields: ["root_hash", "generated_at", "summary.statuses"]
    - agent: "thea-agent-v0.2.0"     # Optional: cross-signed
      key_id: "thea-master-key-001"
      signature: "base64-encoded-signature..."
      signed_fields: ["root_hash", "generated_at", "summary.statuses"]
```

### 5.2 Compact Report Format

For quick sharing (e.g., in a Telegram message or web UI):

```yaml
audit_report:
  id: "arx-audit-2026-07-09-001"
  root_hash: "sha256:abc..."
  summary:
    proof: "RH Ch3 §2"
    steps: 47
    proven: 35
    contradicted: 2
    overall: "conditional"
  recipe:
    ontology: "sha256:..."
    rules: "sha256:..."
    backends: ["lean:4.15.0", "z3:4.13.0", "lmfdb:2026-07-01"]
  signatures: 2
```

The full report (with DAG and provenance) is stored separately and referenced by the root hash.

### 5.3 Report Storage

Reports are stored by root hash in a content-addressable store:

```
/arx/reports/
  sha256:abc.../
    report.yaml              # Full report
    dag/                     # Full DAG (one file per node, named by hash)
      sha256:node1...
      sha256:node2...
      ...
    recipe.yaml              # Reproducibility recipe (extracted for convenience)
    signatures/              # Signatures
      arx-master-key-001.sig
      thea-master-key-001.sig
```

**For v0.1:** Store reports as gzipped JSON files in the filesystem.
**For v0.2+:** Store reports in a content-addressable object store (IPFS or similar).

---

## §6: Ontology Snapshots

### 6.1 Snapshot Format

An ontology snapshot is a point-in-time capture of the entire ontology:

```yaml
ontology_snapshot:
  version: "0.1"
  snapshot_hash: "sha256:..."
  parent_hash: "sha256:..."           # Previous snapshot hash (for incremental diffs)
  created_at: "2026-07-09T14:00:00Z"
  source_hashes:
    lmfdb: "sha256:..."               # Hash of LMFDB data at time of snapshot
    mathgloss: "sha256:..."           # Hash of MathGLOSS data at time of snapshot
    our_ontology: "sha256:..."        # Hash of our ontology at time of snapshot
  nodes:
    - id: "elliptic_curve_37_a1"
      type: "mathematical_object"
      source: "lmfdb"
      confidence: 0.95
      properties:
        conductor: 37
        rank: 0
        torsion: "Z3"
      edges:
        - type: "instance_of"
          target: "elliptic_curve"
          confidence: 1.0
        - type: "has_lfunction"
          target: "L_function_conductor_37"
          confidence: 0.95
    - id: "elliptic_curve"
      type: "mathematical_structure"
      source: "mathgloss"
      confidence: 0.85
      properties:
        qid: "Q..."
      edges:
        - type: "subclass_of"
          target: "algebraic_variety"
          confidence: 0.9
  equivalence_mappings:
    - source_a: "lmfdb"
      object_a: "37.a1"
      source_b: "mathgloss"
      object_b: "QID:Q..."
      confidence: 0.95
      rule: "hand-curated, Phase 0.5"
```

### 6.2 Snapshot Lifecycle

1. **Creation:** A snapshot is created at the start of every audit (or on demand)
2. **Storage:** Snapshots are stored by hash in a content-addressable store
3. **Retention:** Snapshots are retained indefinitely (they are needed for re-audit)
4. **Pruning:** Only the hash is needed to verify a snapshot; the full content can be pruned if storage is constrained, but the hash must remain in the index

### 6.3 Incremental Snapshots

For efficiency, snapshots can be stored as incremental diffs from a base snapshot:

```
snapshot_v1 (base): full dump
snapshot_v2: diff from v1
snapshot_v3: diff from v2
...
snapshot_full: periodic full dump (weekly)
```

A re-audit that references snapshot_v15 needs to reconstruct it by applying diffs from the nearest full dump. This is more storage-efficient but adds reconstruction cost.

**For v0.1:** Full dumps only. Storage is cheap and audit volume is low.
**For v0.2+:** Incremental diffs with periodic full dumps.

---

## §7: Cross-Signing Protocol

### 7.1 When Cross-Signing Is Used

Cross-signing is optional and used for:
- High-stakes audits (regulatory, financial, legal)
- Audits where the proof is contested
- Audits where the result has significant consequences

The audit profile determines whether cross-signing is required:
- `auditor` profile: cross-signing optional
- `verifier` profile: cross-signing recommended
- `reviewer` profile: cross-signing required

### 7.2 Cross-Signing Flow

1. **Primary agent** (Arx) completes the audit and produces the DAG
2. **Primary agent** shares the DAG root hash and reproducibility recipe with the secondary agent (Thea)
3. **Secondary agent** either:
   - **Option A (lightweight):** Verifies the DAG structure and re-runs a spot-check (Level 2 verification)
   - **Option B (full):** Re-runs the entire audit independently (Level 4 verification)
4. **Secondary agent** produces a cross-signature on the root hash
5. **Primary agent** appends the cross-signature to the report

### 7.3 Cross-Signature Format

```yaml
signatures:
  - agent: "arx-agent-v0.2.0"
    key_id: "arx-master-key-001"
    signature: "base64..."
    verification_level: 3            # Full re-audit
    signed_fields: ["root_hash", "generated_at", "summary.statuses"]
  - agent: "thea-agent-v0.2.0"
    key_id: "thea-master-key-001"
    signature: "base64..."
    verification_level: 2            # Spot-check
    signed_fields: ["root_hash"]
    spot_check_results:
      total: 5
      passed: 5
      failed: 0
```

### 7.4 Key Management

Each agent has a cryptographic key pair for signing:

- **Private key:** Stored securely, never shared
- **Public key:** Published in a key directory, verifiable by third parties
- **Key rotation:** Keys are rotated periodically (every 6 months for v0.1)
- **Revocation:** Compromised keys are revoked via a revocation list

**For v0.1:** Use Ed25519 keys. Simple, fast, well-understood.
**For v0.2+:** Support hardware-backed keys for higher assurance.

---

## §8: Integration with Arx Architecture

### 8.1 New Components

The reproducibility system adds three new components to the Arx architecture:

1. **DAG Builder** — Constructs the audit DAG as steps are executed. Each component (verification layer, event loop, θ-Rule engine) emits DAG nodes to the builder.

2. **Snapshot Manager** — Creates and manages ontology snapshots. Provides the current snapshot hash to the DAG builder at audit start.

3. **Report Generator** — Assembles the final report from the DAG, reproducibility recipe, and signatures. Produces the report in YAML format.

### 8.2 Integration Points

| Arx Component | Integration with Reproducibility System |
|---|---|
| **Verification Layer** | Emits `verification_step` nodes to DAG Builder after each backend call |
| **Event Loop** | Emits `decomposition` and `aggregation` nodes; captures conversation turns that produce audit decisions |
| **θ-Rule Engine** | Emits `rule_enforcement` nodes after each rule check |
| **Ontology Layer** | Provides snapshot hash at audit start; emits `cross_reference` nodes |
| **Profile System** | Provides agent configuration for the reproducibility recipe |
| **Report Generator** | New component; assembles final report from all DAG nodes |

### 8.3 Data Flow

```
Audit Start
    │
    ├──► Snapshot Manager: capture ontology snapshot → hash
    ├──► Rule Engine: capture rule set → hash
    ├──► Profile: capture agent configuration
    └──► DAG Builder: create input_proof node
              │
              ▼
    For each step:
              │
              ├──► Verification Layer: run backend → emit verification_step node
              ├──► Ontology Layer: run cross-reference → emit cross_reference node
              ├──► θ-Rule Engine: check rules → emit rule_enforcement node
              └──► DAG Builder: create status_assignment node
              │
              ▼
    Audit Complete
              │
              ├──► DAG Builder: create aggregation node
              ├──► DAG Builder: create report_root node
              ├──► Report Generator: assemble report
              ├──► Signing: sign report root
              └──► Storage: store report by root hash
```

### 8.4 Storage Requirements

| Artifact | Size Estimate | Retention | Notes |
|----------|--------------|-----------|-------|
| DAG node | ~1 KB | Indefinite | Per node; ~300 nodes per 100-step audit |
| Full DAG | ~300 KB | Indefinite | Per audit |
| Ontology snapshot | ~10 MB | Indefinite | Per snapshot; ~1 snapshot per audit |
| Rule set | ~100 KB | Indefinite | Per rule set version |
| Report | ~50 KB | Indefinite | Per audit |
| Backend image | ~1 GB | Indefinite | Per backend version; shared across audits |

**Total per audit:** ~10.5 MB (dominated by ontology snapshot).
**For 1000 audits:** ~10.5 GB. Manageable for v0.1.

---

## §9: Implementation Phases

### Phase 0: Foundation (Weeks 1-2)

- Define DAG node schema and hash computation
- Implement DAG Builder (in-memory, serializes to JSON)
- Implement basic structural verification (Level 1)
- Store reports as gzipped JSON files

**Acceptance Gate R0:**
- DAG Builder produces valid nodes with correct hashes
- Structural verification passes on a test DAG
- Report can be serialized and deserialized without data loss

### Phase 1: Environment Capture (Weeks 3-4)

- Implement Snapshot Manager (creates ontology snapshots at audit start)
- Implement reproducibility recipe generation
- Capture backend versions and agent configuration
- Store snapshots by hash

**Acceptance Gate R1:**
- Snapshot Manager produces valid snapshots with correct hashes
- Reproducibility recipe captures all required environment fields
- A re-audit with the same recipe produces the same root hash

### Phase 2: Verification (Weeks 5-6)

- Implement spot-check verification (Level 2)
- Implement full re-audit verification (Level 3)
- Build `arx-verify` CLI tool
- Test against a corpus of 10 test audits

**Acceptance Gate R2:**
- Spot-check verification correctly identifies tampered reports
- Full re-audit produces matching root hashes for all 10 test audits
- `arx-verify` CLI tool is functional

### Phase 3: Signing (Weeks 7-8)

- Implement Ed25519 key generation and management
- Implement report signing
- Implement cross-signing protocol
- Publish public keys in a key directory

**Acceptance Gate R3:**
- Signatures are valid and verifiable
- Cross-signing protocol works end-to-end
- Key rotation and revocation are functional

### Phase 4: Integration (Weeks 9-10)

- Integrate DAG Builder with verification layer
- Integrate DAG Builder with event loop
- Integrate DAG Builder with θ-Rule engine
- Integrate Snapshot Manager with ontology layer
- End-to-end test: full audit with DAG, recipe, signing

**Acceptance Gate R4:**
- Full audit produces a valid DAG with all node types
- Reproducibility recipe is complete and accurate
- Report is signed and verifiable
- Cross-signing works with a second agent

### Phase 5: Production Readiness (Weeks 11-12)

- Performance optimization (DAG serialization, snapshot creation)
- Storage management (retention policies, pruning)
- Monitoring and alerting (failed verifications, key expirations)
- Documentation (verifier tool user guide, API reference)

**Acceptance Gate R5:**
- Audit with 100-step proof completes in < 5 minutes including DAG construction
- Snapshot creation adds < 10 seconds to audit start time
- Report generation adds < 5 seconds to audit completion
- All integration tests pass

---

## §10: Open Questions

1. **LLM determinism at temperature=0.** Most LLM APIs claim temperature=0 is deterministic, but floating-point non-determinism across hardware can produce different outputs. We should test this across multiple runs before relying on it for reproducibility.

2. **Storage cost for large audits.** A 1000-step proof with 5 backends per step would produce ~5000 DAG nodes. At ~1 KB per node, that's ~5 MB for the DAG alone. Plus the ontology snapshot (~10 MB). Total ~15 MB per audit. For 10,000 audits, that's 150 GB. Manageable but worth planning for.

3. **Verifier tool distribution.** The `arx-verify` CLI tool needs to be distributed to third parties. For v0.1, distribute as a static binary. For v0.2+, a web-based verifier that runs in the browser.

4. **Cross-signing cost.** Independent re-audit (Level 4) doubles the computational cost. For high-stakes audits this is acceptable, but for routine audits it's prohibitive. The protocol should default to Level 2 (spot-check) for cross-signing, with Level 4 available as an option.

5. **Key management at scale.** With multiple agents and key rotation, key management becomes complex. For v0.1, a simple key directory with manual rotation is sufficient. For v0.2+, consider a public key infrastructure (PKI) with automated rotation.

6. **Transparency log.** Publishing backend output hashes to a public transparency log (like Certificate Transparency) would enable verification without backend access. This is a v0.2+ feature but worth designing for.

7. **Report format evolution.** The report format will evolve as the system matures. Version the report schema (v0.1, v0.2, etc.) and maintain backward compatibility for verification of old reports.

---

## §11: Glossary

| Term | Definition |
|------|------------|
| **Content-addressable** | Stored and retrieved by cryptographic hash of content |
| **DAG** | Directed Acyclic Graph — the structure of audit steps with hash-chained dependencies |
| **Nonce** | Random seed used for non-deterministic steps, enabling deterministic replay |
| **Reproducibility** | The ability to produce the same audit result given the same inputs |
| **Verifiability** | The ability for a third party to check that an audit report is correct and untampered |
| **Cross-signing** | A second agent independently verifying and signing the audit report |
| **Root hash** | The cryptographic hash of the report root node, committing to the entire audit |
| **Reproducibility recipe** | The set of environment parameters needed to reproduce an audit |
| **Ontology snapshot** | A point-in-time capture of the ontology, stored by hash |
| **Spot-check** | Verifying a random sample of steps by re-running the backends |
