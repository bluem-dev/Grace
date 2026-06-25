# Grace — Canonical Schemas

All Grace communication is structured, schema-bound, and versioned. Unstructured natural
language is never valid protocol input — if Grace receives free text, that text must first pass
through Etta (or an equivalent intake normalizer) before it reaches any of the schemas below.

---

## Base Contract Structure

Every contract in the system — regardless of type — shares this envelope:

```json
{
  "contract_id": "string",
  "version": 1,
  "timestamp": "ISO8601",
  "source": "string",
  "target": "string",
  "payload": {},
  "signature": "string | null",
  "trace_id": "string"
}
```

**Rules:**
- Contracts are immutable once emitted; corrections require a new versioned contract, never an edit.
- Non-conforming payloads are rejected outright — there is no partial acceptance.
- Every contract must be traceable end-to-end: from originating Etta intent, through Grace
  execution, to final artifact state.

---

## Intent Contract (Etta → Grace)

```json
{
  "intent_id": "string",
  "goal": "string",
  "constraints": ["string"],
  "context": {},
  "priority": 1,
  "expected_artifact_type": "string"
}
```

Rule: Grace must not expand this structure. It may only collapse it into a single execution
path. If the intent is ambiguous beyond what Convergence can resolve deterministically, Grace
rejects it back to Etta rather than guessing.

---

## Artifact Contract / Artifact Identity

```json
{
  "artifact_id": "string",
  "type": "string",
  "owner_domain": "string",
  "canonical_state": {},
  "version": 1,
  "lineage_ref": "string | null",
  "canonical_flag": true,
  "status": "draft | active | canonical | superseded | archived",
  "metadata": {},
  "ownership": "string",
  "timestamps": {
    "created": "ISO8601",
    "updated": "ISO8601"
  }
}
```

**Canonical Artifact Principle:** only one canonical artifact may exist per `artifact_id` at any
given moment. No parallel canonical states are ever allowed.

**Lifecycle:** `Draft → Active → Canonical → Superseded → Archived`

**Realization states** (orthogonal to lifecycle, describes execution progress toward
canonicalization): `Planned → In Progress → Verified → Canonicalized → Archived | Deprecated`

**Commit semantics:**

| Operation | Meaning |
|---|---|
| `CREATE` | New artifact identity established |
| `REPLACE` | Canonical state updated; identity unchanged |
| `MERGE` | Multiple revisions consolidated into one canonical state ("version collapse") |
| `COLLAPSE` | Multiple candidate states reduced to the single surviving canonical state |
| `ARCHIVE` | Artifact retired; no longer eligible for canonical status |

---

## Execution Contract

```json
{
  "operation_graph": [],
  "step_results": [],
  "logs": [],
  "runtime_state": "string",
  "success_flag": true
}
```

Execution contracts must map deterministically to MCP-compatible operations — there is no
"interpret at runtime" step once a graph has been built.

---

## Execution Graph Node Schema

```json
{
  "id": "string",
  "name": "string",
  "node_type": "mcp | compute | llm",
  "dependencies": ["string"],
  "status": "pending | ready | running | completed | failed | blocked | retrying | halted",
  "payload": {}
}
```

**Ready rule:** a node becomes eligible for execution only when every entry in `dependencies`
has `status: completed`. The Scheduler computes this set fresh on every cycle — readiness is
never cached.

**Graph properties (non-negotiable):**
- **Directed** — every edge has a defined direction, `A → B`.
- **Acyclic** — no path may loop back on itself.
- **Deterministic** — the same intent must always compile to the same graph shape, unless the
  underlying context has explicitly changed.

---

## Verification Contract

```json
{
  "validity_checks": [],
  "integrity_checks": [],
  "completion_status": "string",
  "mismatch_report": {}
}
```

Verification confirms that execution outcomes match the original intent specification. Failure
triggers rollback or a reconciliation loop — it never triggers a silent partial commit.

---

## Reconciliation Contract

```json
{
  "expected_state": {},
  "observed_state": {},
  "divergence_type": "string",
  "resolution_action": "retry | rollback | re_converge | escalate",
  "resolved": false
}
```

Resolves mismatches between the Workspace's expected artifact state and what an MCP/external
system actually reports.

---

## State Contract (Workspace)

```json
{
  "artifact_registry": {},
  "execution_logs": [],
  "lineage_graph": {},
  "last_reconciled": "ISO8601"
}
```

Rule: Workspace must never contain two conflicting canonical artifacts for the same
`artifact_id`. Workspace is the single source of truth for reconciliation across every other
system in the ecosystem.

---

## Global Message Envelope (Inter-Engine / Inter-System)

Shared structurally with Etta's envelope — both systems speak the same wire format so messages
can cross the Etta ↔ Grace boundary without translation:

```json
{
  "message_id": "string",
  "timestamp": "ISO8601",
  "source": "string",
  "destination": "string",
  "contract_type": "string",
  "version": 1,
  "payload": {},
  "trace_id": "string"
}
```

---

## Failure / Error Schema

```json
{
  "failure_id": "string",
  "domain": "execution | convergence | artifact | reconciliation | protocol | system",
  "severity": "S0 | S1 | S2 | S3 | S4 | S5",
  "detectability": "immediate | delayed | latent | emergent",
  "recoverability": "auto_recoverable | retry_recoverable | rollback_recoverable | reconciliation_recoverable | non_recoverable",
  "propagation_scope": "string",
  "causal_origin": "string",
  "recovery_path": "string | null",
  "trace_id": "string"
}
```

**Error taxonomy** (maps onto `domain` above): `IntentError`, `SchemaError`, `ExecutionError`,
`MCPError`, `VerificationError`, `ReconciliationError`.

See `references/governance.md` for the full severity matrix, detectability spectrum, and
recovery algorithm that operate on this schema.

---

## Engine I/O Contracts (Grace-internal)

### Convergence Engine
- Input: `{ intent, candidate_interpretations, context }`
- Output: `{ converged_path, rejected_paths, confidence }`
- Rule: must select exactly one path or explicitly reject the intent — never returns more than one live path.

### Execution Graph Builder
- Input: `{ converged_path, workspace_context }`
- Output: `{ nodes: [ExecutionGraphNode], edges: [[from, to]] }`
- Rule: output must be a valid DAG (directed, acyclic, deterministic).

### Scheduler
- Input: `{ graph }`
- Output: `{ ready_nodes: [node_id] }`
- Rule: a node is ready iff all of its dependencies have `status: completed`.

### MCP / Tool Executor
- Input: `{ node, permissions, trace_id }`
- Output: `{ status, result, artifacts, error }`
- Rule: stateless per call; never assumes correctness of the underlying tool's response.

### Verification Engine
- Input: `{ execution_results, original_intent, identity_rules }`
- Output: `{ passed: bool, mismatch_report }`
- Rule: always returns an explicit result; cannot silently approve a mismatch.

### Artifact Manager
- Input: `{ verified_results, target_artifact_id }`
- Output: `{ artifact, commit_operation, version }`
- Rule: sole authority for canonical state mutation; enforces the one-canonical-per-identity invariant.

### Governance Core
- Input: `{ intent | result | conflict_pair }`
- Output: `{ approved: bool, violated_invariants: [], resolution: any | null }`
- Rule: highest authority in the system; if it cannot resolve a conflict, it returns a halt signal rather than a guess.
