# Execution Graph Specification

**Version:** 1.0
**Status:** Stable
**Document Type:** Public Specification

---

# Purpose

This document defines how Grace transforms a converged intent into an Execution Graph — a Directed Acyclic Graph (DAG) in which each node is a unit of execution and each edge is a dependency.

For the runtime phases surrounding graph construction and execution, refer to:

```text
specifications/grace_runtime_spec.md
```

For the executable reference implementation of every component described here, refer to:

```text
skill/references/implementation.md
```

---

# Objective

Transform:

```text
Intent
```

into:

```text
Execution Graph (DAG)
```

where node = unit of execution and edge = dependency.

---

# Why a Graph, Not a Sequence

A naive model treats work as a strict sequence:

```text
A
↓
B
↓
C
↓
D
```

Most real work does not actually have this shape. Grace instead models work as:

```text
      A
     / \
    B   C
     \ /
      D
```

where, for example, A = Analyze, B = Research, C = Extract, D = Consolidate.

B and C both depend only on A, and neither depends on the other — so they can execute in parallel, and D waits for both to complete.

---

# DAG Properties

Every execution graph Grace produces must satisfy three properties, without exception:

## Directed

A direction always exists between dependent nodes: `A → B` means B depends on A, never the reverse implicitly.

## Acyclic

A graph where a path loops back on itself is invalid:

```text
A → B
↑   ↓
└───┘
```

A cycle of this shape can never be scheduled — at least one node in it would have an unsatisfiable dependency — so Grace rejects any graph containing one rather than attempting to execute it partially.

## Deterministic

The same intent must produce the same graph shape, unless the underlying context has explicitly changed. A non-deterministic graph-construction step is, by definition, a Convergence Domain failure (see `specifications/grace_error_ontology_spec.md`).

---

# Core Model

## ExecutionNode

Each node carries:

* `id`
* `name`
* `node_type` (`mcp`, `compute`, or `llm`)
* `dependencies` — a list of node ids that must complete before this node becomes eligible
* `status` (`pending`, `ready`, `running`, `completed`, `failed`, `blocked`, `retrying`, `halted`)
* `payload`

## ExecutionGraph

The graph itself is a collection of nodes, addressable by id, with no separate edge list — dependencies are stored on the dependent node, and the graph's edges are derived from them.

---

# Example Graph

Given the intent:

> Analyze MediaInfo modern window architecture and produce a Go implementation proposal.

A representative graph is:

```text
N1  Analyze Architecture
N2  Extract UI Patterns        — depends on N1
N3  Extract Rendering Model    — depends on N1
N4  Produce Go Design          — depends on N2, N3
N5  Generate Documentation     — depends on N4
```

N2 and N3 are independent of each other and may execute in parallel once N1 completes. N4 cannot start until both N2 and N3 have completed. N5 cannot start until N4 completes.

---

# Scheduler

The Scheduler determines what can run now.

## Ready Rule

A node is eligible for execution if and only if every entry in its `dependencies` list has reached `status: completed`.

Given the example graph above, once N1 completes:

* N2 — ready
* N3 — ready
* N4 — blocked (N2 and N3 not yet both completed)

The Scheduler computes this ready set fresh on every cycle. Readiness is never cached, because a single node completing or failing can change the readiness of every node downstream of it.

---

# Parallel Execution

This is where the graph model earns its complexity over a plain sequence.

Without parallelism, a four-step chain takes four time units:

```text
N1 → N2 → N3 → N4    (4 units)
```

With the DAG model, where N2 and N3 are independent:

```text
N1 → (N2 ∥ N3) → N4    (3 units)
```

The Scheduler's job is to find every opportunity of this shape and exploit it, without ever executing a node whose dependencies have not yet completed.

---

# Asynchronous Execution

Nodes are dispatched to an asynchronous executor, which runs every node in the current ready set concurrently and waits for all of them to settle before the Scheduler recomputes the next ready set.

This produces the runtime loop:

```text
while graph.has_pending():
    ready = scheduler.get_ready_nodes(graph)
    results = run_parallel(ready, executor)
    update_graph(results)
```

A cycle that finds no ready nodes while pending nodes remain has encountered either a failure-induced block or a structural problem in the graph, and halts rather than spinning indefinitely.

---

# Governance Integration

Governance participates at both boundaries of node execution:

* **Before execution** — `governance.validate_node(node)` confirms the node is approved to run.
* **After execution** — `governance.validate_result(result)` confirms the result is trustworthy enough to feed forward to dependent nodes.

Neither boundary may be skipped, even for nodes that have succeeded on a prior retry.

---

# Artifact Generation

Each node, on successful execution, generates an artifact fragment — for example:

```json
{
  "artifact_id": "...",
  "node": "N3",
  "output": "...",
  "version": 1
}
```

These fragments are not individually canonical. They become canonical only after the Reconciliation Stage below combines and validates them.

---

# Reconciliation Stage

Once the DAG completes, all of its node-level artifact fragments enter reconciliation:

```text
all artifacts
↓
reconciliation
↓
canonical artifact
```

For example, the outputs of Research, Implementation Proposal, and Documentation nodes are reconciled into a single Final Canonical Artifact, governed by the rules in `specifications/grace_artifact_identity_spec.md`.

---

# Failure Recovery

When a node fails — for example, N3 in the example graph — its direct dependents are blocked:

```text
N3 failed
↓
N4 blocked
N5 blocked
```

Grace then applies, in order, whichever of the following the Governance Core's escalation rules permit for that failure's classification:

```text
retry → rollback → re-plan → halt
```

The full classification behind this sequence is defined in `specifications/grace_error_ontology_spec.md`.

---

# Execution States

A node occupies exactly one of the following states at any time:

```text
PENDING
READY
RUNNING
COMPLETED
FAILED
BLOCKED
RETRYING
HALTED
```

---

# Workspace Integration

Every state transition is persisted to Workspace as it happens, not batched until the graph completes:

```text
Node Created
Node Running
Node Completed
Artifact Created
Artifact Canonicalized
```

This gives any observer — human or Etta — a live, accurate view of graph progress at all times, and ensures that a crash mid-graph leaves a recoverable trail rather than an opaque gap.

---

# MCP Integration Within the Graph

Nodes of type `mcp` are the graph's boundary with the outside world. Dispatching one follows a fixed path:

```text
ExecutionEngine
   ↓
MCPAdapter
   ↓
MCPClient
   ↓
External Tool (Claude / API / system)
   ↓
Validated Response
   ↓
Artifact Manager
```

`MCPAdapter` is responsible for invoking Governance both immediately before the call (is this call allowed?) and immediately after (is the result trustworthy enough to feed forward?) — the same before/after pattern described under Governance Integration above, applied specifically to the MCP boundary.

Grace never assumes a tool's output is correct merely because the underlying HTTP call succeeded; that determination belongs to Verification, not to the MCP layer.

---

# Execution Graph Summary

The Execution Graph is how Grace turns a single converged intent into a set of independently schedulable, independently verifiable units of work — without sacrificing the determinism and traceability that the rest of the architecture depends on.
