# Protocol Specification

**Version:** 1.0
**Status:** Stable
**Document Type:** Public Specification

---

# Purpose

This document defines the communication protocols and structured contracts governing interactions between Etta (cognition), Grace (realization), Workspace (storage), and MCP (execution) layers.

This specification focuses on the shape and direction of communication. For the data structures each message carries, refer to:

```text
specifications/grace_contract_spec.md
```

For the canonical JSON schemas themselves, refer to:

```text
skill/references/schemas.md
```

---

# Protocol Overview

Grace Protocol defines a deterministic, schema-based communication system ensuring structured intent transfer, artifact identity preservation, and execution traceability.

Unstructured natural language is never valid protocol input. If a participant needs to send free text, that text must first pass through Etta — or an equivalent intake normalizer — before it reaches any Grace protocol boundary.

---

# System Participants

| Participant | Role |
|---|---|
| Etta | Cognitive Producer |
| Grace | Convergence & Execution Engine |
| Workspace | Shared State Layer |
| MCP | External Execution Adapters |

---

# Communication Model

All communication is structured, schema-bound, and versioned.

A message that does not validate against its declared schema is rejected outright — there is no partial acceptance and no best-effort parsing.

---

# Protocol Layers

Communication is organized into six layers:

* Intent Layer
* Convergence Layer
* Artifact Layer
* Execution Layer
* Verification Layer
* Feedback Layer

Each layer carries a distinct contract type, defined in `specifications/grace_contract_spec.md`.

---

# Etta → Grace Protocol

Etta emits structured intent objects containing goals, constraints, hypotheses, and recommended paths.

Grace consumes these as Intent Contracts. It does not negotiate their content — it either converges them into a single executable path or rejects the intent back to Etta.

---

# Grace → Etta Protocol

Grace returns canonical artifacts, execution logs, verification results, and state updates.

This is the only channel through which Grace communicates outcomes back to cognition. Grace never writes feedback anywhere other than through this protocol path.

---

# Grace → Workspace Protocol

Grace writes and updates canonical artifacts, execution states, and reconciliation records into Workspace.

Workspace is treated as a passive but authoritative store: Grace decides what becomes canonical, and Workspace simply persists that decision durably.

---

# Workspace → Grace Protocol

Workspace provides artifact state, historical lineage, and external system snapshots for reconciliation.

This is the read path that feeds the Context Resolution phase of the runtime (see `specifications/grace_runtime_spec.md`).

---

# Grace → MCP Protocol

Grace translates canonical operations into MCP-compatible CRUD operations without exposing semantic intent.

An MCP receiving a Grace-originated call never sees "realize this artifact" — it sees a structured tool call with a payload, indistinguishable from any other caller's request.

---

# MCP → Grace Protocol

MCP returns execution results, file states, and system responses without semantic guarantees.

Grace treats every MCP response as unverified data until it has passed through the Verification phase of the runtime.

---

# Intent Schema

An intent passed across this protocol must include:

* `intent_id`
* `goal`
* `constraints`
* `context`
* `priority`
* `expected_artifact_type`

The full JSON shape is defined in `skill/references/schemas.md` under Intent Contract.

---

# Artifact Schema

An artifact passed across this protocol must include:

* `artifact_id`
* `canonical_state`
* `version`
* `lineage`
* `status`
* `metadata`
* `ownership`

The full JSON shape is defined in `specifications/grace_artifact_identity_spec.md`.

---

# Execution Schema

Execution-layer messages must include:

* `operation_graph`
* `step_results`
* `logs`
* `runtime_state`
* `success_flag`

---

# Verification Schema

Verification-layer messages must include:

* `validity_checks`
* `integrity_checks`
* `completion_status`
* `mismatch_report`

---

# Protocol Determinism Rule

Given identical inputs, protocol exchanges must produce identical structured outputs across all participants.

This rule extends the runtime's determinism constraint into the communication layer: a protocol exchange that produces different output shapes for the same input is itself a protocol violation, independent of whether the underlying execution was deterministic.

---

# Identity Preservation Rule

Artifact identity must remain stable across all protocol transformations, regardless of the underlying storage system that ultimately persists it.

An artifact translated from a Grace internal representation into an MCP-compatible CRUD payload, and back, must resolve to the same `artifact_id`.

---

# State Synchronization Rule

Workspace is the single source of truth for artifact state reconciliation across systems.

When Grace and an MCP disagree about an artifact's state, the protocol always resolves toward what Workspace records as canonical, pending reconciliation — never toward whichever system reported most recently.

---

# Message Immutability Rule

Protocol messages are immutable once emitted.

Changes require new, explicitly versioned messages. Nothing on the wire is ever edited in place.

---

# Versioning Protocol

All protocol schemas are versioned.

Breaking changes require explicit migration paths, approved through the Governance Core's versioning model (see `specifications/grace_governance_core_spec.md`).

---

# Failure Protocol

Protocol-level failures are categorized as:

* Intent Rejection
* Convergence Failure
* Execution Failure
* MCP Failure
* Reconciliation Failure

Each maps onto the domain classification defined in `specifications/grace_error_ontology_spec.md`.

---

# Reconciliation Protocol

Grace resolves inconsistencies between expected artifact state and MCP-reported state through iterative reconciliation loops, continuing until consistency is restored or a non-recoverable failure is declared.

---

# Security Protocol

Only validated participants may emit or modify canonical artifact states.

A message arriving without a resolvable `source` identity is rejected before it is evaluated for any other property.

---

# Workspace Consistency Protocol

Workspace must never contain conflicting canonical artifacts for the same `artifact_id`, even transiently during a reconciliation cycle.

---

# Feedback Loop Protocol

Execution results are fed back into Etta for knowledge-graph updates and future intent refinement.

This is the same channel described under Grace → Etta Protocol above; it is repeated here to make explicit that feedback is a protocol obligation, not an optional courtesy.

---

# MCP Abstraction Rule

MCPs are treated as non-semantic execution endpoints.

No protocol layer may assume semantic correctness from MCP outputs purely on the basis of a successful response code.

---

# Cross-System Traceability

Every artifact mutation must be traceable from the originating Etta intent, through Grace execution, to the final MCP operation that realized it.

A `trace_id` introduced at Intake must be carried, unmodified, through every subsequent message in the cycle.

---

# Protocol Integrity Model

If protocol integrity is violated — a malformed message, a broken trace chain, an unresolvable source — Grace halts execution and initiates reconciliation rather than attempting to proceed around the violation.

---

# Extensibility Model

The protocol is designed to support future expansion into distributed, multi-Workspace and multi-MCP environments without breaking the contracts defined here. See the Distributed Realization Network direction in `docs/grade_roadmap.md`.

---

# Protocol Summary

Grace Protocol ensures deterministic, identity-preserving communication across cognitive, execution, and storage layers, enabling coherent realization of structured intent into canonical artifacts.
