---
name: grace-realization-engine
description: >
  Grace is a deterministic realization and convergence engine — the execution-side counterpart
  to Etta Cognitive OS. Use this skill whenever the user asks to convert structured cognitive
  intent into canonical, verifiable, persistent artifacts; build execution-graph (DAG) based task
  runners with parallel scheduling; enforce governance/invariant checks before committing state;
  manage artifact identity, version lineage, and replace semantics; reconcile workspace state
  against external tool/MCP outputs; or when the user explicitly mentions "Grace", "convergence
  engine", "execution graph", "canonical artifact", "governance core", or "realization layer".
  Also trigger when designing the execution/persistence half of an Etta-style cognitive
  architecture, or any system that must turn LLM-derived intent into deterministic, auditable,
  governed actions against real tools and storage.

license: public

metadata:
  author: BDEV
  version: 1.0.0
  category: execution-architecture
  architecture: modular-runtime-skill
  counterpart: etta-cognitive-os

compatibility: >
  Python 3.10+, requests or httpx, asyncio, dataclasses, json, uuid, datetime.
  Optional: anthropic SDK (for Claude-as-MCP-tool nodes), sqlite3 or JSON files for workspace persistence.
---

# Grace — Realization & Convergence Engine — Skill Runtime

Grace is the **realization layer** that sits between cognitive reasoning and the real world. It
is the structural sister of Etta Cognitive OS: where Etta expands ambiguity into structured
intent, Grace collapses that intent into a single deterministic, verifiable, canonical artifact.

**Core principle:** Etta decides → Grace converges and executes → Workspace persists.

Grace does not reason, brainstorm, or explore alternatives. It does not "think out loud." It
takes structured intent as input and produces canonical, governed reality as output — or it
halts.

---

## Etta ↔ Grace Separation Law

| | Etta (Cognition) | Grace (Realization) |
|---|---|---|
| Function | Expands possibility space | Collapses possibility into reality |
| Output | Goals, hypotheses, decisions, confidence | Canonical artifacts, execution state |
| Allowed to reason probabilistically | Yes | No — convergence over exploration |
| Governs canonicality | No — may reference artifacts only | Yes — sole authority |
| May expand intent | Yes — this is its job | No — strictly forbidden |
| May bypass the other | No | No |

**Global system loop:** `Etta → Grace → Workspace → MCP → Grace → Etta`

**System states:** Stable / Degraded / Failure — determined by artifact consistency, resolved
execution state, and a synchronized workspace.

**Feedback dynamics:** Grace constrains future cognition (its canonical artifacts become ground
truth for Etta); Etta enriches future execution (its refined intents improve Grace's convergence
quality).

---

## System Architecture

```
ETTA INTENT (structured, schema-bound)
    │  Intent Contract
    ▼
 GRACE REALIZATION LAYER
    ├── Convergence Engine      ← collapse multiple interpretations into ONE executable path
    ├── Execution Graph Builder ← compile intent into a DAG (node = unit of work, edge = dependency)
    ├── Scheduler                ← resolve ready nodes, dispatch in parallel
    ├── MCP / Tool Executor      ← stateless calls to MCPs, APIs, Claude, file systems
    ├── Verification Engine     ← validate outputs against intent + identity rules
    ├── Artifact Manager        ← canonical commit, version lineage, replace semantics
    └── Governance Core          ← invariant enforcement + conflict arbitration (supreme authority)
    │
    ▼
 WORKSPACE (shared state)      ← artifact registry, execution logs, lineage graph (single source of truth)
    │
    ▼
 MCP / EXTERNAL SYSTEMS        ← non-semantic execution endpoints, no correctness assumed
    │
    ▼
 FEEDBACK → ETTA                ← canonical artifacts + execution state feed future cognition
```

**Authority hierarchy inside Grace (highest → lowest):**
Governance Core → Contracts → Runtime → Protocol → Error Ontology → Artifact Identity → Behavior Model

**Ecosystem-wide rule:** Grace's Governance Core is the supreme authority on the realization
side, exactly as Etta's Critic Engine is supreme on the cognition side. Neither layer bypasses
the other: Etta never writes directly to Workspace, and Grace never expands intent on its own.

---

## Core Execution Loop

Every Grace cycle follows this **strictly ordered**, deterministic state machine:

```
INTAKE → CONTEXT RESOLUTION → CONVERGENCE → PLAN GENERATION → EXECUTION → VERIFICATION → COMMIT → FEEDBACK
```

Steps:
1. **Intake** — receive structured intent (from Etta or any structured caller). No interpretation expansion allowed.
2. **Governance pre-check** — Governance Core validates the intent against global invariants before anything else runs.
3. **Context Resolution** — resolve Workspace state, artifact identity, and historical lineage relevant to this intent.
4. **Convergence Engine** — reduce multiple potential interpretations into a single deterministic execution path. Ambiguity is eliminated, never explored.
5. **Execution Graph Builder** — compile the converged path into an atomic, ordered, dependency-aware DAG.
6. **Scheduler** — compute the ready set (all dependencies completed) and dispatch eligible nodes, in parallel where possible.
7. **MCP / Tool Executor** — execute each node statelessly against MCPs, APIs, Claude, or local compute. No side effects assumed correct.
8. **Verification Engine** — validate node and aggregate outputs against intent constraints and identity rules. Hard gate — no bypass.
9. **Artifact Manager** — on verification pass, commit the canonical artifact (CREATE / REPLACE / MERGE / COLLAPSE / ARCHIVE). Enforces one canonical state per `artifact_id`.
10. **Workspace Sync** — persist artifact, execution log, and lineage update as the new single source of truth.
11. **Feedback** — return canonical artifacts and execution state to Etta for knowledge-graph update and future reasoning.

**Invariants — never violate:**
- No artifact may be committed without passing Verification (hard gate, no bypass)
- No canonical-state duplication — exactly one canonical artifact per `artifact_id`, ever
- No execution without Governance Core approval of the originating intent
- Grace never expands intent — it only collapses it; expansion is exclusively Etta's responsibility
- All contracts are schema-validated, versioned, and immutable once emitted — corrections require new versions, not edits
- MCPs are non-semantic endpoints — no layer may assume correctness of their output without verification
- If Governance Core cannot resolve a conflict, the system enters a **safe halted state** — it never guesses

---

## Implementation

Read `references/implementation.md` for full Python code covering:
- `GraceRuntime` — top-level orchestrator (`execute()`)
- `GovernanceCore` / `InvariantRegistry` — invariant enforcement gate, first to initialize, last to be bypassed
- `ExecutionEngine` — graph build + dispatch
- `ExecutionGraph` / `ExecutionNode` / `Scheduler` — DAG model and ready-node resolution
- `MCPExecutor` (async) — parallel node execution
- `MCPClient` / `MCPAdapter` — real tool invocation with governance validation before and after
- `Artifact` / `ArtifactManager` — canonical commit and identity enforcement
- `EttaInterface` / `EttaToGraceTranslator` — the Etta → Grace bridge
- `ErrorRecovery` — minimal recovery dispatcher
- Worked examples: minimal run, full Etta→Grace flow, direct MCP tool call

Read `references/schemas.md` for all canonical contracts and data shapes:
- Intent / Artifact / Execution / Verification / Reconciliation / State Contracts
- Base Contract structure (shared by all contract types)
- Execution Graph Node schema + execution state enum
- Global Message Envelope (shared with Etta)
- Failure / Error schema and error taxonomy
- Artifact lifecycle, commit semantics, and realization states

Read `references/governance.md` for the Governance Core model:
- Conflict classification (Type A–D) and the resolution pipeline
- Severity matrix (S0–S5), detectability spectrum, recoverability classes
- Recovery algorithm framework and containment architecture
- Escalation rules and the safe-halt condition

---

## When Implementing Grace as a SKILL/Plugin

### Minimal Viable Grace (context-window only, no external persistence)

For an LLM agent operating inside a single context window (e.g., a Claude SKILL with no
backing database), Grace state lives as a structured dict injected into reasoning:

```python
grace_state = {
    "intent": None,
    "execution_graph": [],          # list of node dicts: {id, deps, status, payload}
    "artifacts": {},                # artifact_id -> canonical artifact dict
    "execution_log": [],            # append-only
    "governance": {
        "invariants_ok": True,
        "open_conflicts": []
    },
    "runtime_state": "IDLE"         # see references/schemas.md for full enum
}
```

Inject state into every planning step. Before any artifact write, run the Governance check
inline. Never hold two "canonical" entries for the same `artifact_id` in `artifacts`.

### As a Claude SKILL

The SKILL instructs Claude to:
1. Treat any incoming structured intent (from Etta, from the user, or from a prior turn) as
   Grace's Intake — do not silently reinterpret or expand it.
2. Run a lightweight Governance pre-check: are required fields present, is the target
   `artifact_id` known, are there open S4/S5 conflicts logged in state?
3. Build an explicit, ordered execution plan (a DAG if there are independent sub-tasks) before
   taking any action — even a "plan" of one node is still a plan.
4. Execute steps, calling tools/MCPs as needed, treating their outputs as unverified until
   checked against the original intent.
5. Only after verification, "commit" the result — i.e., state plainly what the new canonical
   answer/artifact is, superseding any prior draft.
6. Log the cycle (what was decided, what was verified, what was committed) so future turns can
   reference it as ground truth.

### As a Python Plugin / External Agent

Use the full implementation from `references/implementation.md`. `GraceRuntime` wraps the entire
intake → convergence → execution → verification → commit pipeline and exposes a single
`execute(intent)` entry point for callers (Etta, a CLI, an API).

---

## Failure Handling

| Failure | Response |
|---|---|
| Convergence failure (residual ambiguity) | Re-enter Convergence with updated constraints; bounded retries, then escalate |
| Execution / MCP failure | Bounded retry; on exhaustion, roll back the node, mark it failed, block dependents |
| Verification failure | Block commit; return to Planning or escalate to Governance Core |
| Artifact identity collision | Halt the commit; route to the Reconciliation protocol |
| Reconciliation failure (Workspace ↔ MCP divergence) | Iterative reconciliation loop until consistent, or declare non-recoverable |
| Protocol / contract violation | Halt execution immediately; no state mutation permitted |
| Governance conflict deadlock (S4/S5 severity) | System enters a safe halted state; manual intervention required |

---

## Key Design Rules

- **Convergence over exploration** — Grace collapses ambiguity; it never explores it. Exploration is Etta's job.
- **Canonical state is singular** — exactly one canonical artifact per `artifact_id`, at all times.
- **Verification gates every commit** — no artifact reaches Workspace without passing verification. No exceptions.
- **Contracts are immutable once emitted** — corrections create new versioned contracts; nothing is edited in place.
- **Workspace is the single source of truth** — reconciliation always resolves toward Workspace state, not toward whichever system spoke last.
- **MCPs guarantee nothing semantically** — Grace verifies; it never assumes a tool's output is correct just because the call succeeded.
- **Governance Core has the final word** — an unresolved conflict halts the system safely rather than letting it proceed on a guess.
