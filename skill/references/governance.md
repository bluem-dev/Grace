# Grace — Governance Core & Failure Ontology

The Governance Core is **not a module** — it is a constraint layer sitting over every other
module in Grace. Nothing in Grace, including the Runtime itself, may mutate canonical state
without passing through it.

---

## Authority Hierarchy

```
Governance Core
    │  (highest authority — resolves contradictions between everything below)
    ▼
Contracts
    ▼
Runtime
    ▼
Protocol
    ▼
Error Ontology
    ▼
Artifact Identity
    ▼
Behavior Model
```

A lower-authority spec or subsystem can never invalidate or override a higher one without an
explicit, versioned migration approved by the Governance Core.

---

## Global System Invariants

Stored in a centralized **Invariant Registry**, referenced by every subsystem:

- **Determinism** — identical input + identical state → identical output, always.
- **Traceability** — every mutation traces back to an originating intent.
- **Identity Integrity** — artifact identity never silently changes.
- **Verification Closure** — nothing is "done" until verification has explicitly passed.
- **Contract Compliance** — no payload is accepted outside its schema.
- **Canonical State Uniqueness** — exactly one canonical artifact per `artifact_id`.

No subsystem may override a global invariant without Governance Core approval.

---

## Conflict Classification

| Type | Name | Description |
|---|---|---|
| A | Schema Conflict | Payload structure diverges from its declared contract |
| B | Semantic Conflict | Two valid-looking payloads disagree on meaning |
| C | State Conflict | Workspace and an external system disagree on artifact state |
| D | Systemic Conflict | Cross-layer contradiction that no single subsystem can resolve alone |

---

## Resolution Pipeline

```
Detect → Classify → Evaluate Impact → Select Resolution Strategy → Apply Migration or Override → Validate
```

Resolution priority is based on **system impact**, **dependency depth**, and **canonical
stability risk** — not on which subsystem raised the conflict first.

**Truth arbitration rule:** when two conflicting "truths" exist, the Governance Core selects the
canonical one based on invariant compliance and system-stability impact, not recency or
authorship.

**Escalation rule:** critical (Type D, or any S4/S5 severity) conflicts escalate straight to
system halt or forced reconciliation mode — they are never resolved optimistically.

---

## Failure Ontology Model

Every failure in Grace is a structured entity, not a bare exception string:

```json
{
  "failure_id": "string",
  "domain": "execution | convergence | artifact | reconciliation | protocol | system",
  "severity": "S0..S5",
  "detectability": "immediate | delayed | latent | emergent",
  "recoverability": "auto_recoverable | retry_recoverable | rollback_recoverable | reconciliation_recoverable | non_recoverable",
  "propagation_scope": "string",
  "causal_origin": "string",
  "recovery_path": "string | null"
}
```

### Domain Classification

- **Execution Domain** — tool failure cascades, external API instability, runtime desync, execution-graph corruption.
- **Convergence Domain** — multi-path collapse instability, constraint contradiction loops, unresolved ambiguity residue.
- **Artifact Integrity Domain** — identity collision clusters, lineage fragmentation, partial canonical commits, version entanglement.
- **Reconciliation Domain** — Workspace ↔ MCP divergence, stale-state propagation chains, inconsistent canonical mapping.
- **Protocol Domain** — schema violations, version mismatches, contract breaches, broken message-immutability.
- **System Domain** — feedback-loop instability, recursive degradation, cross-layer desync, global state corruption.

### Severity Matrix

| Level | Meaning |
|---|---|
| S0 | Negligible |
| S1 | Low |
| S2 | Moderate |
| S3 | High |
| S4 | Critical |
| S5 | System-Corrupting |

**Hard rule:** no canonical artifact may be committed while any unresolved S4 or S5 failure
exists anywhere in the system.

### Detectability Spectrum

`Immediate` (visible at runtime) → `Delayed` (visible only post-execution) → `Latent` (hidden
until a dependency triggers it) → `Emergent` (appears only under load).

### Recoverability Classes

`Auto-Recoverable` → `Retry-Recoverable` → `Rollback-Recoverable` → `Reconciliation-Recoverable`
→ `Non-Recoverable`.

---

## Recovery Algorithm Framework

```
Detect → Classify → Isolate → Attempt Recovery Path → Validate State → Commit or Escalate
```

- **Retry policy** — bounded and deterministic; retries must never amplify convergence
  instability (i.e., don't retry into a worse ambiguity than you started with).
- **Rollback semantics** — restores the last verified canonical snapshot, with full lineage
  preserved (rollback never erases history, it only stops advancing it).
- **Re-convergence algorithm** — rebuilds the execution graph from the last stable convergence
  node under updated constraints, rather than starting over from intake.
- **Partial recovery** — allowed only if artifact integrity remains intact and verification
  still passes on the partial result.
- **Non-recoverable handling** — triggers a system halt, full audit log, and a manual
  intervention requirement. Grace does not invent a resolution it cannot verify.

---

## Containment Architecture

Isolation layers prevent a failure in one domain from contaminating another. Failures propagate
downstream by default **unless** a containment boundary is explicitly defined at the execution
unit (node) level — so a single failed node should block its dependents without corrupting
sibling branches of the DAG.

---

## Detection Signals

Failures are detected via: state invariant checks, schema validation, checksum/identity
mismatch, and execution divergence metrics. Any violation of canonical artifact identity or
execution determinism triggers immediate failure registration — detection is invariant-driven,
not exception-driven.

---

## Error Compression & Auditability

- Repeated identical failures are compressed into a single canonical failure record rather than
  spamming the log with duplicates.
- Every failure is versioned and stored as an **immutable diagnostic artifact** — failures
  follow the same immutability rule as everything else Grace commits.
- Every failure must be traceable end-to-end, from its origin trigger to its final resolution or
  halt state.

---

## Final Governance Rule

If the Governance Core cannot resolve a conflict through this pipeline, the system enters a
**safe halted state**. It does not fall back to a best guess, it does not silently proceed with
a "probably fine" canonical artifact, and it does not let Grace's determinism guarantee become
a fiction. Halting is success here — it means the invariant that protected the system actually
held.
