# Runtime Specification

**Version:** 1.0
**Status:** Stable
**Document Type:** Public Specification

---

# Purpose

This document defines the live execution behavior of Grace: how intents are processed, how convergence is achieved, how artifacts are committed, and how runtime state is maintained and transitioned.

It specifies how Grace behaves at runtime as a convergence and realization engine, distinct from the static architectural description in:

```text
specifications/grace_architecture_spec.md
```

For the executable reference implementation of every component described here, see:

```text
skill/references/implementation.md
```

---

# Runtime Model Overview

Grace operates as a deterministic state machine, transforming structured intent into executed, verified, and canonical artifacts through a closed-loop lifecycle.

Given equivalent input and equivalent workspace state, Grace must produce equivalent output. This is not an aspiration — it is the runtime's central correctness criterion.

---

# Core Runtime Loop

Every Grace cycle follows this strictly ordered sequence:

```text
INTAKE → CONTEXT RESOLUTION → CONVERGENCE → PLAN GENERATION → EXECUTION → VERIFICATION → COMMIT → FEEDBACK
```

No phase may be skipped. No phase may be reordered. A cycle that cannot complete a phase transitions to a Failed state rather than proceeding optimistically into the next one.

---

# Runtime State Machine

The runtime occupies exactly one of the following states at any moment:

* Idle
* Intake
* Resolving Context
* Converging
* Planning
* Executing
* Verifying
* Committing
* Reconciling
* Completed
* Failed

Each state transition is deterministic and traceable: given the current state and the triggering event, the next state is fully determined, and the transition is recorded against the cycle's `trace_id`.

---

# Phase 1 — Intake

Grace receives structured intent from Etta or from another structured caller.

No interpretation expansion is allowed at this stage. If the incoming payload is unstructured natural language, it is rejected back to its source rather than reinterpreted by Grace — expansion is exclusively Etta's responsibility.

---

# Phase 2 — Context Resolution

Before convergence begins, Grace resolves:

* Current workspace state for any artifact referenced by the intent.
* The artifact's identity and current canonical version, if one exists.
* Relevant historical lineage that may constrain or inform the realization path.

---

# Phase 3 — Convergence

Multiple potential interpretations of the intent are reduced into a single executable trajectory.

Ambiguity is eliminated, not explored. If residual ambiguity cannot be resolved deterministically within bounded retries, the cycle escalates rather than guessing. See the Convergence Failure model in `specifications/grace_error_ontology_spec.md`.

---

# Phase 4 — Plan Generation

A deterministic execution graph is generated from the converged path.

Steps are atomic, ordered, and dependency-resolved. The full graph model — nodes, edges, scheduling, and parallel execution — is defined in `specifications/grace_execution_graph_spec.md`.

---

# Phase 5 — Execution

Grace executes operations through MCPs, APIs, file systems, or workspace connectors.

Execution is stateless and tool-driven: each node executes independently, and no node may assume a side effect performed by another node beyond what its declared dependencies guarantee.

---

# Phase 6 — Verification

Outputs are validated against intent constraints, artifact identity rules, and expected state transitions.

Verification is a hard gate. No output proceeds to Commit without passing it, and no partial pass is silently treated as a full pass.

---

# Phase 7 — Commit

Only verified outputs are committed into canonical artifact state.

Commit enforces replace semantics and identity continuity, as defined in `specifications/grace_artifact_identity_spec.md`. Governance Core is the final authority gating this phase; see `specifications/grace_governance_core_spec.md`.

---

# Phase 8 — Feedback

Execution results and artifact updates are returned to Etta for knowledge-graph update and future reasoning cycles.

Grace does not expose its internal convergence logic to Etta during feedback — only the resulting canonical artifacts, execution state, and any unresolved conflicts.

---

# Artifact State Handling

The runtime enforces canonical artifact identity resolution before any write operation.

No duplicate canonical states are permitted for the same `artifact_id` at any point during execution, including transiently mid-cycle.

---

# Workspace Synchronization

Workspace acts as the shared runtime memory layer. Every runtime state transition is reflected in workspace artifacts — execution logs, lineage updates, and canonical pointers are written as the cycle progresses, not only at completion.

---

# MCP Execution Layer

MCPs are treated as external execution adapters.

Grace translates semantic operations into MCP-compatible calls before dispatch, and never assumes an MCP response is semantically correct simply because the call returned successfully.

---

# Failure Handling Model

Runtime failures are categorized into five types, each with a deterministic recovery pathway:

* Intake Failure
* Convergence Failure
* Execution Failure
* Verification Failure
* Reconciliation Failure

The full severity, detectability, and recoverability classification for each type is defined in `specifications/grace_error_ontology_spec.md`.

---

# Retry and Recovery Logic

Grace may re-enter Convergence or Planning after a recoverable failure.

Grace may never bypass Verification before Commit, regardless of how many retries have already occurred or how close the cycle is to a deadline.

---

# Determinism Constraint

Given identical input and identical workspace state, Grace must produce identical execution outputs.

This constraint applies across the full cycle, not only within a single phase — a deterministic Convergence phase feeding a non-deterministic Execution phase still violates the runtime's correctness criterion.

---

# Concurrency Model

Grace supports isolated execution contexts per intent.

Parallel execution does not share mutable runtime state: two concurrently executing cycles never write to the same in-memory state object, even if they happen to target related artifacts. Any apparent conflict between them is resolved through Reconciliation, not through shared mutable state.

---

# Reconciliation Runtime

If an external system diverges from canonical artifact state, Grace initiates reconciliation loops until consistency is restored or a non-recoverable failure is declared.

---

# Observability Layer

All runtime transitions, decisions, and artifact mutations are logged for traceability and debugging.

A complete runtime log must be sufficient, on its own, to answer: what happened, why it happened, and what state the system reached as a result.

---

# Etta Interaction Model

Etta provides intent and receives feedback.

Grace does not expose internal convergence logic to Etta — only structured inputs and structured outputs cross the Etta ↔ Grace boundary.

---

# Execution Boundaries

Grace cannot expand intent — it can only collapse it.

Expansion is strictly Etta's responsibility. A runtime that finds itself needing to expand an intent in order to proceed has received an incomplete intent, and must reject it rather than infer the missing structure.

---

# Artifact Commit Enforcement

No artifact may be persisted without passing both verification and identity validation.

---

# Runtime Integrity Rules

* If identity resolution fails, execution halts.
* If verification fails, commit is forbidden.
* If convergence fails, retry is required before the cycle may proceed.

---

# Lifecycle Completion

A runtime instance is considered complete only when its artifact has been canonicalized and feedback has been returned to Etta.

A cycle that produces a canonical artifact but never returns feedback is not considered complete — completion requires both halves of the loop.

---

# System Stability Model

Grace prioritizes deterministic convergence over performance optimization or speculative execution.

A faster but non-deterministic path is never preferred over a slower deterministic one.

---

# The Etta → Grace Bridge

The runtime's Intake phase is the formal entry point for the Etta → Grace bridge — the mechanism that converts raw cognitive output into a structured Intent Contract the runtime can process.

The bridge performs, in order:

1. **Raw Input Reception** — text, an Etta-shaped payload, or any structured caller's request.
2. **Intent Extraction** — derives a goal from the raw input.
3. **Constraint Extraction** — derives any explicit constraints present in the input.
4. **Mode Detection** — classifies the intent as, for example, analytical or exploratory, to inform downstream convergence.
5. **Intent Contract Emission** — produces the structured payload that Intake consumes.

This bridge is implemented as `EttaInterface` and `EttaToGraceTranslator` in `skill/references/implementation.md`. `EttaInterface` performs steps 1–4; `EttaToGraceTranslator` consumes the result and performs the actual Execution Graph construction described in `specifications/grace_execution_graph_spec.md`.

The bridge is intentionally thin: it normalizes shape, not meaning. Any ambiguity it cannot resolve is carried forward into Convergence rather than guessed at the bridge layer.

---

# Runtime Summary

Grace's runtime is a closed, deterministic loop transforming structured intent into canonical reality through controlled convergence, execution, and verification.

It does not optimize for speed of action. It optimizes for the probability that the resulting artifact still makes sense the next time anyone — human or Etta — looks at it.
