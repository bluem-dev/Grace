# Error Ontology Specification

**Version:** 1.0
**Status:** Normative
**Document Type:** Public Specification
**Scope:** Comprehensive failure ontology across Execution, Convergence, Artifact, Reconciliation, Protocol, and System layers.

---

# Purpose

This document defines a formal ontology of every failure state recognized within Grace: structured failure taxonomy, detection signals, severity matrices, propagation rules, and deterministic recovery algorithms.

It is the normative source for the failure model summarized operationally in:

```text
skill/references/governance.md
```

and referenced throughout `specifications/grace_runtime_spec.md`, `specifications/grace_protocol_spec.md`, and `specifications/grace_contract_spec.md`.

---

# Executive Summary

Every failure in Grace is modeled as a structured entity — never a bare exception string.

This specification defines that structure, classifies failures by domain and severity, and defines the deterministic recovery process each classification implies.

---

# Failure Ontology Model

Failures are modeled with the following attributes:

* `failure_id`
* `domain`
* `severity`
* `detectability`
* `recoverability`
* `propagation_scope`
* `causal_origin`
* `recovery_path`

The literal schema is defined in `skill/references/schemas.md` under Failure / Error Schema.

---

# Domain Classification System

Every failure belongs to exactly one of six domains:

* **Execution Domain** — tool failure cascades, external API instability, runtime desynchronization, execution-graph corruption.
* **Convergence Domain** — multi-path collapse instability, constraint contradiction loops, unresolved ambiguity residuals.
* **Artifact Integrity Domain** — identity collision clusters, lineage fragmentation, partial canonical commits, version entanglement.
* **Reconciliation Domain** — Workspace ↔ MCP divergence, stale-state propagation chains, inconsistent canonical mapping.
* **Protocol Domain** — schema violations, version mismatches, contract breaches, broken message-immutability.
* **System Domain** — feedback-loop instability, recursive degradation, cross-layer desynchronization, global state corruption.

---

# Severity Matrix

| Level | Meaning |
|---|---|
| S0 | Negligible |
| S1 | Low |
| S2 | Moderate |
| S3 | High |
| S4 | Critical |
| S5 | System-Corrupting |

No canonical artifact may be committed while any unresolved S4 or S5 failure exists anywhere in the system.

---

# Detectability Spectrum

Failures are further classified by how readily they surface:

```text
Immediate (runtime-visible) → Delayed (post-execution) → Latent (hidden until a dependency triggers it) → Emergent (appears only under load)
```

---

# Recoverability Classes

```text
Auto-Recoverable → Retry-Recoverable → Rollback-Recoverable → Reconciliation-Recoverable → Non-Recoverable
```

These classes form an ordering of last resort: Grace always attempts the least invasive applicable recovery class before escalating to a more invasive one.

---

# Execution Failure Deep Model

Execution-domain failures include:

* Tool failure cascades — a single tool failure that propagates across dependent nodes.
* External API instability.
* Runtime desynchronization between the execution graph's recorded state and the actual state of in-flight work.
* Execution graph corruption — a graph that no longer satisfies the Directed/Acyclic/Deterministic properties defined in `specifications/grace_execution_graph_spec.md`.

---

# Convergence Failure Deep Model

Convergence-domain failures include:

* Multi-path collapse instability — the Convergence Engine cannot settle on a single executable path.
* Constraint contradiction loops — constraints that, taken together, admit no valid path.
* Unresolved ambiguity residuals — ambiguity that survives convergence rather than being eliminated by it.

---

# Artifact Failure Deep Model

Artifact-domain failures include:

* Identity collision clusters — multiple operations contending for the same `artifact_id` in ways that cannot all be honored.
* Lineage fragmentation — a break in the version chain defined in `specifications/grace_artifact_identity_spec.md`.
* Partial canonical commits — a commit that completed some but not all of its required side effects.
* Version entanglement — ambiguity over which version of an artifact a given operation actually targeted.

---

# Reconciliation Failure Deep Model

Reconciliation-domain failures include:

* Workspace–MCP divergence that cannot be resolved within bounded retries.
* Stale state propagation chains — outdated state that continues to inform downstream decisions after it should have been superseded.
* Inconsistent canonical mapping — Workspace and an MCP each report a different artifact as canonical for the same identity.

---

# Protocol Failure Model

Protocol-domain failures include:

* Schema violation.
* Version mismatch between participants.
* Contract breach — a participant acting outside what its contract type permits.
* Invalid message-immutability breaks — an attempt to alter an already-emitted message rather than emitting a new version.

---

# System Failure Model

System-domain failures include:

* Feedback-loop instability between Etta and Grace.
* Recursive degradation — a failure whose own recovery attempt introduces a further failure of the same or worse severity.
* Cross-layer desynchronization.
* Global state corruption.

---

# Failure Causal Graphs

All failures are nodes in directed causal graphs representing dependency propagation and systemic impact chains.

A failure's position in this graph — not just its domain or severity — determines how far its containment boundary must extend.

---

# Failure Propagation Rules

Failures propagate downstream by default, unless isolated by an explicit containment boundary defined at the execution-unit (node) level.

A single failed node blocks its direct dependents without corrupting sibling branches of the execution graph.

---

# Containment Architecture

Isolation layers prevent cross-domain contamination through bounded execution contexts.

A failure originating in the Execution Domain, for example, must not be allowed to silently corrupt Artifact Integrity Domain state simply because the failing node happened to be mid-commit.

---

# Detection Signal Model

Failures are detected via:

* State invariant checks.
* Schema validation.
* Checksum or identity mismatch.
* Execution divergence metrics.

---

# Invariant-Based Detection

Any violation of canonical artifact identity or execution determinism triggers immediate failure registration.

Detection in Grace is invariant-driven, not exception-driven: a failure is recognized because an invariant no longer holds, not merely because something threw an exception.

---

# Recovery Algorithm Framework

```text
Detect → Classify → Isolate → Attempt Recovery Path → Validate State → Commit or Escalate
```

---

# Retry Policy Model

Retries are bounded and deterministic.

Retries must never amplify convergence instability — a retry that would produce a worse ambiguity than the one it started with is rejected in favor of escalation.

---

# Rollback Semantics

Rollback restores the last verified canonical state snapshot, with full lineage preservation.

Rollback never erases history; it only stops the lineage from advancing further along a failed path.

---

# Re-convergence Algorithm

Re-convergence rebuilds the execution graph from the last stable convergence node, under updated constraints, rather than restarting the entire cycle from Intake.

---

# Partial Recovery Model

Partial recovery is allowed only if artifact integrity remains intact and verification still passes on the partial result.

A partial result that cannot pass Verification on its own terms is not eligible for partial recovery — it is treated as a full failure of that node.

---

# Non-Recoverable State Handling

A non-recoverable failure triggers:

* A system halt.
* A full audit log of the failure and everything that led to it.
* A manual intervention requirement.

Grace does not invent a resolution it cannot verify.

---

# Cross-Domain Failure Mapping

Failures in one domain must map to dependent domains via the dependency-graph propagation rules described above — a Reconciliation Domain failure that originated from an Artifact Integrity Domain issue must be traceable back to that origin, not recorded as if it arose independently.

---

# Systemic Stability Index

System health is measured via a stability index derived from failure frequency, propagation depth, and recovery success rate over time.

This index is observational, not load-bearing: it informs `docs/grade_roadmap.md`'s Observability Layer (v1.1) but does not itself gate any commit.

---

# Error Compression Principle

Repeated identical failures are compressed into a single canonical failure record, to avoid redundancy explosion in the failure log.

---

# Failure Versioning

All failures are versioned and stored as immutable diagnostic artifacts, following the same immutability rule that governs every other artifact Grace commits.

---

# Auditability Requirement

Every failure must be fully traceable, from its origin trigger to its final system resolution or halt state.

---

# Grace Integrity Rule

No canonical artifact may be committed while unresolved S4 or S5 failures exist.

---

# Normative Summary

This extended ontology defines a complete deterministic failure space, enabling auditability, recovery, and system resilience across every layer of Grace.
