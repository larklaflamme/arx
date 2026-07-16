# Design Review: AUDIT_REPRODUCIBILITY_v0.1.md

**Date of Review:** 2026-07-08  
**Document Reviewed:** [AUDIT_REPRODUCIBILITY_v0.1.md](file:///home/ubuntu/arx/design/AUDIT_REPRODUCIBILITY_v0.1.md)  
**Review Status:** Adversarial & Critical Security Review  

---

## 1. Executive Summary

[AUDIT_REPRODUCIBILITY_v0.1.md](file:///home/ubuntu/arx/design/AUDIT_REPRODUCIBILITY_v0.1.md) proposes a framework for audit report verifiability, reproducibility, and cryptographic integrity. It structures the audit process as a content-addressable, hash-chained Directed Acyclic Graph (DAG) and details a "reproducibility recipe" to allow third parties to re-run audits.

While the mathematical formulation is clean, an **adversarial review** reveals several critical design flaws that would prevent reproducibility in real-world scenarios, introduce security vulnerabilities in key management, and fail to guarantee bit-identical output. This review details these flaws and proposes concrete architectural fixes.

---

## 2. Critical Flaws & Security Vulnerabilities

### 2.1 The Fallacy of LLM Determinism at Temperature = 0.0
* **Flaw:** Section 3.4 asserts that setting `temperature=0.0` for LLM calls eliminates the need for nonce tracking because it is *"deterministic in most LLM APIs and eliminates the need for nonce tracking."*
* **Adversarial Reality:** This is factually incorrect for modern LLM APIs (Gemini, OpenAI, Anthropic). Due to parallel reduction ordering of floating-point operations across massive GPU clusters, non-associative float addition introduces tiny rounding errors. These errors can cause different tokens to be sampled even at `temperature=0.0`, particularly under high server load or when requests are routed to different hardware classes.
* **Impact:** Since proof decomposition (`decomposition` node) and autoformalization rely on LLMs, a change in a single character of their output will alter the hash of that node and propagate down the DAG, changing the report's root hash. Level 3 verification (Full Re-Audit) will consistently fail.
* **Fix:** The `reproducibility_recipe` must store the **exact raw token outputs** and full response payloads of all LLM calls inside the DAG nodes. Re-verification must consume these cached outputs rather than querying live LLMs. Re-running LLM inference should only occur during Level 4 (Independent Re-Audit) where semantic equivalence—not hash-identical matching—is evaluated.

### 2.2 Time-Dependency in Hash Chain Computations (Breaks Replay)
* **Flaw:** Section 2.4 includes the Unix `timestamp` in the node ID hash computation:
  `id = SHA256(concat(..., str(timestamp), nonce || ""))`
* **Adversarial Reality:** If a third party attempts to reproduce the audit (Level 3 Full Re-Audit) at a later date, the steps will be executed at a different time. If the nodes are hashed with the current execution time, the hashes will not match the original report's hashes.
* **Impact:** Out-of-the-box re-execution will always fail to reproduce the original root hash unless the verifier explicitly mocks the system clock for every step. Mocking system time across containerized backends (like Lean and Z3) is highly complex and error-prone.
* **Fix:** Exclude the execution timestamp from the cryptographic hash of intermediate DAG nodes. The timestamp should be stored as metadata in the node's payload, but not included in the hash input.

### 2.3 Non-Deterministic API Invariants and Formatting Shifts (LMFDB)
* **Flaw:** Section 3.3 suggests that for non-containerized external APIs like LMFDB, it is sufficient to capture the API version and cache responses.
* **Adversarial Reality:** External web APIs are outside the system's control. A minor change in LMFDB's JSON serializer (e.g., changing key order, adding metadata fields, or converting integer types to floats) will completely alter the response hash, even if the underlying mathematical data is identical.
* **Impact:** If the local cache is cleared or if a verifier attempts to run a clean verification hitting the live endpoint, the hash of the `cross_reference` node will change, invalidating the DAG.
* **Fix:** The `reproducibility_recipe` must package and commit to a **signed database dump or request-response replay log** (containing the raw headers and byte payloads of all external HTTP requests). Verifiers must run against this local mock database to ensure identical results.

### 2.4 Lack of Cognitive State Capture (AXIOMA Substrate)
* **Flaw:** The formula defines the audit as `Audit = f(Proof, Ontology, Rules, Backends, Config)`.
* **Adversarial Reality:** Arx operates on the AXIOMA substrate. Cognitive vitals—such as disorientation ($\theta$), fragmentation ($\Delta\Phi$), and Mathematical Working Memory (MWM) status—can alter search depth, LLM prompt variations, or tool selection.
* **Impact:** If the agent is in a highly fragmented state during the original audit, it might invoke the LLM fallback earlier. A clean agent running the re-audit might not experience this fragmentation, causing it to take a different cognitive path and generate a different DAG structure.
* **Fix:** The initial and turn-by-turn state of the agent's cognitive vitals and MWM keys must be captured and logged as part of the environment snapshot.

---

## 3. Security & Key Management Concerns

### 3.1 Plaintext Private Key Exposure on Disk
* **Concern:** Section 7.4 states that the private key for signing reports is *"stored securely, never shared."*
* **Risk:** In practice, agents run as automated scripts. If private keys are stored in plaintext in the repository configuration or environment variables, any local file read exploit or compromised python package dependency can extract the keys, allowing attackers to sign false audit reports.
* **Recommendation:** Mandate the use of a secure Key Vault, Hardware Security Module (HSM), or secure enclave (like Intel SGX or AWS Nitro Enclaves). The audit report should include a hardware attestation block verifying that the signature was generated inside an untampered enclave.

### 3.2 Cascading Failures in Incremental Ontology Snapshots
* **Concern:** Section 6.3 proposes storing ontology snapshots as incremental diffs.
* **Risk:** If a single historical diff file in the chain is corrupted, deleted, or experiences a disk write failure, the entire downstream chain of snapshots is rendered un-reconstructible.
* **Recommendation:** Snapshots of the SQLite graph database should be stored as full content-addressed files. If diffs are used, they must be structured as Merkle Directed Acyclic Graphs where each diff contains a hash commitment of the fully reconstructed state, and is backed up to multiple mirrors.

---

## 4. Architectural Gaps & Implementation Obstacles

### 4.1 Absence of Mathematical Normalization Layer
* **Gap:** Input proofs (LaTeX or plaintext) can differ by whitespace, formatting, or comments without changing the mathematical content.
* **Impact:** Hashing raw user-uploaded strings directly means that two mathematically identical proofs with different trailing newlines will yield completely different DAGs.
* **Recommendation:** Implement a pre-processing normalization layer that strips comments, standardizes white space, and parses math formulas into a canonical AST representation prior to the initial `input_proof` hashing.

### 4.2 Level 4 Verification Escalation Hazards
* **Gap:** Section 10.4 notes that Level 4 verification (Independent Re-Audit) doubles the cost.
* **Issue:** The document does not define the threshold or policy for resolving discrepancies between two independent audits. If Agent A stamps a step as `proven-with-contradiction` and Agent B stamps it as `corroborated`, the overall report is in conflict.
* **Recommendation:** Define a consensus and arbitration protocol (e.g., escalating to a third independent peer or human reviewer) to resolve conflicting multi-agent audit reports.
