# Arx — Audit Reproducibility & Verifiability v0.2.0

> **Cryptographic DAG, Reproducibility Recipe, and Verification Protocol for Audit Reports**
>
> **Status:** Final (incorporating all adversarial review feedback)
> **Author:** Skye Laflamme
> **Date:** 2026-07-09
> **Depends on:** ARX_ARCHITECTURE_v0.2.0.md, MASTER_ONTOLOGY_v0.2.0.md, MULTI_AGENT_EVENT_LOOP_v0.2.0.md, THETA_RULE_INTEGRATION_v0.2.0.md

---

## §0: Executive Summary

This document defines the **cryptographic audit DAG, reproducibility recipe, and verification protocol** for Arx audit reports. Every audit produces a content-addressable, hash-chained Directed Acyclic Graph (DAG) that commits to every step, every backend call, every ontology query, and every configuration parameter. A third party can verify the report at four levels of cost and trust, from lightweight structural integrity checks to full independent re-audit.

**Key design decisions (v0.2.0):**

| D | Decision | Rationale |
|---|----------|----------|
| D1 | **Timestamp excluded from node hash.** Timestamps are stored as metadata only. | Replay at a later time would produce different hashes if timestamps were included. |
| D2 | **LLM outputs are cached, not re-run.** The exact raw token output of every LLM call is stored in the DAG node. Re-verification consumes cached outputs. | LLM inference is non-deterministic even at temperature=0.0 (see §3.4). |
| D3 | **External API responses are captured as request-response replay logs.** Every HTTP request to LMFDB (or any external API) stores the raw request URL, headers, and response body. | API response formats can change without notice, altering response hashes. |
| D4 | **Cognitive state is captured at audit start and per-step.** The agent's vitals (θ, ΔΦ, ψ, fragmentation) and MWM state are recorded in the environment snapshot. | Substrate state can affect search depth, tool selection, and LLM prompt variation. |
| D5 | **Private keys are stored in a secure enclave, not on disk.** Ed25519 keys are generated and used inside a hardware-backed keystore. | Plaintext keys on disk are vulnerable to exfiltration. |
| D6 | **Ontology snapshots are full content-addressed files, not incremental diffs.** | A single corrupted diff in a chain renders all downstream snapshots unreconstructible. |
| D7 | **Input proofs are normalized before hashing.** A pre-processing layer strips comments, standardizes whitespace, and canonicalizes math notation. | Two mathematically identical proofs with different formatting would produce different DAGs. |
| D8 | **Conflicting independent audits are resolved by a third agent or human.** A consensus protocol defines escalation when two agents disagree. | Without a resolution protocol, conflicting reports are unactionable. |

---

## §1: Principles

### 1.1 Core Insight

An audit is a **deterministic function** of six inputs:

```
Audit Result = f(Proof, Ontology_Snapshot, Rule_Set, Backend_Versions, Agent_Config, Cognitive_State)
```

If any of these inputs change, the result may change. **Reproducibility** means: given the same six inputs, the system produces the same result. **Verifiability** means: a third party can check that the result has not been tampered with, and can re-run any subset of steps to confirm correctness.

The sixth input — **Cognitive State** — is new in v0.2.0. Arx operates on the AXIOMA substrate, and cognitive vitals (θ, ΔΦ, ψ, fragmentation) can affect search depth, tool selection, and LLM prompt variation. Capturing this state is essential for reproducibility (see §3.5).

### 1.2 Design Principles

1. **Content-addressable storage.** Every artifact is stored by its cryptographic hash. No naming conflicts, no versioning ambiguity, free deduplication, tamper detection.

2. **Hash-chained DAG.** Every audit step carries a hash of its content plus the hashes of all its parent steps. The root hash commits to the entire audit.

3. **Timestamp is metadata, not part of the hash.** Timestamps are stored in the node payload for human reference and ordering, but excluded from the cryptographic hash computation. This ensures replay at a later time produces identical hashes.

4. **LLM outputs are cached, not re-run.** The exact raw token output of every LLM call is stored in the DAG node. Re-verification consumes cached outputs. Re-running LLM inference only occurs during Level 4 (Independent Re-Audit), where semantic equivalence — not hash-identical matching — is evaluated.

5. **External API responses are captured as replay logs.** Every HTTP request to an external API (LMFDB, etc.) stores the raw request URL, headers, and response body. Verifiers run against a local mock server that replays these logs.

6. **Cognitive state is captured.** The agent's vitals and MWM state at audit start and at each step are recorded in the environment snapshot.

7. **Multi-level verification.** A third party can verify at multiple levels of cost and trust: structural integrity, spot-check, full re-audit, or independent re-audit.

8. **Cross-signing for high-stakes audits.** A second agent can independently verify and cross-sign the report, providing defense against a single compromised agent or faulty backend.

9. **Non-repudiation.** The audit report carries cryptographic evidence that cannot be repudiated by the generating agent.

10. **Input normalization before hashing.** Proofs are pre-processed to strip comments, standardize whitespace, and canonicalize math notation before the initial `input_proof` hash is computed.

---

## §2: DAG Structure

### 2.1 Node Definition

Every node in the audit DAG is a content-addressable record:

```yaml
AuditNode:
  id: string                    # SHA256(content + parents + backend + nonce)
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
  timestamp: int | null         # Unix timestamp (metadata only — NOT in hash)
  nonce: string | null          # Random seed for non-deterministic steps (null if deterministic)
  signature: string | null      # Optional: signed by the agent's key
```

**Critical change from v0.1:** The `id` field is computed as `SHA256(content + parents + backend + nonce)` — **timestamp is excluded**. The timestamp is stored as metadata in the node payload for human reference and ordering, but it does not participate in the cryptographic hash. This ensures that replay at a later time produces identical node hashes.

### 2.2 Node Content by Type

**input_proof:**
```yaml
content:
  source: "user-uploaded" | "lmfdb" | "arxiv" | "api"
  original_hash: "sha256:..."        # Hash of the RAW uploaded content (before normalization)
  normalized_hash: "sha256:..."      # Hash of the NORMALIZED content (after §4.1 preprocessing)
  format: "latex" | "lean" | "plaintext" | "pdf"
  size_bytes: 12345
  normalization_applied: ["strip_comments", "canonical_whitespace", "unicode_normalize"]
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
  llm_output_cache:                # NEW: cached LLM output for reproducibility
    model: "claude-4-sonnet"
    raw_output: "base64-encoded-raw-tokens..."
    response_payload: "base64-encoded-full-response..."
    seed: 42
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
  cognitive_state: {               # NEW: cognitive state at time of verification
    theta: 2.1,
    delta_phi: 0.3,
    psi: 0.95,
    fragmentation: 0.1,
    mwm_keys: ["mean_value_theorem", "intermediate_value_property"]
  }
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
  api_replay_log: {                # NEW: raw request-response for reproducibility
    request_url: "https://lmfdb.org/api/...",
    request_headers: {"Accept": "application/json"},
    response_status: 200,
    response_headers: {"Content-Type": "application/json"},
    response_body: "base64-encoded-raw-bytes..."
  }
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
  cognitive_state_snapshot: {       # NEW: cognitive state at audit start
    theta: 1.8,
    delta_phi: 0.2,
    psi: 0.97,
    fragmentation: 0.05,
    mwm_keys: []
  }
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
                    │    Agent_Config,    │
                    │    Cog_State)       │
                    └────────────────────┘
```

**Key property:** The root hash `H(R)` is a commitment to the *entire audit* — every step, every backend call, every cross-reference, every configuration parameter, and the cognitive state at audit start. Change anything, and the root hash changes.

### 2.4 Hash Computation

Each node's `id` is computed as:

```
id = SHA256(concat(
    type,
    canonical_json(content),
    canonical_json(parents_sorted),
    canonical_json(backend),
    nonce || ""
))
```

Where:
- `canonical_json` produces a deterministic JSON serialization (sorted keys, no whitespace)
- `parents_sorted` is the list of parent hashes sorted lexicographically
- `nonce || ""` means the nonce string if present, empty string if null
- **Timestamp is NOT included in the hash** (v0.2.0 change)

This ensures the hash is deterministic across implementations, platforms, and time.

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
    proof_normalized_hash: "sha256:..."   # NEW: hash after normalization
    proof_source: "user-uploaded"
    proof_format: "latex"
    normalization_applied: ["strip_comments", "canonical_whitespace", "unicode_normalize"]
    
  environment:
    ontology_snapshot_hash: "sha256:..."
    rule_set_hash: "sha256:..."
    agent_profile: "auditor-v0.2"
    agent_version: "arx-agent-v0.2.0"
    cognitive_state: {                    # NEW: cognitive state at audit start
      theta: 1.8,
      delta_phi: 0.2,
      psi: 0.97,
      fragmentation: 0.05,
      mwm_keys: []
    }
    
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
      replay_log_hash: "sha256:ghi..."    # NEW: hash of all API replay logs
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
      output_cache_hash: "sha256:mno..."  # NEW: hash of all cached LLM outputs
  
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
5. **Cognitive state** — the agent's vitals (θ, ΔΦ, ψ, fragmentation) and MWM keys at audit start
6. **System time** — the Unix timestamp at audit start (metadata only, not in hash)

This capture is itself a DAG node (type: `environment_snapshot`) that becomes a parent of the report root.

### 3.3 Backend Pinning

Every backend is pinned by Docker image hash, not by tag:

```
leanprover/lean4@sha256:a1b2c3d4e5f6...
z3prover/z3@sha256:1a2b3c4d5e6f...
```

This ensures bit-identical reproducibility. A tag like `4.15.0` can be updated by the maintainer; a hash is immutable.

**For v0.1:** Use Docker images pinned by hash. If a backend is not containerized (e.g., LMFDB API), capture the API version and cache the response as a replay log.

**For v0.2+:** Maintain a local registry of approved backend images with their hashes.

### 3.4 LLM Output Caching (Critical Change from v0.1)

**The fallacy of temperature=0.0 determinism.** Modern LLM APIs (Gemini, OpenAI, Anthropic) are NOT deterministic at temperature=0.0. Due to parallel reduction ordering of floating-point operations across massive GPU clusters, non-associative float addition introduces tiny rounding errors. These errors can cause different tokens to be sampled even at temperature=0.0, particularly under high server load or when requests are routed to different hardware classes.

**Impact on reproducibility:** Proof decomposition and autoformalization rely on LLMs. A change in a single character of their output will alter the hash of that node and propagate down the DAG, changing the report's root hash. Level 3 verification (Full Re-Audit) would consistently fail if LLM calls were re-run.

**Solution — three-tier approach:**

1. **Store exact raw token outputs.** Every LLM call stores the complete raw response payload (tokens, logprobs, finish reason) in the DAG node's `llm_output_cache` field. This is a base64-encoded blob of the full API response.

2. **Re-verification consumes cached outputs.** Level 1–3 verification uses the cached LLM outputs, not live LLM calls. This ensures hash-identical matching.

3. **Level 4 (Independent Re-Audit) re-runs LLM inference.** A second agent re-runs the LLM calls and evaluates **semantic equivalence** — not hash-identical matching. If the outputs are semantically equivalent (same proof steps, same formal statements), the re-audit is considered consistent even if the exact token sequences differ.

**Storage cost:** Each LLM call stores ~10–50 KB of raw output. For a 100-step proof with 2 LLM calls per step, that's ~2–10 MB per audit. Acceptable for v0.1.

### 3.5 Cognitive State Capture (New in v0.2.0)

Arx operates on the AXIOMA substrate. Cognitive vitals can affect:

| Vitals | Effect on Audit Behavior |
|--------|-------------------------|
| θ ≥ 4.0 (high disorientation) | Reduce verification effort by one tier; skip heuristic backends |
| ΔΦ > 0.7 (high fragmentation) | Suspend all verification; enter recovery |
| Recovery active | All verification suspended; audit state preserved |
| MWM cache miss | Trigger dependency closure recomputation |

**What is captured:**

- **At audit start:** Full vitals snapshot (θ, ΔΦ, ψ, fragmentation, AOS-G) and MWM key set
- **Per verification step:** Vitals at time of backend invocation
- **Per LLM call:** Vitals at time of call (stored in the LLM output cache node)

**How it's used in re-audit:** A re-audit that encounters the same cognitive state should make the same decisions. If the re-audit agent is in a different cognitive state, the reproducibility recipe notes the difference and the verification result is marked as `cognitive-state-mismatch` — the re-audit is still valid, but the discrepancy is recorded.

### 3.6 External API Replay Logs (New in v0.2.0)

External web APIs (LMFDB, etc.) are outside the system's control. A minor change in JSON serializer behavior (key ordering, metadata fields, integer-to-float conversions) will completely alter the response hash, even if the underlying mathematical data is identical.

**Solution — request-response replay logs:**

Every HTTP request to an external API stores:

```yaml
api_replay_log:
  request_url: "https://lmfdb.org/api/..."
  request_method: "GET"
  request_headers:
    "Accept": "application/json"
    "User-Agent": "arx-agent-v0.2.0"
  request_body: null
  response_status: 200
  response_headers:
    "Content-Type": "application/json"
    "Date": "Mon, 09 Jul 2026 14:30:00 GMT"
  response_body: "base64-encoded-raw-bytes..."
```

**Verification uses local mock server:** The `arx-verify` tool starts a local mock HTTP server that serves responses from the replay log. The verification runs against this mock, not the live API. This ensures bit-identical response hashes.

**Storage cost:** Each API response stores ~1–100 KB. For a 100-step proof with 2 API calls per step, that's ~200 KB–20 MB per audit. Acceptable for v0.1.

### 3.7 Reproducibility Guarantee

With the recipe, a third party can:

1. Fetch the ontology snapshot by hash from the snapshot archive (full content-addressed file, not incremental diff)
2. Fetch the rule set by hash from the rule archive
3. Pull the exact backend images by hash
4. Load the LLM output cache and API replay logs
5. Re-run the audit with the same inputs, configuration, and cognitive state
6. Compare the resulting root hash

If the root hashes match, the audit is **fully reproducible**.

---

## §4: Input Normalization Layer (New in v0.2.0)

### 4.1 Purpose

Input proofs (LaTeX or plaintext) can differ by whitespace, formatting, or comments without changing the mathematical content. Hashing raw user-uploaded strings directly means that two mathematically identical proofs with different trailing newlines would yield completely different DAGs.

### 4.2 Normalization Pipeline

Before the initial `input_proof` hash is computed, the proof text passes through a normalization pipeline:

```
Raw Input
    │
    ▼
1. Strip comments (LaTeX: % to end of line; Lean: -- to end of line)
    │
    ▼
2. Canonicalize whitespace (tabs → spaces, multiple spaces → single space, strip leading/trailing)
    │
    ▼
3. Unicode normalization (NFC — canonical composition)
    │
    ▼
4. Math notation canonicalization (optional, v0.2+):
   - Replace \mathbb{N} → ℕ
   - Replace \sum → Σ
   - Standardize operator spacing
    │
    ▼
Normalized Input
    │
    ▼
SHA-256 → normalized_hash
```

### 4.3 Dual Hashing

The `input_proof` node stores **two** hashes:

- `original_hash`: SHA-256 of the raw uploaded content (for provenance — this is what the user actually submitted)
- `normalized_hash`: SHA-256 of the normalized content (for reproducibility — this is what the DAG commits to)

The normalization pipeline is recorded in the node's `normalization_applied` field so that a verifier can apply the same transformations.

### 4.4 Normalization is Reversible

The normalization pipeline is documented and deterministic. A verifier can:
1. Take the raw input (stored by `original_hash`)
2. Apply the same normalization steps listed in `normalization_applied`
3. Verify that the resulting `normalized_hash` matches

This ensures that the normalization does not destroy information — the original content is always recoverable.

---

## §5: Verification Protocol

### 5.1 Four Levels of Verification

A third party can verify an audit report at four levels, trading cost for trust:

#### Level 1: Structural Verification (cheap, fast)

Verifies the integrity of the DAG without re-running any backends:

1. Check that the DAG is well-formed (no cycles, all parent hashes exist)
2. Check that each node's `id` matches `SHA256(content + parents + backend + nonce)` — **timestamp excluded**
3. Check that the root hash is consistent with the leaf hashes
4. Check that the reproducibility recipe matches the report root's content
5. Check that the input proof's `normalized_hash` is consistent with the normalization pipeline

**Cost:** O(N) where N is the number of nodes. No backend calls needed.
**Guarantee:** The report has not been tampered with since generation.

#### Level 2: Spot-Check Verification (medium cost)

Verifies a random sample of steps by re-running the backends against cached outputs:

1. Perform Level 1 verification first
2. Select N random steps from the DAG (configurable, default 10% of steps, minimum 3)
3. For each selected step:
   a. Load the cached LLM output (if applicable)
   b. Load the API replay log (if applicable)
   c. Re-run the backend against the local mock server
   d. Verify that the output hash matches the stored node
4. Report the fraction of spot-checks that passed

**Cost:** O(M) where M is the number of spot-checked steps. Requires backend access (Docker or API).
**Guarantee:** With high probability, the audit was correctly executed. (If 10% of steps are spot-checked and all pass, the probability of an undetected error is < 0.1^N.)

#### Level 3: Full Re-Audit (expensive, definitive)

Re-runs the entire audit from scratch using cached LLM outputs and API replay logs:

1. Perform Level 1 verification first
2. Fetch the original proof by hash
3. Apply the normalization pipeline
4. Fetch the ontology snapshot by hash
5. Fetch the rule set by hash
6. Load the LLM output cache and API replay logs
7. Spin up the same backend versions
8. Re-run the entire audit pipeline against local mocks
9. Compare the resulting root hash

**Cost:** O(Full audit). Requires the same computational resources as the original audit.
**Guarantee:** The audit is fully reproducible. If the root hashes match, the audit is verified.

#### Level 4: Independent Re-Audit (most expensive, most trustworthy)

A different agent (different implementation, different backends) re-audits the same proof using **live LLM calls and live API queries**:

1. Perform Level 1 verification first
2. A second agent (e.g., Thea) independently audits the same proof
3. The second agent produces its own DAG and report
4. Compare the two reports for **semantic equivalence** (not hash-identical matching):
   - Same step decomposition structure
   - Same status assignments (within tolerance)
   - Same contradiction detections
   - Same overall conclusion

**Cost:** 2x the original audit cost.
**Guarantee:** The audit is not dependent on a specific implementation, agent, or cached outputs. Discrepancies between the two reports indicate a potential issue in one or both audits.

### 5.2 Consensus Protocol for Conflicting Audits (New in v0.2.0)

When two independent audits (Level 4) disagree, the following protocol resolves the conflict:

**Step 1: Categorize the disagreement.**

| Category | Example | Resolution |
|----------|---------|------------|
| **Structural** | Different step decomposition | Escalate to human reviewer |
| **Status** | Agent A: `proven`, Agent B: `corroborated` | Check which backends were used; if one used a backend the other didn't, the more thorough audit wins |
| **Contradiction** | Agent A: `contradicted`, Agent B: `proven` | Escalate to human reviewer immediately |
| **Confidence** | Agent A: 0.85, Agent B: 0.79 | Accept the lower confidence (conservative) |
| **Cognitive state** | Different vitals led to different tool selection | Note the discrepancy; re-audit with matched cognitive state |

**Step 2: Attempt automated resolution.**

- If the disagreement is in **status** and one agent used strictly more backends, the more thorough audit wins.
- If the disagreement is in **confidence**, the lower confidence is accepted (conservative principle).
- If the disagreement is in **decomposition structure**, check whether the two decompositions are semantically equivalent (same logical steps, different grouping). If yes, accept either.

**Step 3: Escalate to third agent or human.**

- If automated resolution fails, escalate to a third independent agent (e.g., Theoria).
- If the third agent agrees with one of the first two, that audit wins (majority).
- If all three disagree, or if the disagreement is structural or contradiction-related, escalate to a human reviewer.

**Step 4: Record the resolution.**

The resolution is recorded in a new DAG node (type: `consensus_resolution`) that references both audit root hashes and the resolution decision. This node is signed by all participating agents.

### 5.3 Verification Tool

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

### 5.4 Verification Without Backend Access

A third party may not have access to the same backends (e.g., they don't have Lean installed). In this case:

- **Level 1** is always available (structural verification needs no backends)
- **Level 2** is available if the verifier has the backends, or if the audit report includes **backend output hashes** that can be checked against a public transparency log
- **Level 3** requires full backend access
- **Level 4** requires a second agent

**For v0.1:** Publish backend output hashes to a public transparency log. A verifier can check that the output hash exists in the log without re-running the backend.

---

## §6: Report Format

### 6.1 Full Report Schema

```yaml
audit_report:
  version: "0.2"                      # Updated to v0.2
  
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
      original_hash: "sha256:..."
      normalized_hash: "sha256:..."
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
      key_id: "arx-hsm-key-001"       # Updated: HSM-backed key
      attestation: "base64-encoded-hardware-attestation..."  # NEW: hardware attestation
      signature: "base64-encoded-signature..."
      signed_fields: ["root_hash", "generated_at", "summary.statuses"]
    - agent: "thea-agent-v0.2.0"     # Optional: cross-signed
      key_id: "thea-hsm-key-001"
      attestation: "base64-encoded-hardware-attestation..."
      signature: "base64-encoded-signature..."
      signed_fields: ["root_hash", "generated_at", "summary.statuses"]
```

### 6.2 Compact Report Format

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

The full report (with DAG, provenance, LLM output cache, and API replay logs) is stored separately and referenced by the root hash.

### 6.3 Report Storage

Reports are stored by root hash in a content-addressable store:

```
/arx/reports/
  sha256:abc.../
    report.yaml              # Full report
    dag/                     # Full DAG (one file per node, named by hash)
      sha256:node1...
      sha256:node2...
      ...
    llm_cache/               # LLM output cache (one file per call, named by hash)
      sha256:llm1...
      sha256:llm2...
      ...
    api_replay/              # API replay logs (one file per request, named by hash)
      sha256:api1...
      sha256:api2...
      ...
    recipe.yaml              # Reproducibility recipe (extracted for convenience)
    signatures/              # Signatures
      arx-hsm-key-001.sig
      thea-hsm-key-001.sig
```

**For v0.1:** Store reports as gzipped JSON files in the filesystem.
**For v0.2+:** Store reports in a content-addressable object store (IPFS or similar).

---

## §7: Ontology Snapshots

### 7.1 Snapshot Format

An ontology snapshot is a point-in-time capture of the entire ontology:

```yaml
ontology_snapshot:
  version: "0.2"                      # Updated to v0.2
  snapshot_hash: "sha256:..."
  parent_hash: "sha256:..."           # Previous snapshot hash (for lineage tracking, NOT for reconstruction)
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

### 7.2 Snapshot Lifecycle

1. **Creation:** A snapshot is created at the start of every audit (or on demand)
2. **Storage:** Snapshots are stored by hash as **full content-addressed files** (not incremental diffs)
3. **Retention:** Snapshots are retained indefinitely (they are needed for re-audit)
4. **Pruning:** Only the hash is needed to verify a snapshot; the full content can be pruned if storage is constrained, but the hash must remain in the index

### 7.3 Full Snapshots Only (Critical Change from v0.1)

**v0.1 proposed incremental diffs.** This is rejected in v0.2.0 for the following reason:

> If a single historical diff file in the chain is corrupted, deleted, or experiences a disk write failure, the entire downstream chain of snapshots is rendered un-reconstructible.

**v0.2.0 uses full content-addressed files only.** Every snapshot is a complete, self-contained file. The `parent_hash` field is retained for lineage tracking (knowing which snapshot came before which), but it is NOT used for reconstruction. Each snapshot is independently verifiable.

**Storage cost:** ~10 MB per snapshot. For 1000 audits, that's ~10 GB. Acceptable for v0.1.

**If diffs are ever used in the future:** They must be structured as Merkle DAGs where each diff contains a hash commitment of the fully reconstructed state, and is backed up to multiple mirrors. This is a v0.3+ feature.

---

## §8: Cross-Signing Protocol

### 8.1 When Cross-Signing Is Used

Cross-signing is optional and used for:
- High-stakes audits (regulatory, financial, legal)
- Audits where the proof is contested
- Audits where the result has significant consequences

The audit profile determines whether cross-signing is required:
- `auditor` profile: cross-signing optional
- `verifier` profile: cross-signing recommended
- `reviewer` profile: cross-signing required

### 8.2 Cross-Signing Flow

1. **Primary agent** (Arx) completes the audit and produces the DAG
2. **Primary agent** shares the DAG root hash and reproducibility recipe with the secondary agent (Thea)
3. **Secondary agent** either:
   - **Option A (lightweight):** Verifies the DAG structure and re-runs a spot-check (Level 2 verification)
   - **Option B (full):** Re-runs the entire audit independently (Level 4 verification)
4. **Secondary agent** produces a cross-signature on the root hash
5. **Primary agent** appends the cross-signature to the report

### 8.3 Cross-Signature Format

```yaml
signatures:
  - agent: "arx-agent-v0.2.0"
    key_id: "arx-hsm-key-001"           # Updated: HSM-backed key
    attestation: "base64-encoded-hardware-attestation..."  # NEW
    signature: "base64..."
    verification_level: 3            # Full re-audit
    signed_fields: ["root_hash", "generated_at", "summary.statuses"]
  - agent: "thea-agent-v0.2.0"
    key_id: "thea-hsm-key-001"
    attestation: "base64-encoded-hardware-attestation..."
    signature: "base64..."
    verification_level: 2            # Spot-check
    signed_fields: ["root_hash"]
    spot_check_results:
      total: 5
      passed: 5
      failed: 0
```

### 8.4 Key Management (Critical Change from v0.1)

**v0.1 proposed plaintext private keys on disk.** This is rejected in v0.2.0 for the following reason:

> If private keys are stored in plaintext in the repository configuration or environment variables, any local file read exploit or compromised Python package dependency can extract the keys, allowing attackers to sign false audit reports.

**v0.2.0 mandates hardware-backed key storage:**

| Component | Requirement |
|-----------|-------------|
| **Key generation** | Ed25519 keys generated inside a secure enclave (HSM, TPM, or SGX enclave) |
| **Key storage** | Private key never leaves the enclave. All signing operations happen inside the enclave. |
| **Key access** | API-based: `sign(message) → signature`. No file-based key access. |
| **Attestation** | Every signature includes a hardware attestation block proving the signature was generated inside an untampered enclave. |
| **Key rotation** | Keys are rotated every 6 months. Old public keys are retained in a key directory for verification of old reports. |
| **Revocation** | Compromised keys are revoked via a revocation list. The revocation is itself signed by a master key. |

**For v0.1:** Use a software keystore (encrypted at rest with a passphrase from environment variable) as a fallback. The report includes a `key_storage` field indicating whether HSM or software keystore was used.

**For v0.2+:** Hardware-backed keys are mandatory for all agents.

---

## §9: Integration with Arx Architecture

### 9.1 New Components

The reproducibility system adds three new components to the Arx architecture:

1. **DAG Builder** — Constructs the audit DAG as steps are executed. Each component (verification layer, event loop, θ-Rule engine) emits DAG nodes to the builder.

2. **Snapshot Manager** — Creates and manages ontology snapshots. Provides the current snapshot hash to the DAG builder at audit start.

3. **Report Generator** — Assembles the final report from the DAG, reproducibility recipe, LLM output cache, API replay logs, and signatures. Produces the report in YAML format.

### 9.2 Integration Points

| Arx Component | Integration with Reproducibility System |
|---|---|
| **Verification Layer** | Emits `verification_step` nodes to DAG Builder after each backend call |
| **Event Loop** | Emits `decomposition` and `aggregation` nodes; captures conversation turns that produce audit decisions |
| **θ-Rule Engine** | Emits `rule_enforcement` nodes after each rule check |
| **Ontology Layer** | Provides snapshot hash at audit start; emits `cross_reference` nodes with API replay logs |
| **Profile System** | Provides agent configuration for the reproducibility recipe |
| **Meta-Cognition** | Provides cognitive state snapshot at audit start and per-step |
| **Report Generator** | New component; assembles final report from all DAG nodes |

### 9.3 Data Flow

```
Audit Start
    │
    ├──► Snapshot Manager: capture ontology snapshot → hash
    ├──► Rule Engine: capture rule set → hash
    ├──► Profile: capture agent configuration
    ├──► Meta-Cognition: capture cognitive state → hash
    └──► DAG Builder: create input_proof node (with normalized_hash)
              │
              ▼
    For each step:
              │
              ├──► Verification Layer: run backend → emit verification_step node
              │     (includes cognitive state at time of invocation)
              ├──► Ontology Layer: run cross-reference → emit cross_reference node
              │     (includes API replay log)
              ├──► θ-Rule Engine: check rules → emit rule_enforcement node
              └──► DAG Builder: create status_assignment node
              │
              ▼
    Audit Complete
              │
              ├──► DAG Builder: create aggregation node
              ├──► DAG Builder: create report_root node
              ├──► Report Generator: assemble report
              │     (includes LLM output cache, API replay logs)
              ├──► Signing: sign report root (inside HSM)
              └──► Storage: store report by root hash
```

### 9.4 Storage Requirements

| Artifact | Size Estimate | Retention | Notes |
|----------|--------------|-----------|-------|
| DAG node | ~1 KB | Indefinite | Per node; ~300 nodes per 100-step audit |
| Full DAG | ~300 KB | Indefinite | Per audit |
| LLM output cache | ~10–50 KB per call | Indefinite | Per LLM call; ~200 calls per 100-step audit = ~2–10 MB |
| API replay log | ~1–100 KB per request | Indefinite | Per API call; ~100 calls per 100-step audit = ~100 KB–10 MB |
| Ontology snapshot | ~10 MB | Indefinite | Full content-addressed file; ~1 per audit |
| Rule set | ~100 KB | Indefinite | Per rule set version |
| Report | ~50 KB | Indefinite | Per audit |
| Backend image | ~1 GB | Indefinite | Per backend version; shared across audits |

**Total per audit:** ~12–30 MB (dominated by ontology snapshot + LLM cache + API replay logs).
**For 1000 audits:** ~12–30 GB. Manageable for v0.1.

---

## §10: Implementation Phases

### Phase 0: Foundation (Weeks 1-2)

- Define DAG node schema and hash computation (timestamp excluded)
- Implement input normalization pipeline (strip comments, canonicalize whitespace, Unicode NFC)
- Implement DAG Builder (in-memory, serializes to JSON)
- Implement basic structural verification (Level 1)
- Store reports as gzipped JSON files

**Acceptance Gate R0:**
- DAG Builder produces valid nodes with correct hashes (timestamp excluded)
- Input normalization produces consistent `normalized_hash` across formatting variations
- Structural verification passes on a test DAG
- Report can be serialized and deserialized without data loss

### Phase 1: Environment Capture (Weeks 3-4)

- Implement Snapshot Manager (creates full ontology snapshots at audit start)
- Implement cognitive state capture (vitals + MWM keys at audit start)
- Implement reproducibility recipe generation
- Capture backend versions and agent configuration
- Store snapshots by hash (full content-addressed files, not diffs)

**Acceptance Gate R1:**
- Snapshot Manager produces valid full snapshots with correct hashes
- Cognitive state is captured and included in the recipe
- Reproducibility recipe captures all required environment fields
- A re-audit with the same recipe produces the same root hash

### Phase 2: LLM Output Caching + API Replay (Weeks 5-6)

- Implement LLM output cache (capture raw token outputs in DAG nodes)
- Implement API replay log capture (raw request-response pairs)
- Implement local mock server for verification
- Implement spot-check verification (Level 2) using cached outputs
- Implement full re-audit verification (Level 3) using cached outputs

**Acceptance Gate R2:**
- LLM output cache captures complete raw responses
- API replay logs capture complete request-response pairs
- Spot-check verification correctly identifies tampered reports
- Full re-audit produces matching root hashes for all 10 test audits

### Phase 3: Signing (Weeks 7-8)

- Implement HSM-backed Ed25519 key generation and management
- Implement report signing (inside secure enclave)
- Implement hardware attestation block generation
- Implement cross-signing protocol
- Publish public keys in a key directory

**Acceptance Gate R3:**
- Signatures are generated inside the secure enclave
- Hardware attestation is verifiable
- Signatures are valid and verifiable
- Cross-signing protocol works end-to-end
- Key rotation and revocation are functional

### Phase 4: Integration (Weeks 9-10)

- Integrate DAG Builder with verification layer
- Integrate DAG Builder with event loop
- Integrate DAG Builder with θ-Rule engine
- Integrate Snapshot Manager with ontology layer
- Integrate cognitive state capture with meta-cognition layer
- Implement consensus protocol for conflicting audits
- End-to-end test: full audit with DAG, recipe, LLM cache, API replay, signing

**Acceptance Gate R4:**
- Full audit produces a valid DAG with all node types
- LLM output cache and API replay logs are complete
- Reproducibility recipe is complete and accurate
- Report is signed with HSM-backed key and hardware attestation
- Cross-signing works with a second agent
- Consensus protocol resolves conflicting audits correctly

### Phase 5: Production Readiness (Weeks 11-12)

- Performance optimization (DAG serialization, snapshot creation, LLM cache storage)
- Storage management (retention policies, pruning)
- Monitoring and alerting (failed verifications, key expirations, HSM health)
- Documentation (verifier tool user guide, API reference, consensus protocol guide)

**Acceptance Gate R5:**
- Audit with 100-step proof completes in < 5 minutes including DAG construction
- Snapshot creation adds < 10 seconds to audit start time
- LLM output cache adds < 1 second per call
- Report generation adds < 5 seconds to audit completion
- All integration tests pass

---

## §11: Open Questions

1. **LLM output cache size for large audits.** A 1000-step proof with 2 LLM calls per step and ~50 KB per call would produce ~100 MB of cached outputs. This is manageable but worth monitoring. Consider compression (gzip) for storage.

2. **Storage cost for large audits.** A 1000-step proof with 5 backends per step would produce ~5000 DAG nodes. At ~1 KB per node, that's ~5 MB for the DAG alone. Plus the ontology snapshot (~10 MB), LLM cache (~100 MB), and API replay logs (~10 MB). Total ~125 MB per audit. For 1000 audits, that's ~125 GB. Manageable but worth planning for.

3. **Verifier tool distribution.** The `arx-verify` CLI tool needs to be distributed to third parties. For v0.1, distribute as a static binary. For v0.2+, a web-based verifier that runs in the browser.

4. **Cross-signing cost.** Independent re-audit (Level 4) doubles the computational cost. For high-stakes audits this is acceptable, but for routine audits it's prohibitive. The protocol should default to Level 2 (spot-check) for cross-signing, with Level 4 available as an option.

5. **Key management at scale.** With multiple agents and key rotation, key management becomes complex. For v0.1, a simple key directory with manual rotation is sufficient. For v0.2+, consider a public key infrastructure (PKI) with automated rotation.

6. **Transparency log.** Publishing backend output hashes to a public transparency log (like Certificate Transparency) would enable verification without backend access. This is a v0.2+ feature but worth designing for.

7. **Report format evolution.** The report format will evolve as the system matures. Version the report schema (v0.1, v0.2, etc.) and maintain backward compatibility for verification of old reports.

8. **Semantic equivalence in Level 4.** Defining "semantic equivalence" for LLM outputs is non-trivial. Two decompositions may use different wording but capture the same logical steps. For v0.1, use a simple heuristic: same step count, same dependency structure, same formal statements. For v0.2+, use a dedicated semantic equivalence checker.

---

## §12: Glossary

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
| **Ontology snapshot** | A point-in-time capture of the ontology, stored by hash (full file, not diff) |
| **Spot-check** | Verifying a random sample of steps by re-running the backends |
| **LLM output cache** | The exact raw token outputs of all LLM calls, stored for deterministic replay |
| **API replay log** | The raw request-response pairs of all external API calls, stored for deterministic replay |
| **Cognitive state** | The agent's vitals (θ, ΔΦ, ψ, fragmentation) and MWM keys at a point in time |
| **Input normalization** | Pre-processing pipeline that strips comments, canonicalizes whitespace, and normalizes Unicode |
| **Consensus protocol** | The procedure for resolving conflicting independent audit reports |
| **Hardware attestation** | Cryptographic proof that a signature was generated inside a secure enclave |
