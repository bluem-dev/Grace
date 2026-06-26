# Contract Specification

**Version:** 1.0
**Status:** Stable
**Document Type:** Public Specification

---

# Purpose

This document defines the strict interface contracts governing every interaction within the Grace realization ecosystem.

Where `specifications/grace_protocol_spec.md` defines *who talks to whom and in what direction*, this document defines *the exact shape of what they exchange*.

For the literal JSON schemas implementing these contracts, refer to:

```text
skill/references/schemas.md
```

---

# Contract Model Overview

All system interactions are defined as versioned, schema-validated contracts.

No implicit behavior is allowed: if a behavior is not described by a contract, Grace does not perform it.

---

# Contract Participants

| Participant | Role |
|---|---|
| Etta | Intent Provider |
| Grace | Execution & Convergence Engine |
| Workspace | State Layer |
| MCP | Execution Adapters |

---

# Contract Types

Six contract types govern the entire system:

* Intent Contract
* Artifact Contract
* Execution Contract
* Verification Contract
* Reconciliation Contract
* State Contract

---

# Base Contract Structure

Every contract, regardless of type, shares the same envelope:

* `contract_id`
* `version`
* `timestamp`
* `source`
* `target`
* `payload`
* `signature`
* `trace_id`

No contract type is permitted to omit any of these fields, even when a given field is not semantically meaningful for that type — in that case it is explicitly set to `null`, never absent.

---

# Intent Contract

Defines structured intent emitted from Etta, including goal, constraints, context, and expected outputs.

This is the only contract type Etta is permitted to emit toward Grace. Etta cannot emit an Artifact Contract, an Execution Contract, or any other type — only Grace may produce those, as a consequence of processing an Intent Contract.

---

# Artifact Contract

Defines canonical artifact identity, state, version, lineage, and ownership rules.

The normative model behind this contract — lifecycle states, commit semantics, replace semantics — is defined in full in `specifications/grace_artifact_identity_spec.md`.

---

# Execution Contract

Defines operation graphs, execution steps, tool invocations, and runtime state transitions.

---

# Verification Contract

Defines validation rules, success criteria, integrity checks, and failure conditions.

A Verification Contract always resolves to an explicit pass or fail. It is never left in an ambiguous or pending state once Verification has been entered.

---

# Reconciliation Contract

Defines resolution rules for mismatches between Workspace state and MCP outputs.

---

# State Contract

Defines workspace state representation, including the artifact registry, execution logs, and lineage graph.

---

# Contract Versioning Rule

All contracts are strictly versioned.

Breaking changes require new contract versions. No silent updates are allowed — a contract consumer that receives an unrecognized version must reject the message rather than attempt best-effort interpretation.

---

# Schema Enforcement Rule

All payloads must conform to their defined schemas.

Non-conforming messages are rejected at the boundary, before they reach any business logic.

---

# Contract Immutability Rule

Once emitted, a contract cannot be modified.

Corrections require a new, explicitly versioned contract. This mirrors the Message Immutability Rule in `specifications/grace_protocol_spec.md`, applied at the data-shape level rather than the wire level.

---

# Identity Binding Rule

All contracts operating on an artifact must reference a valid `artifact_id`.

A contract that attempts to mutate artifact state without a resolvable identity is invalid by construction, regardless of any other field's validity.

---

# Execution Binding Rule

Execution contracts must map deterministically to MCP-compatible operations.

There is no "interpret at runtime" step once an Execution Contract has been constructed — its mapping to concrete tool calls is fixed at construction time.

---

# Etta → Grace Contract

Etta emits Intent Contracts, consumed by Grace without semantic ambiguity.

If an Intent Contract is ambiguous beyond what Convergence can resolve deterministically, Grace rejects it back to Etta rather than guessing at its meaning.

---

# Grace → Workspace Contract

Grace writes Artifact Contracts and State Contracts into Workspace as the canonical source of truth.

---

# Grace → MCP Contract

Grace translates Execution Contracts into MCP-compatible CRUD operations.

---

# MCP → Grace Contract

MCP returns Execution Results Contracts without semantic guarantees.

A successful Execution Results Contract indicates only that the call completed — not that its content is correct. Correctness is established only by Verification.

---

# Validation Pipeline

Every contract passes through, in order:

1. Schema Validation — does the payload match its declared structure?
2. Identity Validation — does any referenced `artifact_id` resolve correctly?
3. Consistency Validation — does the payload agree with current Workspace state?

A contract that fails any stage does not proceed to the next.

---

# Signature Model

Each contract may include a cryptographic or logical signature ensuring integrity and traceability.

Where a signature is present, it must be verified before the contract is treated as authentic. Where absent, the contract is treated as internally generated and is still subject to full schema and consistency validation.

---

# Traceability Requirement

Every contract must be traceable from its origin — the originating Etta intent — to the final artifact state it produced or attempted to produce.

---

# Conflict Resolution Rule

Conflicting contracts trigger the Reconciliation Protocol and block canonical updates until resolved.

A "conflicting contract" is any pair of contracts that, if both were honored, would produce two different canonical states for the same `artifact_id`.

---

# Compatibility Layer

Grace ensures backward compatibility via contract adapters and version mapping, so that older contract versions already in flight are not silently broken by a schema evolution.

---

# Failure Contract Model

Failures are expressed as explicit Failure Contracts with structured error metadata, never as bare exception strings.

---

# Error Taxonomy

Contract-level failures map onto one of six categories:

* `IntentError`
* `SchemaError`
* `ExecutionError`
* `MCPError`
* `VerificationError`
* `ReconciliationError`

The detailed domain, severity, and recoverability model behind this taxonomy is defined in `specifications/grace_error_ontology_spec.md`.

---

# Contract Lifecycle

A contract moves through the following lifecycle:

```text
Draft → Validated → Executed → Verified → Committed → Archived
```

A contract that fails Validation never reaches Executed. A contract that fails Verification never reaches Committed.

---

# System Integrity Rule

If contract validation fails, execution is halted and no state mutation is allowed — regardless of how much of the rest of the cycle had already succeeded.

---

# Contract Summary

Contracts define the strict structural backbone of Grace, ensuring deterministic, verifiable, and traceable interactions across every system component.
